---
title: OpenCV_005-OpenCV 鼠标作为画笔
date: 2022-04-10 09:35:49
categories: 图形图像
tags:
- 图形图像
- OpenCV
---

本文主要内容来自于 [OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html) 的 [OpenCV 中的 GUI 功能](https://docs.opencv.org/4.5.5/dc/d4d/tutorial_py_table_of_contents_gui.html) 部分，这个部分的主要内容如下：
<!--more-->
*   [图像操作入门](https://docs.opencv.org/4.5.5/db/deb/tutorial_display_image.html)
    学习加载一幅图像，显示它，并保存它
*   [视频入门](https://docs.opencv.org/4.5.5/dd/d43/tutorial_py_video_display.html)
    学习播放视频，从摄像头捕捉视频，以及写入视频
*   [OpenCV 中的绘制功能](https://docs.opencv.org/4.5.5/dc/da5/tutorial_py_drawing_functions.html)
    学习通过 OpenCV 绘制线、矩形、椭圆形和圆形等等
*   [鼠标作为画笔](https://docs.opencv.org/4.5.5/db/d5b/tutorial_py_mouse_handling.html)
    用鼠标画东西
*   [轨迹栏作为调色板](https://docs.opencv.org/4.5.5/d9/dc8/tutorial_py_trackbar.html)
    创建轨迹栏以控制某些参数

## 目标

 * 学习如何在 OpenCV 中处理鼠标事件
 * 我们将学习这些函数：**[cv.setMouseCallback()](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html#ga89e7806b0a616f6f1d502bd8c183ad3e "Sets mouse handler for the specified window. ")**

## 简单的演示程序

这里，我们将创建一个简单的应用程序，当双击鼠标左键时，它在一幅图像上绘制一个圆。

首先我们创建一个鼠标事件回调函数，当鼠标事件发生时它将被执行。鼠标事件可以是任何与鼠标相关的事件，比如左键按下，左键抬起，左键双击等等。它给我们每个鼠标事件的坐标 (x,y)。通过这个事件和位置，我们可以做任何我们想做的。要列出所有可用的事件，可以在 Python 终端中执行如下的代码：
```
events = [i for i in dir(cv) if 'EVENT' in i]
print('\n'.join(events))
```

执行这段代码将得到类似下面这样的输出：
```
EVENT_FLAG_ALTKEY
EVENT_FLAG_CTRLKEY
EVENT_FLAG_LBUTTON
EVENT_FLAG_MBUTTON
EVENT_FLAG_RBUTTON
EVENT_FLAG_SHIFTKEY
EVENT_LBUTTONDBLCLK
EVENT_LBUTTONDOWN
EVENT_LBUTTONUP
EVENT_MBUTTONDBLCLK
EVENT_MBUTTONDOWN
EVENT_MBUTTONUP
EVENT_MOUSEHWHEEL
EVENT_MOUSEMOVE
EVENT_MOUSEWHEEL
EVENT_RBUTTONDBLCLK
EVENT_RBUTTONDOWN
EVENT_RBUTTONUP
```

创建鼠标事件回调函数具有特定的格式，在任何地方都是相同的。它们仅在函数做什么方面不同。即鼠标事件回调函数可以是参数列表满足条件的任何函数。我们的鼠标事件回调函数只做一件事，它在双击发生的位置绘制一个圆。参见下面的代码。代码是自解释的：
```
def draw_circle_follow_mouse():
    img = np.zeros((512, 512, 3), np.uint8)

    # mouse callback function
    def draw_circle(event, x, y, flags, param):
        if event == cv.EVENT_LBUTTONDBLCLK:
            # Create a black image, a window and bind the function to window
            img.fill(0)
            cv.circle(img, (x, y), 100, (255, 0, 0), -1)
            cv.imshow('image', img)

    cv.namedWindow('image')
    cv.imshow('image', img)
    cv.setMouseCallback('image', draw_circle)
    while (1):
        if cv.waitKey(20) & 0xFF == 27:
            break
    cv.destroyAllWindows()
```

这里的鼠标事件回调函数其实是个闭包，它绑定了局部上下文。为了防止两次鼠标双击事件中的绘制相互干扰，每次在鼠标事件回调函数中绘制之前都会先清空图像。ASCII 码 27 表示 ESC 键，即按下 ESC 键是应用程序退出。

## 更高级的演示程序

现在我们继续开发一个更好的应用程序。这次，我们像在 Paint 应用程序中一样，通过拖动鼠标绘制矩形或者圆（依赖我们选择的模式）。因此我们的鼠标事件回调函数有两部分，一部分用于绘制矩形，另一部分用于绘制圆形。这个具体的例子将非常有助于创建和理解一些交互式应用程序，如对象跟踪、图像分割等。
```
def draw_shape_follow_mouse():
    drawing = False  # true if mouse is pressed
    mode = True  # if True, draw rectangle. Press 'm' to toggle to curve
    ix, iy = -1, -1

    img = np.zeros((512, 512, 3), np.uint8)
    # mouse callback function
    def draw_shape(event, x, y, flags, param):
        nonlocal ix, iy, drawing, mode
        if event == cv.EVENT_LBUTTONDOWN:
            drawing = True
            ix, iy = x, y
            img.fill(0)
        elif event == cv.EVENT_MOUSEMOVE:
            if drawing == True:
                if mode == True:
                    cv.rectangle(img, (ix, iy), (x, y), (0, 255, 0), -1)
                else:
                    cv.circle(img, (x, y), 5, (0, 0, 255), -1)
        elif event == cv.EVENT_LBUTTONUP:
            drawing = False
            if mode == True:
                cv.rectangle(img, (ix, iy), (x, y), (0, 255, 0), -1)
            else:
                cv.circle(img, (x, y), 5, (0, 0, 255), -1)

        cv.imshow('image', img)

    cv.namedWindow('image')
    cv.imshow('image', img)
    cv.setMouseCallback('image', draw_shape)
    while (1):
        key = cv.waitKey(20) & 0xFF
        if key == ord('m'):
            mode = not mode
        elif key == 27:
            break
    cv.setMouseCallback(None)
    cv.destroyAllWindows()
```

这里的鼠标事件回调函数同样是闭包，通过 `cv.setMouseCallback()` 将回调函数绑定到 OpenCV 窗口。在主循环中，我们应该为键 'm' 设置一个键盘绑定，以在矩形和圆形之间切换。

**参考文档**

[Mouse as a Paint-Brush](https://docs.opencv.org/4.5.5/db/d5b/tutorial_py_mouse_handling.html)

Done.
