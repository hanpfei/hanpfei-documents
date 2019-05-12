---
title: Android QEMU 模拟器移植 - 编译
date: 2018-12-23 19:05:49
categories: 虚拟化
tags:
- 虚拟化
---

Android QEMU 模拟器是我们日常 Android 开发中一个非常有力的工具，也是一套有很多值得玩的地方的系统，它还是开源的，主要由 Google Android 在维护。（Android QEMU 模拟器的源码下载、编译及调试方法可参考[Android 模拟器下载、编译及调试](https://www.wolfcstech.com/2017/09/11/android_emulator_dev/)）目前大多数开发者所用的开发机器都是 X86 架构的，因而 Google 官方对于Android QEMU 模拟器的编译和运行，也假设都是在 X86 架构平台进行的。这种假设在整个 Android QEMU 模拟器的编译和运行方面都有所体现，如编译配置仅提供了 X86 的相关选项，编译 Android QEMU 模拟器所需的一些预编译第三方库仅提供了X86 或 X86_64 的版本，编译脚本里仅有对 X86 或 X86_64 主机平台的行为定义，在代码中运行时一些关键的操作上仅有X86 或 X86_64 主机平台的定义等。
<!--more-->
Android QEMU 模拟器的玩法很多，某些场景下会有将 Android QEMU 模拟器移植到不同的主机 CPU 架构上的需要，特别是 ARM64 平台。当前已经出现了不少 ARM64 的服务器硬件平台，可以运行完整的 Ubuntu Linux 等系统。大多数 Android 终端采用 ARM 低功耗处理器，Android 系统运行于 ARM 平台，因而大多数 Android 应用对于 ARM 平台的支持会更好一点。ARM 目标 Android 系统运行于我们日常的 X86 开发机的模拟器上时，性能都非常差。在 ARM64 平台上运行 ARM 版模拟器，同时开启 KVM，则可以获得 Android QEMU 非常好的运行性能。（KVM/ARM 虚拟化技术的更多信息，可参考[ KVM/ARM: AN OPEN-SOURCE ARM VIRTUALIZATION SYSTEM]([http://systems.cs.columbia.edu/projects/kvm-arm/](http://systems.cs.columbia.edu/projects/kvm-arm/)
)）ARM64 服务器 + ARM 版模拟器 + KVM 也就顺利成章地成为许多有趣的场景的技术基础。

本文以 ARM64 平台为例介绍把 Android QEMU 模拟器移植到 ARM64 Ubuntu Linux 平台的过程，其中 Code base 为 2019 年 3 月拉取，qemu 所用的 branch 为 emu-2.3-release。移植过程主要分为几个步骤：
 - 配置编译
    * 配置脚本成功执行
    * 预编译库的编译移植。在 Android QEMU 模拟器的源码库中有编译安装这些库的脚本，但可能需要有针对性地对脚本做些移植。具体需要移植的库主要包括如下这些：
        -  Zlib
        -  Libpng
        -  Libxml2
        -  LibCURL
        -  LibANGLEtranslation
        -  Protobuf
        -  Breakpad
        -  Qt
        -  e2fsprogs
        -  ffmpeg
        -  x264
    * 构建配置文件的移植修改，主要指 mk 文件
 - 运行时问题。代码里面有一些与特定硬件平台相关的代码，需要专门针对 ARM 平台做一些适配
 - nowindow 及功能扩充。对于某些需要运行模拟器的机器，显示模拟器的窗口可能不太方便，因而需要将模拟器中窗口显示的部分拆除掉。功能扩充则指的是，某些运行模拟器的机器，渲染能力比较弱，则可以将模拟器中虚拟 Android 系统相关的逻辑和图形渲染相关的逻辑分开在不同的机器上运行。

解决了上面的重重困难之后，Android QEMU 模拟器应用程序本身总算是具备了运行的条件。但要真正让 Android QEMU 模拟器运行起来，还缺少一项关键的东西，那就是 AVD。有了 AVD，Android QEMU 模拟器应用程序才能知道要虚拟什么样的 Android 系统。AVD 的创建，就像我们平常在 X86 的开发机器上所做的那样，需要通过 Android SDK 来完成。除了 AVD，我们日常 Android 开发工具中必不可少的一些工具，如 adb 等也需要能够运行起来，这也离不开 Android SDK。但 Android SDK 不能用我们平常用的那个 X86 的版本，而需要用专门为 ARM 平台移植的版本。在上面的移植步骤之后增加一个关键的步骤：
  - Android SDK 的移植

本文主要关注 Android QEMU 模拟器应用程序本身的配置编译问题。

# 1. 编译问题

Android QEMU 模拟器的源码中提供了构建脚本，可以用来构建生成二进制可执行文件。具体而言，是 `qemu/android/rebuild.sh`。切换进 qemu 源码根目录，像下面这样执行 `qemu/android/rebuild.sh` 可以进行编译：
```
qemu$ android/rebuild.sh --no-tests
Configuring build.
Building sources.
Ignoring unit tests suite.
Done. !!
```

岁月静好，一切都是安安静静的。执行 `rebuild.sh` 脚本，并加上 `--help` 参数，可以看到这个脚本更详细的用法：
```
qemu$ android/rebuild.sh --help
Configuring build.

Usage: rebuild.sh [options]
Options: [defaults in brackets after descriptions]
Standard options:
  --help                      Print this message
  --cc=PATH                   Specify C compiler [gcc]
  --cxx=PATH                  Specify C++ compiler [g++]
  --no-strip                  Do not strip emulator executables.
  --strip                     Strip emulator executables (default).
  --symbols                   Generating Breakpad symbol files.
  --no-symbols                Do not generate Breakpad symbol files (default).
  --crash-[staging,prod]      Send crashes to specific server (no crash reporting by default).
  --gles=dgl                  Build the OpenGLES to Desktop OpenGL Translator (default)
  --gles=angle                Build the OpenGLES to ANGLE wrapper
  --aosp-prebuilts-dir=<path> Use specific prebuilt toolchain root directory [emulator/prebuilts]
  --out-dir=<path>            Use specific output directory [objs/]
  --mingw                     Build Windows executable on Linux
  --verbose                   Verbose configuration
  --debug                     Build debug version of the emulator
  --sanitizer=[...]           Build with LLVM sanitizer (sanitizer=[address, thread])
  --no-pcbios                 Disable copying of PC Bios files
  --no-tests                  Don't run unit test suite
  --benchmarks                Build benchmark programs.
  --lto                       Force link-time optimization.

ERROR: Configuration error, please run ./android/configure.sh to see why.
```

在执行 `rebuild.sh` 脚本时，加上 `--no-tests` 参数，可以在编译成功之后不运行单元测试，以加快编译完成时间，加上 `--verbose` 参数可以输出更详细的编译过程信息。

`rebuild.sh` 脚本的功能可以简单地理解为，清理之前编译遗留下来的配置文件和中间文件，执行 `qemu/android/configure.sh` 脚本完成配置，然后执行 `make` 完成编译。了解了这一点对于我们分析配置和编译过程中的错误非常有意义——当我们为出现的某个错误修改了代码时，不用每次都执行 `rebuild.sh` 脚本完整的进行一次构建来验证。

## 1.1 配置

如我们上面提到的，配置在执行 `rebuild.sh` 脚本时，它调用 `qemu/android/configure.sh` 脚本完成。`qemu/android/configure.sh` 脚本也可以单独执行进行配置，如下面这样：
```
qemu$ android/configure.sh --no-tests --verbose
```

在 ARM 64 Ubuntu 平台上执行 `qemu/android/configure.sh` 脚本时，遇到的第一个问题是如下的报错：
```
qemu$ android/configure.sh --no-tests --verbose
ERROR: aarch64 builds are not supported!
```

这主要是由 `qemu/android/configure.sh` 脚本中设置 BUILD_ARCH 的逻辑引起的，如下面这段代码（在 41 行左右）：
```
BUILD_ARCH=$(uname -m)
case "$BUILD_ARCH" in
    i?86) BUILD_ARCH=x86
    ;;
    x86_64|amd64) BUILD_ARCH=x86_64
    ;;
    *) panic "$BUILD_ARCH builds are not supported!"
    ;;
esac
```

需要在上面这段代码中加入对与 ARM 64 平台的支持。具体怎么加？上面的代码其实已经给出了提示，ARM 64 的 arch 值可以通过 `uname -m` 命令获得：
```
qemu$ uname -m
aarch64
```

加完之后，脚本中配置 BUILD_ARCH 的代码将像下面这样：
```
BUILD_ARCH=$(uname -m)
case "$BUILD_ARCH" in
    i?86) BUILD_ARCH=x86
    ;;
    x86_64|amd64) BUILD_ARCH=x86_64
    ;;
    aarch64) BUILD_ARCH=aarch64
    ;;
    *) panic "$BUILD_ARCH builds are not supported!"
    ;;
esac
```

再次执行 `qemu/android/configure.sh` 脚本，遇到如下的错误：
```
qemu$ android/configure.sh --no-tests --verbose
Auto-config: --gles=dgl
Auto-config: --out-dir=objs
Prebuilt   : CCACHE=/usr/bin/ccache
ERROR: Cannot generate SDK toolchain!
```

这个错误信息在 `qemu/android/configure.sh` 脚本中抛出，但原因是在执行 `qemu/android/scripts/gen-android-sdk-toolchain.sh` 脚本时出错导致，错误的原因有多个，一是在 `qemu/android/scripts/utils/common.shi` 脚本中定义的 `get_build_arch()` 函数有 bug 所致，该函数的定义如下：
```
# Return the build machine's CPU architecture.
# Valid return values are:
#     x86
#     x86_64
get_build_arch () {
    local TEST
    if [ -z "$_SHU_BUILD_ARCH" ]; then
        case $(get_build_os) in
            linux|darwin)
                _SHU_BUILD_ARCH=$(uname -m)
                # Kernel bitness might not match user space, so test
                # the bitness of our shell to know what the user is
                # really running.
                TEST=$(/usr/bin/file -L $SHELL 2>/dev/null | grep 'x86[_-]64')
                if [ "$TEST" ]; then
                    _SHU_BUILD_ARCH=x86_64
                fi
                ;;
            windows|cygwin)
                case $PROCESSOR_ARCHITECTURE in
                    ADM64)
                        _SHU_BUILD_ARCH=x86_64
                        ;;
                    *)
                        _SHU_BUILD_ARCH=x86
                        ;;
                esac
                ;;
            *)
                _SHU_BUILD_ARCH=$(uname -p)
                ;;
        esac
    fi
    echo "$_SHU_BUILD_ARCH"
}
```

其中 bug 主要出在 `TEST=$(/usr/bin/file -L $SHELL 2>/dev/null | grep 'x86[_-]64')`。这一行想通过 file 命令获取 shell 程序的文件属性中 CPU 架构有关的信息来获取主机的 CPU 架构信息，但当 `grep 'x86[_-]64'` 失败时，会导致整个脚本以非零错误值退出。另外这个函数本身没有考虑对非 x86 架构的支持。修正这个问题：
```
# Return the build machine's CPU architecture.
# Valid return values are:
#     x86
#     x86_64
get_build_arch () {
    local TEST
    if [ -z "$_SHU_BUILD_ARCH" ]; then
        case $(get_build_os) in
            linux|darwin)
                _SHU_BUILD_ARCH=$(uname -m)
                # Kernel bitness might not match user space, so test
                # the bitness of our shell to know what the user is
                # really running.
                TEST=$(/usr/bin/file -L $SHELL 2>/dev/null | grep 'x86[_-]64' | cat)
                if [ "$TEST" ]; then
                    _SHU_BUILD_ARCH=x86_64
                fi
                if [ -z "$_SHU_BUILD_ARCH" ]; then
                    _SHU_BUILD_ARCH=$(uname -p)
                fi
                ;;
            windows|cygwin)
                case $PROCESSOR_ARCHITECTURE in
                    ADM64)
                        _SHU_BUILD_ARCH=x86_64
                        ;;
                    *)
                        _SHU_BUILD_ARCH=x86
                        ;;
                esac
                ;;
            *)
                _SHU_BUILD_ARCH=$(uname -p)
                ;;
        esac
    fi
    echo "$_SHU_BUILD_ARCH"
}
```

修改完问题依旧:
```
qemu$ android/configure.sh --no-tests  --verbose
Auto-config: --gles=dgl
Auto-config: --out-dir=objs
Prebuilt   : CCACHE=/usr/bin/ccache
ERROR: Host system 'linux-aarch64' is not supported by this script!
ERROR: Cannot generate SDK toolchain!
```

在 `qemu/android/scripts/gen-android-sdk-toolchain.sh` 中对于 `GNU_CONFIG_HOST` 和 `EXTRA_CFLAGS` 的配置也需要有对 ARM 64 平台的支持：
```
    case $CURRENT_HOST in
        linux-aarch64)
            GNU_CONFIG_HOST=aarch64-linux
            ;;
        linux-x86_64)
            GNU_CONFIG_HOST=x86_64-linux
            ;;
        linux-x86)
            GNU_CONFIG_HOST=i686-linux
            ;;
        windows-x86)
            GNU_CONFIG_HOST=i686-w64-mingw32
            ;;
        windows-x86_64)
            GNU_CONFIG_HOST=x86_64-w64-mingw32
            ;;
        darwin-*)
            # Use host compiler.
            GNU_CONFIG_HOST=
            ;;
        *)
            panic "Host system '$CURRENT_HOST' is not supported by this script!"
            ;;
    esac
 . . . . . .
    case $CURRENT_HOST in
        darwin-x86_64)

            common_FLAGS="-target x86_64-apple-darwin12.0.0"
            var_append common_FLAGS " -isysroot $OSX_SDK_ROOT"
            var_append common_FLAGS " -mmacosx-version-min=$OSX_DEPLOYMENT_TARGET"
            var_append common_FLAGS " -DMACOSX_DEPLOYMENT_TARGET=$OSX_DEPLOYMENT_TARGET"
            EXTRA_CFLAGS="$common_FLAGS"
            EXTRA_CXXFLAGS="$common_FLAGS"
            if [ "$OPT_CXX11" ]; then
                var_append EXTRA_CXXFLAGS "-stdlib=libc++"
            fi
            EXTRA_LDFLAGS="-syslibroot $OSX_SDK_ROOT"
            ;;
        darwin-x86)
            panic "Host system '$CURRENT_HOST' is not supported by this script!"
            ;;
        *-x86)
            EXTRA_CFLAGS="-m32"
            EXTRA_CXXFLAGS="-m32"
            ;;
        *-x86_64)
            EXTRA_CFLAGS="-m64"
            EXTRA_CXXFLAGS="-m64"
            ;;
        *-aarch64)
            EXTRA_CFLAGS=""
            EXTRA_CXXFLAGS="-std=c++11"
            EXTRA_LDFLAGS=""
            ;;
        *)
            panic "Host system '$CURRENT_HOST' is not supported by this script!"
            ;;
    esac
```

再次执行 `qemu/android/configure.sh` 脚本，遇到如下的错误：
```
$ android/configure.sh --no-tests  --verbose
Auto-config: --gles=dgl
Auto-config: --out-dir=objs
Prebuilt   : CCACHE=/usr/bin/ccache
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-29561-test.o -c  /tmp/android-29561-test.c
your C compiler doesn't seem to work: objs/build/toolchain/aarch64-linux-gcc
ccache: error: execv of emulator/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8/bin/x86_64-linux-gcc failed: Exec format error
```

这个是在 `android/configure.sh` 脚本中（大约115 行），调用由 `qemu/android/scripts/gen-android-sdk-toolchain.sh` 生成的 gcc 编译器包装器执行编译测试时发生的错误。`qemu/android/scripts/gen-android-sdk-toolchain.sh` 脚本会在 `objs/build/toolchain/` 目录下生成构建工具链的包装器脚本：
```
qemu$ ls objs/build/toolchain/
aarch64-linux-ar  aarch64-linux-c++  aarch64-linux-g++  aarch64-linux-ld  aarch64-linux-objcopy  aarch64-linux-ranlib   aarch64-linux-strip
aarch64-linux-as  aarch64-linux-cpp  aarch64-linux-gcc  aarch64-linux-nm  aarch64-linux-objdump  aarch64-linux-strings  pkgconfig
```

这些脚本的内容类似与下面这样：
```
qemu$ cat objs/build/toolchain/aarch64-linux-gcc 
#!/bin/sh
# Auto-generated by gen-android-sdk-toolchain.sh, DO NOT EDIT!!

# Environment setup


# Tool invokation.
/usr/bin/ccache emulator/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8/bin/x86_64-linux-gcc   "$@"
```

即 `qemu/android/scripts/gen-android-sdk-toolchain.sh` 脚本根据系统配置，将构建工具链的路径统一化。对于 ARM 64 平台的编译，上面的脚本内容显然是错误的。但这些内容是怎么来的呢？它们是在 `qemu/android/scripts/gen-android-sdk-toolchain.sh` 脚本通过 prepare_build_for_host -> gen_wrapper_toolchain -> gen_wrapper_program 这一系列函数调用产生的。路径中的一些前缀，在 prepare_build_for_host 函数中，通过调用定义在 `qemu/android/scripts/utils/aosp_dir.shi` 文件中的 `aosp_prebuilt_toolchain_subdir_for` 和 `aosp_prebuilt_toolchain_prefix_for` 函数获得，这两个函数缺乏对于 ARM 64 的支持。

修正这个问题（即修改 `qemu/android/scripts/utils/aosp_dir.shi` 文件中的 `aosp_prebuilt_toolchain_subdir_for` 和 `aosp_prebuilt_toolchain_prefix_for` 函数定义）：
```
# Return the AOSP subdirectory that contains the prebuilt toolchain to be
# used for a given host system.
# $1: System name (e.g. 'linux' or 'linux-x86_64')
aosp_prebuilt_toolchain_subdir_for () {
    case $1 in
        linux-aarch64)
            printf "prebuilts/gcc/linux-aarch64/host/gcc-arm-none-eabi"
            ;;
        linux*)
            printf "prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8"
            ;;
        darwin*)
            printf "prebuilts/clang/darwin-x86/sdk/3.5"
            ;;
        windows*)
            printf "prebuilts/gcc/linux-x86/host/x86_64-w64-mingw32-4.8"
            ;;
    esac
}
. . . . . .
aosp_prebuilt_toolchain_prefix_for () {
    case $1 in
        linux-aarch64)
            printf "arm-none-eabi-"
            ;;
        linux*)
            printf "x86_64-linux-"
            ;;
        darwin*)
            printf ""
            ;;
        windows*)
            printf "x86_64-w64-mingw32-"
            ;;
    esac
}
```

再次执行 `qemu/android/configure.sh` 脚本，遇到如下的错误：
```
Copying Qt prebuilt libraries from emulator/prebuilts/android-emulator-build/qt
Copying e2fsprogs binaries.
Copying emulator/prebuilts/android-emulator-build/common/e2fsprogs/linux-x86_64/sbin/* -> objs/bin64/
Copying toolchain libraries to bundle with the program
  Bundling libstdc++
cp: cannot stat 'emulator/prebuilts/gcc/linux-aarch64/host/gcc-arm-none-eabi/arm-none-eabi/lib/libstdc++.so': No such file or directory
ERROR: Cound't copy emulator/prebuilts/gcc/linux-aarch64/host/gcc-arm-none-eabi/arm-none-eabi/lib/libstdc++.so --> objs/lib/libstdc++/libstdc++.so
```

这个是在拷贝 C++ 标准库是出现的问题。需要修改几个地方：一是 `qemu/android/scripts/utils/aosp_dir.shi` 文件中的 `aosp_prebuilt_toolchain_sysroot_subdir_for` 函数，提供对 ARM 64 的支持，找到正确的文件路径：
```
# Return the AOSP subdirectory corresponding to the toolchain's sysroot.
# $1: System name (e.g. 'linux-x86' or 'linux-x86_64')
aosp_prebuilt_toolchain_sysroot_subdir_for () {
    local PREBUILT_DIR
    PREBUILT_DIR="$(aosp_prebuilt_toolchain_subdir_for "$1")"
    case $1 in
        linux-aarch64)
            printf "${PREBUILT_DIR}/arm-none-eabi"
            ;;
        linux*)
            printf "${PREBUILT_DIR}/x86_64-linux"
            ;;
        darwin*)
            # We use the developer machine's system SDK.
            printf ""
            ;;
        windows*)
            printf "${PREBUILT_DIR}/x86_64-w64-mingw32"
            ;;
    esac
}
```

二是 `qemu/android/configure.sh` 脚本中的 copy_toolchain_lib 函数及该函数调用的地方：
```
#   $3: Name of the library (without the .so suffix).
copy_toolchain_lib () {
    local DESTDIR SRCDIR NAME
    local FROM REALNAME
    local SYMLINK SYMNAME
    DESTDIR="$1"
    SRCDIR="$2"
    NAME="$3.so"

    if [ ! -f "${SRCDIR}/${NAME}" ]; then
        NAME="$3.a"
    fi

    mkdir -p "${DESTDIR}" || panic "Failed to created ${DESTDIR}"
. . . . . .
            for BUNDLED_LIB in libstdc++; do
                log "  Bundling ${BUNDLED_LIB}"

                case "$BUILD_ARCH" in
                    aarch64) copy_toolchain_lib "${OUT_DIR}/lib/libstdc++" "${TOOLCHAIN_SYSROOT}/lib" "${BUNDLED_LIB}"
                    ;;
                    *) copy_toolchain_lib "${OUT_DIR}/lib/libstdc++" "${TOOLCHAIN_SYSROOT}/lib32" "${BUNDLED_LIB}"
                    ;;
                esac
                copy_toolchain_lib "${OUT_DIR}/lib64/libstdc++" "${TOOLCHAIN_SYSROOT}/lib64" "${BUNDLED_LIB}"
            done
```

这是由于没有32 位 ARM 的平台的 C++ 标准库的动态库文件。

再次执行 `qemu/android/configure.sh` 脚本，终于成功完成：
```
qemu_3.1$ android/configure.sh --no-tests --verbose
Auto-config: --gles=dgl
Auto-config: --out-dir=objs
Prebuilt   : CCACHE=/usr/bin/ccache
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test.o -c  /tmp/android-25912-test.c
CC         : compiler check ok (objs/build/toolchain/aarch64-linux-gcc)
CC_VER     : arm-none-eabi-gcc (Ubuntu/Linaro 5.4.0-6ubuntu1~16.04.11) 5.4.0 20160609
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Link      : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test /tmp/android-25912-test.o 
LD         : linker check ok (objs/build/toolchain/aarch64-linux-gcc)
AR         : archiver (objs/build/toolchain/aarch64-linux-ar)
GLES       : Found GPU emulation sources: android/android-emugl
PC Bios    : Probing emulator/prebuilts/qemu-kernel/x86/pc-bios
PC Bios    : Copying bios.bin
PC Bios    : Copying vgabios-cirrus.bin
PC Bios    : Copying bios-256k.bin
PC Bios    : Copying efi-virtio.rom
PC Bios    : Copying kvmvapic.bin
PC Bios    : Copying linuxboot.bin
PC Bios    : Copying linuxboot_dma.bin
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test.o -c  /tmp/android-25912-test.c
CC         : compiler check ok (objs/build/toolchain/aarch64-linux-gcc)
CC_VER     : arm-none-eabi-gcc (Ubuntu/Linaro 5.4.0-6ubuntu1~16.04.11) 5.4.0 20160609
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Link      : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test /tmp/android-25912-test.o 
LD         : linker check ok (objs/build/toolchain/aarch64-linux-gcc)
AR         : archiver (objs/build/toolchain/aarch64-linux-ar)
Zlib prebuilts dir: emulator/prebuilts/android-emulator-build/qemu-android-deps
Libpng prebuilts dir: emulator/prebuilts/android-emulator-build/qemu-android-deps
Libxml2 prebuilts dir: emulator/prebuilts/android-emulator-build/common/libxml2
LibCURL prebuilts dir: emulator/prebuilts/android-emulator-build/curl
LibANGLEtranslation prebuilts dir: emulator/prebuilts/android-emulator-build/common/ANGLE
Protobuf prebuilts dir: emulator/prebuilts/android-emulator-build/protobuf
Breakpad prebuilts dir: emulator/prebuilts/android-emulator-build/common/breakpad
Qt prebuilts dir: emulator/prebuilts/android-emulator-build/qt
e2fsprogs prebuilts dir: emulator/prebuilts/android-emulator-build/common/e2fsprogs
ffmpeg prebuilts dir: emulator/prebuilts/android-emulator-build/common/ffmpeg
x264 prebuilts dir: emulator/prebuilts/android-emulator-build/common/x264
HeaderCheck: <byteswap.h>
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test.o -c   /tmp/android-25912-test.c
HeaderCheck: <byteswap.h> [yes]
HeaderCheck: <machine/bswap.h>
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test.o -c   /tmp/android-25912-test.c
HeaderCheck: <machine/bswap.h> [no]
HeaderCheck: <fnmatch.h>
Object     : objs/build/toolchain/aarch64-linux-gcc -o /tmp/android-25912-test.o -c   /tmp/android-25912-test.c
HeaderCheck: <fnmatch.h> [yes]
Copying Swiftshader prebuilt libraries from emulator/prebuilts/android-emulator-build/common/swiftshader
Copying Mesa prebuilt libraries from emulator/prebuilts/android-emulator-build/mesa
Copying Qt prebuilt libraries from emulator/prebuilts/android-emulator-build/qt
Copying e2fsprogs binaries.
Copying emulator/prebuilts/android-emulator-build/common/e2fsprogs/linux-x86_64/sbin/* -> objs/bin64/
Copying toolchain libraries to bundle with the program
  Bundling libstdc++
Mingw      : No libwinpthread-1.dll available.
Generate   : objs/build/config.make
QEMU2      : android/..  [auto-detect]
Generate   : objs/build/config-host.h
Generate   : objs/build/qemu1-qapi-auto-generated
QEMU2    : Configuring.
QEMU2 Dependencies prebuilts dir: emulator/prebuilts/android-emulator-build/qemu-android-deps
QEMU2    : Version []
Ready to go. Type 'make' to build emulator
```

至此，总算可以执行编译配置。

## 1.2 依赖
`qemu/android/configure.sh` 脚本吐出来的信息包含了构建 Android QEMU 模拟器时所需的主要的库及其预编译库文件的路径。
```
Zlib prebuilts dir: emulator/prebuilts/android-emulator-build/qemu-android-deps
Libpng prebuilts dir: emulator/prebuilts/android-emulator-build/qemu-android-deps
Libxml2 prebuilts dir: emulator/prebuilts/android-emulator-build/common/libxml2
LibCURL prebuilts dir: emulator/prebuilts/android-emulator-build/curl
LibANGLEtranslation prebuilts dir: emulator/prebuilts/android-emulator-build/common/ANGLE
Protobuf prebuilts dir: emulator/prebuilts/android-emulator-build/protobuf
Breakpad prebuilts dir: emulator/prebuilts/android-emulator-build/common/breakpad
Qt prebuilts dir: emulator/prebuilts/android-emulator-build/qt
e2fsprogs prebuilts dir: emulator/prebuilts/android-emulator-build/common/e2fsprogs
ffmpeg prebuilts dir: emulator/prebuilts/android-emulator-build/common/ffmpeg
x264 prebuilts dir: emulator/prebuilts/android-emulator-build/common/x264
```

解决依赖问题主要是为这些库编译出 ARM 64 平台的二进制库文件，并放置在上面的位置。编译上面的这些库，可以通过 `qemu/android/scripts/` 目录下，名为 `build-xxx.sh` 的脚本完成：
```
qemu$ ls android/scripts/
build-ANGLE.sh      build-ffmpeg.sh     build-protobuf.sh           build-wine.sh                 gen-charmap.py             gles3translatorgen    set-cpu-governor.sh
build-breakpad.sh   build-libxml2.sh    build-qemu-android-deps.sh  build-x264.sh                 gen-entries.py             migrate-glue-code.py  tests
build-curl.sh       build-mesa-deps.sh  build-qemu-android.sh       download-sources.sh           generate-android-icons.sh  offset_layout.py      upload-symbols.sh
build-e2fsprogs.sh  build-mesa.sh       build-qt.sh                 gen-android-sdk-toolchain.sh  gen-hw-config.py           package-release.sh    util
```

对于不同库，上面的这些脚本可用性不太一样，有些脚本可以直接为 ARM 64 平台编译出二进制文件，有些则需要做更多的移植修改。这里暂时先不对这些依赖库的构建做更详细的说明。

## 1.3 编译配置

如上面成功执行 `qemu/android/configure.sh` 脚本之后看到的那样，配置完成之后，可以通过执行 `make` 开始编译：
```
qemu$ make
Build: Unsupported host architecture: aarch64    
android/build/core/definitions-init.mk:251: *** Build: Aborting    .  Stop.
```

只是一上来，就报了错，在 `android/build/core/definitions-init.mk` 的 251 行左右：
```
    else # BUILD_HOST_OS_BASE != windows
        UNAME := $(shell uname -m)
        ifneq (,$(findstring 86,$(UNAME)))
            BUILD_HOST_ARCH := x86
            ifneq (,$(shell $(BUILD_HOST_FILE_PROGRAM) -L $(SHELL) | grep 'x86[_-]64'))
                BUILD_HOST_ARCH64 := x86_64
            endif
        endif
        # We should probably should not care at all
        ifneq (,$(findstring Power,$(UNAME)))
            BUILD_HOST_ARCH := ppc
        endif
        ifeq ($(BUILD_HOST_ARCH),)
            $(call -build-info,Unsupported host architecture: $(UNAME))
            $(call -build-error,Aborting)
        endif
    endif # BUILD_HOST_OS_BASE != windows
```

上面这个报错是由于在 `android/build/core/definitions-init.mk` 构建配置文件中缺乏对于 ARM 64 平台的支持。将上面那段代码修改为下面这样：
```
    else # BUILD_HOST_OS_BASE != windows
        UNAME := $(shell uname -m)
        ifneq (,$(findstring 86,$(UNAME)))
            BUILD_HOST_ARCH := x86
            ifneq (,$(shell $(BUILD_HOST_FILE_PROGRAM) -L $(SHELL) | grep 'x86[_-]64'))
                BUILD_HOST_ARCH64 := x86_64
            endif
        endif
        # We should probably should not care at all
        ifneq (,$(findstring Power,$(UNAME)))
            BUILD_HOST_ARCH := ppc
        endif
        ifneq (,$(findstring aarch64,$(UNAME)))
            BUILD_HOST_ARCH := aarch64
        endif
        ifeq ($(BUILD_HOST_ARCH),)
            $(call -build-info,Unsupported host architecture: $(UNAME))
            $(call -build-error,Aborting)
        endif
    endif # BUILD_HOST_OS_BASE != windows
```

再次执行 `make`，出现了另外的 error：
```
qemu_3.1$ make
PrebuiltLibrary: objs/build/intermediates64/emulator-zlib/emulator-zlib.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libcurl/emulator-libcurl.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libssl/emulator-libssl.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libcrypto/emulator-libcrypto.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libuuid/emulator-libuuid.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libxml2/emulator-libxml2.a
Compile: emulator-libsparse <= src/backed_block.c
arm-none-eabi-gcc: error: unrecognized command line option ‘-m64’
android/build/emulator/binary.make:84: recipe for target 'objs/build/intermediates64/emulator-libsparse/src/backed_block.o' failed
make: *** [objs/build/intermediates64/emulator-libsparse/src/backed_block.o] Error 1
```

这主要是在 `android/build/emulator/binary.make` 构建配置文件中，配置的 ‘-m64’ 等编译选项，ARM 平台的编译器不支持。为这个配置文件中，添加 ‘-m64’ 等编译选项的代码，加上判断，仅在 `x86_64` 平台上才添加这些编译选项，如下面这样：
```
ifeq (x86_64,$(BUILD_HOST_ARCH64))
# Ensure only one of -m32 or -m64 is being used and place it first.
LOCAL_CFLAGS := \
    -m$(LOCAL_BITS) \
    $(filter-out -m32 -m64, $(LOCAL_CFLAGS))

LOCAL_LDFLAGS := \
    -m$(LOCAL_BITS) \
    $(filter-out -m32 -m64, $(LOCAL_LDFLAGS))
endif
```

再次执行 `make`，不出意外仍然编译失败，报错如下：
```
PrebuiltLibrary: objs/build/intermediates64/emulator-libbreakpad/emulator-libbreakpad.a
PrebuiltLibrary: objs/build/intermediates64/emulator-libdisasm/emulator-libdisasm.a
Compile: emulator-libjpeg <= jcapimin.c
arm-none-eabi-gcc: error: unrecognized command line option ‘-msse2’
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/emulator-libjpeg/jcapimin.o' failed
make: *** [objs/build/intermediates64/emulator-libjpeg/jcapimin.o] Error 1
```

这是由于在 `android/build/emulator/binary.make` 编译配置文件中，编译 C 文件时，编译选项 ‘-msse2’ 无法被 ARM 平台上的编译器识别所致：
```
$(foreach src,$(LOCAL_C_SOURCES), \
    $(eval $(call compile-c-source,$(src))) \
)
```

修改 `qemu/android/third_party/jpeg-6b/libjpeg.mk` 文件中如下的这段代码：
```
LOCAL_CFLAGS += \
    -DAVOID_TABLES \
    -O3 \
    -fstrict-aliasing \
    -DANDROID_INTELSSE2_IDCT \
    -msse2 \
    -DANDROID_TILE_BASED_DECODE \
    -Wno-all \
```

修改为下面这样：
```
LOCAL_CFLAGS += \
    -DAVOID_TABLES \
    -O3 \
    -fstrict-aliasing \
    -DANDROID_TILE_BASED_DECODE \
    -Wno-all \

ifneq (aarch64,$(BUILD_HOST_ARCH64))
LOCAL_CFLAGS += \
    -DANDROID_INTELSSE2_IDCT \
    -msse2 \

endif
```

以使得在 ARM 平台上编译的时候，不会加上 ‘-msse2’ 编译选项。再次执行 `make`，依然编译失败，报错如下：
```
Compile: android-emu <= android/wear-agent/PairUpWearPhone.cpp
Library: objs/build/intermediates64/android-emu/android-emu.a
Qt uic: ui_extended.h <-- android/android-emu/android/skin/qt/extended.ui
emulator/prebuilts/android-emulator-build/qt/linux-x86_64/bin/uic: 1: emulator/prebuilts/android-emulator-build/qt/linux-x86_64/bin/uic: Syntax error: "(" unexpected
android/build/emulator/binary.make:106: recipe for target 'objs/build/intermediates64/emulator-libui/ui_extended.h' failed
make: *** [objs/build/intermediates64/emulator-libui/ui_extended.h] Error 2
```

这是由于在 qemu/android/third_party/Qt5.mk 构建配置文件中，缺少对 ARM 平台的支持，如下面这样：
```
QT_TOP_DIR := $(QT_PREBUILTS_DIR)/$(BUILD_TARGET_TAG)
QT_TOP64_DIR := $(QT_PREBUILTS_DIR)/$(BUILD_TARGET_OS)-x86_64
QT_MOC_TOOL := $(QT_TOP64_DIR)/bin/moc
QT_RCC_TOOL := $(QT_TOP64_DIR)/bin/rcc
```

修改这个配置文件，增加对 ARM 64 平台的支持，来解决这个问题：
```
QT_TOP_DIR := $(QT_PREBUILTS_DIR)/$(BUILD_TARGET_TAG)
ifeq (aarch64,$(BUILD_HOST_ARCH))
QT_TOP64_DIR := $(QT_PREBUILTS_DIR)/$(BUILD_TARGET_OS)-aarch64
else
QT_TOP64_DIR := $(QT_PREBUILTS_DIR)/$(BUILD_TARGET_OS)-x86_64
endif
QT_MOC_TOOL := $(QT_TOP64_DIR)/bin/moc
QT_RCC_TOOL := $(QT_TOP64_DIR)/bin/rcc
```

再次执行 `make`，依然编译失败，报错如下：
```
Qt rcc (dynamic): resources.rcc <-- android/android-emu/android/skin/qt/resources.qrc
Install: objs/resources/resources.rcc
Compile: emulator-libui <= android/skin/charmap.c
arm-none-eabi-gcc: error: unrecognized command line option ‘-mmmx’
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/emulator-libui/android/skin/charmap.o' failed
make: *** [objs/build/intermediates64/emulator-libui/android/skin/charmap.o] Error 1
```

又是一个由于编译选项编译器无法识别的问题。这通过修改 qemu/android/android-emu/android/skin/sources.mk 文件中如下的代码来解决：
```
# enable MMX code for our skin scaler
ANDROID_SKIN_CFLAGS += -DUSE_MMX=1 -mmmx
```

修改为：
```
# enable MMX code for our skin scaler
ifneq ($(BUILD_TARGET_ARCH),aarch64)
    ANDROID_SKIN_CFLAGS += -DUSE_MMX=1 -mmmx
endif
```

同时修改 qemu/android/build/Makefile.top.mk 文件中的如下代码段：
```
ifdef EMULATOR_BUILD_64BITS
BUILD_TARGET_BITS := 64
BUILD_TARGET_ARCH := x86_64
BUILD_TARGET_SUFFIX := 64

include $(LOCAL_PATH)/android/build/Makefile.common.mk
endif
```

修改为：
```
ifdef EMULATOR_BUILD_64BITS
    BUILD_TARGET_BITS := 64
    ifeq (aarch64,$(BUILD_HOST_ARCH))
        BUILD_TARGET_ARCH := aarch64
    else
        BUILD_TARGET_ARCH := x86_64
    endif
    BUILD_TARGET_SUFFIX := 64

    include $(LOCAL_PATH)/android/build/Makefile.common.mk
endif
```

以增加对 ARM 64 平台的支持。Android QEMU 的构建系统在编译时，会同时编译出QEMU1 和 QEMU2 的模拟器，这主要在 android/build/Makefile.common.mk 文件中控制：
```
include $(LOCAL_PATH)/android/android-emu/Makefile.crash-service.mk

include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-common.mk

# We want to build all variants of the emulator binaries. This makes
# it easier to catch target-specific regressions during emulator development.
EMULATOR_TARGET_ARCH := arm
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk

# Note: the same binary handles x86 and x86_64
EMULATOR_TARGET_ARCH := x86
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk

EMULATOR_TARGET_ARCH := mips
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk
```

再次执行 `make`，在 ARM 平台上编译 QEMU1 时会报出如下错误：
```
Compile: emulator-libqemu <= audio/esdaudio.c
android/qemu1/audio/esdaudio.c:25:17: fatal error: esd.h: No such file or directory
compilation terminated.
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/emulator-libqemu/audio/esdaudio.o' failed
make: *** [objs/build/intermediates64/emulator-libqemu/audio/esdaudio.o] Error 1
```

可以修改上面那段代码，避免编译我们不需要的目标文件：
```
ifneq (aarch64,$(BUILD_HOST_ARCH64))
include $(LOCAL_PATH)/android/android-emu/Makefile.crash-service.mk

include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-common.mk

# We want to build all variants of the emulator binaries. This makes
# it easier to catch target-specific regressions during emulator development.
EMULATOR_TARGET_ARCH := arm
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk

# Note: the same binary handles x86 and x86_64
EMULATOR_TARGET_ARCH := x86
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk

EMULATOR_TARGET_ARCH := mips
include $(LOCAL_PATH)/android/qemu1/Makefile.qemu1-target.mk
endif
```

再次执行 `make`，依然编译失败，报错如下：
```
Library: objs/build/intermediates64/libGLcommon/libGLcommon.a
Compile: libOpenGLESDispatch <= EGLDispatch.cpp
android/android-emugl/host/libs/libOpenGLESDispatch/EGLDispatch.cpp: In function ‘bool init_egl_dispatch()’:
android/android-emugl/host/libs/libOpenGLESDispatch/EGLDispatch.cpp:38:59: error: ‘EMUGL_LIBNAME’ was not declared in this scope
android/build/emulator/binary.make:122: recipe for target 'objs/build/intermediates64/libOpenGLESDispatch/EGLDispatch.o' failed
make: *** [objs/build/intermediates64/libOpenGLESDispatch/EGLDispatch.o] Error 1
```

这是由于在 ARM 平台上编译时，缺少了常量 `EMUGL_LIBNAME` 的定义：
```
#if defined(__x86_64__)
#  define EMUGL_LIBNAME(name) "lib64" name
#elif defined(__i386__)
#  define EMUGL_LIBNAME(name) "lib" name
#else
/* This header is included by target w/o using EMUGL_LIBNAME().  Don't #error, leave it undefined */
#endif
```

修改这段代码，以便于为 ARM 平台增加常量定义：
```
#if defined(__x86_64__) || defined(__aarch64__)
#  define EMUGL_LIBNAME(name) "lib64" name
#elif defined(__i386__)
#  define EMUGL_LIBNAME(name) "lib" name
#else
/* This header is included by target w/o using EMUGL_LIBNAME().  Don't #error, leave it undefined */
#endif
```

再次执行 `make`，依然编译失败，报错如下：
```
Compile: libqemu2-common <= accel.c
In file included from /workData/zhuyb/android_emu_2.5/external/qemu_3.1/accel.c:26:0:
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/include/qemu/osdep.h:30:25: fatal error: config-host.h: No such file or directory
compilation terminated.
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/libqemu2-common/accel.o' failed
make: *** [objs/build/intermediates64/libqemu2-common/accel.o] Error 1
```

这是由于缺少了一个配置头文件，我们参考 `qemu/android-qemu2-glue/config/linux-x86_64/config-host.h` 文件来创建 `qemu/android-qemu2-glue/config/linux-aarch64/config-host.h` 文件：
```
#define HOST_AARCH64 1
#define CONFIG_POSIX 1
#define CONFIG_LINUX 1
#define CONFIG_SLIRP 1
#define CONFIG_SMBD_COMMAND "/usr/sbin/smbd"
#define CONFIG_AUDIO_DRIVERS \
    &no_audio_driver,\

#define CONFIG_PA 1
#define CONFIG_AUDIO_PT_INT 1
#define CONFIG_BDRV_RW_WHITELIST\
    NULL
#define CONFIG_BDRV_RO_WHITELIST\
    NULL
#define CONFIG_VNC 1
#define CONFIG_FNMATCH 1
#define QEMU_VERSION "2.7.0"
#define CONFIG_SDL 1
#define CONFIG_SDLABI 2.0
#define CONFIG_UTIMENSAT 1
#define CONFIG_PIPE2 1
#define CONFIG_ACCEPT4 1
#define CONFIG_SPLICE 1
#define CONFIG_EVENTFD 1
#define CONFIG_FALLOCATE 1
#define CONFIG_POSIX_FALLOCATE 1
#define CONFIG_SYNC_FILE_RANGE 1
#define CONFIG_FIEMAP 1
#define CONFIG_DUP3 1
#define CONFIG_PPOLL 1
#define CONFIG_PRCTL_PR_SET_TIMERSLACK 1
#define CONFIG_EPOLL 1
#define CONFIG_EPOLL_CREATE1 1
#define CONFIG_SENDFILE 1
#define CONFIG_TIMERFD 1
#define CONFIG_INOTIFY 1
#define CONFIG_INOTIFY1 1
#define CONFIG_BYTESWAP_H 1
#define CONFIG_HAS_GLIB_SUBPROCESS_TESTS 1
#define CONFIG_TLS_PRIORITY "NORMAL"
#define HAVE_IFADDRS_H 1
#define CONFIG_VHOST_SCSI 1
#define CONFIG_IOVEC 1
#define CONFIG_PREADV 1
#define CONFIG_FDT 1
#define CONFIG_SIGNALFD 1
#define CONFIG_FDATASYNC 1
#define CONFIG_MADVISE 1
#define CONFIG_POSIX_MADVISE 1
#define CONFIG_QOM_CAST_DEBUG 1
#define CONFIG_COROUTINE_BACKEND ucontext
#define CONFIG_COROUTINE_POOL 1
#define CONFIG_LINUX_MAGIC_H 1
#define CONFIG_PRAGMA_DIAGNOSTIC_AVAILABLE 1
#define CONFIG_HAS_ENVIRON 1
#define CONFIG_CPUID_H 1
#define CONFIG_INT128 1
#define CONFIG_TPM 1
#define CONFIG_TPM_PASSTHROUGH 1
#define CONFIG_TRACE_NOP 1
#define CONFIG_TRACE_FILE trace
#define CONFIG_IASL iasl
#define HOST_DSOSUF ".so"
```

同时我们也不再编译 QEMU2 的部分目标平台模拟器，将如下这段代码：
```
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-glue.mk

QEMU2_TARGET := x86
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := x86_64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := arm
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := arm64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := mips
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := mips64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

LOCAL_PATH := $(QEMU2_OLD_LOCAL_PATH)
```

修改为：
```
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-glue.mk

ifneq (aarch64,$(BUILD_HOST_ARCH64))
QEMU2_TARGET := x86
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := x86_64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := arm
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := mips
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

QEMU2_TARGET := mips64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk
endif

QEMU2_TARGET := arm64
include $(LOCAL_PATH)/android-qemu2-glue/build/Makefile.qemu2-target.mk

LOCAL_PATH := $(QEMU2_OLD_LOCAL_PATH)
```

再次执行 `make`，依然编译失败，报错如下：
```
Compile: libqemu2-common <= util/compatfd.c
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/util/compatfd.c: In function ‘qemu_signalfd’:
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/util/compatfd.c:103:19: error: ‘SYS_signalfd’ undeclared (first use in this function)
     ret = syscall(SYS_signalfd, -1, mask, _NSIG / 8);
                   ^
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/util/compatfd.c:103:19: note: each undeclared identifier is reported only once for each function it appears in
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/libqemu2-common/util/compatfd.o' failed
make: *** [objs/build/intermediates64/libqemu2-common/util/compatfd.o] Error 1
```

这是由于系统调用常量的定义在 ARM 64 平台上有些不同。修改 `qemu/util/compatfd.c` 文件中如下这段代码：
```
#if defined(CONFIG_SIGNALFD)
    int ret;

    ret = syscall(SYS_signalfd, -1, mask, _NSIG / 8);
    if (ret != -1) {
        qemu_set_cloexec(ret);
        return ret;
    }
#endif
```

修改为：
```
#if defined(CONFIG_SIGNALFD)
    int ret;

#ifdef __aarch64__
    ret = syscall(SYS_signalfd4, -1, mask, _NSIG / 8);
#else
    ret = syscall(SYS_signalfd, -1, mask, _NSIG / 8);
#endif
    if (ret != -1) {
        qemu_set_cloexec(ret);
        return ret;
    }
#endif
```

再次执行 `make`，依然编译失败，报错如下：
```
Compile: libqemu2-system-i386 <= tcg/tcg.c
In file included from /workData/zhuyb/android_emu_2.5/external/qemu_3.1/tcg/tcg.c:255:0:
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/tcg/i386/tcg-target.inc.c:111:19: fatal error: cpuid.h: No such file or directory
compilation terminated.
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/libqemu2-system-i386/tcg/tcg.o' failed
make: *** [objs/build/intermediates64/libqemu2-system-i386/tcg/tcg.o] Error 1
```

编译 tcg 失败，又是由于缺少对 ARM 64 平台的支持所致。修改 `qemu/android-qemu2-glue/build/Makefile.qemu2-target.mk` 文件中的如下这段代码：
```
QEMU2_SYSTEM_INCLUDES := \
    $(QEMU2_INCLUDES) \
    $(QEMU2_DEPS_TOP_DIR)/include \
    $(call qemu2-if-linux,$(LOCAL_PATH)/linux-headers) \
    $(LOCAL_PATH)/android-qemu2-glue/config/target-$(QEMU2_TARGET) \
    $(LOCAL_PATH)/target-$(QEMU2_TARGET_TARGET) \
    $(LOCAL_PATH)/tcg \
    $(LOCAL_PATH)/tcg/i386 \
```

修改为：
```
QEMU2_SYSTEM_INCLUDES := \
    $(QEMU2_INCLUDES) \
    $(QEMU2_DEPS_TOP_DIR)/include \
    $(call qemu2-if-linux,$(LOCAL_PATH)/linux-headers) \
    $(LOCAL_PATH)/android-qemu2-glue/config/target-$(QEMU2_TARGET) \
    $(LOCAL_PATH)/target-$(QEMU2_TARGET_TARGET) \
    $(LOCAL_PATH)/tcg \

ifeq (x86_64,$(BUILD_HOST_ARCH64))
QEMU2_SYSTEM_INCLUDES += \
    $(LOCAL_PATH)/tcg/i386 \

endif

ifeq (aarch64,$(BUILD_HOST_ARCH))
QEMU2_SYSTEM_INCLUDES += \
    $(LOCAL_PATH)/tcg/aarch64 \

endif
```

再次执行 `make`，依然编译失败，报错如下：
```
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/util/coroutine-ucontext.c: In function ‘qemu_coroutine_new’:
/workData/zhuyb/android_emu_2.5/external/qemu_3.1/util/coroutine-ucontext.c:86:24: error: ‘co’ is used uninitialized in this function [-Werror=uninitialized]
     CoroutineUContext *co;
                        ^
cc1: all warnings being treated as errors
android/build/emulator/binary.make:86: recipe for target 'objs/build/intermediates64/libqemu2-common/util/coroutine-ucontext.o' failed
make: *** [objs/build/intermediates64/libqemu2-common/util/coroutine-ucontext.o] Error 1
```

这是由于加了 `-Werror` 编译选项。注释掉 `qemu/android/build/Makefile.top.mk` 文件中加的 `-Werror` 编译选项即可：
```
ifneq (,$(filter windows linux, $(BUILD_TARGET_OS)))
# BUILD_TARGET_CFLAGS += -Werror
endif
```

还可以移除对 qemu-upstream 的编译，以节省时间。删除 `qemu/android-qemu2-glue/build/Makefile.qemu2-target.mk` 文件中如下这段代码即可：
```
# The upstream version of QEMU2, without any Android-specific hacks.
# This uses the regular SDL2 UI backend.

$(call start-emulator-program,qemu-upstream-$(QEMU2_TARGET_SYSTEM))

LOCAL_WHOLE_STATIC_LIBRARIES += \
    libqemu2-system-$(QEMU2_TARGET_SYSTEM) \
    libqemu2-common \

LOCAL_STATIC_LIBRARIES += \
    $(QEMU2_SYSTEM_STATIC_LIBRARIES) \

LOCAL_CFLAGS += \
    $(QEMU2_SYSTEM_CFLAGS) \

LOCAL_C_INCLUDES += \
    $(QEMU2_SYSTEM_INCLUDES) \
    $(QEMU2_SDL2_INCLUDES) \

LOCAL_SRC_FILES += \
    $(call qemu2-if-target,x86 x86_64, \
        hw/i386/acpi-build.c \
        hw/i386/pc_piix.c \
        ) \
    $(call qemu2-if-windows, \
        stubs/win32-stubs.c \
        ) \
    ui/sdl2.c \
    ui/sdl2-input.c \
    ui/sdl2-2d.c \
    vl.c \

LOCAL_LDFLAGS += $(QEMU2_SYSTEM_LDFLAGS)

LOCAL_LDLIBS += \
    $(QEMU2_SYSTEM_LDLIBS) \
    $(QEMU2_SDL2_LDLIBS) \

LOCAL_INSTALL_DIR := qemu/$(BUILD_TARGET_TAG)

$(call end-emulator-program)
```

更多编译错误问题解决的细节，此处不再赘述。
Android QEMU 模拟器在 ARM Ubuntu Linux 平台上的编译就到这里。
Done。
