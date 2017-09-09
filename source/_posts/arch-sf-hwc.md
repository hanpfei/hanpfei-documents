---
title: SurfaceFlinger 和 Hardware Composer
date: 2017-07-21 9:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
- 翻译
---

拥有图形数据缓冲区是很精彩的，但是当你在你的设备屏幕上看到它们时生活甚至更美好。那就是 SurfaceFlinger 和 Hardware Composer HAL 做的事情。
<!--more-->
# SurfaceFlinger
SurfaceFlinger 的角色是接收来自于多个源的数据缓冲区，组合它们，并将它们发送给显示设备。曾经一段时期这是通过软将数据块传送到硬件 framebuffer (比如 /dev/graphics/fb0) 完成的，但那些日子已经过去许久了。

当一个应用来到前台，WindowManager 服务向 SurfaceFlinger 请求一块绘制 surface。SurfaceFlinger 创建一个 layer（其主要的组件是一个 BufferQueue），而
 SurfaceFlinger 将作为其消费者。生产者端的 Binder 对象通过 WindowManager 被传递给应用，然后它可以开始直接向 SurfaceFlinger 发送帧。

注意：这一节使用 SurfaceFlinger 术语，WindowManager 使用术语 *window* 而不是 *layer*. . . 并使用 layer 表示其它一些东西。( 可以认为 SurfaceFlinger 应该被称为LayerFlinger。)

大多数应用于任何时间在屏幕上具有三个 layers：屏幕顶部的状态栏，底部或侧面的导航栏，以及应用程序 UI。一些应用具有更多，一些更少（比如默认的 home 应用有一个单独的壁纸 layer，而全屏游戏可能会隐藏状态栏）。每个 layer 可以被独立地更新。状态栏和导航栏由一个系统进程渲染，而应用 layers 有应用渲染，两者之间没有协调。

设备显示器以一定频率刷新，在手机和平板上典型的是每秒钟 60 帧。如果显示的内容在刷新中间更新，则将看到花屏；因此只在周期中间更新内容很重要。当可以安全更新内容时，系统收到来自于显示器的信号。出于历史原因我们称它为 VSYNC 信号。

刷新频率可能会随着时间而改变，比如依赖于当前的条件，一些设备的范围在 58 到 62fps。对于 HDMI 连接的电视机，理论上可以下降到 24 或 48Hz 来匹配视频。由于每个刷新周期我们只能更新屏幕一次，以 200 fps 提交缓冲区来显示将是巨大的浪费，因为大多数帧将从不会被看到。

不是在应用提交缓冲区时采取行动，SurfaceFlinger 而是在显示器为显示一些新东西做好准备时才唤醒。

当 VSYNC 信号到达时，SurfaceFlinger 遍历它的 layers 列表寻找新的缓冲区。如果它找到了一个新的，它获取它；如果没有，它继续使用之前获取的缓冲区。SurfaceFlinger 总是想要一些东西来显示，因此它会挂在一个缓冲区上。如果没有缓冲区已经提交给一个 layer，则该 layer 被忽略。

在 SurfaceFlinger 收集了所有可见的 layers 的缓冲区之后，它询问 Hardware Composer 应该如何执行组合。

# Hardware Composer
Hardware Composer HAL (HWC) 在 Android 3.0 中被引入，并已经稳定发展多年。它的主要目标是通过可用硬件确定组合缓冲区的最有效方式。作为 HAL，其实现是特定于设备的，且通常由显示设备硬件 OEM 完成。

当考虑 *覆盖平面(overlay planes)* 时，这种方法的价值很容易识别，其目的是在显示硬件而不是 GPU 中将多个缓冲区组合在一起。比如，考虑一个典型的竖直方向的 Android 手机，其状态栏在顶部，导航栏在底部，应用内容在其余的地方。每个 layers 的内容在单独的缓冲区中。你可以使用下列方法中的一种处理组合：

 * 将应用内容渲染到暂存缓冲区中，然后将状态栏渲染在它的上面，导航栏位于其上，最后将暂存缓冲区传递给显示硬件。

 * 将所有三个缓冲区传递给显示硬件，并告诉它从不同的缓冲区为不同的屏幕部分读取数据。

后一种方法可以显着提高效率。

显示处理器功能差异很大。overlays 的数量，layers 是否可以旋转或混合，以及位置和重叠上的限制，可能非常难以通过一个 API 来描述。HWC 试图通过一系列的决定容纳这些多样性：

 1. SurfaceFlinger 为 HWC 提供完整的 layers 的列表并询问，“你想要如何处理它？”。

 2. HWC 通过将每个 layer 标记为 overlay 或 GLES composition 来进行响应。

 3. SurfaceFlinger 关心任何 GLES composition，并把输出缓冲区传给 HWC，让 HWC 处理其余的事情。

由于硬件供应商可以定制或裁剪决定作出的代码，因此可以从每个设备中获得最佳性能。

当屏幕上的东西没有改变时，overlay 平面可能比 GL composition 更低效。当 overlay 内容具有透明像素且覆盖的 layers 被混合在一起时尤其如此。在这种情况下，HWC 可以选择为一些或所有 layers 请求 GLES composition 并保留组合的缓冲区。如果 SurfaceFlinger 回来请求组合相同的缓冲区集合，HWC 可以继续展示之前组合好的临时缓冲区。这可以提升空闲的设备的电池续航能力。

运行 Android 4.4 及更新版本的设备典型地支持四个 overlay 平面。试图组合比 overlays 更多的 layers 会导致系统为它们中的一些使用 GLES composition，这意味着一个应用使用的 layers 的数量可能对电源消耗和性能有着重大的影响。

# 虚拟显示器
SurfaceFlinger 支持一个主显示器（比如手机或平板内置的东西），一个外部显示器（比如通过 HDMI 连接的电视机），以及一个或多个使组合后的输出在系统中可用的虚拟显示器。虚拟显示器可用于记录屏幕或通过网络发送。

虚拟显示器可以共享主显示器相同的 layers 集合（layer 栈）或拥有它们自己的集合。虚拟显示器没有 VSYNC，因此主显示器的 VSYNC 用于触发所有显示器的组合
 (composition) 。

在老版本的 Android 中，虚拟显示器总是通过 GLES 组合，而 Hardware Composer 只管理主显示器的组合。在 Android 4.4 中，Hardware Composer 获得了参与虚拟显示器组合的能力。

如你期待的那样，为一个虚拟显示器产生的帧被写入 BufferQueue。

# 案例研究：screenrecord
[screenrecord 命令](https://android.googlesource.com/platform/frameworks/av/+/marshmallow-release/cmds/screenrecord/) 允许你将屏幕上出现的所有东西记录为磁盘上的 .mp4 文件。为了实现它，我们不得不从 SurfaceFlinger 接收组合之后的帧，将它们写入视频编码器，然后将编码的视频数据写入一个文件。视频编解码由一个单独的进程 (mediaserver) 管理，因此我们不得不在系统中移动巨大的图形缓冲区。使这件事情更具挑战性的是，我们还要试图以全解析度记录 60fps 的视频。使这件事情高效工作的关键是 BufferQueue。

MediaCodec 类允许一个应用以缓冲区中的原始字节提供数据，或通过一个 [Surface](https://source.android.com/devices/graphics/arch-sh.html)。当 screenrecord 请求访问一个视频编码器时，mediaserver 创建一个 BufferQueue，将它自己与消费者一端相连，然后将生产者端作为一个 Surface 传回给 screenrecord。

screenrecord 命令然后请求 SurfaceFlinger 创建一个镜像主显示器的虚拟显示器 (比如它具有所有相同的 layers)，然后指示它将输出发送给来自于 mediaserver 的 Surface。在这种情况下，SurfaceFlinger 是缓冲区的生产者而不是消费者。

配置完成之后，screenrecord 等待编码的数据出现。随着应用的绘制，它们的缓冲区进入 SurfaceFlinger，SurfaceFlinger 将它们组合为一个单独的缓冲区并直接发送给 mediaserver 中的视频编码器。screenrecord 进程甚至从来不会看到完整的帧。在内部，mediaserver 有着它自己的移动缓冲区的方式，即通过句柄传递数据，以最小化开销。

# 案例研究：模拟二次显示
WindowManager 可以请求 SurfaceFlinger 创建一个可见的 layer，其中 SurfaceFlinger 作为 BufferQueue 的消费者。它还可以请求 SurfaceFlinger 创建一个虚拟显示器，其中 SurfaceFlinger 作为 BufferQueue 的生产者。如果你将它们连接到一起会如何呢，配置一个虚拟显示器显示渲染到一个可见 layer 的东西？

你创建了一个闭环，其中组合后的屏幕出现在一个窗口中。然后窗口现在是组合后的输出的一部分了，因此在下一次刷新窗口内组合后的图像将也显示窗口的内容（然后 [海龟下面还是海龟一路下来](https://en.wikipedia.org/wiki/Turtles_all_the_way_down)）。为了看到这种行为，启用设置中的 [Developer options](http://developer.android.com/tools/index.html)，选择 **Simulate secondary displays**，并启用一个窗口。为了加分，请使用 screenrecord 捕获启用显示的行为，然后逐帧播放。 

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-sf-hwc)
