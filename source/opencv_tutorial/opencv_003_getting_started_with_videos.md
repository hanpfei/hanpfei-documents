---
title: OpenCV_003-视频入门
date: 2022-04-09 19:35:49
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

 * 学习读取视频，显示视频，和保存视频
 * 学习从摄像头采集视频并显示它
 * 我们将学习这些函数：**[cv.VideoCapture()](https://docs.opencv.org/4.5.5/d8/dfe/classcv_1_1VideoCapture.html "Class for video capturing from video files, image sequences or cameras. ")**，**[cv.VideoWriter()](https://docs.opencv.org/4.5.5/dd/d9e/classcv_1_1VideoWriter.html "Video writer class. ")**

 ## 从摄像头采集视频

通常，我们必须用摄像头捕捉实时流。OpenCV 提供了一个非常简单的接口来做这些。让我们从摄像头采集一段视频（我使用我笔记本电脑上内置的 webcam），把它转换为灰度视频并显示。只是一个入门的简单任务。

要捕捉视频，我们需要创建一个 **VideoCapture** 对象。它的参数可以是设备索引或视频文件的文件名。设备索引只是一个用于指定使用那个摄像头的数字。通常连接了一个摄像头（就我而言）。因而，我简单地传入 0 (或 -1)。你可以通过传入 1 选择第二个摄像头等等。随后你可以一帧一帧地捕捉图像。最后，不要忘记释放 capture。

```
#!/use/bin/env python
import sys

import numpy as np
import cv2 as cv


def main():
    cap = cv.VideoCapture(0)
    if not cap.isOpened():
        print("Cannot open camera")
        sys.exit()

    while True:
        # Capture frame-by-frame
        ret, frame = cap.read()

        # if frame is read correctly ret is True
        if not ret:
            print("Can't receive frame() (stream end?). Exiting ...")
            break
        # Our operations on the frame come here
        gray = cv.cvtColor(frame, cv.COLOR_BGR2GRAY)
        # Display the resulting frame
        cv.imshow('frame', gray)
        if cv.waitKey(1) == ord('q'):
            break

    cap.release()
    cv.destroyAllWindows()


if __name__ == "__main__":
    main()
```

[`cap.read()`](https://docs.opencv.org/4.5.5/d2/d75/namespacecv.html#a9afba2f5b9bf298c62da8cf66184e41f) 返回一个布尔值 (`True`/`False`)。如果正确地读取了帧，它为 `True`。因此可以通过检查这个返回值来确认视频是否结束。

有时，cap 可能还没有初始化捕捉。在那种情况下，这段代码将显示错误。我们可以通过 **cap.isOpened()** 方法检查它是否初始化。如果它是 `True`，则 OK。否则使用 **[`cap.open()`](https://docs.opencv.org/4.5.5/d6/dee/group__hdf5.html#ga243d7e303690af3c5c3686ca5785205e "Open or create hdf5 file. ")** 打开它。

我们还可以使用 **cap.get(propId)** 方法访问这个视频的一些功能，其中 *propId* 是一个从 0 到 68 的数字。每个数字表示视频的一个属性（如果它适用于该视频的话）。完整的描述可以在这里找到：**[cv::VideoCapture::get()](https://docs.opencv.org/4.5.5/d8/dfe/classcv_1_1VideoCapture.html#aa6480e6972ef4c00d74814ec841a2939 "Returns the specified VideoCapture property. ")**。这些值中的一些可以使用 **cap.set(propId, value)** 来修改。*value* 是想要的新值。

比如，我们可以通过 `cap.get(`[`cv.CAP_PROP_FRAME_WIDTH`](https://docs.opencv.org/4.5.5/d4/d15/group__videoio__flags__base.html#ggaeb8dd9c89c10a5c63c139bf7c4f5704dab26d2ba37086662261148e9fe93eecad "Width of the frames in the video stream. ")`)` 和 `cap.get(`[`cv.CAP_PROP_FRAME_HEIGHT`](https://docs.opencv.org/4.5.5/d4/d15/group__videoio__flags__base.html#ggaeb8dd9c89c10a5c63c139bf7c4f5704dad8b57083fd9bd58e0f94e68a54b42b7e "Height of the frames in the video stream. ")`)` 检查帧的宽度和高度。它默认给出了 640x480。但我们想要把它修改为 320x240。则使用 `ret = cap.set(`[`cv.CAP_PROP_FRAME_WIDTH`](https://docs.opencv.org/4.5.5/d4/d15/group__videoio__flags__base.html#ggaeb8dd9c89c10a5c63c139bf7c4f5704dab26d2ba37086662261148e9fe93eecad "Width of the frames in the video stream. ")`,320)` 和 `ret = cap.set(`[`cv.CAP_PROP_FRAME_HEIGHT`](https://docs.opencv.org/4.5.5/d4/d15/group__videoio__flags__base.html#ggaeb8dd9c89c10a5c63c139bf7c4f5704dad8b57083fd9bd58e0f94e68a54b42b7e "Height of the frames in the video stream. ")`,240)`。如：
```
    width = cap.get(cv.CAP_PROP_FRAME_WIDTH)
    height = cap.get(cv.CAP_PROP_FRAME_WIDTH)
    fps = cap.get(cv.CAP_PROP_FPS)
    print("FPS:{}, width x height: {} x {}".format(fps, width, height))
```

可以得到如下输出：
```
FPS:30.0, width x height: 640.0 x 640.0
```

> 注意：如果遇到了错误，则可以使用任何其它的摄像头应用程序（比如 Linux 下的 Cheese）来确认摄像头是否工作良好。

## 播放视频文件

播放视频文件与从摄像头采集类似，只是把摄像头索引改为视频文件的文件名。同样在显示视频帧的时候，传入适当的时间调用 **[`cv.waitKey()`](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html#ga5628525ad33f52eab17feebcfba38bd7 "Waits for a pressed key. ")**。如果时间值太低，则视频会非常快，如果它太高，则视频会非常慢（好吧，这就是我们可以慢动作显示视频的方式）。一般情况下 25 毫秒就不错。
```
def play_video_file():
    cv.samples.addSamplesDataSearchPath("/media/data/my_multimedia/opencv-4.x/samples/data")
    cap = cv.VideoCapture(cv.samples.findFile("vtest.avi"))
    while cap.isOpened():
        ret, frame = cap.read()
        # if frame is read correctly ret is True
        if not ret:
            print("Can't receive frame (stream end?). Exiting ...")
            break
        gray = cv.cvtColor(frame, cv.COLOR_BGR2GRAY)
        cv.imshow('frame', gray)
        if cv.waitKey(25) == ord('q'):
            break
    cap.release()
    cv.destroyAllWindows()
```

**cap.get(propId)** 可用的 `propId` 由一些枚举值定义，这其中部分适用于摄像头的 `VideoCapture`，部分则适用于视频文件的 `VideoCapture`。对于视频文件，我们可以通过给 **cap.get(propId)** 传入 `cv.CAP_PROP_FRAME_COUNT` 获得视频文件的视频帧总数。

> 注意，请确保安装了适当版本的 ffmpeg 或 gstreamer。有时使用 video capture 比较头疼，这大多数是由于 ffmpeg/gstreamer 的错误安装。

## 保存视频

我们捕捉了视频，并能一帧一帧地处理它，但我们还想保存视频。对于图像，它很简单：使用 [`cv.imwrite()`](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html#gabbc7ef1aa2edfaa87772f1202d67e0ce "Saves an image to a specified file. ") 就好。在这里，需要做更多的工作。

这次我们创建一个 **VideoWriter** 对象。我们应该指定输出文件名（比如：output.avi）。然后我们应该指定 **FourCC** 码（下一段会有详细说明）。应该传入每秒多少帧 (fps) 以及帧大小。最后一个是 **isColor** 标记。如果它是 `True`，则编码器期望是彩色帧，否则，它使用灰度帧。

[FourCC](https://en.wikipedia.org/wiki/FourCC) 是一个用于指定视频编解码器的 4 字节码。在其官网 [fourcc.org](http://www.fourcc.org/codecs.php) 可以找到可用的编解码器的列表。它是平台独立的。以下编解码器对我来说很好用。

 * 在 Fedora 中：DIVX，XVID，MJPG，X264，WMV1，WMV2。(XVID 是更优选的。 MJPG 会产生大尺寸的视频。 X264 提供非常小尺寸的视频)
 * 在 Windows 上：DIVX （更多有待测试和添加）
 * 在 OSX 上：MJPG (.mp4)，DIVX (.avi)，X264 (.mkv)。

以 `cv.VideoWriter_fourcc('M','J','P','G')` 或 `cv.VideoWriter_fourcc(*'MJPG')` 的形式为 MJPG 传 FourCC    
 码。

如下的代码从摄像头捕捉图形，在垂直方向翻转每一帧，并保存视频。
```
def save_video_file():
    cap = cv.VideoCapture(0)
    # Define the codec and create VideoWriter object
    fourcc = cv.VideoWriter_fourcc(*'XVID')
    out = cv.VideoWriter('output.avi', fourcc, 20.0, (640, 480))
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            print("Can't receive frame (stream end?). Exiting ...")
            break
        frame = cv.flip(frame, 0)
        # write the flipped frame
        out.write(frame)
        cv.imshow('frame', frame)
        if cv.waitKey(1) == ord('q'):
            break
    # Release everything if job is finished
    cap.release()
    out.release()
    cv.destroyAllWindows()
```

Done.

参考文档 [Getting Started with Videos](https://docs.opencv.org/4.5.5/dd/d43/tutorial_py_video_display.html)
