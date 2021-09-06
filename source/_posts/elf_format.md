---
title: ELF 文件格式
date: 2021-06-13 21:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

`<elf.h>` 头文件定义了 ELF 可执行二进制文件的格式。这些文件包括普通的可执行文件，即可以直接执行的应用程序文件；可重定位目标文件，即 *.o 文件；核心转储 core 文件；和共享目标文件，即共享库 *.so 文件。

使用 ELF 文件格式的可执行文件的组成是这样的：一个 ELF 文件头，后面是一个程序头表，或者是一个节（即 section，后文也用节指代 section，用段指代 segment）头表，或两者都有。ELF 文件头总是位于文件中偏移量为 0 的位置。程序头表和节头表在文件中的偏移量则由 ELF 文件头定义。这两个表描述了整个 ELF 文件其余部分的细节。
<!--more-->
这个头文件以 C 结构体的形式描述了上面提到的那些头，它也包含动态节，重定位节，和符号表的结构体。

### 基本类型

N-bit 架构（N=32，64，ElfN 代表 Elf32 或 Elf64，uintN_t 代表 uint32_t 或 uint64_t）使用了下面这些数据类型：

| 数据类型 | 说明 |
|---|---|
| ElfN_Addr | 无符号程序地址，uintN_t |
| ElfN_Off | 无符号文件偏移量，uintN_t |
| ElfN_Section | 无符号节索引，uint16_t |
| ElfN_Versym | 无符号版本符号信息，uint16_t |
| Elf_Byte | unsigned char |
| ElfN_Half | uint16_t |
| ElfN_Sword | int32_t |
| ElfN_Word | uint32_t |
| ElfN_Sxword | int64_t |
| ElfN_Xword | uint64_t |

（注意：BSD 的术语有点不一样。Elf64_Half 是 Elf32_Half 大小的两倍大，Elf64Quarter 用作 uint16_t。为了避免混淆，在下文中这些类型由显式的替换。）

文件格式定义的所有数据结构遵循相关类型的 “自然” 大小及对齐规则。如果有需要，4 字节对象的数据结构可以包含显式的填充以确保 4 字节对齐，来强制结构体大小为 4 的整数倍，等等。

### ELF 文件头（Ehdr）
ELF 文件头由类型 Elf32_Ehdr 或 Elf64_Ehdr 描述：
```
           #define EI_NIDENT 16

           typedef struct {
               unsigned char e_ident[EI_NIDENT];
               uint16_t      e_type;
               uint16_t      e_machine;
               uint32_t      e_version;
               ElfN_Addr     e_entry;
               ElfN_Off      e_phoff;
               ElfN_Off      e_shoff;
               uint32_t      e_flags;
               uint16_t      e_ehsize;
               uint16_t      e_phentsize;
               uint16_t      e_phnum;
               uint16_t      e_shentsize;
               uint16_t      e_shnum;
               uint16_t      e_shstrndx;
           } ElfN_Ehdr;
```

这些字段的含义如下：

**e_ident**  这个字节数组描述了如何来解释这个文件，其依赖的处理器或文件其余部分的内容。这个数组中的每一样东西都由以 `EI_` 为前缀的宏命名，且可能包含以 `ELF` 为前缀的宏值。这些宏有如下这些：
  - EI_MAG0  Magic number 的第一个字节。它的值必须是 ELFMAG0。(0: 0x7f)
  - EI_MAG1  Magic number 的第二个字节。它的值必须是 ELFMAG1。(1: 'E')
  - EI_MAG2  Magic number 的第三个字节。它的值必须是 ELFMAG2。(2: 'L')
  - EI_MAG3  Magic number 的第四个字节。它的值必须是 ELFMAG3。(3: 'F')
  - EI_CLASS  第五个字节描述了这个文件的架构：
    * ELFCLASSNONE 无效类别。
    * ELFCLASS32 这个值定义了 32 位架构。它支持文件和虚拟地址空间最多 4 Gigabytes 的机器。
    * ELFCLASS64 这个值定义了 64 位架构。
  - EI_DATA  第六个字节描述了文件中处理器特有数据的编码方式。目前支持的编码方式有如下这些：
    * ELFDATANONE 未知数据格式。
    * ELFDATA2LSB 二进制补码，小尾端。
    * ELFDATA2MSB 二进制补码，大尾端。
  - EI_VERSION  第七个字节是 ELF 规范的版本号：  
    * EV_NONE 无效版本。
    * EV_CURRENT 目前的版本。
  - EI_OSABI  第八个字节描述了这个目标文件的目标操作系统和 ABI。其它的 ELF 结构中的一些字段有一些对于特定的平台有意义的标记和值；对于那些字段的解释由这个字节的值决定。比如：

| ABI 值 | 含义 |
|--|--|
| ELFOSABI_NONE | 与 ELFOSABI_SYSV 相同 |
| ELFOSABI_SYSV | UNIX System V ABI |
| ELFOSABI_HPUX | HP-UX ABI |
| ELFOSABI_NETBSD | NetBSD ABI |
| ELFOSABI_LINUX | Linux ABI |
| ELFOSABI_SOLARIS | Solaris ABI |
| ELFOSABI_IRIX | IRIX ABI |
| ELFOSABI_FREEBSD | FreeBSD ABI |
| ELFOSABI_TRU64 | TRU64 UNIX ABI |
| ELFOSABI_ARM | ARM 架构 ABI |
| ELFOSABI_STANDALONE | Stand-alone (embedded) ABI |

  - EI_ABIVERSION  第九个字节描述了目标文件的目标 ABI 版本。该字段用于区分 ABI 的不兼容版本。这个版本号的解释依赖于由 EI_OSABI 字段描述的 ABI。符合本规范的应用程序使用值 0。
  - EI_PAD  填充值的开始。这些字节保留且被设置为 0。读取它们的程序应该忽略它们。如果目前未使用的字节被赋予了意义，则 EI_PAD 的值在未来将会改变。
  - EI_NIDENT  e_ident 数组的大小。

**e_type**  该结构体的这个成员描述了目标文件的类型：

| 类型值 | 含义 |
|--|--|
| ET_NONE | 一种未知类型 |
| ET_REL | 可重定位文件，即 *.o 文件 |
| ET_EXEC | 可执行文件 |
| ET_DYN | 共享目标文件，即 *.so 文件 |
| ET_CORE | core 文件，即 crash 时的核心转储文件 |

**e_machine**  该成员为单个文件指定所需的体系结构。比如：

| 机器值 | 含义 |
|--|--|
| EM_NONE | 未知机器类型 |
| EM_M32 | AT&T WE 32100 |
| EM_SPARC | Sun Microsystems SPARC |
| EM_386 | Intel 80386 |
| EM_68K | Motorola 68000 |
| EM_88K | Motorola 88000 |
| EM_860 | Intel 80860 |
| EM_MIPS | MIPS RS3000 (big-endian only) |
| EM_PARISC | HP/PA |
| EM_SPARC32PLUS | SPARC with enhanced instruction set |
| EM_PPC | PowerPC |
| EM_PPC64 | PowerPC 64-bit |
| EM_S390 | IBM S/390 |
| EM_ARM | Advanced RISC Machines |
| EM_SH | Renesas SuperH |
| EM_SPARCV9 | SPARC v9 64-bit |
| EM_IA_64 | Intel Itanium |
| EM_X86_64 | AMD x86-64 |
| EM_VAX | DEC Vax |

**e_entry**  这个成员给出了系统首次控制转移的目标虚拟地址，这将启动进程。如果文件没有关联的入口点，则这个成员的值为 0。
**e_phoff**  这个成员是程序头表（program header table）在文件中的字节偏移量。如果文件没有程序头表，则这个成员的值为 0。
**e_shoff**  这个成员是节头表（section header table）在文件中的字节偏移量。如果文件没有节头表，则这个成员的值为 0。
**e_flags**  这个成员为与文件关联的处理器特有标记。标记名的形式为 EF_`machine_flag`。目前，还没有定义任何标记。
**e_ehsize**  这个成员是 ELF 头的字节大小。
**e_phentsize**  这个成员是文件的程序头表中一个项的字节大小；所有的项具有相同大小。
**e_phnum**  这个成员是程序头表中的程序头个数。这样 e_phentsize 和 e_phnum 的乘积给出了这个表的字节大小。如果文件没有程序头，则 e_phnum 的值为 0。
如果程序头表中项的个数大于等于 PN_XNUM (0xffff)，则这个成员的值为 PN_XNUM (0xffff)，而真实的程序头表中的项数由节头表中的*初始项*的 sh_info 成员给出。否则*初始项*的 sh_info 成员的值为 0。
   - PN_XNUM  这个值被定义为 0xffff，它是 e_phnum 能具有的最大值，给出了实际的程序头个数的位置。
**e_shentsize**  这个成员为节头的字节大小。节头是节头表中的一项；所有项的大小都相同。
**e_shnum**  这个成员为接头表中的项数。这样 e_shentsize 和 e_shnum 的乘积给出了节头表的字节大小。如果一个文件没有节头表，则 e_shnum 的值为 0。
如果节头表中的项数大于等于 SHN_LORESERVE (0xff00)，则 e_shnum 的值为 0，且节头表中的项数的真实值位于节头表的初始项的 sh_size 成员中。否则节头表的初始项的 sh_size 成员的值为 0。
**e_shstrndx**  这个字段为与节名称字符串表关联的节的节头在节头表中的索引。如果文件没有节名称字符串表，则这个成员的值为 SHN_UNDEF 。
如果节名称字符串表的节的索引大于等于 SHN_LORESERVE (0xff00)，则这个字段的值为 SHN_XINDEX (0xffff)，而节名称字符串表节的真正索引位于节头表初始项的 sh_link 成员中。否则，节头表中的初始项的 sh_link 成员值为 0。

### 程序头（Phdr）

可执行文件或共享目标文件的程序头表是一个结构体的数组，其中的每一个都描述了一个段或系统用于为执行做准备的其它信息。一个目标文件的段包含一个或多个节。程序头只对可执行文件和共享目标文件有意义。文件用 ELF 文件头的 e_phentsize 和 e_phnum 成员描述它自己的程序头大小。根据具体的架构，ELF 程序头用类型 Elf32_Phdr 或 Elf64_Phdr 描述：

```
           typedef struct {
               uint32_t   p_type;
               Elf32_Off  p_offset;
               Elf32_Addr p_vaddr;
               Elf32_Addr p_paddr;
               uint32_t   p_filesz;
               uint32_t   p_memsz;
               uint32_t   p_flags;
               uint32_t   p_align;
           } Elf32_Phdr;

           typedef struct {
               uint32_t   p_type;
               uint32_t   p_flags;
               Elf64_Off  p_offset;
               Elf64_Addr p_vaddr;
               Elf64_Addr p_paddr;
               uint64_t   p_filesz;
               uint64_t   p_memsz;
               uint64_t   p_align;
           } Elf64_Phdr;
```

32 位和 64 位程序头的主要区别在于 p_flags 成员在结构体中的位置。

**p_type**  结构体的这个成员表示这个数组元素描述的是何种类型的段，或如何解释数组元素的信息。

  - PT_NULL  该数组元素是未使用的，且其它成员的值是未定义的。这使得该程序头被忽略。
  - PT_LOAD  该数组元素表示一个可加载段，由 p_filesz 和 p_memsz 描述。文件中这部分的字节被映射到内存段的开始位置。如果段的内存大小 p_memsz 大于文件大小 p_filesz，“额外的”自己被定义为值 0，且跟在段的已初始化部分后面。段的文件大小可以不大于内存大小。程序头表中的可加载段项以升序出现，按 p_vaddr 成员的值排序。
  - PT_DYNAMIC  该数组元素表示动态链接信息。
  - PT_INTERP  该数组元素表示一个以 null 终止的将被调用以作为解释器的路径名的位置和大小。这个段类型只对可执行文件（尽管它可以出现在共享目标文件中）有意义。然而，它在一个文件中不会出现多次。如果出现，它必须出现在任何可加载段项的前面。
  - PT_NOTE  该数组元素表示 notes 的位置（ElfN_Nhdr）。
  - PT_SHLIB  这个段类型保留，但语义不明。包含此类型数组元素的程序不符合ABI。
  - PT_PHDR  该数组元素，如果出现，表示程序头表自身的位置和大小，在程序的文件和内存镜像中都是。这个段类型在一个文件中不会出现多次。此外，如果程序头表是程序的内存镜像的一部分时，它可能出现。如果出现，它必须出现在任何可加载段项的前面。
  - PT_LOPROC, PT_HIPROC  [PT_LOPROC, PT_HIPROC] 范围内的值被保留用于处理器特有的语义。
  - PT_GNU_STACK  GNU 扩展，Linux 内核会使用它来通过 p_flags 成员设置的标记控制栈的状态。

**p_offset**  这个成员表示这个段的第一个字节从文件开始位置处的偏移量。
**p_vaddr**  这个成员表示这个段的第一个字节在内存中的虚拟地址。
**p_paddr**  在物理地址是相对寻址的系统上，这个成员保留用作段的物理地址。在 BSD 下这个成员未使用，且必须是 0。
**p_filesz**  这个成员表示段在文件镜像中的字节大小。它可能是 0。
**p_memsz**  这个成员表示段在内存镜像中的字节大小。它可能是 0。
**p_flags**  这个成员表示与段相关的标记的位掩码：
  - PF_X  可执行段。
  - PF_W  可读段。
  - PF_R  可写段。

文本段通常具有标记 PF_X 和 PF_R。数据段通常具有标记 PF_X，PF_W 和 PF_R。

**p_align**  这个字段的值是段在内存和文件中的对齐方式。可加载的进程段对 p_vaddr 和 p_offset 必须具有一致的值，为页大小的模。值为 0 和 1 表示不需要对齐。否则，p_align 应该是个正值，2 的指数，且 p_vaddr 应该等于 p_offset，模 p_align。

### 节头（Shdr）

文件的节头表让我们可以定位文件所有的节。节头表是 Elf32_Shdr 或 Elf64_Shdr 结构体的数组。ELF 文件头的 e_shoff 成员给出了节头表到文件开始位置处的字节偏移量。
e_shnum 成员为节头表包含的项数。
e_shentsize 的值为每个项的字节大小。

节头表索引是这个数组的下标。一些接头表索引被保留：初始项和 SHN_LORESERVE 及 SHN_HIRESERVE 之间的索引。初始项用于 e_phnum，e_shnum 和 e_strndx 的 ELF扩展，在其它情况下，初始项中的每个字段被设置为 0。目标文件不具有如下这些特殊索引的节：

SHN_UNDEF  这个值标记一个未定义的，丢失的，不相关的，或其它无意义的节参考。
SHN_LORESERVE  这个值表示保留的索引的下界。
SHN_LOPROC, SHN_HIPROC  大于 [SHN_LOPROC, SHN_HIPROC] 范围的值被保留用于处理器特有语义。
SHN_ABS  这个值指定了对应引用的绝对值。比如，一个符号相对于节号 SHN_ABS 定义则具有绝对值，且不受重定位影响。
SHN_COMMON  相对于这个节定义的符号是通用符号，比如 FORTRAN COMMON 或未分配的 C 外部变量。
SHN_HIRESERVE  这个值表示保留的索引范围的上界。系统保留 SHN_LORESERVE 和 SHN_HIRESERVE 之间的范围，包括。节头表不包含这些保留索引的项。

节头的结构如下：

```
           typedef struct {
               uint32_t   sh_name;
               uint32_t   sh_type;
               uint32_t   sh_flags;
               Elf32_Addr sh_addr;
               Elf32_Off  sh_offset;
               uint32_t   sh_size;
               uint32_t   sh_link;
               uint32_t   sh_info;
               uint32_t   sh_addralign;
               uint32_t   sh_entsize;
           } Elf32_Shdr;

           typedef struct {
               uint32_t   sh_name;
               uint32_t   sh_type;
               uint64_t   sh_flags;
               Elf64_Addr sh_addr;
               Elf64_Off  sh_offset;
               uint64_t   sh_size;
               uint32_t   sh_link;
               uint32_t   sh_info;
               uint64_t   sh_addralign;
               uint64_t   sh_entsize;
           } Elf64_Shdr;
```

32 位和 64 位节头没有真正的差异。

**sh_name**  这个成员表示节的名称。它的值是到节头字符串表节的索引，给出了一个以 null 结尾的字符串的位置。
**sh_type**  这个成员给节的内容和语义做了分类。
  - SHT_NULL  这个值把节头标记为 inactive。它没有与之关联的节。该节头的其它成员具有未定义值。
  - SHT_PROGBITS  该节持有由程序定义的信息，其格式和含义完全由程序决定。
  - SHT_SYMTAB  这个节是符号表。典型地，SHT_SYMTAB 为链接编辑提供了符号，尽管它可能也用于动态链接。
作为一个完整的符号表，它可能包含许多不需要动态链接的符号。目标文件也可以包含一个 SHT_DYNSYM 节。
  - SHT_STRTAB  这个节是一个字符串表。一个目标文件可以有多个字符串表节。
  - SHT_RELA  这个节包含具有显式的附加的重定位项，例如，目标文件的 32 位类的类型Elf32_Rela。一个目标文件可以有多个重定位节。
  - SHT_HASH  这个节是符号哈希表。参与动态链接的目标文件必须包含一个符号哈希表。一个目标文件可能只有一个哈希表。
  - SHT_DYNAMIC  这个节包含用于动态链接的信息。一个目标文件可以只有一个动态节。
  - SHT_NOTE  这个节包含 notes（ElfN_Nhdr）。
  - SHT_NOBITS  这种类型的节在文件中不占用空间，但它与 SHT_PROGBITS 类似。尽管这种节不包含数据，但其 sh_offset 成员包含概念上的文件偏移量。
  - SHT_REL  这个节包含不具有显式的附加的重定位偏移，例如，目标文件的 32 位类的类型 Elf32_Rel。一个目标文件可以有多个重定位节。
  - SHT_SHLIB  该节是保留的，但具有未指定的语义。
  - SHT_DYNSYM  该节包含动态链接符号的最小集合。一个目标文件也可以包含一个 SHT_DYNSYM 节。
  - SHT_LOPROC, SHT_HIPROC  [SHT_LOPROC, SHT_HIPROC] 范围内的值是为处理器特有语义保留的。
  - SHT_LOUSER  这个值是为应用程序保留的索引范围的下界。
  - SHT_HIUSER  这个值是为应用程序保留的索引范围的上界。SHT_LOUSER 和 SHT_HIUSER 之间的节类型可以用于应用程序，而不会与当前或未来系统定义的节类型冲突。
**sh_flags**  节支持描述杂项属性的一位标志。如果给 sh_flags 设置了一个标记位，则这个节的属性为 "on"。否则，属性为 "off" 或不应用。未定义的属性设置为 0。

  - SHF_WRITE  该节包含的数据在进程执行期间应该可写。
  - SHF_ALLOC  该节在进程执行期间占用内存。有些控制节不驻留在目标文件的内存映像中。对于那些节来说这个属性为 off。
  - SHF_EXECINSTR  该节包含可执行的机器指令。
  - SHF_MASKPROC  这个掩码中包含的所有位为处理器特有语义保留。

*sh_addr*  如果该节出现在进程的内存镜像中，则这个成员为该节的第一个字节应该所处的地址。否则，这个成员包含 0。
*sh_offset*  这个成员的值为这个节中的第一个字节到文件开始位置处的字节偏移量。一种节类型，SHT_NOBITS，在文件中不占用空间，但它的 sh_offset 成员定位了在文件中概念上的位置。
*sh_size*  这个成员的值为该节的字节大小。除非节的类型为 SHT_NOBITS，则该节在文件中占有 sh_size 字节。SHT_NOBITS 类型的节可以具有非 0 的大小，但它在文件中不占用空间。
*sh_link*  该成员为节头表索引链接，其解释依赖于节类型。
*sh_info*  这个成员包含额外的信息，其解释依赖于节类型。
*sh_addralign*  一些节具有地址对齐限制。如果一个节包含双字，则系统必须为整个节确保双字对齐。即，sh_addr 的值必须全等于 0，模 sh_addralign 的值。只能是 0 和 2 的正整数幂。值为 0 或 1 意味着该节没有对齐限制。
*sh_entsize*  一些节包含固定大小的项的表，比如符号表。对于这样的节，该成员给出了每一项的字节大小。如果该节不包含固定大小的项的表则该成员包含 0。

各种各样的节包含有程序和控制信息：

*.bss* 本节包含未初始化但会占用程序的内存镜像空间的数据。根据定义，系统在程序开始运行时以0 初始化数据。
本节的类型为 **SHT_NOBITS**。属性类型为 **SHF_ALLOC** 和 **SHF_WRITE**。

*.comment* 这个节包含版本控制信息。这个节的类型为 **SHT_PROGBITS**。没有属性类型。

*.ctors* 本节包含指向 C++ 构造函数的已初始化指针。本节的类型为 **SHT_PROGBITS**。属性类型为 **SHF_ALLOC** 和 **SHF_WRITE**。

*.data* 本节包含用于程序内存镜像的已初始化数据。本节的类型为 **SHT_PROGBITS**。属性类型为 **SHF_ALLOC** 和 **SHF_WRITE**。

*.data1* 本节包含用于程序内存镜像的已初始化数据。本节的类型为 **SHT_PROGBITS**。属性类型为 **SHF_ALLOC** 和 **SHF_WRITE**。

*.debug* 本节包含符号调试的信息。内容未指定。本节的类型为 **SHT_PROGBITS**。不使用任何属性类型。

*.dtors* 本节包含指向 C++ 析构函数的已初始化指针。本节的类型为 **SHT_PROGBITS**。属性类型为 **SHF_ALLOC** 和 **SHF_WRITE**。

*.dynamic* 本节包含动态链接信息。本节的属性将包含 **SHF_ALLOC** 位。**SHF_WRITE** 位是否被置位依赖于处理器。本节的类型为 **SHT_DYNAMIC**。参考上面的属性。

*.dynstr* 本节包含动态链接所需的字符串，最常见的是与符号表项关联的表示名字的字符串。本节的类型为 **SHT_STRTAB**。用到的属性为 **SHF_ALLOC**。

*.dynsym* 本节包含动态链接符号表。本节的类型为 **SHT_DYNSYM**。用到的属性为 **SHF_ALLOC**。

*.fini* 本节包含用于进程终止代码的可执行指令。当程序正常退出时，系统安排执行本节的代码。本节的类型为 **SHT_PROGBITS**。用到的属性为 **SHF_ALLOC** 和 **SHF_EXECINSTR**。

*.gnu.version* 本节包含版本符号表，一个 *ElfN_Half* 元素的数组。本节的类型为 **SHT_GNU_versym**。用到的属性类型为 **SHF_ALLOC**。

*.gnu.version_d* 本节包含版本符号定义，一个 *ElfN_Verdef* 结构的表。本节的类型为 **SHT_GNU_verdef**。用到的属性类型为 **SHF_ALLOC**。

*.gnu.version_r* 本节包含元素需要的版本符号，一个 *ElfN_Verneed* 结构的表。本节的类型为 **SHT_GNU_versym**。用到的属性类型为 **SHF_ALLOC**。

*.got* 本节包含全局偏移表。本节的类型为 **SHT_PROGBITS**。属性是特定于处理器的。

*.hash* 本节包含符号哈希表。本节的类型为 **SHT_HASH**。用到的属性为 **SHF_ALLOC**。

*.init* 本节包含用于进程初始化代码的可执行指令。当程序开始运行时，系统在调用主程序的入口点之前安排执行本节的代码。本节的类型为 **SHT_PROGBITS**。用到的属性为 **SHF_ALLOC** 和 **SHF_EXECINSTR**。

*.interp* 本节包含程序解释器的路径名。如果程序具有包含该节的段，则该节的属性将包含 **SHF_ALLOC** 位。否则，**SHF_ALLOC** 位将为 off。本节的类型为 **SHT_PROGBITS**。

*.line* 本节包含符号调试的行号信息，其描述了程序源码和机器码之间的对应关系。内容未指定。本节的类型为 **SHT_PROGBITS**。没有用到任何属性类型。

*.note* 本节包含各种 notes。本节的类型为 **SHT_NOTE**。没有用到任何属性类型。

*.note.ABI-tag* 本节用于声明期望的 ELF 镜像的运行时 ABI。它包含操作系统名和它的运行时版本。本节的类型为 **SHT_NOTE**。只用到了 **SHF_ALLOC** 属性。

*.note.gnu.build-id* 本节用于包含唯一标识 ELF 镜像内容的 ID。具有相同 build ID 的不同文件应该包含相同的可执行内容。参考 GNU 链接器 (**ld** (1)) 的 **--build-id** 选项来了解更多细节。本节的类型为 **SHT_NOTE**。只用到了 **SHF_ALLOC** 属性。

*.note.GNU-stack* 本节用于声明栈属性的 Linux 目标文件中。本节的类型为 **SHT_PROGBITS**。唯一使用的属性是 **SHF_EXECINSTR**。本节向 GNU 链接器表示目标文件需要一个可执行栈。

*.note.openbsd.ident* OpenBSD 本地可执行文件通常包含这个节来标识它们自己，以使内核在加载文件时可以绕开任何兼容性 ELF 二进制仿真测试。

*.plt* 本节包含过程链接表。本节的类型为 **SHT_PROGBITS**。属性是特定于处理器的。

*.relNAME* 本节包含如下面所述的重定位信息。如果文件具有包含重定位的可加载段，本节的属性将包含 **SHF_ALLOC** 位。否则，该位为 off。按照惯例，"NAME" 由重定位应用的节提供。这样 *.text* 的重定位节通常的名字将为 *.rel.text*。本节的类型为 **SHT_REL**。

*.relaNAME* 本节包含如下面所述的重定位信息。如果文件具有包含重定位的可加载段，本节的属性将包含 **SHF_ALLOC** 位。否则，该位为 off。按照惯例，"NAME" 由重定位应用的节提供。这样 *.text* 的重定位节通常的名字将为 *.rela.text*。本节的类型为 **SHT_RELA**。

*.rodata* 本节包含只读数据，典型地用于进程镜像的非可写段。本节的类型为 **SHT_PROGBITS**。用到的属性为 **SHF_ALLOC**。

*.rodata1* 本节包含只读数据，典型地用于进程镜像的非可写段。本节的类型为 **SHT_PROGBITS**。用到的属性为 **SHF_ALLOC**。

*.shstrtab* 本节包含节名。本节的类型为 **SHT_STRTAB**。不使用任何属性类型。

*.strtab* 本节包含字符串，最常见的是与符号表项关联的表示名字的字符串。如果文件具有一个包含符号字符串表的可加载段，本节的属性将包含 **SHF_ALLOC** 位。否则，此位将是 off 的。本节的类型为 **SHT_STRTAB**。

*.symtab* 本节包含符号表。如果文件具有一个包含符号表的可加载段，本节的属性将包含 **SHF_ALLOC** 位。否则，此位将是 off 的。本节的类型为 **SHT_SYMTAB**。

*.text* 本节包含 "text"，或程序的可执行指令。本节的类型为 **SHT_PROGBITS**。用到的属性为 **SHF_ALLOC** 和 **SHF_EXECINSTR**。

### 字符串和符号表

字符串表包含以 null 结尾的字符序列（复数），通常称为字符串（复数）。目标文件使用这些字符串表示符号和节名。使用字符串的地方通过字符串在字符串表节内的索引来引用。第一个字节，其索引为 0，被定义为包含一个 null 字节 ('\0')。类似地，字符串表的最后一个字节被定义为包含一个 null 字节，以确保所有的字符串均以 null 终止。

目标文件的符号表包含定位和重定位一个程序的符号定义和引用的信息。符号表索引是到这个数组的下标。

```
           typedef struct {
               uint32_t      st_name;
               Elf32_Addr    st_value;
               uint32_t      st_size;
               unsigned char st_info;
               unsigned char st_other;
               uint16_t      st_shndx;
           } Elf32_Sym;

           typedef struct {
               uint32_t      st_name;
               unsigned char st_info;
               unsigned char st_other;
               uint16_t      st_shndx;
               Elf64_Addr    st_value;
               uint64_t      st_size;
           } Elf64_Sym;
```

32 位和 64 位版本具有相同的成员，只是顺序不同。

*st_name* 这个成员包含到目标文件的符号字符串表的索引，其包含表示符号名称的字符。如果该值非 0，则它表示给出符号名称的字符串表索引。否则，符号没有名称。

*st_value* 这个成员给出了与符号关联的值。

*st_size* 许多符号具有关联的大小。如果符号不具有大小则该成员包含 0，否则是一个未知大小。

*st_info* 这个成员指定了符号的类型和绑定属性：
  - **STT_NOTYPE** 符号的类型未定义。
  - **STT_OBJECT** 符号与一个数据对象关联。
  - **STT_FUNC** 符号与一个函数或其它可执行代码关联。
  - **STT_SECTION** 符号与一个节关联。具有这种类型的符号表项主要用于重定位，且通常具有 **STB_LOCAL** 绑定类型。
  - **STT_FILE** 按照惯例，该符号的名称给出了与目标文件关联的源文件的名称。文件符号具有 **STB_LOCAL** 绑定，它的节索引是 **SHN_ABS**，且如果出现的话，它出现在文件中的其它 **STB_LOCAL** 符号的前面。
  - **STT_LOPROC, STT_HIPROC** 在 [**STT_LOPROC**, **STT_HIPROC**] 范围内的值是为特定于处理器的语义保留的。
  - **STB_LOCAL** 本地符号在包含它们的定义的目标文件之外不可见。出现在多个文件中的具有相同名称的本地符号彼此之间互不干扰。
  - **STB_GLOBAL** 全局符号对于被合并的所有目标文件都可见。全局符号的一个文件的定义将满足另一个文件对相同符号的未定义引用。
  - **STB_WEAK** 弱符号与全局符号类似，但它们的定义具有更低优先级。
  - **STB_LOPROC, STB_HIPROC** 在 [**STB_LOPROC**, **STB_HIPROC**] 范围内的值是为特定于处理器的语义保留的。

有一些宏可以用于打包和解包绑定和类型字段：
  - **ELF32_ST_BIND(** *info* **)**，**ELF64_ST_BIND(** *info* **)** 从 *st_info* 值中提取绑定。
  - **ELF32_ST_TYPE(** *info* **)**，**ELF64_ST_TYPE(** *info* **)** 从 *st_info* 值中提取类型。
  - **ELF32_ST_INFO(** *bind*，*type* **)**，**ELF64_ST_INFO(** *bind*，*type* **)** 将绑定和类型转换为 *st_info* 值。

*st_other* 这个成员定义了符号可见性。

  - **STV_DEFAULT** 默认符号可见性规则。全局的和弱符号对于其它模块可用：本地模块的引用可以由其它模块的定义插入。
  - **STV_INTERNAL** 特定于处理器的隐藏类型。
  - **STV_HIDDEN** 符号对其它模块不可用：本地模块中的引用总是解析到本地的符号（比如，符号无法由其它模块的定义插入）。
  - **STV_PROTECTED** 符号对其它模块可用，但本地模块中的引用总是解析到本地的符号。

这些宏用于提取可见性类型：
**ELF32_ST_VISIBILITY(** other **)** 或 **ELF64_ST_VISIBILITY(** other **)**

*st_shndx* 每个符号表项均是根据某些节 “定义” 的。该成员包含相关的节头表索引。

### 重定位项（Rel & Rela）

重定位是把符号引用和符号定义连接起来的过程。可重定位文件必须具有描述如何修改它们的节内容的信息，这样使得可执行文件和共享目标文件可以为进程的程序镜像包含正确的信息。重定位项就是这些数据。

不需要附加的重定位项：
```
           typedef struct {
               Elf32_Addr r_offset;
               uint32_t   r_info;
           } Elf32_Rel;

           typedef struct {
               Elf64_Addr r_offset;
               uint64_t   r_info;
           } Elf64_Rel;
```

需要附加的重定位项：
```
           typedef struct {
               Elf32_Addr r_offset;
               uint32_t   r_info;
               int32_t    r_addend;
           } Elf32_Rela;

           typedef struct {
               Elf64_Addr r_offset;
               uint64_t   r_info;
               int64_t    r_addend;
           } Elf64_Rela;
```

*r_offset* 这个成员给出了采取重定位行动的位置。对于一个可重定位文件，该值是从节的开始位置处到被重定位影响到的存储单元的字节偏移量。对于可执行文件或共享目标文件，该值是被重定位影响到的存储单元的虚拟地址。

*r_info* 该成员给出了必须执行重定位的符号表索引和应用的重定位的类型。重定位类型是特定于处理器的。当 text 引用了一个重定位项的重定位类型或符号表索引，它意味着应用 **ELF[32|64]_R_TYPE** 或 **ELF[32|64]_R_SYM**，分别地，到项的 *r_info* 成员的结果。

*r_addend* 这个成员指定了一个常量附加用于计算将被存储进可重定位字段中的值。

### 动态标签（Dyn）
*.dynamic* 节包含了一系列包含与动态链接相关的信息的结构体。*d_tag* 成员控制 *d_un* 的解释。
```
           typedef struct {
               Elf32_Sword    d_tag;
               union {
                   Elf32_Word d_val;
                   Elf32_Addr d_ptr;
               } d_un;
           } Elf32_Dyn;
           extern Elf32_Dyn _DYNAMIC[];

           typedef struct {
               Elf64_Sxword    d_tag;
               union {
                   Elf64_Xword d_val;
                   Elf64_Addr  d_ptr;
               } d_un;
           } Elf64_Dyn;
           extern Elf64_Dyn _DYNAMIC[];
```

*d_tag* 这个成员可以具有下面的任何值：

  - **DT_NULL** 标记动态节的结束。
  - **DT_NEEDED** 所需的库的名称的字符串表索引
  - **DT_PLTRELSZ** PLT 重定位项的字节大小
  - **DT_PLTGOT** PLT 和/或 GOT 的地址
  - **DT_HASH** 符号哈希表的地址
  - **DT_STRTAB** 字符串表的地址
  - **DT_SYMTAB** 符号表的地址
  - **DT_RELA** Rela 重定位表的地址
  - **DT_RELASZ** Rela 重定位表的字节大小
  - **DT_RELAENT** 重定位表项的字节大小
  - **DT_STRSZ** 字符串表的字节大小
  - **DT_SYMENT** 符号表项的字节大小
  - **DT_INIT** 初始函数的地址
  - **DT_FINI** 终止函数的地址
  - **DT_SONAME** 共享目标文件名称的字符串表偏移量
  - **DT_RPATH** 库搜索路径的字符串表偏移量（废弃）
  - **DT_SYMBOLIC** 通知链接器在为可执行文件搜索符号之前先搜索该共享目标文件。
  - **DT_REL** Rel 重定位表的地址。
  - **DT_RELSZ** Rel 重定位表的字节大小。
  - **DT_RELENT** Rel 重定位表项的字节大小。
  - **DT_PLTREL**  PLT 引用的重定位项的类型（Rela or Rel）
  - **DT_DEBUG** 用于调试的未定义内容
  - **DT_TEXTREL** 缺失这个项表示不应该为一个非可写段应用重定位项。
  - **DT_JMPREL** 仅与 PLT 相关联的重定位表项的地址
  - **DT_BIND_NOW** 指示链接器在将控制权转移给可执行程序前处理所有的重定位
  - **DT_RUNPATH** 库搜索路径的字符串表偏移量
  - **DT_LOPROC, DT_HIPROC** [**DT_LOPROC**, **DT_HIPROC**] 范围内的值为特定于处理器的语义保留。

*d_val* 该成员表示具有各种解释的整数值。

*d_ptr* 该成员表示程序虚拟地址。当解释这些地址时，实际的地址应当基于原始文件值和内存基地址计算得出。文件不包含修正这些地址的重定位项。

*_DYNAMIC* 包含 *.dynamic* 节中的所有动态数据结构的数组。它是由链接器自动填充的。

### Notes（Nhdr）

ELF notes 允许附加任意的信息给系统使用。它们主要用于核心转储文件(*e_type* 为 **ET_CORE**)，但许多项目定义了它们自己的扩展集合。比如，GNU 工具链使用 ELF notes 从链接器向 C 库传递信息。

Note 节包含一系列 notes（参考下面的 *struct* 定义）。每个 note 后跟名称字段（其长度由 *n_namesz* 定义），然后是描述符字段（其长度由 *n_descsz* 定义），且其开始地址 4 字节对齐。由于它们的任意长度，两个字段都没有在 note 结构中定义。

解析两个连续的 notes 应该阐明它们在内存中的布局的例子：
```
           void *memory, *name, *desc;
           Elf64_Nhdr *note, *next_note;

           /* The buffer is pointing to the start of the section/segment */
           note = memory;

           /* If the name is defined, it follows the note */
           name = note->n_namesz == 0 ? NULL : memory + sizeof(*note);

           /* If the descriptor is defined, it follows the name
              (with alignment) */

           desc = note->n_descsz == 0 ? NULL :
                  memory + sizeof(*note) + ALIGN_UP(note->n_namesz, 4);

           /* The next note follows both (with alignment) */
           next_note = memory + sizeof(*note) +
                                ALIGN_UP(note->n_namesz, 4) +
                                ALIGN_UP(note->n_descsz, 4);
```

记住 *n_type* 的解释依赖于由 *n_namesz* 字段定义的命名空间。如果 *n_namesz* 字段没有设置（比如，为 0），则有两个 notes 集合：一个用于核心转储文件，另一个用于所有其它 ELF 类型。如果命名空间未知，则工具通常也将 fallback 到这些 notes 集合。

```
           typedef struct {
               Elf32_Word n_namesz;
               Elf32_Word n_descsz;
               Elf32_Word n_type;
           } Elf32_Nhdr;

           typedef struct {
               Elf64_Word n_namesz;
               Elf64_Word n_descsz;
               Elf64_Word n_type;
           } Elf64_Nhdr;
```

*n_namesz* 名称字段的字节长度。在内存中内容将紧跟在这个 note 后面。名称以 null 终止。比如，如果名称是 "GNU"，则 *n_namesz* 将被设置为 4。

*n_descsz* 描述符字段的字节长度。在内存中内容将紧跟在名称字段后面。

*n_type* 依赖于名称字段的值，这个成员可以具有下面的任何值：

 - **核心转储文件 (e_type = ET_CORE)**
 Notes 被所有核心转储文件使用。这些是与操作系统和架构高度相关的，且通常需要与内核，C 库，和调试器紧密配合的。当命名空间为默认值时(例如，*n_namesz* 将被设置为0)，或者当命名空间未知时使用回退。
   - **NT_PRSTATUS**  prstatus struct
   - **NT_FPREGSET**  fpregset struct
   - **NT_PRPSINFO**  prpsinfo struct
   - **NT_PRXREG**  prxregset struct
   - **NT_TASKSTRUCT**  task structure
   - **NT_PLATFORM**  来自于 sysinfo(SI_PLATFORM) 的字符串
   - **NT_AUXV**  auxv array
   - **NT_GWINDOWS**  gwindows struct
   - **NT_ASRS**  asrset struct
   - **NT_PSTATUS**  pstatus struct
   - **NT_PSINFO**  psinfo struct
   - **NT_PRCRED**  prcred struct
   - **NT_UTSNAME**  utsname struct
   - **NT_LWPSTATUS**  lwpstatus struct
   - **NT_LWPSINFO**  lwpinfo struct
   - **NT_PRFPXREG**  fprxregset struct
   - **NT_SIGINFO**  siginfo_t (size might increase over time)
   - **NT_FILE**  包含映射文件的信息
   - **NT_PRXFPREG**  user_fxsr_struct
   - **NT_PPC_VMX**  PowerPC Altivec/VMX 寄存器
   - **NT_PPC_SPE**  PowerPC SPE/EVR 寄存器
   - **NT_PPC_VSX**  PowerPC VSX 寄存器
   - **NT_386_TLS**  i386 TLS 槽 (struct user_desc)
   - **NT_386_IOPERM**  x86 io permission bitmap (1=deny)
   - **NT_X86_XSTATE**  x86 extended state using xsave
   - **NT_S390_HIGH_GPRS**  s390 upper register halves
   - **NT_S390_TIMER**  s390 timer register
   - **NT_S390_TODCMP**  s390 time-of-day (TOD) clock comparator register
   - **NT_S390_TODPREG**  s390 time-of-day (TOD) programmable register
   - **NT_S390_CTRS**  s390 control registers
   - **NT_S390_PREFIX**  s390 prefix register
   - **NT_S390_LAST_BREAK**  s390 breaking event address
   - **NT_S390_SYSTEM_CALL**  s390 system call restart data
   - **NT_S390_TDB**  s390 transaction diagnostic block
   - **NT_ARM_VFP**  ARM VFP/NEON registers
   - **NT_ARM_TLS**  ARM TLS register
   - **NT_ARM_HW_BREAK**  ARM hardware breakpoint registers
   - **NT_ARM_HW_WATCH**  ARM hardware watchpoint registers
   - **NT_ARM_SYSTEM_CALL**  ARM system call number

 - **n_name = GNU**
 GNU 工具链使用的扩展。
   - **NT_GNU_ABI_TAG**  操作系统 ABI 信息。desc 字段将有 4 个字：
      * word 0：OS 描述符（**ELF_NOTE_OS_LINUX**，**ELF_NOTE_OS_GNU**，等等）
      * word 1：ABI 主版本号
      * word 2：ABI 小版本号
      * word 3：ABI的次级版本号
   - **NT_GNU_HWCAP**  合成 hwcap 信息。desc 字段以 2 个字开头：
      * word 0：项的个数
      * word 1：启用的项的位掩码
      然后是可变长度的项，一个字节后跟一个以空结尾的 hwcap 名称字符串。该字节给出了如果启用的话，要测试的位数，(1U << bit) & bit mask。
   - **NT_GNU_BUILD_ID**  由 GNU [ld(1)](https://man7.org/linux/man-pages/man1/ld.1.html) **--build-id** 生成的惟一的 build ID。desc 包含任何非零字节数。
   - **NT_GNU_GOLD_VERSION**  desc 包含使用的 GNU Gold 链接器版本。

 - **默认的/未知命名空间（e_type != ET_CORE）**
 当命名空间为默认（比如，*n_namesz* 将被设置为0）时，或当命名空间未知而回退时使用。
   - **NT_VERSION**  一些排序的版本字符串。
   - **NT_ARCH**  架构信息。

https://man7.org/linux/man-pages/man5/elf.5.html

可重定位目标文件的结构及其链接过程

动态链接过程

可执行文件加载及其执行过程

一个简单的 ELF 文件解析器：
```
#include <elf.h>
#include <fcntl.h>
#include <iostream>
#include <map>
#include <memory>
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

using namespace std;

template <typename Ehdr, typename Shdr>
int parse_section_header_table(Ehdr *ehdr, Shdr *shdr,
                               const char *string_table) {
  /*
   * Print each section header name and address. Notice we get the index into
   * the string table that contains each section header name with shdr.sh_name
   * member
   */
  printf("Section header list:\n");
  for (int i = 0; i < ehdr->e_shnum; ++i) {
    printf("%d-%s: %p\n", i, &string_table[shdr[i].sh_name],
           reinterpret_cast<void *>(shdr[i].sh_addr));
  }
  return 0;
}

template<typename FlagType>
const char* getFlagDescription(FlagType type) {
  static std::map<FlagType, const char*> sFlagToDescMap = {
      { PF_R, "R" },
      { PF_W, " W" },
      { PF_X, "  X" },
      { PF_R | PF_W, "RW" },
      { PF_R | PF_X, "R X" },
      { PF_W | PF_X, " WX" },
      { PF_R | PF_W | PF_X, "RWX" },
  };

  if (sFlagToDescMap.find(type) != sFlagToDescMap.end()) {
    return sFlagToDescMap[type];
  }
  return "";
}

template<typename PType>
const char* getPTypeDescription(PType type) {
  static std::map<PType, const char*> sPTypeToDescMap = {
      { PT_LOAD, "LOAD" },
      { PT_INTERP, "INTERP" },
      { PT_NOTE, "NOTE" },
      { PT_DYNAMIC, "DYNAMIC" },
      { PT_PHDR, "PHDR" },
      { PT_GNU_EH_FRAME, "GNU_EH_FRAME" },
      { PT_GNU_STACK, "GNU_STACK" },
      { PT_GNU_RELRO, "GNU_RELRO" },
  };

  if (sPTypeToDescMap.find(type) != sPTypeToDescMap.end()) {
    return sPTypeToDescMap[type];
  }
  return "";
}

template <typename Ehdr, typename Phdr>
int parse_program_header_table(Ehdr *ehdr, Phdr *phdr) {
  /*
   * Print out each segment name, and address. Except for PT_INTERP we print the
   * path to the dynamic linker (Interpreter).
   */
  printf("Program header list:\n");

  printf("  %-15s%-18s %-18s %-18s %-18s %-18s %-6s %-5s\n", "Type", "Offset", "VirtAddr",
         "PhysAddr", "FileSiz", "MemSiz", "Flags", "Align");

  const char *segment_format_str = "  %-15s0x%016x 0x%016x 0x%016x 0x%016x 0x%016x %-6s 0x%-x\n";
  for (int i = 0; i < ehdr->e_phnum; ++i) {
    printf(segment_format_str, getPTypeDescription(phdr[i].p_type),
        static_cast<int>(phdr[i].p_offset), static_cast<int>(phdr[i].p_vaddr),
        static_cast<int>(phdr[i].p_paddr), static_cast<int>(phdr[i].p_filesz),
        static_cast<int>(phdr[i].p_memsz), getFlagDescription(phdr[i].p_flags),
        phdr[i].p_align);
  }
  printf("\n");

  return 0;
}

template<typename EType>
const char* getFileTypeDescription(EType type) {
  static std::map<EType, const char*> sTypeToDescMap = {
      { ET_NONE, "NONE (None)" },
      { ET_REL, "REL (Relocatable file)" },
      { ET_EXEC, "EXEC (Executable file)" },
      { ET_DYN, "DYN (Shared object file)" },
      { ET_CORE, "CORE (Core file)" },
  };

  if (sTypeToDescMap.find(type) != sTypeToDescMap.end()) {
    return sTypeToDescMap[type];
  }
  return "";
}

template <typename Ehdr, typename Phdr, typename Shdr>
int elf_parse(void *mem, const char *filepath) {
  Ehdr *ehdr = reinterpret_cast<Ehdr *>(mem);

  /*
   * We are only parsing executables with this code.
   * So ET_EXEC marks an executable.
   */
  printf("\nELF file type is: %s.\n", getFileTypeDescription(ehdr->e_type));
  if (ehdr->e_type != ET_EXEC && ehdr->e_type != ET_DYN) {
    return -1;
  }
  printf("Program entry point: %p\n", reinterpret_cast<void *>(ehdr->e_entry));
  printf("Size of ELF file header: 0x%x.\n\n", static_cast<int>(sizeof(*ehdr)));

  /*
   * The shdr table and phdr table offsets are given by e_shoff and e_phoff
   * member of the Elf32_Ehdr
   */
  Phdr *phdr = reinterpret_cast<Phdr *>(((uint8_t *)mem) + ehdr->e_phoff);
  parse_program_header_table(ehdr, phdr);

  Shdr *shdr = reinterpret_cast<Shdr *>(((uint8_t *)mem) + ehdr->e_shoff);

  /*
   * We find the string table for the section header names with e_shstrndx which
   * gives the index of which section holds the string table.
   */
  char *string_table = ((char *)mem) + shdr[ehdr->e_shstrndx].sh_offset;
  printf("sh_offset: %u, string_table: %p\n",
         (uint32_t)shdr[ehdr->e_shstrndx].sh_offset, string_table);

  parse_section_header_table(ehdr, shdr, string_table);

  return 0;
}

class MmapObject {

public:
  ~MmapObject();
  void *GetMemBase();
  size_t GetMemSize();

public:
  static std::shared_ptr<MmapObject> Create(int fd);

private:
  MmapObject(void *mem_base, size_t length);

private:
  void *mem_base_;
  size_t length_;
};

MmapObject::MmapObject(void *mem_base, size_t length)
    : mem_base_(mem_base), length_(length) {}

MmapObject::~MmapObject() {
  if (mem_base_) {
    if (munmap(mem_base_, length_) < 0) {
      printf("Memory unmap failed.\n");
    }
    mem_base_ = nullptr;
    length_ = 0;
  }
}

void *MmapObject::GetMemBase() {
  return mem_base_;
}

size_t MmapObject::GetMemSize() {
  return length_;
}

std::shared_ptr<MmapObject> MmapObject::Create(int fd) {
  if (fd <= 0) {
    return nullptr;
  }

  struct stat file_stat;
  if (fstat(fd, &file_stat) < 0) {
    perror("fstat");
    return nullptr;
  }

  /* Map the executable into memory */
  void *mem = mmap(nullptr, file_stat.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
  if (mem == MAP_FAILED) {
    perror("mmap");
    return nullptr;
  }

  auto mem_object = std::shared_ptr<MmapObject>(new MmapObject(mem, file_stat.st_size));
  return mem_object;
}

int main(int argc, const char *argv[]) {
  if (argc < 2) {
    printf("Usage: %s <executable>\n", argv[0]);
    return 0;
  }
  int fd = -1;
  if ((fd = open(argv[1], O_RDONLY)) < 0) {
    perror("open");
    return -1;
  }

  struct stat file_stat;
  if (fstat(fd, &file_stat) < 0) {
    perror("fstat");
    return -1;
  }

  auto mem_object = MmapObject::Create(fd);
  if (!mem_object) {
    return -1;
  }

  void *mem = mem_object->GetMemBase();
  char *ident = reinterpret_cast<char *>(mem);
  /*
   * Check to see if the ELF magic (The first 4 bytes) match up as 0x7f E L F
   */
  if (strncmp(ident, ELFMAG, 4) != 0) {
    fprintf(stderr, "%s is not an ELF file.\n", argv[1]);
    return -1;
  }

  if (ident[EI_CLASS] == ELFCLASS32) {
    elf_parse<Elf32_Ehdr, Elf32_Phdr, Elf32_Shdr>(mem, argv[1]);
  } else if (ident[EI_CLASS] == ELFCLASS64) {
    elf_parse<Elf64_Ehdr, Elf64_Phdr, Elf64_Shdr>(mem, argv[1]);
  }

  getchar();
  return 0;
}
```
