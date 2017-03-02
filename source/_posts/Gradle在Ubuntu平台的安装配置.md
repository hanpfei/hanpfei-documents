---
title: Gradle在Ubuntu平台的安装配置
date: 2017-03-01 17:05:49
categories: Java开发
tags:
- 后台开发
- Java开发
---

Gradle 是以 Groovy 语言为基础，面向 [Java](http://lib.csdn.net/base/javase) 应用为主。基于DSL（领域特定语言）语法的自动化构建工具。现在 Google 官方提供的IDE Android Studio 用它来编译APK程序。然而，Gradle 不仅可以用于构建Android 项目，在 JVM 后端开发它的应用同样非常广泛。

 <!--more-->
# Gradle Wrapper安装

在我们用 Android Studio 中的 Gradle 构建 Android 应用时，除了偶尔需要在 `gradle/wrapper/gradle-wrapper.properties` 修改一下用到的 Gradle 版本外，几乎感知不到 Gradle 工具的安装配置过程的存在。其实，在我们修改了依赖的 Gradle 版本之后，IDE 会自动帮我们下载 Gradle 的整个构建系统并完成配置，下载的这些文件存放于 `~/.gradle/wrapper/dists/` 下，以 2.14.1 版的 Gradle 为例，这个构建系统的目录结构如下：
```
~/.gradle/wrapper/dists/gradle-2.14.1-all/8bnwg5hd3w55iofp58khbp6yv/gradle-2.14.1$ ls
bin  changelog.txt  docs  getting-started.html  init.d  lib  LICENSE  media  NOTICE  samples  src
```

Android 中用到的这种 Gradle 安装配置方式称为 `Gradle Wrapper`，我们无需手动执行 Gradle 构建系统的安装。这种安装方式所需要的东西如下：
```
gradlesystem$ find .
.
./gradle.properties
./gradle
./gradle/wrapper
./gradle/wrapper/gradle-wrapper.properties
./gradle/wrapper/gradle-wrapper.jar
./gradlew
./gradlew.bat
```
`gradle/wrapper/gradle-wrapper.properties` 文件用于配置要安装的 Gradle 的下载地址，下载之后存放的位置等等。`gradle/wrapper/gradle-wrapper.jar` 是一个Java可执行程序，主要依靠它来处理整个下载配置过程。`gradlew` 和 `gradlew.bat` 是两个可执行脚本，它们是 Gradle 构建工具的包装器，我们可以用它们执行 task，或安装构建系统（类Unix系统用 `gradlew`，Windows 系统用 `gradlew.bat` ）。用它们执行 task 时，如果已经安装了 Gradle，这些包装器脚本将定位并使用正确的 Gradle 版本，否则它将下载并安装正确的 Gradle 版本。

看一个 `gradle/wrapper/gradle-wrapper.properties` 的例子：
```
#Fri Nov 25 20:14:20 CST 2016
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-3.4-all.zip
```

在命令行中输入 `gradlew`，Gradle 构建系统将被自动地安装：
```
~/data/MyProjects/gradlesystem$ ./gradlew 
Downloading https://services.gradle.org/distributions/gradle-3.4-all.zip
................................................................................................................................................................................................................................................................................................................................................................................................................................................................
Unzipping ~/.gradle/wrapper/dists/gradle-3.4-all/4bi7dnjj1pmknw0wphqavp2sz/gradle-3.4-all.zip to ~/.gradle/wrapper/dists/gradle-3.4-all/4bi7dnjj1pmknw0wphqavp2sz
Set executable permissions for: ~/.gradle/wrapper/dists/gradle-3.4-all/4bi7dnjj1pmknw0wphqavp2sz/gradle-3.4/bin/gradle
Starting a Gradle Daemon (subsequent builds will be faster)
:help

Welcome to Gradle 3.4.

To run a build, run gradlew <task> ...

To see a list of available tasks, run gradlew tasks

To see a list of command-line options, run gradlew --help

To see more detail about a task, run gradlew help --task <task>

BUILD SUCCESSFUL

Total time: 3 mins 26.676 secs
```

# 手动安装

然而，Gradle 不仅可以用于构建 Android 应用，它也常见于 Java 后端的项目中，比如 Kafka 就是用的 Gradle。我们也可以手动安装 Gradle。步骤如下：

### 步骤一：[下载](https://gradle.org/releases)最新的 Gradle 发行版
当前的 Gradle 发行版是 3.4，在 2017.2.20 发布。分发zip文件有两种类型：
* [仅包含二进制文件](https://services.gradle.org/distributions/gradle-3.4-bin.zip)
* [完整包](https://services.gradle.org/distributions/gradle-3.4-all.zip) (包含文档和源码)

想要用其它发现版的，可以参考 [Releases page](https://gradle.org/releases)。

### 步骤二，解压缩分发包
将分发包解压到选定的目录下，比如：
```
$ mkdir /opt/gradle
$ unzip -d /opt/gradle gradle-3.4-bin.zip
$ ls /opt/gradle/gradle-3.4
LICENSE NOTICE bin getting-started.html init.d lib media
```

### 步骤三，配置系统环境
配置 `PATH` 环境变量包含解压的分发包的 `bin` 目录，比如修改 `~/.bashrc` 文件，添加如下的两行：
```
export PATH=$PATH:/opt/gradle/gradle-3.4/bin
export GRADLE_HOME=/opt/gradle/gradle-3.4
```

### 步骤四，验证安装
打开一个新的终端，或者执行如下命令：
```
source ~/.bashrc
```
来更新当前的环境变量。

然后，运行 `gradle -v` 来执行 `gradle` 并显示出版本，比如：
```
/media/data/dev_tools/gradle-3.4/bin$ gradle -v

------------------------------------------------------------
Gradle 3.4
------------------------------------------------------------

Build time:   2017-02-20 14:49:26 UTC
Revision:     73f32d68824582945f5ac1810600e8d87794c3d4

Groovy:       2.4.7
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_121 (Oracle Corporation 25.121-b13)
OS:           Linux 4.4.0-62-generic amd64
```

### [打赏](https://www.wolfcstech.com/about/donate.html)

# 参考文档
[Ubuntu之安装Gradle](http://blog.csdn.net/stwstw0123/article/details/47809189)

Done。