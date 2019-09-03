---
title: Linux 下的 AddressSanitizer
date: 2019-05-19 11:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

AddressSanitizer 是一个性能非常好的 C/C++ 内存错误探测工具。它由编译器的插桩模块（目前，LLVM 通过）和替换了 `malloc` 函数的运行时库组成。这个工具可以探测如下这些类型的错误：
<!--more-->
 * 对堆，栈和全局内存的访问越界（堆缓冲区溢出，栈缓冲区溢出，和全局缓冲区溢出）
 * UAP（Use-after-free，悬挂指针的解引用，或者说野指针）
 * Use-after-return（无效的栈上内存，运行时标记 `ASAN_OPTIONS=detect_stack_use_after_return=1`）
 * Use-After-Scope（作用域外访问，clang 标记 -fsanitize-address-use-after-scope ）
 * 内存的重复释放
 * 初始化顺序的 bug
 * 内存泄漏

这个工具非常快。通常情况下，内存问题探测这类调试工具的引入，会导致原有应用程序运行性能的大幅下降，比如大名鼎鼎的 valgrind 据说会导致应用程序性能下降到正常情况的十几分之一，但引入 AddressSanitizer 只会减慢运行速度的一半。

# AddressSanitizer 的使用

自 LLVM 的版本 3.1 和 GCC 的版本 4.8 开始，[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 就是它们的一部分。如果需要的话，也可以从源码编译 [AddressSanitizerHowToBuild](https://github.com/google/sanitizers/wiki/AddressSanitizerHowToBuild)。

查看自己的 LLVM 版本和 GCC 版本来确认是否内置了对 AddressSanitizer 的支持：
```
$ clang --version
clang version 3.9.1-4ubuntu3~16.04.2 (tags/RELEASE_391/rc2)
Target: x86_64-pc-linux-gnu
Thread model: posix
InstalledDir: /usr/bin

$ gcc --version
gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

我本地的工具虽然版本比较老，但对 AddressSanitizer 还是支持的。

看一下前面提到的 AddressSanitizer 的运行时库：
```
$ locate asan
/usr/lib/gcc/x86_64-linux-gnu/5/libasan.a
/usr/lib/gcc/x86_64-linux-gnu/5/libasan.so
/usr/lib/gcc/x86_64-linux-gnu/5/libasan_preinit.o
/usr/lib/gcc/x86_64-linux-gnu/5/32/libasan.a
/usr/lib/gcc/x86_64-linux-gnu/5/32/libasan.so
/usr/lib/gcc/x86_64-linux-gnu/5/32/libasan_preinit.o
/usr/lib/gcc/x86_64-linux-gnu/5/include/sanitizer/asan_interface.h
/usr/lib/gcc/x86_64-linux-gnu/5/x32/libasan.a
/usr/lib/gcc/x86_64-linux-gnu/5/x32/libasan.so
/usr/lib/gcc/x86_64-linux-gnu/5/x32/libasan_preinit.o
/usr/lib/gcc/x86_64-linux-gnu/7/libasan.a
/usr/lib/gcc/x86_64-linux-gnu/7/libasan.so
/usr/lib/gcc/x86_64-linux-gnu/7/libasan_preinit.o
/usr/lib/gcc/x86_64-linux-gnu/7/include/sanitizer/asan_interface.h
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/libasan.a
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/libasan.so
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/libasan_preinit.o
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/include/sanitizer/asan_interface.h
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/sf/libasan.a
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/sf/libasan.so
/usr/lib/gcc-cross/arm-linux-gnueabihf/5/sf/libasan_preinit.o
/usr/lib/llvm-3.9/lib/clang/3.9.1/asan_blacklist.txt
/usr/lib/llvm-3.9/lib/clang/3.9.1/include/sanitizer/asan_interface.h
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-i386.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-i386.so
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-i686.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-i686.so
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-preinit-i386.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-preinit-i686.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-preinit-x86_64.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-x86_64.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-x86_64.a.syms
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan-x86_64.so
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan_cxx-i386.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan_cxx-i686.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan_cxx-x86_64.a
/usr/lib/llvm-3.9/lib/clang/3.9.1/lib/linux/libclang_rt.asan_cxx-x86_64.a.syms
/usr/lib/llvm-5.0/lib/clang/5.0.2/asan_blacklist.txt
/usr/lib/llvm-5.0/lib/clang/5.0.2/include/sanitizer/asan_interface.h
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-i386.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-i386.so
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-i686.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-i686.so
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-preinit-i386.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-preinit-i686.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-preinit-x86_64.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-x86_64.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-x86_64.a.syms
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan-x86_64.so
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan_cxx-i386.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan_cxx-i686.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan_cxx-x86_64.a
/usr/lib/llvm-5.0/lib/clang/5.0.2/lib/linux/libclang_rt.asan_cxx-x86_64.a.syms
/usr/lib/x86_64-linux-gnu/libasan.so.2
/usr/lib/x86_64-linux-gnu/libasan.so.2.0.0
/usr/lib/x86_64-linux-gnu/libasan.so.4
/usr/lib/x86_64-linux-gnu/libasan.so.4.0.0
/usr/lib32/libasan.so.2
/usr/lib32/libasan.so.2.0.0
/usr/libx32/libasan.so.2
/usr/libx32/libasan.so.2.0.0
```

为了使用 [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)，需要在使用 GCC 或 Clang 编译链接程序时加上 `-fsanitize=address` 开关。为了获得合理的性能，可以加上 `-O1` 或更高。为了在错误信息中获得更友好的栈追踪信息可以加上 `-fno-omit-frame-pointer`。为了获得完美的栈追踪信息，还可以禁用内联（使用 `-O1`）和尾调用消除（`-fno-optimize-sibling-calls`）

下面是一段存在内存访问错误的代码：
```
// main.cpp
int main(int argc, char **argv) {
  int *array = new int[100];
  delete [] array;
  return array[argc];  // BOOM
}
```

使用 GCC 编译并运行：
```
$ gcc -fsanitize=address -fno-omit-frame-pointer -O1 -g -o main main.cpp
$ ./main 
=================================================================
==5385==ERROR: AddressSanitizer: heap-use-after-free on address 0x61400000fe44 at pc 0x0000004007d4 bp 0x7ffddf0bafb0 sp 0x7ffddf0bafa0
READ of size 4 at 0x61400000fe44 thread T0
    #0 0x4007d3 in main addresssanitizer_demo/main.cpp:4
    #1 0x7f297fc2282f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)
    #2 0x4006b8 in _start (addresssanitizer_demo/main+0x4006b8)

0x61400000fe44 is located 4 bytes inside of 400-byte region [0x61400000fe40,0x61400000ffd0)
freed by thread T0 here:
    #0 0x7f2980065caa in operator delete[](void*) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x99caa)
    #1 0x4007a8 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:3
    #2 0x7f297fc2282f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

previously allocated by thread T0 here:
    #0 0x7f29800656b2 in operator new[](unsigned long) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x996b2)
    #1 0x400798 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:2
    #2 0x7f297fc2282f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

SUMMARY: AddressSanitizer: heap-use-after-free addresssanitizer_demo/main.cpp:4 main
Shadow bytes around the buggy address:
  0x0c287fff9f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fa0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0c287fff9fc0: fa fa fa fa fa fa fa fa[fd]fd fd fd fd fd fd fd
  0x0c287fff9fd0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9fe0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9ff0: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c287fffa000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fffa010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==5385==ABORTING
```

AddressSanitizer 在探测到内存错误之后，向 stderr 打印了错误信息并以非 0 值返回码退出。AddressSanitizer 在发现第一个错误时退出程序。这主要是基于如下的设计：

 * 这种方法允许 AddressSanitizer 产生更快和更小的生成码（总共 ~5%）。
 * 解决 bug 变得无法避免。AddressSanitizer 不产生误报。一旦内存崩溃发生，则程序进入不一致的状态，这可能导致令人费解的结果和潜在的误导性的后续报告。

如果进程运行在沙盒中且运行在 OS X 10.10 或更早的版本上，则需要设置`DYLD_INSERT_LIBRARIES` 环境变量并把它指向由编译器打包用于构建可执行文件的 ASan 库。（可以搜索名字中包含 `asan` 的动态链接库来找到这个库。）如果没有设置环境变量，则进程将试图重新执行。同时记住，当把可执行文件移动到另一台机器时，ASan 库也需要复制过去。

编译时如果遗漏了 `-g` 参数，导致可执行文件中缺乏调试信息，则探测到内存错误时，AddressSanitizer 吐出来的错误信息中，无法显示具体的出错的代码行，就像下面这样：
```
$ gcc -fsanitize=address -fno-omit-frame-pointer -O1  -o main main.cpp 
$ ./main 
=================================================================
==5403==ERROR: AddressSanitizer: heap-use-after-free on address 0x61400000fe44 at pc 0x0000004007d4 bp 0x7ffd981fce20 sp 0x7ffd981fce10
READ of size 4 at 0x61400000fe44 thread T0
    #0 0x4007d3 in main (addresssanitizer_demo/main+0x4007d3)
    #1 0x7f9de8af482f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)
    #2 0x4006b8 in _start (addresssanitizer_demo/main+0x4006b8)

0x61400000fe44 is located 4 bytes inside of 400-byte region [0x61400000fe40,0x61400000ffd0)
freed by thread T0 here:
    #0 0x7f9de8f37caa in operator delete[](void*) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x99caa)
    #1 0x4007a8 in main (addresssanitizer_demo/main+0x4007a8)
    #2 0x7f9de8af482f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

previously allocated by thread T0 here:
    #0 0x7f9de8f376b2 in operator new[](unsigned long) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x996b2)
    #1 0x400798 in main (addresssanitizer_demo/main+0x400798)
    #2 0x7f9de8af482f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

SUMMARY: AddressSanitizer: heap-use-after-free ??:0 main
Shadow bytes around the buggy address:
  0x0c287fff9f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fa0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0c287fff9fc0: fa fa fa fa fa fa fa fa[fd]fd fd fd fd fd fd fd
  0x0c287fff9fd0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9fe0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9ff0: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c287fffa000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fffa010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==5403==ABORTING
```

上面的 `-g` 选项也可以用 `-ggdb` 选项（尽可能的生成 gdb 的可以使用的调试信息）替换。
```
$ gcc -fsanitize=address -fno-omit-frame-pointer -O1 -ggdb -o main main.cpp 
$ ./main 
=================================================================
==5495==ERROR: AddressSanitizer: heap-use-after-free on address 0x61400000fe44 at pc 0x0000004007d4 bp 0x7fff7014ca10 sp 0x7fff7014ca00
READ of size 4 at 0x61400000fe44 thread T0
    #0 0x4007d3 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:4
    #1 0x7fdddb19982f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)
    #2 0x4006b8 in _start (addresssanitizer_demo/main+0x4006b8)

0x61400000fe44 is located 4 bytes inside of 400-byte region [0x61400000fe40,0x61400000ffd0)
freed by thread T0 here:
    #0 0x7fdddb5dccaa in operator delete[](void*) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x99caa)
    #1 0x4007a8 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:3
    #2 0x7fdddb19982f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

previously allocated by thread T0 here:
    #0 0x7fdddb5dc6b2 in operator new[](unsigned long) (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x996b2)
    #1 0x400798 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:2
    #2 0x7fdddb19982f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

SUMMARY: AddressSanitizer: heap-use-after-free /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:4 main
Shadow bytes around the buggy address:
  0x0c287fff9f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fa0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0c287fff9fc0: fa fa fa fa fa fa fa fa[fd]fd fd fd fd fd fd fd
  0x0c287fff9fd0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9fe0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9ff0: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c287fffa000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fffa010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==5495==ABORTING
```

使用 LLVM/Clang 编译与上面使用 GCC 编译基本相同：
```
$ clang++ -fsanitize=address -fno-omit-frame-pointer -O1 -g -o test main.cpp
$ ./test 
=================================================================
==5508==ERROR: AddressSanitizer: heap-use-after-free on address 0x61400000fe44 at pc 0x00000050770f bp 0x7ffea20137b0 sp 0x7ffea20137a8
READ of size 4 at 0x61400000fe44 thread T0
    #0 0x50770e in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:4:10
    #1 0x7fdeb802382f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291
    #2 0x4192b8 in _start (addresssanitizer_demo/test+0x4192b8)

0x61400000fe44 is located 4 bytes inside of 400-byte region [0x61400000fe40,0x61400000ffd0)
freed by thread T0 here:
    #0 0x504c20 in operator delete[](void*) (addresssanitizer_demo/test+0x504c20)
    #1 0x5076de in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:3:3
    #2 0x7fdeb802382f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291

previously allocated by thread T0 here:
    #0 0x5045a0 in operator new[](unsigned long) (addresssanitizer_demo/test+0x5045a0)
    #1 0x5076d3 in main /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:2:16
    #2 0x7fdeb802382f in __libc_start_main /build/glibc-Cl5G7W/glibc-2.23/csu/../csu/libc-start.c:291

SUMMARY: AddressSanitizer: heap-use-after-free /home/hanpfei0306/data/MyProjects/addresssanitizer_demo/main.cpp:4:10 in main
Shadow bytes around the buggy address:
  0x0c287fff9f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9f90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fa0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fff9fb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0c287fff9fc0: fa fa fa fa fa fa fa fa[fd]fd fd fd fd fd fd fd
  0x0c287fff9fd0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9fe0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c287fff9ff0: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c287fffa000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c287fffa010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==5508==ABORTING
```

可以通过 readelf 看一下 GCC 和 LLVM 生成的可执行文件有什么差别：
```
# GCC 生成的可执行文件
$ readelf -s -W main | grep asan
Symbol table '.dynsym' contains 15 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __asan_report_load4
    10: 0000000000400650     0 FUNC    GLOBAL DEFAULT  UND __asan_init_v4
    44: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS asan_preinit.cc
    57: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __asan_report_load4
    62: 0000000000400650     0 FUNC    GLOBAL DEFAULT  UND __asan_init_v4
    69: 0000000000600dd0     8 OBJECT  GLOBAL HIDDEN    19 __local_asan_preinit

# LLVM 生成的可执行文件
$ readelf -s -W test | grep asan
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __asan_default_options
    16: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __asan_on_error
    49: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __asan_default_suppressions
    66: 0000000000426740   466 FUNC    GLOBAL DEFAULT   13 __asan_stack_malloc_0
    70: 00000000004269b0   504 FUNC    GLOBAL DEFAULT   13 __asan_stack_malloc_1
. . . . . .
   271: 0000000000428000   134 FUNC    GLOBAL DEFAULT   13 __asan_stack_free_8
   272: 000000000042b1e0    44 FUNC    GLOBAL DEFAULT   13 __asan_register_image_globals
   273: 00000000004d7550    44 FUNC    GLOBAL DEFAULT   13 __asan_report_load4_noabort
   275: 00000000004282a0   134 FUNC    GLOBAL DEFAULT   13 __asan_stack_free_9
. . . . . .
   635: 00000000004d7460    44 FUNC    GLOBAL DEFAULT   13 __asan_report_load2
   636: 00000000004d74f0    44 FUNC    GLOBAL DEFAULT   13 __asan_report_load4
   644: 00000000004d7580    44 FUNC    GLOBAL DEFAULT   13 __asan_report_load8
. . . . . .
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS asan_allocator.cc.o
    38: 0000000000419370   254 FUNC    LOCAL  DEFAULT   13 _ZN6__asanL10RZSize2LogEj
    39: 0000000000421c90   214 FUNC    LOCAL  DEFAULT   13 _ZN11__sanitizer20SizeClassAllocator64ILm105553116266496ELm4398046511104ELm0ENS_12SizeClassMapILm17ELm128ELm16EEEN6__asan20AsanMapUnmapCallbackEE15DeallocateBatchEPNS_14AllocatorStatsEmPNS2_13TransferBatchE.isra.32
    40: 0000000000419470  1241 FUNC    LOCAL  DEFAULT   13 _ZN6__asan9AsanChunk8UsedSizeEb.part.19
  1162: 0000000000422670  1101 FUNC    WEAK   HIDDEN    13 _ZN11__sanitizer28SizeClassAllocatorLocalCacheINS_20SizeClassAllocator64ILm105553116266496ELm4398046511104ELm0ENS_12SizeClassMapILm17ELm128ELm16EEEN6__asan20AsanMapUnmapCallbackEEEE6RefillEPS6_m
  1178: 00000000004d74f0    44 FUNC    GLOBAL DEFAULT   13 __asan_report_load4
  1194: 00000000004cff10    57 FUNC    GLOBAL HIDDEN    13 _ZN6__asan10AsanTSDGetEv
  1196: 00000000004d6ea0     8 FUNC    GLOBAL DEFAULT   13 __asan_get_report_sp
  1202: 00000000004a75e0    63 FUNC    GLOBAL HIDDEN    13 _ZN6__asan13SetThreadNameEPKc
  1212: 00000000004d7b50    96 FUNC    GLOBAL DEFAULT   13 __asan_load1_noabort
. . . . . .
  2202: 00000000004d9780    96 FUNC    GLOBAL DEFAULT   13 __asan_init
  2208: 000000000041a720  2468 FUNC    GLOBAL HIDDEN    13 _ZN6__asan22FindHeapChunkByAddressEm
  2219: 00000000004d9800     7 FUNC    GLOBAL HIDDEN    13 _ZN6__asan20GetMallocContextSizeEv
. . . . . .
  3790: 0000000000419ac0    19 FUNC    GLOBAL HIDDEN    13 _ZN6__asan13AsanChunkView11IsAllocatedEv
  3796: 00000000004d08e0    97 FUNC    GLOBAL HIDDEN    13 _ZN6__asan25ThreadNameWithParenthesisEPNS_17AsanThreadContextEPcm
```

主要看两个可执行文件中都有的符号 `__asan_report_load4` 和 `__asan_init`，可以看到 GCC 生成的可执行文件动态链接 ASan 库，LLVM 静态链接。

# 符号化输出

[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 收集如下事件的调用栈：

 * `malloc` 和 `malloc`
 * 线程创建
 * 失败

`malloc` 和 `malloc` 发生的相对频繁，且它对于快速解开调用栈非常重要。[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 使用一个依赖帧指针的简单的 unwinder。

如果不关心 `malloc`/`free` 调用栈，简单地完全禁用（使用 `malloc_context_size=0` 运行时标记）unwinder。

每个栈帧需要被符号化（当然，如果二进制文件编译时带有调试信息）。给定一台 PC，我们需要输出：
```
#0xabcdf function_name file_name.cc:1234
```

AddressSanitizer 使用 Clang 包中的 [llvm-symbolizer](http://llvm.org/docs/CommandGuide/llvm-symbolizer.html) 符号化栈追踪信息（注意理想的 llvm-symbolizer  版本必须与 ASan 运行时库匹配）。为了使 AddressSanitizer 符号化它的输出，需要设置 `ASAN_SYMBOLIZER_PATH` 环境变量指向 `llvm-symbolizer` 二进制文件，或确保 `llvm-symbolizer` 在 `$PATH` 中：
```
$ ASAN_SYMBOLIZER_PATH=/usr/local/bin/llvm-symbolizer ./a.out
==9442== ERROR: AddressSanitizer heap-use-after-free on address 0x7f7ddab8c084 at pc 0x403c8c bp 0x7fff87fb82d0 sp 0x7fff87fb82c8
READ of size 4 at 0x7f7ddab8c084 thread T0
    #0 0x403c8c in main example_UseAfterFree.cc:4
    #1 0x7f7ddabcac4d in __libc_start_main ??:0
0x7f7ddab8c084 is located 4 bytes inside of 400-byte region [0x7f7ddab8c080,0x7f7ddab8c210)
freed by thread T0 here:
    #0 0x404704 in operator delete[](void*) ??:0
    #1 0x403c53 in main example_UseAfterFree.cc:4
    #2 0x7f7ddabcac4d in __libc_start_main ??:0
previously allocated by thread T0 here:
    #0 0x404544 in operator new[](unsigned long) ??:0
    #1 0x403c43 in main example_UseAfterFree.cc:2
    #2 0x7f7ddabcac4d in __libc_start_main ??:0
==9442== ABORTING
```

`llvm-symbolizer` 符号化工具属于 llvm 包，Ubuntu 下具体的安装方法可以参考 [LLVM Debian/Ubuntu nightly packages](http://apt.llvm.org/)。

如果上面的方法不起作用，可以使用一个单独的脚本来离线地符号化结果（在线的符号化可以通过设置 `ASAN_OPTIONS=symbolize=0`，或者设置一个空的 `ASAN_SYMBOLIZER_PATH` 环境变量（`$ export ASAN_SYMBOLIZER_PATH=`）来强制禁用）：
```
$ ASAN_OPTIONS=symbolize=0 ./a.out 2> log
$ projects/compiler-rt/lib/asan/scripts/asan_symbolize.py / < log | c++filt
==9442== ERROR: AddressSanitizer heap-use-after-free on address 0x7f7ddab8c084 at pc 0x403c8c bp 0x7fff87fb82d0 sp 0x7fff87fb82c8
READ of size 4 at 0x7f7ddab8c084 thread T0
    #0 0x403c8c in main example_UseAfterFree.cc:4
    #1 0x7f7ddabcac4d in __libc_start_main ??:0
...
```

这个脚本接收一个可选的参数 `-- a file prefix`。子串 `.*prefix` 将被从文件名中移除。

上面的 `c++filt` 用于解函数名的符号重组，这也可以通过给 `asan_symbolize.py` 脚本添加 `-d` 参数来完成。

在 OS X 上可能需要对二进制文件运行 `dsymutil` 以在 AddressSanitizer 报告中获得 `file:line` 信息。

还可以引入自己的栈追踪格式，使用 `stack_trace_format` 运行时标记完成。例如：
```
% ./a.out
    ...
    #0 0x4b615d in main /home/you/use-after-free.cc:12:3
    ...
% ASAN_OPTIONS='stack_trace_format="[frame=%n, function=%f, location=%S]"' ./a.out
    ...
    [frame=0, function=main, location=/home/you/use-after-free.cc:12:3]
```

# AddressSanitizer 算法

## 简单的版本
运行时库替换 `malloc` 和 `free` 函数。把 malloc 分配的内存区域（红色区域）附近放入一些特定的字节（使中毒）。把 `free` 的内存放入隔离区，并且也放入一些特定的字节（使中毒）。程序中的每次内存访问由编译器以下面的方式做一个转换。
之前：
```
*address = ...;  // or: ... = *address;
```

之后：
```
if (IsPoisoned(address)) {
  ReportError(address, kAccessSize, kIsWrite);
}
*address = ...;  // or: ... = *address;
```

棘手的部分是如何把 `IsPoisoned` 实现的高效，且 `ReportError` 紧凑。同时，对某些访问的插桩可能被证明是冗余的。

## 内存映射

虚拟地址空间被分割为 2 个互斥的类别：

 * 主应用内存（`Mem`）：这种内存由常规的应用代码使用。
 * 阴影内存（`Shadow`）：这种内存包含阴影值（或元数据）。阴影和主应用程序内存之间存在对应关系。在主内存中 **使中毒** 一个字节意味着在对应的阴影区写入一些特殊的值。

这两种类别的内存应该以阴影内存（`MemToShadow`）可以被快速计算出来的方式进行组织。

编译器执行的插桩如下：
```
shadow_address = MemToShadow(address);
if (ShadowIsPoisoned(shadow_address)) {
  ReportError(address, kAccessSize, kIsWrite);
}
```

## 映射
[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 把 8 个字节的应用内存映射为 1 个字节的阴影内存。

对于任何 8 字节对齐的应用内存只有 9 个不同的值：

 * qword 中的所有 8 字节是未中毒的（比如，可寻址）。阴影值为 0。
 * qword 中的所有 8 字节是中毒的（比如，不可寻址）。阴影值为负数。
 * 起始的 `k` 字节是未中毒的，其余的 `8-k` 字节是中毒的。阴影值为 `k`。这主要由 `malloc` 的行为保证，即 `malloc` 总是返回 8 字节对齐的内存块。一个 qword 对齐的不同字节具有不同状态的仅有的情况是 malloc 的区域的尾部。比如，如果我们调用 `malloc(13)`，我们将拥有一个完整的未中毒的 qword 和一个开头 5 字节未中毒的 qword。

插桩看起来像下面这样：
```
byte *shadow_address = MemToShadow(address);
byte shadow_value = *shadow_address;
if (shadow_value) {
  if (SlowPathCheck(shadow_value, address, kAccessSize)) {
    ReportError(address, kAccessSize, kIsWrite);
  }
}
```

```
// Check the cases where we access first k bytes of the qword
// and these k bytes are unpoisoned.
bool SlowPathCheck(shadow_value, address, kAccessSize) {
  last_accessed_byte = (address & 7) + kAccessSize - 1;
  return (last_accessed_byte >= shadow_value);
}
```

`MemToShadow(ShadowAddr)` 落入不可寻址的 `ShadowGap` 区域。因此，如果程序试图直接访问阴影区域中的内存位置，它将崩溃。

## 64-bit

```
Shadow = (Mem >> 3) + 0x7fff8000;
```

| [0x10007fff8000, 0x7fffffffffff] |  HighMem |
|-------|------|
| [0x02008fff7000, 0x10007fff7fff] | HighShadow |
| [0x00008fff7000, 0x02008fff6fff] | ShadowGap |
| [0x00007fff8000, 0x00008fff6fff] | LowShadow |
| [0x000000000000, 0x00007fff7fff] | LowMem |

## 32 bit

```
Shadow = (Mem >> 3) + 0x20000000;
```
| [0x40000000, 0xffffffff] | HighMem |
|-------|------|
| [0x28000000, 0x3fffffff] | HighShadow |
| [0x24000000, 0x27ffffff] | ShadowGap |
| [0x20000000, 0x23ffffff] | LowShadow |
| [0x00000000, 0x1fffffff] | LowMem |

## 超紧凑的阴影区

使用更紧凑的阴影区内存也是可能的，比如：
```
Shadow = (Mem >> 7) | kOffset;
```

还在实验中。

## 报告错误

`ReportError` 可以被实现为一个调用（当前默认就是这样），但也有一些其它的，稍微更加高效和/或更加紧凑的方案。此刻默认的行为**是**：

 * 把失败地址拷贝到 `%rax` (`%eax`)
 * 执行 `ud2` （产生 SIGILL）
 * 在 `ud2` 之后的一个字节指令中编码访问类型和大小。整体上这 3 个指令需要 5-6 个字节的机器码。

仅使用单个指令（比如 `ud2`）也是可能的，但这需要在运行时库中有一个完整的反汇编器（或一些其它的 hacks）。

## 栈

为了捕获栈溢出，[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 插桩的代码像这样：

原始的代码：
```
void foo() {
  char a[8];
  ...
  return;
}
```

插桩后的代码：
```
void foo() {
  char redzone1[32];  // 32-byte aligned
  char a[8];          // 32-byte aligned
  char redzone2[24];
  char redzone3[32];  // 32-byte aligned
  int  *shadow_base = MemToShadow(redzone1);
  shadow_base[0] = 0xffffffff;  // poison redzone1
  shadow_base[1] = 0xffffff00;  // poison redzone2, unpoison 'a'
  shadow_base[2] = 0xffffffff;  // poison redzone3
  ...
  shadow_base[0] = shadow_base[1] = shadow_base[2] = 0; // unpoison all
  return;
}
```

## 插桩的代码示例（x86_64）
```
# long load8(long *a) { return *a; }
0000000000000030 <load8>:
  30:	48 89 f8             	mov    %rdi,%rax
  33:	48 c1 e8 03          	shr    $0x3,%rax
  37:	80 b8 00 80 ff 7f 00 	cmpb   $0x0,0x7fff8000(%rax)
  3e:	75 04                	jne    44 <load8+0x14>
  40:	48 8b 07             	mov    (%rdi),%rax   <<<<<< original load
  43:	c3                   	retq   
  44:	52                   	push   %rdx
  45:	e8 00 00 00 00       	callq  __asan_report_load8
```

```
# int  load4(int *a)  { return *a; }
0000000000000000 <load4>:
   0:	48 89 f8             	mov    %rdi,%rax
   3:	48 89 fa             	mov    %rdi,%rdx
   6:	48 c1 e8 03          	shr    $0x3,%rax
   a:	83 e2 07             	and    $0x7,%edx
   d:	0f b6 80 00 80 ff 7f 	movzbl 0x7fff8000(%rax),%eax
  14:	83 c2 03             	add    $0x3,%edx
  17:	38 c2                	cmp    %al,%dl
  19:	7d 03                	jge    1e <load4+0x1e>
  1b:	8b 07                	mov    (%rdi),%eax    <<<<<< original load
  1d:	c3                   	retq   
  1e:	84 c0                	test   %al,%al
  20:	74 f9                	je     1b <load4+0x1b>
  22:	50                   	push   %rax
  23:	e8 00 00 00 00       	callq  __asan_report_load4
```

## 未对齐的访问

当前紧凑的映射将不捕获未对齐的部分越界访问：
```
int *x = new int[2]; // 8 bytes: [0,7].
int *u = (int*)((char*)x + 6);
*u = 1;  // Access to range [6-9]
```

[https://github.com/google/sanitizers/issues/100](https://github.com/google/sanitizers/issues/100) 中描述了一个可行的方案，但它付出了性能的代价。

## 运行时库

### Malloc

运行时库替换 `malloc`/`free`，并提供错误报告函数，如 `__asan_report_load8`。

`malloc` 分配由红区围绕的请求数量的内存。阴影值对应的红区被下毒，主内存区域的阴影值被清除。

`free` 用阴影值对整个区域下毒，并把内存块放入一个隔离区（这样在一定时间内这个内存块将不会再次被 malloc 返回）。

## 二进制兼容性

二进制兼容性问题指的是，分别用 GCC 和 clang 编译的不同工程，它们之间存在依赖，或要一起使用时出现的问题。如本人遇到的场景，有一个用 clang 编译的动态链接库 A，有一个 GCC 编译的动态链接库 B，还有一个用 GCC 编译的 C/C++ 应用程序 C。它们之间的依赖关系为，B 依赖于 A，C 依赖于 A 和 B。其中只有编译 A 时，开启了 ASAN 以检查内存问题。

上述场景中，在用 CMake 通过 G++ 编译链接 C 应用程序时报出了如下的错误：
```
[100%] Linking CXX executable AgoraSDKDemoApp
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_store4’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_poison_cxx_array_cookie’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_exp_store2’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_stack_free_7’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_exp_load_n’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_load4’未定义的引用
```

看上去是由于没有链接 ASAN 库的原因。通过为 G++ 加上对 ASAN 库的链接依赖（`-lasan`），再看一下。

这次的错误少了许多，但链接还是失败了：
```
$ make
[  5%] Linking CXX shared library libAgoraSDKWrapper.so
[ 80%] Built target AgoraSDKWrapper
[ 85%] Linking CXX executable AgoraSDKDemoDlApp
[ 90%] Built target AgoraSDKDemoDlApp
[ 95%] Linking CXX executable AgoraSDKDemoApp
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_unregister_elf_globals’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_register_elf_globals’未定义的引用
collect2: error: ld returned 1 exit status
CMakeFiles/AgoraSDKDemoApp.dir/build.make:95: recipe for target 'AgoraSDKDemoApp' failed
make[2]: *** [AgoraSDKDemoApp] Error 1
CMakeFiles/Makefile2:141: recipe for target 'CMakeFiles/AgoraSDKDemoApp.dir/all' failed
make[1]: *** [CMakeFiles/AgoraSDKDemoApp.dir/all] Error 2
Makefile:83: recipe for target 'all' failed
make: *** [all] Error 2
```

为 B 和 C 工程的编译加上编译标记 `-fsanitize=address`，再次用 g++ 编译：
```
[100%] Linking CXX executable AgoraSDKDemoApp
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_unregister_elf_globals’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_register_elf_globals’未定义的引用
collect2: error: ld returned 1 exit status
CMakeFiles/AgoraSDKDemoApp.dir/build.make:95: recipe for target 'AgoraSDKDemoApp' failed
make[2]: *** [AgoraSDKDemoApp] Error 1
CMakeFiles/Makefile2:141: recipe for target 'CMakeFiles/AgoraSDKDemoApp.dir/all' failed
make[1]: *** [CMakeFiles/AgoraSDKDemoApp.dir/all] Error 2
Makefile:83: recipe for target 'all' failed
make: *** [all] Error 2
```
错误还是一样的。

为 g++ 添加 `-v` 参数，查看编译过程的详细信息，可以看到最后链接时执行的命令如下：
```
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 7.4.0-1ubuntu1~18.04.1' --with-bugurl=file:///usr/share/doc/gcc-7/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++ --prefix=/usr --with-gcc-major-version-only --program-suffix=-7 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-libmpx --enable-plugin --enable-default-pie --with-system-zlib --with-target-system-zlib --enable-objc-gc=auto --enable-multiarch --disable-werror --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu
Thread model: posix
gcc version 7.4.0 (Ubuntu 7.4.0-1ubuntu1~18.04.1) 
COMPILER_PATH=/usr/lib/gcc/x86_64-linux-gnu/7/:/usr/lib/gcc/x86_64-linux-gnu/7/:/usr/lib/gcc/x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/7/:/usr/lib/gcc/x86_64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/x86_64-linux-gnu/7/:/usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/7/../../../../lib/:/lib/x86_64-linux-gnu/:/lib/../lib/:/usr/lib/x86_64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/x86_64-linux-gnu/7/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-std=c++11' '-fsanitize=address' '-v' '-g' '-o' 'AgoraSDKDemoApp' '-L/usr/local/lib' '-L/usr/local/lib64' '-L/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
 /usr/lib/gcc/x86_64-linux-gnu/7/collect2 -plugin /usr/lib/gcc/x86_64-linux-gnu/7/liblto_plugin.so -plugin-opt=/usr/lib/gcc/x86_64-linux-gnu/7/lto-wrapper -plugin-opt=-fresolution=/tmp/cczW5GpK.res -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc --sysroot=/ --build-id --eh-frame-hdr -m elf_x86_64 --hash-style=gnu --as-needed -dynamic-linker /lib64/ld-linux-x86-64.so.2 -pie -z now -z relro -o AgoraSDKDemoApp /usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu/Scrt1.o /usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/7/crtbeginS.o -L/usr/local/lib -L/usr/local/lib64 -L/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk -L/usr/lib/gcc/x86_64-linux-gnu/7 -L/usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/7/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/7/../../.. /usr/lib/gcc/x86_64-linux-gnu/7/libasan_preinit.o --push-state --no-as-needed -lasan --pop-state CMakeFiles/AgoraSDKDemoApp.dir/AgoraSDKDemoApp/src/main.cpp.o -rpath /usr/local/lib:/usr/local/lib64:/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk:/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/build -logg -lpthread -lopusfile -lopus -lagora_rtc_sdk -lpthread libAgoraSDKWrapper.so -lagora_rtc_sdk -logg -lpthread -lopusfile -lopus -lagora_rtc_sdk -lstdc++ -lm -lgcc_s -lgcc -lc -lgcc_s -lgcc /usr/lib/gcc/x86_64-linux-gnu/7/crtendS.o /usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu/crtn.o
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_unregister_elf_globals’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_register_elf_globals’未定义的引用
collect2: error: ld returned 1 exit status
CMakeFiles/AgoraSDKDemoApp.dir/build.make:95: recipe for target 'AgoraSDKDemoApp' failed
make[2]: *** [AgoraSDKDemoApp] Error 1
CMakeFiles/Makefile2:141: recipe for target 'CMakeFiles/AgoraSDKDemoApp.dir/all' failed
make[1]: *** [CMakeFiles/AgoraSDKDemoApp.dir/all] Error 2
Makefile:83: recipe for target 'all' failed
make: *** [all] Error 2
```

通过 `readelf -s -W` 查看 g++ 链接的 asan 动态链接库中的符号：
```
$ readelf -s -W /usr/lib/x86_64-linux-gnu/libasan.so.4 | grep asan_register
   206: 0000000000036cb0    21 FUNC    GLOBAL DEFAULT   12 __asan_register_globals
   279: 0000000000037200    44 FUNC    GLOBAL DEFAULT   12 __asan_register_image_globals
```

确实没有看到上面报错信息中提到的 `__asan_unregister_elf_globals` 和 `__asan_register_elf_globals` 这两个符号。

不用 g++ 编译 B 和 C 工程，改用 clang，去掉对 ASAN 库的依赖：
```
1 warning generated.
[ 90%] Linking CXX executable AgoraSDKDemoDlApp
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_store4’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_poison_cxx_array_cookie’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_exp_store2’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_stack_free_7’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_report_exp_load_n’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_load4’未定义的引用
```

遇到了和用 g++ 编译时遇到的同样的找不到 ASAN 的符号的问题。加上对 ASAN 库的依赖：
```
[ 85%] Linking CXX executable AgoraSDKDemoDlApp
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_unregister_elf_globals’未定义的引用
/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk/libagora_rtc_sdk.so：对‘__asan_register_elf_globals’未定义的引用
clang: error: linker command failed with exit code 1 (use -v to see invocation)
CMakeFiles/AgoraSDKDemoDlApp.dir/build.make:95: recipe for target 'AgoraSDKDemoDlApp' failed
```

遇到了和用 g++ 编译时遇到的相同的问题。为 B 和 C 工程的编译加上编译标记 `-fsanitize=address`，再次用 clang 编译，终于编译通过。然而在运行时却遇到了问题：
```
$ build/AgoraSDKDemoDlApp  -m 1 -h
==33058==Your application is linked against incompatible ASan runtimes.
```

用 clang 编译时，去掉对 ASAN 库的依赖。再次编译运行，终于 OK。给 clang++ 加上 `-v` 参数，可以看到最终的链接过程如下：
```
[100%] Linking CXX executable AgoraSDKDemoApp
clang version 7.1.0-svn353565-1~exp1~20190406090509.61 (branches/release_70)
Target: x86_64-pc-linux-gnu
Thread model: posix
InstalledDir: /usr/bin
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/8
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.4.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0
Candidate multilib: .;@m64
Selected multilib: .;@m64
 "/usr/bin/ld" -z relro --hash-style=gnu --build-id --eh-frame-hdr -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o AgoraSDKDemoApp /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/../../../x86_64-linux-gnu/crt1.o /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/../../../x86_64-linux-gnu/crti.o /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/crtbegin.o -L/usr/local/lib -L/usr/local/lib64 -L/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk -L/usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0 -L/usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/../../../x86_64-linux-gnu -L/lib/x86_64-linux-gnu -L/lib/../lib64 -L/usr/lib/x86_64-linux-gnu -L/usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/../../.. -L/usr/lib/llvm-7/bin/../lib -L/lib -L/usr/lib --whole-archive /usr/lib/llvm-7/lib/clang/7.1.0/lib/linux/libclang_rt.asan-x86_64.a --no-whole-archive --dynamic-list=/usr/lib/llvm-7/lib/clang/7.1.0/lib/linux/libclang_rt.asan-x86_64.a.syms --whole-archive /usr/lib/llvm-7/lib/clang/7.1.0/lib/linux/libclang_rt.asan_cxx-x86_64.a --no-whole-archive --dynamic-list=/usr/lib/llvm-7/lib/clang/7.1.0/lib/linux/libclang_rt.asan_cxx-x86_64.a.syms CMakeFiles/AgoraSDKDemoApp.dir/AgoraSDKDemoApp/src/main.cpp.o -rpath /usr/local/lib:/usr/local/lib64:/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/libs/agora_media_sdk:/home/hanpfei/dev_data/home/hanpfei/data/CorpProjects/media_sdk_script/media_sdk3/app/linux/AgoraSDKDemo/build -logg -lpthread -lopusfile -lopus -lagora_rtc_sdk -lpthread libAgoraSDKWrapper.so -lagora_rtc_sdk -logg -lpthread -lopusfile -lopus -lagora_rtc_sdk -lstdc++ -lm --no-as-needed -lpthread -lrt -lm -ldl -lgcc_s -lgcc -lc -lgcc_s -lgcc /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/crtend.o /usr/bin/../lib/gcc/x86_64-linux-gnu/7.4.0/../../../x86_64-linux-gnu/crtn.o
[100%] Built target AgoraSDKDemoApp
```

clang 链接了 `/usr/lib/llvm-7/lib/clang/7.1.0/lib/linux/libclang_rt.asan-x86_64.a`。

总结一下，有一个用 clang 编译，且开启了 ASAN debug 的库，在用 g++ 编译依赖该库的库和应用时遇到链接问题，尝试的几种方法的实际结果：

 - 1. g++ 直接编译 ===> 失败，找不到所有的 ASAN 符号
 - 2. g++ + 链接 asan 库 ===> 失败，找不到符号 `__asan_unregister_elf_globals` 和 `__asan_register_elf_globals`。
 - 3. g++ + `-fsanitize=address` 标记 ===> 失败，和链接 asan 库遇到相同的链接错误。
 - 4. clang++ 直接编译 ===> 失败，与 1 中遇到的相同的错误。
 - 5. clang++ + 链接 asan 库 ===> 失败，与 2 中遇到的相同的错误。
 - 6. clang++ + 链接 asan 库 + `-fsanitize=address` 标记 ===> 编译成功，但运行失败。
 - **7. clang++ + `-fsanitize=address` 标记 ===> 编译成功，运行成功，PASS**。

因而，需要用 clang++ + `-fsanitize=address` 标记编译。

### 参考文档
 * [Clang AddressSanitizer](http://clang.llvm.org/docs/AddressSanitizer.html)
 * [AddressSanitizerAlgorithm](https://github.com/google/sanitizers/wiki/AddressSanitizerAlgorithm)
 * [llvm-symbolizer](http://llvm.org/docs/CommandGuide/llvm-symbolizer.html)
 * [Address Sanitizer 用法](https://www.jianshu.com/p/3a2df9b7c353)
 * [sanitizers](https://github.com/google/sanitizers)
 * [AddressSanitizerExampleHeapOutOfBounds](https://github.com/google/sanitizers/wiki/AddressSanitizerExampleHeapOutOfBounds)
 * [AddressSanitizerCallStack](https://github.com/google/sanitizers/wiki/AddressSanitizerCallStack)
 * [LLVM Debian/Ubuntu nightly packages](http://apt.llvm.org/)
