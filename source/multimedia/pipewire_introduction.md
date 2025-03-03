---
title: PipeWire 简介
date: 2025-03-01 21:05:49
categories: 音视频开发
tags:
- 音视频开发
---

[PipeWire](https://pipewire.org/) 是一个底层的多媒体框架，旨在替代 PulseAudio 和 JACK 这些 Linux 平台更为传统的音频服务器，它聚焦于处理多媒体数据 (主要是音频、视频和 MIDI)，提供更灵活和高效的音频、视频处理能力。

PipeWire 项目的创始人及最核心贡献者是 **Wim Taymans**，他是一位著名的软件工程师和开源开发者，特别是在多媒体和音视频领域。除了 PipeWire 项目，**Wim Taymans** 还是 GStreamer 多媒体框架的核心贡献者之一，同时对于 PulseAudio、Linux 内核中与音频和多媒体相关的部分，以及 WebRTC 的实现中音频处理相关的代码，也多有贡献。**Wim Taymans** 曾在多家知名科技公司工作，包括 **Red Hat** 和 **Collabora**。在 Red Hat 工作期间，他领导了 PipeWire 项目的开发，并推动了其在 Linux 生态系统中的采用。在 Collabora 工作期间，作为 Collabora 的工程师，他参与了 GStreamer 和其它多媒体项目的开发。**Wim Taymans** 的 GitHub 个人主页为 [wtay](https://github.com/wtay)。

作为 PipeWire 要替代的主要对象的 PulseAudio，它的项目创建者和核心开发者是 **Lennart Poettering**。**Lennart Poettering** 在 2004 年创建了 PulseAudio，旨在为 Linux 提供一个现代化的音频服务器。尽管 PulseAudio 为 Linux 系统的音频部分做出了巨大贡献，但如今看来，它已经有些过时了。**Lennart Poettering** 也是 Linux 系统软件领域的一位著名的软件工程师和开源开发者，除了 PulseAudio，他还创建了 systemd 和 Avahi 等项目，参与了 D-Bus 消息总线系统的开发，对 Linux 生态系统产生了深远的影响。Lennart Poettering 的职业背景包括 **Red Hat** 和 **Microsoft**。他在 Red Hat 工作期间，创建了 PulseAudio 和 systemd，并推动了它们在 Linux 生态系统中的采用。在 2022 年，Lennart Poettering 加入 Microsoft，继续从事系统软件和开源项目的工作。**Lennart Poettering** 的 GitHub 个人主页为 [poettering](https://github.com/poettering)。

再来看一下更多常见的多媒体项目的创建时间：

| 项目 | 创建时间 | 创建者和核心贡献者 | 说明 |
|--|--|:--|:--|
| VLC | 1996 年 | 最初由法国巴黎中央理工学院的学生在 1996 年开发，作为学术项目。 <br>**Jean-Baptiste Kempf**：VLC 的核心开发者之一，现任 VideoLAN 协会主席。<br>**Rafaël Carré**：VLC 的核心开发者，贡献了大量底层代码。|  |
| GStreamer | 1999 年 | **Erik Walthinsen** | 旨在为 Linux 提供一个模块化的多媒体框架。 |
| FFmpeg | 2000 年 | **Fabrice Bellard**：FFmpeg 的创始人，也是 QEMU 和 Tiny C Compiler 的创建者。<br>**Michael Niedermayer**：FFmpeg 的核心开发者之一，贡献了大量编解码器和优化代码。 |  |
| CoreAudio | 2001 年 | **苹果公司** | macOS 和 iOS 操作系统中的音频框架 |
| JACK | 2002 年 | **Paul Davis**：JACK 的创始人和核心开发者，也是 Ardour 数字音频工作站的创建者。 |  |
| PulseAudio | 2004 年 | **Lennart Poettering** |  |
| OpenMAX | 2006 年 | **Khronos Group** | Khronos Group 是一个由多家科技公司组成的联盟，成员包括 NVIDIA、Intel、ARM、Qualcomm 等公司。 |
| AudioFlinger | 2008 年 | **谷歌（Google）公司** | 安卓（Android）操作系统的一部分。 |
| WebRTC | 2011 年 | **谷歌（Google）公司** | 基于 Google 收购的 GIPS（Global IP Solutions）技术 |
| PipeWire | 2018 年 | **Wim Taymans** |  |

**Wim Taymans** 在 GStreamer、WebRTC 和 PulseAudio 等项目中的经历无可避免地会影响到他在 PipeWire 项目中的设计决策，同时对 CoreAudio、AudioFlinger 等多媒体框架的设计思想会有所借鉴。遵循之前十年中创建的多个有影响力的 Linux 系统服务的倾向，PipeWire 的设计也选择更加充分地利用 Linux 内核独有的 API，跨操作系统平台不再成为重要的设计目标。**Wim Taymans** 和 **Lennart Poettering** 都服务于 Linux 系统生态中的巨头 **Red Hat** 公司，对于 PipeWire 在 Linux 生态系统中的采用，特别是率先用于 Fedora。更多关于 PipeWire 发展历程的内容可以参考 [PipeWire 和 AGL [PDF]](https://wiki.automotivelinux.org/_media/pipewire_agl_20181206.pdf) 和 [PipeWire：Linux 音频/视频总线 (LWN)]()。

PipeWire 提供如下特性：

 * 基于图的处理。
 * 支持进程外处理图，且开销极小。
 * 灵活且可扩展的媒体格式协商和缓冲区分配。
 * 硬实时功能插件。
 * 音频和视频处理非常低的延迟。
### 概念
#### PipeWire 服务器
PipeWire 是一个基于图的处理框架，它聚焦于处理多媒体数据 (主要是音频、视频和 MIDI)。

PipeWire 的图由节点 (node) 组成。每个节点接受任意个数称为端口 (port) 的输入，对这些多媒体数据执行一些处理，然后从其输出端口将数据发送出去。图中的边在这里称为链接 (link)。它们能够将一个输出端口和一个输入端口连接起来。

节点可以有任意数量的端口。只有一个输出端口的节点通常称为 source，sink 是只处理输入端口的节点。

PipeWire 服务器本身提供了其中一些节点的实现。最重要的是，它像任何其它 ALSA 客户端一样使用 alsa-lib 将静态配置的 ALSA 设备公开为节点。比如：

 * 一个立体声 ALSA PCM 播放设备可以显示为具有 2 个输入端口的 sink：左前和右前，或
 * 一个虚拟 ALSA 设备，尝试使用 ALSA 的客户端可以直接连接到它，可以显示为具有 2 个输出端口的 source：左前和右前。

类似的机制出现在与使用 JACK 或 Pulseaudio 的应用程序交互或通信的地方。

注意：`pw-jack` 修改 `LD_LIBRARY_PATH` 环境变量，因而该应用程序将加载 JACK 客户端库的 PipeWire 实现，而不是 JACK 自己的库。这使得 JACK 客户端被重定向到 PipeWire。

其它的节点由 PipeWire 客户端实现。
#### PipeWire 客户端
PipeWire 客户端可以是任何进程。它们可以使用 PipeWire 本地协议通过 UNIX 域 socket 与 PipeWire 服务器对话。除了实现节点外，它们也可以控制图。
##### *图控制*
PipeWire 服务器本身不执行任何图管理；上下文相关的行为，如监视新的 ALSA 设备，配置它们以使它们显示为节点，或链接节点不会自动地完成。相反，它提供了一个允许生成、链接和控制这些节点的 API。客户端依赖这个 API 来控制图的结构，而无需担心图的执行过程。

经常使用的推荐模式是将单个客户端作为处理会话和策略管理的守护进程。目前已知有两种实现：

 * pipewire-media-session，它是会话管理器的第一个实现。今天，它主要用于调试场景。
 * WirePlumber，它采用模块化方法：相对于 PipeWire 的 API，它提供了另一个更高层的 API，并运行使用所谓的 API 来实现管理逻辑的 Lua 脚本。它附带了默认的脚本和配置，用于处理链接策略，以及监控和自动生成 ALSA、bluez、libcamera 和 v4l2 设备。这个 API 任何进程都可以用，而不仅仅是 WirePlumber 的 Lua 脚本。
##### *节点实现*
通过它们实现的节点，客户端可以将多媒体数据发送到图中，也可以从图中获取多媒体数据。一个客户端可以创建多个 PipeWire 节点。这允许一个客户端创建更复杂的应用程序。比如以浏览器为例，它可以为请求音频播放能力的每个标签页创建一个节点，让会话管理器处理路由：这允许用户将不同的标签页 source 路由到不同的 sink。另一个例子是需要许多输入的应用程序。
#### API 语义
PipeWire 服务器及其能力的当前状态，和 PipeWire 图向客户端公开 - 包括 `pw-dump` 这样的调查工具 - 为对象集合，其中每个具有一个特定的类型。这些对象具有关联的参数、属性、方法、事件和权限。

对象的参数是具有特定的定义良好的含义的数据，它可以通过 PipeWire API 以受控的方式修改和读取。它们用于在运行时配置对象。参数是允许 WirePlumber 通过提供下列信息与节点协商数据格式和端口配置的关键：

 * 多个，支持的采样率
 * 通道数
 * 位置采样格式
 * 可用的监视端口

对象的属性是附加在模块上的额外数据，并且 PipeWire 本身无法理解这些数据。按照约定，特定对象类型需要某些属性。

每个对象类型具有一个它需要实现的方法列表。

会话管理器负责定义每个客户端拥有的权限列表。每个权限项是一个对象 ID 和四个标志。四个标志是：

 * 读取 (Read)：对象可以被看到，事件可以被接收；
 * 写入 (Write)：对象可以被修改，常常通过方法 (这需要执行标志)；
 * 执行 (eXecute)：方法可以被调用；
 * 元数据 (Metadata)：可以为对象设置元数据；
 * 链接 (Link)：任何链接都可以建立，甚至可以连接到端口所有者不可见的端口。
##### *对象类型*
以下是已知的类型及其最重要的专用参数和方法：
###### *Core*
core 是 PipeWire 服务器的核心。每个服务器只能有一个 core，它的标识符为 0。它表示服务器的全局属性。PulseAudio 的守护进程中也有一个角色类似的 core。
###### *客户端*
客户端对象表示客户端进程与服务器之间打开的连接。
###### *模块*
模块是运行时在客户端和服务器中加载的动态库，并执行任意操作，例如创建设备或提供创建链接、节点等的方法。

PipeWire 中的模块只能在它们自己的进程中加载。客户端，比如，不能在服务器中加载模块。这种行为与 PulseAudio 不同。
###### *节点*
节点是 PipeWire 中的核心数据处理实体。它们可以生产数据 (采集设备，信号发生器，等等等)，消费数据 (播放设备，网络端点，等等等)，或两者都是 (过滤器，filters)。节点有一个方法 `process`，它从输入端点吃进数据，并给每个输出端口提供数据。
###### *端口*
端口是节点的数据入口点和出口点。端口可以用于输入或输出 (但不能同时是两者)。用于音频数据处理的节点，配置的一种类型是它们是否具有 `dsp` 端口或 `passthrough` 端口。在 `dsp` 模式中，多通道音频的每个通道都有一个端口 (比如，立体声音频有 2 个端口)，数据总是 32 位浮点格式。在 `passthrough` 模式中，有一个端口用于多通道数据，其格式在端口之间进行协商。
###### *链接*
当它们的端口之间有链接时，数据在节点之间流动。链接可能处于 `"passive"` 状态，其中链接的存在不会自动地导致数据在那些节点之间流动 (图中的某些链接必须处于 `"passive"` 状态，才能使图具有数据流)。
###### *设备*
设备是表示底层 API 的句柄，然后用于创建节点或其它设备。设备的例子有 ALSA PCM 声卡或 V4L2 设备。设备有一个配置文件，允许用户配置它们。
###### *工厂*
工厂是一个对象，它的唯一功能是创建其它对象。一旦创建了工厂，它就只能发出它所声明的对象类型。这些通常作为一个模块交付：模块创建工厂并保持活动状态以使客户端可以访问它。
###### *公共参数和方法*
每个对象至少实现 `add_listener` 方法，这允许任何客户端注册事件监听器。事件用于通过 PipeWire API 公开可能会随着时间的推移而改变的对象信息 (比如，节点的状态)。
#### 上下文
PipeWire 服务器和 PipeWire 客户端通过它们各自的 [pw_context](https://docs.pipewire.org/structpw__context.html) 使用 PipeWire API，所谓的 PipeWire 上下文。当 PipeWire 上下文创建时，它根据加载配置文件的规则从文件系统查找并解析配置文件。

参考文档：

[PipeWire and AGL](https://wiki.automotivelinux.org/_media/pipewire_agl_20181206.pdf)

[PipeWire Under The Hood](https://venam.net/blog/unix/2021/06/23/pipewire-under-the-hood.html)

[PipeWire: The Linux audio/video bus (LWN)](https://lwn.net/Articles/847412)

[Intoduction to PipeWire](https://bootlin.com/blog/an-introduction-to-pipewire/)

[A custom PipeWire node](https://bootlin.com/blog/a-custom-pipewire-node/)

[SPA Plugins](https://docs.pipewire.org/page_spa_plugins.html)

[PipeWire document](https://docs.pipewire.org/)

Done.
