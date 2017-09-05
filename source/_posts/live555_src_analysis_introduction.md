---
title: live555 源码分析：简介
date: 2017-08-28 22:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

live555 是一个 C++ 开发的流媒体项目，它主要由几个用于多媒体流的库组成，其官方网站地址为 http://www.live555.com/。live555 使用开放的标准协议 (RTP/RTCP，RTSP，SIP)，方便与其它标准的流媒体组件互操作。这些库可以为 Unix-like（包括 Linux 和 Mac OS X），Windows，和 QNX （及其它 POSIX 兼容系统）等系统进行编译，它们可以被用于构建流媒体应用。除了库之外，live555 还包含了两个流媒体应用程序 "[LIVE555 Media Server](http://www.live555.com/mediaServer/)" 和 "[LIVE555 Proxy Server](http://www.live555.com/proxyServer/)"，它们都是 RTSP 服务器应用程序。
<!--more-->
live555 的库可以被用于处理 MPEG，H.265，H.264，H.263+，DV 或 JPEG 视频，及多种音频格式。它们还可以非常简单地进行扩展，以支持其它的音频或视频编解码格式，并可以被用于构建基本的 [RTSP](http://www.live555.com/openRTSP/) 或 [SIP](http://www.live555.com/playSIP/) 客户端和服务器。

# 源码下载及编译

live555 的源代码是开放的，可以方便地供所有音视频开发研究爱好者学习研究，或者针对自己实际的项目进行扩展。其源码下载地址为 http://www.live555.com/liveMedia/public/ ：

![](https://www.wolfcstech.com/images/1315506-0708b1e8de87f8ac.png)

其中 [live555-latest.tar.gz](http://www.live555.com/liveMedia/public/live555-latest.tar.gz) 为最新版源码，[live.2017.07.18.tar.gz](http://www.live555.com/liveMedia/public/live.2017.07.18.tar.gz) 为最近一个正式的版本的源码。除了源码之外，live555 还提供了许多用于开发测试的音视频文件，如 `264` 目录下的是原始 H.264 码流测试文件， `265` 目录下的是原始 H.265 码流测试文件等。

这里使用 live555 的最新版本源码，[live555-latest.tar.gz](http://www.live555.com/liveMedia/public/live555-latest.tar.gz)，使用的操作系统为 64 位的 Ubuntu 16.04 版。下载源码，然后通过如下命令解压缩：

```
hanpfei0306@ThundeRobot:/media/data/osprojects$ tar xf live555-latest.tar.gz
hanpfei0306@ThundeRobot:/media/data/osprojects$ cd live
```

可以通过如下命令编译 live555：
```
hanpfei0306@ThundeRobot:/media/data/osprojects/live$ ./genMakefiles linux-64bit
hanpfei0306@ThundeRobot:/media/data/osprojects/live$ make
cd liveMedia ; make
. . . . . .
```

其中 `genMakefiles` 脚本用于产生 Makefile 文件，它需要一个操作系统版本的版本号作为参数，该脚本文件的内容如下：
```
#!/bin/sh

usage() {
    echo "Usage: $0 <os-platform>"
    exit 1
}

if [ $# -ne 1 ]
then
    usage $*
fi

platform=$1
subdirs="liveMedia groupsock UsageEnvironment BasicUsageEnvironment testProgs mediaServer proxyServer"
    
for subdir in $subdirs
do
    /bin/rm -f $subdir/Makefile
    cat $subdir/Makefile.head config.$platform $subdir/Makefile.tail > $subdir/Makefile
    chmod a-w $subdir/Makefile
done

/bin/rm -f Makefile
cat Makefile.head config.$1 Makefile.tail > Makefile
chmod a-w Makefile
```

这个脚本就是把各个目录下的多个文件预先定义的 Makefile 内容文件合并起来，产生最终的 Makefile 文件。每个文件夹下面都预定义了平台相关的配置文件，如，在项目根目录下：
```
hanpfei0306@ThundeRobot:/media/data/osprojects/live$ ls | grep config
config.aix
config.alpha
config.armeb-uclibc
config.armlinux
config.avr32-linux
config.bfin-linux-uclibc
config.bfin-uclinux
config.bsplinux
config.cris-axis-linux-gnu
config.cygwin
config.cygwin-for-vlc
config.freebsd
config.iphoneos
config.iphone-simulator
config.irix
config.linux
config.linux-64bit
config.linux-gdb
config.linux-with-shared-libraries
config.macosx
config.macosx-32bit
config.macosx-before-version-10.4
config.mingw
config.openbsd
config.qnx4
config.solaris-32bit
config.solaris-64bit
config.sunos
config.uClinux
```

提供给 `genMakefiles` 脚本的操作系统版本的版本号参数，需要与这些配置文件中，要编译的目标操作系统对应的那个配置文件的后缀名匹配。编译完成后，由各个库产生 `.a` 文件，各个库及各个应用程序的目标文件都位于它们自己的目录中。

媒体服务应用程序的可执行文件位于 `live/mediaServer/live555MediaServer`。

# 使用 live555MediaServer 提供流媒体服务

live555 中的流媒体服务器应用程序 `live555MediaServer` 可以非常方便地用来提供流媒体服务。在存放流媒体文件的目录下执行 `live555MediaServer`：
```
$ ./live555MediaServer
```

然后就可以通过如下格式的 URL 播放流媒体文件了：
```
rtsp://10.240.248.20:8554/<filename>
```
其中 `<filename>` 为执行 `live555MediaServer` 命令的目录下的流媒体文件，IP 地址为主机的 IP 地址。

`live555MediaServer` 程序运行起来之后，可以使用播放器软件，如 VLC Media Player 和 ffplay 播放流媒体内容。如：
```
$ ffplay rtsp://10.240.248.20:8000/raw_h264_stream.264
```

如果只是想玩一下 `live555MediaServer` 的话，还可以直接下载它的编译好的二进制文件，[地址](http://www.live555.com/mediaServer/)。

# live555 源码结构
接着来看 live555 的源码结构。首先为 live555 创建一个 Eclipse 的 C++ Project，方法为选择菜单栏的 File -> New -> C++ Project，弹出如下对话框：

![](https://www.wolfcstech.com/images/1315506-e6d9248984b721ee.png)

`Project Name: ` 一栏输入工程名字，这里用 `live555`；反选 `Use default location`，然后在 `Location: ` 一栏中输入 live555 源码的路径；在 `Project type: ` 下选择 `Makefile project` -> `Empty Project`；在 `Toolchains: ` 下选择 `Linux GCC`。

然后点击右下角的 `Finish` 按钮，创建工程。live555 源码结构如下：

![](https://www.wolfcstech.com/images/1315506-b9c143589ed1076b.png)

live555 源码主要由八个部分组成：UsageEnvironment，BasicUsageEnvironment，groupsock，liveMedia，mediaServer，proxyServer，testProgs，WindowsAudioInputDevice。

各个部分的代码量如下表所示：

子模块| 文件个数 | 代码量（行） |
-------|----------|----------|
UsageEnvironment | 3 | 162 |
BasicUsageEnvironment | 6 | 1187 |
groupsock | 8 | 2672 |
liveMedia | 168 | 49552 |
mediaServer | 2 | 332 |
proxyServer | 1 | 251 |
WindowsAudioInputDevice | 4 | 1037 |
testProgs | 32 | 6510 |
总共 | 224 | 61703 |

各个部分的简要说明如下。

## UsageEnvironment 和 BasicUsageEnvironment
UsageEnvironment 中的 "[UsageEnvironment](http://www.live555.com/liveMedia/doxygen/html/classUsageEnvironment.html)" 和 "[TaskScheduler](http://www.live555.com/liveMedia/doxygen/html/classTaskScheduler.html)" 类用于调度延迟的事件，为异步的读事件分配处理程序，以及输出错误/警告消息。UsageEnvironment 中的 "[HashTable](http://www.live555.com/liveMedia/doxygen/html/classHashTable.html)" 类还为范型哈希表定义了接口，由其余的代码使用。UsageEnvironment 中的都是抽象类；它们必须在实现中被继承。这些子类可以利用它运行的环境的特定属性，比如它的 GUI 和/或脚本环境。

BasicUsageEnvironment 库则定义了 UsageEnvironment 中的类的一个具体实现，用于简单的终端应用程序。读取事件和延迟操作使用一个 `select()` 循环处理。

## groupsock
这个库中的类封装了网络接口和 sockets。特别是其中的 "[Groupsock](http://www.live555.com/liveMedia/doxygen/html/classGroupsock.html)" 类封装了一个 socket，用于发送（和/或接收）组播数据报。

## liveMedia
这个库是 live555 的核心所在。其中定义了一个类层次体系，以 "[Medium](http://www.live555.com/liveMedia/doxygen/html/classMedium.html)" 为顶层基类，用于各种各样的流媒体类型和编解码。

## mediaServer 和 proxyServer
mediaServer 目录下的是 "LIVE555 Media Server"，它是一个完整的 RTSP 服务器应用程序。它可以把几种媒体文件转为流，如前面看到的，这些文件必须位于当前工作目录。这些文件包括：

 * MPEG TS 文件（文件后缀名为 ".ts"）
 * Matroska 或 WebM 文件（文件后缀名为 ".mkv" 或 ".webm"）
 * Ogg 文件（文件后缀名为 ".ogg"，"ogv" 或 ".opus"）
 * MPEG-1 或 2 程序流文件（文件后缀名为 ".mpg"）
 * MPEG-4 Video Elementary Stream 文件（文件后缀名为 ".m4e"）
 * H.264 Video Elementary Stream 文件（文件后缀名为".264"）
 * H.265 Video Elementary Stream 文件（文件后缀名为".265"）
 * VOB 视频+音频文件（文件后缀名为".vob"）
 * DV 视频文件（文件后缀名为".dv"）
 * MPEG-1 或 2 (包括 layer III - 比如 'MP3') 音频文件（文件后缀名为".mp3"）
 * WAV (PCM) 音频文件（文件后缀名为".wav"）
 * AMR 音频文件（文件后缀名为".amr"）
 * AC-3 音频文件（文件后缀名为".ac3"）
 * AAC (ADTS 格式) 音频文件（文件后缀名为".aac"）

proxyServer 目录下的是 "LIVE555 Proxy Server"，它是一个单播 RTSP 服务器，它为一个或多个 “后端” 单播或多播 RTSP/RTP 流扮演 “ 代理”的角色。

## WindowsAudioInputDevice
这是 "liveMedia" 库的 "AudioInputDevice" 抽象类的一个实现。它可以被 Windows 应用程序用于从输入设备读取 PCM 音频采样。

## testProgs
这个目录实现了一些简单的程序，使用 "BasicUsageEnvironment" 来演示如何使用这些库开发应用程序。其中除了包括用于测试库的测试应用之外，还包括 "[openRTSP](http://www.live555.com/openRTSP/)" - 命令行的 RTSP 客户端，"[playSIP](http://www.live555.com/playSIP/)" - 命令行的 SIP 会话记录器，"[vobStreamer](http://www.live555.com/vobStreamer/)" - 网络 DVD 播放器等工具。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# live555 源码分析系列文章
[live555 源码分析：简介](https://www.wolfcstech.com/2017/08/28/live555_src_analysis_introduction/)
[live555 源码分析：基础设施](https://www.wolfcstech.com/2017/08/30/live555_src_analysis_infrasture/)
[live555 源码分析：MediaSever](https://www.wolfcstech.com/2017/08/31/live555_src_analysis_mediaserver/)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](https://www.wolfcstech.com/2017/09/01/live555_src_analysis_rtsp_rtp_rtcp_wireshark/)
[live555 源码分析：RTSPServer](https://www.wolfcstech.com/2017/09/03/live555_src_analysis_rtspserver/)
[live555 源码分析： DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)
[live555 源码分析： SETUP 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_setup/)
