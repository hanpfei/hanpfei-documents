---
title: 在 Mac 客户端 C++ 代码中使用 breakpad
date: 2022-07-01 21:17:39
categories: 音视频开发
tags:
- 音视频开发
---

本文概述在 Mac 平台的 C++ 可执行程序或动态链接库中使用 Breakpad 的方法。

如 [Breakpad 入门](https://www.jianshu.com/p/fbe8f7a975ac) 中的说明，整个 breakpad 系统既包括在 C++ 程序中接入的 breakpad 客户端库，也包括生成符号的工具 dump_syms 和将生成的 minidump 转为符号的 minidump_stackwalk 工具以及上传 minidump 文件的工具。本文将简要说明所有这些二进制的编译及使用方法。
<!--more-->
## 构建 Breadpad

这里说明基于 **WebRTC** / **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 的 GN + ninja 构建系统的构建方法。

通过 Git 下载 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 的源码。然后在 OpenRTCClient 的源码根目录中执行如下命令 `./build_system/webrtc_pack mac debug crash_catch_system`，这将生成客户端静态库文件 **build/mac/debug/libbreakpad_client.a**，其中包含 arm64 和 x64   两个 CPU 架构的静态库，同时还会生成用于生成符号文件和将生成的 minidump 转为符号的工具，它们位于 **build/mac/x64/debug**。

## 把 Breakpad 集成进你的程序

首先，配置你的构建过程把 **libbreakpad_client.a** 的目录地址添加到链接库文件的搜索路径，同时为链接添加 **breakpad_client** 库，把 **libbreakpad_client.a** 链接进你的二进制文件，然后设置 include 路径包含 breakpad 源码树中的 **src** 目录。

Google breakpad 本身提供的 `ExceptionHandler` 类接口在各个平台上虽然差别也不大，但也不完全一样。这给 breakpad 的接入带来了一定的负担。

在 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 中，笔者实现了一个简单的封装，即 `open_rtc::InstallCrashHandler()` 和 `open_rtc::UninstallCrashHandler()`。要使用这个接口，包含崩溃处理器的头文件：

```
#include "client/crash_handler.h"
```

调用 `open_rtc::InstallCrashHandler()` 安装崩溃处理器。之后在调用 `open_rtc::UninstallCrashHandler()` 之前，异常处理都是激活的，因此你应该在你的程序启动过程中，尽可能早地调用 `open_rtc::InstallCrashHandler()`，并使其尽可能接近关闭状态一直保持激活。要做任何有用的事情，`open_rtc::InstallCrashHandler()` 都需要一个可以写入 minidump 的路径，以及一个回调函数来接收有关已写入的 minidump 的信息：

```
#include <stdlib.h>

#include "client/crash_handler.h"

bool minidumpCallback(const char* dump_dir,
                      const char* minidump_id,
                      void* context,
                      bool succeeded) {
  printf("Dump path: %s, minidump_id %s\n", dump_dir, minidump_id);
  return succeeded;
}

static void crashfunc() {
  volatile int* a = (int*)(NULL);
  *a = 1;
}

int main(int argc, const char *argv[]) {
  open_rtc::InstallCrashHandler(".", nullptr, minidumpCallback, nullptr);
  crashfunc();
  return 0;
}
```

编译并运行这个示例程序，应该会在当前目录下生成一个 minidump 文件，而且它应该会在退出之前打印出 minidump 文件的目录路径，和 id，id 即为 minidump 文件的文件名，实际的 minidump 文件带有文件扩展名 ".dmp"。

也可以直接使用 Google breakpad 提供的 `ExceptionHandler` 类来接入 breakpad。此时首先包含头文件：

```
#include "client/mac/handler/exception_handler.h"
```

然后创建 `ExceptionHandler` 类对象。`ExceptionHandler` 对象的整个生命周期中异常处理都是激活的，因此应该在程序启动过程中，尽可能早实例化它，并使其尽可能接近关闭状态一直保持激活。要做任何有用的事情，`ExceptionHandler` 构造函数都需要一个可以写入 minidump 的路径，以及一个回调函数来接收有关已写入的 minidump 的信息：

```
#include <stdio.h>
#include "client/mac/handler/exception_handler.h"

bool MinidumpCallback(const char* dump_dir,
                      const char* minidump_id,
                      void* context,
                      bool succeeded) {
  printf("Dump path: %s, minidump_id %s\n", dump_dir, minidump_id);
  return succeeded;
}

int main(int argc, const char *argv[]) {
  google_breakpad::ExceptionHandler eh(".", NULL, MinidumpCallback, NULL,
                                       true, nullptr);
  crashfunc();
}
```

编译并运行这个程序与前面那个程序的运行结果相同。

按照 WebRTC 的构建配置，`USE_PROTECTED_ALLOCATIONS=1` 宏定义是开启的 (`webrtc/third_party/breakpad/BUILD.gn`)。但 breakpad 中有如下这样的一段代码 (`webrtc/third_party/breakpad/breakpad/src/client/mac/handler/exception_handler.cc`)：
```
bool ExceptionHandler::InstallHandler() {
  // If a handler is already installed, something is really wrong.
  if (gProtectedData.handler != NULL) {
    return false;
  }
  if (!IsOutOfProcess()) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask, SIGABRT);
    sa.sa_sigaction = ExceptionHandler::SignalHandler;
    sa.sa_flags = SA_SIGINFO;

    scoped_ptr<struct sigaction> old(new struct sigaction);
    if (sigaction(SIGABRT, &sa, old.get()) == -1) {
      return false;
    }
    old_handler_.swap(old);
    gProtectedData.handler = this;
#if USE_PROTECTED_ALLOCATIONS
    assert(((size_t)(gProtectedData.protected_buffer) & PAGE_MASK) == 0);
    mprotect(gProtectedData.protected_buffer, PAGE_SIZE, PROT_READ);
#endif
  }

  try {
#if USE_PROTECTED_ALLOCATIONS
    previous_ = new (gBreakpadAllocator->Allocate(sizeof(ExceptionParameters)) )
      ExceptionParameters();
#else
    previous_ = new ExceptionParameters();
#endif
  }
  catch (std::bad_alloc) {
    return false;
  } 
```

这段代码在 `ExceptionHandler` 对象创建时执行。全局变量 `gBreakpadAllocator` 实际上是在 `webrtc/third_party/breakpad/breakpad/src/client/mac/Framework/Breakpad.mm` 初始化的，这段代码只有通过 `BreakpadCreate()` 在 OC 代码中接入 breakpad 客户端才会执行。这样在直接通过 breakpad 的 C++ 接口接入 breakpad 客户端时，全局变量 `gBreakpadAllocator` 值为空，进而导致 `ExceptionHandler` 对象创建时由于空指针而使程序崩溃。

在 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 项目中，通过把宏 `USE_PROTECTED_ALLOCATIONS=1` 的定义去掉来解决这个通过 C++ 接口接入 breakpad 客户端导致的问题。当然检查 `gBreakpadAllocator` 的值也可以解决这个问题。

## 为程序生成符号

如上面的说明，通过 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 构建 breakpad 会一并生成各种工具。这里以 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 的 smoke_test 为例说明这个过程。

构建 `smoke_test`：

```
OpenRTCClient %  ./build_system/webrtc_build build:smoke_test mac x64 debug
```

生成符号文件：

```
OpenRTCClient %  ./build/mac/x64/debug/dump_syms -g build/mac/x64/debug/smoke_test.dSYM build/mac/x64/debug/smoke_test > build/mac/x64/debug/smoke_test.sym
```

**[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 所基于的 M98 版 WebRTC 代码的 `webrtc/build/toolchain/apple/linker_driver.py` 似乎有些问题，最终链接生成可执行文件时报类似如下这些 warning：

```
warning: (x86_64)  could not find object file symbol for symbol __ZNSt3__17forwardIPN6webrtc20AudioEncoderG722ImplEEEOT_RNS_16remove_referenceIS4_E4typeE
warning: (x86_64)  could not find object file symbol for symbol __ZNSt3__122__compressed_pair_elemIPN6webrtc12AudioEncoderELi0ELb0EEC2IPNS1_20AudioEncoderG722ImplEvEEOT_
warning: (x86_64)  could not find object file symbol for symbol __ZNSt3__122__compressed_pair_elemINS_14default_deleteIN6webrtc12AudioEncoderEEELi1ELb1EEC2INS1_INS2_20AudioEncoderG722ImplEEEvEEOT_
warning: (x86_64)  could not find object file symbol for symbol __ZNSt3__114default_deleteIN6webrtc12AudioEncoderEEC2INS1_20AudioEncoderG722ImplEEERKNS0_IT_EEPNS_9enable_ifIXsr14is_convertibleIPS6_PS2_EE5valueEvE4typeE
warning: (x86_64)  could not find object file symbol for symbol __ZN6webrtc16AudioEncoderOpus11SdpToConfigERKNS_14SdpAudioFormatE
warning: (x86_64)  could not find object file symbol for symbol __ZN6webrtc16AudioEncoderOpus16MakeAudioEncoderERKNS_22AudioEncoderOpusConfigEiN4absl8optionalINS_16AudioCodecPairIdEEE
warning: (x86_64)  could not find object file symbol for symbol __ZN6webrtc16AudioEncoderOpus17QueryAudioEncoderERKNS_22AudioEncoderOpusConfigE
warning: (x86_64)  could not find object file symbol for symbol __ZN6webrtc16AudioEncoderOpus23AppendSupportedEncodersEPNSt3__16vectorINS_14AudioCodecSpecENS1_9allocatorIS3_EEEE
. . . . . .
warning: (x86_64)  could not find object file symbol for symbol _dav1d_itx_dsp_init_x86_8bpc
warning: (x86_64)  could not find object file symbol for symbol _iclip_u8
warning: (x86_64)  could not find object file symbol for symbol _dav1d_loop_filter_dsp_init_8bpc
warning: (x86_64)  could not find object file symbol for symbol _dav1d_loop_filter_dsp_init_x86_8bpc
warning: (x86_64)  could not find object file symbol for symbol _iclip_u8
warning: (x86_64)  could not find object file symbol for symbol _dav1d_loop_restoration_dsp_init_8bpc
warning: (x86_64)  could not find object file symbol for symbol _dav1d_loop_restoration_dsp_init_x86_8bpc
warning: (x86_64)  could not find object file symbol for symbol _iclip_u8
warning: (x86_64)  could not find object file symbol for symbol _dav1d_mc_dsp_init_8bpc
warning: (x86_64)  could not find object file symbol for symbol _dav1d_mc_dsp_init_x86_8bpc
warning: (x86_64)  could not find object file symbol for symbol _dav1d_get_cpu_flags_x86
 build success 
```

Chromium 的邮件列表中也有在讨论这个问题，[macOS build warnings](https://groups.google.com/a/chromium.org/g/chromium-dev/c/0DErUBlignQ)。这个问题似乎会导致生成符号文件失败：

```
smoke_test.dSYM/Contents/Resources/DWARF/smoke_test: the DIE at offset 0x134c917 has a DW_AT_specification attribute referring to the DIE at offset 0x134c8c6, which was not marked as a declaration
smoke_test.dSYM/Contents/Resources/DWARF/smoke_test: the DIE at offset 0x134ca01 has a DW_AT_specification attribute referring to the DIE at offset 0x134c9b1, which was not marked as a declaration
smoke_test.dSYM/Contents/Resources/DWARF/smoke_test: the DIE at offset 0x134caeb has a DW_AT_specification attribute referring to the DIE at offset 0x134ca9b, which was not marked as a declaration
smoke_test.dSYM/Contents/Resources/DWARF/smoke_test: the DIE at offset 0x134cc0b has a DW_AT_specification attribute referring to the DIE at offset 0x134cbc9, which was not marked as a declaration
Assertion failed: (it_debug == externs_.end() || (*it_debug)->address >= range.address + range.size), function AddFunction, file module.cc, line 175.
```

链接生成的二进制文件，导致 `build/mac/x64/debug/dump_syms` 崩溃执行失败。

在 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 项目中，通过把 `webrtc/build/toolchain/apple/linker_driver.py` 替换为 M88 的 `webrtc/build/toolchain/mac/linker_driver.py` 来解决这个问题。这样即可成功生成符号文件。

为了通过 `minidump_stackwalk` 工具使用这些符号文件，需要把它们放在一个特定的目录结构下。符号文件的第一行包含了生成这种目录结构所需的信息，比如（你的输出可能不太一样）：

```
OpenRTCClient % ./build/mac/x64/debug/dump_syms -g build/mac/x64/debug/smoke_test.dSYM build/mac/x64/debug/smoke_test > build/mac/x64/debug/smoke_test.sym
OpenRTCClient % head build/mac/x64/debug/smoke_test.sym                                                                                                    
MODULE mac x86_64 4C4C44D755553144A13B271E21EDCA370 smoke_test
FILE 0 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/System/Library/Frameworks/CoreFoundation.framework/Headers/CFBase.h
FILE 1 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/System/Library/Frameworks/CoreFoundation.framework/Headers/CFByteOrder.h
FILE 2 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/_ctype.h
FILE 3 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/_wctype.h
FILE 4 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/architecture/byte_order.h
FILE 5 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/dispatch/once.h
FILE 6 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/dispatch/queue.h
FILE 7 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/libkern/i386/_OSByteOrder.h
FILE 8 /Users/henryhan/Projects/opensource/OpenRTCClient/build/mac/x64/debug/../../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.0.sdk/usr/include/math.h
OpenRTCClient % mkdir -p build/mac/x64/debug/symbols/smoke_test/4C4C44D755553144A13B271E21EDCA370
OpenRTCClient % mv build/mac/x64/debug/smoke_test.sym build/mac/x64/debug/symbols/smoke_test/4C4C44D755553144A13B271E21EDCA370
```

## 处理 minidump 生成栈追踪

这里先模拟一个崩溃的发生。在 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 项目的 smoke_test 中有几个测试用例可以产生崩溃：

```
OpenRTCClient % ./build/mac/x64/debug/smoke_test '--gtest_filter=CrashCatchTest.*' --gtest_also_run_disabled_tests
Note: Google Test filter = CrashCatchTest.*
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from CrashCatchTest
[ RUN      ] CrashCatchTest.DISABLED_crash_catch_common
Dump path: ~/OpenRTCClient/build/mac/x64/debug, minidump_id A25EA989-A9EC-400F-B44F-AF4B517101A1
```

崩溃产生时，会有一些捕获的崩溃的 minidump 文件相关的信息吐出来。根据这些信息我们可以知道，崩溃文件的完整路径为 `~/OpenRTCClient/build/mac/x64/debug/A25EA989-A9EC-400F-B44F-AF4B517101A1.dmp`。

Breakpad 包含了一个称为 `minidump_stackwalk` 的工具，它接收一个 minidump 及它对应的文本格式的符号，生成一个符号化的栈追踪。如果按照上面的指示编译了 Breakpad 源码，它应该位于 `OpenRTCClient/build/mac/x64/debug` 目录下。简单地把 minidump 和符号路径作为命令行参数传入：

```
OpenRTCClient % build/mac/x64/debug/minidump_stackwalk ./build/mac/x64/debug/A25EA989-A9EC-400F-B44F-AF4B517101A1.dmp ./build/mac/x64/debug/symbols 
Operating system: Mac OS X
                  11.6.7 20G630
CPU: amd64
     family 6 model 158 stepping 10
     12 CPUs

GPU: UNKNOWN

Crash reason:  EXC_BAD_ACCESS / KERN_INVALID_ADDRESS
Crash address: 0x0
Process uptime: 0 seconds

Thread 0 (crashed)
 0  smoke_test!crashfunc() [crash_catch_test.cpp : 48 + 0x0]
    rax = 0x0000000000000000   rdx = 0x00000000000fb340
    rcx = 0x0000000000000000   rbx = 0x0000000000000000
    rsi = 0x00000000d587bb72   rdi = 0x0000000107d11080
    rbp = 0x00007ffeeebff1c0   rsp = 0x00007ffeeebff1c0
     r8 = 0x00000000000000d7    r9 = 0x0000000000000008
    r10 = 0x00007ff972a00000   r11 = 0x0000000000000000
    r12 = 0x0000000000000000   r13 = 0x0000000000000000
    r14 = 0x0000000000000000   r15 = 0x0000000000000000
    rip = 0x000000010101744e
    Found by: given as instruction pointer in context
 1  smoke_test!CrashCatchTest_DISABLED_crash_catch_common_Test::TestBody() [crash_catch_test.cpp : 53 + 0x5]
    rbp = 0x00007ffeeebff200   rsp = 0x00007ffeeebff1d0
    rip = 0x00000001010173ff
    Found by: previous frame's frame pointer
 2  smoke_test!void testing::internal::HandleSehExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) [gtest.cc : 2631 + 0x2]
    rbp = 0x00007ffeeebff260   rsp = 0x00007ffeeebff210
    rip = 0x0000000101095e4b
    Found by: previous frame's frame pointer
 3  smoke_test!void testing::internal::HandleExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) [gtest.cc : 2686 + 0x15]
    rbp = 0x00007ffeeebff2d0   rsp = 0x00007ffeeebff270
    rip = 0x00000001010719c7
    Found by: previous frame's frame pointer
 4  smoke_test!testing::Test::Run() [gtest.cc : 2706 + 0x24]
    rbp = 0x00007ffeeebff330   rsp = 0x00007ffeeebff2e0
    rip = 0x0000000101071911
    Found by: previous frame's frame pointer
 5  smoke_test!testing::TestInfo::Run() [gtest.cc : 2885 + 0x5]
    rbp = 0x00007ffeeebff3b0   rsp = 0x00007ffeeebff340
    rip = 0x0000000101072455
    Found by: previous frame's frame pointer
 6  smoke_test!testing::TestSuite::Run() [gtest.cc : 3044 + 0x5]
    rbp = 0x00007ffeeebff430   rsp = 0x00007ffeeebff3c0
    rip = 0x00000001010732fb
    Found by: previous frame's frame pointer
 7  smoke_test!testing::internal::UnitTestImpl::RunAllTests() [gtest.cc : 5915 + 0x5]
    rbp = 0x00007ffeeebff510   rsp = 0x00007ffeeebff440
    rip = 0x000000010107f763
    Found by: previous frame's frame pointer
 8  smoke_test!bool testing::internal::HandleSehExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) [gtest.cc : 2631 + 0x2]
    rbp = 0x00007ffeeebff570   rsp = 0x00007ffeeebff520
    rip = 0x000000010109bccb
    Found by: previous frame's frame pointer
 9  smoke_test!bool testing::internal::HandleExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) [gtest.cc : 2686 + 0x15]
    rbp = 0x00007ffeeebff5e0   rsp = 0x00007ffeeebff580
    rip = 0x000000010107f277
    Found by: previous frame's frame pointer
10  smoke_test!testing::UnitTest::Run() [gtest.cc : 5482 + 0x27]
    rbp = 0x00007ffeeebff650   rsp = 0x00007ffeeebff5f0
    rip = 0x000000010107f15f
    Found by: previous frame's frame pointer
11  smoke_test!RUN_ALL_TESTS() [gtest.h : 2497 + 0x5]
    rbp = 0x00007ffeeebff660   rsp = 0x00007ffeeebff660
    rip = 0x000000010104ce41
    Found by: previous frame's frame pointer
12  smoke_test!main [main.cpp : 12 + 0x5]
    rbp = 0x00007ffeeebff690   rsp = 0x00007ffeeebff670
    rip = 0x000000010104cdf2
    Found by: previous frame's frame pointer
13  libdyld.dylib + 0x15f3d
    rbp = 0x00007ffeeebff6a0   rsp = 0x00007ffeeebff6a0
    rip = 0x00007fff2036bf3d
    Found by: previous frame's frame pointer

Loaded modules:
0x101000000 - 0x104299fff  smoke_test  ???  (main)
0x7fff20088000 - 0x7fff20089fff  libsystem_blocks.dylib  ???
0x7fff2008a000 - 0x7fff200bffff  libxpc.dylib  ???
0x7fff200c0000 - 0x7fff200d7fff  libsystem_trace.dylib  ???
0x7fff200d8000 - 0x7fff20175fff  libcorecrypto.dylib  ???
0x7fff20176000 - 0x7fff201a2fff  libsystem_malloc.dylib  ???
0x7fff201a3000 - 0x7fff201e7fff  libdispatch.dylib  ???
0x7fff201e8000 - 0x7fff20221fff  libobjc.A.dylib  ???
```

它在 stderr 上产生详细输出，在 stdout 上产生堆栈跟踪，因此你可能需要重定向 stderr。

**参考文档**

[mac下利用Breakpad的dump文件进行调试](https://cloud.tencent.com/developer/article/1084368)

[在 Linux 程序中使用 breakpad](https://www.jianshu.com/p/eb34b4f21f77)

[breakpad的正确编译和常规用法](https://www.jianshu.com/p/1e15640fae7a)

[Breakpad在mac/ios上的跨平台的调用方式](https://blog.csdn.net/shuangxuyu3220/article/details/111719807)

[[APM笔记]CLI Windows/macOS/Linux使用Breakpad](https://juejin.cn/post/7021756633621463053)
