---
title: RTC 技术知识体系
date: 2020-03-22 21:05:49
categories:
- 音视频开发
tags:
- 音视频开发
---

RTC（Real-time Communications），直译或者广义指实时通信，狭义一般称为实时音视频，在这次全球大爆发的新冠肺炎疫情中，作为视频会议、视频通话、远程办公、远程医疗和互动直播等应用的底层技术，为全社会的尽力运转提供了巨大的支持。
<!--more-->
实时音视频本身并不是最近才出现的新技术，很早以前的网络教科书就已经在介绍 RTP 和 RTCP 了，如道格拉斯·科默 (Douglas E.Comer) 的 《用TCP/IP进行网际互联》。互联网语音通话、视频通话和视频会议等应用，也不是刚刚出现的新东西，几十年前这些应用就已经出现在许多地方了。只是受限于硬件的运算能力、网络传输带宽、网络传输技术和网络应用技术的发展，相关应用的部署、成本和体验，一直不太尽如人意，因而应用范围也就比较受限。

前些年网络带宽，网络技术如浏览器的快速进步，大大提升了视频网站的用户体验，并使之得到了广泛认可和应用，甚至使传统的音视频下载分发网站的市场大大萎缩。近些年及未来的计算能力提升，5G 网络高带宽低延迟传输技术提升，及音视频处理技术的发展等，RTC 应用的用户体验极大提升和广泛应用相信就在眼前了。

一般来说，一个完整的音视频系统大概是这样的：

![音视频系统](/images/1315506-a7efe2e196617036.png)

一个完整的音视频系统一般都会包含音视频采集，音视频数据的处理，音视频的编码，音视频编码数据的封装、保存，音视频编码数据的传输和分发，音视频的解码，音视频数据的处理，和音视频的播放和渲染。

很多年以前，大家依赖于音视频下载网站来欣赏音视频的时代中，完整的音视频系统中各个部分的角色和分工大概是这样的：专业的音视频制作团队完成音视频的数据采集、处理、编码和封装保存，产生最终的如 mp3 文件，mp4 文件，flv 文件，mkv 文件等媒体文件；音视频网站拿到这些音视频文件放在他们的网站上，我们大家从音视频网站上下载这些文件，如曾经我们常常以百度为入口下载各种音视频文件的网站；在我们本地的 PC 机，Mac，Android 或 iOS 设备中安装有专门的播放器来播放这些文件，如很多年以前的千千静听，Winamp，超级解霸，RealPlayer 等，后来出现的暴风影音，VLC，QQ 影音等，从而欣赏到音视频资源。这个时代的音视频系统大概是这样的：

![媒体文件时代的音视频系统](/images/1315506-5af2e8bd74343817.png)

这个时代中，音视频产业链中的不同团队可以更加专注于其中的一些环节，如音视频采集、处理、编码和封装保存到文件由专门的团队来做，音视频文件的分发下载由专门的团队来做，音视频文件的分发下载所用到的技术和其它各种文件的分发下载技术基本上没有本质任何区别，有专门的播放器团队提供播放器供用户下载使用。**这个时代中，不同部分的工作，独立性比较强，相互之间的影响较弱。**

下载媒体文件通常要耗费比较多的时间，下载到本地之后又需要占用大量的磁盘空间来保存，这些问题都常常造成不好的用户体验。随着网络带宽的增加，网络传输技术的发展，以及浏览器的发展，解决媒体文件时代的问题的条件逐渐成熟，于是我们进入了视频网站时代，或者说流媒体时代。此时的音视频系统大概是这样的：

![流媒体时代的音视频系统](/images/1315506-a73117f31e7d1393.png)

在流媒体时代，用户欣赏音视频资源所需经历的链路大为缩短，成本降低，用户体验大为提升。流媒体时代的音视频资源主要还是由专门的制作团队在做。流媒体网站拿到音视频文件，通常需要转码为更适宜分片传输下载的格式，流媒体网站整合浏览器的一些技术或流媒体播放技术，及传输技术，如 HLS，ffmpeg，HTML 5 等，使得用户打开浏览器或者 APP 就能直接欣赏音视频资源。本质上，此时在网络中传输的还是静态的音视频文件。网络如果偶尔卡顿一下，播放端通常通过数据的预取或缓冲等方式来解决。

人民群众的创造力才真正是无穷无尽的。随着音视频制作技术和工具的发展成熟和普及，特别是我们日常使用的手机等随身携带的设备，都具备了强大的音视频数据采集、处理和编码等强大能力，我们进入了音视频用户生成内容（UGC）的时代。此时在许多视频网站上出现了大量用户自己制作和上传的内容。相对于流媒体时代，此时的变化主要在于，充满创造力的广大人民，完成了大量的音视频内容制作的事情。用户欣赏音视频资源的方式主要还是浏览器和 APP。此时的音视频系统大概如下面这样：

![音视频用户生成内容（UGC）时代的音视频系统](/images/1315506-da167b6780e368ef.png)

音视频用户生成内容（UGC）还催生了短视频等极大丰富人们生活的工具。

此后，网络传输技术进一步发展，以降低延迟，并提升用户体验，于是出现了火爆的网络视频直播，我们进入视频直播时代。立足于之前音视频数据采集、处理、编码和播放技术的发展，网络带宽的增加，网络传输技术，如 RTMP，CDN 等的发展，各种形式的视频直播，如娱乐，电商等大量出现。此时从内容的制作，到这些内容触达用户的时间和链路都大为缩短。从用户侧来看，这和视频网站似乎没有太大的区别，然而底层支撑技术则已是有了巨大的改变。音视频内容，在直播中，保存为各种格式封装的文件不再那么必要。无封装的裸的各种媒体流，由各种媒体数据传输协议运载，穿行于我们的网络中。尽管此时从内容的制作到这些内容触达用户的时间和链路大为缩短，但这其中的延迟，几百 ms 一般还是有的。直播中的实时互动性也没有那么强。直播系统大致如下图所示：

![直播时代的音视频系统](/images/1315506-427af5f80fa0fabc.png)

同时直播系统对于互动性的要求没有那么强。技术上，传输过程中的问题，网络状况如带宽和延迟的变化，向前面的采集编码和处理，及后面的解码和处理的反馈无需那么及时，影响也不会很大。

技术继续向前发展，音视频数据的采集、处理、编解码和传输技术的进一步提升，人们随手持有的终端设备的计算能力大为提升，网络带宽提升和延迟降低，RTC 无处不在终于成为了可能。实时音视频系统大致如下图所示：

![实时音视频时代的音视频系统](/images/1315506-5917501187ac7bac.png)

实时音视频系统，相对于之前的音视频系统，最大的区别在于传输，以及传输和音视频数据处理、编解码之间的相互作用相互关系。

网络中通过 TCP 这种通用的可靠传输协议传输的数据，会由 TCP 层通过重传排序等机制，解决互联网数据传输天然具有的丢失、乱序和重复到达等问题。但在实时音视频中，对数据传输的及时性的要求通常要高于可靠性的要求。如发送端采集的一帧编码数据丢失了，对于接收播放端可能并没有太大的影响，接收播放端可以利用收到的前面和后面的帧，通过补帧等技术，实现同样好的用户体验，再如一帧音频数据丢失了，接收端可以用 NetEQ 等技术，根据收到的前面和后面的数据，用算法填上这一帧的数据，而不会降低用户体验。这是实时音视频中与一般的网络传输不同的地方。

另外，在实时音视频中，网络状况变化时，会对音视频的采集编码产生巨大的影响。在实时音视频系统中，探测到网络状况变差时，这种信息会被反馈给音视频的采集编码模块中，音视频的采集编码模块则会根据一定的规则，降低分辨率或降低码率等，以保证音视频的流畅性。在实时音视频中，一个终端，通常还需要扮演音视频内容的制作者和播放渲染者两种角色。

ffmpeg，x264，openh264，fdkaac，live555 等项目的开源和应用，使一般音视频系统的实现，相对于几十年前变得容易了许多，如播放器几乎变得无处不在。Google 对 WebRTC 的开源，也使得实时音视频系统的实现变得更加方便。不过，实时音视频依然有着一个非常复杂的技术知识体系，如下图所示：

![RTC 技术知识体系](/images/1315506-86d7a6b03ebab973.png)

音视频的采集和播放渲染，通常是与终端系统平台紧密相关的。实时音视频系统通常借助于终端系统平台提供的 API 和能力来完成这一功能。如对于音频的采集和播放，Android 有 AudioTrack、OpenSLES 和 AAudio，Linux 有 ALSA 和 Pulse 等，iOS、Mac 和 Windows 系统也都各有自己的音频采集和播放 API。对于视频的采集，通常不同的系统平台也都有各自访问摄像头的 API，和完成屏幕录制的 API。对于视频的播放和渲染，则需要借助于不同系统平台提供的图形和渲染接口来实现，如 OpenGL ES。

音频数据的处理，需要完成回声消除 AEC、降噪 ANS 和自动增益控制 AGC，AEC 和 ANS 在做什么可能比较容易理解，AGC 在字面上含义可能不那么直观。AGC 用于对录制的声音做一些自动的放大或缩小，以使得声音的接收者听到的声音大小不会有剧烈的变动，比如说话者说话的声音比较小，AGC 会做一些放大，是声音的接收者也能听得清。此外，抗丢包也是音频数据处理的重要内容。数据包在网络中传输时丢失是再正常不过的事了，不像通用的可靠传输协议如 TCP 和 QUIC 那样，对时效性要求不是那么高，且需要对数据的传输可靠性做很强的保证，在 RTC 这种对时效性要求极高的场景中，一般通过音频数据处理来解决，比如传输 3 个包，第1个和第3个收到了，但第二个丢失了，音频数据处理会通过一些软件算法，补上第二帧数据的内容，尽管声音可能会有一定的失真，但用户体验还是会好很多。WebRTC 的代码中包含了大量用于完成音频数据处理的内容，其中 3A 主要由 AudioProcessing 完成，抗丢包则由 NetEQ 完成。 还可以对声音应用一些音效，如混响，变声等等等。音频数据处理，特别是抗丢包和 3A 是 RTC 中一个非常精巧的部分，也是 RTC 相对于其它音视频系统的一个比较大的不同点。

音频数据的编码和解码，主要分为硬编硬解和软便软解。硬编硬解主要借助于终端设备上专门的硬件，通过特定终端系统提供的专门 API 来完成。软编软解则通过跨平台的一些专门的编解码库来完成，如 WebRTC 中包含有 OPUS 等格式的音频软件编解码器，fdkaac 库可以用于 AAC 的编码解码等。RTC 的编解码立足于其它音视频系统编解码的长久发展，但也有一些它自己特别的地方：一是码率的动态变化，即网络传输状态的信息反馈到编解码模块中，编码码率会动态的变化，以实现更好的延迟和音质的平衡；二是专门为 RTC 而生的编解码方式，如 OPUS 等。

视频数据的编码解码，同样分为硬编硬解和软编软解。硬编硬解同样主要借助于终端设备上专门的硬件，通过特定终端系统提供的专门 API 来完成。软编软解同样通过跨平台的一些专门的编解码库来完成，如 WebRTC 中包含有 VP8、VP9 等格式的视频软件编解码器，x264 和 openh264 库可以用于 H264 的编码解码，x265 可以用于 H265 的编码解码等。相对于音频，视频的硬编硬解通常要更加重要一点。软编软解对于许多终端的系统平台而言，实现高分辨率高帧率的流畅的编码解码和播放几乎是不可能实现的。与音频类似，RTC 中的视频编码码率也可能会根据网络传输状况的变化而变化。

不仅要高分辨率的流畅低延迟播放，对视频数据的处理，做出各种酷炫的效果，是吸引到众多用户的良好手段。

实时音视频中，除了音视频的内容外，网络传输相关的技术也十分重要。实时音视频传输目前应用最多的就是基于 UDP 的 RTP/RTCP 了。与 TCP 和 QUIC 这类通用的可靠传输协议不同，不同编解码格式的数据，在通过 RTP/RTCP 传输时，通常都有一份专门的 RFC 协议文档来定义具体的方法。如 [RFC6184](https://tools.ietf.org/html/rfc6184) （RTP Payload Format for H.264 Video），[RFC7741](https://tools.ietf.org/html/rfc7741) （RTP Payload Format for VP8 Video），[RFC7587](https://tools.ietf.org/html/rfc7587) （RTP Payload Format for the Opus Speech and Audio Codec），[RFC7587](https://tools.ietf.org/html/rfc7798) （RTP Payload Format for High Efficiency Video Coding (HEVC)），及 How to Write an RTP Payload Format 的 [RFC8088](https://tools.ietf.org/html/rfc8088) 等（更多相关的 RTC 文档可以在 RTC 的 index 页 [https://tools.ietf.org/rfc/index](https://tools.ietf.org/rfc/index) 搜 "RTP Payload Format"）。实现了不同音视频编解码格式针对 RTP 的打包解包之后，还需要借助于 RTCP 报告的网络状况信息，实现各种精细的网络传输控制，即拥塞控制 CC，以实现良好的用户体验。对于拥塞控制，除了丢包重传，FEC 前向冗余纠错也是比较常见的一个手段。RTSP 是基于 RTP/RTCP 设计的一种得到广泛应用的实时音视频流传输协议。QUIC 是另外一种基于 UDP 的，有可能可以用于实时音视频传输的网络协议。在实时音视频的开发中，对于相关的网络协议的了解几乎是必不可少的。

如我们前面提到的，实时音视频技术相对于之前的音视频技术有一个巨大的不同，即网络传输的部分检测到的网络状况，对于音视频的采集编码处理和播放都有巨大的影响。在 WebRTC 中有相当一部分代码在处理这部分逻辑。

网络协议也好，编解码也好，都需要具体实现的库提供支持，才能真正地应用于项目之中。之前的音视频系统中会用到的大量相关库在实时音视频系统中同样会被用到，如 ffmpeg，libx264，fdkaac，openh264 等。WebRTC 是实时音视频领域的一个集大成者，其中继集成了许多早已存在的音视频相关的库，同时也实现了许许多多实时音视频领域中专有的逻辑。

RTC 技术还要完成的一个重要的部分，就是提供良好的抽象，把前述各个子部分组合为一个灵活强大完整的 PIPELINE，并呈现给用户。

Done.