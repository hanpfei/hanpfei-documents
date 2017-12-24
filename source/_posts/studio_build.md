---
title: Android Studio 构建
date: 2017-12-24 15:15:49
categories: Android开发
tags:
- Android开发
- 翻译
---

# 获得源码

## 分支

当前我们具有如下老版本 Android Studio 的分支：
<!--more-->

| dev branch    | release branch     | IntelliJ     | Notes     |
|:-------|-------------:|-------------:|-------------:|
| studio-1.0-dev | studio-1.0-release | idea13-dev | 这是 1.0 的分支，***已经关闭*** |
| studio-1.1-dev | studio-1.1-release | idea13-1.1-dev | 这是 1.1 的分支，***已经关闭*** |
| studio-1.2-dev | studio-1.2-release | idea14-1.2-dev | 这是 1.2 的分支，***已经关闭*** |
| studio-1.3-dev | studio-1.3-release | idea14-1.3-dev | 这是 1.3 的分支，***已经关闭*** |
|  |  |  |  |
| **studio-master-dev** | **studio-master-dev** | **studio-master-dev** |  |

`ub-tools-idea133` 和 `ub-tools-master` 分支已经废弃掉了。我们也不使用 `master` 分支。

## 开发分支

像 Android 操作系统一样，Android Studio 也是开源的，且可以自由的控制它。在每个稳定版发布之后，Android 将源码发布到 Android Open Source Project (AOSP)，如 [这里](http://source.android.com/source/code-lines.html) 描述的那样。自 Android Studio 1.4 起，Android Studio 使用了相同的在每个稳定版发布之后发布源码的模式。对于那些为 Android Studio 贡献代码的同学来说，代码提交流程基本上与 Android 平台一样。我们期待继续每隔近 2 - 4 个月发布一个稳定版本的 Android Studio，且每个这样的发布时，源码也将变得可用。请继续为 Android Studio AOSP 分支提交补丁。我们将做 code review 并把修改合并进后续的 Android Studio 版本。我们非常感激所有社区中的你们的合作以及在 Android Studio 上的努力工作。

## 标签

有下列发布标签可用：

 * studio-3.0
 * studio-2.3
 * studio-2.2
 * studio-2.0
 * studio-1.5
 * studio-1.4
 * . . .

gradle 的如下：

 * gradle_3.0.0
 * gradle_2.3.0
 * gradle_2.2.0
 * gradle_2.0.0
 * gradle_1.5.0
 * . . .

## 代码检出

首先，你需要为你的平台安装前提条件。这意味着你需要 git，C 编译器，等等。这里有一些步骤，它们依赖于具体的平台，因此请跳转到官方构建指南页面，其中有详细的指导：[http://source.android.com/source/initializing.html](http://source.android.com/source/initializing.html)。

有些要求是不需要的（如大小写敏感的文件系统），除非你也打算构建平台。如果你在 Mac 上，你将依然需要 XCode 来构建模拟器。

一旦你已经配置了所有东西，则通过如下的指导下载 `repo` 工具：[http://source.android.com/source/downloading.html](http://source.android.com/source/downloading.html)。

然后你可以在 shell 中使用如下命令检出源码：
```
$ mkdir studio-master-dev
$ cd studio-master-dev
$ repo init -u https://android.googlesource.com/platform/manifest -b studio-master-dev
$ repo sync
```

（顶级目录的名字你可以随意确定；我们中那些检出多个分支的同学可以根据分支的名字来命名目录。）

在 `repo init` 期间，它将询问你你的名字和 e-mail 地址；后面如果你决定检入修改集并上传它们以 review，这些信息将被用到。

如果你想检出并构建 2.3 发布版标签，则使用如下的命令：
```
$ repo init -u https://android.googlesource.com/platform/manifest -b studio-3.0
```

后面是 `repo sync`，就像前面看到的那样。

## 执行特定发布版的检出

我们开始给发布版打标签。这意味着你可以使用标签来获得特定版本的源码。当前我们使用如下标签：

| Gradle    | gradle_x.y.z      |
|:-------|-------------:|
| Studio | studio-x.y |

你可以在这里查看所有可用的标签：[https://android.googlesource.com/platform/manifest/+refs](https://android.googlesource.com/platform/manifest/+refs)。

比如，你可以通过如下命令检出 3.0.0 版本的 Gradle 插件：
```
$ repo init -u https://android.googlesource.com/platform/manifest -b gradle_3.0.0
$ repo sync
```

# 构建

通过 `studio-*` 分支构建的 SDK 部分只有 IDE 组件和 SDK Tools。由于构建系统的不同，每个组件通过不同的方式构建。

它们都不使用平台的基于 make 的构建系统。

# 构建 Android Studio

在历史上，构建 Android 工具也需要构建完整的 Android SDK，因为，比如系统镜像所需的模拟器。

然而，我们已经很好地迁移了工具源码为一个更独立的设置，现在你可以构建 Android Studio IDE 而无需一个完整 Android 检出及 C 编译器等等。

## 设置 IntelliJ 以开发 Android Studio

 * 下载最新的 IJ 社区版。
 * 给它添加一个 JDK：Project Structure | SDKs | 添加一个新的 SDK，并命名为 "IDEA jdk"。（**注意这个 SDK 应该是一个标准的 JDK，而不是一个“IntelliJ Platform Plugin SDK”**）

    - 请使用 **JDK 1.6**，因为我们依然支持将 IDE 运行在 Java 6上。你可以使用更新版本的 JDK，但是你可能偶然地访问 1.6 版不可用的 APIs，因此如果你打算上传你的改动的话，请确保你使用的是 JDK 1.6 作为你的 IDEA 的 jdk。

 * 如果你不是在 Mac OSX 上，**请把你的 JDK 中的 tools.jar 也添加** 到你的 IDEA jdk 的 classpath 中。（位于 **<JDK Path>/lib/tools.jar**）

 * （注意：你必须已经启用了 Groovy 和 UI Designer。它们应该是，默认情况下，但是如果你在 .groovy 文件中遇到了编译错误。）

通过上面的步骤检出代码之后，Android 插件的代码位于 `tools/adt/idea`，IntelliJ IDE 的源码位于 `tools/idea/`，及大量的共享库位于 `tools/base/`。

## 编译 IDEA

在 IntelliJ 中，通过选择 Open Project  并选择文件夹 **tools/idea/**，来打开 Android Studio 工程。现在你可以编译、运行及调试工程了。

通过如下命令来编译：
```
$ cd tools/idea
$ ./build_studio.sh
```

（如果是在 Windows 上，且无法运行 .sh 脚本，则运行 "ant" 来替代；脚本将首先设置一些环境变量。）

在 **out/artifacts** 中查看编译结果。

# 构建插件

检出代码之后，Gradle Plugin 的代码位于 **tools/base**。

所有的工程在一个多模块 Gradle 工程中一起构建。那个工程的根目录是 **tools/**。

当前的 Gradle Plugin 以 Gradle 4.0 构建。为了确认你正在使用正确的版本，请在工程的根目录中构建时，使用 gradle 包装脚本（**gradlew**）。

你可以通过如下命令构建 Gradle 插件（及相关的库）：
```
$ ./gradlew assemble
```

如果第一次 **assemble** 执行失败，则试一下如下命令：
```
$ ./gradlew clean assemble
```

要测试插件，你需要运行如下的命令：
```
$ ./gradlew check
```

此外，你应该把一个设备连接到你的工作站并运行：
```
$ ./gradlew connectedIntegrationTest
```

为了运行特定的 connectedIntegrationTest，则运行：
```
$ ./gradlew connectedIntegrationTest -D:base:integration-test:connectedIntegrationTest.single=BasicTest
```

原文：

[Build Overview](http://tools.android.com/build)
[Building Android Studio](http://tools.android.com/build/studio)
[Building the Android Gradle Plugin](http://tools.android.com/build/gradleplugin)

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
