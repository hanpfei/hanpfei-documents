---
title: Android 模拟器下载、编译及调试
date: 2017-09-11 19:05:49
categories: Android开发
tags:
- Android开发
---

# Android 模拟器源码下载

Android 模拟器源码的下载与 Android AOSP 源码库的下载过程类似，可以参考 Google 官方提供的 [Android 源码下载文档](https://source.android.com/source/downloading?hl=zh-cn) 来了解这个过程。
<!--more-->
不同的地方在于，下载 Android 源码，在初始化 repo 客户端，初始化对某个分支的下载时，通过如下的命令指定该 Android 分支：
```
$ repo init -u https://android.googlesource.com/platform/manifest -b android-4.0.1_r1
```

而在下载模拟器源码时，则需要指定一个模拟器的分支。在 https://android.googlesource.com/platform/manifest/+refs 可以看到所有可以指定的分支，包括 Android 分支和模拟器分支，其中模拟器分支主要有如下这些：

```
emu-1.4-release
emu-1.5-release
emu-2.0-release
emu-2.2-release
emu-2.3-release
emu-2.4-arc
emu-2.4-release
emu-2.5-release
emu-master-dev
```

在初始化时，需要通过如下命令初始化对模拟器的下载，比如要下载最新的 2.5 版的 Release 版：
```
$ repo init -u https://android.googlesource.com/platform/manifest -b emu-2.5-release
```

后面同样通过 `repo sync` 命令下载整个源码树。

可以将模拟器源码分支理解为特殊的 Android 源码分支。

# Android 模拟器编译
得到了 Android 模拟器的源码之后，进入下面的文件夹：
```
$ cd external/qemu/android/
```

执行如下命令编译源码：
```
./rebuild.sh --no-tests
```
其中的 `--no-tests` 告诉编译系统，编译完成之后不要执行测试程序，以节省时间，提高效率。

编译完成之后，产生的模拟器可执行文件及库文件都位于 `external/qemu/objs/` 目录下：
```
~/emu-2.4-release/external/qemu/android$ ../objs/
~/emu-2.4-release/external/qemu/objs$ ls
android_emu64_unittests           emulator64-mips
android_emu_metrics64_unittests   emulator64_simg2img
bin64                             emulator64_test_crasher
build                             emulator64-x86
emugl64_common_host_unittests     emulator-check
emulator                          lib
emulator64-arm                    lib64
emulator64_crashreport_unittests  lib64GLcommon_unittests
emulator64-crash-service          lib64OpenglRender_unittests
emulator64_img2simg               qemu
emulator64_libui_unittests        resources
emulator64_make_ext4fs
```

后面就可以像执行 SDK 中的模拟器那样，执行我们编译的模拟器了：
```
~/emu-2.4-release/external/qemu/objs$ ./emulator -avd Nexus_5_API_21_arm
```

# Android 模拟器调试
要想调试 Android 模拟器，就需要生成带有调试符号等信息的可执行文件和库。这需要对我们前面执行的编译脚本程序 `rebuild.sh` 做一点微小的修改，在这个文件中会调用 `android/configure.sh` 程序来多编译过程做配置：
```
run android/configure.sh --out-dir=$OUT_DIR "$@" ||
    panic "Configuration error, please run ./android/configure.sh to see why."
```

默认情况下，这个配置程序生成的配置文件，指导编译过程生成不含调试符号信息的可执行文件和库。但可以为 `android/configure.sh` 程序的执行加上 `--symbols` 以生成带有调试符号信息的可执行文件和库。

`rebuild.sh` 修改之后，大概就像下面这样：
```
run android/configure.sh --symbols --out-dir=$OUT_DIR "$@" ||
    panic "Configuration error, please run ./android/configure.sh to see why."
```

修改之后，重新进入 `external/qemu/android/` 目录下并执行 `rebuild.sh`。

这次将产生带有调试符号信息的可执行文件和库文件，这些文件位于 `external/qemu/objs/build/debug_info` 目录下：
```
~/emu-2.4-release/external/qemu/objs/build/debug_info$ ls
android_emu64_unittests           emulator64_img2simg         emulator-check
android_emu_metrics64_unittests   emulator64_libui_unittests  lib64
emugl64_common_host_unittests     emulator64_make_ext4fs      lib64GLcommon_unittests
emulator                          emulator64-mips             lib64OpenglRender_unittests
emulator64-arm                    emulator64_simg2img         qemu
emulator64_crashreport_unittests  emulator64_test_crasher
emulator64-crash-service          emulator64-x86
```

原来不带调试符号信息的文件依然位于 `external/qemu/objs/` 目录下。

然后就可以通过 GDB 来调试带符号信息的可执行文件和库了。进入 `external/qemu/objs/build/debug_info` 目录下，执行如下命令：
```
~/emu-2.4-release/external/qemu/objs/build/debug_info$ gdb ./emulator
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from emulator...done.
```

该命令用于加载可执行文件。随后，在 GDB 的调试会话中，为可执行文件设置命令行参数，并设置端点：
```
(gdb) set args -avd Nexus_5_API_21_arm
(gdb) break Thread_pthread.cpp:66
(gdb) break emug::RenderThread::main
```

需要注意的是，为一个类函数设置端点时，需要带上它的命名空间。

然后启动可执行程序：
```
(gdb) run
```

随后在程序执行到我们加端点的位置时，程序将被断下来。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.