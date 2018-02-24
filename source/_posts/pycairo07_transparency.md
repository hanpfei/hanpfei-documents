---
title: PyCairo 中的透明度
date: 2018-01-12 15:56:49
categories: 图形图像
tags:
- 图形图像
- Python
- 翻译
---

在 PyCairo 教程的这个部分，我们将讨论透明度。我们将提供一些基本的定义和三个有趣的透明度的例子。

透明度是指透过一种材料能够看到的品质。理解透明度最简单的方法是想象一块玻璃或水。技术上来说，光线可以穿过玻璃，因而我们可以看到玻璃后面的物体。

在计算机图形学中，我们可以用 *alpha 合成* 实现透明度效果。Alpha 合成是一个将一幅图片和背景结合起来创建部分透明的外观的过程。合成过程使用 *alpha 通道*。在用于表达半透明（透明度）的图像文件格式中，alpha 通道是一个 8-bit 的layer。每像素额外的 8 bits 被用作一个 mask 并代表256级的半透明度。
<!--more-->
# 透明的矩形

第一个例子将绘制 10 个具有不同半透明度的矩形。

```
    def on_draw(self, wid, cr):
        for i in range(1, 11):
            cr.set_source_rgba(0, 0, 1, i * 0.1)
            cr.rectangle(50 * i, 20, 40, 40)
            cr.fill()
```
`set_source_rgba()` 方法有一个 alpha 参数来提供透明度。

```
        for i in range(1, 11):
            cr.set_source_rgba(0, 0, 1, i * 0.1)
            cr.rectangle(50 * i, 20, 40, 40)
            cr.fill()
```
这段代码创建了 10 个矩形，它们的 alpha 值分别为 0.1，...，1。

![图：透明的矩形](https://www.wolfcstech.com/images/1315506-f61723c2c3f0ec24.jpg)

# Puff 效果

在下面的例子中，我们创建一个 puff 效果。例子将显示一个增长的居中的文字，并从一些点开始逐渐褪色。这是一个非常常见的效果，我们经常可以在闪光动画中看到它。`paint_with_alpha()` 方法对于创建这个效果非常重要。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program creates a 'puff'
effect.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk, GLib
import cairo


class cv(object):
    SPEED = 14
    TEXT_SIZE_MAX = 20
    ALPHA_DECREASE = 0.01
    SIZE_INCREASE = 0.8


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()

    def init_ui(self):

        self.darea = Gtk.DrawingArea()
        self.darea.connect("draw", self.on_draw)
        self.add(self.darea)

        self.timer = True
        self.alpha = 1.0
        self.size = 1.0

        GLib.timeout_add(cv.SPEED, self.on_timer)

        self.set_title("Puff")
        self.resize(350, 200)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def on_timer(self):

        if not self.timer: return False

        self.darea.queue_draw()
        return True

    def on_draw(self, wid, cr):

        w, h = self.get_size()

        cr.set_source_rgb(0.5, 0, 0)
        cr.paint()

        cr.select_font_face("Courier", cairo.FONT_SLANT_NORMAL,
                            cairo.FONT_WEIGHT_BOLD)

        self.size = self.size + cv.SIZE_INCREASE

        if self.size > cv.TEXT_SIZE_MAX:
            self.alpha = self.alpha - cv.ALPHA_DECREASE

        cr.set_font_size(self.size)
        cr.set_source_rgb(1, 1, 1)

        (x, y, width, height, dx, dy) = cr.text_extents("ZetCode")

        cr.move_to(w / 2 - width / 2, h / 2)
        cr.text_path("ZetCode")
        cr.clip()
        cr.paint_with_alpha(self.alpha)

        if self.alpha <= 0:
            self.timer = False


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
这个例子在窗口中创建一段不断增长并褪色的文本。

```
class cv(object):
    SPEED = 14
    TEXT_SIZE_MAX = 20
    ALPHA_DECREASE = 0.01
    SIZE_INCREASE = 0.8
```
这里我们定义了一些将在例子中使用的常量。

```
        self.alpha = 1.0
        self.size = 1.0
```
这两个变量存储当前的 alpha 值和字体大小。

```
        GLib.timeout_add(cv.SPEED, self.on_timer)
```
每隔 14 ms，`on_timer()` 方法被调用一次。

```
    def on_timer(self):

        if not self.timer: return False

        self.darea.queue_draw()
        return True
```
在 `on_timer()` 方法中，我们通过 `queue_draw()` 方法重绘 DrawingArea widget。

```
    def on_draw(self, wid, cr):

        w, h = self.get_size()

        cr.set_source_rgb(0.5, 0, 0)
        cr.paint()

        cr.select_font_face("Courier", cairo.FONT_SLANT_NORMAL,
                            cairo.FONT_WEIGHT_BOLD)
    . . .
```
在 `on_draw()` 方法中，我们获取窗口的客户区域的宽度和高度。这些值被用于使文字居中。我们将以某种暗红色填充窗口的背景。我们为文字选择一个 Courier 字体。

```
        (x, y, width, height, dx, dy) = cr.text_extents("ZetCode")
```
我们获取文字的一些度量值。我们将只使用文字宽度。

```
        cr.move_to(w / 2 - width / 2, h / 2)
```
我们移动到一个可以使文字在窗口中居中的位置。

```
        cr.text_path("ZetCode")
        cr.clip()
        cr.paint_with_alpha(self.alpha)
```
我们用 `text_path()` 方法获取文字的 path。我们用 `clip()` 方法将绘制限定在当前的path。`paint_with_alpha()` 方法使用一个 alpha 值的 mask，在当前的裁剪区域内，绘制当前的 source。

![图：Puff 效果](https://www.wolfcstech.com/images/1315506-b7498655449364d9.jpg)

# 图像倒影

在下一个例子中，我么将秀出一个倒影图像。这个效果创造了一种图像倒映在水中的感觉。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program creates an image reflection.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo
import sys


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.load_image()
        self.init_vars()

    def init_ui(self):

        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Reflection")
        self.resize(300, 350)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def load_image(self):

        try:
            self.s = cairo.ImageSurface.create_from_png("slanec.png")
        except Exception as e:
            print(e)
            sys.exit(1)

    def init_vars(self):

        self.imageWidth = self.s.get_width()
        self.imageHeight = self.s.get_height()
        self.gap = 40
        self.border = 20

    def on_draw(self, wid, cr):

        w, h = self.get_size()

        lg = cairo.LinearGradient(w / 2, 0, w / 2, h * 3)
        lg.add_color_stop_rgba(0, 0, 0, 0, 1)
        lg.add_color_stop_rgba(h, 0.2, 0.2, 0.2, 1)

        cr.set_source(lg)
        cr.paint()

        cr.set_source_surface(self.s, self.border, self.border)
        cr.paint()

        alpha = 0.7
        step = 1.0 / self.imageHeight

        cr.translate(0, 2 * self.imageHeight + self.gap)
        cr.scale(1, -1)

        i = 0

        while (i < self.imageHeight):
            cr.rectangle(self.border, self.imageHeight - i,
                         self.imageWidth, 1)

            i = i + 1

            cr.save()
            cr.clip()
            cr.set_source_surface(self.s, self.border,
                                  self.border)

            alpha = alpha - step

            cr.paint_with_alpha(alpha)
            cr.restore()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```

在语法方面，需要注意一下 Python 2.7 和 Python 3 在异常处理方面的差异。一个倒映的城堡的废墟就显示在窗口中了。

```
    def load_image(self):

        try:
            self.s = cairo.ImageSurface.create_from_png("slanec.png")
        except Exception as e:
            print(e)
            sys.exit(1)
```
在 `load_image()` 方法中，由一幅 PNG 图片创建一个图像 surface。

```
    def init_vars(self):

        self.imageWidth = self.s.get_width()
        self.imageHeight = self.s.get_height()
        self.gap = 40
        self.border = 20
```
在 `init_vars()` 方法中，我们获取图像的宽度和高度。同时定义两个变量。

```
        lg = cairo.LinearGradient(w / 2, 0, w / 2, h * 3)
        lg.add_color_stop_rgba(0, 0, 0, 0, 1)
        lg.add_color_stop_rgba(h, 0.2, 0.2, 0.2, 1)

        cr.set_source(lg)
        cr.paint()
```
窗口的背景由一个渐变绘制填充。绘制是一个平滑的由黑色到深灰色的混合。

```
        cr.translate(0, 2 * self.imageHeight + self.gap)
        cr.scale(1, -1)
```
这段代码翻转图像，并将它平移到原始图像的下方。平移操作是必须的，因为放缩操作使得图像翻转并将图像向上平移。为了理解发生了什么，可以简单地拍一张照片，将它放在桌子上，然后反转它。

```
        i = 0

        while (i < self.imageHeight):
            cr.rectangle(self.border, self.imageHeight - i,
                         self.imageWidth, 1)

            i = i + 1

            cr.save()
            cr.clip()
            cr.set_source_surface(self.s, self.border,
                                  self.border)

            alpha = alpha - step

            cr.paint_with_alpha(alpha)
            cr.restore()
```
这是最后的部分。我们使第二幅图片变得透明。但透明度不是固定的。这幅图片渐渐地褪色。倒影图像是一行接一行绘制的。`clip()` 方法将绘制限定在高度为 1 的矩形中。`paint_with_alpha()` 在绘制图像 surface 的当前裁剪区域时会将透明度也考虑进来。

![图：倒影图像](https://www.wolfcstech.com/images/1315506-a31ab6a320e508f7.jpg)

# 等待效果 Demo

在这个例子中，我们使用透明效果创建一个等待效果的 demo。我们将绘制 8 跳线，它们逐渐的淡出，以创造一种假象，好像线在移动一般。这种效果经常被用于告知用户，一个耗时比较久的任务正在后台运行。一个例子是 Interne 上的流视频。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program creates a 'waiting' effect.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk, GLib
import cairo
import math


class cv(object):
    trs = (
        (0.0, 0.15, 0.30, 0.5, 0.65, 0.80, 0.9, 1.0),
        (1.0, 0.0, 0.15, 0.30, 0.5, 0.65, 0.8, 0.9),
        (0.9, 1.0, 0.0, 0.15, 0.3, 0.5, 0.65, 0.8),
        (0.8, 0.9, 1.0, 0.0, 0.15, 0.3, 0.5, 0.65),
        (0.65, 0.8, 0.9, 1.0, 0.0, 0.15, 0.3, 0.5),
        (0.5, 0.65, 0.8, 0.9, 1.0, 0.0, 0.15, 0.3),
        (0.3, 0.5, 0.65, 0.8, 0.9, 1.0, 0.0, 0.15),
        (0.15, 0.3, 0.5, 0.65, 0.8, 0.9, 1.0, 0.0,)
    )

    SPEED = 100
    CLIMIT = 1000
    NLINES = 8


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()

    def init_ui(self):

        self.darea = Gtk.DrawingArea()
        self.darea.connect("draw", self.on_draw)
        self.add(self.darea)

        self.count = 0

        GLib.timeout_add(cv.SPEED, self.on_timer)

        self.set_title("Waiting")
        self.resize(250, 150)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def on_timer(self):

        self.count = self.count + 1

        if self.count >= cv.CLIMIT:
            self.count = 0

        self.darea.queue_draw()

        return True

    def on_draw(self, wid, cr):

        cr.set_line_width(3)
        cr.set_line_cap(cairo.LINE_CAP_ROUND)

        w, h = self.get_size()

        cr.translate(w / 2, h / 2)

        for i in range(cv.NLINES):
            cr.set_source_rgba(0, 0, 0, cv.trs[self.count % 8][i])
            cr.move_to(0.0, -10.0)
            cr.line_to(0.0, -40.0)
            cr.rotate(math.pi / 4)
            cr.stroke()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
我们以八个不同的 alpha 值绘制了八条直线。

```
class cv(object):
    trs = (
        (0.0, 0.15, 0.30, 0.5, 0.65, 0.80, 0.9, 1.0),
        (1.0, 0.0, 0.15, 0.30, 0.5, 0.65, 0.8, 0.9),
        (0.9, 1.0, 0.0, 0.15, 0.3, 0.5, 0.65, 0.8),
        (0.8, 0.9, 1.0, 0.0, 0.15, 0.3, 0.5, 0.65),
        (0.65, 0.8, 0.9, 1.0, 0.0, 0.15, 0.3, 0.5),
        (0.5, 0.65, 0.8, 0.9, 1.0, 0.0, 0.15, 0.3),
        (0.3, 0.5, 0.65, 0.8, 0.9, 1.0, 0.0, 0.15),
        (0.15, 0.3, 0.5, 0.65, 0.8, 0.9, 1.0, 0.0,)
    )
. . .
```
这是一个 Demo 中会用到的透明度值的二维元组。有 8 行，每一行表示一种状态。8 条直线中的每一条将连续使用这些值。

```
    SPEED = 100
    CLIMIT = 1000
    NLINES = 8
```
`SPEED` 常量控制动画的速度。`CLIMIT` 是 `self.count` 变量的最大值。达到这个限制之后，变量会被重置为 0。`NLINES` 是这个例子中绘制的直线的条数。

```
        GLib.timeout_add(cv.SPEED, self.on_timer)
```
我们使用定时器函数创建动画。每一个 `cv.SPEED` ms，on_timer() 方法会被调用一次。

```
    def on_timer(self):

        self.count = self.count + 1

        if self.count >= cv.CLIMIT:
            self.count = 0

        self.darea.queue_draw()

        return True
```
在 `on_timer()` 方法中，我们增加 `self.count` 变量。如果变量达到了 `cv.CLIMIT`
 常量，它会被设置为 0。我们避免溢出，并且不使用非常大的数字。

```
    def on_draw(self, wid, cr):

        cr.set_line_width(3)
        cr.set_line_cap(cairo.LINE_CAP_ROUND)
    . . .
```
我们使线更粗一点，以使得它们更容易被看到。我们用圆形的帽子绘制直线。

```
        w, h = self.get_size()

        cr.translate(w / 2, h / 2)
```
我们将我们的绘制定位在窗口的中心。

```
        for i in range(cv.NLINES):
            cr.set_source_rgba(0, 0, 0, cv.trs[self.count % 8][i])
            cr.move_to(0.0, -10.0)
            cr.line_to(0.0, -40.0)
            cr.rotate(math.pi / 4)
            cr.stroke()
```
在 `for` 循环中，我们用不同的透明度值，绘制了八条旋转的直线。线之间由 45 度角隔开。

![图：等待效果 demo](https://www.wolfcstech.com/images/1315506-11696edaebc7c57d.jpg)

本章我们讨论了透明度。

[原文](http://zetcode.com/gfx/pycairo/transparency/)

Done.
