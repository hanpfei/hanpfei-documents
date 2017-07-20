---
title: QEMU 构建系统架构
date: 2017-07-20 15:35:49
categories: Android开发
tags:
- Android开发
- 翻译
---

这份文档旨在帮助开发者理解 QEMU 构建系统的架构。正如使用 GNU autotools 的项目一样，QEMU 构建系统有两个阶段，第一步开发者运行 `configure` 脚本确定本地构建环境特性，然后执行 `make` 构建整个项目。与 GNU autotools 的相似之处仅此而已，因此请忘掉你已知关于它们的东西。
<!--more-->
# 第一步：configure

QEMU 配置脚本是直接用 shell 写的，且应该与任何 POSIX shell 兼容，因此它使用 `#!/bin/sh`。这种做法的一个重要影响是，在以 bash 为主 host 的开发平台上避免使用 bash-isms 很重要。

与 autoconf 脚本相反，QEMU 的配置在检查功能特性时是静默的。只有在出错时它才显示输出，或者在完成时展示最后启用的功能特性总结。

为配置脚本添加新的检查项通常包括如下任务：

 * 初始化一个或多个包含默认状态的变量。
   理想的功能特性应该自动检查它们是否存在，因此要努力避免通过硬编码初始状态来启用或禁用，那样强迫用户在每次调用 configure 时传入一个 `--enable-XXX / --disable-XXX` 标记。

 * 为命令行参数解析器添加支持来处理功能特性 XXX 所需要的新的 `--enable-XXX / --disable-XXX` 标记。

 * 为帮助输出消息添加信息以报告新的功能特性标记。

 * 添加代码执行实际的功能特性检查。如上面看到的那样，努力完全动态地检查启用/禁用。

 * 添加代码在完成时的配置总结中打印功能特性状态。

 * 在完成时为 $config_host_mak 添加新的 makefile 变量。

以在 configure 中探测 gnutls （一个简化版本）为例，我们有如下的片段：

```
  # Initial variable state
  gnutls=""

  ..snip..

  # Configure flag processing
  --disable-gnutls) gnutls="no"
  ;;
  --enable-gnutls) gnutls="yes"
  ;;

  ..snip..

  # Help output feature message
  gnutls          GNUTLS cryptography support

  ..snip..

  # Test for gnutls
  if test "$gnutls" != "no"; then
     if ! $pkg_config --exists "gnutls"; then
        gnutls_cflags=`$pkg_config --cflags gnutls`
        gnutls_libs=`$pkg_config --libs gnutls`
        libs_softmmu="$gnutls_libs $libs_softmmu"
        libs_tools="$gnutls_libs $libs_tools"
        QEMU_CFLAGS="$QEMU_CFLAGS $gnutls_cflags"
        gnutls="yes"
     elif test "$gnutls" = "yes"; then
        feature_not_found "gnutls" "Install gnutls devel"
     else
        gnutls="no"
     fi
  fi

  ..snip..

  # Completion feature summary
  echo "GNUTLS support    $gnutls"

  ..snip..

  # Define make variables
  if test "$gnutls" = "yes" ; then
     echo "CONFIG_GNUTLS=y" >> $config_host_mak
  fi
```

# 辅助函数
配置脚本提供了大量的辅助函数来帮助开发者检查系统功能特性：

 * do_cc $ARGS...
   尝试运行系统 C 编译器并给它传递 $ARGS...

 * do_cxx $ARGS...
   尝试运行系统 C++ 编译器并给它传递 $ARGS...

 * compile_object $CFLAGS
   尝试以系统 C 编译器使用 $CFLAGS 编译一个测试程序。测试程序必须已经在前面写入了称为 $TMPC 的文件。

 * compile_prog $CFLAGS $LDFLAGS
   尝试以系统 C 编译器使用 $CFLAGS 编译一个测试程序，并以系统链接器用 $LDFLAGS 进行链接。测试程序必须已经在前面写入了称为 $TMPC 的文件。

 * has $COMMAND
   确定当前环境中是否存在 $COMMAND，可以是一个 shell 内建命令或可执行二进制文件，成功时返回 0。

 * path_of $COMMAND
   返回 $COMMAND 的完全限定路径，将它打印到 stdout，成功时返回 0。

 * check_define $NAME
   检查系统 C 编译器是否定义了宏 $NAME。

 * check_include $NAME
   检查 $NAME 头文件对系统 C 编译器是否可用。

 * write_c_skeleton
   向 $TMPC 指向的临时文件写入一个最小的 C 程序 `main()` 函数。

 * feature_not_found $NAME $REMEDY
   功能特性 $NAME 在系统上不可用时向 stderr 打印一条消息，提示用户尝试 $REMEDY 定位问题。

 * error_exit $MESSAGE $MORE...
   向 stderr 打印 $MESSAGE，后面跟 $MORE... ，然后以非零状态码从配置脚本中退出。

 * query_pkg_config $ARGS...
   以 $ARGS 为参数运行 `pkg-config`。如果 QEMU 正在执行静态构建，则 `--static` 将被自动地加进 `$ARGS`。

# 第二步：makefiles

QEMU 构建系统需要使用 GNU make。

尽管源码分布于多个子目录下，构建系统本质上应该被认为是非递归的，与在 `automake` 中看到的通常的实践相反。有一些 make 的递归调用，但这与要构建的东西有关，而不是源码目录的结构。

QEMU 当前支持 VPATH 和非 VPATH 构建，因此有三种通用方式调用 `configure` 并执行构建。

 * VPATH，完全在 QEMU 源码树之外构建产品
```
$ cd ../
$ mkdir build
$ cd build
$ ../qemu/configure
$ make
```

 * VPATH，在 QEMU 源码树的一个子目录中构建产品
```
$ mkdir build
$ cd build
$ ../configure
$ make
```

 * 非 VPATH，在任何地方构建产品
```
$ ./configure
$ make
```

QEMU 的维护者通常建议开发者使用 VPATH 构建。QEMU 的补丁期待确保 VPATH 构建依然有效。

# 模块结构

QEMU 构建系统有大量的重要输出：

 * 工具 - qemu-img，qemu-nbd，qga (guest agent)，等等
 * 系统模拟器 - qemu-system-$ARCH
 * 用户空间模拟器 - qemu-$ARCH
 * 单元测试

源码是高度模块化的，分割多个文件，以便尽可能少地重复编译所有这些组件。可以认为是两个不同的文件组，那些独立于 QEMU 仿真目标的文件和依赖于
 QEMU 仿真目标的文件组。

独立于仿真目标的文件集合中是各种通用辅助代码，比如错误处理基础设施，标准数据结构，平台移植性封装函数，等等。这些代码可以只被编译一次，而把它们的 .o 文件链接到所有的输出二进制文件。

依赖于仿真目标的文件集合中是 CPU 模拟，设备模拟和许多胶水代码。这有时还不得不编译多次，为每个目标编译一次。

所有二进制文件都用到的实用代码被编译为一个称为 libqemuutil.a 静态包，它会被链接进所有的二进制文件。为了提供只有一部分二进制文件需要的钩子，libqemuutil.a 中的代码可能依赖于其它不完全由所有 QEMU 二进制实现的函数。为了处理这种情况，还有另一个称为 libqemustub.a 的库，它为所有这些函数提供了 dummy stubs。如果没有真实的实现，则它们将被延迟链接进二进制中。以这种方式，libqemustub.a 静态库可以被看作一个弱符号概念的可移植实现。所有的二进制文件应该同时链接 libqemuutil.a 和 libqemustub.a。比如

```
 qemu-img$(EXESUF): qemu-img.o ..snip.. libqemuutil.a libqemustub.a
```

# Windows 平台可移植性

在 Windows 平台上，所有的二进制文件都有后缀 '.exe'，因此所有创建二进制文件的 Makefile 规则都必须在二进制文件名中包含 `$(EXESUF)` 变量。比如，

```
 qemu-img$(EXESUF): qemu-img.o ..snip..
```

这在 Windows 平台上展开为 '.exe'，或者在其它平台上为 ''。

系统模拟器二进制文件的另一重复杂性是需要生成两个单独的二进制文件。主二进制文件（例如 qemu-system-x86_64.exe）与 Windows 控制台运行时子系统链接。这些预期将从命令提示符窗口运行，因此，将 stderr 打印到启动它们的控制台。

生成的第二个二进制文件在它们的名字的最后有一个 'w' (比如 qemu-system-x86_64w.exe )，它们与 Windows 图形运行时子系统链接。这些预计将直接从桌面运行，且将打开一个专门的终端窗口用于 stderr 输出。

Makefile.target 将首先为图形子系统生成二进制文件，然后使用 objcopy 重新与终端子系统链接生成第二个二进制文件。

# 目标变量命名

QEMU 约定是用于定义变量来列出不同的目标文件组的。它们的命名约定为 $PREFIX-obj-y。比如，libqemuutil.a 文件将与变量 `util-obj-y` 列出的所有目标文件链接。因此，比如，util/Makefile.obj 将包含一系列看起来像这样的定义：

```
  util-obj-y += bitmap.o bitops.o hbitmap.o
  util-obj-y += fifo8.o
  util-obj-y += acl.o
  util-obj-y += error.o qemu-error.o
```

当有一个目标文件需要基于主机系统的一些特性有条件地构建时，配置脚本将为条件定义一个变量。比如，在 Windows 上它将定义 $(CONFIG_POSIX) 指为 'n'，而 $(CONFIG_WIN32) 值为 'y'。现在可以在列出目标文件时使用配置变量了。比如，

```
  util-obj-$(CONFIG_WIN32) += oslib-win32.o qemu-thread-win32.o
  util-obj-$(CONFIG_POSIX) += oslib-posix.o qemu-thread-posix.o
```

在 Windows 上被扩展为

```
  util-obj-y += oslib-win32.o qemu-thread-win32.o
  util-obj-n += oslib-posix.o qemu-thread-posix.o
```

由于 libqemutil.a 被链接进 $(util-obj-y) ，在 Windows 平台构建中，$(util-obj-n) 中列出的 POSIX 特有文件被忽略。


# CFLAGS / LDFLAGS / LIBS 处理

有许多不同的二进制文件为了不同目的而构建，其中一些甚至可能是通过git子模块拉入的第三方库。因此通常在 QEMU 中要避免使用全局的 CFLAGS 变量，由于它将应用于太多的构建目标。

所有的 QEMU 代码（比如，*除了* GIT子模块工程）需要的标记都被放进了 $(QEMU_CFLAGS) 变量。对于链接器标志，有时会使用 $(LIBS) 变量，但是优先选择几个更有针对性的变量。$(libs_softmmu) 用于必须链接到系统模拟器的库，$(LIBS_TOOLS) 用于像 qemu-img，qemu-nbd，等等的工具，$(LIBS_QGA) 用于 QEMU guest agent。当前没有专门用于用户空间模拟器的变量作为全局 $(LIBS)，或者下面显示的更有针对性的变量就足够了。

除了这些变量，可以针对各个源代码文件提供 cflags 和 libs，通过以 $FILENAME-cflags 和 $FILENAME-libs 的形式定义变量。比如，curl 块驱动需要链接 libcurl 库，于是 block/Makefile 定义了一些变量：

```
  curl.o-cflags      := $(CURL_CFLAGS)
  curl.o-libs        := $(CURL_LIBS)
```

这两个变量的影响范围有一点不同。当链接任何包含 curl.o 目标文件的二进制文件时 libs 都会被用到，而 cflags 只有在编译 curl.c 时被用到。

# 静态定义的文件

下面的重要文件是在源码树中静态定义的，以构建 QEMU 所需的规则进行。它们的行为受稍后列出的动态创建的文件影响。

 * Makefile
调用 `make` 构建 QEMU 的所有组件的主入口点。默认的 'all' target 自然地构建每一个组件。各种工具和辅助二进制文件直接通过一个非递归的规则集合构建。
每个系统/用户空间模拟器需要一些稍有不同的 make 规则/变量。因此，make 将为每个模拟器递归地调用。
递归调用将最终处理顶层的 Makefile.target 文件（稍后再说）。

 * */Makefile.objs
由于源码分布于多个目录下，每个文件的规则类似地都被模块化了。因此，每个子目录包含 .c 文件的也将常常包含一个 Makefile.objs 文件。这些文件不直接由一个递归的 make 调用，而是被导入到顶层的 Makefile 和/或 Makefile.target。
每个 Makefile.objs 通常只声明一系列变量列出当前目录下需要从源码文件构建的 .o 文件。它们还定义任何定制的链接器或编译器标志。比如在 block/Makefile.objs 中：
```
  block-obj-$(CONFIG_LIBISCSI) += iscsi.o
  block-obj-$(CONFIG_CURL) += curl.o

  ..snip...

  iscsi.o-cflags     := $(LIBISCSI_CFLAGS)
  iscsi.o-libs       := $(LIBISCSI_LIBS)
  curl.o-cflags      := $(CURL_CFLAGS)
  curl.o-libs        := $(CURL_LIBS)
```
如果 Makefile.objs 文件定义了规则，则它们都应该使用 $(obj) 作为 target 的前缀，比如：
```
  $(obj)/generated-tcg-tracers.h: $(obj)/generated-tcg-tracers.h-timestamp
```

 * Makefile.target
这个文件提供了构建每个独立的系统或用户空间模拟器 target 的入口点。每个开启的 target 拥有它自己的子目录。比如如果 configure 以参数 '--target-list=x86_64-softmmu' 运行，则将创建 'x86_64-softmu' 子目录，包含一个 'Makefile'，符号链接回 Makefile.target。
因此当调用递归的 '$(MAKE) -C x86_64-softmmu' 时，它将最终使用 Makefile.target 构建规则。

 * rules.mak
这个文件为调用构建工具提供了通用的辅助规则，特别是编译器和链接器。这也包含了用于将源树中所有 Makefile.objs 的变量定义合并到主 Makefile 上下文中的magic（hairy）'unnest-vars' 函数。

 * default-configs/*.mak
default-configs/ 下的文件控制每个 QEMU 系统和用户空间模拟器内置什么模拟的硬件。它们仅仅包含长长的配置变量定义列表。比如，default-configs/x86_64-softmmu.mak 有：
```
  include pci.mak
  include sound.mak
  include usb.mak
  CONFIG_QXL=$(CONFIG_SPICE)
  CONFIG_VGA_ISA=y
  CONFIG_VGA_CIRRUS=y
  CONFIG_VMWARE_VGA=y
  CONFIG_VIRTIO_VGA=y
  ...snip...
```
这些文件几乎不需要修改，除非为一个特定的系统/用户空间模拟器启用新的设备/硬件。

 * tests/Makefile
编译单元测试的规则。这个文件直接被包含进顶层的 Makefile，因此这个文件中定义的任何东西将影响整个构建系统。当为测试编写规则时需要小心，确保它们只应用于单元测试的执行/构建。

 * tests/docker/Makefile.include
Docker 测试的规则。像 tests/Makefile 一样，这个文件直接被包含进顶层的 Makefile，这个文件中定义的任何东西将影响整个构建系统。

 * po/Makefile
从文本 .po 文件源构建和安装二进制消息目录的规则。它几乎从不需要因任何原因而修改。

# 动态创建的文件

下面的文件是由 configure 动态生成，来控制静态定义的 makefiles 的行为的。这使得 QEMU makefiles 无需像在 autotools 中看到的做预处理，其中 Makefile.am 生成  Makefile.in 生成 Makefile。

 * config-host.mak
当 configure 已经确定了构建主机的特性时，它将向 config-host.mak 文件写入一个长长的变量列表。这提供了各种安装目录，编译器/链接器标记和大量与启用的可选功能特性有关的 CONFIG_* 变量。它被顶层的 Makefile 引入以裁剪构建输出。
这里定义的变量适用于所有的 QEMU 构建输出。每个模拟器目标文件可能不同的变量定义在下一个文件. . .
它还被用作一个独立的检查机制。如果 make 看到 configure 的修改时间戳比 config-host.mak 的更新，则 configure 将重新运行。

 * config-host.h
源码使用 config-host.h 文件来确定启用了那些功能特性。它是使用 scripts/create_config 程序根据 config-host.mak 的内容生成的。它提取所有的 CONFIG_* 变量，大部分 HOST_* 变量和其它一些来自于 config-host.mak 的杂项变量，以 C 预处理器宏格式化它们。

 * $TARGET-NAME/config-target.mak
TARGET-NAME 是系统或用户空间模拟器的名字，比如 x86_64-softmmu 表示 x86_64 架构的系统模拟器。这个文件包含在每个模拟器上需要变化的变量。比如，它将指出是否为目标文件启用 KVM 或 Xen，及链接目标文件所需的其它潜在的定制库。

 * $TARGET-NAME/config-devices.mak
TARGET-NAME 是系统或用户空间模拟器的名字。config-devices.mak 文件是 make 使用 scripts/make_device_config.sh 程序自动生成的，以 default-configs/$TARGET-NAME 文件作为输入。

 * $TARGET-NAME/Makefile
这是 make 递归地构建单独的系统或用户空间模拟器时所使用的入口点。它仅仅是到顶层的 Makefile.target 的符号链接。

译自 qemu 官方文档 qemu/docs/build-system.txt 。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。
