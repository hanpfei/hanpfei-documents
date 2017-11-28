---
title: Anbox
date: 2017-11-28 13:05:49
categories: Android开发
tags:
- Android开发
- 翻译
---

Anbox 是在像 Ubuntu 这样的普通 GNU/Linux 系统上，一个基于容器的启动完整 Android 系统的方法。换句话说：Android 将使你在你的 Linux 系统上运行 Android，而无需虚拟化的开销。
<!--more-->
# 概述

Anbox 使用 Linux 命名空间机制（user，pid，uts，net，mount，ipc）在容器中运行完整的 Android 系统，并在任何基于 GNU/Linux 的平台上提供 Android 应用。

容器内的 Android 没有直接访问任何硬件的权限。所有的硬件访问通过主机上的 anbox 守护进程。我们复用基于 QEMU 的模拟器中为 Android 所做的 OpenGL ES 加速渲染的实现。容器内的 Android 系统使用不同的管道与主机系统通信，并通过它们发送所有的硬件访问命令。

更多细节可以查看如下的文档：

 *   [Android 硬件 OpenGL ES 模拟设计概述](https://www.wolfcstech.com/2017/09/16/opengles_android_emulation/)
 *   [Android QEMU fast pipes](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMU-PIPE.TXT)
 *   [The Android "qemud" multiplexing daemon](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMUD.TXT)
 *   [Android qemud services](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMUD-SERVICES.TXT)

Anbox 当前适合桌面使用用例，但也可以被用在移动操作系统上，比如 Ubuntu Touch，Sailfish OS 或者 Lune OS。然而，由于 Android 应用的映射当前是专用于桌面的，因此这还需要额外的工作来支持窗口用户界面栈。

# 安装

安装过程当前有一些步骤组成，它们将向你的主机系统中添加额外的组件。这些组件包括：

 * 由于没有发行版内核同时启用了它们，因此需要 binder 和 ashmem 的源码树之外的内核模块
 * 为 `/dev/binder` 和 `/dev/ashmem` 设置正确权限的 udev 规则
 * upstart 任务，用于作为用户会话的一部分启动 Anbox 会话管理器

为了使这个过程尽可能的简单，我们已经在一个 snap (参考 [https://snapcraft.io](https://snapcraft.io/)) 中打包了必须的步骤，称为 "anbox-installer"。安装器将执行所有必须的步骤。你可以通过运行如下命令，在系统上安装它提供对 snaps 的支持：
```
$ snap install --classic anbox-installer
```

另外，你可以通过如下命令获得安装器脚本：
```
$ wget https://raw.githubusercontent.com/anbox/anbox-installer/master/installer.sh -O anbox-installer
```

请注意，我们还不支持除那之外的任何 Linux 发行版。请看一下下面的章节，来了解支持的发行版的列表。

要执行安装过程就简单地调用：
```
$ anbox-installer
```

这将指导你完成安装过程。

**注意：** Anbox 当前还处于 **pre-alpha 开发状态**。不要期待它是一个用于生产环境，且包含你所需要的所有功能的完整有效系统。你一定会看到错误和崩溃。你过看到了，请不要迟疑并报告它们。

**注意：** Anbox snap 目前 **完全没有限制**，且由于这只能从边缘通道获得。适当的限制是我们想在未来实现的事，但由于 Anbox 的性质和复杂性，这不是一个简单的任务。

# 支持的 Linux 发行版

此刻我们官方支持如下的 Linux 发行版：

 * Ubuntu 16.04 (xenial)

还没测试，但可能可以工作的发行版如下：

 * Ubuntu 14.04 (trusty)
 * Ubuntu 16.10 (yakkety)
 * Ubuntu 17.04 (zesty)

# 安装并运行 Android 应用

# 从源码构建

## 要求

要编译 Anbox 运行时本身，没有什么需要特别了解的。我们使用 cmake 作为构建系统。你的主机系统中需要存在一些构建依赖：

 * libdbus
 * google-mock
 * google-test
 * libboost
 * libboost-filesystem
 * libboost-log
 * libboost-iostreams
 * libboost-program-options
 * libboost-system
 * libboost-test
 * libboost-thread
 * libcap
 * libdbus-cpp
 * mesa (libegl1, libgles2)
 * glib-2.0
 * libsdl2
 * libprotobuf
 * protobuf-compiler
 * lxc

在 Ubuntu 系统上，你可以通过如下命令安装所有的构建依赖：
```
$ sudo apt install build-essential cmake cmake-data debhelper dbus google-mock \
    libboost-dev libboost-filesystem-dev libboost-log-dev libboost-iostreams-dev \
    libboost-program-options-dev libboost-system-dev libboost-test-dev \
    libboost-thread-dev libcap-dev libdbus-1-dev libdbus-cpp-dev libegl1-mesa-dev \
    libgles2-mesa-dev libglib2.0-dev libglm-dev libgtest-dev liblxc1 \
    libproperties-cpp-dev libprotobuf-dev libsdl2-dev libsdl2-image-dev lxc-dev \
    pkg-config protobuf-compiler
```

我们建议 Ubuntu 16.04 (xenial) 及 **GCC 5.x** 作为你的构建系统。

## 构建

之后，你可以用如下命令安装 Anbox：
```
$ git clone https://github.com/anbox/anbox.git
$ cd anbox
$ mkdir build
$ cd build
$ cmake ..
$ make
```

简单的
```
$ sudo make install
```

将向你的系统中安装必须的东西。

如果你想构建 anbox snap，则你可以通过如下的步骤来完成：
```
$ mkdir android-images
$ cp /path/to/android.img android-images/android.img
$ snapcraft
```

结果将是一个 .snap 文件，你可以把它安装到支持 snaps 的系统上：
```
$ snap install --dangerous --devmode anbox_1_amd64.snap
```

# 运行 Anbox

从一个本地构建运行 Anbox 需要你了解更多的事情。请看一下 ["运行时设置"](https://github.com/anbox/anbox/blob/master/docs/runtime-setup.md) 文档。

# 文档

你可以在 Anbox 项目源码树的 *docs* 子目录下找到更多的文档。

来看看有趣的事情：

 *   [Runtime Setup](https://github.com/anbox/anbox/blob/master/docs/runtime-setup.md)
 *   [Build Android image](https://github.com/anbox/anbox/blob/master/docs/build-android.md)
 *   [Generate Android emugl source](https://github.com/anbox/anbox/blob/master/docs/generate-emugl-source.md)

# 报告 bugs

如果你发现了一个 Anbox 的问题，请 [提交一个 bug](https://github.com/anbox/anbox/issues/new)。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://github.com/anbox/anbox)
