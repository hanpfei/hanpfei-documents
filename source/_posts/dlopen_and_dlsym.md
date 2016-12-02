---
title: android下运行时动态链接dlopen()和dlsym()的实现
date: 2013-7-13 11:43:49
tags:
- Android
---

﻿在android中，就如同在Linux下一样，我们也可以在app中，运行时动态加载一些动态链接库，执行调用其中的函数等操作。实现这一切最终依靠的就是dlopen()等几个函数。关于这几个函数的原型机这些API的用法，可以参考 LINUX下动态链接库的使用-dlopen dlsym dlclose dlerror这一篇。而此处我们就来看一下，在android c标准库的bionic中，这些函数究竟是如何实现的。

<!--more-->

# dlopen()函数

首先是dlopen()函数。我们给这个函数传递一个动态连接库的文件名或路径，及一些flag，这个函数返回给我们一个动态链接库的句柄，以方便我们后续执行查找symbol等操作。这个函数的代码在android codebase中的路径为bionic/linker/dlfcn.c。我们来看一下它的实现：
```
void *dlopen(const char *filename, int flag)
{
    soinfo *ret;

    pthread_mutex_lock(&dl_lock);
    ret = find_library(filename);
    if (unlikely(ret == NULL)) {
        set_dlerror(DL_ERR_CANNOT_LOAD_LIBRARY);
    } else {
        soinfo_call_constructors(ret);
        ret->refcount++;
    }
    pthread_mutex_unlock(&dl_lock);
    return ret;
}
```
可以看到，这个函数本身的结构都还蛮清晰的。它主要完成几个事情：

1. 获取到一个互斥量。这个互斥量用于保护动态连接库的链表。
2. 根据动态连接库的路径名或文件名，查找一个动态链接库。这个过程实际上可能会返回一个已经加载了的动态连接库的句柄，但也可能会去新加载一个动态连接库，并返回其句柄。
3. 如果上一步查找动态连接库的操作成功完成，即找到了一个已经加载的动态连接库或者是成功的新加载了一个动态连接库。则它会去执行该动态连接库的初始化部分代码，并增加动态连接库的引用计数。而如果上一步的查找没有成功，则此处会设置出错信息，以便于user可以通过dlerror()了解到执行出错的情况。
4. 最后就是释放互斥量，并将结果返回给调用者。

此处我们可以看一下，那个所谓的动态连接库的句柄究竟是个什么东西。可以看到，它是一个soinfo的对象。

接下来我们先偏个楼，来看一下linker提供给user用的了解error信息的接口。

# dlerror()和set_dlerror()

当open一个动态链接库出错时，dl族函数的调用者可以通过dlerror()函数来对出错的原因有更多的了解，这个函数返回一个关于出错状况的error message。在linker open一个动态连接库出错时，它会自动的设置相关的error message。我们来看一下dlerror()函数和set_dlerror()函数的实现。

首先是dlerror()的code：
```
const char *dlerror(void)
{
    const char *tmp = dl_err_str;
    dl_err_str = NULL;
    return (const char *)tmp;
}
```
可以看到这个函数的实现非常简单，就只是返回一条error message dl_err_str而已。但这个error message究竟是如何设置的呢？它当然是set_dlerror()函数设置的：
```
/* This file hijacks the symbols stubbed out in libdl.so. */

#define DL_SUCCESS                    0
#define DL_ERR_CANNOT_LOAD_LIBRARY    1
#define DL_ERR_INVALID_LIBRARY_HANDLE 2
#define DL_ERR_BAD_SYMBOL_NAME        3
#define DL_ERR_SYMBOL_NOT_FOUND       4
#define DL_ERR_SYMBOL_NOT_GLOBAL      5

static char dl_err_buf[1024];
static const char *dl_err_str;

static const char *dl_errors[] = {
    [DL_ERR_CANNOT_LOAD_LIBRARY] = "Cannot load library",
    [DL_ERR_INVALID_LIBRARY_HANDLE] = "Invalid library handle",
    [DL_ERR_BAD_SYMBOL_NAME] = "Invalid symbol name",
    [DL_ERR_SYMBOL_NOT_FOUND] = "Symbol not found",
    [DL_ERR_SYMBOL_NOT_GLOBAL] = "Symbol is not global",
};

#define likely(expr)   __builtin_expect (expr, 1)
#define unlikely(expr) __builtin_expect (expr, 0)

pthread_mutex_t dl_lock = PTHREAD_RECURSIVE_MUTEX_INITIALIZER;

static void set_dlerror(int err)
{
    format_buffer(dl_err_buf, sizeof(dl_err_buf), "%s: %s", dl_errors[err],
             linker_get_error());
    dl_err_str = (const char *)&dl_err_buf[0];
};
```
set_dlerror()这个函数所完成的事情就是向一个缓冲区中格式化输出一条error message，并设置dl_err_str。可以看到这条error message将主要由两部分组成，一部分来自于dl_errors数组，set_dlerror()函数会依据错误类型来选择数组的成员，从而这个部分将是关于出错的类型的信息。另一部分则来自于linker_get_error()函数。接下来我们就来看一下linker_get_error()函数到底返回了一个什么鸟东西：
```
static char tmp_err_buf[768];
static char __linker_dl_err_buf[768];
#define BASENAME(s) (strrchr(s, '/') != NULL ? strrchr(s, '/') + 1 : s)
#define DL_ERR(fmt, x...) \
    do { \
        format_buffer(__linker_dl_err_buf, sizeof(__linker_dl_err_buf), \
                      "%s(%s:%d): " fmt, \
                      __FUNCTION__, BASENAME(__FILE__), __LINE__, ##x); \
        ERROR(fmt "\n", ##x); \
    } while(0)

const char *linker_get_error(void)
{
    return (const char *)&__linker_dl_err_buf[0];
}
```
这个函数倒也简单，返回了一个字串而已__linker_dl_err_buf。那这个__linker_dl_err_buf又是从哪儿来的呢？可以看上面的那个宏DL_ERR，它正来自于这个宏。在加载动态链接库过程出错的那个点上，宏DL_ERR会被调用，由这个宏的实现，我们可以看到，__linker_dl_err_buf将会包含出错点的函数名，文件名，行号等信息，即error message的这个部分，主要给调用者提供一些关于出错点的详细的信息。

# 查找一个动态链接库

回归主题，来看linker到底是如何查找一个动态链接库的。首先来看一下find_library()函数是怎么实现的：
```
soinfo *find_library(const char *name)
{
    soinfo *si;

#if ALLOW_SYMBOLS_FROM_MAIN
    if (name == NULL)
        return somain;
#else
    if (name == NULL)
        return NULL;
#endif

    si = find_loaded_library(name);
    if (si != NULL) {
        if(si->flags & FLAG_ERROR) {
            DL_ERR("\"%s\" failed to load previously", name);
            return NULL;
        }
        if(si->flags & FLAG_LINKED) return si;
        DL_ERR("OOPS: recursive link to \"%s\"", si->name);
        return NULL;
    }

    TRACE("[ %5d '%s' has not been loaded yet.  Locating...]\n", pid, name);
    si = load_library(name);
    if(si == NULL)
        return NULL;
    return init_library(si);
}
```
先不去关心那些错误处理的部分，即假设传入的动态连接库文件名为一个有效的文件名，而各个函数的返回值在预期范围内的情况。这个函数的执行流程大致为：

1. 在已经加载的动态连接库链表里面找一个动态连接库。如果找到了，并且没有错误情况发生，则将找到的动态连接库返回给调用者。否则，执行下面的第2个步骤。
2. 此时，即意味着用户请求的动态连接库还没有被加载。则此处会调用load_library()来加载一个动态连接库，调用init_library()函数来完成对于动态连接库句柄的初始化，并将结果返回给调用者。

此处还有一点值得我们关注的就是，***当传入的name参数为空的情况，可以看到在bionic中，是会返回somain动态连接库的句柄，根据code中，对于这个对象的注释，该对象指向main process，而并不是一个全局的符号表*** ：
```
#if ALLOW_SYMBOLS_FROM_MAIN
static soinfo *somain; /* main process, always the one after libdl_info */
#endif
```

接下来我们来看一下，linker到底是如何在一加载动态连接库中来查找一个动态连接库的。来看find_loaded_library()函数的实现：
```
static soinfo *find_loaded_library(const char *name)
{
    soinfo *si;
    const char *bname;

    // TODO: don't use basename only for determining libraries
    // http://code.google.com/p/android/issues/detail?id=6670

    bname = strrchr(name, '/');
    bname = bname ? bname + 1 : name;

    for(si = solist; si != NULL; si = si->next){
        if(!strcmp(bname, si->name)) {
            return si;
        }
    }
    return NULL;
}
```
由此我们也可以非常准确的了解到， ***在系统中，linker是用一个链表来组织各个动态连接库句柄的***。并且，可以看到，***linker就只是依据于动态连接库文件的文件名来查找一个动态链接库而已***，尽管由注释看起来写这个code的人也觉得这样不是很合理。

接下来我们来看一下，新加载一个动态连接库，即创建动态连接库句柄的更详细的执行过程，来看load_library()函数的实现：
```
static soinfo* load_library(const char* name)
{
    // Open the file.
    scoped_fd fd;
    fd.fd = open_library(name);
    if (fd.fd == -1) {
        DL_ERR("library \"%s\" not found", name);
        return NULL;
    }

    // Read the ELF header.
    Elf32_Ehdr header[1];
    int ret = TEMP_FAILURE_RETRY(read(fd.fd, (void*)header, sizeof(header)));
    if (ret < 0) {
        DL_ERR("can't read file \"%s\": %s", name, strerror(errno));
        return NULL;
    }
    if (ret != (int)sizeof(header)) {
        DL_ERR("too small to be an ELF executable: %s", name);
        return NULL;
    }
    if (verify_elf_header(header) < 0) {
        DL_ERR("not a valid ELF executable: %s", name);
        return NULL;
    }

    // Read the program header table.
    const Elf32_Phdr* phdr_table;
    phdr_ptr phdr_holder;
    ret = phdr_table_load(fd.fd, header->e_phoff, header->e_phnum,
                          &phdr_holder.phdr_mmap, &phdr_holder.phdr_size, &phdr_table);
    if (ret < 0) {
        DL_ERR("can't load program header table: %s: %s", name, strerror(errno));
        return NULL;
    }
    size_t phdr_count = header->e_phnum;

    // Get the load extents.
    Elf32_Addr ext_sz = phdr_table_get_load_size(phdr_table, phdr_count);
    TRACE("[ %5d - '%s' wants sz=0x%08x ]\n", pid, name, ext_sz);
    if (ext_sz == 0) {
        DL_ERR("no loadable segments in file: %s", name);
        return NULL;
    }

    // We no longer support pre-linked libraries.
    if (is_prelinked(fd.fd, name)) {
        return NULL;
    }

    // Reserve address space for all loadable segments.
    void* load_start = NULL;
    Elf32_Addr load_size = 0;
    Elf32_Addr load_bias = 0;
    ret = phdr_table_reserve_memory(phdr_table,
                                    phdr_count,
                                    &load_start,
                                    &load_size,
                                    &load_bias);
    if (ret < 0) {
        DL_ERR("can't reserve %d bytes in address space for \"%s\": %s",
               ext_sz, name, strerror(errno));
        return NULL;
    }

    TRACE("[ %5d allocated memory for %s @ %p (0x%08x) ]\n",
          pid, name, load_start, load_size);

    /* Map all the segments in our address space with default protections */
    ret = phdr_table_load_segments(phdr_table,
                                   phdr_count,
                                   load_bias,
                                   fd.fd);
    if (ret < 0) {
        DL_ERR("can't map loadable segments for \"%s\": %s",
               name, strerror(errno));
        return NULL;
    }

    soinfo_ptr si(name);
    if (si.ptr == NULL) {
        return NULL;
    }

    si.ptr->base = (Elf32_Addr) load_start;
    si.ptr->size = load_size;
    si.ptr->load_bias = load_bias;
    si.ptr->flags = 0;
    si.ptr->entry = 0;
    si.ptr->dynamic = (unsigned *)-1;
    si.ptr->phnum = phdr_count;
    si.ptr->phdr = phdr_table_get_loaded_phdr(phdr_table, phdr_count, load_bias);
    if (si.ptr->phdr == NULL) {
        DL_ERR("can't find loaded PHDR for \"%s\"", name);
        return NULL;
    }

    return si.release();
}
```
基本上即是依据于ELF文件格式，来load一个动态连接库。更详细信息，此处先不做过多讨论。

# dlsym()函数

然后来看一下dlsym()函数，通过一个动态连接库句柄，来查找一个symbol的过程。dlsym()函数的实现如下：
```
void *dlsym(void *handle, const char *symbol)
{
    soinfo *found;
    Elf32_Sym *sym;
    unsigned bind;

    pthread_mutex_lock(&dl_lock);

    if(unlikely(handle == 0)) {
        set_dlerror(DL_ERR_INVALID_LIBRARY_HANDLE);
        goto err;
    }
    if(unlikely(symbol == 0)) {
        set_dlerror(DL_ERR_BAD_SYMBOL_NAME);
        goto err;
    }

    if(handle == RTLD_DEFAULT) {
        sym = lookup(symbol, &found, NULL);
    } else if(handle == RTLD_NEXT) {
        void *ret_addr = __builtin_return_address(0);
        soinfo *si = find_containing_library(ret_addr);

        sym = NULL;
        if(si && si->next) {
            sym = lookup(symbol, &found, si->next);
        }
    } else {
        found = (soinfo*)handle;
        sym = soinfo_lookup(found, symbol);
    }

    if(likely(sym != 0)) {
        bind = ELF32_ST_BIND(sym->st_info);

        if(likely((bind == STB_GLOBAL) && (sym->st_shndx != 0))) {
            unsigned ret = sym->st_value + found->base;
            pthread_mutex_unlock(&dl_lock);
            return (void*)ret;
        }

        set_dlerror(DL_ERR_SYMBOL_NOT_GLOBAL);
    }
    else
        set_dlerror(DL_ERR_SYMBOL_NOT_FOUND);

err:
    pthread_mutex_unlock(&dl_lock);
    return 0;
}
```
整体而言，这个函数做了几件事情：

1. 获取到一个互斥量。这个互斥量用于保护动态连接库的链表。
2. 在指定的动态连接库上，为特定的符号查找一个Elf32_Sym对象，及包含此Elf32_Sym对象的动态连接库的句柄。
3. 通过获取到的动态连接库的句柄，即soinfo对象，及Elf32_Sym对象及算出符号的地址。
4. 释放互斥量。并将计算出来的符号的地址返回给调用者。

接下来我们看一下，为特定的符号查找一个Elf32_Sym对象和包含此Elf32_Sym对象的动态连接库句柄及计算符号地址的过程。

## 为一个特定的符号查找Elf32_Sym对象

首先我们先来看一下为特定的符号查找一个Elf32_Sym对象及包含此Elf32_Sym对象的动态连接库句柄的过程。由前面dlsym()函数的code，可以看到这个过程是依据于handle的类型而分为3种情况来执行的，即handle为RTLD_DEFAULT，RTLD_NEXT及其他。

首先是handle为RTLD_DEFAULT的情况，可以看到它就仅仅是简单调用了lookup()函数来完成这整个搜索过程而已。我们来看一下lookup()函数的定义：
```
/* This is used by dl_sym().  It performs a global symbol lookup.
 */
Elf32_Sym *lookup(const char *name, soinfo **found, soinfo *start)
{
    unsigned elf_hash = elfhash(name);
    Elf32_Sym *s = NULL;
    soinfo *si;

    if(start == NULL) {
        start = solist;
    }

    for(si = start; (s == NULL) && (si != NULL); si = si->next)
    {
        if(si->flags & FLAG_ERROR)
            continue;
        s = soinfo_elf_lookup(si, elf_hash, name);
        if (s != NULL) {
            *found = si;
            break;
        }
    }

    if(s != NULL) {
        TRACE_TYPE(LOOKUP, "%5d %s s->st_value = 0x%08x, "
                   "si->base = 0x%08x\n", pid, name, s->st_value, si->base);
        return s;
    }

    return NULL;
}
```
我们知道，在handle为RTLD_DEFAULT的情况中，调用lookup()时，其start soinfo是一个NULL值。由上面的code来看，则意味着，在这种情况下，linker将遍历系统中已经加载的所有动态链接库，即系统的soinfo对象链表，并通过调soinfo_elf_lookup()函数来在每一个动态连接库里查找那个符号(关于soinfo_elf_lookup()函数更详细的信息，会在后面讨论)。

也就是说，***当我们想要从全局符号表中查找一个符号时，我们给dlsym()函数传递的handle参数应该为RTLD_DEFAULT，而不是用NULL filename来调用dlopen()函数而得到的那个返回的handle***。

接下来，我们来看handle为RTLD_NEXT的情况。不过那个__builtin_return_address(0)到底是一个什么东西呢？它其实是一个gcc的内置函数，用于帮助获取给定函数的调用地址，此处即是要获取dlsym()函数的调用地址。__builtin_return_address()接收一个称为 level 的参数。这个参数定义希望获取 返回地址的调用堆栈级别。例如，如果指定 level 为 0，那么就是请求当前函数的返回地址。如果指定 level 为 1，那么就是请求 调用了当前函数 的函数的返回地址，依此类推。(关于的更多信息，请参考IBM developer works上的一份文档， [Linux 内核中的 GCC 特性](www.ibm.com/developerworks/cn/linux/l-gcc-hacks/) )。

那获取到的dysym()函数的返回地址又有什么用呢？接下来来看一下find_containing_library(ret_addr)函数又做些了什么事情：
```
soinfo *find_containing_library(const void *addr)
{
    soinfo *si;

    for(si = solist; si != NULL; si = si->next)
    {
        if((unsigned)addr >= si->base && (unsigned)addr - si->base < si->size) {
            return si;
        }
    }

    return NULL;
}
```
可以看到，这个函数所完成的事情即是查找到dlsym()函数的调用者所属的那个动态连接库的句柄。接下来，handle为RTLD_NEXT的情况下查找特定的符号的一个Elf32_Sym对象及包含此Elf32_Sym对象的动态连接库句柄的过程即是，同样调用lookup()函数来在链表里搜索。即 ***handle为RTLD_NEXT的情况下，会从dlsym()函数的调用者所属的动态连接库的下一个开始，在动态连接库链表中来查找特定的符号的一个Elf32_Sym对象及包含此Elf32_Sym对象的动态连接库句柄***。

然后是第三种情况，即handle来自于一个dlopen()函数调用的返回值的情况，或者说其他情况。此种情况下，则会直接调用soinfo_lookup()函数来在一个特定动态连接库句柄上查找一个特定的符号。我们来看soinfo_lookup()函数的实现：
```
static Elf32_Sym *soinfo_elf_lookup(soinfo *si, unsigned hash, const char *name)
{
    Elf32_Sym *s;
    Elf32_Sym *symtab = si->symtab;
    const char *strtab = si->strtab;
    unsigned n;

    TRACE_TYPE(LOOKUP, "%5d SEARCH %s in %s@0x%08x %08x %d\n", pid,
               name, si->name, si->base, hash, hash % si->nbucket);
    n = hash % si->nbucket;

    for(n = si->bucket[hash % si->nbucket]; n != 0; n = si->chain[n]){
        s = symtab + n;
        if(strcmp(strtab + s->st_name, name)) continue;

            /* only concern ourselves with global and weak symbol definitions */
        switch(ELF32_ST_BIND(s->st_info)){
        case STB_GLOBAL:
        case STB_WEAK:
            if(s->st_shndx == SHN_UNDEF)
                continue;

            TRACE_TYPE(LOOKUP, "%5d FOUND %s in %s (%08x) %d\n", pid,
                       name, si->name, s->st_value, s->st_size);
            return s;
        }
    }

    return NULL;
}


/* This is used by dl_sym().  It performs symbol lookup only within the
   specified soinfo object and not in any of its dependencies.
 */
Elf32_Sym *soinfo_lookup(soinfo *si, const char *name)
{
    return soinfo_elf_lookup(si, elfhash(name), name);
}
```
soinfo_lookup()函数本身则是直接调用soinfo_elf_lookup()函数来完成整个的查找的过程。而在soinfo_elf_lookup()函数中，则是依据于soinfo对象中存储Elf32_Sym对象的这种特殊的方式，来遍历查找。

## 计算符号的地址

dlsym()返回的符号的地址，是算出来的，而不是直接从什么地方读出来的。可以看到这种计算的方法，即symbol_address = (sym->st_value + found->base)。符号的地址即为动态连接库的基地址 + 该符号的Elf32_Sym对象的st_value值。

Done。
