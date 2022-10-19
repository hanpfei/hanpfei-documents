---
title: 在 Android C/C++ 代码中接入 breakpad
date: 2022-07-08 20:31:49
categories: 音视频开发
tags:
- 音视频开发
---

本文概述在 Android 的 C++ 代码中使用 Breakpad 的方法。

与其它平台接入 Breakpad 的方法类似，主要有如下几步：

1. 编译 breakpad 客户端库。
2. 在代码中集成 breakpad 客户端库。在这一步中配置生成的 minidump 文件的保存目录路径。
3. 生成符号文件。通过 breakpad 提供的 `dump_syms` 工具，为要分析的二进制文件（动态链接库或可执行文件）生成符号文件。
4. 程序崩溃时，生成 minidump 文件。
5. 利用 breakpad 提供的 `minidump_stackwalk` 工具，以前面生成的符号文件和程序崩溃时生成的 minidump 文件为输入，获得符号化的堆栈。
<!--more-->
## 构建 Breadpad

这里说明基于 **WebRTC** / **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 的 GN + ninja 构建系统的构建方法。

通过 Git 下载 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 的源码。然后在 OpenRTCClient 的源码根目录中执行如下命令：

```
OpenRTCClient$ ./build_system/webrtc_build gen android arm64 debug
OpenRTCClient$ ./build_system/webrtc_build build:crash_catch_system android arm64 debug
```

这将为 Android 生成 ARM64 debug 版的客户端静态库文件 **build/android/arm64/debug/obj/third_party/breakpad/libbreakpad_client.a**，同时还会生成用于生成符号文件和将生成的 minidump 转为符号的工具，它们位于 **build/mac/x64/debug**：

```
OpenRTCClient$ ls -alh build/android/arm64/debug/                         
total 24792
drwxr-xr-x  21 zhangsan  staff   672B  7  8 16:28 .
drwxr-xr-x   3 zhangsan  staff    96B  7  8 16:18 ..
-rw-r--r--   1 zhangsan  staff    45K  7  8 16:26 .ninja_deps
-rw-r--r--   1 zhangsan  staff    15K  7  8 16:28 .ninja_log
-rw-r--r--   1 zhangsan  staff   939B  7  8 16:18 args.gn
-rw-r--r--   1 zhangsan  staff   1.9M  7  8 16:20 build.ninja
-rw-r--r--   1 zhangsan  staff    49K  7  8 16:20 build.ninja.d
-rw-r--r--   1 zhangsan  staff   777B  7  8 16:19 build_vars.json
drwx------   9 zhangsan  staff   288B  7  8 16:28 clang_x64
-rwxr-xr-x   1 zhangsan  staff    55K  7  8 16:24 core-2-minidump
lrwxr-xr-x   1 zhangsan  staff    19B  7  8 16:25 dump_syms -> clang_x64/dump_syms
drwxr-xr-x   4 zhangsan  staff   128B  7  8 16:24 exe.unstripped
drwx------  12 zhangsan  staff   384B  7  8 16:19 gen
drwx------   5 zhangsan  staff   160B  7  8 16:20 gen.runtime
lrwxr-xr-x   1 zhangsan  staff    29B  7  8 16:27 microdump_stackwalk -> clang_x64/microdump_stackwalk
-rwxr-xr-x   1 zhangsan  staff    29K  7  8 16:24 minidump-2-core
lrwxr-xr-x   1 zhangsan  staff    23B  7  8 16:28 minidump_dump -> clang_x64/minidump_dump
lrwxr-xr-x   1 zhangsan  staff    28B  7  8 16:27 minidump_stackwalk -> clang_x64/minidump_stackwalk
drwx------  29 zhangsan  staff   928B  7  8 16:19 obj
lrwxr-xr-x   1 zhangsan  staff    19B  7  8 16:26 symupload -> clang_x64/symupload
-rw-r--r--   1 zhangsan  staff   9.1M  7  8 16:20 toolchain.ninja
```

只能在 Linux 操作系统下为 Android 生成 `dump_syms` 和 `minidump_stackwalk` 这样的工具，因而尽管可以 Mac 平台下为 Android 生成 breakpad 客户端静态库文件，但上面的命令的成功完整执行需要在 Linux 平台下进行。

## 把 Breakpad 客户端库集成进你的程序

首先，配置构建过程把 **libbreakpad_client.a** 的目录地址添加到链接库文件的搜索路径，同时为链接添加 **breakpad_client** 库，把 **libbreakpad_client.a** 链接进你的二进制文件，然后设置 include 路径包含 breakpad 源码树中的 **src** 目录。

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
//  auto thread = std::make_unique<std::thread>([&]() {
      char dump_path[512] = { 0 };
      snprintf(dump_path, sizeof(dump_path), "%s/%s.dmp", dump_dir, minidump_id);
//      RTC_LOG(INFO) << "Minidump file path: " << dump_path;
      chmod(dump_path, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
//  });
//
//  thread->join();
  return succeeded;
}

static void crashfunc() {
  volatile int* a = (int*)(NULL);
  *a = 1;
}

extern "C" jint JNIEXPORT JNICALL JNI_OnLoad(JavaVM* jvm, void* reserved) {
  jint ret = InitGlobalJniVariables(jvm);
  RTC_DCHECK_GE(ret, 0);
  if (ret < 0)
    return -1;

   . . . . . .
  open_rtc::InstallCrashHandler("/sdcard/Android/data/com.example.cpp/cache", nullptr, minidumpCallback, nullptr);
  crashfunc();
  return ret;
}

extern "C" void JNIEXPORT JNICALL JNI_OnUnLoad(JavaVM* jvm, void* reserved) {
  open_rtc::UnInstallCrashHandler();
}
```

编译并运行这个示例程序，应该会在手机的 `/sdcard/Android/data/com.example.cpp/cache` 目录下生成一个 minidump 文件，而且它应该会在退出之前打印出 minidump 文件的目录路径和 id，id 即为 minidump 文件的文件名，实际的 minidump 文件带有文件扩展名 ".dmp"。

也可以直接使用 Google breakpad 提供的 `ExceptionHandler` 类来接入 breakpad。此时首先包含头文件：

```
#include "client/linux/handler/exception_handler.h"
```

然后创建 `ExceptionHandler` 类对象。`ExceptionHandler` 对象的整个生命周期中异常处理都是激活的，因此应该在程序启动过程中，尽可能早实例化它，并使其尽可能接近关闭状态一直保持激活。要做任何有用的事情，`ExceptionHandler` 构造函数都需要一个可以写入 minidump 的路径，以及一个回调函数来接收有关已写入的 minidump 的信息：

```
#include <stdlib.h>

#include <unistd.h>

#include "src/client/linux/handler/exception_handler.h"

static bool DumpCallback(const google_breakpad::MinidumpDescriptor &descriptor,
    void *context, bool success) {
//  auto thread = std::make_unique<std::thread>([&]() {
    if (!success) {
      static const char msg[] = "Failed to write minidump\n";
      // RTC_LOG(INFO) << msg;
      return false;
    } else {
//        RTC_LOG(INFO) << "Minidump file path: " << descriptor.path();
        chmod(descriptor.path(), S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
    }
//  });
//
//  thread->join();
  return succeeded;
}

static void DoSomethingWhichCrashes() {
  int local_var = 1;
  *reinterpret_cast<volatile char*>(NULL) = 1;
}

int main(int argc, const char *argv[]) {
  google_breakpad::MinidumpDescriptor minidump("/sdcard/Android/data/com.example.cpp/cache");
  google_breakpad::ExceptionHandler breakpad(minidump, NULL, DumpCallback, NULL,
      true, -1);
  DoSomethingWhichCrashes();
  return 0;
}
```

编译并运行这个程序与前面那个程序的运行结果相同。

Android 接入 breakpad 客户端时，需要注意传入的 minidump 文件目录，需要应用程序对其有写权限。此外，为了使生成的 minidump 文件能被成功地从 Android 设备中 pull 出来，在回调函数中修改了 minidump 文件的权限。

另外，在发生崩溃，回调被调用时，在回调中不能通过 JNI 调用 Java 代码。对于这个问题，StackOverflow 上有讨论，[Unable to make JNI call from c++ to java in android lollipop using jni](https://stackoverflow.com/questions/27223005/unable-to-make-jni-call-from-c-to-java-in-android-lollipop-using-jni)，这是一个已知问题，Android 的 ART 为信号处理使用了备用的栈导致了这个问题。如果想在回调中通过 JNI 调用 Java 代码，需要开专门的线程来执行。

Android 中各种不同的本地层崩溃捕获方案都采用了相同的机制，具体来说是覆盖默认的信号处理器，来获得关于崩溃的详细信息。不同的本地层崩溃捕获方案可能会相互干扰。同一个应用中最好只集成一个崩溃捕获方案。

## 为程序生成符号

如上面的说明，通过 **[OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)** 构建 breakpad 会一并生成各种工具。如果集成了 breakpad 的动态链接库是由 Android Studio 编译的，则相应的带符号动态链接库位于 `build/intermediates/cmake/debug/obj/arm64-v8a`。通过上面生成的 `dump_syms` 来生成符号文件。这个过程相对于其它平台没有什么特别的地方，与 Linux 的基本相同。

同样，生成的符号文件要按特定的目录结构放置，如 [在 Linux 程序中使用 breakpad](https://www.jianshu.com/p/eb34b4f21f77)。

## 处理 minidump 生成栈追踪

Android 程序发生了崩溃之后，从 Android 设备获取 minidump 文件。随后通过 `minidump_stackwalk` 生成栈追踪的方法也与 Linux 的相同，如 [在 Linux 程序中使用 breakpad](https://www.jianshu.com/p/eb34b4f21f77)。

它在 stderr 上产生详细输出，在 stdout 上产生堆栈跟踪，因此你可能需要重定向 stderr。

**参考文档**

[在 Linux 程序中使用 breakpad](https://www.jianshu.com/p/eb34b4f21f77)
