---
title: Android 图形系统
date: 2017-07-11 11:05:49
categories: Android开发
tags:
- Android开发
- 图形图像
- 翻译
---

Android framework 为2D 和 3D 提供了各种各样的图形渲染 APIs 来与设备制造商的图形驱动实现交互，因此对于那些 API 在上层如何工作有一个好的理解非常重要。这一页介绍驱动基于其构建的图形硬件抽象层 (HAL)。
<!--more-->
应用程序开发者以两种方式将图像绘制到屏幕上：通过 Canvas 或 OpenGL。参考 [系统级图形架构](https://source.android.com/devices/graphics/architecture.html) 来了解 Android 图形组件的详细描述。

[android.graphics.Canvas](http://developer.android.com/reference/android/graphics/Canvas.html) 是一个 2D 图形 API，且是开发者中最流行的图形 API。Canvas 操作在 Android 中绘制所有的 stock 和 custom [android.view.View](http://developer.android.com/reference/android/view/View.html)s。在 Android 中，Canvas APIs 的硬件加速通过称为 OpenGLRenderer 的绘图库来完成，它将 Canvas 操作转换为 OpenGL 操作，以使它们可以在 GPU 上执行。

自 Android 4.0 开始，硬件加速的 Canvas 是默认开启的。因此，对于 Android 4.0 及更新的设备，支持 OpenGL ES 2.0 的应该 GPU 是必须的。参考 [硬件加速指南](https://developer.android.com/guide/topics/graphics/hardware-accel.html) 来了解硬件加速绘制路径如何工作，以及它与软件绘制路径之间在行为上的不同。

除了 Canvas，开发者渲染图形的另外一个主要方式是使用 OpenGL ES 直接渲染到一个 surface。Android 在 [android.opengl](http://developer.android.com/reference/android/opengl/package-summary.html) 包中提供了 OpenGL ES 接口，开发者可以使用 SDK 或
 [Android NDK](https://developer.android.com/tools/sdk/ndk/index.html) 中提供的本地层 APIs 来调用他们的 GL 实现。

Android 实现者可以使用  [drawElements Quality Program](https://source.android.com/devices/graphics/testing.html)，也称为 deqp 测试 OpenGL ES 功能。

# Android 图形组件

无论开发者使用什么渲染 API，所有的东西都是在一个 "surface" 上渲染的。surface 表示一个 buffer queue 的生产者端，而 buffer queue 常常由 SurfaceFlinger 消费。Android 平台上创建的每个窗口都是由一个 surface 支持的。所有可见的 surfaces 渲染由 SurfaceFlinger 组合到显示设备上。

下面的图展示了关键组件如何一起工作：

![ape_fwk_graphics.png](https://www.wolfcstech.com/images/1315506-66a2aeedf0bd905a.png)

**图 1.** surfaces 如何渲染

主要的组件在下面描述：

## 图像流生产者

图像流生产者可以是为消费生产图形 buffers 的任何东西。例子包括 OpenGL ES，Canvas 2D，和 mediaserver 视频解码器。

## 图像流消费者

最常见的图形流消费者是 SurfaceFlinger，消费当前可见的 surfaces 并使用
 Window Manager 提供的信息将它们组合到显示设备上的系统服务。SurfaceFlinger 是仅有的可以修改显示设备内容的服务。SurfaceFlinger 使用 OpenGL 和 Hardware Composer 组合一组 surfaces。

其它的 OpenGL ES 应用也可以消费图像流，比如 camera 应用消费一个 camera 预览图像流。非 GL 应用也可以是消费者，比如 ImageReader 类。

## Window Manager

控制窗口的 Android 系统服务，窗口是 views 的容器。一个窗口总是由一个 surface 支持的。这个服务监督生命周期，输入和焦点事件，屏幕分辨率，过渡，动画，位置，转换，z-order，和一个窗口的许多其它方面。Window Manager 发送所有的 window metadata 给 SurfaceFlinger，以使 SurfaceFlinger 可以使用那些数据在显示设备上组合 surface。

## Hardware Composer

显示子系统的硬件抽象。SurfaceFlinger 将某些组合工作委托给 Hardware Composer，来从 OpenGL 和 GPU 卸载工作。SurfaceFlinger 只是扮演了另一个 OpenGL 客户端的角色。因此当 SurfaceFlinger 正在活跃地组合一个 buffer 或两个到第三个时，比如，它正在使用 OpenGL ES。相对于让 GPU 主导所有的计算，这使组合工作的功耗更低。

[Hardware Composer HAL](https://source.android.com/devices/graphics/architecture.html#hwcomposer) 执行另一半工作，且是所有 Android 图形渲染的中心点。Hardware Composer 必须支持事件，其中一个是 VSYNC（另一个是 plug-and-playHDMI 支持的 hotplug）。

## Gralloc

图形内存分配器 (Gralloc) 用来分配图像生产者请求的内存。更多详情，请参考 [Gralloc HAL](https://source.android.com/devices/graphics/architecture.html#gralloc_HAL)。

# 数据流

参看下图来获得 Android 图形流水线的描述：

![](https://www.wolfcstech.com/images/1315506-8e6633ba459ba6b7.png)

**图 2.** Android 中的图形数据流

左边的对象是渲染器生产图形 buffers，比如 home screen，状态栏，system UI。SurfaceFlinger 是制图者，Hardware Composer 是合成器。

## BufferQueue
BufferQueues 提供了 Android 图形组件之间的胶水。有一对队列中转从生产者到消费者的固定的循环缓冲区。一旦生产者切换了它们的 buffers，SurfaceFlinger 负责将所有的东西合成到显示设备上。

参考下面的图来了解 BufferQueue 的通信过程。

![](https://www.wolfcstech.com/images/1315506-0c6a0f79be572037.png)

**图 3.** BufferQueue 的通信过程

BufferQueue 包含了将图像流生产者和图像流消费者绑在一起的逻辑。图像生产者的一些例子是 camera 生产 camera 预览或 OpenGL ES 游戏。图像消费者的一些例子是 SurfaceFlinger 或显示 OpenGL ES 流的另一个应用，比如 camera 应用显示 camera viewfinder。

BufferQueue 是结合 buffer 池和一个队列，并使用 Binder IPC 在进程间传递 buffers 的数据结构。生产者接口，或传递给想要生成图形缓冲区的人的东西，是 IGraphicBufferProducer ( [SurfaceTexture](http://developer.android.com/reference/android/graphics/SurfaceTexture.html) 的一部分)。BufferQueue 常常被用于渲染到一个 Surface，并用一个 GL Consumer 消费及其他任务。BufferQueue 可以三种模式操作：

*同步式模式* -  BufferQueue 默认以同步式模式操作，在这种模式下每个来自于生产者的 buffer 出去到消费者。这种模式下没有 buffer 会被丢弃。且如果生产者太快，创建 buffers 的速度比消费的速度快，它将阻塞并等待释放 buffers。

*非阻塞模式* -  BufferQueue也可以以非阻塞模式操作，这种模式中在那些情况下它产生一个 error 而不是等待一个 buffer。这种模式下没有 buffer 会被丢弃。这对在那些可能不理解图形框架的复杂依赖的应用中避免潜在的死锁很有用。

*丢弃模式* -  最后，BufferQueue 可以被配置为丢弃旧的 buffers 而不是产生 errors 或等待。比如，如果执行 GL 渲染到一个 texture view 并绘制的尽可能块，buffers 必须被丢弃。

为了执行这些中的大部分工作，SurfaceFlinger 只是扮演了另一个 OpenGL 客户端的角色。因此当 SurfaceFlinger 正在活跃地组合一个 buffer 或两个到第三个时，比如，它正在使用 OpenGL ES。

Hardware Composer HAL 执行另一半工作，且是所有 Android 图形渲染的中心点。

## 同步框架

由于 Android 图形没有提供显式的并行，厂商一直在自己的驱动程序中实现自己的隐式同步。在 Android 图形同步框架下不需要了。参考 [Explicit synchronization](https://source.android.com/devices/graphics/implement-vsync.html#explicit_synchronization) 一节了解实现指导。

同步框架显式地描述系统中不同异步操作间的依赖。框架提供了一个简单的 API 让组件在 buffers 释放时发送信号。它也允许同步原语在驱动间从内核到用户空间及用户空间进程本身间进行传递。

例如，应用程序可以排队在 GPU 中执行的工作。然后 GPU 开始绘制那幅图。尽管图像还没有被画进内存，buffer 指针依然可以伴随着指示何时 GPU
 工作将结束的栅栏传递给 window 合成器。然后 window 合成器可以提前开始处理，并将工作移交给显示控制器。通过这种方式，GPU 的工作可以提前结束。一旦 GPU 结束，显示控制器可以立即显示图像。

同步框架也允许实现者在它们自己的硬件组件中利用同步资源。最后，框架提供对图形管道的可见性，以帮助调试。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/)
