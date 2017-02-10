---
title: 使用LeakTracer检测android NDK C/C++代码中的memory leak
date: 2017-01-12 20:35:49
tags: 
- Android
---

Memory issue是C/C++开发中比较常遇到，经常带给人比较大困扰，debug起来又常常让人无从下手的一类问题，memory issue主要又分为memory leak，野指针，及其它非法访问等问题。在android平台上，使用NDK开发C/C++ code，由于没有其它成熟的平台，如Windows，Linux等上面可用的许多工具，使得memory issue变得更为棘手。

<!--more-->

问题存在，那解决办法总是有的。比较好用又可靠的一个debug android NDK C/C++ code memory issue的办法就是，把android NDK的C/C++代码，移植到其它平台上并运行起来，然后使用那个平台下的工具，比如Linux desktop Ubuntu等，我们就可以使用诸如valgrind等异常强大的工具了，StackOverflow上有一道题[**Detect memory leak in android native code**](http://stackoverflow.com/questions/5926736/detect-memory-leak-in-android-native-code)，其中一个答案总结了一些memory leak的检测工具。特别是android和Linux desktop都使用相同的linux kernel，使得移植这项工作并不是特别复杂。

当然还有另外一种解决问题的方法，那就是把传统上其它平台的一些工具给移植到android上来使用，比如valgrind就可以用在android上，但只是不太方便而已。而这里，我们就是将LeakTracer这一linux平台上常用的memory leak检测工具给移植到android上来使用。

# LeakTracer的下载、编译

LeakTracer [official site](http://www.andreasen.org/LeakTracer/)。LeakTracer [github repo](https://github.com/fredericgermain/LeakTracer)。可以通过git clone将LeakTracer的code download下来，这个项目的结构如下：
![](https://www.wolfcstech.com/images/1315506-44c1330f9c14410c.png)

helpers目录下是一些辅助脚本，用来帮助分析产生的trace文件的；libleaktracer目录下是主要用于trace memory leak的代码，也是需要我们集成进我们项目的代码；test目录下的test 可以参考来对LeakTracer进行集成；README则说明了使用LeakTracer的方法。

可以以3种方式来使用使用LeakTracer：

 * 将自己的程序与libleaktracer.a进行链接，也就是将自己的程序一个静态链接库libleaktracer.a进行链接，我们知道静态链接是会将库的代码揉进我们自己项目的目标代码so中的。

* 将自己的程序与libleaktracer.so进行链接。需要将-lleaktracer选项做为链接命令的第一个选项。当对程序执行"objdump -p"时，应该能看到leaktracer.so是Dynamic Section的第一个NEEDED entry才对。

* 通过LD_PRELOAD环境变量来使得libleaktracer.so在任何其它动态链接库之前被加载，然后不需要对程序做任何的更改，还可以通过环境变量来对LeakTracer的行为进行定制。

这三种方法中的第一种，可以通过将libleaktracer的code复制到我们的项目中，与我们项目中的其它代码一起编译来实现。第二种和第三种都是想要使得leaktracer.so成为程序第一个被加载的library，我们知道，在android上zygote在fork一个应用进程时，也会连带将它之前加载的动态链接库一并传给我们的应用进程，这项任务看起来似乎并不是太容易实现。

这里我们就将libleaktracer的代码复制进我们的项目中，放在jni/3rd/目录下。然后修改我们的jni的Android.mk，主要改动内容为在适当的位置增加如下内容：
```
LOCAL_C_INCLUDES += $(LOCAL_PATH)/3rd/libleaktracer/include/
```

```
LOCAL_SRC_FILES += 3rd/libleaktracer/src/AllocationHandlers.cpp \
	3rd/libleaktracer/src/MemoryTrace.cpp
```
同时呢，还要修改jni的Application.mk，增加如下内容：
```
APP_OPTIM := debug
```
这样才能在编译的时候，带进更多的debug信息进目标文件。

进行到这里，进行编译，一切都ok。但想要通过Eclipse启动运行app则遇到了麻烦。Eclipse检测到libleaktracer下有个LeakTracerC.c文件libleaktracer/src/LeakTracerC.c，这个文件主要用于纯C的项目，而我们这里是一个C++的项目，因而并不会用到这个文件。但Eclipse检测到这个C文件中，却用了C++的语法，因而会标示语法错误。我们可以将这个文件直接删掉或者将它的后缀改为cpp来解决问题。

# LeakTracer的集成

要使用LeakTracer的最后一公里，也就是启动trace，并在结束trace时，将检测到的memory leak信息写入文件。参考LeakTracer/tests/test.cc的code，我们可以在我们自己的library的初始化函数中加入如下的code来启动trace：
```
// starting monitoring allocations
    leaktracer::MemoryTrace::GetInstance().startMonitoringAllThreads();
```
然后在结束时的销毁函数中加入如下的code来将memory leak信息写入文件：
```
leaktracer::MemoryTrace::GetInstance().stopAllMonitoring();

    Poco::Thread::sleep(3000);
    LOGI("To writeLeaksToFile %s.", "/sdcard/leaks.out");
    leaktracer::MemoryTrace::GetInstance().writeLeaksToFile("/sdcard/leaks.out");
```

这个地方为什么要sleep呢？这是为了等待我们library的其它资源的释放，比如一些线程握有的资源等，以减少LeakTracer的false alarm。

集成结束，开始运行来检测memory leak。我们的app刚一运行起来，就发生了一个SIGSEGV的crash：
![](https://www.wolfcstech.com/images/1315506-9652bc1a0d639d49.png)

看上去是一个空指针，这空指针产生略诡异。我们试图通过在System.loadLibrary()前加一段sleep 5s的代码，并用Eclipse的"Debug As"的"Android Native Application"运行程序，以期能获得更多空指针发生的位置的信息，但似乎并不能获得更多这一crash发生的backtrace的信息。

但不难想到，这个空指针很可能发生在libleaktracer初始化的代码里。这个library并没有太多的code，我们可以将这个library初始化相关的所有函数的开始结束处都加上log，来追查空指针到底发生在什么地方。主要包括如下的这些函数：
```
MemoryTrace::init_no_alloc_allowed()
MemoryTrace::init_full_from_once()
MemoryTrace::init_full()
int MemoryTrace::Setup(void)
void MemoryTrace::MemoryTraceOnInit(void)


void* operator new(size_t size)
void* operator new[] (size_t size)
void operator delete (void *p)
void operator delete[] (void *p)
void *malloc(size_t size)
void free(void* ptr)
void* realloc(void *ptr, size_t size)
void* calloc(size_t nmemb, size_t size)
```
加完了log，再次运行我们的程序，这次能够看到这样的一些信息：
![](https://www.wolfcstech.com/images/1315506-48aefe5b78e0d45b.png)

可以看到，这个crash发生在libleaktracer的malloc函数中，调用栈为operator new() -> MemoryTrace::Setup(void) -> MemoryTrace::init_full_from_once() -> MemoryTrace::init_full() -> malloc()。

malloc的代码如下：
```
void *malloc(size_t size)
{
	void *p;
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadUp();
	p = LT_MALLOC(size);
	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadDown();
	leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, false);
	return p;
}
```
我们就继续加log，在任意两行之间都加上log。这次运行我们的程序，可以看到这样的一些log输出：
![](https://www.wolfcstech.com/images/1315506-422bfe07447c1857.png)

由此不难看出，正是如下的这一行访问了空指针：
```
p = LT_MALLOC(size);
```

LT_MALLOC是一个宏(jni/3rd/libleaktracer/include/ObjectsPool.hpp)，是一个函数指针的别名：
```
#define LT_MALLOC  (*lt_malloc)
#define LT_FREE    (*lt_free)
#define LT_REALLOC (*lt_realloc)
#define LT_CALLOC  (*lt_calloc)
```

函数指针则定义在jni/3rd/libleaktracer/src/AllocationHandlers.cpp中：
```
void* (*lt_malloc)(size_t size);
void  (*lt_free)(void* ptr);
void* (*lt_realloc)(void *ptr, size_t size);
void* (*lt_calloc)(size_t nmemb, size_t size);
```

函数指针则定义在jni/3rd/libleaktracer/src/AllocationHandlers.cpp中：
```
void* (*lt_malloc)(size_t size);
void  (*lt_free)(void* ptr);
void* (*lt_realloc)(void *ptr, size_t size);
void* (*lt_calloc)(size_t nmemb, size_t size);
```

这些个函数指针都通过结构体数组static libc_alloc_func_t libc_alloc_funcs，在MemoryTrace::init_no_alloc_allowed()中进行初始化( jni/3rd/libleaktracer/src/MemoryTrace.cpp )：
```
typedef struct {
  const char *symbname;
  void *libcsymbol;
  void **localredirect;
} libc_alloc_func_t;

static libc_alloc_func_t libc_alloc_funcs[] = {
  { "calloc", (void*)__libc_calloc, (void**)(&lt_calloc) },
  { "malloc", (void*)__libc_malloc, (void**)(&lt_malloc) },
  { "realloc", (void*)__libc_realloc, (void**)(&lt_realloc) },
  { "free", (void*)__libc_free, (void**)(&lt_free) }
};


void
MemoryTrace::init_no_alloc_allowed()
{
	libc_alloc_func_t *curfunc;
	unsigned i;

 	for (i=0; i<(sizeof(libc_alloc_funcs)/sizeof(libc_alloc_funcs[0])); ++i) {
		curfunc = &libc_alloc_funcs[i];
		if (!*curfunc->localredirect) {
			if (curfunc->libcsymbol) {
				*curfunc->localredirect = curfunc->libcsymbol;
			} else {
				*curfunc->localredirect = dlsym(RTLD_NEXT, curfunc->symbname);
			}
		}
	}
```
MemoryTrace::init_no_alloc_allowed()中进行初始化的这段代码，主要是找到系统本来的那组分配/释放内存的函数的地址，并保存在lt_malloc这一组函数指针中。

android的标准C库中，并没有__libc_malloc这一组符号，因而lt_malloc的值应该来自于dlsym(RTLD_NEXT, curfunc->symbname)。

我们知道dlsym()函数可以与dlopen()配合，动态加载一个动态链接库，并获取里面的函数指针。dlsym()函数接收两个参数，一个是dlopen()打开的动态链接库的handle，另一个符号名。动态链接库的handle也可以不来自于dlopen()，而是RTLD_DEFAULT和RTLD_NEXT这两个特殊的值，其中前者表示从当前应用加载的第一个动态链接库开始查找符号地址，而后者则表示从当前so的下一个so开始查找符号地址。

看到这里，也就不难理解为何需要先加载libleaktracer了。总结一下，有两个原因，一是在动态链接时，程序对于分配/释放内存的函数的调用能链接到libleaktracer的实现，二是这里能够找到系统的那组分配/释放内存的函数。

但在android平台上，我们是没有办法让libleaktracer的code早于其它所有的动态链接库加载的，因而我们需要**将这里的****RTLD_NEXT****给替换成****RTLD_DEFAULT**，以便于能够找到系统的那组内存分配/释放函数的地址。

至此libleaktracer的初始化过程终于可以正常的执行了。内存的分配/释放函数调用的频率实在是太高了，因而去掉刚刚加的那些log，准备迎接下一步的挑战。

但运行起来之后，又出现了crash了。
![](https://www.wolfcstech.com/images/1315506-c2d6168a67c01d1f.png)

这次倒是可以通过Eclipse的"Debug As"的"Android Native Application"抓到发生crash的整个backtrace：
![](https://www.wolfcstech.com/images/1315506-1b668c12b693edbc.png)

MemoryTrace::storeAllocationStack()的code如下：
```
inline void MemoryTrace::storeAllocationStack(void* arr[ALLOCATION_STACK_DEPTH])
{
	unsigned int iIndex = 0;
#ifdef USE_BACKTRACE
	void* arrtmp[ALLOCATION_STACK_DEPTH+1];
	iIndex = backtrace(arrtmp, ALLOCATION_STACK_DEPTH + 1) - 1;
	memcpy(arr, &arrtmp[1], iIndex*sizeof(void*));
#else
	void *pFrame;
	// NOTE: we can't use "for" loop, __builtin_* functions
	// require the number to be known at compile time
	arr[iIndex++] = (                  (pFrame = __builtin_frame_address(0)) != NULL) ? __builtin_return_address(0) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(1)) != NULL) ? __builtin_return_address(1) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(2)) != NULL) ? __builtin_return_address(2) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(3)) != NULL) ? __builtin_return_address(3) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(4)) != NULL) ? __builtin_return_address(4) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(5)) != NULL) ? __builtin_return_address(5) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(6)) != NULL) ? __builtin_return_address(6) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(7)) != NULL) ? __builtin_return_address(7) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(8)) != NULL) ? __builtin_return_address(8) : NULL; if (iIndex == ALLOCATION_STACK_DEPTH) return;
	arr[iIndex++] = (pFrame != NULL && (pFrame = __builtin_frame_address(9)) != NULL) ? __builtin_return_address(9) : NULL;
#endif
	// fill remaining spaces
	for (; iIndex < ALLOCATION_STACK_DEPTH; iIndex++)
		arr[iIndex] = NULL;
}
```

可以看到，这个函数主要是利用gcc的内置函数__builtin_frame_address()和__builtin_return_address()获取一次内存分配发生的整个的callstack。由于对内存访问的权限限制的原因，而会发生SIGSEGV错误。但又不是在第一次调用这些内置函数时发生的crash。那就先把保存的callstack的深度调浅一点好了，比如把ALLOCATION_STACK_DEPTH的值改为3。这倒是不crash了，但每次获得的backtrace都只有一层，而且每个backtrace都一样，这些信息真是毫无用处，它们都指向libleaktracer的malloc。

看来通过gcc的内置函数__builtin_frame_address()和__builtin_return_address()获取backtrace这条路是行不同了。那android平台上native层到底有没有其它可以获取backtrace的方法呢？答案当然是，有～，而且那个接口比gcc的内置函数还有好用许多。这个好用的接口就是**_Unwind_Backtrace****()**。include标准库头文件#include <unwind.h>就可以使用_Unwind_Backtrace()了，这个函数的原型如下：
```
typedef struct _Unwind_Context _Unwind_Context;

  /* @@@ Use unwind data to perform a stack backtrace.  The trace callback
     is called for every stack frame in the call chain, but no cleanup
     actions are performed.  */
  typedef _Unwind_Reason_Code (*_Unwind_Trace_Fn) (_Unwind_Context *, void *);
  _Unwind_Reason_Code _Unwind_Backtrace(_Unwind_Trace_Fn,
					void*);
```

_Unwind_Backtrace()接收两个参数，一个是_Unwind_Reason_Code (*) (_Unwind_Context *, void *)类型的函数指针_Unwind_Trace_Fn，另外一个是userdata，会被作为每次调用 _Unwind_Trace_Fn的第二个参数传入。 _Unwind_Backtrace()的执行，会针对调用链中的每一级stack frame调用trace callback _Unwind_Trace_Fn，在这个callback的实现中，我们可以保存stack frame的信息。

我们重写MemoryTrace::storeAllocationStack()函数：
```
struct TraceHandle {
    void **backtrace;
    int pos;
};

_Unwind_Reason_Code Unwind_Trace_Fn(_Unwind_Context *context, void *hnd) {
    struct TraceHandle *traceHanle = (struct TraceHandle *) hnd;
    _Unwind_Word ip = _Unwind_GetIP(context);
    if (traceHanle->pos != ALLOCATION_STACK_DEPTH) {
        traceHanle->backtrace[traceHanle->pos] = (void *) ip;
        ++traceHanle->pos;
        return _URC_NO_REASON;
    }
    return _URC_END_OF_STACK;
}

// stores allocation stack, up to ALLOCATION_STACK_DEPTH
// frames
void MemoryTrace::storeAllocationStack(void* arr[ALLOCATION_STACK_DEPTH])
{
    unsigned int iIndex = 0;

    TraceHandle traceHandle;
    traceHandle.backtrace = arr;
    traceHandle.pos = 0;
    _Unwind_Backtrace(Unwind_Trace_Fn, &traceHandle);

    // fill remaining spaces
    for (iIndex = traceHandle.pos; iIndex < ALLOCATION_STACK_DEPTH; iIndex++)
        arr[iIndex] = NULL;
}
```
在这里我们通过**_Unwind_Backtrace****()**来获得调用栈。系统还提供了**_Unwind_GetIP****()**用来帮助我们获取每一个stack frame的指令指针IP。

在我们的library销毁时保存trace信息到文件中，从设备上将该文件/sdcard/leaks.out pull出来，可以看到如下的这样一些内容：
```
# LeakTracer report diff_utc_mono=1448719075.832670
leak, time=9077.889212, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa321407c 0xa3213fe8, size=12, data=..i.X.i...j.
leak, time=9077.889945, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32b8a94 0xa32ba1b8, size=27, data=.mi.8A......42.62.105.193.j
leak, time=9077.889151, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32b8a94 0xa32b96e4, size=84, data=G...G.......{"size":1885076,"spd":1.23937e+06,"tim
leak, time=9074.898368, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32039ec 0xa3209cec, size=88, data= .0.......i.d.r.......................i...........
leak, time=9084.530140, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32b8a94 0xa32ba1b8, size=19, data=............ipPriv.
leak, time=9088.118207, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32b8a94 0xa32b8c68, size=13, data=.............
leak, time=9084.530354, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa31a8e80 0xa31a8bac, size=28, data=....\$j..%j..%j.\"j......$j.
leak, time=9084.530232, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa31ab740 0xa31ab7f4, size=24, data=0A.......#j.H&j..&j.....
leak, time=9084.530537, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa31ab114 0xa3212be4, size=4, data=.._.
leak, time=9084.530628, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa32b8a94 0xa32ba1b8, size=18, data=............ipPub.
leak, time=9084.530720, stack=0xa3241568 0xa323fac0 0xa323fdd4 0xa31a8e80 0xa31a8bac, size=28, data=.....%j......&j..$j......$j.
```

既然有了 没被正确释放的内存分配时候的backtrace，那就将它们转换到代码文件的行数吧。LeakTracer/helpers下的leak-analyze-addr2line工具可以帮我们完成这些。leak-analyze-addr2line的用法如下：
```
Usage: /usr/bin/leak-analyze-addr2line <PROGRAM> <LEAKFILE>
```
于是我们输入如下的命令：
```
leak-analyze-addr2line /media/data/CorpProjects/git_repos/P2PClient/peerTester/obj/local/armeabi-v7a/libmoretvp2p.so leaks.out
```

注意这里的so文件的路径，不是libs/armeabi-v7a下面的那个，那个so中是没有debug信息的，而是obj/local/armeabi-v7a/下面的。

但执行上面的命令，我们却只得到了这样的一些信息：
```
Processing "leaks.out" log for "/media/data/CorpProjects/git_repos/P2PClient/peerTester/obj/local/armeabi-v7a/libmoretvp2p.so"
Matching addresses to "/media/data/CorpProjects/git_repos/P2PClient/peerTester/obj/local/armeabi-v7a/libmoretvp2p.so"
found 166 leak(s)
252 bytes lost in 9 blocks (one of them allocated at 9084.533162), from following call stack:
	??:0
	??:0
	??:0
	??:0
	??:0
4 bytes lost in 1 blocks (one of them allocated at 9084.532643), from following call stack:
	??:0
	??:0
	??:0
	??:0
	??:0
317 bytes lost in 15 blocks (one of them allocated at 9075.580071), from following call stack:
	??:0
	??:0
	??:0
	??:0
	??:0
4 bytes lost in 1 blocks (one of them allocated at 9084.532307), from following call stack:
	??:0
	??:0
	??:0
	??:0
	??:0
```

貌似完全无法得到backtrace对应的代码行数呢。问题出在哪里了呢？

回头再看/sdcard/leaks.out中的backtrace，可以看到内存地址都是进程地址空间的绝对地址，动态链接库在每次加载是都可能被映射在进程内存地址空间的不同位置，因而addr2line无法根据符号的地址空间绝对地址转换到代码行数也容易理解。难道我们要先通过/proc/[pid]/maps找到我们的动态链接库映射的内存基地址，然后手动算出backtrace每个地址对应的动态链接库内部的偏移地址，再通过addr2line来将内存地址转换到代码文件的行号？

这是比较难办到的：一来许多production的设备，根本不允许我们访问进程的内存映射/proc/[pid]/maps；二是backtrace的内存地址太多，手动转要猴年马月才能赚得完。

看起来只要我们能够获取我们的library映射到的内存的基地址，一切问题也就迎刃而解了，但android平台是否存在这样的一种方法呢？答案当然是肯定的，这个函数也就是int          dladdr(const void* addr, Dl_info *info)。dladdr()与dlsym()一样，同在libdl中。

于是我们定义一个静态的Dl_info结构对象s_P2pSODlInfo，并在MemoryTrace::init_no_alloc_allowed()中，初始化lt_calloc等函数指针的那段code下面加一行对dladdr()的调用：
```
dladdr((const void*)init_no_alloc_allowed, &s_P2pSODlInfo);
 	LOGI("s_P2pSODlInfo, dli_fbase = %p, dli_fname = %s, dli_saddr = %p, dli_sname = %s",
 	     s_P2pSODlInfo.dli_fbase, s_P2pSODlInfo.dli_fname, s_P2pSODlInfo.dli_saddr, s_P2pSODlInfo.dli_sname);
```

这里我们就通过函数init_no_alloc_allowed的地址来获得整个library内存映射的基地址。同时修改_Unwind_Backtrace的Unwind_Trace_Fn为如下这样：
```
_Unwind_Reason_Code Unwind_Trace_Fn(_Unwind_Context *context, void *hnd) {
    struct TraceHandle *traceHanle = (struct TraceHandle *) hnd;
    _Unwind_Word ip = _Unwind_GetIP(context);
    if (traceHanle->pos != ALLOCATION_STACK_DEPTH) {
        traceHanle->backtrace[traceHanle->pos] = (void *) (ip - (_Unwind_Word) s_P2pSODlInfo.dli_fbase);
        ++traceHanle->pos;
        return _URC_NO_REASON;
    }
    return _URC_END_OF_STACK;
}
```

将我们通过_Unwind_GetIP()获得的IP值都减去library 内存映射的基地址。

再次运行我们的程序，产生memory leak的trace文件并pull下来，并用leak-analyze-addr2line工具进行分析，终于可以获得leak的内存分配时的callstack了，如下面这样：
```
Processing "leaks.out" log for "/media/data/CorpProjects/git_repos/P2PClient/peerTester/obj/local/armeabi-v7a/libmoretvp2p.so"
Matching addresses to "/media/data/CorpProjects/git_repos/P2PClient/peerTester/obj/local/armeabi-v7a/libmoretvp2p.so"
found 165 leak(s)
24 bytes lost in 1 blocks (one of them allocated at 10597.290280), from following call stack:
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/MemoryTrace.cpp:316
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/include/MemoryTrace.hpp:368
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/AllocationHandlers.cpp:29
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/json/src/Value.cpp:361
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/json/src/Value.cpp:368 (discriminator 1)
69 bytes lost in 3 blocks (one of them allocated at 10585.557730), from following call stack:
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/MemoryTrace.cpp:316
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/include/MemoryTrace.hpp:368
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/AllocationHandlers.cpp:29
	libgcc2.c:?
	libgcc2.c:?
12 bytes lost in 1 blocks (one of them allocated at 10587.229148), from following call stack:
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/MemoryTrace.cpp:316
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/include/MemoryTrace.hpp:368
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/3rd/libleaktracer/src/AllocationHandlers.cpp:29
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/src/core/PlaySession.cpp:38 (discriminator 7)
	/media/data/CorpProjects/git_repos/P2PClient/peerTester/jni/src/core/PlaySessionInitializer.cpp:241 (discriminator 3)
```

可能由于内存释放的时延等原因，LeakTracer报出来的问题不一定是真正的memory leak，也可能只是false alarm，具体问题还需要根据LeakTracer产生的报告再来做分析。

# LeakTracer的设计与实现

这里我们再来分析一下LeakTracer的设计与实现。

LeakTracer主要的设计思路为：

1. 实现一组内存的分配/释放函数，这组函数的函数原型与系统的那一组完全一样，让被trace的library对于内存的分配/释放函数的调用都链接到自己实现的这一组函数中以override掉系统的那组内存/分配释放函数；

2. 自己实现的这组函数中的内存分配函数记录分配相关的信息，包括分配的内存的大小，callstack等，并调用系统本来的内存分配函数去分配内存；

3. 自己实现的这组函数中的内存释放函数则销毁内存分配的相关记录，并使用系统的内存释放函数真正的释放内存；

4. 在trace结束时，遍历所有保存的内存分配记录的信息，并把这些信息保存进文件以供进一步的分析。

LeakTracer实现的用于override系统内存分配/释放函数的那组函数在jni/3rd/libleaktracer/src/AllocationHandlers.cpp中定义：
```
void* (*lt_malloc)(size_t size);
void  (*lt_free)(void* ptr);
void* (*lt_realloc)(void *ptr, size_t size);
void* (*lt_calloc)(size_t nmemb, size_t size);

void* operator new(size_t size) {
	void *p;
	leaktracer::MemoryTrace::Setup();

	p = LT_MALLOC(size);
	leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, false);

	return p;
}


void* operator new[] (size_t size) {
	void *p;
	leaktracer::MemoryTrace::Setup();

	p = LT_MALLOC(size);
	leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, true);

	return p;
}


void operator delete (void *p) {
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().registerRelease(p, false);
	LT_FREE(p);
}


void operator delete[] (void *p) {
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().registerRelease(p, true);
	LT_FREE(p);
}

/** -- libc memory operators -- **/

/* malloc
 * in some malloc implementation, there is a recursive call to malloc
 * (for instance, in uClibc 0.9.29 malloc-standard )
 * we use a InternalMonitoringDisablerThreadUp that use a tls variable to prevent several registration
 * during the same malloc
 */
void *malloc(size_t size)
{
	void *p;
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadUp();
	p = LT_MALLOC(size);
	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadDown();
	leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, false);

	return p;
}

void free(void* ptr)
{
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().registerRelease(ptr, false);
	LT_FREE(ptr);
}

void* realloc(void *ptr, size_t size)
{
	void *p;
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadUp();

	p = LT_REALLOC(ptr, size);

	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadDown();

	if (p != ptr)
	{
		if (ptr)
			leaktracer::MemoryTrace::GetInstance().registerRelease(ptr, false);
		leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, false);
	}
	else
	{
		leaktracer::MemoryTrace::GetInstance().registerReallocation(p, size, false);
	}

	return p;
}

void* calloc(size_t nmemb, size_t size)
{
	void *p;
	leaktracer::MemoryTrace::Setup();

	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadUp();
	p = LT_CALLOC(nmemb, size);
	leaktracer::MemoryTrace::GetInstance().InternalMonitoringDisablerThreadDown();
	leaktracer::MemoryTrace::GetInstance().registerAllocation(p, nmemb*size, false);

	return p;
}
```
系统的那组内存分配/释放函数的函数指针也在这个文件中定义。如我们前面看到的，系统的那组内存分配/释放函数的函数指针在MemoryTrace::init_no_alloc_allowed()中通过dlsym(RTLD_DEFAULT, curfunc->symbname)进行初始化。

LeakTracer通过leaktracer::MemoryTrace::GetInstance().registerAllocation(p, size, false)记录每一次内存分配的相关信息：
```
// adds all relevant info regarding current allocation to map
inline void MemoryTrace::registerAllocation(void *p, size_t size, bool is_array)
{
	allocation_info_t *info = NULL;
	if (!AllMonitoringIsDisabled() && (__monitoringAllThreads || getThreadOptions().monitoringAllocations) && p != NULL) {
		MutexLock lock(__allocations_mutex);
		info = __allocations.insert(p);
		if (info != NULL) {
			info->size = size;
			info->isArray = is_array;
			storeTimestamp(info->timestamp);
		}
	}
 	// we store the stack without locking __allocations_mutex
	// it should be safe enough
	// prevent a deadlock between backtrave function who are now using advanced dl_iterate_phdr function
 	// and dl_* function which uses malloc functions
	if (info != NULL) {
		storeAllocationStack(info->allocStack);
	}

	if (p == NULL) {
		InternalMonitoringDisablerThreadUp();
		// WARNING
		InternalMonitoringDisablerThreadDown();
	}
}
```

并通过leaktracer::MemoryTrace::GetInstance().registerRelease(ptr, false)销毁一个内存分配记录：
```
// removes allocation's info from the map
inline void MemoryTrace::registerRelease(void *p, bool is_array)
{
	if (!AllMonitoringIsDisabled() && __monitoringReleases && p != NULL) {
		MutexLock lock(__allocations_mutex);
		allocation_info_t *info = __allocations.find(p);
		if (info != NULL) {
			if (info->isArray != is_array) {
				InternalMonitoringDisablerThreadUp();
				// WARNING
				InternalMonitoringDisablerThreadDown();
			}
			__allocations.release(p);
		}
	}
}
```
整体来看LeakTracer的设计与实现都并不复杂，因而能够trace的memory issue也就有限。比如，LeakTracer就无法trace多次释放等问题。但它也足以作为我们编写更强大的memory issue trace工具的基础了。

Done。

参考文档：
[Android下打印调试堆栈方法](http://blog.csdn.net/freshui/article/details/9456889)
[dladdr - 获取某个地址的符号信息](http://blog.csdn.net/dragon101788/article/details/18673323)
[LeakTracer for Android](https://github.com/hanpfei/LeakTracer)
http://androidxref.com/5.0.0_r2/xref/hardware/ti/omap4-aah/stacktrace.c
http://www.newsmth.net/nForum/#!article/KernelTech/413
