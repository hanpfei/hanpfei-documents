---
title: PyCairo简介
date: 2018-01-12 13:46:49
categories: 图形图像
tags:
- 图形图像
- Python
- 翻译
---

这里是 PyCairo 教程。这份教程将以 Python 语言，教给你 Cairo 2D 库基本的和一些高级的主题。在大多数例子中，我们将使用 Python GTK 作为后端来产生输出。本教程中所用到的图片可以在 [此处](http://zetcode.com/img/gfx/pycairo/images.zip) 下载。
<!--more-->
# 计算机图形学

有两种不同的计算机图形学 —— 向量图形学和光栅图形学。光栅图形学以像素的集合表示图片。向量图形学使用几何元素，如点、直线、曲线或者多边形表示图片。这些元素使用数学方程式创建。

两种计算机图形类型都有优点和缺点。向量图相对于光栅图的优点是：

 * 占用空间小
 * 具有无限放大的能力
 * 移动、缩放、填充或者旋转不会降低图片的质量

# Cairo

Cairo 是一个用于创建 ***2D向量图*** 的库。它是用 C 语言写的。目前已经出现其它计算机语言的绑定了，包括 `Python`，`Perl`，`C++`，`C#`，`Java`。Cairo 是一个多平台库，它工作于 Linux，BSDs 和 OSX 上。

Cairo 支持多种后端。后端是显示所创建的图形的输出设备。

 * X Window System
 * Win32 GDI
 * Mac OS X Quartz
 * PNG
 * PDF
 * PostScript
 * SVG

这意味着，我们可以使用 Cairo 库在 Linux/BSDs，Windows，OSX 的窗口中绘制图形，同时也可以使用这个库创建 PNG 图像，PDF 文件，PostScript 文件和 SVG 文件。

我们可以对比 Cairo 库和 Windows OS 上的 GDI+ 库，以及 Mac OS 上的 Quartz 2D库。Cairo 是一个开源软件库，自 2.8 版起，Cairo 就是 GTK 系统的一部分。

# 定义

这里我们提供一些有用的定义。为了使用 PyCairo 绘制一些东西，我们必须先创建一个 ***绘制上下文***。绘制上下文持有描述如何绘制的所有图形状态参数。这包括线的宽度，颜色，绘制的目的 **Surface** 和许多其它东西的信息。这使得实际的绘图函数可以接收更少的参数从而简化接口。

***path*** 是用于创建基本形状如直线，圆弧和曲线等的点的集合。有两种类型的paths:开放的和闭合的 paths。在闭合 path 中，起点和终点相接。在开放 path 中，起点与终点不相接。在 PyCairo 中，我们以一个空的 path 开始。首先，我们定义一个 path，然后通过 Stroking 和/或填充它们使其可见。每一次调用 `stroke()` 或者 `fill()` 方法之后，path 会被清空。我们不得不定义一个新的 path。如果我们想要在绘制之后保持既有的 path，可以使用 `stroke_preserve()` 和 `fill_preserve()` 方法。path 由 subpaths 组成。

***Source*** 是我们绘制时所用的画笔。我们可以把 source 看作一支笔或者墨水，我们使用它们来画轮廓线或者填充形状。有四种类型的基本 source，颜色(colors)，渐变(gradients)，模式(patterns) 和图像(images)。

***Surface*** 是我们将要绘制的目的地。我们可以使用 PDF 或者 PostScript surfaces 渲染文档，或者可以通过 Xlib 和 Win32 surfaces 直接绘制到平台上。

在 source 被应用于 surface 之前，它会先被过滤。***Mask*** 被用作一个滤镜。它决定什么地方的 source 被应用，什么地方的不应用。Mask 不透明的部分允许复制自source。透明的部分不允许由 source 复制到 surface。

***Pattern*** 代表向一个 surface 绘制时的 source。在 PyCairo 中，pattern 是你可以从中读取，并用作一个绘制操作的 source 或者 mask 之类的东西。Patterns 可能是纯净的，基于 surface 的或者渐变的。

# 来源

为了创建这份教程，我们使用了下列材料。包括 [Apple Cocoa drawing guide](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CocoaDrawingGuide/Introduction/Introduction.html)，[PyCairo reference](http://cairographics.org/documentation/pycairo/2/index.html) 和 [Cairo documentation](http://cairographics.org/documentation/).

[原文](http://zetcode.com/gfx/pycairo/introduction/)

Done.
