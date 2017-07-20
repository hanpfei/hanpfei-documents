---
title: Android 图形架构
date: 2017-07-11 19:05:49
categories: Android开发
tags:
- Android开发
- 图形图像
- 翻译
---

*每一个开发者都应该了解的关于 Surface，SurfaceHolder，EGLSurface，SurfaceView，GLSurfaceView，SurfaceTexture，TextureView，SurfaceFlinger，和 Vulkan 的东西。*
<!--more-->
本页描述 Android 系统级图形架构的必要元素及应用框架和多媒体系统如何使用它们。重点是图形数据的缓冲区如何在系统中移动。如果你曾经想知道为什么 SurfaceView 和 TextureView 有着那样的行为，或 Surface 和 EGLSurface 如何交互，那你就来对地方了。

假设读者熟悉 Android 设备和应用开发。你不需要关于应用框架的详细知识，这里只会提到少量的 API 调用，但这里的资料与其它公开的文档不重叠。目标是提供关于渲染一帧用以输出中牵涉的重要事件的详情，以帮助你在设计应用时做出明智的选择。为了实现这一点，我们从底层开始，描述 UI 类如何工作，而不是它们如何使用。

这一节包含几页，覆盖了从背景资料到 HAL 细节到使用案例的所有东西。这里从解释 Android 图形缓冲区开始，描述合成和显示机制，然后是提供了数据合成器的更高层机制。我们建议你按下面列出的顺序阅读，而不是直接跳到听起来很有趣的主题。

# 底层组件

 * [BufferQueue 和 gralloc](https://source.android.com/devices/graphics/arch-bq-gralloc.html)。BufferQueue 连接生成图形数据缓冲区的东西（*生产者*）和接收数据来显示或进一步处理的东西（*消费者*）。缓冲区分配通过 *gralloc* 内存分配器执行， *gralloc* 内存分配器通过一个特定于供应商的 HAL 接口实现。

 * [SurfaceFlinger，Hardware Composer，和虚拟显示设备](https://source.android.com/devices/graphics/arch-sf-hwc.html)。SurfaceFlinger 接受来自于多个源的数据缓冲区，组合它们，然后发送给显示设备。Hardware Composer HAL (HWC) 决定通过可用硬件和虚拟显示设备以最高效的方式组合缓冲区使合成的输出在系统内可用（记录屏幕或通过网络发送屏幕）。

 * [Surface，Canvas，和 SurfaceHolder](https://source.android.com/devices/graphics/arch-sh.html)。Surface 生产常常由 SurfaceFlinger 消费的缓冲区队列。当在一个 Surface 上渲染时，结果最终被放入缓冲区中，并被传送给消费者。Canvas APIs 为直接在一个 Surface （OpenGL ES 的低级替代品）绘制提供了一个软件实现（通过硬件加速支持）。与 View 有关的任何事情都涉及到 SurfaceHolder，通过它的 API 可以获取或设置 Surface 参数，比如大小和格式等。

 * [EGLSurface 和 OpenGL ES](https://source.android.com/devices/graphics/arch-egl-opengl.html)。OpenGL ES (GLES) 定义了一个与 EGL 结合使用的图形渲染 API，一个知道如何通过操作系统创建和访问窗口的库（绘制纹理多边形，使用 GLES 调用；在屏幕上放置渲染结果，使用EGL调用；）。这页还描述了 ANativeWindow，Java Surface 类的 C/C++ 等价物，用于在本地层代码中创建一个 EGL 窗口。

 * [Vulkan](https://source.android.com/devices/graphics/arch-vulkan.html)。Vulkan 是一个低开销，跨平台的高性能 3D 图形 API。像 OpenGL ES 一样，Vulkan 提供了在应用中创建高品质实时图形的工具。Vulkan 的优势包括降低 CPU 开销并支持 [SPIR-V Binary Intermediate](https://www.khronos.org/spir) 语言。

# 高层组件

 * [SurfaceView 和 GLSurfaceView](https://source.android.com/devices/graphics/arch-sv-glsv.html)。SurfaceView 结合了一个 Surface 和一个 View。SurfaceView 的 View 组件由 SurfaceFlinger 合成 (而不是应用)，可以在一个分开的线程/进程中渲染，并与应用的 UI 渲染隔离。GLSurfaceView 提供了辅助类来管理 EGL contexts，线程间通信，及与 Activity 生命周期的交互（但使用 GLES 不是必需的）。

 * [SurfaceTexture](https://source.android.com/devices/graphics/arch-st.html)。SurfaceTexture 结合了一个 Surface 和 GLES texture 来创建一个 BufferQueue，你的应用是该 BufferQueue 的消费者。当生产者入队了一个新 buffer，它通知你的应用，这反过来释放了之前持有的 buffer，从队列中获取新的 buffer，执行 EGL 调用使得 buffer 可以作为 GLES 的一个外部 texture 使用。Android 7.0 添加了对安全 texture video playback的支持，使得 GPU 可以后处理受保护的视频内容。

 * [TextureView](https://source.android.com/devices/graphics/arch-tv.html)。TextureView 结合了一个 View 和一个 SurfaceTexture。TextureView 包装了一个 SurfaceTexture 并负责响应回调和获取新 buffers。当绘制时，TextureView 使用最近接收的 buffer 的内容作为它的数据源，根据 View 的状态绘制它。View 合成总是由 GLES 执行，意味着更新内容可能导致其它的 View 元素也被重绘。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/architecture)
