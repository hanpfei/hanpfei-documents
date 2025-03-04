---
title: 深入 PipeWire
date: 2025-03-08 21:05:49
categories: 音视频开发
tags:
- 音视频开发
---

## 简介

随着它的成熟，PipeWire 项目正在慢慢地变得流行。它的[文档](https://docs.pipewire.org/index.html)依然相对稀少，但正在逐渐增长。然而，让项目外部的人尝试用他们自己的语言来理解和解释它总是一个好主意，重申想法，从他们自己的角度来看待它。

在之前的一篇[文章](https://venam.net/blog/unix/2021/02/07/audio-stack.html)中，我讨论了 Unix 上的通用音频栈，并有一节提到了 PipeWire。不幸的是，因为当时我没有找到足够的文档，并且无法理解一些概念，我认为我没有对项目进行公正的处理，甚至可能混淆了某些部分。

在这篇文章中，我将尝试用可能的最简单的方式来解释 PipeWire，让那些想要追随这个很酷的新项目，但又不知道从哪里开始的人能够接触到它。尤其重要的是，这样做可以为更多的人打开一扇门，可以让他们加入并关注当前的发展，这正在快速进行。

*免责声明*：首先我想说的是，我并不是参与这个项目的开发者，只是一个感兴趣的互联网旅行者。我可能仍然犯了错误，没有涵盖所有内容，所以一定要留下评论或电子邮件，以便我可以纠正或添加信息。

*PS*：如果你喜欢深入研究和类似的讨论，那就去 [nixers](https://nixers.net/) 社区看看，那里有很多喜欢这样做的人。

*PPS*：我在这篇文章中使用了一个与令人难以置信的 [PulseAudio Under the Hood](https://gavv.github.io/articles/pulseaudio-under-the-hood/) 一文类似的名字，但我不认为我能像 PulseAudio 的作者所做的那样详细介绍 PulseAudio。

## 什么是 PipeWire —— 快速试运行

PipeWire 是一个媒体处理图，这可能不太清晰，所以让我重新表述一下。PipeWire 是一个守护进程，它提供了类似于 shell 管道的东西，但是用于媒体：音频和视频。

在这个图中存放的是可以代表多种事物的节点，从耳机或网络摄像头等真实设备到音频过滤器等虚拟事物。这些节点具有端口，这些端口可以链接在一起，媒体数据从第一个节点的 **source** 流向下一个节点的 **sink**。每个节点中发生的事情取决于它们以及它们提供的接口或功能。

实际上，节点、链接、端口和其它都是扩展基本类型的不同对象，并且存在于此图中。这些对象也不一定是与媒体相关的，它们可以做很多其它的事情。我们将看到它们使用特殊的类型系统、插件系统和序列化格式/存储。

总而言之，PipeWire 是一个图，“它是机制而不是策略”。这意味着要创建图并与之交互，我们需要另一个软件。标准的方法是依赖 PipeWire 称为**会话管理器**或策略管理器的东西。这种软件的角色是依赖于环境创建并管理图中的实体，比如当设备插入时，或者设置了恢复音量策略，或者在允许一个客户端访问设备之前需要检查权限。

目前，有两个这样的会话管理器实现：默认的称为 **pipewire-media-session**，还有一个还在开发中，但非常有前途和有趣的，称为**WirePlumber**。而且，你可以选择依赖外部工具来管理 PipeWire 图中的内容来构建自己的会话管理工作流。

让我们到此为止，并看看如何运行 PipeWire。从你的包管理器获取它，并在终端中执行如下命令。
```
$ pipewire
```

在另一个终端中执行如下命令：
```
$ pipewire-media-session
```

在某些情况下，正如我们将在 pipewire 的配置中看到的，你可能不需要执行第二个命令，pipewire 可以被设置为自动启动 `pipewire-media-session`。事先试一试 `ps` 以防万一。

如果没有应用程序可以与 PipeWire 对话，那么上述操作也不会让你有所收获 —— 目前只有少数应用程序可以。此外，PipeWire GUI 和工具仍然缺乏，我们将看到。这也就是为什么 PipeWire 要提供三个兼容层：通过一个 PCM 设备与 **ALSA** 兼容，通过 pipewire-pulse 服务器与 **PulseAudio** 兼容，通过 `pw-jack` 命令与 **JACK** 兼容。

要确保你获得了 ALSA 的兼容性，请检查 */etc/alsa/conf.d/50-pipewire.conf* 文件，它应该为 pipewire 定义了一个 pcm，它通常在安装 **pipewire-alsa** 包时创建。然后通过 `alsactl dump-cfg` 或 [aconfdump](https://github.com/venam/aconfdump) 查看 ALSA 配置，来检查它是否已经变为了默认 pcm，并验证 `pcm.default` 项（通常通过 *99-pipewire-default.conf* 这样的东西）。然而，如果你仍然想要依赖 PulseAudio，它就不需要了，在这种情况下，它将是 `type pulse`。

对于 PulseAudio 层，它允许使用 PulseAudio 工具来与 PipeWire 通信，安装 `pipewire-pulse` 并启动它。你的发行版甚至可能通过 init/service 管理器与会话管理器一起自动启动它。

最后，对于 JACK，安装 `pipewire-jack` 包并在任何 JACK 相关的命令之前发出 `pw-jack`，则它们将自动地使用 PipeWire。比如：
```
$ pw-jack qjackctl
```

此外，对于视频功能，你应该安装桌面 portal，它是一个实现了 [xdg-desktop-portal](https://flatpak.github.io/xdg-desktop-portal/portal-docs.html) 规范的 D-Bus 服务，它的工作是检查是否允许客户端请求访问视频。有许多这样的软件：

 * xdg-desktop-portal-gtk
 * xdg-desktop-portal-kde
 * xdg-desktop-portal-wlr

如果你正在运行 Wayland 合成器，你应该熟悉这些，因为这是在 Wayland 上访问视频 (网络摄像头，屏幕共享，截屏) 的唯一方法。

自此，PipeWire 运行起来了！然而，还有更多东西要看，比如如何配置 PipeWire 和会话管理器，如何编写依赖 PipeWire 的软件，其中蕴含的思想，如何管理图和依赖工具，以及大量其它示例。

## 与 GStreamer 的关系

## 构建块：POD/SPA

### POD —— Plain Old Data

### SPA —— Simple Plugin API

## PipeWire Lib，来自 Wayland 的灵感

## 配置

### PipeWire 服务器配置

### PipeWire 客户端配置

### PipeWire 会话管理器配置

### PipeWire 媒体会话配置

### WirePlumber 配置

## 工具 & 调试

### 本地 PipeWire 工具

PipeWire 带有一系列工具，可以用来执行常见的任务，如与服务器交互，调试或性能调优。如果你熟悉 PulseAudio 带的工具集，你将发现它们很相似。

下面是其中一些有趣的工具：

 * `pw-cat`：用于播放音频
 * `pw-play`，`pw-record`，`pw-midirecord`，`pw-midiplay`：指向 `pw-cat` 的符号链接
 * `pw-record`：用于录制音频
 * `pw-loopback`：创建虚拟环回节点
 * `pw-link`：端口和链接管理器，用于列出端口，监视它们，并创建链接
 * `pw-dump`：用于转储图中的节点或整个图
 * `pw-dot`：与 `pw-dump` 类似，但以 graphviz 格式转储
 * `pw-top`，`pw-profiler`：用于监视图中对象之间的流量效率
 * `pw-mon`：用于监视图中发生的任何事件
 * `pw-metadata`：用于修改图中的元数据，它当前被用于存储默认的节点信息
 * `pw-cli`：与 PipeWire 守护进程交互的通用命令行工具，允许用来转储，加载和卸载模块，列出对象，创建链接和节点，设置参数，等等。

这些工具，特别是 `pw-cli`，可以被用来创建对象并管理运行中的图，或调查图中发生的事。

这里有一个在两个节点的两个左前端口间创建链接的例子：
```
$ pw-cli create-link "TestSink" 'monitor_FL' \
"alsa_output.usb-C-Media_Electronics_Inc.device.iec958-stereo" \
'playback_FL'
```

或者转储所有可用的工厂：
```
$ pw-cli dump Factory
```

或者创建一个虚拟节点：
```
pw-cli create-node adapter { \
factory.name=support.null-audio-sink node.name=my-mic \
media.class=Audio/Duplex object.linger=1 \
audio.position=FL,FR }
```

除了命令行工具之外，目前还存在一个名为 [helvum](https://gitlab.freedesktop.org/pipewire/helvum) 的基于 Rust-Gtk 的本地 GUI。它仍然是一个 WIP 项目，提供通过点击两个边的端口连接节点的基本功能。除此之外，PipeWire 还缺乏前端，特别是那些允许适当地操纵/表示任何类型媒体（包括音频和视频）的图的前端。

helvum 看起来是这样的：

![](./images/)

### 依赖于 PulseAudio 的工具

### 依赖于 Jack 的工具



### 依赖于 GStreamer 的工具

最后，正如我们前面提到的，我们可以依赖使用 gstreamer 的任何程序和 gstreamer 命令行工具来处理媒体。再次，通过如下插件：

 * `pipewiresrc`：PipeWire source
 * `pipewiresink`：PipeWire sink
 * `pipewiredeviceprovider`：PipeWire Device Provider

我们可以创建一个 PipeWire sink，它将提供来自网络的音频：
```
gst-launch-1.0 \
uridecodebin 'uri=http://podcast.nixers.net/feed/download.php?filename=nixers-podcast-2020-07-221.mp3' ! \
pipewiresink mode=provide \
stream-properties="props,media.class=Audio/Source,node.description=podcast"
```

然后将端口链接到一个音频设备，通过命令行工具或 helvum 这样的 GUI。
```
pw-link gst-launch-1.0:capture_1 \
alsa_output.usb-C-Media_Electronics_Inc._Microsoft_LifeChat_LX-3000-00.iec958-stereo:playback_FL
pw-link gst-launch-1.0:capture_2 \
alsa_output.usb-C-Media_Electronics_Inc._Microsoft_LifeChat_LX-3000-00.iec958-stereo:playback_FR
```

理论上，这也应该允许我们管理视频，然而，正如我上面提到的，我遇到了一些问题。我已经测试了如下命令，并尝试通过 `cheese` 网络摄像头应用打开它，但视频只展示了 2s 就停止了：
```
gst-launch-1.0 \
uridecodebin uri=https://venam.net/workflow-compil-venam-2020.webm ! \
pipewiresink mode=provide \
stream-properties="props,media.class=Video/Source,node.description=compilation"
```

尽管如此，这种可能性还是很吸引人的：在运行中轻松地对视频（包括相机）进行修改和添加滤镜。

## 结论







[原文](https://venam.net/blog/unix/2021/06/23/pipewire-under-the-hood.html)。

Done.
