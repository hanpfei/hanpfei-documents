---
title: Android AVD
date: 2017-11-21 19:05:49
categories: 虚拟化
tags:
- 虚拟化
- Android开发
---

# Android AVD 的文件

Android AVD 的文件由两大部分，一部分是由所有的 AVD 共享的 AVD 系统文件；另一部分是每个 AVD 特有的 AVD 数据文件。
<!--more-->
AVD 系统文件位于 AVD 系统目录下，其中包含了模拟器用于仿真操作系统的 Android 系统镜像。它具有平台特有的、只读的为所有相同类型的 AVD 共享的文件，包括 API level，CPU 架构，和 Android variant。默认的位置如下：

 * Mac OS X 和 Linux - ~/Library/Android/sdk/system-images/android-***apiLevel***/***variant***/***arch***/
 * Microsoft Windows XP - C:\Documents and Settings\\***user***\Library\Android\sdk\system-images\android-***apiLevel***\\***variant***\\***arch***\
 * Windows Vista - C:\Users\\***user***\Library\Android\sdk\system-images\android-***apiLevel***\\***variant***\\***arch***\

其中：

 * ***apiLevel*** 是数字 API level。
 * ***variant*** 是一个名字，对应于系统镜像实现的特定功能；比如，***google_apis*** 或 ***android-wear***。
 * ***arch*** 是目标 CPU 架构；比如，x86。

使用 `-sysdir` 选项来为 AVD 指定一个不同的系统目录。

AVD 系统目录下包含如下这些 AVD 系统文件：

```
kernel-qemu
kernel-ranchu
system.img
userdata.img
ramdisk.img

advancedFeatures.ini

build.prop
NOTICE.txt
package.xml
source.properties
data/
```

这些文件又分为三类，第一类是系统镜像文件；第二类是模拟器选项配置文件；第三类是与系统镜像本身有关的其它文件。

第一类系统镜像文件主要包括如下这些：

| 文件 | 描述 | 指定一个不同的文件的选项|
| --- | ---- | ---- |
| kernel-qemu 或 kernel-ranchu | AVD 的二进制内核镜像。`kernel-ranchu` 是 QEMU 2 模拟器，即最新版本 | `-kernel` |
| system.img | 系统镜像的只读初始版本；特别地，该分区包含对应于 API level 和 variant 的系统库和数据 | `-system` |
| ramdisk.img | 启动分区镜像。它是在系统镜像挂载之前，内核最初加载的 `system.img` 的子集。它典型地只包含一些二进制文件和初始化脚本 | `-ramdisk` |
| userdata.img | `data` 分区的 *初始* 版本，在模拟的系统中它以 `data/` 呈现，并包含 AVD 所有可写的数据。当你创建一个新的 AVD 或使用 `‑wipe-data` 选项时，模拟器使用该文件。更多信息，请参考后面描述的 `userdata-qemu.img` 文件 | `-initdata` 或 `-init-data` |

第二类模拟器选项配置文件包括如下这些：

 * advancedFeatures.ini：高级功能选项配置文件
 
 第三类与系统镜像本身有关的其它文件包括如下这些：
 
 * build.prop：系统镜像 `system.img`中包含有 `build.prop` 文件。这里的 `build.prop` 文件，与客户 Android 系统使用的 `build.prop` 无关。
 * NOTICE.txt：文件系统镜像文件的注意事项
 * package.xml：Android SDK 许可证协议文件
 * source.properties：该文件主要保存了系统镜像下载相关的信息
 * data/：无用
 
Android AVD 的另一部分文件为 AVD 特有的文件，它们位于 AVD 数据目录，也被称为内容目录，它们包括 AVD 所有可修改的数据。

默认的位置如下，其中 ***name*** 是 AVD 的名字：

 * Mac OS X 和 Linux - ~/.android/avd/name.avd/
 * Microsoft Windows XP - C:\Documents and Settings\\***user***\.android\\***name***.avd\
 * Windows Vista, and higher - C:\Users\\***user***\.android\\***name***.avd\

使用 `-datadir` 选项来指定一个不同的 AVD 数据目录。

***用户数据隔离设计：为每个用户创建单独的用户数据目录；在用户数据目录下，创建 AVD 的数据目录；为用户启动 AVD 时，通过 `-datadir`选项来指定 AVD 数据目录的路径。***

在 AVD 刚刚创建好时，AVD 数据目录包含如下这些文件：
```
config.ini
sdcard.img
userdata.img
```

创建 AVD 时，伴随 AVD 数据目录一起，且位于系统 AVD 管理目录下，还会有一个 AVD 的信息文件被创建出来，如 `~/.android/avd/Nexus_5_API_25.ini`，其内容如下：
```
avd.ini.encoding=UTF-8
path=/home/hanpfei0306/.android/avd/Nexus_5_API_25.avd
path.rel=avd/Nexus_5_API_25.avd
target=android-25
```

这个文件描述了 AVD 数据目录等信息。AVD 数据目录下，在创建 AVD 时，也会有一个配置文件 `config.ini` 被创建出来，这个文件描述了 AVD 的硬件配置等信息，模拟器在启动 AVD 时将加载这个文件，以为 AVD 虚拟适当的硬件等。

随着 AVD 的运行，AVD 数据目录下将会有一些新的文件创建出来。最终将包括如下这些文件：
```
config.ini
emulator-user.ini
hardware-qemu.ini
cache.img
cache.img.qcow2
sdcard.img
sdcard.img.qcow2
system.img.qcow2
userdata.img
userdata-qemu.img
userdata-qemu.img.qcow2
version_num.cache
```

最重要的是一些镜像文件，如下：

| 文件 | 描述 | 指定一个不同的文件的选项|
| --- | ---- | ---- |
| userdata-qemu.img | `data` 分区的内容，在模拟的系统中，它以 `data/` 呈现。当你创建一个新的 AVD，或当你使用 `-wipe-data` 选项复位 AVD 为出厂状态时，模拟器拷贝系统目录下的 `userdata.img` 文件来创建该文件。    每个虚拟的设备实例使用一个可写的用户数据镜像来存储用户和会话特有的数据。比如，它使用镜像存储唯一的用户的已安装应用数据，设置，数据库和文件。每个用户具有一个不同的 `ANDROID_SDK_HOME` 目录，其为用户创建的 AVDs 存储数据目录；每个 AVD 具有一个单独的 `userdata-qemu.img` 文件 | `-data` |
| cache.img | `cache` 分区镜像，在模拟的系统中，它以 `cache/` 呈现。当你首次创建一个 AVD 或使用 `-wipe-data` 选项时它是空的。它存储临时下载的文件，由下载管理器，而有时是系统放置，比如，在模拟器运行期间浏览器使用它缓存下载的 web 页和图像。当你关闭虚拟设备时，文件被删除。你可以通过使用 `-cache` 选项来持久化文件。 | `-cache` |
| sdcard.img | （可选的）一个 SD 卡分区镜像，让你在一个虚拟设备上模拟一个 SD 卡。你可以在 AVD Manager 中，或使用`mksdcard` 工具创建一个 SD 卡镜像文件。这个文件存储在你的开发电脑上，且必须在启动时加载。  当在 AVD Manager 中定义一个 AVD 时，有选项让你选择自动管理的 SD 卡文件，或你通过 `mksdcard` 工具创建的文件。你可以在 AVD Manager 中查看与 AVD 关联的 `sdcard.img` 文件。`-sdcard` 选项覆盖 AVD 中指定的 SD 卡文件。    你可以在虚拟设备运行的时候，使用模拟器 UI 或 adb 工具来浏览，发送文件到仿真的 SD，从仿真的 SD 卡拷贝和移除文件。你不能从一个运行中的虚拟设备中移除仿真的 SD 卡。   要在 SD 卡文件挂载之前向它拷贝文件，你可以把镜像文件挂载为一个 loop 设备，然后拷贝文件。或者使用实用程序，比如 `mtools` 包直接拷贝文件到镜像。    模拟器将文件作为字节的 pool，因而 SD 卡的格式无关紧要。   注意 `-wipe-data` 选项选项不影响这个文件。如果你想要清除这个文件，你需要删除文件并使用 AVD Manager 或 `mksdcard` 工具重建它。改变文件的大小也将删除文件并重建一个新的。 | `-sdcard` |

***AVD 数据目录下为什么要有一份 userdata.img 文件呢？***

# 客户 Android 系统中的分区与 AVD 的磁盘镜像文件之间的对应关系

| 客户 Android 系统中的分区 | 磁盘镜像文件 |
| ---------------------------------- | ------------------|
| /cache | cache.img |
| /cache | cache.img.qcow2 |
| /system | system.img |
| /system | system.img.qcow2 |
| /data | userdata.img |
| /data | userdata-qemu.img |
| /data | userdata-qemu.img.qcow2 |
| /sdcard | /data/media/0/ |

默认情况下，`sdcard.img` 和 `sdcard.img.qcow2` 似乎并没有呈现为模拟的系统中的 `/sdcard`。模拟的系统中的 `/sdcard` 目录实际上是 `/data/media/0/`，因而其中的文件是存储在 `userdata-qemu.img` 中的。

# 命令行创建及启动 AVD 的方法

Android SDK 提供了命令行工具 `android` 用于 SDK，AVD，和项目管理。它提供的所有功能，可以从它支持的命令看出来：
```
$ android list
*************************************************************************
The "android" command is deprecated.
For manual SDK, AVD, and project management, please use Android Studio.
For command-line tools, use tools/bin/sdkmanager and tools/bin/avdmanager
*************************************************************************
Invalid or unsupported command "list"

Supported commands are:
android list target
android list avd
android list device
android create avd
android move avd
android delete avd
android list sdk
android update sdk
```

命令行工具 `android` 是位于 `Sdk/tools/` 目录下的一个脚本。AVD 管理相关的功能通过另一个脚本，即位于 `Sdk/tools/bin/` 目录下的 `avdmanager` 来完成。`avdmanager` 实际上是 Java 工具的一个包装，它最终执行的命令类似下面这样：
```
java -Dcom.android.sdkmanager.toolsdir=/media/data/dev_tools/Android/Sdk/tools -classpath /media/data/dev_tools/Android/Sdk/tools/lib/dvlib-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/jimfs-1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/jsr305-1.3.9.jar:/media/data/dev_tools/Android/Sdk/tools/lib/repository-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/j2objc-annotations-1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/layoutlib-api-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/gson-2.3.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpcore-4.2.5.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-logging-1.1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-compress-1.12.jar:/media/data/dev_tools/Android/Sdk/tools/lib/annotations-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/error_prone_annotations-2.0.18.jar:/media/data/dev_tools/Android/Sdk/tools/lib/animal-sniffer-annotations-1.14.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpclient-4.2.6.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-codec-1.6.jar:/media/data/dev_tools/Android/Sdk/tools/lib/common-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/kxml2-2.3.0.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpmime-4.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/annotations-12.0.jar:/media/data/dev_tools/Android/Sdk/tools/lib/sdklib-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/guava-22.0.jar com.android.sdklib.tool.AvdManagerCli list device
```

其中该 Java 应用的入口类 `com.android.sdklib.tool.AvdManagerCli` 位于 `/media/data/dev_tools/Android/Sdk/tools/lib/sdklib-26.0.0-dev.jar` 中。这个 jar 包的源码位于 Android 源码库中。

我们通过如下命令下载 AOSP 的 studio-master-dev branch：
```
$ mkdir studio-master-dev
$ cd studio-master-dev
$ repo init -u https://android.googlesource.com/platform/manifest -b studio-master-dev
$ repo sync -j8
```

sdklib-26.0.0-dev.jar 的源码位于 `studio-master-dev/tools/base/sdklib`。切换到 `studio-master-dev/tools` 目录下，我们可以通过 gradle 编译源码。
```
$ cd tools
$ ./gradlew assemble
```

最终生成的 jar 包将位于 `studio-master-dev/out/build/base/sdklib/build/libs/` 目录下。

关于 AVD 创建，avdmanager，即 sdklib 提供了如下功能：
```
$ android create avd
*************************************************************************
The "android" command is deprecated.
For manual SDK, AVD, and project management, please use Android Studio.
For command-line tools, use tools/bin/sdkmanager and tools/bin/avdmanager
*************************************************************************
Running /media/data/dev_tools/Android/Sdk/tools/bin/avdmanager create avd

Error: The parameter --name must be defined for action 'create avd'

Usage:
      avdmanager [global options] create avd [action options]
      Global options:
  -s --silent     : Silent mode, shows errors only.
  -v --verbose    : Verbose mode, shows errors, warnings and all messages.
     --clear-cache: Clear the SDK Manager repository manifest cache.
  -h --help       : Help on a specific command.

Action "create avd":
  Creates a new Android Virtual Device.
Options:
  -a --snapshot: Place a snapshots file in the AVD, to enable persistence.
  -c --sdcard  : Path to a shared SD card image, or size of a new sdcard for
                 the new AVD.
  -g --tag     : The sys-img tag to use for the AVD. The default is to
                 auto-select if the platform has only one tag for its system
                 images.
  -p --path    : Directory where the new AVD will be created.
  -k --package : Package path of the system image for this AVD (e.g.
                 'system-images;android-19;google_apis;x86').
  -n --name    : Name of the new AVD. [required]
  -f --force   : Forces creation (overwrites an existing AVD)
  -b --abi     : The ABI to use for the AVD. The default is to auto-select the
                 ABI if the platform has only one ABI for its system images.
  -d --device  : The optional device definition to use. Can be a device index
                 or id.
```

`-n` 参数用于指定 AVD 的名字。`-p` 参数用于指定 AVD 数据目录的路径，每个 AVD 特有的镜像文件等将保存在这个目录下，不指定时，AVD 数据将如前面提到的默认路径那样，位于系统 AVD 管理目录下。`-k` 参数用于指定 AVD 的系统镜像的路径，在不提供时，sdklib 会提供一个默认值。`-d` 参数用与指定要使用的设备定义，即 AVD 的硬件配置，不提供时，sdklib 将使用默认值，或逐个询问用户获得硬件配置，而具体可以为 AVD 做哪些硬件配置，决定于 `Sdk/emulator/lib/hardware-properties.ini` 。

sdklib 在创建 AVD 时，提供的获得硬件配置的方式只有如下简单粗暴的 3 种：

 * 全部采用默认的硬件配置。
 * 全部的硬件配置都通过询问获得
 * 全部的硬件配置都通过预先定义的一个硬件配置文件中获得。

对于第三种方式，硬件配置文件可以描述的硬件配置选项比较有限，比如 GPU 模式，内存大小，用户数据镜像大小等都无法描述。

在命令行中，比较理想的为创建的 AVD 配置硬件的方式，应该为，提供一个硬件配置文件，并通过另外的一个参数提供额外的一些配置信息。因此而扩展 sdklib 的功能，为 `com.android.sdklib.tool.AvdManagerCli` 添加对另外一个参数的支持，即 `-h` 参数，以支持从命令行中为创建 AVD
 提供额外的硬件配置。

最终在命令中创建 AVD 的方式将类似下面这样该：
```
java -Dcom.android.sdkmanager.toolsdir=/media/data/dev_tools/Android/Sdk/tools -classpath /media/data/dev_tools/Android/Sdk/tools/lib/dvlib-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/jimfs-1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/jsr305-1.3.9.jar:/media/data/dev_tools/Android/Sdk/tools/lib/repository-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/j2objc-annotations-1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/layoutlib-api-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/gson-2.3.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpcore-4.2.5.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-logging-1.1.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-compress-1.12.jar:/media/data/dev_tools/Android/Sdk/tools/lib/annotations-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/error_prone_annotations-2.0.18.jar:/media/data/dev_tools/Android/Sdk/tools/lib/animal-sniffer-annotations-1.14.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpclient-4.2.6.jar:/media/data/dev_tools/Android/Sdk/tools/lib/commons-codec-1.6.jar:/media/data/dev_tools/Android/Sdk/tools/lib/common-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/kxml2-2.3.0.jar:/media/data/dev_tools/Android/Sdk/tools/lib/httpmime-4.1.jar:/media/data/dev_tools/Android/Sdk/tools/lib/annotations-12.0.jar:/home/hanpfei0306/data/Androids/studio-master-dev/out/build/base/sdklib/build/libs/sdklib-26.0.0-dev.jar:/media/data/dev_tools/Android/Sdk/tools/lib/guava-22.0.jar com.android.sdklib.tool.AvdManagerCli create avd --name test1 -k "system-images;android-25;google_apis;arm64-v8a" -p /home/hanpfei0306/data/AndroidVirtualDevices/hanpfei1 -d GameEmuHardware -h hw.ramSize=2048,hw.gpu.enabled=yes,disk.dataPartition.size=10240M,hw.cpu.ncore=4,hw.gpu.mode=auto,hw.initialOrientation=landscape,hw.keyboard=yes,vm.heapSize=256
```

其中最为关键的是，`sdklib-26.0.0-dev.jar` 被替换为了我们自己编译的位于 `studio-master-dev/out/build/base/sdklib/build/libs/` 下的版本。

Android Studio 中，菜单栏的 Toos -> Android -> AVD Manager 相关的代码，同样位于 `studio-master-dev`，目测具体位置为 `studio-master-dev/tools/adt/idea/android/src/com/android/tools/idea/avdmanager`。

# QEMU 根据 AVD ID/名称搜索 AVD 相关文件的过程

QEMU 从 Android SDK 目录下查找 AVD 的系统镜像文件，并从 `~/.android/avd/` 目录下 AVD 对应的配置文件中读取 AVD 数据目录的地址，然后加载该 AVD 的用户数据镜像及硬件配置等信息。

从代码层面来看，模拟器中查找相关目录路径的代码位于 `emulator_2.3_release/android/android-emu/android/emulation/ConfigDirs.cpp`：
```
// Name of the Android configuration directory under $HOME.
static const char kAndroidSubDir[] = ".android";
// Subdirectory for AVD data files.
static const char kAvdSubDir[] = "avd";

// static
std::string ConfigDirs::getUserDirectory() {
    System* system = System::get();
    std::string home = system->envGet("ANDROID_EMULATOR_HOME");
    if (!home.empty()) {
        return home;
    }

    home = system->envGet("ANDROID_SDK_HOME");
    if (!home.empty()) {
        // In v1.9 emulator was changed to use $ANDROID_SDK_HOME/.android
        // directory, but Android Studio has always been using $ANDROID_SDK_HOME
        // directly. Put a workaround here to make sure it works both ways,
        // preferring the one from AS.
        auto homeOldWay = PathUtils::join(home, kAndroidSubDir);
        return system->pathIsDir(homeOldWay) ? homeOldWay : home;
    }
    home = system->getHomeDirectory();
    if (home.empty()) {
        home = system->getTempDir();
        if (home.empty()) {
            home = "/tmp";
        }
    }
    return PathUtils::join(home, kAndroidSubDir);
}

// static
std::string ConfigDirs::getAvdRootDirectory() {
    System* system = System::get();

    std::string avdRoot = system->envGet("ANDROID_AVD_HOME");
    if ( !avdRoot.empty() && system->pathIsDir(avdRoot) ) {
        return avdRoot;
    }

    // No luck with ANDROID_AVD_HOME, try ANDROID_SDK_HOME
    avdRoot = system->envGet("ANDROID_SDK_HOME");

    if ( !avdRoot.empty() ) {
        // ANDROID_SDK_HOME is defined
        avdRoot = PathUtils::join(avdRoot, kAndroidSubDir);
        if (isValidAvdRoot(avdRoot)) {
            // ANDROID_SDK_HOME is good
            return PathUtils::join(avdRoot, kAvdSubDir);
        }
        // ANDROID_SDK_HOME is defined but bad. In this case,
        // Android Studio tries $USER_HOME and $HOME. We'll
        // do the same.
        avdRoot = system->envGet("USER_HOME");
        if ( !avdRoot.empty() ) {
            avdRoot = PathUtils::join(avdRoot, kAndroidSubDir);
            if (isValidAvdRoot(avdRoot)) {
                return PathUtils::join(avdRoot, kAvdSubDir);
            }
        }
        avdRoot = system->envGet("HOME");
        if ( !avdRoot.empty() ) {
            avdRoot = PathUtils::join(avdRoot, kAndroidSubDir);
            if (isValidAvdRoot(avdRoot)) {
                return PathUtils::join(avdRoot, kAvdSubDir);
            }
        }
    }

    // No luck with ANDROID_SDK_HOME / USER_HOME / HOME
    // Try even more.
    avdRoot = PathUtils::join(getUserDirectory(), kAvdSubDir);
    return avdRoot;
}

// static
std::string ConfigDirs::getSdkRootDirectoryByEnv() {
    auto system = System::get();

    std::string sdkRoot = system->envGet("ANDROID_HOME");
    if ( isValidSdkRoot(sdkRoot) ) {
        return sdkRoot;
    }

    // ANDROID_HOME is not good. Try ANDROID_SDK_ROOT.
    sdkRoot = system->envGet("ANDROID_SDK_ROOT");
    if (sdkRoot.size()) {
        // Unquote a possibly "quoted" path.
        if (sdkRoot[0] == '"') {
            assert(sdkRoot.back() == '"');
            sdkRoot.erase(0, 1);
            sdkRoot.pop_back();
        }
        if (isValidSdkRoot(sdkRoot)) {
            return sdkRoot;
        }
    }
    return std::string();
}

std::string ConfigDirs::getSdkRootDirectoryByPath() {
    auto system = System::get();

    auto parts = PathUtils::decompose(system->getLauncherDirectory());
    parts.push_back("..");
    PathUtils::simplifyComponents(&parts);

    std::string sdkRoot = PathUtils::recompose(parts);
    if ( isValidSdkRoot(sdkRoot) ) {
        return sdkRoot;
    }
    return std::string();
}

// static
std::string ConfigDirs::getSdkRootDirectory() {
    std::string sdkRoot = getSdkRootDirectoryByEnv();
    if (!sdkRoot.empty()) {
        return sdkRoot;
    }

    // Otherwise, infer from the path of the emulator's binary.
    return getSdkRootDirectoryByPath();
}

// static
bool ConfigDirs::isValidSdkRoot(const android::base::StringView& rootPath) {
    if (rootPath.empty()) {
        return false;
    }
    System* system = System::get();
    if ( !system->pathIsDir(rootPath) || !system->pathCanRead(rootPath) ) {
        return false;
    }
    std::string platformsPath = PathUtils::join(rootPath, "platforms");
    if ( !system->pathIsDir(platformsPath) ) {
        return false;
    }
    std::string platformToolsPath = PathUtils::join(rootPath, "platform-tools");
    if ( !system->pathIsDir(platformToolsPath) ) {
        return false;
    }

    return true;
}

// static
bool ConfigDirs::isValidAvdRoot(const android::base::StringView& avdPath) {
    if (avdPath.empty()) {
        return false;
    }
    System* system = System::get();
    if ( !system->pathIsDir(avdPath) || !system->pathCanRead(avdPath) ) {
        return false;
    }
    std::string avdAvdPath = PathUtils::join(avdPath, "avd");
    if ( !system->pathIsDir(avdAvdPath) ) {
        return false;
    }

    return true;
}
```

模拟器需要获得的两个主要目录是 SDK 的根目录路径，和 AVD 根目录路径。模拟器需要通过 SDK 的根目录路径，查找到系统镜像文件；并通过 AVD 根目录路径查找到特定 AVD 的 数据目录路径。

模拟器查找 SDK 的根目录路径的顺序如下：
***$ANDROID_HOME***  ---> 
***$ANDROID_SDK_ROOT*** ===> 
***模拟器的启动目录的上一级目录***

模拟器查找 AVD 根目录路径的顺序如下：
***$ANDROID_AVD_HOME***  ===> 
***$ANDROID_SDK_HOME/.android/avd/*** ---> 
***$USER_HOME/.android/avd/*** ---> 
***$HOME/.android/avd/*** ---> 
***$ANDROID_EMULATOR_HOME/avd/*** ===> 
***$ANDROID_SDK_HOME/.android/avd/*** ---> 
***$ANDROID_SDK_HOME/avd/*** ---> 
***~/.android/avd/*** 

将 AVD 的数据目录移动位置，同时修改 `~/.android/avd` 目录下，该 AVD 的 `*.ini` 配置文件的 `path` 使其指向新的位置，但在启动该 AVD 时，将报出找不到镜像文件的错误。

如将名为 `Nexus_5_API_25_X86` 的 AVD 的数据目录，从 `/home/hanpfei0306/.android/avd/Nexus_5_API_25_X86.avd` 移动到 `/home/hanpfei0306/.android/Nexus_5_API_25_X86.avd`，同时 `/home/hanpfei0306/.android/avd/Nexus_5_API_25_X86.ini` 修改如下：
```
avd.ini.encoding=UTF-8
path=/home/hanpfei0306/.android/Nexus_5_API_25_X86.avd
target=android-25
```

但在启动该 AVD 时，将报出如下错误：
```
$ objs/emulator -avd Nexus_5_API_25_X86
qemu-system-i386: -drive if=none,overlap-check=none,cache=unsafe,index=1,id=cache,file=/home/hanpfei0306/Nexus_5_API_25_X86.avd/cache.img.qcow2,l2-cache-size=1048576: Could not open backing file: Could not open '/home/hanpfei0306/.android/avd/Nexus_5_API_25_X86.avd/cache.img': 没有那个文件或目录
```

即提示找不到 `cache.img` 文件。这是由于 `cache.img.qcow2` 镜像文件中写入了其原始的镜像文件 `cache.img` 的绝对路径。通过十六进制编辑器 `hexedit` 打开 `/home/hanpfei0306/Nexus_5_API_25_X86.avd/cache.img.qcow2`，可以看到如下的内容：
```
00000000   51 46 49 FB  00 00 00 03  00 00 00 00  00 00 01 18  00 00 00 3F  00 00 00 10  QFI................?....
00000018   00 00 00 00  04 20 00 00  00 00 00 00  00 00 00 01  00 00 00 00  00 03 00 00  ..... ..................
00000030   00 00 00 00  00 01 00 00  00 00 00 01  00 00 00 00  00 00 00 00  00 00 00 00  ........................
00000048   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
00000060   00 00 00 04  00 00 00 68  E2 79 2A CA  00 00 00 03  72 61 77 00  00 00 00 00  .......h.y*.....raw.....
00000078   68 03 F8 57  00 00 00 90  00 00 64 69  72 74 79 20  62 69 74 00  00 00 00 00  h..W......dirty bit.....
00000090   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
000000A8   00 00 00 00  00 00 00 00  00 01 63 6F  72 72 75 70  74 20 62 69  74 00 00 00  ..........corrupt bit...
000000C0   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
000000D8   00 00 00 00  00 00 00 00  01 00 6C 61  7A 79 20 72  65 66 63 6F  75 6E 74 73  ..........lazy refcounts
000000F0   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
00000108   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  2F 68 6F 6D  65 2F 68 61  ................/home/ha
00000120   6E 70 66 65  69 30 33 30  36 2F 2E 61  6E 64 72 6F  69 64 2F 61  76 64 2F 4E  npfei0306/.android/avd/N
00000138   65 78 75 73  5F 35 5F 41  50 49 5F 32  35 5F 58 38  36 2E 61 76  64 2F 63 61  exus_5_API_25_X86.avd/ca
00000150   63 68 65 2E  69 6D 67 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  che.img.................
00000168   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
00000180   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
00000198   00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ........................
```

尽管 `avdmanager` 或者说 `sdklib` 提供了 move avd 的功能，但该功能也仅限于将 AVD 的数据目录换一个位置，并修改 `~/.android/avd/` 目录下 AVD 的配置文件指向新的 AVD 数据目录，AVD 数据目录下的 `*.qcow2` 镜像文件中写入的其对应的原始镜像文件的路径并没有改变。

由此可见，AVD 一旦启动过，就不再适合移动位置了，除非删掉之前生成的 `*.qcow2` 镜像文件。

需要预先创建 AVD，并启动它进入 Launcher。在 Android 设备第一次启动时，将为预装的 Android 系统应用生成 odex 等用于优化应用性能的文件。

# AVD 实例管理及用户数据隔离

![image.png](https://www.wolfcstech.com/images/1315506-bf4704dc8fafa1a5.png)

根据不同的场景，创建不同类型的 AVD 实例：可复用的 AVD 实例，即这种 AVD 实例仅提供试玩，每次 AVD 结束之后，生成的用户数据将被清除，下次有新的用户需要试玩时，可以复用这种 AVD 实例；用户 AVD 实例，即专门为特定用户分配的 AVD 实例，每次在 AVD 结束之后，用户数据将保存。

***AVD 第一次启动时，由于需要为系统应用生成 dex 的优化文件等，因而过程极为耗时。***通过如下方式优化 AVD 启动时间：

 * AVD 管理服务检测 AVD 实例的使用情况，并在需要的时候，预先创建 AVD 实例，启动它进入 Laucher 并结束它
 * 请求 AVD 资源时，需要为用户分配预先创建好，并启动过的 AVD。
 * 移除客户 Android 系统中非必须的系统应用：Browser2、Calendar、DeskClock、Email、ExactCalculator、HTMLViewer、LatinIME、Music、OpenWnn、messaging。这一优化同时也将减少单个 AVD 的磁盘消耗。
 * 预装部分应用：搜狗输入法、游戏应用

AVD 管理服务管理的 AVD 池中，将包含三种 AVD 实例：

 * AVD 管理服务预先分配，还没有被使用的 AVD
 * 可复用的 AVD 实例
 * 特定用户专属 AVD 实例

AVD 实例相关镜像的创建：

 * 环境中定义 ***$ANDROID_HOME*** 环境变量，指向定制的系统镜像文件的安装目录。自己编译的系统镜像文件安装到这个目录下。
 * 环境中定义 ***$ANDROID_AVD_HOME*** 环境变量，指向定制的 AVD 根目录。创建的所有 AVD 的根配置文件都将保存在该目录下。
 * 创建一个专门的目录，用于保存各个 AVD 的AVD 数据。

即每个 AVD 的数据将包含三个部分：系统镜像位于系统镜像目录下；AVD 用户数据镜像位于 AVD 数据目录下；AVD 信息文件位于 AVD 主目录下。

参考资料
[创建和管理虚拟设备](https://developer.android.com/studio/run/managing-avds.html)
[Start the Emulator from the Command Line](https://developer.android.com/studio/run/emulator-commandline.html)

### [打赏](https://www.wolfcstech.com/about/donate.html)
