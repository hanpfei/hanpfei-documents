---
title: PyCairo 中的图片
date: 2018-01-12 16:26:49
categories: 图形图像
tags:
- 图形图像
- Python
- 翻译
---

PyCairo 教程的这个部分，我们将讨论图片。我们将演示如何在 GTK 窗口中显示一幅 PNG 或JPEG 图片。我们也将在图片上绘制一些文字。
<!--more-->
# 显示一幅 PNG 图片

在第一个例子中，我们将显示一幅 PNG 图片。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program shows how to draw
an image on a GTK window in PyCairo.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.load_image()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Image")
        self.resize(300, 170)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def load_image(self):
        self.ims = cairo.ImageSurface.create_from_png("stmichaelschurch.png")

    def on_draw(self, wid, cr):
        cr.set_source_surface(self.ims, 10, 10)
        cr.paint()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
这个例子显示一幅图片。

```
        self.ims = cairo.ImageSurface.create_from_png("stmichaelschurch.png")
```
我们由一幅 PNG 图片创建一个图片 surface。

```
        cr.set_source_surface(self.ims, 10, 10)
```
我们将前面创建的图像 surface 设为 source 用于绘制。

```
        cr.paint()
```
我们将 source 绘制在窗口中。

![图：展示一幅图片](https://www.wolfcstech.com/images/1315506-08ee8e2f154a99e8.jpg)

# 显示一幅 JPEG 图片

PyCairo 只内建了对 PNG 图片的支持。其它的图片可以通过 `gtk.gdk.Pixbuf` 对象来显示。它是一个用于管理图像的 GTK 对象。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program shows how to draw
an image on a GTK window in PyCairo.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk, Gdk, GdkPixbuf
import cairo


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.load_image()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Image")
        self.resize(300, 170)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def load_image(self):
        self.pb = GdkPixbuf.Pixbuf.new_from_file("stmichaelschurch.jpg")

    def on_draw(self, wid, cr):
        Gdk.cairo_set_source_pixbuf(cr, self.pb, 5, 5)
        cr.paint()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
在这个例子中，我们在窗口中显示了一幅 JPEG 图片。

```
from gi.repository import Gtk, Gdk, GdkPixbuf
```
除了Gtk，我们还需要 Gdk 和 GdkPixbuf 模块。

```
        self.pb = GdkPixbuf.Pixbuf.new_from_file("stmichaelschurch.jpg")
```
我们由一个 JPEG 文件创建一个 `GdkPixbuf.Pixbuf`。

```
        Gdk.cairo_set_source_pixbuf(cr, self.pb, 5, 5)
        cr.paint()
```
`Gdk.cairo_set_source_pixbuf()` 方法将 pixbuf 设为 source 以用于绘制。

![图：展示一幅图片](https://www.wolfcstech.com/images/1315506-2cb2ada8f6997fc3.jpg)

# 水印

在图片上绘制信息很常见。绘制到图片上的文字称为水印。水印用于标识图片。它们可能是版权信息或图片的创建时间。

```
#!/usr/bin/python

'''
ZetCode PyCairo tutorial

This program draws a watermark
on an image.

author: Jan Bodnar
website: zetcode.com
last edited: August 2012
'''

import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
import cairo


class Example(Gtk.Window):
    def __init__(self):
        super(Example, self).__init__()

        self.init_ui()
        self.load_image()
        self.draw_mark()

    def init_ui(self):
        darea = Gtk.DrawingArea()
        darea.connect("draw", self.on_draw)
        self.add(darea)

        self.set_title("Watermark")
        self.resize(350, 250)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("delete-event", Gtk.main_quit)
        self.show_all()

    def load_image(self):
        self.ims = cairo.ImageSurface.create_from_png("beckov.png")

    def draw_mark(self):
        cr = cairo.Context(self.ims)
        cr.set_font_size(11)
        cr.set_source_rgb(0.9, 0.9, 0.9)
        cr.move_to(20, 30)
        cr.show_text(" Beckov 2012 , (c) Jan Bodnar ")
        cr.stroke()

    def on_draw(self, wid, cr):
        cr.set_source_surface(self.ims, 10, 10)
        cr.paint()


def main():
    app = Example()
    Gtk.main()


if __name__ == "__main__":
    main()
```
我们在一幅图片上绘制版权信息。

```
    def load_image(self):
        self.ims = cairo.ImageSurface.create_from_png("beckov.png")
```
在 `load_image()` 方法中，我们由一幅 PNG 图片创建一个图片 surface。

```
    def draw_mark(self):
        cr = cairo.Context(self.ims)
    . . .
```
在 `draw_mark()` 方法中，我们将版权信息绘制到图片上。首先，我们由图像surface 创建一个绘制上下文。

```
        cr.set_font_size(11)
        cr.set_source_rgb(0.9, 0.9, 0.9)
        cr.move_to(20, 30)
        cr.show_text(" Beckov 2012 , (c) Jan Bodnar ")
        cr.stroke()
```
然后以白色绘制一段小文字。

```
    def on_draw(self, wid, cr):
        cr.set_source_surface(self.ims, 10, 10)
        cr.paint()
```
最后，将图片 surface 绘制到窗口中。

![图：水印](https://www.wolfcstech.com/images/1315506-ba8923d9fbe2ffb2.jpg)

这一章，我们讨论了 PyCairo 中的图片。

[原文](http://zetcode.com/gfx/pycairo/images/)

Done.
