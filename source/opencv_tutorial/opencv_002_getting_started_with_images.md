---
title: OpenCV_002-图像操作入门
date: 2022-04-09 16:35:49
categories: 图形图像
tags:
- 图形图像
- OpenCV
---

遵循软件开发者学习一项新技术的传统流程，先来看一个 "Hello, world" 程序。
<!--more-->
## 目标

在这份教程中，我们将学习如何：

*   从文件中读取一幅图像 (使用 [cv::imread](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html#ga288b8b3da0892bd651fce07b3bbd3a56))
*   在 OpenCV 窗口中显示一幅图像 (使用 [cv::imshow](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html#ga453d42fe4cb60e5723281a89973ee563))
*   向文件中写入一幅图像 (使用 [cv::imwrite](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html#gabbc7ef1aa2edfaa87772f1202d67e0ce))

## C++ 版本

### 源代码

**可下载的代码**：点击 [这里](https://github.com/opencv/opencv/tree/4.x/samples/cpp/tutorial_code/introduction/display_image/display_image.cpp)

**代码一览**：
```
#include <opencv2/core.hpp>
#include <opencv2/imgcodecs.hpp>
#include <opencv2/highgui.hpp>
#include <iostream>
using namespace cv;

int main() {
  samples::addSamplesDataSearchPath("/media/data/my_multimedia/opencv-4.x/samples/data");
  std::string image_path = samples::findFile("starry_night.jpg");
  Mat img = imread(image_path, IMREAD_REDUCED_COLOR_2);
  if (img.empty()) {
    std::cout << "Could not read the image: " << image_path << std::endl;
    return 1;
  }
  imshow("Display window", img);
  int k = waitKey(0); // Wait for a keystroke in the window
  if (k == 's') {
    imwrite("starry_night.png", img);
  }
  return 0;
}
```

### 源码说明

在 OpenCV 4 中我们有多个模块。其中的每个都关注图像处理的一个不同领域或方法。你可能已经从这些教程本身的用户指南的结构看到了这一点。在你使用它们中的任何一个之前，你首先需要包含头文件，其中声明了每个独立的模块的内容。

你几乎总是会使用：

 * [core](https://docs.opencv.org/4.5.5/d0/de1/group__core.html) 部分，因为这里定义了库的基本构建块

 * [imgcodecs](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html) 模块，提供了用于读取和写入的功能

 * [highgui](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html) 模块，因为它包含了在窗口中显示图像的功能

我们还包含了 *iostream* 以方便控制台行输出和输入。

通过声明 `using namespace cv;`，在下面的代码中，库函数可以无需显式声明命名空间即可访问。
```
#include <opencv2/core.hpp>
#include <opencv2/imgcodecs.hpp>
#include <opencv2/highgui.hpp>

#include <iostream>

using namespace cv;
```

现在让我们分析一下主要的代码。第一步，我们从 OpenCV 示例中读取图像 "starry_night.jpg" 文件。为了做到这一点，调用 [cv::imread](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html#ga288b8b3da0892bd651fce07b3bbd3a56) 函数使用由第一个参数指定的文件路径加载图像。第二个参数是可选的，它指定了想要的图像的格式。这个值可以是：

 * ***IMREAD_COLOR*** 以 BGR 8 位格式加载图像。这个是这里使用的**默认值**。
 * ***IMREAD_UNCHANGED*** 按原本的格式加载图像（包括 alpha 通道，如果有的话）
 * ***IMREAD_GRAYSCALE*** 将图像加载为一个灰度图
 * ***IMREAD_ANYDEPTH***，当输入具有对应的深度时返回 16位/32位 图像，否则把它转为 8 位
 * ***IMREAD_ANYCOLOR***，以任何可能的格式读取色彩格式
 * ***IMREAD_LOAD_GDAL***，使用 gdal 驱动加载图像
 * ***IMREAD_REDUCED_GRAYSCALE_2***，总是把图像转为单通道灰度图，且图像大小减为原来的 1/2
 * ***IMREAD_REDUCED_COLOR_2***，总是把图像转为 3 通道 BGR 彩色图像，且图像大小减为原来的 1/2
 * ***IMREAD_REDUCED_GRAYSCALE_4***，总是把图像转为单通道灰度图，且图像大小减为原来的 1/4
 * ***IMREAD_REDUCED_COLOR_4***，总是把图像转为 3 通道 BGR 彩色图像，且图像大小减为原来的 1/4
 * ***IMREAD_REDUCED_GRAYSCALE_8***，总是把图像转为单通道灰度图，且图像大小减为原来的 1/8
 * ***IMREAD_REDUCED_COLOR_8***，总是把图像转为 3 通道 BGR 彩色图像，且图像大小减为原来的 1/8
 * ***IMREAD_IGNORE_ORIENTATION***，不要根据 EXIF 的方向标记旋转图像。

读取之后，图像数据将被保存在 [cv::Mat](https://docs.opencv.org/4.5.5/d3/d63/classcv_1_1Mat.html) 对象中。
```
  samples::addSamplesDataSearchPath("/media/data/my_multimedia/opencv-4.x/samples/data");
  std::string image_path = samples::findFile("starry_night.jpg");
  Mat img = imread(image_path, IMREAD_REDUCED_COLOR_2);
```

OpenCV 的源码仓库中包含了一些测试文件，位于 `opencv/samples/data` 目录下，`samples::findFile()` 默认在当前目录中查找文件。但通过 `samples::addSamplesDataSearchPath()` 函数，可以为示例文件的查找添加搜索目录。对于 Eclipse 来说，默认的程序运行的当前目录就是项目的根目录，但可以通过执行配置中 "Arguments" 标签下的 "Working directory:" 进行配置。

> 注意：OpenCV 提供了对 Windows 位图 (bmp)，可移植图像格式 (pbm，pgm，ppm) 和 Sun raster (sr，ras) 图像格式的支持。借助于插件 （如果是自己编译库的话，需要指定使用它们，尽管如此，在我们提供的包中默认包含它们）你也可以加载像 JPEG (jpeg，jpg，jpe)，JPEG 2000 (jp2 - 在 CMake 中的编码名称为 Jasper)，TIFF 文件 (tiff，tif) 和可移植网络图形 (png) 这样的图像格式。

`imread()` 函数在读取文件并解码之后，还可以对文件做格式转换，具体的转换方式由这个函数的第二个参数指定。

之后，执行了一个检查，检查图像是否被正确加载。
```
  if (img.empty()) {
    std::cout << "Could not read the image: " << image_path << std::endl;
    return 1;
  }
```

然后，使用一个对 [cv::imshow](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html#ga453d42fe4cb60e5723281a89973ee563) 函数的调用显示图像。第一个参数是窗口的标题，第二个参数是将要显示的 [cv::Mat](https://docs.opencv.org/4.5.5/d3/d63/classcv_1_1Mat.html) 对象。

由于我们想要我们的窗口一直显示，直到用户按下了一个键（否则程序将过快结束），我们使用 [cv::waitKey](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html#ga5628525ad33f52eab17feebcfba38bd7) 函数，它只接受一个参数，表示应该等用户输入多长时间（以毫秒为单位）。0 表示一直等待。返回值为按下的键。
```
  imshow("Display window", img);
  int k = waitKey(0); // Wait for a keystroke in the window
```

最后，当按下的键是 "s" 键时，图像被写入文件。为了做到这一点，以文件路径和 [cv::Mat](https://docs.opencv.org/4.5.5/d3/d63/classcv_1_1Mat.html "n-dimensional dense array class ") 对象为参数调用 [cv::imwrite](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html#gabbc7ef1aa2edfaa87772f1202d67e0ce "Saves an image to a specified file. ") 函数。
```
  if (k == 's') {
    imwrite("starry_night.png", img);
  }
```

## Python 版本

### 源代码

**可下载的代码**：点击 [这里](https://github.com/opencv/opencv/tree/4.x/samples/python/tutorial_code/introduction/display_image/display_image.py)

**代码一览**：
```
#!/use/bin/env python

import cv2 as cv
import sys


def main():
    cv.samples.addSamplesDataSearchPath("/media/data/my_multimedia/opencv-4.x/samples/data")
    img = cv.imread(cv.samples.findFile("starry_night.jpg"))
    if img is None:
        sys.exit("Could not read the image.")
    print(img.__class__.__name__)

    cv.imshow("Display window", img)

    k = cv.waitKey(0)
    if k == ord("s"):
        cv.imwrite("starry_night.png", img)


if __name__ == "__main__":
    main()
```

### 源码说明

这里可以看一下 C++ 代码和 Python 代码之间的转换。对于 Python 程序，首先需要导入 OpenCV python 库。执行这个操作的正确方法是，另外为其分配名称 *cv*，以下将使用这个名称来引用 OpenCV python 库。
```
import cv2 as cv
import sys
```

随后，从 OpenCV 示例中读取 "starry_night.jpg" 文件的过程与 C++ 版完全相同：借助于 `cv.samples.addSamplesDataSearchPath()` 和 `cv.samples.findFile()` 查找文件的路径；通过调用 `cv.imread()` 读取文件。各个函数的接口和参数也与 C++ 版完全相同。Python 版的 `cv.imread()` 同样支持 C++ 版的所有加载标记参数。

`cv.imread()` 函数返回一个 *numpy* 的 `ndarray` 对象。
```
    cv.samples.addSamplesDataSearchPath("/media/data/my_multimedia/opencv-4.x/samples/data")
    img = cv.imread(cv.samples.findFile("starry_night.jpg"))
```

OpenCV 支持加载的图像文件格式与 C++ 完全相同。

图像文件加载之后，执行检查，检查图像是否被正确加载。
```
    if img is None:
        sys.exit("Could not read the image.")
    print(img.__class__.__name__)
```

随后通过 `cv.imshow()` 函数在窗口中显示图像，通过 `cv.waitKey(0)` 等待用户按下按键：
```
    cv.imshow("Display window", img)

    k = cv.waitKey(0)
```

这两个接口与 C++ 版完全相同。

最后，如果用户按下 "s" 保存文件（与 C++ 版完全相同）。
```
    if k == ord("s"):
        cv.imwrite("starry_night.png", img)
```

Done.

参考 [Getting Started with Images](https://docs.opencv.org/4.5.5/db/deb/tutorial_display_image.html)
