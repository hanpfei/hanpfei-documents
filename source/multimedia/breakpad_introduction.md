---
title: Breakpad 入门
date: 2022-06-24 22:23:49
categories: 音视频开发
tags:
- 音视频开发
---

## 简介

Breakpad 是一个库和工具集，它使你可以给用户分发移除了编译器提供的调试信息的应用程序，但以兼容 "minidump" 的文件格式记录崩溃，把它们发送回你的服务器，并从这些 minidumps 生成 C 和 C++ 的栈追踪。Breakpad 也可以在收到请求时写入没有崩溃的程序的 minidumps。
<!--more-->
Breakpad 当前已经用在了 Google Chrome，Firefox，Google Picasa，Camino，Google Earth，及其它许多项目中。

![breakpad.png](images/1315506-d27ccd31f9514283.png)

Breakpad 有三个主要的组件：

 * **client** 是你包含在你的应用程序中的库。它可以写入 minidump 文件，捕获当前线程的状态，以及当前加载的可执行文件和共享库的识别信息。你可以配置客户端在崩溃发生时，或在显式请求时写入 minidump。

 * **symbol dumper** 是一个读取编译器产生的调试信息，并生成 **符号文件** 的程序，以 [Breakpad 自己的格式](https://github.com/hanpfei/breakpad/blob/main/docs/symbol_files.md)。

 * **processor** 是读取 minidump 文件，为 minidump 涉及的可执行文件和共享库版本，查找适当的符号文件，并生成一个人类可读的 C/C++ 栈追踪。

## minidump 文件格式

minidump 文件格式类似于 core 文件，但它是由 Microsoft 为它们的崩溃上报设施开发的。一个 minidump 文件包含：

 * 创建 dump 时进程中加载的可执行文件和共享库的列表。这个列表包含加载的那些文件的特定版本的文件名和标识符。

 * 进程中出现的线程。对于每个线程，minidump 包含处理器寄存器的状态，以及线程的栈内存的内容。这些数据都是未经解释的字节流，Breakpad 客户端通常没有调试信息可用来生成函数名或行号，或者甚至识别栈帧边界。

 * 关于收集 dump 所在的系统的其它信息：处理器和操作系统版本，dump 的原因，等等等。

 Breakpad 在所有平台上都使用 Windows 的 minidump 文件，而不是传统的 core 文件，出于如下一些原因：

 * Core 文件可能非常大，使它们无法通过网络发送到收集器进行处理。Minidumps 更小，因为它们本来就是设计来这样用的。

 * Core 文件格式的文档记录很差。比如 Linux Standards Base 没有描述在 `PT_NOTE` 段中寄存器是如何存储的。

 * 让 Windows 机器生成 core dump 文件比让其它机器写 minidump 文件更难。

 * 只支持一种文件格式简化了 Breakpad 处理器。

## minidump 概述/生命周期

minidump 通过调用 Breakpad 库生成。默认情况下，初始化 Breakpad 安装一个异常/信号处理器，它在发生异常时把 minidump 写入磁盘。在 Windows 上，这通过 `SetUnhandledExceptionFilter()` 完成；在 OS X 上，这通过创建一个线程等待在 Mach 异常端口上来完成；而在 Linux 上，这通过为各种各样的异常，比如 `SIGILL`，`SIGSEGV` 等安装信号处理器来完成。

一旦生成了 minidump，每个平台都有略微不同的方式来上传崩溃转储。在 Windows 和 Linux 上，提供了一个单独的函数库，可以调用它来执行上传。在 OS X 上，会生成一个单独的进程，提示用户授予权限（如果配置为这样做）并发送文件。

## 术语

**进程内 vs 进程外异常处理** - 通常认为在崩溃的进程内写入 minidump 是不安全的 - 关键的进程数据结构可能已经被破坏，或者异常处理器所运行的栈可能已经被覆写，等等。所有 3 个平台都支持所谓的 “进程外” 异常处理。

## 集成概述

### Breakpad 代码概述

所有的客户端代码可以通过访问位于 https://chromium.googlesource.com/breakpad/breakpad 的 Google Project 找到。`src` 目录具有如下的目录结构：

 * **processor** 包含 minidump 处理的代码，它们用在服务端，而不是用在客户端。

 * **client** 包含所有平台的客户端 minidump 生成库

 * **tools** 包含在各个平台上构建各种工具的源码和工程。

（在其他目录中）

 * [Windows Integration Guide](https://github.com/hanpfei/breakpad/blob/main/docs/windows_client_integration.md)

 * [Mac Integration Guide](https://github.com/hanpfei/breakpad/blob/main/docs/mac_breakpad_starter_guide.md)

 * [Linux Integration Guide](https://github.com/hanpfei/breakpad/blob/main/docs/linux_starter_guide.md)

### 构建过程细节（符号生成）

这适用于所有平台。`src/tools/{platform}/dump_syms` 内有一个工具，它可以读取各个平台的调试信息 (比如 OS X/Linux 的 DWARF 和 STABS，和 Windows 的 PDB 文件)，并生成一个 Breakpad 符号文件。这个工具应该针对你 strip 之前的二进制文件运行（在 OS X/Linux 的情况下），且符号文件需要存储在某个 minidump 处理器可以找到的地方。还有另一个工具，`symupload`，如果你已经编写了一个可以接收它们的服务器，则可以用来上传符号文件。

参考文档

[Getting started with breakpad](https://github.com/hanpfei/breakpad/blob/main/docs/getting_started_with_breakpad.md)
