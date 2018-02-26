---
title: PyCairo 中的变换
date: 2018-01-12 16:06:49
categories: 图形图像
tags:
- 图形图像
- Python
- 翻译
---

在 PyCairo 图形学编程教程的这个部分，我们将讨论变换。

一个 *仿射变换* 由 0 个或多个线性变换（旋转，放缩或切变）和平移（移位）组成。多个线性变换可以结合为以单个矩阵表示。 *旋转* 是将一个刚体围绕一个固定点移动的变换。*放缩* 是放大或缩小对象的变换。放缩系数在所有方向上都是相同的。*平移* 是在特定的方向上，将每个点都移动固定距离的变换。*切变* 是将物体垂直于给定轴移动，同时保持轴的一侧的值比另一侧的值大的变换。

是一个将一个对象正交的移动向给定的轴，同时保持轴某一侧的值比另一侧更大的变换。

来源: （wikipedia.org，freedictionary.com）
<!--more-->
# 平移

下面的例子描述了一个简单的平移。

```
    def on_draw(self, wid, cr):

        cr.set_source_rgb(0.2, 0.3, 0.8)
        cr.rectangle(10, 10, 30, 30)
        cr.fill()

        cr.translate(20, 20)
        cr.set_source_rgb(0.8, 0.3, 0.2)
        cr.rectangle(0, 0, 30, 30)
        cr.fill()

        cr.translate(30, 30)
        cr.set_source_rgb(0.8, 0.8, 0.2)
        cr.rectangle(0, 0, 30, 30)
        cr.fill()

        cr.translate(40, 40)
        cr.set_source_rgb(0.3, 0.8, 0.8)
        cr.rectangle(0, 0, 30, 30)
        cr.fill()
```
这个例子先绘制了一个矩形。然后我们做平移，并多次绘制相同的矩形。

```
        cr.translate(20, 20)
```
`translate()` 函数通过平移用户空间原点修改当前的变换矩阵。在我们的例子中，我们在两个方向上将原点移动 20 个单位。

![图：平移操作](https://www.wolfcstech.com/images/1315506-531c01053b5b104d.jpg)

# 切变

在下面的例子中，我们执行一个切变操作。切变是沿着一个特定的轴的物体变形。这个操作没有切变方法。我们需要创建我们自己的变换矩阵。注意，每个仿射变换都可以通过创建变换矩阵来执行。

```
    def on_draw(self, wid, cr):

        cr.set_source_rgb(0.6, 0.6, 0.6)
        cr.rectangle(20, 30, 80, 50)
        cr.fill()

        mtx = cairo.Matrix(1.0, 0.5,
                           0.0, 1.0,
                           0.0, 0.0)

        cr.transform(mtx)
        cr.rectangle(130, 30, 80, 50)
        cr.fill()
```
在这段代码中，我们执行一个简单的切变操作。

```
        mtx = cairo.Matrix(1.0, 0.5,
                           0.0, 1.0,
                           0.0, 0.0)
```
这个变换将 y 值切变为 x 值的 0.5 倍。

```
        cr.transform(mtx)
```
我们通过 `transform()` 方法执行变换。

![图：且变操作](https://www.wolfcstech.com/images/1315506-ae71866cb5012ade.jpg)

# 放缩

下一个例子演示放缩操作。放缩是将对象放大或缩小的变换操作。

```
    def on_draw(self, wid, cr):

        cr.set_source_rgb(0.2, 0.3, 0.8)
        cr.rectangle(10, 10, 90, 90)
        cr.fill()

        cr.scale(0.6, 0.6)
        cr.set_source_rgb(0.8, 0.3, 0.2)
        cr.rectangle(30, 30, 90, 90)
        cr.fill()

        cr.scale(0.8, 0.8)
        cr.set_source_rgb(0.8, 0.8, 0.2)
        cr.rectangle(50, 50, 90, 90)
        cr.fill()
```
我们绘制三个 90 × 90 px 大小的矩形。对于其中的 2 个，我们执行放缩操作。

```
        cr.scale(0.6, 0.6)
        cr.set_source_rgb(0.8, 0.3, 0.2)
        cr.rectangle(30, 30, 90, 90)
        cr.fill()
```
我们均匀地用系数 0.6 缩小矩形。

```
        cr.scale(0.8, 0.8)
        cr.set_source_rgb(0.8, 0.8, 0.2)
        cr.rectangle(50, 50, 90, 90)
        cr.fill()
```
这里我们用系数 0.8 执行另一个放缩操作。如果我们查看图片，我们将看到，第三个黄色矩形是最小的。即使我们已经使用了另一个更小的放缩因子。这是由于变换操作是累积的。事实上，第三个矩形是由放缩因子 0.528 (0.6 ×0.8) 来放缩的。

![图：放缩操作](https://www.wolfcstech.com/images/1315506-0c93d58c8b908065.jpg)

# 隔离变换

变换操作是累积的。为了隔离各个操作，我们可以使用 `save()` 和 `restore()` 方法。`save()` 方法创建一份绘制上下文当前状态的拷贝，并将它存进一个保存状态的内部栈中。`restore()` 方法将重建上下文为保存的状态。

```
    def on_draw(self, wid, cr):

        cr.set_source_rgb(0.2, 0.3, 0.8)
        cr.rectangle(10, 10, 90, 90)
        cr.fill()

        cr.save()
        cr.scale(0.6, 0.6)
        cr.set_source_rgb(0.8, 0.3, 0.2)
        cr.rectangle(30, 30, 90, 90)
        cr.fill()
        cr.restore()

        cr.save()
        cr.scale(0.8, 0.8)
        cr.set_source_rgb(0.8, 0.8, 0.2)
        cr.rectangle(50, 50, 90, 90)
        cr.fill()
        cr.restore()
```
这个例子中我们放缩两个矩形。这次我们隔离各个放缩操作。

```
        cr.save()
        cr.scale(0.6, 0.6)
        cr.set_source_rgb(0.8, 0.3, 0.2)
        cr.rectangle(30, 30, 90, 90)
        cr.fill()
        cr.restore()
```
我们通过将 `scale()` 方法放到 `save()` 和 `restore()` 方法之间来隔离放缩操作。

![图：隔离变换](https://www.wolfcstech.com/images/1315506-4d23a35dda0bad94.jpg)

现在第三个黄色矩形比第二个红色的大。

# 甜甜圈

在下面的例子中，我们通过旋转一束椭圆创建一个复杂的形状。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program creates a 'donut' shape
in PyCairo.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo
import math


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Donut")
        self.resize(350, 250)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def on_draw(self, wid, cr):
        cr.set_line_width(0.5)

        w, h = self.get_size()

        cr.translate(w / 2, h / 2)
        cr.arc(0, 0, 120, 0, 2 * math.pi)
        cr.stroke()

        for i in range(36):
            cr.save()
            cr.rotate(i * math.pi / 36)
            cr.scale(0.3, 1)
            cr.arc(0, 0, 120, 0, 2 * math.pi)
            cr.restore()
            cr.stroke()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
我们将执行旋转和放缩操作。我们也将保存和恢复 PyCairo 上下文。

```
        cr.translate(w / 2, h / 2)
        cr.arc(0, 0, 120, 0, 2 * math.pi)
        cr.stroke()
```
在 GTK 窗口中间，我们创建一个圆形。这将是我们的椭圆形的边界圆形。

```
        for i in range(36):
            cr.save()
            cr.rotate(i * math.pi / 36)
            cr.scale(0.3, 1)
            cr.arc(0, 0, 120, 0, 2 * math.pi)
            cr.restore()
            cr.stroke()
```
我们沿着我们的边界圆形路径创建 36 个椭圆形。我们通过`save()` 和 `restore()` 方法，将各个旋转和放缩操作隔离开。

![图：甜甜圈](https://www.wolfcstech.com/images/1315506-c35cbdde9fd281df.jpg)

# 星形

下一个例子展示一个旋转和放缩的星形。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This is a star example which
demonstrates scaling, translating and
rotating operations in PyCairo.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk, GLib
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

    SPEED = 20
    TIMER_ID = 1


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.init_vars()

    def init_ui(self):

        self.darea = Gtk.DrawingArea()
        self.darea.connect("draw", self.on_draw)
        self.add(self.darea)

        self.set_title("Star")
        self.resize(400, 300)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def init_vars(self):

        self.angle = 0
        self.scale = 1
        self.delta = 0.01

        GLib.timeout_add(cv.SPEED, self.on_timer)

    def on_timer(self):

        if self.scale < 0.01:
            self.delta = -self.delta

        elif self.scale > 0.99:
            self.delta = -self.delta

        self.scale += self.delta
        self.angle += 0.01

        self.darea.queue_draw()

        return True

    def on_draw(self, wid, cr):

        w, h = self.get_size()

        cr.set_source_rgb(0, 0.44, 0.7)
        cr.set_line_width(1)

        cr.translate(w / 2, h / 2)
        cr.rotate(self.angle)
        cr.scale(self.scale, self.scale)

        for i in range(10):
            cr.line_to(cv.points[i][0], cv.points[i][1])

        cr.fill()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
在这个例子中，我们创建一个星形对象。我们将平移它，旋转它并放缩它。

```
class cv(object):
    points = (
        (0, 85),
        (75, 75),
        (100, 10),
        (125, 75),
        (200, 85),
. . .
```

星形对象将从这些点创建。

```
    def init_vars(self):

        self.angle = 0
        self.scale = 1
        self.delta = 0.01
    . . .
```
在 `init_vars()` 方法中，我们初始化三个变量。`self.angle` 被用于旋转，`self.scale` 被用于放缩星形对象。`self.delta` 变量控制何时星形不断放大而何时又不断缩小。

```
        GLib.timeout_add(cv.SPEED, self.on_timer)
```
每隔 `cv.SPEED` ms，`on_timer()` 方法会被调用。

```
        if self.scale < 0.01:
            self.delta = -self.delta

        elif self.scale > 0.99:
            self.delta = -self.delta
```
这几行控制星形是逐渐变大还是变小。

```
        cr.translate(w / 2, h / 2)
        cr.rotate(self.angle)
        cr.scale(self.scale, self.scale)
```
我们将星形移动到窗口的中心。旋转并放缩它。

```
        for i in range(10):
            cr.line_to(cv.points[i][0], cv.points[i][1])

        cr.fill()
```
我们在这里绘制星形对象。

![图：星形](https://www.wolfcstech.com/images/1315506-8b6725f02eb3c00c.jpg)

在 PyCairo 教程的这个部分，我们讨论了变换。

[原文](http://zetcode.com/gfx/pycairo/transformations/)

Done.
