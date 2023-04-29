---
title: 为 Android 模拟器编译定制版内核
date: 2023-04-22 13:51:29
categories: Linux 内核
tags:
- Linux 内核
- 自动驾驶
---

## 构建内核

官方构建指南：[https://source.android.com/setup/build/building-kernels](https://source.android.com/setup/build/building-kernels)

这些说明可以指导你完成选择正确的源代码，构建内核，以及将结果嵌入到从Android 开放源代码项目 (AOSP) 构建的系统映像的过程。

## 查找内核 Makefile

Android 目录树只包含了预编译的内核二进制文件。当编译 AOSP 系统镜像时，它将把预编译的内核镜像复制到输出目录。这里有一个 Python 脚本 *show_make_tree.py*，可以以树的形式列出一个 makefile 文件包含的所有 Makefiles：
```
#!/usr/bin/python3

import sys
import os
from os.path import exists

__FILE    = "\u001b[32mFile   : " # green
__INHERIT = "\u001b[33mInherit: " # yellow
__DUPLICATE = "\u001b[33mDuplica: " # yellow
__SEARCH  = "\u001b[31mFound  : " # read
__INCLUDE = "\u001b[36mInclude: " # cyan
__NONE    = "\u001b[0m"           # reset


def print_usage():
    print(__SEARCH, "Usage: %s $android_root $root_mk_file $target" % sys.argv[0])
    print(__NONE)


def indent(level, type, line, linenum = -1):
    print(type, end="")
    for _ in range(level):
        print("  ", end="")
    print(line, end="")
    if linenum != -1:
        print(" (%d)" % linenum, end="")
    print(__NONE)


def find(android_root, mk_file, level, type, search_target, mk_files):
    file = android_root + os.path.sep + mk_file
    if exists(file):
        if file in mk_files:
            indent(level, __DUPLICATE, file)
            return
        else:
            mk_files.add(file)
        indent(level, type, file)
        with open(file, "r") as f:
            for index, line in enumerate(f.readlines()):
                line = line.strip()
                if line.startswith("$(call inherit-product"):
                    line = line.replace("$(SRC_TARGET_DIR)", "build/target")
                    line = line.split(",")[1]
                    line = line[:-1]
                    line = line.strip()
                    
                    find(android_root, line, level+1, __INHERIT, search_target, mk_files)
                if line.startswith("include"):
                    line = line.split(" ")[1]
                    line = line.strip()
                    find(android_root, line, level+1, __INCLUDE, search_target, mk_files)
                if search_target in line:
                    indent(level+1, __SEARCH, line, index + 1)


if len(sys.argv) >= 4:
    android_root = sys.argv[1]
    root_mk_file = sys.argv[2]
    search_target = sys.argv[3]

    mk_files = set()

    print("Search for\n\u001b[31m" + search_target + "\u001b[0m\nin");
    find(android_root, root_mk_file, 0, __FILE, search_target, mk_files)
else:
    print_usage()
```

一个 mk 文件可以通过 `call inherit-product`、`call inherit-product-if-exists` 和 `include` 等命令引入其它 mk 文件，被引入的 mk 文件用该文件相对于 android 源码根目录的相对路径表示。上面的脚本解析 mk 文件的内容及其引入的其它 mk 文件的内容查找目标。

对于编译目标 **sdk_car_x86_64**，可以这样来找到对应的根 mk 文件：
```
android-security-12.0.0_r43$ find device -name sdk_car_x86_64.mk 
device/generic/goldfish/car/sdk_car_x86_64.mk
```

在目标 Makefile 文件 *device/generic/goldfish/car/sdk_car_x86_64.mk* 中搜索 `:kernel`：
```
$ python3 show_make_tree.py /media/data2/androids/android-security-12.0.0_r43 device/generic/goldfish/car/sdk_car_x86_64.mk :kernel
Search for
:kernel
in
File   : device/generic/goldfish/car/sdk_car_x86_64.mk
 . . . . . .
Inherit:   build/target/product/sdk_x86_64.mk
Inherit:     build/target/product/sdk_phone_x86_64.mk
 . . . . . .
Inherit:       device/generic/goldfish/x86_64-vendor.mk
Include:         device/generic/goldfish/x86_64-kernel.mk
Found  :         $(EMULATOR_KERNEL_FILE):kernel-ranchu (15)
```

在 *device/generic/goldfish/x86_64-vendor.mk* 文件中可以找到 `:kernel`：
```
include device/generic/goldfish/x86_64-kernel.mk

PRODUCT_PROPERTY_OVERRIDES += \
       vendor.rild.libpath=/vendor/lib64/libgoldfish-ril.so

# This is a build configuration for a full-featured build of the
# Open-Source part of the tree. It's geared toward a US-centric
# build quite specifically for the emulator, and might not be
# entirely appropriate to inherit from for on-device configurations.
PRODUCT_COPY_FILES += \
    device/generic/goldfish/data/etc/config.ini.xl:config.ini \
    device/generic/goldfish/data/etc/advancedFeatures.ini:advancedFeatures.ini \
    device/generic/goldfish/data/etc/encryptionkey.img:encryptionkey.img \
    device/generic/goldfish/task_profiles.json:$(TARGET_COPY_OUT_VENDOR)/etc/task_profiles.json \
    $(EMULATOR_KERNEL_FILE):kernel-ranchu
```

这个文件中将路径为 *$(EMULATOR_KERNEL_FILE)* 的文件拷贝为内核镜像 *kernel-ranchu*。*device/generic/goldfish/x86_64-vendor.mk* 文件包含了 *device/generic/goldfish/x86_64-kernel.mk*，后者的内容如下：
```
TARGET_KERNEL_USE ?= 5.10

KERNEL_MODULES_PATH := kernel/prebuilts/common-modules/virtual-device/$(TARGET_KERNEL_USE)/x86-64

KERNEL_MODULES_EXCLUDE := \
    $(KERNEL_MODULES_PATH)/virt_wifi.ko \
    $(KERNEL_MODULES_PATH)/virt_wifi_sim.ko

BOARD_VENDOR_RAMDISK_KERNEL_MODULES += \
    $(filter-out $(KERNEL_MODULES_EXCLUDE), $(wildcard $(KERNEL_MODULES_PATH)/*.ko))

EMULATOR_KERNEL_FILE := kernel/prebuilts/$(TARGET_KERNEL_USE)/x86_64/kernel-$(TARGET_KERNEL_USE)
```

这就很清楚了，预编译的内核文件版本为 `5.10`：
```
kernel/prebuilts/5.10/x86_64/kernel-5.10
```

将被拷贝到：
```
out/target/product/emulator_car_x86_64/kernel-ranchu
```

## Android 通用内核

[Android 通用内核](https://source.android.com/devices/architecture/kernel/android-common) 用于在模拟器 Emulator 中运行。

### KMI 内核分支

Android 11 引入了 GKI，它把内核分为 Google 维护的内核镜像和供应商维护的模块，它们分别构建。

Android 11：
 * android11-5.4

Android 12：
 * android12-5.4
 * android12-5.10

### 遗留 dessert 内核分支

遗留 dessert 内核的创建是为了保证新功能的开发不会干扰从 Android 公共内核的合并。

Android 10：
 * android-4.9-q
 * android-4.14-q
 * android-4.19-q

Android 11：
 * android-4.14-stable
 * android-4.19-stable

### 遗留发布内核分支

维护发布内核以提供每月 Android 安全公告中引用的补丁的后端口。当有一个新的Android 平台发布时，它们是为每个启动内核创建的。

Android 10：
 * android-4.9-q-release
 * android-4.14-q-release
 * android-4.19-q-release

## 构建定制内核

### 下载内核及目标分支

上面的示例使用了 AOSP `android-security-12.0.0_r43`，它使用了内核 `5.10`。因此，我们可以选择 Android 通用内核的分支 `common-android12-5.10`。
```
export KERNEL_BRANCH="common-android12-5.10"
mkdir kernel-$KERNEL_BRANCH && cd kernel-$KERNEL_BRANCH
repo init \
    -u https://android.googlesource.com/kernel/manifest \
    -b $KERNEL_BRANCH \
    --depth=1
```

然后同步源代码：
```
$ repo sync -c -j$(nproc) && sync
```

### 构建内核

为构建内核安装如下 libs：
```
$ sudo apt install libssl-dev libelf-dev
```

通用内核是通用的，可定制的内核，因此不定义默认配置。我们需要设置一些环境变量：

| 环境变量 | 描述 | 示例 |
|----|---|----|
| `BUILD_CONFIG` | 初始化构建环境的构建配置文件。位置必须定义为相对于 Repo 根目录的相对路径。默认为 build.config。对于通用内核是必选项。 | `BUILD_CONFIG=common/build.config.<target>.x86_64` |
| `CC` | 覆盖正在使用的编译器。回退到 build.config 定义的默认编译器。 | `CC=clang` |
| `DIST_DIR` | 内核发行版的基本输出目录。 | `DIST_DIR=/path/to/my/dist` |
| `OUT_DIR` | 内核构建的基本输出目录。 | `OUT_DIR=/path/to/my/out` |
| `SKIP_DEFCONFIG` | 跳过 make defconfig | `SKIP_DEFCONFIG=1` |
| `SKIP_MRPROPER` | 跳过 make mrproper | `SKIP_MRPROPER=1` |

**bzImage** 和 **vmlinux**

 * Big Z 镜像 `bzImage` 是 `vmlinux` 的压缩版本，它携带有额外的用于启动的文件头。
 * `vmlinux` 是包含 Linux 内核的静态链接可执行文件。

Android 源码库中带的预编译内核二进制文件为 **bzImage** 文件，具体的版本号为 **5.10.43-android12-9-00031-g5b153f189546-ab7618735**，如：
```
android-security-12.0.0_r43$ file out/target/product/emulator_x86_64/kernel-ranchu 
out/target/product/emulator_x86_64/kernel-ranchu: Linux kernel x86 boot executable bzImage, version 5.10.43-android12-9-00031-g5b153f189546-ab7618735 (build-user@build-host) #1 SMP PREEMPT Fri Aug 6 17:53:30 UTC 2021, RO-rootFS, swap_dev 0X15, Normal VGA
```

自 Android 11，Google 引入了 GKI，它把内核分为 Google 维护的内核镜像和供应商维护的模块，它们分别构建。

**构建内核** (为了测试，没有开链接时优化 (LTO))：
```
$ BUILD_CONFIG=common/build.config.gki.x86_64 \
LTO=none \
build/build.sh
 . . . . . .
========================================================
 Installing kernel modules into staging directory
  INSTALL drivers/virtio/virtio_mem.ko
  DEPMOD  5.10.168-android12-9-g7771fe887f13
========================================================
 Generating test_mappings.zip
========================================================
 Copying files
  System.map
  arch/x86/boot/bzImage
  modules.builtin
  modules.builtin.modinfo
  vmlinux
  vmlinux.symvers
========================================================
 Installing UAPI kernel headers:
make: Entering directory '/media/data/androids/kernel-common-android12-5.10/out/android12-5.10/common'
  INSTALL /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/kernel_uapi_headers/usr/include
make: Leaving directory '/media/data/androids/kernel-common-android12-5.10/out/android12-5.10/common'
 Copying kernel UAPI headers to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist/kernel-uapi-headers.tar.gz
========================================================
 Copying kernel headers to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist/kernel-headers.tar.gz
/media/data/androids/kernel-common-android12-5.10/common /media/data/androids/kernel-common-android12-5.10
/media/data/androids/kernel-common-android12-5.10
========================================================
 Copying modules files
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_mem.ko
========================================================
 Files copied to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist
```

构建完成后，检查内核文件：
```
kernel-common-android12-5.10$ ls out/android12-5.10/dist 
abi.prop  kernel-headers.tar.gz       modules.builtin          System.map         virtio_mem.ko  vmlinux.symvers
bzImage   kernel-uapi-headers.tar.gz  modules.builtin.modinfo  test_mappings.zip  vmlinux
kernel-common-android12-5.10$ file out/android12-5.10/dist/bzImage  
out/android12-5.10/dist/bzImage: Linux kernel x86 boot executable bzImage, version 5.10.168-android12-9-g7771fe887f13 (build-user@build-host) #1 SMP PREEMPT Thu Apr 20 15:10:08 UTC 2023, RO-rootFS, swap_dev 0X10, Normal VGA
```

**构建供应商模块**，Android 12 Cuttlefish 和 Goldfish 趋同，因此它们共享相同的内核 `virtual_device`，它们可以用下面的命令构建：
```
BUILD_CONFIG=common-modules/virtual-device/build.config.virtual_device.x86_64 \
LTO=none \
build/build.sh
 . . . . . .
========================================================
 Copying files
  Makefile
  Module.symvers
  System.map
  arch is not a file, skipping
  block is not a file, skipping
  certs is not a file, skipping
  crypto is not a file, skipping
  defconfig
  drivers is not a file, skipping
  fs is not a file, skipping
  host_tools is not a file, skipping
  include is not a file, skipping
  init is not a file, skipping
  io_uring is not a file, skipping
  ipc is not a file, skipping
  kernel is not a file, skipping
  lib is not a file, skipping
  mm is not a file, skipping
  modules-only.symvers
  modules.builtin
  modules.builtin.modinfo
  modules.order
  net is not a file, skipping
  scripts is not a file, skipping
  security is not a file, skipping
  sound is not a file, skipping
  source is not a file, skipping
  test_mapping_files.txt
  tools is not a file, skipping
  usr is not a file, skipping
  virt is not a file, skipping
  vmlinux
  vmlinux.o
  vmlinux.symvers
========================================================
 Installing UAPI kernel headers:
make: Entering directory '/media/data/androids/kernel-common-android12-5.10/out/android12-5.10/common'
  INSTALL /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/kernel_uapi_headers/usr/include
make: Leaving directory '/media/data/androids/kernel-common-android12-5.10/out/android12-5.10/common'
 Copying kernel UAPI headers to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist/kernel-uapi-headers.tar.gz
========================================================
 Copying kernel headers to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist/kernel-headers.tar.gz
/media/data/androids/kernel-common-android12-5.10/common /media/data/androids/kernel-common-android12-5.10
/media/data/androids/kernel-common-android12-5.10
========================================================
 Copying modules files
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/virtio_gpu/virtio-gpu.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/wlan_simulation/virt_wifi_sim.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/virtio_snd/virtio_snd.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/goldfish_drivers/goldfish_battery.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/goldfish_drivers/goldfish_pipe.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/goldfish_drivers/goldfish_sync.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/extra/goldfish_drivers/goldfish_address_space.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/mm/zsmalloc.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/hda/snd-hda-core.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/hda/snd-intel-dspcfg.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/hda/snd-hda-codec.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/hda/snd-hda-codec-realtek.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/hda/snd-hda-intel.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/hda/snd-hda-codec-generic.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/snd-intel8x0.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/pci/ac97/snd-ac97-codec.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/sound/ac97_bus.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/crypto/lzo.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/crypto/lzo-rle.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/media/cec/usb/pulse8/pulse8-cec.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/bluetooth/btintel.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/bluetooth/hci_vhci.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/bluetooth/btusb.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/bluetooth/btrtl.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/cpufreq/dummy-cpufreq.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/input/mouse/psmouse.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/nvdimm/nd_virtio.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/nvdimm/virtio_pmem.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/usb/usbip/usbip-core.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/usb/usbip/vhci-hcd.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_mem.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_dma_buf.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_mmio.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_pci.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_input.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/virtio/virtio_balloon.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/char/hw_random/virtio-rng.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/char/virtio_console.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/char/tpm/tpm.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/char/tpm/tpm_vtpm_proxy.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/rtc/rtc-test.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/leds/trigger/ledtrig-audio.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/gnss/gnss-serial.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/gnss/gnss-cmdline.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/md/md-mod.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/block/zram/zram.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/block/virtio_blk.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/dma-buf/heaps/system_heap.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/net_failover.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/virtio_net.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/wireless/virt_wifi.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/wireless/mac80211_hwsim.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/can/vcan.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/can/slcan.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/drivers/net/can/usb/gs_usb.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/lib/test_meminit.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/lib/test_stackinit.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/net/mac80211/mac80211.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/net/core/failover.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/net/wireless/cfg80211.ko
  lib/modules/5.10.168-android12-9-g7771fe887f13/kernel/net/vmw_vsock/vmw_vsock_virtio_transport.ko
========================================================
 Creating initramfs

========================================================
 Files copied to /media/data/androids/kernel-common-android12-5.10/out/android12-5.10/dist
```

**快速重新构建**。默认情况下，内核总是以 `mrproper` 目标来构建，它将 *移除所有生成的文件 + config + 各种各样的备份文件* 来创建一个干净的构建。

当开发一个或很少的几个模块时，我们可以跳过一些初始化步骤并立即启动重新构建：
```
BUILD_CONFIG=common-modules/virtual-device/build.config.virtual_device.x86_64 \
LTO=none \
FAST_BUILD=1 \
SKIP_MRPROPER=1 \
SKIP_DEFCONFIG=1 \
build/build.sh
```

## 包含定制内核

要给内核创建一个供应商模块，请参考 [Kernel Module](https://www.codeinsideout.com/blog/android/kernel-module/) 指南。

在 AOSP 根目录下创建一个指向构建的定制内核的软链接：
```
android-security-12.0.0_r43$ ln -s /media/data/androids/kernel-common-android12-5.10 kernel-custom
```

编辑内核 make 文件 (*device/generic/goldfish/x86_64-kernel.mk*)：
```
TARGET_KERNEL_USE ?= 5.10

KERNEL_MODULES_PATH := kernel-custom/out/android12-$(TARGET_KERNEL_USE)/dist

KERNEL_MODULES_EXCLUDE := \
    $(KERNEL_MODULES_PATH)/virt_wifi.ko \
    $(KERNEL_MODULES_PATH)/virt_wifi_sim.ko

BOARD_VENDOR_RAMDISK_KERNEL_MODULES += \
    $(filter-out $(KERNEL_MODULES_EXCLUDE), $(wildcard $(KERNEL_MODULES_PATH)/*.ko))

EMULATOR_KERNEL_FILE := $(KERNEL_MODULES_PATH)/bzImage
```

改动主要在 `KERNEL_MODULES_PATH` 和 `EMULATOR_KERNEL_FILE` 这两个变量的定义，它们分别指向要包含的内核模块的路径，和内核镜像文件的路径。

最后，再次 make 系统镜像：
```
$ m all -j$(nproc)
```

然后运行模拟器：
```
$ emulator -no-snapshot -verbose -show-kernel -writable-system
```

使用 `adb` 访问 Emulator shell 并检查内核版本。
```
$ adb root
$ adb remount
$ adb shell
sdk_car_emu_x86_64:/ # uname -r                                                                                                                                                                                  
5.10.168-android12-9-g7771fe887f13
```

运行的内核已经是我们编译出来的，而不是 AOSP 包含的那个预编译的版本了。

参考文档：
[Build Android system and Kernel images](https://www.codeinsideout.com/blog/android/build-aosp/)

Done.
