---
title: Chromium Android编译指南
---

## 先决条件
需要有一台装有**Linux操作系统环境**的主机来做编译，这个环境的搭建配置方法可以参考[Linux-specific build instructions](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)。目前还不支持在其它（Mac/Windows）平台上来为Android编译Chromium。

<!--more-->

## 获取代码
首先需要下载并安装[depot_tools包](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up)。在一个适当得目录下clone depot_tools包：
```
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```
然后修改.bashrc文件，将depot_tools的路径加进环境变量PATH中：
```
export PATH=$PATH:/path/to/depot_tools
```
如果从来没有下载过Chromium的代码的话，为源码创建一个文件夹并下载源码：
```
mkdir ~/chromium && cd ~/chromium
fetch --nohooks android # This will take 30 minutes on a fast connection
```
如果之前下载过Linux版的Chromium代码，则可以通过给.gclient文件（在上面的源码根目录下，即`~/chromium`目录）附上target_os = ['android']来添加对Android的支持：
```
cat > .gclient <<EOF
 solutions = [ ...existing stuff in here... ]
 target_os = [ 'android' ] # Add this to get Android stuff checked out.
EOF
```
执行gclient sync 来获取Android版的代码：
```
gclient sync
```
## （可选）下载LKGR
如果想编译某个良好状态下的Chromium，则可以同步LKGR(“last known good revision”)的代码。可以在[这里](http://chromium-status.appspot.com/lkgr)找到它，还可以在[这里](http://chromium-status.appspot.com/revisions)找到最近的100个版本。然后运行：
```
gclient sync --nohooks -r <lkgr-sha1>
```
这不是一个典型的开发者工作流所需要的；而只是用于Chromium的一次性编译。
## 为编译做配置
Chromium的整个编译流程为：通过GYP或GN产生最终编译所需的ninja配置文件，然后由ninja根据产生的这些配置文件做编译。为Android编译Chromium可以通过GYP或GN来做配置，但GN增量编译最快，不久之后这也将**成为唯一的编译配置方式**。它们都是给Chromium的Android编译产生ninja文件的元构建系统。在构建瀑布中这两种构建都会被经常测试。
### GYP配置 (已废弃 -- 使用GN来代替)
如果你在使用GYP，则在与.gclient位置相同的文件夹下，创建一个名为‘chromium.gyp_env’的文件，其中具有如下的内容：

```
echo "{ 'GYP_DEFINES': 'OS=android target_arch=arm', }" > chromium.gyp_env
```

注意， “arm”是默认的体系架构，它可以省略。如果是给x86或MIPS编译，则可以将target_arch改成“ia32”或“mipsel”。

注意：如果在使用GYP_DEFINES环境变量，则它将覆盖这个文件中的设置。或者清除它，或者把它设置为上面的值，然后执行`gclient runhooks`。

参考 *build/android/developer_recommended_flags.gypi* 来了解其它建议的GYP设置。创建了chromium.gyp_env之后，你需要运行下面的命令来为gyp文件更新工程。你也需要在添加新的gyp文件、更新gyp文件或同步代码库时，再次执行这个命令。
```
gclient runhooks
```
这将下载更多东西，并提示你接受`Terms of Service for Android SDK packages`。

### GN配置（建议采用）

如果你使用GN，则创建一个编译目录，并通过如下命令设置编译标记：
```
$ gn args out/Default
```
直接执行这个命令会报错，比如：
```
hanpfei0306@hanpfei0306:/media/data/Projects/OpenSource/chromium/src$ gn args out/Default
gn.py: Could not find gn executable at: /media/data/Projects/OpenSource/chromium/src/buildtools/linux64/gn
```
提示找不到`chromium/src/buildtools/linux64/gn`命令，如同GYP配置那样，需要[执行 gclient runhooks ](https://groups.google.com/a/chromium.org/forum/#!topic/chromium-dev/ybtMSTN4yHg)。

可以选择out目录内其它的名字来替代out/Default。但不要使用GYP的out/Debug或out/Release，它们可能与GYP构建冲突。

同时要知道某些脚本（比如tombstones.py，adb_gdb.py）需要设置*CHROMIUM_OUTPUT_DIR=out/Default*。

这个命令(`gn args out/Default`)将用**GN构建参数文件**启动编辑器。在这个文件中添加：
```
target_os = "android"
target_cpu = "arm"  # (default)
is_debug = true  # (default)

# Other args you may want to set:
is_component_build = true
is_clang = true
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = true  # Much faster; experimental
```
也可以指定target_cpu的值为“x86”和“mipsel”。以后可以在那个目录下重新运行gn args来编辑标记。参考[GN构建配置](https://www.chromium.org/developers/gn-build-configuration)来了解你可能想要设置的其它标记。

这个命令启动系统默认的编辑器，创建chromium/src/out/Default/args.gn文件，并根据这个文件的内容来产生ninja文件。也就是在对这个文件做了修改，保存退出之后，gn会根据这个文件的内容来产生ninja文件。

但这个文件本身不一定非要通过系统默认的编辑器来创建，也可以用任意一款自己用起来顺手的编辑器创建。只是在创建完了之后，需要再次运行`gn args out/Default`，然后保存直接退出，以便于让gn工具产生后续编译所需要的ninja文件。

#### 安装依赖
通过如下的命令来更新编译所需的系统包：
```
./build/install-build-deps-android.sh
```

还要确保OpenJDK 1.7被选为了默认得JDK:
```
sudo update-alternatives --config javac
sudo update-alternatives --config java
sudo update-alternatives --config javaws
sudo update-alternatives --config javap
sudo update-alternatives --config jar
sudo update-alternatives --config jarsigner
```
#### 同步子目录
```
gclient sync
```
## 编译
模块编译，如：
```
$ ninja -C out/Default/ net
$ ninja -C out/Default/ url
$ ninja -C out/Default/ zlib
```
这将在`chromium/src/out/Default`下产生这些模块的BUILD.gn文件中定义得targets，比如net和url的共享库，zlib得静态库等。

## 参考文档
[Android Build Instructions](https://chromium.googlesource.com/chromium/src/+/master/docs/android_build_instructions.md)
