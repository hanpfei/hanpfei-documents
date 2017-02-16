---
title: Chromium GN构建工具的使用
date: 2016-11-16 16:05:49
categories: Android开发
tags:
- Android开发
- chromium
---

Chromium整体的构建过程大体如下：

![Chromium build flow](https://www.wolfcstech.com/images/1315506-eba64e268085674c.png)

这个过程大体为，先由gn工具根据各个模块的.gn配置文件，或gyp工具根据各个模块的.gyp配置文件，产生.ninja文件，再由ninja工具产生最终的目标文件，比如静态库、动态库、exe可执行文件或者是apk文件等等。gyp工具是用Python写的，gn是用C写的，gn增量构建最快。整个Chromium项目，在构建系统方面，也是逐渐在全部转向gn构建。

gn工具不仅仅在我们构建chromium时可以用来产生.ninja文件，它还可以帮助我们描绘chromium的项目地图，比如获取某个模块依赖的所有其它模块，依赖某个模块的所有其它模块、构建参数等信息，可以帮助我们对构建配置的有效性进行检查等。在基于chromium模块进行开发时，gn工具可以为我们提供巨大的帮助。

这里我们来看一下gn工具的用法，逐个来看gn提供的每个命令。

## gn args
这个命令有两个作用，一是生成.ninja构建配置文件，二是查看当前构建环境的配置参数。

### 生成.ninja

生成.ninja是gn工具的基本功能。如在 [Chromium Android编译指南](https://www.wolfcstech.com/2016/10/16/Chromium_Android%E7%BC%96%E8%AF%91%E6%8C%87%E5%8D%97/) 中所示，要构建chromium，或其中的模块，需要先生成.ninja文件对整个构建环境进行配置。为Android版chromium做构建配置时，所执行的命令为：
```
buildtools/linux64/gn args out/Default
```
这个命令的参数是输出目录的路径。这个命令启动系统的默认编辑器，创建`out/Default/args.gn`文件，我们在编辑器中加入我们自己的配置项，如：
```
target_os = "android"
target_cpu = "arm"  # (default)
is_debug = false  # (default)

# Other args you may want to set:
is_component_build = false
is_clang = false
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = false  # Much faster; experimental
android_ndk_root = "~/dev_tools/Android/android-ndk-r10e"
android_sdk_root = "~/dev_tools/Android/sdk"
android_sdk_build_tools_version = "23.0.2"

disable_file_support = true
disable_ftp_support = true
enable_websockets = false
```
gn根据创建的`out/Default/args.gn`文件，及系统中其它的配置，产生ninja文件。

### 查看当前构建环境的配置参数

gn args命令还可以查看当前构建环境的可以配置的参数，参数的默认值及默认值的配置文件等。chromium用到了许多的配置参数。对于这些配置参数中的大多数，即使用户不指定，chromium的构建系统也会提供默认值。当我们在为不知道可以对构建做些什么样的配置，各个配置项的含义，或者配置项默认值的设置位置而抓狂时，这个命令会显得非常有价值。

对于Chromium的Android构建环境，能配置的参数及其相关信息大体如下：
```
$ buildtools/linux64/gn args --list out/Default/
......
android_ndk_root  Default = "//third_party/android_tools/ndk"
    //build/config/android/config.gni:66

android_ndk_version  Default = "r10e"
    //build/config/android/config.gni:67

android_sdk_build_tools_version  Default = "23.0.1"
    //build/config/android/config.gni:71

android_sdk_root  Default = "//third_party/android_tools/sdk"
    //build/config/android/config.gni:69
......
use_platform_icu_alternatives  Default = true
    //url/features.gni:10
    Enables the use of ICU alternatives in lieu of ICU. The flag is used
    for Cronet to reduce the size of the Cronet binary.
......
```
这个功能的gn args命令参数为`--list [output_dir]`。可见gn向我们展示了能为构建定制的每个参数，参数的默认值，及设置参数默认值的文件的位置。
这个工具展示了每一个标记配置项的名称，默认值，创建该标记配置项的配置文件，以及标记配置项作用的说明。

## gn gen
这个命令也是用于产生.ninja文件的，其参数如下：
```
usage:  gn gen [<ide options>] <out_dir>
```
这个命令根据当前的代码树及配置，产生ninja文件，并把它们放在给定的目录下。输出目录可以是源码库的绝对地址，比如`//out/foo`，也可以是相对于当前目录的地址，如：`out/foo`。上面的`gn args out/Default`实际上等价于，启动编辑器编辑参数创建`out/Default/args.gn`文件之后，执行`gn gen <out_dir>`。

实际在构建chromium及其子模块时，这个命令会更加常用。对于特定的平台，通常我们在完成构建环境的配置之后，就不会经常有修改构建配置参数文件的需要了。

## gn clean
这个命令用于对历史构建进行清理。
```
usage:  gn clean <out_dir>
```
它会删除输出目录下除`args.gn`文件外的所有内容，并创建一个可以重新产生构建配置的ninja构建环境。这个命令也比较常用，用来消除历史构建的影响。

## gn desc
这个命令用于显示关于一个给定target或config的信息。
```
usage:  gn desc <out_dir> <label or pattern> [<what to show>] [--blame] [--format=json]
```
构建参数等相关信息，取自给出的`<out_dir>`。`<label or pattern>`可以是一个target标签，一个config标签，或一个标签模式 (参考"gn help label_pattern")。标签模式只匹配targets。

比如我们要查看chromium的net模块相关的信息：
```
$ gn desc out/Default net
Target //net:net
Type: source_set
Toolchain: //build/toolchain/android:arm

visibility
  *

testonly
  false

check_includes
  true

allow_circular_includes_from

sources
......
  //net/spdy/spdy_buffer.cc
  //net/spdy/spdy_buffer.h
  //net/spdy/spdy_buffer_producer.cc
  //net/spdy/spdy_buffer_producer.h
......

public
  [All headers listed in the sources are public.]

configs (in order applying, try also --tree)
......
  //third_party/boringssl:external_config
  //net:net_resources_grit_config
  //base:android_system_libs
  //sdch:sdch_config
  //third_party/zlib:zlib_config
  //net:jni_includes_net_jni_headers

public_configs (in order applying, try also --tree)
  //net:net_config
  //third_party/protobuf:using_proto
  //third_party/protobuf:protobuf_config
  //build/config/compiler:no_size_t_to_int_warning
  //third_party/boringssl:external_config

all_dependent_configs (in order applying, try also --tree)
  //base/allocator:wrap_malloc_symbols

asmflags
  -fno-strict-aliasing
  --param=ssp-buffer-size=4
  -fstack-protector
  -funwind-tables
  -fPIC
......

cflags
  -fno-strict-aliasing
  --param=ssp-buffer-size=4
  -fstack-protector
  -funwind-tables
  -fPIC
......

cflags_cc
  -fno-threadsafe-statics
  -fvisibility-inlines-hidden
  -std=gnu++11
  -Wno-narrowing
  -fno-rtti
  -isystem../../../../../../~/dev_tools/Android/android-ndk-r10e/sources/cxx-stl/llvm-libc++/libcxx/include
  -isystem../../../../../../~/dev_tools/Android/android-ndk-r10e/sources/cxx-stl/llvm-libc++abi/libcxxabi/include
  -isystem../../../../../../~/dev_tools/Android/android-ndk-r10e/sources/android/support/include
  -fno-exceptions

cflags_objcc
  -fno-threadsafe-statics
  -fvisibility-inlines-hidden
  -std=gnu++11
  -fno-rtti
  -fno-exceptions

defines
......
  DISABLE_FTP_SUPPORT=1
  GOOGLE_PROTOBUF_NO_RTTI
  GOOGLE_PROTOBUF_NO_STATIC_INITIALIZER
  HAVE_PTHREAD

include_dirs
  //
  //out/Default/gen/
  /usr/include/kerberosV/
  //third_party/protobuf/src/
  //out/Default/gen/protoc_out/
  //third_party/protobuf/src/
......

ldflags
  -Wl,--fatal-warnings
  -fPIC
......

Direct dependencies (try also "--all", "--tree", or even "--all --tree")
  //base:base
......

libs
  c++_static
  /home/~/dev_tools/Android/android-ndk-r10e/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a
  c
  atomic
  log

lib_dirs
 ~/dev_tools/Android/android-ndk-r10e/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a/
```
可以看到gn desc命令向我们展示了，编译net模块时，包含的所有源文件，该模块发布的头文件，依赖的库，依赖的头文件路径，依赖的库文件的路径，依赖的其它模块，编译参数，链接参数，预定义宏等等等，应有尽有。在我们不喜欢chromium的构建系统及其项目的文件组织结构，而想要将某个模块转换为其它的文件组织结构，比如Eclipse或其它IDE常见的项目文件组织方式时，或者要将编译出来的动态链接库so文件用在其它项目里，被预定义宏的差异搞得焦头烂额时，这个命令就能派上用场了。

借助于这个工具，我们可以很方便的开发自动化的工具，来将chromium的模块单独抽出来用在其它地方，比如我们可以提取net模块发布的头文件，将net模块用在android中。我们之前就有做过这样一个小工具：
```
#!/usr/bin/env python

import os
import shutil
import sys

def print_usage_and_exit():
    print sys.argv[0] + " [chromium_src_root]" + "[out_dir]" + " [target_name]" + " [targetroot]"
    exit(1)

def copy_file(src_file_path, target_file_path):
    if os.path.exists(target_file_path):
        return
    if not os.path.exists(src_file_path):
        return
    target_dir_path = os.path.dirname(target_file_path)
    if not os.path.exists(target_dir_path):
        os.makedirs(target_dir_path)

    shutil.copy(src_file_path, target_dir_path)

def copy_all_files(source_dir, all_files, target_dir):
    for one_file in all_files:
        source_path = source_dir + os.path.sep + one_file
        target_path = target_dir + os.path.sep + one_file
        copy_file(source_path, target_path)

if __name__ == "__main__":
    if len(sys.argv) < 4 or len(sys.argv) > 5:
        print_usage_and_exit()
    chromium_src_root = sys.argv[1]
    out_dir = sys.argv[2]
    target_name = sys.argv[3]
    target_root_path = "."
    if len(sys.argv) == 5:
        target_root_path = sys.argv[4]
    target_root_path = os.path.abspath(target_root_path)

    os.chdir(chromium_src_root)

    cmd = "gn desc " + out_dir + " " + target_name
    outputs = os.popen(cmd).readlines()
    source_start = False
    all_headers = []

    public_start = False
    public_headers = []

    for output_line in outputs:
        output_line = output_line.strip()
        if output_line.startswith("sources"):
            source_start = True
            continue
        elif source_start and len(output_line) == 0:
            source_start = False
            continue
        elif source_start and output_line.endswith(".h"):
            output_line = output_line[1:]
            all_headers.append(output_line)
        elif output_line == "public":
            public_start = True
            continue
        elif public_start and len(output_line) == 0:
            public_start = False
            continue
        elif public_start:
            public_headers.append(output_line)

    if len(public_headers) == 1:
        public_headers = all_headers
    if len(public_headers) > 1:
        copy_all_files(chromium_src_root, public_headers, target_dir=target_root_path)
```

这个命令还可以帮我们dump出模块的整个依赖树，如net模块的依赖树：
```
$ gn desc out/Default //net deps --tree
//base:base
  //base:base_jni_headers
    //base:android_runtime_jni_headers
      //base:android_runtime_jni_headers__jni_Runtime
    //base:base_jni_headers__jni_gen
  //base:base_paths
  //base:base_static
  //base:build_date
......
//url:url_features
```
这个命令可以为我们展示不同类型信息的每一项具体来自于哪个文件。如net模块的C编译标记选项(cflags)的信息如下：
```
$ gn desc out/Default //net cflags --blame
From //build/config/compiler:compiler
     (Added by //build/config/BUILDCONFIG.gn:461)
  -fno-strict-aliasing
  --param=ssp-buffer-size=4
  -fstack-protector
  -funwind-tables
  -fPIC
......
```

## gn ls
这个命令展示给定构建目录下，匹配某个模式的所有targets。
```
usage: gn ls <out_dir> [<label_pattern>] [--all-toolchains] [--as=...] [--type=...] [--testonly=...]
```
默认情况下，除非明确地提供了工具链参数，只有默认工具链中的target会被匹配。如果没有指定标签参数，则显示所有的targets。标签模式不是常规的正则表达式 (可以参考"gn help label_pattern")。如net下的targets：
```
$ gn ls out/Default //net/*
......
//net:http_server
//net:net
......
//net:net_quic_proto
......
//net:quic_client
//net:quic_packet_printer
//net:quic_server
......
```

其它一些关于gn ls命令的例子：
```
  gn ls out/Debug
      这个命令会列出所有的targets。

  gn ls out/Debug "//base/*"
      Lists all targets in the directory base and all subdirectories.

  gn ls out/Debug "//base:*"
      Lists all targets defined in //base/BUILD.gn.

  gn ls out/Debug //base --as=output
      Lists the build output file for //base:base

  gn ls out/Debug --type=executable
      Lists all executables produced by the build.

  gn ls out/Debug "//base/*" --as=output | xargs ninja -C out/Debug
      Builds all targets in //base and all subdirectories.

  gn ls out/Debug //base --all-toolchains
      Lists all variants of the target //base:base (it may be referenced in multiple toolchains).
```
## gn path
这命令查找两个taregets之间的依赖路径。
```
usage: gn path <out_dir> <target_one> <target_two>
```
每个依赖路径输出为一个组，组与组之间用新行分割。参数中两个targets的顺序不限。默认情况下，只打印一个路径。如果某个路径只包含公共依赖，则会输出最短的公共路径。否则，输出最短的公共或私有路径。如果指定了--with-data，则数据依赖也会考虑。如果有多个最短路径，则会从中选出一个。加上--all则会输出两个targets之间的所有依赖路径。

```
$ gn path out/Default //base //net --all
//net:net --[private]-->
//base:base
......
```

## gn refs
这个命令可以用来查找反向的依赖(也就是引用了某些东西的targets)。
```
gn refs <out_dir> (<label_pattern>|<label>|<file>|@<response_file>)* [--all] [--all-toolchains] [--as=...] [--testonly=...] [--type=...]
```
输入可以是如下这些：
 - Target标签
 - Config标签
 - 标签模式
 - 文件名
 - 响应文件

这个命令输出依赖于参数中的输入得targets，比如：
```
$ gn refs out/Default/ net
//:both_gn_and_gyp
//components/cronet/android:cronet
//components/cronet/android:cronet_static
//net:balsa
//net:crypto_message_printer
//net:disk_cache_memory_test
//net:http_server
//net:quic_client
//net:quic_packet_printer
//net:quic_server
//net:simple_quic_tools
//net:stale_while_revalidate_experiment_domains
```
其它一些关于gn refs命令的例子：
```
  gn refs out/Debug //tools/gn:gn
      Find all targets depending on the given exact target name.

  gn refs out/Debug //base:i18n --as=buildfiles | xargs gvim
      Edit all .gn files containing references to //base:i18n

  gn refs out/Debug //base --all
      List all targets depending directly or indirectly on //base:base.

  gn refs out/Debug "//base/*"
      List all targets depending directly on any target in //base or its subdirectories.

  gn refs out/Debug "//base:*"
      List all targets depending directly on any target in //base/BUILD.gn.

  gn refs out/Debug //base --tree
      Print a reverse dependency tree of //base:base

  gn refs out/Debug //base/macros.h
      Print target(s) listing //base/macros.h as a source.

  gn refs out/Debug //base/macros.h --tree
      Display a reverse dependency tree to get to the given file. This will show how dependencies will reference that file.

  gn refs out/Debug //base/macros.h //base/at_exit.h --all
      Display all unique targets with some dependency path to a target containing either of the given files as a source.

  gn refs out/Debug //base/macros.h --testonly=true --type=executable --all --as=output
      Display the executable file names of all test executables potentially affected by a change to the given file.
```
## gn check
这个命令可以用来检查头文件依赖的有效性。其参数为：
```
gn check <out_dir> [<label_pattern>] [--force]
```
如：
```
$ gn check out/Default/ net
Header dependency check OK
```
## gn help
这个命令会向我们展示gn工具的帮助信息，可以用来获取关于上面所有的gn命令，及内置得targets，内建预定义变量的所有相关信息。如：
```
$ gn help
Commands (type "gn help <command>" for more details):
  args: Display or configure arguments declared by the build.
  check: Check header dependencies.
  clean: Cleans the output directory.
  desc: Show lots of insightful information about a target or config.
  format: Format .gn file.
  gen: Generate ninja files.
  help: Does what you think.
  ls: List matching targets.
  path: Find paths between two targets.
  refs: Find stuff referencing a target or file.

Target declarations (type "gn help <function>" for more details):
  action: Declare a target that runs a script a single time.
  action_foreach: Declare a target that runs a script over a set of files.
  bundle_data: [iOS/OS X] Declare a target without output.
  copy: Declare a target that copies files.
  create_bundle: [iOS/OS X] Build an OS X / iOS bundle.
......
```
