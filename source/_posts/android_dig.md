---
title: android  版 dig
date: 2017-04-19 21:17:49
categories: 网络调试
tags:
- 网络调试
---

最近由于项目需要，移植了一下 PC/Mac 平台非常常用的 dig 工具。Dig 是我们在 PC/Mac 平台常用的 DNS 调试工具，它可以帮我们 dump 出来由域名获得 IP 地址的完整过程。
<!--more-->
整个项目有开源在 GitHub 上，[地址](https://github.com/hanpfei/dig-android)。

这个项目的 build target 是一个 Android ARM 平台上的本地层可执行文件。尽管 Project 的整个目录结构 follow 了常规 Android Studio 工程的目录结构，但构建主要还是通过 NDK 的 ndk-build 工具完成。

# 构建

进入 dig/src/main/jni 目录下，执行如下命令：
```
$ ndk-build 
[armeabi] Compile thumb  : dig <= dig.c
[armeabi] Compile thumb  : dig <= dighost.c
[armeabi] Compile thumb  : dnsutils <= acache.c
[armeabi] Compile thumb  : dnsutils <= acl.c
[armeabi] Compile thumb  : dnsutils <= adb.c
[armeabi] Compile thumb  : dnsutils <= byaddr.c
. . . . . .
```

最终生成的 Android 可执行文件将位于 `dig/src/main/libs`，生成的 `.a` 文件将位于 `dig/src/main/obj/local/`。

可执行文件可 push 进手机的 `/system/bin/` 目录下，并修改可执行文件权限：

```
$ chmod 777 /syste/bin/dig
```

之后就可以像运行普通的应用程序那样执行 dig。

用法与 PC 端 dig 工具是完全一样的。
