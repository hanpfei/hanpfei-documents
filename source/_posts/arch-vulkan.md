---
title: Vulkan
date: 2017-07-21 17:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
- 翻译
---

Android 7.0 添加了对 [Vulkan](https://www.khronos.org/vulkan/) 的支持，一个高性能 3D 图形的低开销跨平台 API。像 OpenGL ES 一样，Vulkan 提供了在应用中创建高质量，实时图形的工具。Vulkan 的优势包括 CPU 开销降低及支持 [SPIR-V Binary Intermediate](https://www.khronos.org/spir) 语言。
<!--more-->
片上系统生产商（SoCs）比如 GPU 独立硬件供应商（IHVs）可以为 Android 编写 Vulkan 驱动；OEMs 简单地需要为特定的硬件集成这些驱动。关于 Vulkan 驱动如何与系统交互，GPU 特有工具应该如何安装，以及 Android 特有的要求的细节，请参考 [实现 Vulkan](https://source.android.com/devices/graphics/implement-vulkan.html)。

应用程序开发人员可以利用 Vulkan 来创建在 GPU 上执行命令并大大减少开销的应用程序。Vulkan 还提供了一个更直观的到当前图形硬件中发现的功能的映射，最大限度地减少驱动程序错误的可能性，并减少开发人员的测试时间（例如更少的时间来排除 Vulkan
 错误）。

关于 Vulkan 的一般信息，请参考 [Vulkan 概述](http://khr.io/vulkanlaunchoverview) 或查看下面的 [资源](https://source.android.com/devices/graphics/arch-vulkan#resources) 列表。

# Vulkan 组件
Vulkan 支持包含如下组件：

![](https://www.wolfcstech.com/images/1315506-d5dc09fd4a23f2cb.png)

图 1：Vulkan 组件

 * **Vulkan 验证层** (*在 Android NDK 中提供*)。开发者在开发 Vulkan 应用期间使用的一系列库。来自于图形供应商的 Vulkan 运行时库和 Vulkan 驱动不包含保持 Vulkan 运行时有效的运行时错误检查。相反，验证库用于 (只在开发期间) 查找应用中使用 Vulkan API 时的错误。Vulkan 验证库在开发期间被链接进应用并执行这种错误检查。所有的 API 用法错误被找到之后，应用不再需要包含这些库了。

 * **Vulkan 运行时** (*由 Android 提供*)。一个本地库 ( `libvulkan.so` ) ，它提供了称为
 [Vulkan](https://www.khronos.org/vulkan) 的新的公共本地层 API。大多数功能由 GPU 供应商提供的驱动实现；运行时封装了驱动，提供 API 拦截功能（用于调试及其它开发者工具），并管理驱动和依赖平台的组件如 BufferQueue 之间的交互。

 * **Vulkan 驱动** (*由 SoC 提供*)。将 Vulkan API 映射为硬件特有的 GPU 命令，并与内核层的图形驱动交互。

# 修改的组件
Android 7.0 修改了下列已有的图形组件来支持 Vulkan：

 * **BufferQueue**。Vulkan 运行时通过现有的 `ANativeWindow` 接口与现有的 BufferQueue 组件交互。包括对 `ANativeWindow` 和 BufferQueue 最小的改动（新的枚举值和新的方法），但没有架构级的改动。

 * **Gralloc HAL**。包含一个新的，可选的接口来发现一个给定的格式是否可被用于特定的生产者/消费者结合而无需实际的分配缓冲区。

关于这些组件的更详细信息，请参考 [BufferQueue 和 gralloc](https://source.android.com/devices/graphics/arch-bq-gralloc.html) (关于 ANativeWindow 的细节，请参考 [EGLSurface 和 OpenGL ES](https://source.android.com/devices/graphics/arch-egl-opengl.html))。

# Vulkan API
Android 平台包含一个来自于 Khronos Group 的 [Vulkan API 规范](https://www.khronos.org/vulkan/) 的 [Android 特定实现](https://developer.android.com/ndk/guides/graphics/index.html) 。Android 应用必须使用 [Window System Integration (WSI) 扩展](https://source.android.com/devices/graphics/implement-vulkan.html#wsi) 输出它们的渲染。

# 资源
使用如下的资源来学习更多关于 Vulkan 的东西：

 * [Vulkan Loader ](https://android.googlesource.com/platform/frameworks/native/+/master/vulkan/#)(libvulkan.so) 位于 `platform/frameworks/native/vulkan`。包含 Android 的 Vulkan 加载器，以及一些对平台开发者非常有用的 Vulkan 有关的工具。

 * [Vulkan 实现者指南](https://android.googlesource.com/platform/frameworks/native/+/master/vulkan/doc/implementors_guide/implementors_guide.html)。旨在帮助 GPU IHV 为 Android 编写 Vulkan 驱动程序及 OEM 为特定设备集成那些驱动程序。它描述了 Vulkan 驱动如何与系统交互，特定于 GPU 的工具应该如何安装，以及 Android 特有的要求。

 * [Vulkan 图形 API 指南](https://developer.android.com/ndk/guides/graphics/index.html)。包含关于在应用中使用 Vulkan 的入门的信息，关于Android 平台上 Vulkan 设计指南的详情，如何使用 Vulkan 的 shader 编译器，以及如何使用验证层来帮助确保使用 Vulkan 的应用的稳定性。

 * [Vulkan 新闻](https://www.khronos.org/#slider_vulkan)。包含事件，补丁，指南，和更多与 Vulkan 有关的新闻文章。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://source.android.com/devices/graphics/arch-vulkan)
