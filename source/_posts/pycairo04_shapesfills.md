---
title: PyCairo 中的形状和填充
date: 2018-01-12 15:26:49
categories: 图形图像
tags:
- 图形图像
- Python
- 翻译
---

PyCairo 教程的这个部分，我们创建一些基本的和更高级的形状。我们使用纯色，模式和渐变填充这些形状。渐变将在另一章中讨论。
<!--more-->
# 基本形状

PyCairo 有一些基本的方法可以用来绘制简单的形状。
```
    def on_draw(self, wid, cr):
        cr.set_source_rgb(0.6, 0.6, 0.6)

        cr.rectangle(20, 20, 120, 80)
        cr.rectangle(180, 20, 80, 80)
        cr.fill()

        cr.arc(330, 60, 40, 0, 2 * math.pi)
        cr.fill()

        cr.arc(90, 160, 40, math.pi / 4, math.pi)
        cr.fill()

        cr.translate(220, 180)
        cr.scale(1, 0.7)
        cr.arc(0, 0, 50, 0, 2 * math.pi)
        cr.fill()
```
在这个例子中，我们创建一个矩形，一个方形，一个圆形，一个弧形，和一个椭圆形。

```
        cr.rectangle(20, 20, 120, 80)
        cr.rectangle(180, 20, 80, 80)
```
`rectangle()` 方法用于创建方形和矩形。方形仅是特殊类型的矩形。参数是左上角在窗口中的 x 和 y 坐标及矩形的宽度和高度。

```
        cr.arc(330, 60, 40, 0, 2 * math.pi)
```
`arc()` 方法创建一个圆形。参数是弧心的 x 和 y 坐标，半径，起始和结束角度，以弧度数表示。

```
        cr.arc(90, 160, 40, math.pi / 4, math.pi)
```
这里我们创建一个弧形，圆形的一部分。

```
        cr.scale(1, 0.7)
        cr.arc(0, 0, 50, 0, 2 * math.pi)
```
我们使用 `scale()` 和 `arc()` 方法创建椭圆形。

![](https://www.wolfcstech.com/images/1315506-c32cd1f4133e865e.jpg)
图：基本形状

其它形状可以通过组合基本元素创建。
```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This code example draws another
three shapes in PyCairo.

Author: Jan Bodnar
Website: zetcode.com
Last edited: April 2016
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo


class cv(object):
    points = (
        (0, 85),
        (75, 75),
        (100, 10),
        (125, 75),
        (200, 85),
        (150, 125),
        (160, 190),
        (100, 150),
        (40, 190),
        (50, 125),
        (0, 85)
    )


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Complex shapes")
        self.resize(460, 240)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def on_draw(self, wid, cr):
        cr.set_source_rgb(0.6, 0.6, 0.6)
        cr.set_line_width(1)

        for i in range(10):
            cr.line_to(cv.points[i][0], cv.points[i][1])

        cr.fill()

        cr.move_to(240, 40)
        cr.line_to(240, 160)
        cr.line_to(350, 160)
        cr.fill()

        cr.move_to(380, 40)
        cr.line_to(380, 160)
        cr.line_to(450, 160)
        cr.curve_to(440, 155, 380, 145, 380, 40)
        cr.fill()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
在这个例子中，我们创建了一个星星对象，一个三角形和一个变形的三角形。这些对象使用一些直线和一条曲线创建。

```
        for i in range(10):
            cr.line_to(cv.points[i][0], cv.points[i][1])

        cr.fill()
```
星星通过连接点元组中的所有的点来绘制。`fill()` 方法使用当前颜色填充星星对象。

```
        cr.move_to(240, 40)
        cr.line_to(240, 160)
        cr.line_to(350, 160)
        cr.fill()
```
这些直线创建一个三角形。最后两个点自动连接。

```
        cr.move_to(380, 40)
        cr.line_to(380, 160)
        cr.line_to(450, 160)
        cr.curve_to(440, 155, 380, 145, 380, 40)
        cr.fill()
```
变形的三角形是两条直线和一条曲线的简单组合。

![](https://www.wolfcstech.com/images/1315506-7bc788ee74435d2e.jpg)
图：复杂形状

# 填充

填充填充形状的内部。填充可以是纯色，模式或渐变。

## 纯色

颜色是表示红，绿和蓝 (RGB) 亮度值的组合的对象。PyCairo 有效的 RGB 值在 0 到 1 范围内。

```
    def on_draw(self, wid, cr):

        cr.set_source_rgb(0.2, 0.23, 0.9)
        cr.rectangle(10, 15, 90, 60)
        cr.fill()

        cr.set_source_rgb(0.9, 0.1, 0.1)
        cr.rectangle(130, 15, 90, 60)
        cr.fill()

        cr.set_source_rgb(0.4, 0.9, 0.4)
        cr.rectangle(250, 15, 90, 60)
        cr.fill()
```
在这个例子中，我们画了三个彩色矩形。

```
        cr.set_source_rgb(0.2, 0.23, 0.9)
        cr.rectangle(10, 15, 90, 60)
        cr.fill()
```
`set_source_rgb()` 方法设置 source 为一个不透明的颜色。参数为红，绿，蓝亮度值。通过调用 `fill()` 方法，source 被用于填充矩形内部。

![](https://www.wolfcstech.com/images/1315506-972744c99b520206.jpg)
图：纯色

## 模式

模式是可用于填充形状的复杂图形对象。
```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program shows how to work
with patterns in PyCairo.

Author: Jan Bodnar
Website: zetcode.com
Last edited: April 2016
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.create_surpat()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Patterns")
        self.resize(300, 290)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def create_surpat(self):
        sr1 = cairo.ImageSurface.create_from_png("blueweb.png")
        sr2 = cairo.ImageSurface.create_from_png("maple.png")
        sr3 = cairo.ImageSurface.create_from_png("crack.png")
        sr4 = cairo.ImageSurface.create_from_png("chocolate.png")

        self.pt1 = cairo.SurfacePattern(sr1)
        self.pt1.set_extend(cairo.EXTEND_REPEAT)
        self.pt2 = cairo.SurfacePattern(sr2)
        self.pt2.set_extend(cairo.EXTEND_REPEAT)
        self.pt3 = cairo.SurfacePattern(sr3)
        self.pt3.set_extend(cairo.EXTEND_REPEAT)
        self.pt4 = cairo.SurfacePattern(sr4)
        self.pt4.set_extend(cairo.EXTEND_REPEAT)

    def on_draw(self, wid, cr):
        cr.set_source(self.pt1)
        cr.rectangle(20, 20, 100, 100)
        cr.fill()

        cr.set_source(self.pt2)
        cr.rectangle(150, 20, 100, 100)
        cr.fill()

        cr.set_source(self.pt3)
        cr.rectangle(20, 140, 100, 100)
        cr.fill()

        cr.set_source(self.pt4)
        cr.rectangle(150, 140, 100, 100)
        cr.fill()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
在这个例子中，我们绘制了四个矩形。这次我们用一些模式填充它们。我们使用了来自 *Gimp* 图像管理程序的四个模式图像。我们必须保持那些模式的原始大小，因为我们将展开他们。

我们在 `draw()` 方法外面创建图像 surface。每次窗口需要重绘时，都从硬盘读取不是很高效。

```
        sr1 = cairo.ImageSurface.create_from_png("blueweb.png")
```
图像 surface 用一幅 PNG 图像创建。

```
        self.pt1 = cairo.SurfacePattern(sr1)
        self.pt1.set_extend(cairo.EXTEND_REPEAT)
```
模式由 surface 创建。我们把模式设置为 `cairo.EXTEND_REPEAT`，这将使得模式以重复的方式展开。

```
        cr.set_source(self.pt1)
        cr.rectangle(20, 20, 100, 100)
        cr.fill()
```
此处我们绘制我们的第一个矩形。`set_source()` 方法告诉Cairo 上下文，在绘制的时候使用模式作为 source。图像模式可能不完全适合形状。`rectangle()` 创建一个矩形 path。最后，`fill()` 方法用 source 填充 path。

上面那段代码中所用到的四幅图片。

![blueweb.png](https://www.wolfcstech.com/images/1315506-6a7dfe69e4ce980a.png)

![chocolate.png](https://www.wolfcstech.com/images/1315506-cf0e6641975441ed.png)

![crack.png ](https://www.wolfcstech.com/images/1315506-def86e085c0716de.png)

![maple.png](https://www.wolfcstech.com/images/1315506-a8fe5c43f9169e8d.png)

代码的执行效果如下：

![205908_1Ir8_919237.jpg](https://www.wolfcstech.com/images/1315506-3765ad8d07ec3ed1.jpg)

本章讨论了 PyCairo 的形状和填充。

[原文](http://zetcode.com/gfx/pycairo/shapesfills/)

Done.

