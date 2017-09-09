---
title: Surface 和 SurfaceHolder
date: 2017-07-21 10:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
- 翻译
---

[Surface](http://developer.android.com/reference/android/view/Surface.html) 类自 1.0 版本开始就是公共 API 的一部分了。它的描述简单地说，“处理由屏幕合成器管理的原始缓冲区”。该陈述在最初编写时是准确的，但在现代系统上却与事实相去甚远。

Surface 表示一个常常（但不总是！）由 SurfaceFlinger 消费的 buffer queue 的生产者端。当你渲染到 Surface 上时，结果最终将进入被传递给消费者的缓冲区中。Surface 不简单地是一块你可以随意涂鸦的原始内存块。
<!--more-->
显示 Surface 的 BufferQueue 通常配置为三重缓冲；但缓冲区是根据需要分配的。因此如果生产者产生缓冲区的速度足够慢 - 可能它是在 60fps 的显示器上执行 30fps 的动画 - 队列中可能只有分配的两块内存。这可能有助于最小化内存消耗。你可以在 `dumpsys SurfaceFlinger` 的输出中看到与每个 layer 关联的缓冲区的摘要。

# Canvas 渲染
曾经一段时间，所有的渲染都是通过软件完成的，今天你依然可以这样做。底层的实现是由 Skia 图形库提供的。如果你想绘制一个矩形，你执行一个库调用，然后它将适当地设置缓冲区中的字节。为了确保不会有两个客户端同时更新一块缓冲区，或在显示时被写入，你不得不锁定缓冲区然后访问它。`lockCanvas()` 锁定缓冲区并返回一个 Canvas 用于绘制，然后 `unlockCanvasAndPost()` 解锁缓冲区并把它发送给合成器。

随着时间的推移，带有通用 3D 引擎的设备出现了，Android 围绕 OpenGL ES 重新定位。然而，对于应用及应用框架代码，保持老的 API 正常工作非常重要，所以努力进行
 Canvas API 的硬件加速化。正如你在 [硬件加速](http://developer.android.com/guide/topics/graphics/hardware-accel.html) 页的图中所看到的那样，这是一段颠簸的旅程。特别要注意的是尽管为 View 的 `onDraw()` 方法提供的 Canvas 可能是硬件加速化了的，当一个应用通过 `lockCanvas()` 直接锁定 Surface 时获得的 Canvas 则从不是。

当你锁定 Surface 获得 Canvas 的访问权限时，“CPU 渲染器” 连接到 BufferQueue 的生产者端且直到 Surface 被销毁才断开。大多数其它的生产者（比如 GLES）可以被断开并重连到 Surface，但基于 Canvas 的 “CPU 渲染器” 不能。这意味着如果你曾经为了一个 Canvas 锁定了它，你就不能用 GLES 在一个 surface 上绘制，或者从视频解码器向它发送帧。

生产者第一次从 BufferQueue 请求数据缓冲区时，它被分配并被初始化为 0。为了避免进程间无意的数据共享，初始化是必须的。当你复用缓冲区时，然而，之前的内容将依然存在。如果你重复地调用 `lockCanvas()` 和 `unlockCanvasAndPost()` 而不绘制任何东西，你将在先前渲染的帧之间循环。

Surface 锁定/解锁代码持有一个到先前渲染的缓冲区的引用。当锁定 Surface 时你指定了一个 dirty 区域，它将从先前的缓冲区中拷贝非 dirty 的像素。缓冲区有可能由 SurfaceFlinger 或 HWC 处理；但是由于我们只需要从中读取，所以无需等待独占访问。

应用主要的直接向 Surface 绘制的非 Canvas 方式是通过 OpenGL ES。 [EGLSurface 和 OpenGL ES](https://source.android.com/devices/graphics/arch-egl-opengl) 的部分将描述这些。

# SurfaceHolder
一些使用 Surfaces 的东西需要一个SurfaceHolder，特别是SurfaceView。最初的想法是 Surface 表示原始的合成器管理的缓冲区，而 SurfaceHolder 由应用管理并追踪更高层的信息，比如尺寸和格式。Java 语言定义镜像了底层本地的实现。以这种方式分裂它可能不再有用，但它一直是公共API的一部分。

一般来说，任何与 View 有关的东西将被包含进 SurfaceHolder。一些其它的 APIs，比如 
MediaCodec，将在 Surface 上运行。你可以简单地从 SurfaceHolder 获得 Surface，因此当你拥有 Surface 时，请将其挂在后者上。

获取和设置 Surface 参数的 APIs，比如大小和格式，是通过 SurfaceHolder 实现的。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-sh)
