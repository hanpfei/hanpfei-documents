---
title: EGLSurfaces 和 OpenGL ES
date: 2017-07-21 11:05:49
categories: Android开发
tags:
- Android开发
- 图形图像
- 翻译
---

OpenGL ES 定义了一个渲染图形的 API。它没有定义窗口系统。为了使 GLES 可以工作于各种平台之上，它被设计为与知道如何通过操作系统创建和访问窗口的库相结合。Android 使用的库称为 EGL。如果你想绘制纹理多边形，你使用 GLES 调用；如果你想将渲染的东西放在屏幕上，则使用 EGL 调用。
<!--more-->
在你可以通过 GLES 做任何事情之前，你需要创建一个 GL 上下文。在 EGL 中，这意味着创建一个 EGLContext 和一个 EGLSurface。GLES 操作应用于当前上下文，其通过线程局部存储访问而不是作为参数传递。这意味着你不得不小心你的渲染代码正在哪个线程中执行，以及那个线程的当前上下文是哪个。

# EGLSurfaces
EGLSurface 可以是一个 EGL 分配的离屏缓冲区 (称为 "pbuffer") 或由操作系统分配的窗口。EGL 窗口 surfaces 由 `eglCreateWindowSurface()` 调用创建。它接收一个 "窗口对象"  作为参数，在 Android 上它可以是一个 SurfaceView，SurfaceTexture，SurfaceHolder 或 Surface -- 所有在底层具有 BufferQueue 的东西。当你执行这个调用时，EGL 创建一个新的 EGLSurface 对象，将它与窗口对象的 BufferQueue 的生产者接口连接。从那时起，向那个 EGLSurface 渲染将使得一个缓冲区被取出，向其中渲染，并放回以由消费者使用。（术语“窗口” 表示预期的用途，但请记住，输出可能不会出现在显示器上。）

EGL 不提供锁定/解锁调用。相反，你发出绘制命令并调用 `eglSwapBuffers()` 来提交当前的帧。方法名来自于传统的交换前后台缓冲区，但实际的实现可能非常不同。

同一时间只有一个 EGLSurface 可以与一个 Surface 关联 -- 你只能有一个生产者与一个 BufferQueue 连接 -- 但是如果你销毁了 EGLSurface 它将从 BufferQueue 断开，并允许其它东西连接。

一个给定的线程可以通过修改什么是当前的 EGLSurface 在多个 EGLSurface 之间切换。同一时间一个 EGLSurface 只能是一个线程的当前 EGLSurface。

与 EGLSurface 有关的最常见的错误是假设它只是 Surface 的另一方面（像 SurfaceHolder 一样）。它是相关但独立的概念。你可以在一个后端没有 Surface 支持的 EGLSurface 上绘制，且你可以不通过 EGL 使用 Surface。EGLSurface 仅仅给了 GLES 一个绘制的平面。

# ANativeWindow
公共的 Surface 类是以 Java 编程语言实现的。C/C++ 中等价的东西是 ANativeWindow ，由 [Android NDK](https://developer.android.com/ndk/index.html) 半曝光出来。你可以通过 `ANativeWindow_fromSurface()` 调用从 Surface 获得 ANativeWindow。就像它的 Java 语言兄弟，你可以锁定它，以软件渲染，然后解锁并post。

要在本地层代码中创建一个 EGL 窗口 surface，你传递一个 EGLNativeWindowType 的实例给 `eglCreateWindowSurface()`。EGLNativeWindowType 只是 ANativeWindow 的一个同义词，因此你可以自由地将一个强制类型转换为另一个。

基本的 “本地窗口” 类型只是封装了一个 BufferQueue 的生产者端的事实应该不会让人感到意外。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-egl-opengl)
