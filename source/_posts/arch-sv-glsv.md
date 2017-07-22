---
title: SurfaceView 和 GLSurfaceView
date: 2017-07-22 11:05:49
categories: Android开发
tags:
- Android开发
- 图形图像
- 翻译
---

Android 应用框架 UI 是基于一个从 View 开始的对象层次体系的。所有的 UI 元素经历一个复杂的测量和布局过程来将它们适配入一个矩形区域，所有可见的 View 对象被渲染进一个 SurfaceFlinger 创建的 Surface，而后者由 WindowManager 在应用程序回到前台时建立。应用程序的 UI 线程执行布局并渲染进一个单独的缓冲区（无论 Layouts 和 Views 的个数，也不管 View 是否是硬件加速的）。
<!--more-->
SurfaceView 如同其它 views 一样接收相同的参数，因此你可以给它一个位置和大小，然后围绕它适配其它元素。当它要渲染时，然而，其内容完全是透明的；SurfaceView 的View 部分只是一个透视占位符。

当 SurfaceView 的 View 组件即将可见时，框架请求 WindowManager 请求 SurfaceFlinger 创建一个新的 Surface。（这不是同步发生的，这也是为什么你应该提供一个回调，在 Surface 创建完成时通知你。）默认情况下，新的 Surface 位于应用程序 UI Surface 的后面，但默认的 Z-ordering 可以被覆盖来将 Surface 放到顶部。

无论你向 Surface 渲染了什么，它们都将由 SurfaceFlinger 合成，而不是由应用程序。这正是 SurfaceView 真正强大的地方：你获得的 Surface 可以由单独的线程或单独的进程渲染，完全与由应用程序 UI 执行的任何渲染隔离，而缓冲区直接进入 SurfaceFlinger。你不能完全忽略 UI 线程 - 你依然不得不与 Activity 的生命周期合作，且如果 View 的大小或位置改变了你可能不得不调整一些东西 - 但你自己完全拥有整个 Surface。与应用程序 UI 和其它 layers 的混合由 Hardware Composer 处理。

新的 Surface 是 BufferQueue 的生产者端，其消费者是 SurfaceFlinger 的 layer。你可以以任何能够喂养 BufferQueue 的机制更新 Surface，如 Canvas 函数提供的 surface，附上一个 EGLSurface 并用 GLES 绘制它，或配置一个 MediaCodec 视频解码器写入它。

# 合成和硬件 Scaler
让我们更近地看一下 `dumpsys SurfaceFlinger`。下面的输出是在竖屏的 Nexus 设备上，在 Grafika 的 "Play video (SurfaceView)" activity 播放一个电影时获取的；视频是 QVGA (320x240) 的：
```
    type    |          source crop              |           frame           name
------------+-----------------------------------+--------------------------------
        HWC | [    0.0,    0.0,  320.0,  240.0] | [   48,  411, 1032, 1149] SurfaceView
        HWC | [    0.0,   75.0, 1080.0, 1776.0] | [    0,   75, 1080, 1776] com.android.grafika/com.android.grafika.PlayMovieSurfaceActivity
        HWC | [    0.0,    0.0, 1080.0,   75.0] | [    0,    0, 1080,   75] StatusBar
        HWC | [    0.0,    0.0, 1080.0,  144.0] | [    0, 1776, 1080, 1920] NavigationBar
  FB TARGET | [    0.0,    0.0, 1080.0, 1920.0] | [    0,    0, 1080, 1920] HWC_FRAMEBUFFER_TARGET
```

 * **排列顺序** 是自后向前的：SurfaceView 的 Surface 在后边，应用程序 UI layer 位于它的顶部，然后是状态栏和导航栏在所有其它东西上面。

 * **source crop** 值表示 SurfaceFlinger 将显示的 Surface 的缓冲区的部分。应用程序UI 被赋予了一个等于显示器全尺寸（1080x1920）的 Surface，但是由于被状态栏和导航栏遮挡的像素不会被渲染和合成，所以源被裁剪为一个矩形，从距顶部 75 像素的开始并在距底部144个像素的地方结束。状态栏和导航栏具有更小的 Surface，且 source crop 描述了一个矩形，从左上角的 (0,0) 开始并跨越它们的内容。

 * **frame** 值描述了像素在显示器上显示的矩形。对于应用程序 UI layer，frame 与 source crop 匹配，因为我们正在将显示大小的 layer 的一部分复制（或叠加）到另一个显示大小的 layer 中的相同位置。对于状态栏和导航栏，帧矩形的大小是相同的，但位置要调整以使导航栏出现在屏幕的底部。

 * **SurfaceView layer** 持有我们的视频内容。source crop 与视频大小匹配，SurfaceFlinger 知道该大小，因为 MediaCodec 解码器（缓冲区的生产者）在获取该大小的缓冲区。帧矩形具有完全不同的大小 —— 984x738。

SurfaceFlinger 通过放缩缓冲区的内容来处理大小的差异以适配帧矩形，放大还是缩小则根据需要。之所以选择这个特定的大小是由于它与视频有着相同的长宽比 (4:3)，且在满足 View 布局约束的条件下尽可能的宽（这包括为了美学原因而在屏幕边缘包含的一些填充）。

如果你在相同的 Surface 上起动播放了不同的视频，则底层的 BufferQueue 将自动地重新分配缓冲区为新的大小，SurfaceFlinger 将调整 source crop。如果新视频的长宽比不同，应用将需要强制 re-layout 该 View 来与之匹配，这将导致 WindowManager 告知 SurfaceFlinger 去更新帧矩形。

如果你正在通过一些其它的方式（比如 GELS）在 Surface 上渲染，你可以使用 `SurfaceHolder#setFixedSize()` 调用设置 Surface 的大小。比如，你可以配置一个游戏总是以 1280x720 渲染，这将显著地减少填充 2560x1440 的平板或 4K 电视的屏幕必须处理的像素数。显示处理器处理放缩。如果你不想使你的游戏在固定大小的渲染，你可以通过设置大小来调整游戏的长宽比比，使窄尺寸为 720 像素，但长尺寸设置为保持物理显示器的长宽比 (比如，1152x720 匹配一个 2560x1600 的显示器)。使用这种方法的一个例子，请参考 Grafika 的 "Hardware scaler exerciser" activity。

# GLSurfaceView
GLSurfaceView 类提供了一些辅助类来管理 EGL contexts，线程间通信，及与 Activity 生命周期的交互。仅此而已。你不需要通过使用 GLSurfaceView 来使用 GLES。

比如，GLSurfaceView 创建一个线程来渲染并在那儿配置一个 EGL context。当 Activity pause 时，状态被自动地清除。通过 GLSurfaceView，大多数应用在使用 GLES 时将无需知道任何与 EGL 有关的东西。

大多数情况下，GLSurfaceView 是非常有帮助的，且可以简化对 GLES 的使用。在某些情况下，它会阻碍你的发展。如果它有帮助就使用它，否则就不用。

# SurfaceView 和 Activity 生命周期
当使用 SurfaceView 时，在一个主 UI 线程之外的线程中渲染 Surface 被认为是一个良好的实践。这产生了一些关于那个线程和 Activity 生命周期交互的问题。

对于一个使用了 SurfaceView 的 Activity，有两个分开但相互依赖的状态机：

 1. 应用程序的 onCreate/onResume/onPause

 2. Surface 的 created/changed/destroyed

当 Activity 启动时，你以这样的顺序得到回调：

 * onCreate

 * onResume

 * surfaceCreated

 * surfaceChanged

如果你按了返回键你得到：

 * onPause

 * surfaceDestroyed (在 Surface 消失之前被调用)

如果你旋转屏幕，Activity 被销毁并创建，并将经历一个完整的周期。你可以通过检查 `isFinishing()` 来快速重新启动。有可能 start/stop Activity 太快，以至于 `onPause()` 之后 `surfaceCreated()` 实际可能还没有发生。

如果你按下电源键来锁住屏幕，只有 `onPause()` 被调用 - 没有 `surfaceDestroyed()`。Surface 保持活跃，渲染可以继续进行。如果你请求它们的话你甚至可以继续获得 Choreographer 事件。如果你的锁屏强制强制不同的方向，则当设备解锁时，你的
 Activity 可能会重新启动；如果不是，你可以使用与之前的
 Surface 相同的屏幕区域。

当使用一个单独的渲染线程来使用 SurfaceView 时，这引发了一个基本的问题：那个线程的生命周期应该与 Surface 还是 Activity 挂钩？答案依赖于当屏幕空白时你想要让什么发生：(1). 当 Activity start/stop 时 start/stop 该线程，或者当 Surface create/destroy 时 start/stop 该线程。

选项 1 与应用的生命周期交互良好。我们在 `onResume()` 中启动渲染线程并在 `onPause()` 停止它。在创建和配置线程时，它有点尴尬，因为有时 Surface 将已经存在，有时还没有（比如在通过电源键关闭屏幕时它依然存活）。我们在该线程中执行一些初始化之前不得不等待 surface 创建，但我们不能简单地在 `surfaceCreated()` 回调中完成它，因为如果 Surface 没有被重建的话那将不会再次被调用。因此我们需要查询或者缓存 Surface 状态，并把它转发给渲染线程。

**注意：** 在线程之间传递对象时要小心。最好通过一个 Handler 消息来传递 Surface 或 SurfaceHolder（而不是仅仅把它填入线程）以避免多核系统的问题。更多细节，请参考  [Android SMP Primer](http://developer.android.com/training/articles/smp.html)。

选项 2 之所以吸引人，是因为 Surface 和渲染器在逻辑上是交织在一起的。我们在 Surface 创建之后启动线程，这将避免一些线程间通信问题，Surface created/changed 消息仅仅被简单的转发。我们需要确保在屏幕空白时渲染停止并在屏幕恢复时恢复；告诉 Choreographer 停止调用帧绘制调用可能是一个额简单的问题。当且仅当渲染器线程运行时，我们的 `onResume()` 将需要恢复该调用。事情可能没有那么简单 - 如果我们根据帧之间的时间来做动画，当下一个事件到达时我们可能有一个巨大的间隙；显式的暂停/恢复消息可能是可取的。

**注意：** 选项 2 的一个例子可以参考 Grafika 的 "Hardware scaler exerciser"。

两种选项主要都被渲染器线程如何配置，以及是否执行所困绕。一个相关的问题是在 Activity 被杀掉时 (`在onPause()` 或 `onSaveInstanceState()` 中) 从渲染线程提取状态；在这种情况下，选项 1 最好，因为在渲染器线程已经 joined 之后它的状态无需同步原语就可以访问。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-sv-glsv)
