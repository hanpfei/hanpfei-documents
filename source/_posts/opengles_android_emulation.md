---
title: Android 硬件 OpenGL ES 模拟设计概述
date: 2017-09-16 13:05:49
categories: 虚拟化
tags:
- Android开发
- 图形图像
- 翻译
---

简介
-------------

Android 平台的 OpenGL ES 模拟由多个组件实现，它们是：
<!--more-->
  - 一些宿主机的 “翻译器” 库。它们实现了由 Khronos 定义的 EGL，GLES 1.1 和 GLES 2.0 ABIs，并把对应的函数调用翻译为适当的桌面 API，比如：

      - 实现 EGL 接口的是 GLX (Linux)，AGL (OS X) 或 WGL (Windows)
      - 实现 GLES 1.1 和 GLES 2.0 接口的是桌面 GL 2.0
```
         _________            __________          __________
        |         |          |          |        |          |
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____            ____v_____          _____v____      HOST
        |         |          |          |        |          |     SYSTEM
        |   GLX   |          |  GL 2.0  |        |  GL 2.0  |     LIBRARIES
        |_________|          |__________|        |__________|
```


  - 模拟的客户系统内的一些系统库，它们实现了相同的 EGL / GLES 1.1 和 GLES 2.0 ABIs。

    它们收集 EGL/GLES 函数调用序列并把它们翻译为定制的协议流，通过一个称为 "QEMU Pipe" 的高速通信通道发送给模拟器程序。

    目前为止，你需要知道的所有东西即是，Pipe 由一个定制的内核驱动实现，并提供了非常快速的带宽。从客户系统的角度来看，对 Pipe 所做的所有 `read()` 和 `writes()` 基本上是瞬间完成的。

```
         _________            __________          __________
        |         |          |          |        |          |
        |EMULATION|          |EMULATION |        |EMULATION |     GUEST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     SYSTEM
        |_________|          |__________|        |__________|     LIBRARIES
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____________________v____________________v____      GUEST
        |                                                   |     KERNEL
        |                       QEMU PIPE                   |
        |___________________________________________________|
                                  |
       - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - -
                                  |
                                  v
                               EMULATOR
```
  - 模拟器程序内特定的代码，它能够把协议流发送给理解协议格式的特殊的渲染库或进程（这里称为 “渲染器”）。

```
                                 |
                                 |    PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  EMULATOR |
                           |___________|
                                 |
                                 |   UNMODIFIED PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|
```

  - 渲染器从协议流解码 EGL/GLES 命令，并把它们派发给适当的翻译器库。
```
                                 |
                                 |   PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|
                               | |  |
             +-----------------+ |  +-----------------+
             |                   |                    |
         ____v____            ___v______          ____v_____
        |         |          |          |        |          |     HOST
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES
```


  - 事实上，协议流是双向流动的，尽管大多数命令导致数据从客户系统传送到宿主机。模拟的完整图像将是这样的：

```
         _________            __________          __________
        |         |          |          |        |          |
        |EMULATION|          |EMULATION |        |EMULATION |     GUEST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     SYSTEM
        |_________|          |__________|        |__________|     LIBRARIES
             ^                    ^                    ^
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____________________v____________________v____      GUEST
        |                                                   |     KERNEL
        |                       QEMU PIPE                   |
        |___________________________________________________|
                                 ^
                                 |
      - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - -
                                 |
                                 |    PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  EMULATOR |
                           |___________|
                                 ^
                                 |   UNMODIFIED PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|
                               ^ ^  ^
                               | |  |
             +-----------------+ |  +-----------------+
             |                   |                    |
         ____v____            ___v______          ____v_____
        |         |          |          |        |          |
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES
             ^                    ^                    ^
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____            ____v_____          _____v____      HOST 
        |         |          |          |        |          |     SYSTEM
        |   GLX   |          |  GL 2.0  |        |  GL 2.0  |     LIBRARIES
        |_________|          |__________|        |__________|
```

(注意：只有 Linux 是 'GLX'，OS X 是 'AGL'，Windows 是 'WGL')。

注意，在上图中，只有最底下的宿主系统库 ***不是*** Android 提供的。

设计要求
--------------------

上述设计来自项目初期决定的若干重要要求：

1 - 在模拟器之外的单独进程中运行渲染器的能力非常重要。

  由于各种实际的原因，我们计划通过使用两个不同的进程把核心的 QEMU 模拟与 UI 窗口完全分离。这样，渲染器将实现为 UI 程序内的库，但需要从 QEMU 进程接收协议字节。

  它们两个间的通信通道将是一个快速的 Unix socket 或 Win32 命名管道。如果存在性能问题，具有适当同步原语的共享内存片段也可以使用。

  这解释了为什么模拟器不改变，甚至试图解析协议字节流。它只扮演客户系统与渲染器之间的傻瓜代理。这也避免了在 QEMU 代码内增加太多的 GLES 特有代码，而那将复杂地可怕。

2 - 使用生产商专有桌面 EGL/GLES 库的能力非常重要。

  像 NVidia，AMD，或 ARM 这样的 GPU 生产商都提供了 EGL/GLES 库的主机版本，来模拟它们各自的嵌入式图形芯片。

  渲染库可以配置为使用这些库来替代本项目提供的翻译器库。在更精确地模拟特定设备的行为时这可能很有用。

  此外，这些供应商库通常会暴露翻译器库不提供的特定于供应商的扩展。我们无法在不修改我们的代码的情况下暴露它们，但无需太多痛苦就能够做到这一点很重要。

代码组织
------------------

上面提到的组件的源码分布在 Android 源码树的多个目录下：

  - 模拟器的源码位于 `$ANDROID/external/qemu`，本文档的后面部分我们将用 `$QEMU` 指称它。

  - 客户系统库位于 `$ANDROID/device/generic/goldfish/opengl`，我们将称为 `$EMUGL_GUEST`。

  - 宿主机渲染器和翻译器库位于 `$QEMU/android/android-emugl`，我们将称为 `$EMUGL_HOST`。

  - QEMU Pipe 内核驱动位于 `$KERNEL/drivers/misc/qemupipe` (3.4) 或 `$KERNEL/drivers/platform/goldfish/goldfish_pipe.c` (3.10)。

其中 `$ANDROID` 是开源 Android 源码树的根目录，`$KERNEL` 是 qemu 专有内核源码树的根目录 (这里使用 android-goldfish-xxxx 的一个分支)。

与这个项目相关的模拟器源码有：
```
   $QEMU/hw/android/goldfish/pipe.c -> 实现 QEMU pipe 虚拟硬件。
   $QEMU/android/opengles.c         -> 实现 GLES 初始化。
   $QEMU/android/hw-pipe-net.c      -> 实现 QEMU Pipe 和渲染器库之间的通信通道。
```

其它的源码还有：
```
   $EMUGL_GUEST/system   -> 系统库
   $EMUGL_GUEST/shared   -> 共享库的客户系统拷贝
   $EMUGL_GUEST/tests    -> 各种测试程序

   $EMUGL_HOST/host      -> 宿主机库（翻译器 + 渲染器）
   $EMUGL_HOST/shared    -> 共享库的宿主机拷贝
```

共享库实际不是共享的是有历史原因的：某个时刻客户系统和宿主代码位于相同的地方。随着 Android SDK 开出分支，那被证明是不切实际的，且不支持单独的模拟器二进制文件能够运行多个 Android 发行版的要求。


# 翻译器库

本项目提供了三个主机端的翻译器库：
```
   libEGL_translator       -> EGL 1.2 翻译
   libGLES_CM_translator   -> GLES 1.1 翻译
   libGLES_V2_translator   -> GLES 2.0 翻译
```

库的完整的名字依赖于宿主机系统。为了简单起见，只有库名字的后缀会改变（即在 Windows 上没有删除 'lib' 前缀），比如：
```
   libEGL_translator.so    -> for Linux
   libEGL_translator.dylib -> for OS X
   libEGL_translator.dll   -> for Windows
```

这些库的源码位于 Android 源码树的下列路径下：
```
   $EMUGL_HOST/host/libs/Translator/EGL
   $EMUGL_HOST/host/libs/Translator/GLES_CM
   $EMUGL_HOST/host/libs/Translator/GLES_V2
```
翻译器库也使用如下目录中定义的通用的程序：

```
   $EMUGL_HOST/host/libs/Translator/GLcommon
```

协议概述
----------------------

"协议" 按如下所述实现：

  - EGL/GLES 函数调用通过一些 "规范" 文件描述，它们描述了类型，函数签名，以及各种属性。

  - 这些文件由称为 "emugen" 的工具读取，它基于规范产生 C 源文件和头文件。这些文件对应于编码，解码和 "wrappers"（更详细的内容在后面说明）。

  - 系统 “编码器” 静态库使用这些生成的文件中的一些来编译。它们包含了可以把 EGL/GLES 调用序列化为简单的字节消息，并把它通过通用的 "IOStream" 对象发送的代码。

  - 宿主机 “解码器” 静态库也使用这些生成的文件中的一些来编译。它们的代码从 "IOStream" 对象中提取字节消息，并把它们翻译为函数调用。

IOStream 抽象
----------------------

"IOStream" 是一个非常简单的抽象类，用于在客户系统和宿主系统中发送字节消息。它通过 `$EMUGL/host/include/OpenglRender/IOStream.h` 下的一个共享头文件定义

注意，尽管路径在 `$EMUGL/host` 下，但宿主系统和客户系统源码 ***同时*** 包含这个头文件。IOStream 的主要设计思路是，发送一条消息，每一条做如下这些事情：

  1/ 调用 stream->allocBuffer(size)，这将返回一块大小至少为 'size' 个字节的内存缓冲区的地址。

  2/ 直接将序列化的命令（通常是一个头部 + 一些载荷）的内容写入缓冲区。

  3/ 调用 stream->commitBuffer() 发送它。

另外，也可以通过 stream->alloc() 和 stream->flush() 把多个命令打包为一个缓冲区，如：

  1/ buf1 =  stream->alloc(size1)
  2/ 把第一个命令的字节写入 buf1
  3/ buf2 = stream->alloc(size2)
  4/ 把第二个命令的字节写入 buf2
  5/ stream->flush()

最后，有一些显式的 read/write 方法，比如 stream->readFully() 或 stream->writeFully()，当你不想要中间缓冲区时可以被使用它们。在某些情况下实现会使用它们，比如，当从客户系统向宿主机发送纹理数据时，为了避免中间内存复制。

宿主机 IOStream 实现位于 $EMUGL/shared/OpenglCodecCommon/，特别要看：
```
   $EMUGL_HOST/shared/OpenglCodecCommon/TcpStream.cpp
      -> 使用本地 TCP sockets
   $EMUGL_HOST/shared/OpenglCodecCommon/UnixStream.cpp
      -> 使用 Unix sockets
   $EMUGL_HOST/shared/OpenglCodecCommon/Win32PipeStream.cpp
      -> 使用 Win32 命名管道
```

客户系统 IOStream 实现使用上面的 `TcpStream.cpp`，以及可替代的 QEMU 特有的源码：
```
   $EMUGL_GUEST/system/OpenglSystemCommon/QemuPipeStream.cpp
      -> 从客户系统使用 QEMU pipe
```

由于以下几个原因，QEMU Pipe 执行速度（约20倍）显着增加：

  - 从客户系统的角度看，通过它的所有成功的 read() 和 write() 是瞬间完成的。

  - 所有缓冲区/内存复制直接由模拟器执行，这比内核中通过模拟的 ARM 指令做相同的事情要快得多。

  - 无需浏览把数据打包进 TCP/IP/MAC 包，并把它们送给模拟的以太网设备的内核 TCP/IP 栈，它们本身连接到一个内部的防火墙实现，其将解包数据包，重新汇集它们，并通过 BSD socket 把它们发送给宿主机内核。

然而，如果有需要的话，你可以为客户系统编写一个使用不同的传输方式的 IOStream 实现。如果你要那么做，可以参考 `$EMUGL_GUEST/system/OpenglCodecCommon/HostConnection.cpp`，其包含了用于把客户系统连接到宿主机的代码，在每个线程的基础上。


源码的自动生成
----------------------

`emugen` 工具位于 `$EMUGL_HOST/host/tools/emugen`。有一份 README 文件解释了 `emugen` 是如何工作的。

You can also look at the following specifications files:

你也可以看一下如下的规范文件：

GLES 1.1：
```
    $EMUGL_HOST/host/GLESv1_dec/gl.types
    $EMUGL_HOST/host/GLESv1_dec/gl.in
    $EMUGL_HOST/host/GLESv1_dec/gl.attrib
```

GLES 2.0：
```
    $EMUGL_HOST/host/GLESv2_dec/gl2.types
    $EMUGL_HOST/host/GLESv2_dec/gl2.in
    $EMUGL_HOST/host/GLESv2_dec/gl2.attrib
```

EGL：
```
    $EMUGL_HOST/host/renderControl_dec/renderControl.types
    $EMUGL_HOST/host/renderControl_dec/renderControl.in
    $EMUGL_HOST/host/renderControl_dec/renderControl.attrib
```

注意 EGL 规范文件位于名为 `renderControl_dec` 的目录下，且其文件名以 `renderControl` 开头。

这主要是出于历史原因，但也与这样的事实有关，即协议的这个部分包含了一些函数/调用/规范的支持，但它们不是 EGL 规范本身的一部分，但添加了所有功能都需要的一些功能。比如，它们具有与 `gralloc` 系统库模块有关的调用，`gralloc` 系统库模块用于在比 EGL 更低的层面管理图形 Surfaces。

一般来说，客户系统编码器源码位于名为 `$EMUGL_GUEST/system/<name>_enc/` 的目录下，尽管对应的宿主系统解码器的源码位于 `$EMUGL_HOST/host/libs/<name>_dec/` 下。

然而，所有这些源文件使用位于解码目录下的相同的 spec 文件。

编码器文件由位于 `$EMUGL_HOST` 下的 `emugen` 和 spec 文件构建，并被 `gen-encoder.sh` 脚本复制到位于 `$EMUGL_GUEST` 下的编码器目录内。它们被 check in，以使给定的 Android 版本支持特定的协议版本，甚至是更新的渲染器版本（和未来的 Android 版本）支持了一个更新的协议版本。当协议改变时，这一步需要手动地完成；这些改变也需要伴随着渲染器内的改变，以处理老版本的协议。


系统库
-----------------

## 元 EGL/GLES 系统库，和 egl.cfg
- - - - - - - - - - - - - - - - - - - - - -

对有一点的理解很重要，即模拟器特有的 EGL/GLES 库不是在运行时由应用程序直接链接的。相反，系统提供了一系列 “元” EGL/GLES 库，它们将在第一次使用时加载适当的硬件专用库。

进一步来说，系统 `libEGL.so` 包含了一个 “加载器”，其将尝试加载：


  - 硬件专用的 EGL/GLES 库
  - 基于软件的渲染库 (称为 “libagl”)

系统 `libEGL.so` 还能够将硬件和软件库的 EGL 配置透明地合并到应用程序中。系统 `libGLESv1_CM.so` 与 `libGLESv2.so` 与它一起工作，以确保线程的当前上下文根据所选择的配置被链接到硬件或者软件库。

作为记录，加载器的源码位于 `frameworks/base/opengl/libs/EGL/Loader.cpp`。它依赖于名为 `/system/lib/egl/egl.cfg` 的文件，其包含了看起来像下面这样的两行：
```
    0 1 <name>
    0 0 android
```

每一行的第一个数字是显示号，且必须为 0，因为系统的 EGL/GLES 库不支持任何其它的值。

第二个数字必须用 1 表示硬件库，用 0 表示软件库。对应于硬件库的行，如果存在的话，必须总是出现在软件库对应的行的前面。

第三个字段是对应于共享库后缀的名字。它实际意味着对应的库名字为 `libEGL_<name>.so`，`libGLESv1_CM_<name>.so` 和 `libGLESv2_<name>.so`。此外，这些库必须被放在 `/system/lib/egl/` 下。

名字 `android` 为系统的软件渲染器保留。

来自于本项目的 `egl.cfg` 为硬件库使用名字 `emulation`。这意味着它提供一个 `egl.cfg` 文件，其中包含如下的行：
```
   0 1 emulation
   0 0 android
```

参考 `$EMUGL_GUEST/system/egl/egl.cfg` ，及更通常是下面的构建文件：
```
   $EMUGL_GUEST/system/egl/Android.mk
   $EMUGL_GUEST/system/GLESv1/Android.mk
   $EMUGL_GUEST/system/GLESv2/Android.mk
```
来了解库如何命名并被构建系统放置于 /system/lib/egl/ 下。

## 模拟库
- - - - - - - - - - -

模拟器专用的库位于如下位置：
```
  $EMUGL_GUEST/system/egl/
  $EMUGL_GUEST/system/GLESv1/
  $EMUGL_GUEST/system/GLESv2/
```

GLESv1 和 GLESv2 的代码量很小，因为它主要链接到静态编码库。

EGL 的代码有点复杂，因为它需要动态地处理扩展。比如，如果一个扩展在宿主机系统上不可用，则它不应该在运行时被暴露给库。因此 EGL 代码查询宿主机的可用扩展列表以把它们返回给客户端。类似地，它必须为当前的宿主机系统查询有效的 EGLConfigs 的列表。


## "gralloc" 模块实现
- - - - - - - - - - - - - - - - -

除了 EGL/GLES 库之外，Android 系统也需要硬件专用的库来在比 EGL 更低的层次上管理图形 Surfaces。这个库必须是 Android 领域所谓的 “HAL 模块”。

“HAL 模块” 必须提供由 Android 的 HAL（硬件抽象库，Hardware Abstraction Library）定义的接口。这些接口的定义可以在 `$ANDROID/hardware/libhardware/include/` 下找到。

在所有可能的 HAL 模块中，“gralloc” 被系统的 SurfaceFlinger 用于分配 framebuffers 和其它图形内存区域，以及最终在需要的时候 lock/unlock/swap 它们。

位于 `$EMUGL/system/gralloc/` 下的代码实现了GLES 模拟项目所需的该模块。它不是很长，但这里有一些事情需要注意：

- 首先，它将探测客户系统以确定运行虚拟设备的模拟器是否真的支持 GPU 模拟。在某些情况下，这也许是不可能的。

  如果是这种情况，则该模块将会把所有的调用重定向到 “默认的” gralloc 模块，其通常在只启用了软件渲染时由系统使用。

  探测发生在 `fallback_init` 函数中，当模块首次打开时它会被调到。当需要时，它把 `sFallback` 变量初始化为指向默认的 gralloc 模块的指针。

- 第二，这个模块由 SurfaceFlinger 使用，来显示 “软件 Surfaces”，比如，那些由系统内存像素缓冲区支持，并通过 Skia 图形库直接写入的（比如，非硬件加速的那些）。

  默认的模块简单地把像素数据从 Surface 拷贝到虚拟的 framebuffer i/o 内存，但本项目的 gralloc 模块通过 QEMU Pipe 把它发送给渲染器。

  事实证明，这在整体上使得渲染/帧率更快，因为客户系统内的内存拷贝很慢，而 QEMU pipe 传输是在模拟器中直接完成的。

宿主机渲染器
--------------

宿主机渲染器库位于 `$EMUGL_HOST/host/libs/libOpenglRender` 下，它提供了一个由 `$EMUGL_HOST/host/libs/libOpenglRender/render_api.h` 下（比如用于模拟器）的头文件描述的接口。

简而言之，渲染库负责以下内容：

  - 提供一个虚拟的离屏视频 surface 用于在运行时渲染所有的东西。它的维度必须通过在库初始化之后，紧接着调用 `initOpenglRender()` 来固定。

  - 提供一种方式在一个宿主机应用程序的 UI 中显示虚拟的视频 Surface。这通过调用 `createOpenGLSubWindow()` 完成，它接收 window ID 或父 window 的句柄，一些显示维度和一个旋转角度作为参数。这允许 surface 在显示时被放缩/旋转，甚至是在视频 surface 的维度没有改变时。

  - 提供一种方式监听从客户系统进入的 EGL/GLES 命令。这通过给 `initOpenglRender()` 提供一个所谓的 “端口号” 完成。

    默认情况下，端口号对应一个渲染器将绑定和监听的本地 TCP 端口号。到该端口的每个新连接将对应创建一个新的客户系与统宿主系统间的连接，每个这样的连接对应客户系统中的一个不同的线程。

    处于性能原因，监听 Unix sockets（在 Linux 或 OS X 上），或 Win32 命名管道（在 Windows 上）都是可能的。为了做到这一点，必须在库初始化（比如`initLibrary()`）和构建（比如 `initOpenglRender()`）之间调用 `setStreamType()`。

    注意在这些模式中，端口号依然被用于区分多个模拟器实例。这些细节通常由模拟器代码处理，所以你不应该太在意。

注意更早的接口版本允许渲染器库的客户端提供它自己的 `IOStream` 实现。然而，因为许多原因这不是很方便。如果它有意义，这也许可以再次做到，但现在的性能数字是相当好的。

宿主机模拟器
--------------

位于 `$QEMU/android/opengles.c` 下的代码负责动态地加载渲染库并适当地初始化/构造它。

到 `opengles` 服务的 QEMU pipe 连接通过位于 `$QEMU/android/hw-pipe-net.c` 下的代码管道化。找到 `openglesPipe_init()` 函数，它负责创建一个到渲染器库的连接（依赖于具体配置，而通过一个 TCP socket，或一个 Unix 管道。模拟器中对 Win32 命名管道的支持还没有实现），无论何时客户系统进程通过 `/dev/qemu_pipe` 打开了 `opengles` 服务。

还有一些用于支持显示 GLES framebuffer 的代码（通过渲染器库的 `subwindow`）位于 `$QEMU/skin/window`。

请注意，此刻，放缩和旋转是得到支持的。然而，亮度模拟（用于在显示之前修改来自于硬件 framebuffer 的像素值）还不起作用。

另一个问题是，此刻还不能在 GL subwindow 之上显示任何东西。例如，这将掩盖仿真的轨迹球图像（通常在仿真期间通过Ctrl-T切换，或通过按Delete键启用）。

### [打赏](https://www.wolfcstech.com/about/donate.html)

原文  ---- emu-2.4-release/external/qemu/android/android-emugl/DESIGN

Done.
