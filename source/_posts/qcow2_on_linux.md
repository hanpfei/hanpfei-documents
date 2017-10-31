---
title: 在 Linux 上如何挂载 qcow2 磁盘镜像
date: 2017-10-31 18:35:49
categories: Linux
tags:
- Linux
- 翻译
---

当在一个虚拟层运行客户虚拟机（VM）时，我们可以创建一个或多个磁盘镜像专门用于该虚拟机。作为一个 “虚拟的” 磁盘卷，磁盘镜像代表附加到虚拟机 VM 的存储设备（比如，硬盘驱动器或闪存驱动器）的内容和结构。如果你想要在不启动虚拟机的情况下，修改 VM 的磁盘镜像中的文件，你可以 “挂载” 磁盘镜像。然后你将能够在卸载它之前修改修改磁盘镜像的内容。
<!--more-->
在 Linux 上，有多种方式挂载磁盘镜像，不同类型的磁盘镜像需要不同的方法。如果你在使用 qcow2 类型的磁盘镜像（ QEMU/KVM 使用的），在 Linux 上至少有两种方式挂载它们。

# 方法一：libguestfs

挂载 qcow2 磁盘镜像的第一种方法是使用 ***libguestfs***，它提供了一系列工具来访问和编辑 VM 磁盘镜像。***libguestfs*** 支持几乎所有类型的磁盘镜像，包括 ***qcow2***。你可以像下面这样，在 Linux 上安装 ***libguestfs*** 工具集。

在基于 Debian 的系统上：

```
$ sudo apt-get install libguestfs-tools
```

在基于 Red Hat 的系统上：

```
$ sudo yum install libguestfs-tools
```

一旦 ***libguestfs*** 安装完成，你可以像下面这样使用称为 ***guestmount*** 的命令行工具挂载一个 ***qcow2*** 磁盘镜像。注意，当 VM 运行时，你一定不能以 "read-write" 模式挂载它的磁盘镜像。否则，你就有损坏磁盘镜像的风险。这样，在挂载 VM 的磁盘镜像关闭它总是安全的。

***guestmount*** 的全部命令行参数选项如下：

```
$ guestmount --help
guestmount: FUSE module for libguestfs
guestmount lets you mount a virtual machine filesystem
Copyright (C) 2009-2016 Red Hat Inc.
Usage:
  guestmount [--options] mountpoint
Options:
  -a|--add image       Add image
  -c|--connect uri     Specify libvirt URI for -d option
  --dir-cache-timeout  Set readdir cache timeout (default 5 sec)
  -d|--domain guest    Add disks from libvirt guest
  --echo-keys          Don't turn off echo for passphrases
  --fd=FD              Write to pipe FD when mountpoint is ready
  --format[=raw|..]    Force disk format for -a option
  --fuse-help          Display extra FUSE options
  -i|--inspector       Automatically mount filesystems
  --help               Display help message and exit
  --keys-from-stdin    Read passphrases from stdin
  --live               Connect to a live virtual machine
  -m|--mount dev[:mnt[:opts[:fstype]] Mount dev on mnt (if omitted, /)
  --no-fork            Don't daemonize
  -n|--no-sync         Don't autosync
  -o|--option opt      Pass extra option to FUSE
  --pid-file filename  Write PID to filename
  -r|--ro              Mount read-only
  --selinux            Enable SELinux support
  -v|--verbose         Verbose messages
  -V|--version         Display version and exit
  -w|--rw              Mount read-write
  -x|--trace           Trace guestfs API calls
```

我们可以像下面这样挂载一个 qcow2 格式的磁盘镜像：

```
$ sudo guestmount -a /path/to/qcow2/image -m <device> /path/to/mount/point
```

"-m <device>" 用于指定磁盘镜像内，你想要挂载的分区（比如，/dev/sda1）。如果你不确定磁盘镜像内有什么分区，你可以任意提供一个无效的设备名。***guestmount*** 工具将为你展示所有你可以选择的设备名字。如：

```
$ sudo guestmount  -a sdcard.img.qcow2 -m /dev/sdaqw qcow2_mount_point
libguestfs: error: mount_options: mount_options_stub: /dev/sdaqw: No such file or directory
guestmount: '/dev/sdaqw' could not be mounted.
guestmount: Did you mean to mount one of these filesystems?
guestmount: 	/dev/sda (vfat)
```

在这个例子中，磁盘镜像文件中可选的磁盘设备只有 `/dev/sda`，文件系统为 `vfat`。

比如，要挂载磁盘镜像 `userdata-qemu.img.qcow2` 内的 `/dev/sda`，挂载点为为 `qcow2_mount_point`，则执行如下命令：

```
$ sudo guestmount  -a userdata-qemu.img.qcow2 -m /dev/sda qcow2_mount_point
```

默认情况下，磁盘镜像将以 "read-write" 模式挂载。因此在挂载之后你可以修改 `qcow2_mount_point` 目录下的任何文件。

如果你想要以 "read-only" 模式挂载它，则：

```
$ sudo guestmount  -a userdata-qemu.img.qcow2 -m /dev/sda --ro qcow2_mount_point
```

要卸载它，则执行：

```
$ sudo guestunmount qcow2_mount_point
```

注：***上面挂载的是 Android 模拟器生成的 qcow2 文件。尽管挂载可以成功完成，但挂载之后，挂载点无法访问。***

# 方法二：qemu-nbd

另一种挂载 ***qcow2*** 磁盘镜像的方法是通过 `qemu-nbd`，一个命令行工具，将一个磁盘镜像导出为 "network block device (nbd)"。

你可以像下面这样在 Linux 上安装 `qemu-nbd`。

在基于 Debian 的系统上：

```
$ sudo apt-get install qemu-utils
```

在基于 Red Hat 的系统上：

```
$ sudo yum install qemu-img
```

要挂载 ***qcow2*** 磁盘镜像，首先要把镜像导出到 `nbd`，像这样：

```
$ sudo modprobe nbd max_part=8
$ sudo qemu-nbd --connect=/dev/nbd0 /path/to/qcow2/image
```

第一个命令加载 `nbd` 内核模块。"max_part=N" 选项指定我们想要通过 `nbd` 管理的分区的最大个数。第二个命令将特定的磁盘镜像导出为网络块设备（/dev/nbd0）。作为一个网络块设备，你可以使用  /dev/nbd0，/dev/nbd1，/dev/nbd2，等等中任意没有在使用的。至于磁盘镜像，要确保指定它的 “完整” 路径。

比如，要将镜像 `userdata-qemu.img.qcow2` 导出为 `/dev/nbd0`：

```
$ sudo qemu-nbd --connect=/dev/nbd0 userdata-qemu.img.qcow2
```

此后，如果磁盘镜像中有多个分区的话，磁盘镜像中已有的分区将被映射为 /dev/nbd0p1，/dev/nbd0p2，/dev/nbd0p3 等等，磁盘本身则被映射为 `/dev/nbd0`。

要检查 `nbd` 映射的分区列表，则使用 `fdisk`：

```
$ sudo fdisk /dev/nbd0 -l
```

对于 Android QEMU 的磁盘镜像，它的整个镜像就是一个文件系统，而没有分区的情况，执行上面的命令将报错：

```
$ sudo fdisk /dev/nbd0  -l
fdisk: 打不开 /dev/nbd0: 对设备不适当的 ioctl 操作
```

最后，如果磁盘镜像中存在多个分区，选择任何一个分区（比如，/dev/nbd0p1）并把它挂载到一个本地挂载点（比如，qcow2_mount_point），如果是像 QEMU 的磁盘镜像那样，整个镜像就是一个文件系统，则挂载整个 nbd（比如 /dev/nbd0）。

```
$ sudo mount /dev/nbd0 qcow2_mount_point
```

现在你就可以通过 `qcow2_mount_point` 挂载点访问并修改磁盘镜像的挂载的分区的内容了。

一旦完成了操作，则可以卸载它，并断开磁盘镜像，就像下面这样。

```
$ sudo umount qcow2_mount_point/
$ sudo qemu-nbd --disconnect /dev/nbd0 
/dev/nbd0 disconnected
```

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。

[原文](http://ask.xmodulo.com/mount-qcow2-disk-image-linux.html)

参考资料：
[The QCOW2 Image Format](https://people.gnome.org/~markmc/qcow-image-format.html)
[network block device(nbd)](https://yq.aliyun.com/articles/17222)
[在宿主机挂载客户机虚拟磁盘文件](http://www.xiyang-liu.com/2012/11/18/mount-guest-filesystem-in-host/)
[Modify images](https://docs.openstack.org/image-guide/modify-images.html)
