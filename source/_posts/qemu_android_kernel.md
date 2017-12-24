---
title: 如何预编译 Android 模拟器专用内核
date: 2017-12-23 19:05:49
categories: 虚拟化
tags:
- 虚拟化
- Linux内核
- 翻译
---

I. 辅助脚本
-----------------

我们现在提供了一个辅助脚本来重新构建内核，其位于 `$AOSP/prebuilts/qemu-kernel/build-kernel.sh`。
<!--more-->
请确保使用了 `aosp/master` 的 checkout，而不是 `aosp/studio-XXX` 中的一个，后者不包含重新构建内核所需的预编译目标工具链二进制文件。

你需要位于 `android.googlesource.com/kernel/goldfish.git` 的源文件的分支 `origin/android-goldfish-<version>`，其中 `<version>` 为应用于你的系统镜像的内核版本。

大致说来：

  2.6.27     -> 任何 Gingerbread 之前的版本。(<= Android 2.2)
  2.6.29     -> Gingerbread (2.3) 直到 JellyBean MR2 (4.2.x)
  3.4        -> KitKat (4.4.x) 和 Lollipop 32 位 (5.0/5.1)
  3.10       -> Lollipop 64 位和 Marshmallow (6.0)
  3.18       -> 当前的实验性版本。

此外，你需要选择正确的 'config'，其对应于你想要支持的虚拟的板子。当前有两个可以考虑：

- 传统的虚拟板子称为 'goldfish'，对应于由传统的 QEMU1 代码库所支持的虚拟硬件。

- 最新的虚拟板子称为 'ranchu'，对应于由 QEMU2 代码库所支持的虚拟硬件。

它们之间的主要差别在于 'goldfish' 支持已经废弃的 goldfish 特有 NAND 和 eMMC 虚拟设备，它们在 QEMU2 中已经被 virtio-block 替代（为了好得多的性能和可靠性）。更多细节，请参考 `android/docs/GOLDFISH-VIRTUAL-HARDWARE.TXT`。

通过 `--arch=<name>` 选项来指定要为哪个架构编译，其中 `<name>` 为模拟器的 CPU 架构名称（比如 'arm'，'mips'，'x86'，'x86_64'，'arm64' 和 'mips64'）中的一个。

每个架构都有默认的配置（典型的为 'goldfish'），你可以通过 `--config=<name>` 选项覆盖它，其中 `<name>` 是位于 `$KERNEL/arch/*/configs/` 下的子目录的名字。

默认的输出目录将是 `/tmp/kernel-qemu/<arch>-<kernel_version>/`，但这可以通过 `--out=<dir>` 选项覆盖。

默认情况下，脚本将试图从 `$AOSP/prebuilts/gcc/` 找到一个适当的工具链，但是你可以使用 `--cross=<prefix>` 选项指定一个不同的。

查看 `build-kernel.sh --help` 来了解更多选项和细节。

这里是重新构建流行的内核配置的选项的列表：

  Goldfish:
```
     32-bit ARMv7-A     --arch=arm
     32-bit i386        --arch=x86
     32-bit MIPS        --arch=mips
     64-bit x86_64      --arch=x86_64
```

  Ranchu:
```
     32-bit ARMv7-A     --arch=arm --config=ranchu
     32-bit i386        --arch=x86 --config=i386_ranchu
     32-bit MIPS        --arch=mips --config=ranchu
     64-bit ARMv8       --arch=arm64
     64-bit x86_64      --arch=x86_64 --config=x86_64_ranchu
     64-bit MIPS        --arch=mip64 --config=ranchu
```

脚本将在它的输出目录生成如下的文件：

    kernel-qemu            与 QEMU 一起使用的内核镜像，使用模拟器的 '-kernel <file>' 选项来使用它，其中 `<file>` 为该文件的路径。

    vmlinux-qemu           用于以 GDB 调试内核的符号文件（参考下文）。

    LINUX_KERNEL_COPYING   内核的许可文件，必须与二进制文件的每份拷贝一起提供。

    README                 解释如何编译的小 README 文件。、

II. 以 QEMU 调试内核
-----------------------------

在你需要调试内核的情况下（比如检查内核 panics），你可以执行下面的过程：

  1) 从源码重新构建内核（参考前面的小节）。

  2) 以特殊的选项启动模拟器来使用新内核镜像，启动内核调试，且不要启动 vCPU 来等待 GDB：
```
         emulator -kernel /path/to/kernel-qemu \
                  <other-options> \
                  -qemu -s -S
```

  3) 在另一个终端里，启动一个 GDB 客户端，它将 attach 到模拟器进程，读取内核符号文件之后：
```
         $AOSP/prebuilts/gdb/linux-x86/bin/gdb
         file /path/to/vmlinux-qemu
         target remote :1234
         b panic
         c
```

注意：位于 `$AOSP/prebuilts/` 下的 gdb 版本支持所有的 Android 目标架构。你的 'host' gdb 可能只支持 x86/x86_64。

注意：你可以在使用 'c' 命令启动执行之前用 'b' 命令打任意多个断点。

一旦你命中了断点，使用 'bt' 来打印栈追踪。参考 GDB 手册来了解更多信息。

III. 从头开始重新构建
----------------------------

如果你不想，或不能使用脚本，这里有一份手动的直到：

你需要在你的 path 中有一份适当的交叉工具链（比如，'arm-eabi-gcc --version' 必须能够工作）

然后（对于版本2.6.29）：
```
git clone https://android.googlesource.com/kernel/goldfish.git kernel-goldfish
cd kernel-goldfish
git checkout origin/android-goldfish-2.6.29

export CROSS_COMPILE=arm-eabi-
export ARCH=arm
export SUBARCH=arm
make goldfish_defconfig    # configure the kernel
make -j2                   # build it
```
=> 这将产生名为 `arch/arm/boot/zImage` 的文件

注意：分支 android-goldfish-2.6.27 现在已经废弃。不要使用它。

现在你可以这样使用它：
```
  emulator -kernel path/to/your/new/zImage <other-options>
```

你可以通过在上面的指示中使用 `goldfish_armv7_defconfig` （而不是 `goldfish_defconfig`）来构建一个 ARMv7 兼容的内核镜像。注意，你将需要通过使用 `-cpu cortex-a8` 选项启用 ARMv7 模拟，如下：
```
  emulator -kernel path/to/your/new/zImage <other-options> -qemu -cpu cortex-a8
```

如果你的内核镜像的名字以 `-armv7` 结尾，则模拟器二进制文件将自动地为你启用 ARMv7 模拟，因此执行下面的命令应该是等价的：
```
  emulator -kernel path/to/your/kernel-armv7 <other-options>
```
Voila !

原文 $QEMU/android/docs/ANDROID-KERNEL.TXT

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
