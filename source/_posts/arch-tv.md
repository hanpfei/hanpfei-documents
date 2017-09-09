---
title: TextureView
date: 2017-07-22 21:55:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
- 翻译
---

TextureView 类是在 Android 4.0 中引入的，且是这里讨论的最复杂的 View 对象，它结合了 View 和 SurfaceTexture。
<!--more-->
# 用 GLES 渲染

回忆一下，SurfaceTexture 是一个 "GL 消费者"，消费图形数据缓冲区并使其可用作纹理。TextureView 封装了一个 SurfaceTexture，并接管了响应回调和获取新缓冲区的职责。新缓冲区的到达会导致 TextureView 发出 View invalidate 请求。当被请求绘制时，TextureView 使用最近接收到的缓冲区的内容作为它的数据源，无论何时何地在 View 状态表示它应该渲染时渲染。

你可以像使用 SurfaceView 那样通过 GLES 在 TextureView 上渲染。仅仅将 SurfaceTexture 传递给 EGL 窗口创建调用。然而，这样做暴露了一个潜在的问题。在大部分我们研究的内容中，BufferQueue 已经在不同进程间传递了缓冲区。当用 GLES 向 TextureView 渲染时，生产者和消费者在相同的进程中，且它们甚至可能在同一个线程中处理。假设我们从 UI 线程快速连续地提交了一些缓冲区。EGL 缓冲区交换调用将需要从 BufferQueue 获取一块缓冲区，并且它将停止，直到有可用的缓冲区。在消费者获取一个缓冲区用于渲染之前，将不会有任何可用的缓冲区，但那也会发生在 UI 线程 . . . 所以我们被卡住了。

解决方案是使 BufferQueue 总有一块可用的缓冲区用于获取，因此缓冲区交换从来不会停止。保证这一点的一种方式是使 BufferQueue 在新缓冲区入队时，丢弃之前入队的缓冲区的内容，并对最小缓冲区计数和获取的最大缓冲区计数进行限制。（如果你的队列已经有了三块缓冲区，且所有的三块缓冲区都由消费者获取了，则没有东西可以获取了，且缓冲区交换调用必须挂起或失败。因此我们需要阻止消费者一次获取多于两块缓冲区。）丢弃缓冲区通常是不好的，因此它仅在特定情况下启用，例如生产者和消费者处于相同进程中。

# SurfaceView 或者 TextureView？
SurfaceView 和 TextureView 扮演类似的角色，但实现却差别巨大。为了确定哪个最好，需要对其中的折中有一个了解。

由于 TextureView 是 View 层次体系的一个良好公民，它的行为就像任何其它的 View，且可以覆盖其它元素或被其它元素覆盖。你可以执行任意转换，并可以通过简单的 API 调用将其内容提取为 bitmap。

TextureView 的主要弱点是合成步骤的性能。通过 SurfaceView，内容被写入 SurfaceFlinger 组合的单独的 layer，理想情况下通过一个 overlay。通过 TextureView，View 合成总是由 GLES 执行，且更新它的内容也可能导致其它 View 元素重绘（如果它们的位置在 TextureView 之上的话）。在 View 渲染完成之后，应用程序 UI layer 必须由 SurfaceFlinger 与其它 layers 合成，因此你在有效地合成每个可见的像素两次。对于一个全屏视频播放器，或任何其他实际上只是 UI 元素 layer 在视频之上的应用程序，SurfaceView 提供更好的性能。

如前所述，DRM保护的视频只能在 overlay 平面上呈现。 支持受保护内容的视频播放器必须使用 SurfaceView 实现。

# 案例研究：Grafika 的播放视频 (TextureView)
Grafika 包含了一对视频播放器，一个用 TextureView 实现，另一个用 SurfaceView。视频解码的部分，其仅仅将帧从 MediaCodec 发送到 Surface，两者是相同的。实现之间大部分有趣的差异在请求以正确的长宽比显示的步骤。

尽管 SurfaceView 请求一个定制的 FrameLayout 实现，改变 SurfaceTexture 的大小是一个通过 `TextureView#setTransform()` 配置一个转换矩阵 的简单问题。对于前者，你在通过 WindowManager 向 SurfaceFlinger 发送新的窗口位置和大小值；对于后者，你只是在做不同的渲染。

此外，两种实现按照相同的模式。一旦 Surface 创建好了，播放被启用。当按下 “播放”，视频解码线程被启动，且以 Surface 作为输出目标。之后，应用程序代码不需要做任何事情 -- 合成和显示将由 SurfaceFlinger（对于SurfaceView）或 TextureView 处理。

# 案例研究：Grafika 的双重解码
这个 Activity 演示了 TextureView 内部的 SurfaceTexture 管理。

这个 Activity 的基本结构是一对 TextureViews，它们一边一个展示了两个不同的视频播放。为了模拟视频会议应用的需求，我们想要在 Activity 由于方向改变而 paused 和 resumed 时保持 MediaCodec 解码器存活。技巧在于你不能在不完全重新配置它的情况下改变 MediaCodec 解码器使用的 Surface ，那是一个相当昂贵的操作；因此我们想要保持 Surface 存活。Surface 只是指向SurfaceTexture 的 BufferQueue 中的生产者接口的句柄，且 SurfaceTexture 由 TextureView 管理；因此我们还需要保持 SurfaceTexture 存活。那么我们要如何处理 TextureView 的销毁呢？

恰好 TextureView 提供的 `setSurfaceTexture()` 调用正是我们想要的。我们从 TextureViews 获得 SurfaceTextures 的引用并在静态成员中保存它们。当 Activity 被关闭时，我们从 `onSurfaceTextureDestroyed()` 回调中返回 "false" 来阻止 SurfaceTexture 的销毁。当 Activity 被重启时，我们把老的 SurfaceTexture 放进新的 TextureView 中。TextureView 负责创建并销毁 EGL contexts。

每个视频解码器都是从单独的线程驱动的。乍看起来，似乎我们需要每个线程本地的 EGL contexts；但请记住拥有解码的输出的缓冲区实际上是从 mediaserver 发送到我们的 BufferQueue 消费者（SurfaceTextures）的。TextureViews 负责为我们渲染，且它们在 UI 线程执行。

以 SurfaceView 实现这个 Activity 更难一点。我们不能仅仅创建一对 SurfaceViews 并把输出导向它们，由于 Surfaces 将在屏幕方向转变期间被销毁。此外，那将添加两个 layers，关于可用的 overlays 的数量上的限制强烈地激励我们保持 layers 的数量为最小值。相反，我们想要创建一对 SurfaceTextures 来接收来自于视频解码器的输出，然后在应用中执行渲染，使用 GLES 将两个纹理四边形渲染到 SurfaceView 的 Surface 上。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-tv)
