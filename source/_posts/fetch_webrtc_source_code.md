---
title: 无需翻墙的 WebRTC 源码下载
date: 2019-07-03 23:05:49
categories: 音视频开发
tags:
- 音视频开发
---

关于 WebRTC 的源码下载和 Demo 的编译运行，WebRTC 的官方文档已经有非常详细的说明。以 Linux 为例，过程大概是这样的：

1. 下载并安装 depot_tools。这是 WebRTC 的代码下载及编译工具集，下载即是把源码 clone 下来，所谓安装只是把  depot_tools 的目录路径放进系统的环境变量 PATH 中即可。
<!--more-->
2. 准备目录
```
$ mkdir webrtc
$ cd webrtc
```

3. 下载代码
```
$ fetch --nohooks webrtc
$ gclient sync
```

4. 安装依赖。

5. 编译运行。

安装依赖和编译运行具体可以参考 WebRTC 的官方文档。

在国内，由于大家都懂的原因，通过 WebRTC 的官方流程下载源码比较困难。笔者过去一直期待有人能够建立一套在国内可以访问的 WebRTC 代码的 mirror 仓库，只是可惜一直未能见到这样的东西。于是笔者就花了点时间看了下，将 WebRTC 的源码及其依赖的一些模块，在 GitHub 和 Bitbucket 上做了一份 fork，做了一点小改动，使得同仁基本可以按照 WebRTC 的官方流程下载编译 WebRTC 的源码。

（ WebRTC 及其依赖的大部分模块源码 fork 在 GitHub 上，但由于对 repo 中文件的最大大小有限制，导致无法把 `tools` 的源码推到 GitHub 上，而 Bitbucket 从开源项目 import 创建 repo 非常方便，因而 `tools` 的源码就被放在了 Bitbucket 上。另外一个模块 `third_party`，它的源码比较多，因而推到 GitHub 上也比较困难，所以也通过 import 创建 repo 在 Bitbucket 上建立 mirror。）

# WebRTC 源码下载过程浅析

在上面 WebRTC 源码的官方下载流程中，首先需要执行 `fetch --nohooks webrtc` 命令。这个命令做的最主要的事情是，下载名为 `.gclient` 的源码下载配置文件：
```
webrtc$ ls -al
总用量 24
drwxr-xr-x 5 hanpfei hanpfei 4096 7月   3 14:13 .
drwxrwxr-x 6 hanpfei hanpfei 4096 7月   3 16:15 ..
-rw-r--r-- 1 hanpfei hanpfei  175 7月   3 14:13 .gclient
```

这个文件的内容如下：
```
solutions = [
  {
    "url": "https://webrtc.googlesource.com/src.git",
    "managed": False,
    "name": "src",
    "deps_file": "DEPS",
    "custom_deps": {},
  },
]
```

执行 `fetch` 之后，通过 `gclient sync` 下载完整的源码库。`gclient` 做的事情主要是两件：

1. 克隆 `.gclient` 文件中 `url` 字段指向的 repo，repo 的放置位置则由 `.gclient` 文件中 `name` 字段描述。如上面的 `.gclient` 文件，`gclient` 工具将会克隆 `https://webrtc.googlesource.com/src.git`，并把repo 放在 `src` 出。

2. 根据第一步中下载的 repo 中依赖文件的内容，下载所有的依赖，依赖文件的路径由 `.gclient` 文件中 `deps_file` 字段描述，如上面的 `DEPS`。

WebRTC 源码库根目录下的 `DEPS` 文件描述了它依赖的所有东西，包括源码形式的模块，也包含二进制形式的模块，这个文件的内容如下面这样：
```
# This file contains dependencies for WebRTC.

vars = {
  'chromium_git': 'https://chromium.googlesource.com',
  # By default, we should check out everything needed to run on the main
  # chromium waterfalls. More info at: crbug.com/570091.
  'checkout_configuration': 'default',
  'checkout_instrumented_libraries': 'checkout_linux and checkout_configuration == "default"',
  'webrtc_git': 'https://webrtc.googlesource.com',
  'chromium_revision': 'ce6d12c81b7d61fbff70d4389c098c6a92a1c57c',
  'boringssl_git': 'https://boringssl.googlesource.com',
  # Three lines of non-changing comments so that
  # the commit queue can handle CLs rolling swarming_client
  # and whatever else without interference from each other.
  'swarming_revision': '96f125709acfd0b48fc1e5dae7d6ea42291726ac',
  . . . . . .
}
deps = {
  # TODO(kjellander): Move this to be Android-only once the libevent dependency
  # in base/third_party/libevent is solved.
  'src/base':
    Var('chromium_git') + '/chromium/src/base' + '@' + 'b8d7dd92d6132c1d168f75130ffeb79fbf94ac11',
  'src/build':
    Var('chromium_git') + '/chromium/src/build' + '@' + 'fe39ce9fd1fd675cc28172aa15f919101deb475a',
  'src/buildtools':
    Var('chromium_git') + '/chromium/src/buildtools' + '@' + '80b545b427d95ac8996a887fa32ba1d64919792d',
  # Gradle 4.3-rc4. Used for testing Android Studio project generation for WebRTC.
  'src/examples/androidtests/third_party/gradle': {
    'url': Var('chromium_git') + '/external/github.com/gradle/gradle.git' + '@' +
      '89af43c4d0506f69980f00dde78c97b2f81437f8',
    'condition': 'checkout_android',
  },
  'src/ios': {
    'url': Var('chromium_git') + '/chromium/src/ios' + '@' + '8c6bafc3be9726bacd212c07f0e4d141584842a8',
    'condition': 'checkout_ios',
  },
  'src/testing':
    Var('chromium_git') + '/chromium/src/testing' + '@' + '5398243e97ec98e520895f0c33fe6af0c375d4f0',
  'src/third_party':
    Var('chromium_git') + '/chromium/src/third_party' + '@' + '68164bde6b551633809786d030d3c5cd48041d46',

  'src/buildtools/linux64': {
    'packages': [
      {
        'package': 'gn/gn/linux-amd64',
        'version': Var('gn_version'),
      }
    ],
    'dep_type': 'cipd',
    'condition': 'checkout_linux',
  },

  . . . . . .
}

hooks = [
  {
    # This clobbers when necessary (based on get_landmines.py). It should be
    # an early hook but it will need to be run after syncing Chromium and
    # setting up the links, so the script actually exists.
    'name': 'landmines',
    'pattern': '.',
    'action': [
        'python',
        'src/build/landmines.py',
        '--landmine-scripts',
        'src/tools_webrtc/get_landmines.py',
        '--src-dir',
        'src',
    ],
  },
  {
    # Ensure that the DEPS'd "depot_tools" has its self-update capability
    # disabled.
    'name': 'disable_depot_tools_selfupdate',
    'pattern': '.',
    'action': [
        'python',
        'src/third_party/depot_tools/update_depot_tools_toggle.py',
        '--disable',
    ],
  },
  {
    'name': 'sysroot_arm',
    'pattern': '.',
    'condition': 'checkout_linux and checkout_arm',
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=arm'],
  },
  {
    'name': 'sysroot_arm64',
    'pattern': '.',
    'condition': 'checkout_linux and checkout_arm64',
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=arm64'],
  },
  {
    'name': 'sysroot_x86',
    'pattern': '.',
    'condition': 'checkout_linux and (checkout_x86 or checkout_x64)',
    # TODO(mbonadei): change to --arch=x86.
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=i386'],
  },
  {
    'name': 'sysroot_mips',
    'pattern': '.',
    'condition': 'checkout_linux and checkout_mips',
    # TODO(mbonadei): change to --arch=mips.
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=mipsel'],
  },
  {
    'name': 'sysroot_x64',
    'pattern': '.',
    'condition': 'checkout_linux and checkout_x64',
    # TODO(mbonadei): change to --arch=x64.
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=amd64'],
  },
  . . . . . .
]

recursedeps = []

# Define rules for which include paths are allowed in our source.
include_rules = [
  # Base is only used to build Android APK tests and may not be referenced by
  # WebRTC production code.
  "-base",
  "-chromium",
  "+external/webrtc/webrtc",  # Android platform build.
  "+libyuv",

  # These should eventually move out of here.
  "+common_types.h",

  "+WebRTC",
  "+api",
  "+modules/include",
  "+rtc_base",
  "+test",
  "+rtc_tools",

  # Abseil whitelist. Keep this in sync with abseil-in-webrtc.md.
  "+absl/algorithm/algorithm.h",
  . . . . . .
]
```

`DEPS` 文件的内容主要分为如下几个部分：

1. 变量定义 `vars`：这个部分定义了将会在后面被引用到的一些变量，如 Git 仓库的根地址，依赖的一些模块的 commit 等。

2. 依赖描述 `deps`：这个部分描述了 WebRTC 依赖的所有模块，既包括 Git repo 形式的依赖，也包括 cipd 模块形式的依赖，既包括无条件的依赖，即适用于所有编译环境平台和目标平台的依赖，也包括有条件的依赖，即仅适用与一定条件的依赖。
无条件的 Git repo 形式依赖如：
```
  'src/base':
    Var('chromium_git') + '/chromium/src/base' + '@' + 'b8d7dd92d6132c1d168f75130ffeb79fbf94ac11',
  'src/build':
    Var('chromium_git') + '/chromium/src/build' + '@' + 'fe39ce9fd1fd675cc28172aa15f919101deb475a',
```
有条件的 Git repo 依赖，如仅在为 iOS 编译时才需要的模块：
```
  'src/ios': {
    'url': Var('chromium_git') + '/chromium/src/ios' + '@' + '8c6bafc3be9726bacd212c07f0e4d141584842a8',
    'condition': 'checkout_ios',
  },
```
Git repo 主要用于存放源码，但也可以用于存放二进制文件，如 yasm 的二进制文件：
```
  'src/third_party/yasm/binaries': {
    'url': Var('chromium_git') + '/chromium/deps/yasm/binaries.git' + '@' + '52f9b3f4b0aa06da24ef8b123058bb61ee468881',
    'condition': 'checkout_win',
  },
```
cipd 模块形式的无条件依赖如：
```
  'src/tools/luci-go': {
      'packages': [
        {
          'package': 'infra/tools/luci/isolate/${{platform}}',
          'version': Var('luci_go'),
        },
        {
          'package': 'infra/tools/luci/isolated/${{platform}}',
          'version': Var('luci_go'),
        },
        {
          'package': 'infra/tools/luci/swarming/${{platform}}',
          'version': Var('luci_go'),
        },
      ],
      'dep_type': 'cipd',
  },
```
cipd 模块形式的有条件依赖，如仅适用于 Linux 的 gn 依赖的描述：
```
  'src/buildtools/linux64': {
    'packages': [
      {
        'package': 'gn/gn/linux-amd64',
        'version': Var('gn_version'),
      }
    ],
    'dep_type': 'cipd',
    'condition': 'checkout_linux',
  },
```
由于 cipd 主要是一套维护二进制文件的系统，因而 cipd 模块形式的依赖都是二进制文件。
每个依赖项的描述主要包含三个字段，`packages` 描述依赖项的详细路径和版本，`dep_type` 描述依赖项的类型，`condition` 描述依赖项启用的条件。
3. `hooks`：由于在某些时间点执行的命令或工具。hook 项的描述主要包含四个字段，`name` 为名称，`pattern` 为模式，`condition` 为应用的条件，`action` 为具体执行的命令工具及其运行参数。如：
```
  {
    'name': 'sysroot_x64',
    'pattern': '.',
    'condition': 'checkout_linux and checkout_x64',
    # TODO(mbonadei): change to --arch=x64.
    'action': ['python', 'src/build/linux/sysroot_scripts/install-sysroot.py',
               '--arch=amd64'],
  },
```
4. `recursedeps`：循环依赖。
5. `include_rules`：定义源码中允许的 include 路径的规则。

不难想到，只要我们可以重新定义 `.gclient` 文件和 `DEPS` 文件的内容，就可以把 WebRTC 的依赖都指向我们在 GitHub 和 Bitbucket 的 Mirror。

# WebRTC 源码下载

1. 克隆 WebRTC 壳工程：
```
$ clone https://github.com/asdfghjjklllllaaa/webrtc.git
$ cd webrtc
```

2. 同步代码
```
$ gclient sync
```

代码量比较大，同步代码比较简单，可能需要多次重试。

下载 cipd 依赖时需要访问主机 `chrome-infra-packages.appspot.com`，这个主机经常无法访问。可以先通过 `http://site.ip138.com/chrome-infra-packages.appspot.com/` 查询这个主机的 IP 地址，然后找一个可以访问的 IP，并把域名/IP信息写入 hosts 来解决。

WebRTC 源码的编译，和 Demo 的运行，可以参考网上的其它文档，如 [webrtc所有平台下载编译步骤详细说明](https://www.jianshu.com/p/f5ff16b3c427) 和 [Windows下 WebRTC Demo运行： PeerConnection](https://blog.csdn.net/chenxiemin/article/details/79039350)

gclient 同步代码结束后，会生成一个名为 `.gclient_entries` 的文件，描述实际同步的模块及其版本号，如：
```
entries = {
  'src': 'https://webrtc.googlesource.com/src.git',
  'src/base': 'https://chromium.googlesource.com/chromium/src/base@e56ffbf17eacb22490aa43cdf74110d951bf367d',
  'src/build': 'https://chromium.googlesource.com/chromium/src/build@33e3a26c2df56dd8bca09cc909f794a1f1b6e3b5',
  'src/buildtools': 'https://chromium.googlesource.com/chromium/src/buildtools@d5c58b84d50d256968271db459cd29b22bff1ba2',
  'src/buildtools/clang_format/script': 'https://chromium.googlesource.com/chromium/llvm-project/cfe/tools/clang-format.git@96636aa0e9f047f17447f2d45a094d0b59ed7917',
  'src/buildtools/third_party/libc++/trunk': 'https://chromium.googlesource.com/chromium/llvm-project/libcxx.git@9b96c3dbd4e89c10d9fd8364da4b65f93c6f4276',
  'src/buildtools/third_party/libc++abi/trunk': 'https://chromium.googlesource.com/chromium/llvm-project/libcxxabi.git@0d529660e32d77d9111912d73f2c74fc5fa2a858',
  'src/buildtools/third_party/libunwind/trunk': 'https://chromium.googlesource.com/external/llvm.org/libunwind.git@69d9b84cca8354117b9fe9705a4430d789ee599b',
  'src/buildtools/win:gn/gn/windows-amd64': 'https://chrome-infra-packages.appspot.com/gn/gn/windows-amd64@git_revision:64b846c96daeb3eaf08e26d8a84d8451c6cb712b',
  'src/testing': 'https://chromium.googlesource.com/chromium/src/testing@80417f33667b5442967941d8b0657453f9976064',
  'src/third_party': 'https://chromium.googlesource.com/chromium/src/third_party@096dc92b019d57f57842e652c8d6d393fd9cf3d5',
  'src/third_party/boringssl/src': 'https://boringssl.googlesource.com/boringssl.git@4a8c05ffe826c61d50fdf13483b35097168faa5c',
  'src/third_party/catapult': 'https://chromium.googlesource.com/catapult.git@acbf095c15e9524a0a1116792c3b6698f8e9b85b',
  'src/third_party/colorama/src': 'https://chromium.googlesource.com/external/colorama.git@799604a1041e9b3bc5d2789ecbd7e8db2e18e6b8',
  'src/third_party/depot_tools': 'https://chromium.googlesource.com/chromium/tools/depot_tools.git@1e2cb1573bd2a2c847dee2844a5b6750e9270c73',
  'src/third_party/ffmpeg': 'https://chromium.googlesource.com/chromium/third_party/ffmpeg.git@e02fc00c5da42ea5cdf2bf5b9bab93c323a1e698',
  'src/third_party/freetype/src': 'https://chromium.googlesource.com/chromium/src/third_party/freetype2.git@31757f969fba60d75404f31e8f1168bef5011770',
  'src/third_party/googletest/src': 'https://chromium.googlesource.com/external/github.com/google/googletest.git@b617b277186e03b1065ac6d43912b1c4147c2982',
  'src/third_party/gtest-parallel': 'https://chromium.googlesource.com/external/github.com/google/gtest-parallel@3ca6798e2c2a06708888611bc5147bd1266f97a0',
  'src/third_party/harfbuzz-ng/src': 'https://chromium.googlesource.com/external/github.com/harfbuzz/harfbuzz.git@f3aca6aa267f7687a0406c7c545aefb5eed300b2',
  'src/third_party/icu': 'https://chromium.googlesource.com/chromium/deps/icu.git@35f7e139f33f1ddbfdb68b65dda29aff430c3f6f',
  'src/third_party/jsoncpp/source': 'https://chromium.googlesource.com/external/github.com/open-source-parsers/jsoncpp.git@f572e8e42e22cfcf5ab0aea26574f408943edfa4',
  'src/third_party/libFuzzer/src': 'https://chromium.googlesource.com/chromium/llvm-project/compiler-rt/lib/fuzzer.git@e847d8a9b47158695593d5693b0f69250472b229',
  'src/third_party/libjpeg_turbo': 'https://chromium.googlesource.com/chromium/deps/libjpeg_turbo.git@61a2bbaa9aec89cb2c882d87ace6aba9aee49bb9',
  'src/third_party/libsrtp': 'https://chromium.googlesource.com/chromium/deps/libsrtp.git@650611720ecc23e0e6b32b0e3100f8b4df91696c',
  'src/third_party/libvpx/source/libvpx': 'https://chromium.googlesource.com/webm/libvpx.git@da5be113f3205544658db52201a66273cf7d6e70',
  'src/third_party/libyuv': 'https://chromium.googlesource.com/libyuv/libyuv.git@b36c86fdfe746d7be904c3a565b047b24d58087e',
  'src/third_party/nasm': 'https://chromium.googlesource.com/chromium/deps/nasm.git@076332ea7c414313ab9d6d5b56396641051df5ea',
  'src/third_party/openh264/src': 'https://chromium.googlesource.com/external/github.com/cisco/openh264@6f26bce0b1c4e8ce0e13332f7c0083788def5fdf',
  'src/third_party/usrsctp/usrsctplib': 'https://chromium.googlesource.com/external/github.com/sctplab/usrsctp@7a8bc9a90ca96634aa56ee712856d97f27d903f8',
  'src/third_party/yasm/binaries': 'https://chromium.googlesource.com/chromium/deps/yasm/binaries.git@52f9b3f4b0aa06da24ef8b123058bb61ee468881',
  'src/third_party/yasm/source/patched-yasm': 'https://chromium.googlesource.com/chromium/deps/yasm/patched-yasm.git@720b70524a4424b15fc57e82263568c8ba0496ad',
  'src/tools': 'https://chromium.googlesource.com/chromium/src/tools@1892917dac44cf1ee586849a2980cd9e9e8b0555',
  'src/tools/luci-go:infra/tools/luci/isolate/${platform}': 'https://chrome-infra-packages.appspot.com/infra/tools/luci/isolate/${platform}@git_revision:25958d48e89e980e2a97daeddc977fb5e2e1fb8c',
  'src/tools/luci-go:infra/tools/luci/isolated/${platform}': 'https://chrome-infra-packages.appspot.com/infra/tools/luci/isolated/${platform}@git_revision:25958d48e89e980e2a97daeddc977fb5e2e1fb8c',
  'src/tools/luci-go:infra/tools/luci/swarming/${platform}': 'https://chrome-infra-packages.appspot.com/infra/tools/luci/swarming/${platform}@git_revision:25958d48e89e980e2a97daeddc977fb5e2e1fb8c',
  'src/tools/swarming_client': 'https://chromium.googlesource.com/infra/luci/client-py.git@aa60736aded9fc32a0e21a81f5fc51f6009d01f3',
}
```

笔者基于 `.gclient_entries` 文件的内容，写了一个简单脚本下载并迁移其中的 Git repo，内容如下：
```

```
这个脚本能工作的基础是，在 GitHub 上已经为它们创建了空 repo。

Done.
