---
title: 在 Linux 程序中使用 breakpad
date: 2022-06-24 20:23:49
categories: 音视频开发
tags:
- 音视频开发
---

本文概述在 Linux 平台的可执行程序或动态链接库中使用 Breakpad 的方法。
<!--more-->
## 构建 Breakpad 库

Breakpad 提供了一个 Autotools 构建系统，它将构建 Breakpad 的 Linux 客户端库和处理程序。通过 git 从 **[breakpad](https://github.com/hanpfei/breakpad)** 下载 Breakpad 的源码。然后在 Breakpad 的源码目录中运行 `./configure && make`，这将生成客户端静态库文件 **src/client/linux/libbreakpad_client.a**，它包含了为应用程序或动态链接库生成 minidumps 所需的所有代码。构建过程除了客户端库之外，还生成了必须的工具。

Google 原始的 Breakpad [源码仓库](https://chromium.googlesource.com/breakpad/breakpad/) 没有包含 Linux syscall support 库，这在构建 Breakpad 时会报出如下错误：

```
~/data/opensource/breakpad$ make
depbase=`echo src/tools/linux/core2md/core2md.o | sed 's|[^/]*$|.deps/&|;s|\.o$||'`;\
g++ -DHAVE_CONFIG_H -I. -I./src  -I./src   -Wmissing-braces -Wnon-virtual-dtor -Woverloaded-virtual -Wreorder -Wsign-compare -Wunused-local-typedefs -Wunused-variable -Wvla -Werror -fPIC -g -O2 -MT src/tools/linux/core2md/core2md.o -MD -MP -MF $depbase.Tpo -c -o src/tools/linux/core2md/core2md.o src/tools/linux/core2md/core2md.cc &&\
mv -f $depbase.Tpo $depbase.Po
In file included from ./src/client/linux/dump_writer_common/thread_info.h:37,
                 from ./src/client/linux/minidump_writer/linux_dumper.h:54,
                 from ./src/client/linux/minidump_writer/minidump_writer.h:42,
                 from src/tools/linux/core2md/core2md.cc:34:
./src/common/memory_allocator.h:50:10: fatal error: third_party/lss/linux_syscall_support.h: 没有那个文件或目录
   50 | #include "third_party/lss/linux_syscall_support.h"
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
make: *** [Makefile:5701：src/tools/linux/core2md/core2md.o] 错误 1
```

Breakpad 这样的调试诊断代码有时不太方便访问封装的 C 库，但它依然会有访问系统功能的需要，如打开文件，读写文件等，此时它会通过系统调用直接访问系统功能。Linux syscall support 库为直接通过系统调用访问系统功能提供支持。Linux syscall support 库的源码仓库地址为 https://chromium.googlesource.com/linux-syscall-support.git。笔者在 GitHub 上的 Breakpad fork 仓库 **[breakpad](https://github.com/hanpfei/breakpad)** 已经包含了 Linux syscall support 库，无需做其它什么事情，即可直接编译通过。

## 把 Breakpad 集成进你的程序

首先，配置你的构建过程把 **libbreakpad_client.a** 的目录地址添加到链接库文件的搜索路径，同时为链接添加 **breakpad_client** 库，把 **libbreakpad_client.a** 链接进你的二进制文件，然后设置 include 路径包含 **google-breakpad** 源码树中的 ** src** 目录。**breakpad_client** 库依赖 **pthread** 库，因而也需要添加对 **pthread** 库的依赖。接下来，包含异常处理器的头文件：

```
#include "client/linux/handler/exception_handler.h"
```

现在你可以实例化一个 `ExceptionHandler ` 对象。`ExceptionHandler ` 对象的整个生命周期中异常处理都是激活的，因此你应该在你的程序启动过程中，尽可能早实例化它，并使其尽可能接近关闭状态一直保持激活。要做任何有用的事情，`ExceptionHandler` 构造函数都需要一个可以写入 minidump 的路径，以及一个回调函数来接收有关已写入的 minidump 的信息：

```
#include <stdlib.h>

#include <unistd.h>

#include "src/client/linux/handler/exception_handler.h"
#include "src/common/linux/linux_libc_support.h"
#include "src/third_party/lss/linux_syscall_support.h"

static bool DumpCallback(const google_breakpad::MinidumpDescriptor &descriptor,
    void *context, bool success) {
  if (!success) {
    static const char msg[] = "Failed to write minidump\n";
    sys_write(2, msg, sizeof(msg) - 1);
    return false;
  }

  static const char msg[] = "Wrote minidump: ";
  sys_write(2, msg, sizeof(msg) - 1);
  sys_write(2, descriptor.path(), strlen(descriptor.path()));
  sys_write(2, "\n", 1);

  return true;
}

static void DoSomethingWhichCrashes() {
  int local_var = 1;
  *reinterpret_cast<volatile char*>(NULL) = 1;
}

int main(int argc, const char *argv[]) {
  google_breakpad::MinidumpDescriptor minidump(".");
  google_breakpad::ExceptionHandler breakpad(minidump, NULL, DumpCallback, NULL,
      true, -1);
  DoSomethingWhichCrashes();
  return 0;
}
```

编译并运行这个示例程序，应该会在当前目录下生成一个 minidump 文件，而且它应该会在退出之前打印出 minidump 文件的文件名。你可以在 [exception_handler.h](https://github.com/hanpfei/breakpad/blob/main/src/client/linux/handler/exception_handler.h) 的源文件中读到更多关于 `ExceptionHandler` 构造函数的其它参数的内容。

**注意**：你应该在回调函数中执行尽可能少的操作。你的程序已经处于了不安全的状态。分配内存或调用其它共享库的函数可能是不安全的。最安全的做法是 `fork` 并 `exec` 一个新的进程来完成你需要做的任何工作。如果你必须要在回调中做些事情，Breakpad 源码包含了 [一些 libc 函数的简单重实现](https://github.com/hanpfei/breakpad/blob/main/src/common/linux/linux_libc_support.h)，来避免直接调用 libc，以及 [一个执行 Linux 系统调用的头文件](https://chromium.googlesource.com/linux-syscall-support/+/master)（在 **src/third_party/lss** 中）以避免调用其它共享库。

## 发送 minidump 文件

在实际的应用中，你将想要以某种方式处理 minidump，比如把它发送到服务器以做分析。Breakpad 源码树包含了一些可能有点用的 [HTTP 上传源码](https://github.com/hanpfei/breakpad/blob/main/src/common/linux/http_upload.h)，以及一个 [minidump 上传工具](https://github.com/hanpfei/breakpad/blob/main/src/tools/linux/symupload/minidump_upload.cc)。

## 为你的程序生成符号

为了生成有用的栈追踪，Breakpad 需要你把你的二进制中的调试符号转为 [文本格式的符号文件](https://github.com/hanpfei/breakpad/blob/main/docs/symbol_files.md)。首先，确保你在编译你的二进制时带了 `-g` 参数以包含调试符号。接下来，通过在 Breakpad 源码目录下运行 `configure && make` 编译 `dump_syms` 工具 。接下来，对你的二进制文件运行 `dump_syms` 以生成文本格式的符号。比如，如果你的主二进制名称为 `HelloBreakpad`：

```
$ breakpad/src/tools/linux/dump_syms/dump_syms ./HelloBreakpad > HelloBreakpad.sym
```

为了通过 `minidump_stackwalk` 工具使用这些符号，你将需要把它们放在一个特定的目录结构下。符号文件的第一行包含了你生成这种目录结构所需的信息，比如（你的输出可能不太一样）：

```
~/workspace/HelloBreakpad/Debug$ head -n5 HelloBreakpad.sym 
MODULE Linux x86_64 467289A80132830FD70182B041DC6F960 HelloBreakpad
INFO CODE_ID A889724632010F83D70182B041DC6F96F093FBF2
FILE 0 /home/hanpfei/data/opensource/breakpad/./src/client/linux/handler/exception_handler.h
FILE 1 /home/hanpfei/data/opensource/breakpad/./src/client/linux/handler/microdump_extra_info.h
FILE 2 /home/hanpfei/data/opensource/breakpad/./src/client/linux/handler/minidump_descriptor.h
~/workspace/HelloBreakpad/Debug$ mkdir -p symbols/HelloBreakpad/467289A80132830FD70182B041DC6F960
~/workspace/HelloBreakpad/Debug$ mv HelloBreakpad.sym symbols/HelloBreakpad/467289A80132830FD70182B041DC6F960/
```

你也可以使用 Mozilla 代码仓库中的 [symbolstore.py](https://dxr.mozilla.org/mozilla-central/source/toolkit/crashreporter/tools/symbolstore.py) 脚本，它封装了这些步骤。

## 处理 minidump 生成栈追踪

Breakpad 包含了一个称为 `minidump_stackwalk` 的工具，它接收一个 minidump 及它对应的文本格式的符号，生成一个符号化的栈追踪。如果你按照上面的指示编译了 Breakpad 源码，它应该位于 **breakpad/src/processor** 目录下。简单地把 minidump 和符号路径作为命令行参数传入：

```
~/workspace/HelloBreakpad/Debug$ minidump_stackwalk efebac08-3b27-40b1-456cd9b4-33b33260.dmp symbols/
Operating system: Linux
                  0.0.0 Linux 5.13.0-40-generic #45~20.04.1-Ubuntu SMP Mon Apr 4 09:38:31 UTC 2022 x86_64
CPU: amd64
     family 6 model 158 stepping 10
     1 CPU

GPU: UNKNOWN

Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0
Process uptime: not available

Thread 0 (crashed)
 0  HelloBreakpad!DoSomethingWhichCrashes [HelloBreakpad.cpp : 28 + 0x0]
    rax = 0x0000000000000000   rdx = 0x0000000000000000
    rcx = 0x00005586cc34bf80   rbx = 0x00005586cb3276d0
    rsi = 0x0000000000000000   rdi = 0x00005586cb32f6c0
    rbp = 0x00007fffc54a2b50   rsp = 0x00007fffc54a2b50
     r8 = 0x0000000000000000    r9 = 0x00005586cc34bf78
    r10 = 0x0000000000000008   r11 = 0x00007f3735f21be0
    r12 = 0x00005586cb314df0   r13 = 0x00007fffc54a2e00
    r14 = 0x0000000000000000   r15 = 0x0000000000000000
    rip = 0x00005586cb314ff7
    Found by: given as instruction pointer in context
 1  HelloBreakpad!main [HelloBreakpad.cpp : 35 + 0x5]
    rbx = 0x00005586cb3276d0   rbp = 0x00007fffc54a2d10
    rsp = 0x00007fffc54a2b60   r12 = 0x00005586cb314df0
    r13 = 0x00007fffc54a2e00   r14 = 0x0000000000000000
    r15 = 0x0000000000000000   rip = 0x00005586cb3150c9
    Found by: call frame info
 2  libc.so.6 + 0x240b3
    rbx = 0x00005586cb3276d0   rbp = 0x0000000000000000
    rsp = 0x00007fffc54a2d20   r12 = 0x00005586cb314df0
    r13 = 0x00007fffc54a2e00   r14 = 0x0000000000000000
    r15 = 0x0000000000000000   rip = 0x00007f3735d590b3
    Found by: call frame info
 3  HelloBreakpad!DoSomethingWhichCrashes [HelloBreakpad.cpp : 29 + 0x3]
    rsp = 0x00007fffc54a2d40   rip = 0x00005586cb314ffd
    Found by: stack scanning
 4  HelloBreakpad + 0x156d0
    rbp = 0x00005586cb314ffd   rsp = 0x00007fffc54a2d48
    rip = 0x00005586cb3276d0
    Found by: call frame info
 5  HelloBreakpad!_start + 0x2e
    rsp = 0x00007fffc54a2df0   rip = 0x00005586cb314e1e
    Found by: stack scanning
 6  0x7fffc54a2df8
    rsp = 0x00007fffc54a2df8   rip = 0x00007fffc54a2df8
    Found by: call frame info

Loaded modules:
0x5586cb312000 - 0x5586cb327fff  HelloBreakpad  ???  (main)
0x7f3735be6000 - 0x7f3735c99fff  libm.so.6  ???
0x7f3735d35000 - 0x7f3735ecefff  libc.so.6  ???  (WARNING: No symbols, libc.so.6, E774DB9F17B26CD093172A8243F8547F0)
0x7f3735f27000 - 0x7f3735f3bfff  libgcc_s.so.1  ???
0x7f3735f42000 - 0x7f37360c8fff  libstdc++.so.6  ???
0x7f3736124000 - 0x7f373613afff  libpthread.so.0  ???
0x7f3736161000 - 0x7f3736184fff  ld-linux-x86-64.so.2  ???
0x7fffc54e3000 - 0x7fffc54e4fff  linux-gate.so  ???
```

它在 stderr 上产生详细输出，在 stdout 上产生堆栈跟踪，因此你可能需要重定向 stderr。

参考文档

[Google Breakpad 学习笔记](https://www.jianshu.com/p/295ebf42b05b)

[How To Add Breakpad To Your Linux Application](https://github.com/hanpfei/OpenRTCClient/blob/m98_4758/webrtc/third_party/breakpad/breakpad/docs/linux_starter_guide.md)

[Google Breakpad](https://wiki.rdkcentral.com/display/RDK/Google+Breakpad)

