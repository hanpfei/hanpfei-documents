---
title: OpenCV 官方文档的组织结构
date: 2022-04-09 09:35:49
categories: 图形图像
tags:
- 图形图像
- OpenCV
---

OpenCV (开源计算机视觉库：[http://opencv.org](http://opencv.org/)) 是一个开源库，它包含了几百个计算机视觉算法。学习 OpenCV 库最权威的资料无疑就是 OpenCV 的官方文档了。
<!--more-->
OpenCV 官方提供的文档比较齐全，这些文档主要有两种形式，一是教程，就像书或文章一样，会以 OpenCV 的某个模块或接口为主题，较为详细地说明基本原理，OpenCV 的 API 用法，并提供示例代码和说明；二是 API 参考，会逐个类逐个函数接口的进行说明。要学习 OpenCV，教程形式的官方文档无疑是最好的选择；但日常开发中，一时想不起来某个函数接口的签名及语义，想不起来某个类有哪些函数接口，则翻找 API 参考形式的官方文档更好。

这里梳理一下 OpenCV 官方文档的组织结构，让我们可以对 OpenCV 提供了什么有一个大概的了解，同时又能为我们队 OpenCV 的学习提供一个路线图。

在 OpenCV 官方网站中，[库的 Releases 页面](https://opencv.org/releases/) 列出了 OpenCV 各个版本的一些关键信息，如源码的下载链接，对应版本的 GitHub 链接，不同操作系统平台的二进制安装包下载链接等，当然也包括文档入口的链接。

这里以当前最新的发行版 4.5.5 为例，看一下OpenCV 官方文档的组织结构。

OpenCV 4.5.5 版文档的入口页面位于 https://docs.opencv.org/4.5.5/，这个页面的组织如下：

*   [介绍](https://docs.opencv.org/4.5.5/d1/dfb/intro.html)
*   [OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html)
*   [OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html)
*   [OpenCV.js 教程](https://docs.opencv.org/4.5.5/d5/d10/tutorial_js_root.html)
*   [contrib 模块教程](https://docs.opencv.org/4.5.5/d3/d81/tutorial_contrib_root.html)
*   [经常问的问题 (FAQ)](https://docs.opencv.org/4.5.5/d3/d2d/faq.html)
*   [参考文献](https://docs.opencv.org/4.5.5/d0/de3/citelist.html)
*   主要模块：
    *   core. [核心功能](https://docs.opencv.org/4.5.5/d0/de1/group__core.html)
    *   imgproc. [图像处理](https://docs.opencv.org/4.5.5/d7/dbd/group__imgproc.html)
    *   imgcodecs. [图像文件读取和写入](https://docs.opencv.org/4.5.5/d4/da8/group__imgcodecs.html)
    *   videoio. [视频 I/O](https://docs.opencv.org/4.5.5/dd/de7/group__videoio.html)
    *   highgui. [高级 GUI](https://docs.opencv.org/4.5.5/d7/dfc/group__highgui.html)
    *   video. [视频分析](https://docs.opencv.org/4.5.5/d7/de9/group__video.html)
    *   calib3d. [相机校准和 3D 重建](https://docs.opencv.org/4.5.5/d9/d0c/group__calib3d.html)
    *   features2d. [2D 特征框架](https://docs.opencv.org/4.5.5/da/d9b/group__features2d.html)
    *   objdetect. [目标检测](https://docs.opencv.org/4.5.5/d5/d54/group__objdetect.html)
    *   dnn. [深度神经网络模块](https://docs.opencv.org/4.5.5/d6/d0f/group__dnn.html)
    *   ml. [机器学习](https://docs.opencv.org/4.5.5/dd/ded/group__ml.html)
    *   flann. [多维空间中的聚类和搜索](https://docs.opencv.org/4.5.5/dc/de5/group__flann.html)
    *   photo. [计算摄影](https://docs.opencv.org/4.5.5/d1/d0d/group__photo.html)
    *   stitching. [图像拼接](https://docs.opencv.org/4.5.5/d1/d46/group__stitching.html)
    *   gapi. [图形 API](https://docs.opencv.org/4.5.5/d0/d1e/gapi.html)
*   额外模块：
    *   alphamat. [Alpha 遮罩](https://docs.opencv.org/4.5.5/d4/d40/group__alphamat.html)
    *   aruco. [ArUco标记检测](https://docs.opencv.org/4.5.5/d9/d6a/group__aruco.html)
    *   barcode. [条码检测和解码方法](https://docs.opencv.org/4.5.5/d2/dea/group__barcode.html)
    *   bgsegm. [改进的背景-前景分割方法](https://docs.opencv.org/4.5.5/d2/d55/group__bgsegm.html)
    *   bioinspired. [受生物启发的视觉模型和衍生工具](https://docs.opencv.org/4.5.5/dd/deb/group__bioinspired.html)
    *   ccalib. [用于 3D 重建的自定义校准模式](https://docs.opencv.org/4.5.5/d3/ddc/group__ccalib.html)
    *   cudaarithm. [矩阵运算](https://docs.opencv.org/4.5.5/d5/d8e/group__cudaarithm.html)
    *   cudabgsegm. [背景分割](https://docs.opencv.org/4.5.5/d6/d17/group__cudabgsegm.html)
    *   cudacodec. [视频编码/解码](https://docs.opencv.org/4.5.5/d0/d61/group__cudacodec.html)
    *   cudafeatures2d. [特征检测和描述](https://docs.opencv.org/4.5.5/d6/d1d/group__cudafeatures2d.html)
    *   cudafilters. [图像过滤](https://docs.opencv.org/4.5.5/dc/d66/group__cudafilters.html)
    *   cudaimgproc. [图像处理](https://docs.opencv.org/4.5.5/d0/d05/group__cudaimgproc.html)
    *   cudalegacy. [传统支持](https://docs.opencv.org/4.5.5/d5/dc3/group__cudalegacy.html)
    *   cudaobjdetect. [目标检测](https://docs.opencv.org/4.5.5/d9/d3f/group__cudaobjdetect.html)
    *   cudaoptflow. [光流](https://docs.opencv.org/4.5.5/d7/d3f/group__cudaoptflow.html)
    *   cudastereo. [立体声对应](https://docs.opencv.org/4.5.5/dd/d47/group__cudastereo.html)
    *   cudawarping. [图像变形](https://docs.opencv.org/4.5.5/db/d29/group__cudawarping.html)
    *   cudev. [设备层](https://docs.opencv.org/4.5.5/df/dfc/group__cudev.html)
    *   cvv. [用于计算机视觉程序交互式可视化调试的 GUI](https://docs.opencv.org/4.5.5/df/dff/group__cvv.html)
    *   datasets. [处理不同数据集的框架](https://docs.opencv.org/4.5.5/d8/d00/group__datasets.html)
    *   dnn_objdetect. [用于目标检测的 DNN](https://docs.opencv.org/4.5.5/d5/df6/group__dnn__objdetect.html)
    *   dnn_superres. [用于超分辨率的 DNN](https://docs.opencv.org/4.5.5/d9/de0/group__dnn__superres.html)
    *   dpm. [基于可变形零件的模型](https://docs.opencv.org/4.5.5/d9/d12/group__dpm.html)
    *   face. [人脸分析](https://docs.opencv.org/4.5.5/db/d7c/group__face.html)
    *   freetype. [使用 freetype/harfbuzz 绘制 UTF-8 字符串](https://docs.opencv.org/4.5.5/d4/dfc/group__freetype.html)
    *   fuzzy. [基于模糊数学的图像处理](https://docs.opencv.org/4.5.5/df/d5b/group__fuzzy.html)
    *   hdf. [分层数据格式 I/O 例程](https://docs.opencv.org/4.5.5/db/d77/group__hdf.html)
    *   hfs. [用于高效图像分割的分层特征选择](https://docs.opencv.org/4.5.5/dc/d29/group__hfs.html)
    *   img_hash. [该模块带来了不同图像散列算法的实现。](https://docs.opencv.org/4.5.5/d4/d93/group__img__hash.html)
    *   intensity_transform. [该模块实现了强度变换算法来调整图像对比度。](https://docs.opencv.org/4.5.5/dc/dfe/group__intensity__transform.html)
    *   julia. [OpenCV 的 Julia 绑定](https://docs.opencv.org/4.5.5/d7/d44/group__julia.html)
    *   line_descriptor. [从图像中提取的线的二进制描述符](https://docs.opencv.org/4.5.5/dc/ddd/group__line__descriptor.html)
    *   mcc. [麦克白图表模块](https://docs.opencv.org/4.5.5/dd/d19/group__mcc.html)
    *   optflow. [光流算法](https://docs.opencv.org/4.5.5/d2/d84/group__optflow.html)
    *   ovis. [OGRE 3D 可视化器](https://docs.opencv.org/4.5.5/d2/d17/group__ovis.html)
    *   phase_unwrapping. [相位展开 API](https://docs.opencv.org/4.5.5/df/d3a/group__phase__unwrapping.html)
    *   plot. [Mat 数据的绘图函数](https://docs.opencv.org/4.5.5/db/dfe/group__plot.html)
    *   quality. [图像质量分析 (IQA) API](https://docs.opencv.org/4.5.5/dc/d20/group__quality.html)
    *   rapid. [基于轮廓的 3D 目标跟踪](https://docs.opencv.org/4.5.5/d4/dc4/group__rapid.html)
    *   reg. [图像配准](https://docs.opencv.org/4.5.5/db/d61/group__reg.html)
    *   rgbd. [RGB深度处理]. some other he(https://docs.opencv.org/4.5.5/d2/d3a/group__rgbd.html)
    *   saliency. [显着性 API](https://docs.opencv.org/4.5.5/d8/d65/group__saliency.html)
    *   sfm. [运动结构](https://docs.opencv.org/4.5.5/d8/d8c/group__sfm.html)
    *   shape. [形状距离和匹配](https://docs.opencv.org/4.5.5/d1/d85/group__shape.html)
    *   stereo. [立体声匹配算法](https://docs.opencv.org/4.5.5/dd/d86/group__stereo.html)
    *   structured_light. [结构光 API](https://docs.opencv.org/4.5.5/d1/d90/group__structured__light.html)
    *   superres. [超分辨率](https://docs.opencv.org/4.5.5/d7/d0a/group__superres.html)
    *   surface_matching. [表面匹配](https://docs.opencv.org/4.5.5/d9/d25/group__surface__matching.html)
    *   text. [场景文本检测与识别](https://docs.opencv.org/4.5.5/d4/d61/group__text.html)
    *   tracking. [追踪 API](https://docs.opencv.org/4.5.5/d9/df8/group__tracking.html)
    *   videostab. [视频稳定](https://docs.opencv.org/4.5.5/d5/d50/group__videostab.html)
    *   viz. [3D 可视化器](https://docs.opencv.org/4.5.5/d1/d19/group__viz.html)
    *   wechat_qrcode. [微信二维码检测器，用于检测和解析二维码。](https://docs.opencv.org/4.5.5/dd/d63/group__wechat__qrcode.html)
    *   xfeatures2d. [额外的 2D 特征框架](https://docs.opencv.org/4.5.5/d1/db4/group__xfeatures2d.html)
    *   ximgproc. [扩展图像处理](https://docs.opencv.org/4.5.5/df/d2d/group__ximgproc.html)
    *   xobjdetect. [扩展目标检测](https://docs.opencv.org/4.5.5/d4/d54/group__xobjdetect.html)
    *   xphoto. [其它  照片处理算法](https://docs.opencv.org/4.5.5/de/daa/group__xphoto.html)

OpenCV 官方文档的组成包括一份介绍文档（[介绍](https://docs.opencv.org/4.5.5/d1/dfb/intro.html)），4 个教程（[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html)、[OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html)、[OpenCV.js 教程](https://docs.opencv.org/4.5.5/d5/d10/tutorial_js_root.html) 和 [contrib 模块教程](https://docs.opencv.org/4.5.5/d3/d81/tutorial_contrib_root.html)），一份 FAQ 文档（[经常问的问题 (FAQ)](https://docs.opencv.org/4.5.5/d3/d2d/faq.html)），一份参考资料文档（[参考文献](https://docs.opencv.org/4.5.5/d0/de3/citelist.html)），以及主要模块和额外模块的 API 参考。

主要模块和额外模块的 API 参考，适合用来在开发过程中检索各个模块的 API，对于学习，这些文档会显得非常枯燥且让人迷惑。

参考资料文档（[参考文献](https://docs.opencv.org/4.5.5/d0/de3/citelist.html)）包含了许多参考文献的说明，其中一部分有网页链接。这份文档作为查找扩展阅读材料的入口有一定价值。

介绍文档（[介绍](https://docs.opencv.org/4.5.5/d1/dfb/intro.html)）包含对 OpenCV 整个项目的结构，以及 API 概念的说明，**学习 OpenCV 必读材料**。

[contrib 模块教程](https://docs.opencv.org/4.5.5/d3/d81/tutorial_contrib_root.html)）的质量有点参差不齐，其中不同模块的教程，其繁简程度也大为不同，有些非常简略，有些则很详细。对于要学习还没有进入正式版的 contrib 模块的同学，还是非常有价值的。

OpenCV 官方提供的三份教程是针对不同平台学习 OpenCV 的绝佳材料。[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 对 OpenCV 的主要数据结构，主要概念，及各个模块的 API 都有着很详细的说明，示例代码主要用 C++ 编程语言。

[OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html) 为 Python 开发者提供了关于 OpenCV 的详尽介绍。

[OpenCV.js 教程](https://docs.opencv.org/4.5.5/d5/d10/tutorial_js_root.html)  则为 JavaScript 开发者提供了关于 OpenCV 的详尽介绍。

### *OpenCV 教程* 的内容

OpenCV 官方提供的三份教程中，[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 值得好好学习一下，另两份教程则可以根据自己所用的开发语言选择是深入学习，还是弃之不用。

这里再看下 [OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的主要内容：

*   [OpenCV 介绍](https://docs.opencv.org/4.5.5/df/d65/tutorial_table_of_content_introduction.html) - 在你的计算机上构建和安装 OpenCV
*   [核心功能 (core 模块)](https://docs.opencv.org/4.5.5/de/d7a/tutorial_table_of_content_core.html) - 这个库的基本构造块
*   [图像处理 (imgproc 模块)](https://docs.opencv.org/4.5.5/d7/da8/tutorial_table_of_content_imgproc.html) - 图像处理功能
*   [应用程序实用工具 (highgui，imgcodecs，videoio 模块)](https://docs.opencv.org/4.5.5/de/d3d/tutorial_table_of_content_app.html) - 应用程序实用工具 (GUI，图像/视频 输入/输出)
*   [相机校准和 3D 重建 (calib3d 模块)](https://docs.opencv.org/4.5.5/d6/d55/tutorial_table_of_content_calib3d.html) - 从 2D 图像中提取 3D 世界信息
*   [2D 特征框架 (feature2d 模块)](https://docs.opencv.org/4.5.5/d9/d97/tutorial_table_of_content_features2d.html) - 特征探测器，描述符和匹配框架
*   [深度神经网络 (dnn 模块)](https://docs.opencv.org/4.5.5/d2/d58/tutorial_table_of_content_dnn.html) - 使用内置 *dnn* 模块推断神经网络
*   [图 API (gapi 模块)](https://docs.opencv.org/4.5.5/df/d7e/tutorial_table_of_content_gapi.html) - 基于图的计算机视觉算法构建方法
*   [其它教程 (ml，objdetect，photo，stitching，video)](https://docs.opencv.org/4.5.5/d3/dd5/tutorial_table_of_content_other.html) - 其它模块 (ml，objdetect，photo，stitching，video)
*   [OpenCV iOS](https://docs.opencv.org/4.5.5/d3/dc9/tutorial_table_of_content_ios.html) - 在一个 iDevice 上运行 OpenCV
*   [GPU 加速的计算机视觉 (cuda 模块)](https://docs.opencv.org/4.5.5/da/d2c/tutorial_table_of_content_gpu.html) - 利用显卡的能力运行 CV 算法

[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的 [OpenCV 介绍](https://docs.opencv.org/4.5.5/df/d65/tutorial_table_of_content_introduction.html) 部分，非常详细地介绍了为各种各样的平台搭建 OpenCV 开发环境的过程。[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的其余部分，则分模块介绍 OpenCV 的各项功能。

[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的 [OpenCV 介绍](https://docs.opencv.org/4.5.5/df/d65/tutorial_table_of_content_introduction.html) 部分的内容如下：

*   [OpenCV 安装概述](https://docs.opencv.org/4.5.5/d0/d3d/tutorial_general_install.html)
*   [OpenCV 配置选项参考](https://docs.opencv.org/4.5.5/db/d05/tutorial_config_reference.html)

**Linux**

*   [在 Linux 中安装 OpenCV](https://docs.opencv.org/4.5.5/d7/d9f/tutorial_linux_install.html)
*   [将 OpenCV 与 gdb 驱动的 IDE 结合使用](https://docs.opencv.org/4.5.5/d6/d25/tutorial_linux_gdb_pretty_printer.html)
*   [将 OpenCV 与 gcc 和 CMake 一起使用](https://docs.opencv.org/4.5.5/db/df5/tutorial_linux_gcc_cmake.html)
*   [将 OpenCV 与 Eclipse (插件 CDT) 一起使用](https://docs.opencv.org/4.5.5/d7/d16/tutorial_linux_eclipse.html)

**Windows**

*   [在 Windows 中安装 OpenCV](https://docs.opencv.org/4.5.5/d3/d52/tutorial_windows_install.html)
*   [如何在 “Microsoft Visual Studio” 中使用 OpenCV 构建应用程序](https://docs.opencv.org/4.5.5/dd/d6e/tutorial_windows_visual_studio_opencv.html)
*   [Image Watch：在 Visual Studio 调试器中查看内存中的图像](https://docs.opencv.org/4.5.5/d4/d14/tutorial_windows_visual_studio_image_watch.html)

**Java & Android**

*   [Java 开发简介](https://docs.opencv.org/4.5.5/d9/d52/tutorial_java_dev_intro.html)
*   [在 Eclipse 中使用 OpenCV Java](https://docs.opencv.org/4.5.5/d1/d0a/tutorial_java_eclipse.html)
*   [使用 Clojure 开发 OpenCV 应用简介](https://docs.opencv.org/4.5.5/d7/d1e/tutorial_clojure_dev_intro.html)
*   [Android 开发简介](https://docs.opencv.org/4.5.5/d9/d3f/tutorial_android_dev_intro.html)
*   [OpenCV4Android SDK](https://docs.opencv.org/4.5.5/da/d2a/tutorial_O4A_SDK.html)
*   [使用 OpenCV 进行 Android 开发](https://docs.opencv.org/4.5.5/d5/df8/tutorial_dev_with_OCV_on_Android.html)
*   [在基于 Android 相机预览的 CV 应用中使用 OpenCL](https://docs.opencv.org/4.5.5/d7/dbd/tutorial_android_ocl_intro.html)

**其它平台**

*   [在 MacOS 中安装 OpenCV](https://docs.opencv.org/4.5.5/d0/db2/tutorial_macos_install.html)
*   [为基于 ARM 的 Linux 系统交叉编译 OpenCV](https://docs.opencv.org/4.5.5/d0/d76/tutorial_arm_crosscompile_with_cmake.html)
*   [使用 CUDA 为 Tegra 构建 OpenCV](https://docs.opencv.org/4.5.5/d6/d15/tutorial_building_tegra_cuda.html)
*   [在 iOS 中安装 OpenCV](https://docs.opencv.org/4.5.5/d5/da3/tutorial_ios_install.html)

**基础用法**

*   [图像入门](https://docs.opencv.org/4.5.5/db/deb/tutorial_display_image.html) - 我们将学习如何从文件中加载图像，并使用 OpenCV 显示它

**杂项**

*   [为 OpenCV 编写文档](https://docs.opencv.org/4.5.5/d4/db1/tutorial_documentation.html) - 这份教程描述了新文档的写作过程，及一些有用的 Doxygen 特性。
*   [过渡指南 guide](https://docs.opencv.org/4.5.5/db/dfa/tutorial_transition_guide.html) - 这份文档描述了 2.4 -> 3.0 的转换过程的一些方面。
*   [从其它的 Doxygen 工程交叉引用 OpenCV](https://docs.opencv.org/4.5.5/d3/dff/tutorial_cross_referencing.html) - 本文档概述了如何从其他 Doxygen 项目创建对 OpenCV 文档的交叉引用。

[OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的 [OpenCV 介绍](https://docs.opencv.org/4.5.5/df/d65/tutorial_table_of_content_introduction.html) 部分对于开发环境搭建过程的详细说明，无疑是我们学习 OpenCV 过程中绕不过去的第一步的宝贵参考。

### *OpenCV-Python 教程* 的内容

当前 Python 在 AI 和计算机视觉领域应用广泛，[OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html) 对于 Python 计算机视觉学习和开发的同学极具价值。这里看一下这份教程的内容：

*   [OpenCV 介绍](https://docs.opencv.org/4.5.5/da/df6/tutorial_py_table_of_contents_setup.html)

    学习如何在计算机上搭建 OpenCV-Python 开发环境！

*   [OpenCV 中的 Gui 功能](https://docs.opencv.org/4.5.5/dc/d4d/tutorial_py_table_of_contents_gui.html)

    在这里，我们将学习如何显示和保存图像和视频、控制鼠标事件和创建轨迹栏。的第一步的宝贵

*   [核心操作](https://docs.opencv.org/4.5.5/d7/d16/tutorial_py_table_of_contents_core.html)

    在本节中，我们将学习图像的基本操作，如像素编辑、几何变换、代码优化、一些数学工具等。的第一步的宝贵

*   [OpenCV 中的图像处理](https://docs.opencv.org/4.5.5/d2/d96/tutorial_py_table_of_contents_imgproc.html)

    在这一节，我们将学习 OpenCV 中不同的图像处理功能。

*   [特征探测和描述](https://docs.opencv.org/4.5.5/db/d27/tutorial_py_table_of_contents_feature2d.html)

    在本节中，我们将了解特征探测器和描述符

*   [视频分析 (video 模块)](https://docs.opencv.org/4.5.5/da/dd0/tutorial_table_of_content_video.html)

    在本节中，我们将学习使用对象跟踪等视频的不同技术。

*   [相机校准和 3D 重建](https://docs.opencv.org/4.5.5/d9/db7/tutorial_py_table_of_contents_calib3d.html)

    在本节中，我们将学习相机校准、立体成像等。

*   [机器学习](https://docs.opencv.org/4.5.5/d6/de2/tutorial_py_table_of_contents_ml.html)

    在本节中，我们将学习 OpenCV 中不同的机器学习功能。

*   [计算摄影](https://docs.opencv.org/4.5.5/d0/d07/tutorial_py_table_of_contents_photo.html)

    在本节中，我们将学习不同的计算摄影技术，例如图像去噪等。

*   [目标探测 (objdetect 模块)](https://docs.opencv.org/4.5.5/d2/d64/tutorial_table_of_content_objdetect.html)

    在本节中，我们将学习目标检测技术，例如人脸检测等。

*   [OpenCV-Python 绑定](https://docs.opencv.org/4.5.5/df/da2/tutorial_py_table_of_contents_bindings.html)

    在本节中，我们将看一下 OpenCV-Python 绑定是如何生成的。

这份教程的结构与 [OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 的结构类似，其中的 [OpenCV 介绍](https://docs.opencv.org/4.5.5/da/df6/tutorial_py_table_of_contents_setup.html) 部分同样介绍了开发环境搭建，其余各部分介绍 OpenCV 的各个模块。

[OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html) 的 [OpenCV 介绍](https://docs.opencv.org/4.5.5/da/df6/tutorial_py_table_of_contents_setup.html) 部分内容如下：

*   [OpenCV-Python 教程介绍](https://docs.opencv.org/4.5.5/d0/de3/tutorial_py_intro.html)

    OpenCV-Python 入门

*   [在 Windows 中安装 OpenCV-Python](https://docs.opencv.org/4.5.5/d5/de5/tutorial_py_setup_in_windows.html)

    在 Windows 中搭建 OpenCV-Python 开发环境

*   [在 Fedora 中安装 OpenCV-Python](https://docs.opencv.org/4.5.5/dd/dd5/tutorial_py_setup_in_fedora.html)

    在 Fedora 中搭建 OpenCV-Python 开发环境

*   [在 Ubuntu 中安装 OpenCV-Python](https://docs.opencv.org/4.5.5/d2/de6/tutorial_py_setup_in_ubuntu.html)

    在 Ubuntu 中搭建 OpenCV-Python 开发环境

### 

### 基于官方文档的 OpenCV 学习路线图

官方文档中的 API 参考部分并没有太大的学习价值，可以基于官方提供的几份教程，具体来说，是 [OpenCV 教程](https://docs.opencv.org/4.5.5/d9/df8/tutorial_root.html) 和 [OpenCV-Python 教程](https://docs.opencv.org/4.5.5/d6/d00/tutorial_py_root.html)，建立自己的 C++ 和 Python 编程语言的 OpenCV 学习路线图。这两份教程介绍的主题差别不是很大，但不同主题在各份教程中的编排顺序略有不同。

这里以 *开发环境搭建* -> *OpenCV 的结构和基本概念* -> *OpenCV 库提供的开发实用工具* -> *OpenCV 核心操作和功能* -> *图像处理* -> *其它 OpenCV 主要模块* -> *OpenCV contrib 模块* 这样的思路来制定学习路线图：

1. 搭建 Python 和 C++ 的开发环境

通过学习教程的 *OpenCV 介绍* 部分可以完成。

[OpenCV-Python Tutorials - Install OpenCV-Python in Ubuntu](https://docs.opencv.org/4.5.5/d2/de6/tutorial_py_setup_in_ubuntu.html)
[OpenCV Tutorials - Installation in Linux](https://docs.opencv.org/4.5.5/d7/d9f/tutorial_linux_install.html)
[OpenCV Tutorials - Using OpenCV with Eclipse (plugin CDT)](https://docs.opencv.org/4.5.5/d7/d16/tutorial_linux_eclipse.html)
[OpenCV Tutorials - Using OpenCV with gcc and CMake](https://docs.opencv.org/4.5.5/db/df5/tutorial_linux_gcc_cmake.html)

2. OpenCV 项目的结构和 API 概念

OpenCV 4.5.5 版文档的入口页面中的 [介绍](https://docs.opencv.org/4.5.5/d1/dfb/intro.html) 部分。

3. 应用程序实用工具

[应用程序实用工具 (highgui，imgcodecs，videoio 模块)](https://docs.opencv.org/4.5.5/de/d3d/tutorial_table_of_content_app.html)  （OpenCV 教程）
[OpenCV 中的 Gui 功能](https://docs.opencv.org/4.5.5/dc/d4d/tutorial_py_table_of_contents_gui.html) （OpenCV Python 教程）

4. 核心功能和操作

[核心功能 (core 模块)](https://docs.opencv.org/4.5.5/de/d7a/tutorial_table_of_content_core.html) （OpenCV 教程）
[核心操作](https://docs.opencv.org/4.5.5/d7/d16/tutorial_py_table_of_contents_core.html) （OpenCV Python 教程）

5. 图形处理

[图像处理 (imgproc 模块)](https://docs.opencv.org/4.5.5/d7/da8/tutorial_table_of_content_imgproc.html) （OpenCV 教程）
[OpenCV 中的图像处理](https://docs.opencv.org/4.5.5/d2/d96/tutorial_py_table_of_contents_imgproc.html) （OpenCV Python 教程）

6. 相机校准和 3D 重建

[相机校准和 3D 重建 (calib3d 模块)](https://docs.opencv.org/4.5.5/d6/d55/tutorial_table_of_content_calib3d.html) （OpenCV 教程）
[相机校准和 3D 重建](https://docs.opencv.org/4.5.5/d9/db7/tutorial_py_table_of_contents_calib3d.html) （OpenCV Python 教程）

7. 特征探测和描述 - 特征框架 (feature2d 模块)

[2D 特征框架 (feature2d 模块)](https://docs.opencv.org/4.5.5/d9/d97/tutorial_table_of_content_features2d.html) （OpenCV 教程）
[特征探测和描述](https://docs.opencv.org/4.5.5/db/d27/tutorial_py_table_of_contents_feature2d.html) （OpenCV Python 教程）

8. 计算摄影

[其它教程 (ml，objdetect，photo，stitching，video)](https://docs.opencv.org/4.5.5/d3/dd5/tutorial_table_of_content_other.html) （OpenCV 教程）
[计算摄影](https://docs.opencv.org/4.5.5/d0/d07/tutorial_py_table_of_contents_photo.html) （OpenCV Python 教程）

这其中包含图像去噪、图像修补，和高动态范围 (HDR) 等内容。

9. 目标检测

[其它教程 (ml，objdetect，photo，stitching，video)](https://docs.opencv.org/4.5.5/d3/dd5/tutorial_table_of_content_other.html) （OpenCV 教程）
[目标探测 (objdetect 模块)](https://docs.opencv.org/4.5.5/d2/d64/tutorial_table_of_content_objdetect.html) （OpenCV Python 教程）

10. 视频分析

[其它教程 (ml，objdetect，photo，stitching，video)](https://docs.opencv.org/4.5.5/d3/dd5/tutorial_table_of_content_other.html) （OpenCV 教程）
[视频分析 (video 模块)](https://docs.opencv.org/4.5.5/da/dd0/tutorial_table_of_content_video.html) （OpenCV Python 教程）

11. 机器学习

[其它教程 (ml，objdetect，photo，stitching，video)](https://docs.opencv.org/4.5.5/d3/dd5/tutorial_table_of_content_other.html) （OpenCV 教程）
[机器学习](https://docs.opencv.org/4.5.5/d6/de2/tutorial_py_table_of_contents_ml.html) （OpenCV Python 教程）

12. 深度神经网络

[深度神经网络 (dnn 模块)](https://docs.opencv.org/4.5.5/d2/d58/tutorial_table_of_content_dnn.html) （OpenCV 教程）

13. 图 API

[图 API (gapi 模块)](https://docs.opencv.org/4.5.5/df/d7e/tutorial_table_of_content_gapi.html) （OpenCV 教程）

14. GPU 加速的计算机视觉

[GPU 加速的计算机视觉 (cuda 模块)](https://docs.opencv.org/4.5.5/da/d2c/tutorial_table_of_content_gpu.html) （OpenCV 教程）

15. 背景分割

[bgsegm 模块教程](https://docs.opencv.org/4.5.5/d3/d89/tutorial_table_of_content_bgsegm.html) （contrib 模块教程）

16. 超分辨率

[使用 CNN 的超分辨率](https://docs.opencv.org/4.5.5/d8/df8/tutorial_table_of_content_dnn_superres.html) （contrib 模块教程）

OpenCV 的这些教程，与一般的技术书还是有些不一样，它们并不是某个人或某个组织按照特定的规划，组织编写的结构严谨的教材，而是不同作者针对不同主题写就的文章的合集。

对于 OpenCV 具体模块的学习，不需要严格按照上面的路线图来走，具体模块学习的先后，可以按照自己的开发需要进行。

Done。
