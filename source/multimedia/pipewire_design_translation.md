---
title: PipeWire 设计
date: 2025-03-03 21:05:49
categories: 音视频开发
tags:
- 音视频开发
---

本文是对 PipeWire 的设计的简短介绍。

PipeWire 是一个可以运行多媒体节点图的媒体服务器。节点可以运行在服务器进程的内部或者单独的进程中，并与服务器通信。

PipeWire 的设计目标是：

 * 对于原始视频使用 fd 来传，对于音频通过共享的环形缓冲区来传，以实现高效的处理能力。
 * 能够提供/消费/处理来自任何进程的媒体数据。
 * 提供策略以限制对设备和流的访问。
 * 易于扩展。

这是一个最初的目标，但该设计不仅限于原始视频，还应该能够处理压缩视频和其它媒体。

对于图中的节点，PipeWire 使用 [SPA plugin API](https://docs.pipewire.org/page_spa.html)。SPA 为低延迟和对任何媒体格式的高效处理而设计。SPA 也提供了大量标准 C 库中没有的辅助程序。

我们打算构建的一些应用程序如下：

 * V4l2 设备提供者：提供对 v4l2 设备的受控访问，并在多个进程间共享同一个设备。
 * gnome-shell 视频提供者：GNOME Shell 提供一个节点来给出帧缓冲的内容，以用于屏幕共享或屏幕录制。
 * 音频服务器：混音和播放多个音频流。它的设计更像 CRAS (Chromium 音频服务器) 而不是 PulseAudio，而且还可以将处理安排在图中。
 * 像 JACK 一样专业的音频图处理。
 * 媒体播放后端。

## 协议

本地协议和对象模型与 [Wayland](https://wayland.freedesktop.org/) 类似，但具有自定义的消息序列化/反序列化。这是由于消息中的数据结构更复杂，且不容易用 XML 表示。请参考 [Protocol Native](https://docs.pipewire.org/page_module_protocol_native.html) 来了解更多细节。

## 可扩展性

服务器的功能通过模块和扩展来实现和扩展。模块是服务器端的逻辑位，它连接到不同的地方以提供额外的功能。这主要意味着以某种方式控制处理图。有关当前模块的列表，请参阅 [Modules](https://docs.pipewire.org/page_modules.html)。

扩展是客户端版的模块。大多数扩展同时提供客户端和服务器端的 init 函数。通过模块/扩展可以很容易地添加新的接口或新的对象实现。












