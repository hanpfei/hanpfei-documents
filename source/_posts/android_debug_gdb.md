---
title: 使用 GDB 调试 Android 应用
date: 2017-09-12 19:05:49
categories: Android开发
tags:
- Android开发
---

GNU 工程调试器（GDB）是一个常用的 Unix 调试器。本文详述使用 `gdb` 调试 Android 应用和进程的方法。
<!--more-->
# 调试运行中的应用或进程

`gdbclient` 是源码库中的一个 shell 脚本调试工具，它位于 `android-7.1.1_r22/development/scripts/gdbclient`。该脚本将根据 Android 源码库的根目录，设置端口转发，在设备上启动适当的 `gdbserver`，在主机上启动适当的 `gdb`，配置 `gdb` 查找符号，并将 `gdb` 连接到远程的 `gdbserver`。

在执行 `gdbclient` 首先需要设置 `ANDROID_BUILD_TOP` 环境变量，这个环境变量可以手动设置，如：
```
/media/data/Androids/android-7.1.1_r22$ export ANDROID_BUILD_TOP=/media/data/Androids/android-7.1.1_r22
```

也可以通过如下命令设置：
```
/media/data/Androids/android-7.1.1_r22$ source build/envsetup.sh 
including device/asus/fugu/vendorsetup.sh
including device/generic/mini-emulator-arm64/vendorsetup.sh
including device/generic/mini-emulator-armv7-a-neon/vendorsetup.sh
including device/generic/mini-emulator-mips64/vendorsetup.sh
including device/generic/mini-emulator-mips/vendorsetup.sh
including device/generic/mini-emulator-x86_64/vendorsetup.sh
including device/generic/mini-emulator-x86/vendorsetup.sh
including device/google/dragon/vendorsetup.sh
including device/google/marlin/vendorsetup.sh
including device/htc/flounder/vendorsetup.sh
including device/huawei/angler/vendorsetup.sh
including device/lge/bullhead/vendorsetup.sh
including device/linaro/hikey/vendorsetup.sh
including device/moto/shamu/vendorsetup.sh
including sdk/bash_completion/adb.bash
hanpfei0306@ThundeRobot:/media/data/Androids/android-7.1.1_r22$ lunch 18

============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=7.1.1
TARGET_PRODUCT=aosp_sailfish
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm64
TARGET_ARCH_VARIANT=armv8-a
TARGET_CPU_VARIANT=generic
TARGET_2ND_ARCH=arm
TARGET_2ND_ARCH_VARIANT=armv7-a-neon
TARGET_2ND_CPU_VARIANT=krait
HOST_ARCH=x86_64
HOST_2ND_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-4.4.0-89-generic-x86_64-with-Ubuntu-16.04-xenial
HOST_CROSS_OS=windows
HOST_CROSS_ARCH=x86
HOST_CROSS_2ND_ARCH=x86_64
HOST_BUILD_TYPE=release
BUILD_ID=NMF26X
OUT_DIR=out
============================================
```
也就是 Android 源码库编译前配置。

如果没有设置 `ANDROID_BUILD_TOP` 环境变量的话，在执行 `gdbclient` 时将报出如下的错误：
```
/media/data/Androids/android-7.1.1_r22$ development/scripts/gdbclient
$ANDROID_BUILD_TOP is not set. Source build/envsetup.sh.
```

有了前面的那些配置，即可使用 `gdbclient` 调试 Android 应用程序了。要连接一个已经在运行的应用或本地层守护进程，则以 PID 作为参数执行 `gdbclient`。比如，要调试 PID 为 1234 的进程，则运行：
```
$ gdbclient 1234
```

它会为我们准备一切。

# 调试本地进程启动
要调试进程的启动，则使用 `gdbserver` 或 `gdbserver64` （64 位进程）。比如：
```
$ adb shell gdbserver64 :5039 /system/bin/screenrecord
```

示例输出如下：
```
Process /system/bin/screenrecord created; pid = 12571
Listening on port 5039
```

接着，从 `gdbserver` 的输出中得到应用程序的 PID，并在另一个终端窗口中使用如下命令：
```
$ gdbclient 12571
```

最后，在 gdb 提示符下键入 `continue`。

使用的 `gdbserver` 与实际运行的应用程序格式不匹配时，在执行 `gdbclient` 时将报出如下的错误：
```
$ gdbclient 12484
including device/asus/fugu/vendorsetup.sh
including device/generic/mini-emulator-arm64/vendorsetup.sh
including device/generic/mini-emulator-armv7-a-neon/vendorsetup.sh
including device/generic/mini-emulator-mips64/vendorsetup.sh
including device/generic/mini-emulator-mips/vendorsetup.sh
including device/generic/mini-emulator-x86_64/vendorsetup.sh
including device/generic/mini-emulator-x86/vendorsetup.sh
including device/google/dragon/vendorsetup.sh
including device/google/marlin/vendorsetup.sh
including device/htc/flounder/vendorsetup.sh
including device/huawei/angler/vendorsetup.sh
including device/lge/bullhead/vendorsetup.sh
including device/linaro/hikey/vendorsetup.sh
including device/moto/shamu/vendorsetup.sh
including sdk/bash_completion/adb.bash

It looks like gdbserver is already attached to 12484 (process is traced), trying to connect to it using local port=5039
GNU gdb (GDB) 7.11
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
Reading symbols from out/target/product/sailfish/symbols/system/bin/screenrecord...done.
warning: Selected architecture aarch64 is not compatible with reported target architecture arm
out/target/product/sailfish/gdbclient.cmds:4: Error in sourced command file:
Reply contains invalid hex digit 59
(gdb) break main
Breakpoint 1 at 0x5820: file frameworks/av/cmds/screenrecord/screenrecord.cpp, line 900.
(gdb) r
Starting program: /media/data/Androids/android-7.1.1_r22/out/target/product/sailfish/symbols/system/bin/screenrecord 
/usr/local/google/buildbot/src/android/master-ndk/toolchain/gdb/gdb-7.11/gdb/regcache.c:1056: internal-error: regcache_raw_supply: Assertion `regnum >= 0 && regnum < regcache->descr->nr_raw_registers' failed.
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Quit this debugging session? (y or n) y

This is a bug, please report it.  For instructions, see:
<http://www.gnu.org/software/gdb/bugs/>.

/usr/local/google/buildbot/src/android/master-ndk/toolchain/gdb/gdb-7.11/gdb/regcache.c:1056: internal-error: regcache_raw_supply: Assertion `regnum >= 0 && regnum < regcache->descr->nr_raw_registers' failed.
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Create a core file of GDB? (y or n) y
/media/data/Androids/android-7.1.1_r22/prebuilts/gdb/linux-x86/bin/gdb: 行 3: 12882 已放弃               (核心已转储) PYTHONHOME="$GDBDIR/.." "$GDBDIR/gdb-orig" "$@"
hanpfei0306@ThundeRobot:/media/data/Androids/android-7.1.1_r22$ /bin/bash: /media/data/Androids/android-7.1.1_r22/out/target/product/sailfish/symbols/system/bin/screenrecord: cannot execute binary file: 可执行文件格式错误
/bin/bash: /media/data/Androids/android-7.1.1_r22/out/target/product/sailfish/symbols/system/bin/screenrecord: 成功
```

即我们在 `gdb` 的提示符下输入 `continue` 执行应用程序之后，报出了 `可执行文件格式错误`。

参考文档：

[Using GDB](https://source.android.com/devices/tech/debug/gdb)

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
