---
title: 运行 Anbox
date: 2017-11-28 19:05:49
categories: Android开发
tags:
- Android开发
- 翻译
---

# 概述

Anbox 运行时主要由两个分开的实例构成：

 * 容器管理器
 * 会话管理器
<!--more-->
容器管理器的工作是管理容器的建立，并在它的生命周期内维护它。它的职责是启动我们用以运行 Android 系统的 LXC 环境。

会话管理器运行于登录到 Linux 系统的用户的会话内。它将通过一些 sockets 与运行在容器内的 Android 实例通信，并提供与 Linux 系统的集成。它还扮演多路复用器的角色，将 Android 应用映射为桌面环境的单个窗口。当前所有的应用窗口由相同的进程（会话管理器）所有。应用逻辑本身依然位于 Android 容器内的另外的进程中。

下图展示了一个总体的架构：

![architecture.png](https://www.wolfcstech.com/images/1315506-c4c2bff682e4d697.png)

# 应用映射

Android 应用被映射为桌面环境中单独的窗口。这是通过插入 Android hwcomposer HAL
 模块来实现的，该模块接收一组图层以在屏幕上合成。Anbox 告诉 SurfaceFlinger  通过它的 hwcomposer 实现为每个应用获得图层，并把这与它从 Android WindowManager 接收的其它信息结合，来把独立的图层映射为应用。更多详情请查看如下的实现：

 * android/hwcomposer
 * src/anbox/graphics/layer_composer.cpp
 * src/anbox/wm/manager.cpp

# 编译 Android 镜像

对于 Anbox，我们使用的是 Android 的最小定制版本，但是基于 [Android 开放源代码项目](https://source.android.com/)
 最近版本的所有工作。

要重新构建 Android 镜像，你首先需要获得所有相关的源码。这将消耗你大量的磁盘空间（~40GB）。AOSP 建议至少要有 100 GB 的空闲磁盘空间。也可以查看 [他们的](https://source.android.com/source/requirements.html) 页面。

一般来说，为了构建 Anbox Android 镜像，位于 [AOSP 工程的页面](https://source.android.com/source/requirements.html) 的指南是适用的。此处我们将不再专门描述通常如何构建 Android 系统，而只聚焦于 Anbox 所需的步骤。

## 获得所有相关源码

首先建立一个新的 workspace，你将在其中下载所有的源码。
```
$ mkdir $HOME/anbox-work
```

现在，通过下载 manifest 并启动获取源码来初始化仓库：
```
$ cd $HOME/anbox-work
$ repo init -u https://github.com/anbox/platform_manifests.git -b anbox
$ repo sync -j4
```

依赖于你的网络连接，这将消耗一些时间。

## 构建 Android

当所有的源码都成功地下载之后，你就可以启动构建 Android 本身了。

首先通过 `envsetup.sh` 脚本初始化环境。
```
$ . build/envsetup.sh
```

然后使用 `lunch` 初始化构建。
```
$ lunch anbox_x86_64-userdebug
```

支持的构建目标的完整类表如下：

 * anbox_x86_64-userdebug
 * anbox_armv7a_neon-userdebug
 * anbox_arm64-userdebug

现在通过如下命令构建所有其它的东西：
```
$ make -j8
```

一旦构建完成，我们需要获取结果，并用它们创建适用于 Anbox 的镜像文件。
```
$ cd $HOME/anbox-work/vendor/anbox
$ scripts/create-package.sh  \
    $PWD/../../out/target/product/x86_64/ramdisk.img  \
    $PWD/../../out/target/product/x86_64/system.img
```

这将在当前目录下创建一个名为 *android.img* 的文件。

现在，你就可以在 Anbox 运行时中使用使用你的定制镜像了。

# 以自己构建的 android.img 运行 Anbox

如果你已经在你的系统上安装了 Anbox，你需要先停掉它。在通过 Anbox 安装器脚本完成 Anbox 安装并 snap 之后，Anbox 会自动启动，查看主机的进程列表将看到如下内容：
```
$ ps -aux | grep anbox
root      7113  0.0  0.0 766588 12308 ?        Ssl  14:46   0:00 /snap/anbox/65/usr/bin/anbox container-manager --data-path=/var/snap/anbox/common/ --android-image=/snap/anbox/65/android.img --daemon
hanpfei+  8327  7.2  1.5 2464708 252328 ?      Sl   14:49   0:01 /snap/anbox/65/usr/bin/anbox session-manager
root      8339  0.0  0.0  36776  3616 ?        Ss   14:49   0:00 /snap/anbox/current/libexec/lxc/lxc-monitord /var/snap/anbox/common/containers 14
root      8341  0.0  0.0 772888  8156 ?        Ss   14:49   0:00 [lxc monitor] /var/snap/anbox/common/containers default
100000    8350  0.0  0.0   7920  5912 ?        Ss   14:49   0:00 /system/bin/sh /anbox-init.sh
100000    8423  0.0  0.0  16728  9260 ?        Sl   14:49   0:00 /system/bin/anboxd
110000    8877  0.8  0.5 1038888 95412 ?       Sl   14:49   0:00 org.anbox.appmgr
hanpfei+  9094  0.0  0.0  19300   976 pts/21   S+   14:49   0:00 grep --color=auto anbox
```

此时，可以这样做来停掉 Anbox：
```
$ sudo systemctl stop snap.anbox.container-manager
```

再次查看主机的进程列表，将无法再看到 Anbox 相关的进程。

同时停掉它们是很重要的，容器管理器和会话管理器。

一旦两个服务都被停掉了，你可以通过运行如下命令用定制的 android.img 启动容器管理器：
```
$ datadir=$HOME/anbox-data
$ mkdir -p $datadir/rootfs
$ sudo anbox container-manager \
    --android-image=/path/to/android.img \
    --data-path=$datadir
```

这将启动容器管理器并在特定的数据路径中设置容器根文件系统。
```
$ ls -alh $HOME/anbox-data
total 20K
drwxrwxr-x  5 ubuntu  ubuntu  4,0K Feb 22 08:04 .
drwxrwxr-x 16 ubuntu  ubuntu  4,0K Feb 22 08:04 ..
drwxr-xr-x  2 100000  100000 4,0K Feb 22 08:04 cache
drwxr-xr-x  2 100000  100000 4,0K Feb 22 08:04 data
drwxr-xr-x  2 root    root   4,0K Feb 22 08:04 rootfs
```

**注意：** 如果你查看 $HOME/anbox-data/rootfs 目录，你将不会看到任何东西，因为容器管理派生了一个私有的挂载命名空间，它阻止了外面查看它的挂载点。

*cache* 和 *data* 目录被绑定-挂载到 rootfs，位于 *rootfs/data* 和 *rootfs/cache*。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://github.com/anbox/anbox/blob/master/docs/runtime-setup.md)
[原文](https://github.com/anbox/anbox/blob/master/docs/build-android.md)

