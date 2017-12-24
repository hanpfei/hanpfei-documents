---
title: BufferQueue 和 gralloc
date: 2017-07-17 20:15:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
- Android 图形系统
- 翻译
---

理解 Android 图形系统，我们从场景背后的 BufferQueue 和 gralloc HAL 开始。

BufferQueue 类是 Android 中所有图形的核心。它的角色很简单：连接产生图形数据缓冲区的东西（*生产者*）和接受数据来显示或进一步处理的东西（*消费者*）。几乎所有在系统中移动图形数据缓冲区的东西都依赖于 BufferQueue。
<!--more-->
gralloc 内存分配器执行缓冲区分配，且通过一个供应商特有的 HAL 接口（参考 `hardware/libhardware/include/hardware/gralloc.h`）实现。`alloc()` 期待接收的参数为 (width, height, pixel format) 及一系列使用标记（下面详述）。

# BufferQueue 生产者和消费者

基本的用法很直接：生产者请求一块空闲的缓冲区 (`dequeueBuffer()`)，指定一系列特性，包括宽度，高度，像素格式，和使用标记。生产者填充缓冲区，并将它返回给队列 (`queueBuffer()`)。随后，消费者获得缓冲区(`acquireBuffer()`)  并使用缓冲区的内容。当消费者完成时，它将缓冲区返回给队列(`releaseBuffer()`)。

最近 Android 设备支持 *同步框架*，这使得系统可以与能够异步处理图形数据的硬件组件结合使用。比如，生产者可以提交一系列 OpenGL ES 绘制命令，然后在渲染完成之前加入输出缓冲区队列。缓冲区伴随着内容准备就绪时发出信号的栅栏。当缓冲区返回到空闲列表时，第二个栅栏随附缓冲区，因此消费者可以释放缓冲区，同时内容仍在使用中。当缓冲区移动通过系统时，这种方法可以提升延迟和吞吐量。

队列的一些特性，比如它可以持有的最大缓冲区数量，由生产者和消费者联合决定。然而，BufferQueue 负责根据需要分配缓冲区。除非特性改变，否则缓冲区将保留；比如，如果生产者请求了一个大小不同的缓冲区，老的缓冲区将释放，新的缓冲区将根据需要分配。

生产者和消费者可以位于不同的进程中。当前，消费者总是创建并拥有数据结构。在更老的版本中，只有生产者一端是 binder 化的 (比如生产者可以在一个远程进程中，但消费者必须位于队列创建的进程中)。Android 4.4 及之后的发行版采用了一个更加通用的实现。

BufferQueue 从来不拷贝缓冲区的内容 (像那样移动大量数据将是非常低效的)。相反，缓冲区总是通过句柄传递。

# gralloc HAL 使用标记

gralloc 分配器不仅仅是另外一种在本地堆上分配内存的方式；在某些情形下，分配的内存可能不是高速缓存一致的，或者可能完全不能从用户空间访问。分配的性质由使用标记决定，这包括这样的一些属性：

 * 从软件访问内存的频率有多高 (CPU)
 * 从硬件访问内存的频率有多高 (GPU)
 * 内存是否会被用作 OpenGL ES (GLES) 纹理
 * 内存是否会被视频编码器使用

比如，如果你的格式指定 RGBA 8888 像素，然后你指出缓冲区将从软件访问 (意味着你的应用将直接接触像素)，然后分配器必须一个缓冲区，其中每像素 4 个字节，且以 R-G-B-A 的顺序。相反，如果你说缓冲区将只从硬件访问，并作为一个 GLES 纹理，分配器可以做任何 GLES 驱动想要的事情 － BGRA 顺序，非线性布局，替代颜色格式，等等。允许硬件使用它喜欢的格式可以提升性能。

一些值在某一平台上无法结合。比如，视频编码器标记可以请求 YUV 像素，于是添加软件访问和指定 RGBA 8888 将失败。

gralloc 分配器返回的句柄可以通过 Binder 在进程之间传递。

# 使用systrace跟踪BufferQueue

要真正理解图形缓冲区如何移动，则使用 systrace。系统级的图形代码可以很好地进行探索，就像许多相关的应用程序框架代码一样。如何高效使用 systrace 的完整描述将需要一篇相当长的文档。首先启用 `gfx`，`view` 和 `sched` 标签。你也将在 trace 中看到 BufferQueues。如果你之前已经在使用 systrace 了，你可能已经看到过它们但可能不确定它们是什么。举个例子，如果你在 [Grafika's](https://github.com/google/grafika) "Play video (SurfaceView)" 运行时获取 trace，标签为 * SurfaceView* 的行告诉你在任何给定时刻有多少缓冲区被加入队列。

值在应用活跃时增加 － MediaCodec 解码器触发帧的渲染 － 而在 SurfaceFlinger 工作，消费缓冲区时减小。当视频的帧率为 30 fps 时，队列的值将在 0 到 1 之间变动，因为 ~60 fps 显示可以轻松地跟踪源。(还要注意 SurfaceFlinger 只有在有工作要做时才唤醒，而不是美妙 60 次。系统努力试图避免工作，并且如果没有东西更新屏幕的话完全禁用 VSYNC。)

如果你切换到 Grafika's "Play video (TextureView)" 并获取一个 trace，你将看到标签为 com.android.grafika/com.android.grafika.PlayMovieActivity 的行。这是主 UI 层，它只是另一个 BufferQueue。由于 TextureView 渲染到 UI 层 (而不是一个分离的层)，你将在这里看到所有的视频驱动的更新。

关于 systrace 工具的更多信息，请参考 [Systrace documentation](https://developer.android.com/studio/profile/systrace-commandline.html)。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-bq-gralloc)
