---
title: HiKey960 开发板 android 编译
date: 2017-07-20 18:17:49
categories: Linux内核
tags:
- Linux内核
- Android开发
---

我们可以用 Android Open Source Project (AOSP) 源码和相关的硬件特有二进制文件为 Google 的手机/平板，如 Nexus 系列，Pixel 系列等编译镜像，这有时为我们对 Android 系统的研究调试及开发提供了极大的便利。除此之外，为了 Android 系统能够得到更加广泛的应用，Google 官方还对两款参考开发板提供了支持，及 [HiKey](https://android.googlesource.com/device/linaro/hikey/) 和 HiKey 960 ，因而我们也可以为方便简单地为它们编译镜像。
<!--more-->
HiKey 和 HiKey960 开发板是 Google Android 官方提供支持的开发板。Google 有提供为这两块开发板编译内核的文档，地址为 https://source.android.com/source/devices 。不过需要注意的是，这个页面的中文版，提供的是为 HiKey 开发板编译内核的方法，英文版则同时描述了为 HiKey 和 HiKey960 开发板编译内核的方法。HiKey 和 HiKey960 是两块不同的开发板，它们的硬件配置，内核编译方法还是有着细微的差别。

HiKey960 有 3GB RAM 的配置，而 HiKey 则只有 1GB 和 2GB 的 RAM 配置。HiKey960 板子如下图：

![hikey960](https://www.wolfcstech.com/images/1315506-a966b244e1c000c6.png)

使用下面的命令可以下载，构建并在 HiKey960 开发板上运行 Android。

# 编译用户空间

1. 下载 Android 源码树
``` 
$ repo init -u https://android.googlesource.com/platform/manifest -b master
$ repo sync -j24
``` 

2. 下载并提取二进制文件到 Android 源码树
```
$ wget https://dl.google.com/dl/android/aosp/arm-hikey960-NOU-7ad3cccc.tgz
$ tar xzf arm-hikey960-NOU-7ad3cccc.tgz
$ ./extract-arm-hikey960.sh
```
Android 本身是开源的，但设备本身集成的一些硬件，它们相关的软件，如驱动等，则很有可能是闭源的。这个包主要包含了设备中的硬件所需要的一些闭源的二进制文件，如特有的库等。没有这些文件，很有可能在编译 Android 的时候不会有问题，但在运行的时候会失败。

3. 构建
```
$ . ./build/envsetup.sh
$ lunch hikey960-userdebug
$ make -j2
```
就像为任何设备编译 Android 一样，为 HiKey 960 编译 Android 镜像。

# 刷写镜像
1. 通过打开开关 1 和 3 来进入 fastboot mode。
![](https://www.wolfcstech.com/images/1315506-c2747c1381edf7b0.jpg)
开关在板子的如上图所示的这一面，具体位置是在右上角。在图中的右上角可以清晰地看到 “Ext boot”，“Boot mode” 等字样，开关就位于它们的左边并紧挨它们。开关可以通过拨动白色的拨片开打开关闭。板子上可以清晰地看到这些开关的编号，实际上自上至下这些开关的编号分别为 “3”、“2” 和 “1”。

2. 运行如下的命令刷写镜像
```
fastboot flash boot out/target/product/hikey960/boot.img
fastboot flash dts out/target/product/hikey960/dt.img
fastboot flash system out/target/product/hikey960/system.img
fastboot flash cache out/target/product/hikey960/cache.img
fastboot flash userdata out/target/product/hikey960/userdata.img
```

3. 关闭开关 3 并重启板子的电源

HiKey 960 这块板子在拿到手的时候，已经刷写了 AOSP 编译出的 userdebug 版镜像。将设备脸上 USB，是可以通过 adb 访问设备的。那通过 `adb reboot bootloader` 进入 fastboot 模式岂不是要方便得多么？

在大多数时候，使用 `adb reboot bootloader` 进入 fastboot 模式确实要方便很多，然而 adb 命令也并不是总是可用的，比如修改了内核的代码，修改了 `init` 进程的代码，而且不小心把功能改烂了，那么 Android 的启动过程就不会执行到 adbd 启动，也就是我们可以使用 adb 的阶段。在这个时候，通过设置这些开关，就像手机上通过同时按住音量上下键和电源键进入 fastboot 模式一样，就变得非常有用了。

# 编译内核

1. 运行下面的命令
```
$ git clone https://android.googlesource.com/kernel/hikey-linaro
$ cd hikey-linaro
$ git checkout -b android-hikey-linaro-4.4 origin/android-hikey-linaro-4.4
$ make ARCH=arm64 hikey960_defconfig
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-android- -j24
```
Android 用户空间代码和内核代码是分开存放管理的，AOSP 源码树中并不包含内核的代码，而只包含一些已经编译好内核镜像文件。且通常针对不同硬件设备厂商的 Android linux 内核，会位于不同的 git repo 下。可以专门为内核建立一个目录，比如也命名为 `kernel`，然后在这个目录下 clone 内核代码。除了前面看到的 `hikey-linaro`，还有 `msm`、`mediatek`、`tegra`等等版本的内核代码。这里下载 HiKey 的 Android Linux 内核源码，执行配置并编译。
执行 `make ARCH=arm64 hikey960_defconfig` 命令，会根据预先定义好的一个配置文件，这里也就是 `hikey960_defconfig` 文件，在当前目录下生成后面编译内核所需要的 `.config` 文件。  `hikey960_defconfig` 文件实际位于内核源码树的 `arch/arm64/configs/hikey960_defconfig`，这个文件的内容类似下面这样：
```
CONFIG_POSIX_MQUEUE=y
CONFIG_FHANDLE=y
CONFIG_AUDIT=y
CONFIG_NO_HZ=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_SCHED_WALT=y
CONFIG_BSD_PROCESS_ACCT=y
CONFIG_BSD_PROCESS_ACCT_V3=y
. . . . . . 
```
主要定义了用于控制一些功能特性的打开或关闭的开关选项。要打开或关闭内核的某一个功能特性，修改这个文件是一种非常方便的方法。

2. 更新 boot image 中的内核

 * 将编译产生的 hi3660-hikey960.dtb (arch/arm64/boot/dts/hisilicon/hi3660-hikey960.dtb) 复制到  hikey-kernel 目录下，文件名仍然为 hi3660-hikey960.dtb。hikey-kernel 目录在 AOSP 源码库中的具体位置为 `device/linaro/hikey-kernel`。
 * 将编译生成的 Image 文件 (arch/arm64/boot/Image.gz) 拷贝到 hikey-kernel 目录下，修改文件名为 Image.gz-hikey960。这将覆盖原来存在的同名文件。

3. 编译 boot image
在 Android 源码树的根目录执行如下命令：
```
$ make bootimage -j24
```
编译 Android 用户空间代码时，执行 make 之前所需要做的在执行这个命令之前当然也需要做。需要注意的是，boot image 中除了包含内核之外，还包含了 init 等系统核心可执行文件等。如果 Android 用户空间代码及相关 image 没有更新，可以通过命令 `fastboot flash boot out/target/product/hikey960/boot.img` 只更新 boot image。

# 设置序列号
要设置随机的序列号，则运行：
```
$ fastboot getvar nve:SN@16_DIGIT_NUMBER
```

Bootloader 将通过 `androidboot.serialno=` 把生成的序列号导到内核。

### [打赏](https://www.wolfcstech.com/about/donate.html)

# 参考文档

[Using Reference Boards](https://source.android.com/source/devices)
