---
title: 我在开源项目推广上所做的尝试
date: 2017-05-19 11:43:49
categories: 随想杂谈
tags:
- 随想杂谈
---

我毕业的时候，进的是一家手机 IC 公司，即以 total solution 和华强北山寨机闻名世界的 MTK，在里面做 Android 应用开发和文本渲染 framework 的维护。IC 公司内部，技术氛围一般都很浓厚，MTK 也是这样。期间的 2012 年，开始在 oschina 写博客。尽管在学校的时候，在 China Unix 和 CSDN 上面，写过一些以 Linux kernel 和毕设相关主题的博客，但毕业之后就停掉了更新。2012 年的时候，算是重新把写博客给捡了起来。不过那个时候，写博客的主要目的是为了记录。小学的时候，老师就讲，好记忆不如烂笔头嘛 —— 将编程学习过程中的想法记录下来，方便自己后面有需要的时候查阅。以 IC 公司整个开发部门闷骚的风格，当时也从没有前辈提醒过运营的重要性，要去学习运营。

<!--more-->

后面的 13 年底离开 MTK，同样是在一家 IC 公司工作，运营对于技术、开发部门来说 —— 离得太远。15 年初进入一家互联网公司，然而是一家大多同事来自于非互联网行业的互联网公司。期间有放一些项目进 GitHub，主要是基于一些已有的开源项目，做的到 Android 平台的移植，优化和改进。那时开源项目，主要还是为了记录。在移植项目的时候，经常会遇到各种莫名奇妙的问题，然后就希望可以记录下来，通过一笔笔的 commit，以方便再遇到相同问题的时候，可以有个参考。相关的这些 commit，由于对于公司的项目来说，代码状态仅属办成品，因而不方便直接提交进公司的 repo。感谢公司的政策开放而宽容，开源的代码并没有触碰到保密的红线。

到 16 年 5 月底加入网易，在运营方面，老大提醒的就比较多了。老大经常提醒，要学着去运营自己，学着去运营项目，以提高自己的影响力、团队的影响力，并使自己的项目可以得到更多的使用。而且加入网易做的第一个项目，除了给公司产品用，在一开始就是计划要开源的。自那时起，就时常心系运营这个 topic。然后，有了一些推广运营的尝试，但具体的手法没有做过实际的数据及效果分析。

做一个开源项目出来，不管是从头到尾完全由自己开发，还是基于前人的劳动成果自己做的封装或改进，总是希望项目能得到更多的应用，能够创造更多的实际价值。自己及团队，从中获得影响力，获得更多的机会，获得成就感。此外，做技术本是一件很寂寞的事，如果有外部用户能够用自己的项目，给自己的项目提一些 issue，能讨论交流，则可以大大减轻那种寂寞感。

这里记录一下，前面一段时间，在开源项目的推广运营方面所做的一些尝试。

这里先放上我们之前开源的一些项目及简介。

**[ht-candywebcache-android](https://github.com/NEYouFan/ht-candywebcache-android)**：这是公司项目。它是一个移动端 Web 资源的本地缓存解决方案，能够拦截 webview 的请求，优先使用本地缓存静态资源进行响应，以此来对 Webview 加载页面性能进行优化。

**[brotli-android](https://github.com/NEYouFan/brotli-android)**：Brotli 压缩算法 Android 库，由 Google 的 [brotli](https://github.com/google/brotli) 项目的 C 代码移植封装而来。

**[chromium-net](https://github.com/hanpfei/chromium-net)**：Chromium 移动端网络库，当前移动端 QUIC 支持的良好选择。

**[libcurl-for-android](https://github.com/hanpfei/libcurl-for-android)**：桌面／服务器网络编程利器 libcurl 的 android 移植封装，已通过包含 OpenSSL 内置提供对 HTTPS 的支持，且包含 nghttp2 提供对 HTTP/2 的支持。

**[LeakTracer](https://github.com/hanpfei/LeakTracer)**：Android native 层代码内存泄漏问题调试利器。由 Linux 平台 C/C++ 代码内存调试开源项目 LeakTracer 经移植、改进而来。

**[stund](https://github.com/hanpfei/stund)**：可多物理主机部署的 STUN 服务器。

**[flatbuffers](https://github.com/hanpfei/flatbuffers)**：FlatBuffers Java 库及 gradle 消息文件构建插件。

# 项目的文档要齐全
为了方便有需要的朋友能够更快的上手我们的项目，文档需要尽可能的齐全。

[ht-candywebcache-android](https://github.com/NEYouFan/ht-candywebcache-android) 这个项目，在开源初期，只放了 Readme 文档以介绍项目的接入方法及基本用法。后来考虑到，还是应该帮助用户对我们的项目有更多更深的理解，于是以 Wiki 的形式放了设计文档，SDK 与服务器的通信协议文档，Diff 方式的详细说明文档等。

[chromium-net](https://github.com/hanpfei/chromium-net) 这个项目，同样是最初只放了 Readme 文档，后来偶然想起来，又以 Wiki 的形式补充了之前发在博客中，与这个项目有关的一些文档，比如代码分析，构建工具使用说明，开发环境配置等。

其它的一些项目，由于个人精力有限，还是一如既往的文档匮乏。

# 在问答社区回答相关的问题

为了使自己的项目得到更多的认识和关注，曾尝试在如下的问答社区搜索或回答过与项目有关的问题：
 * 在 [oschina 问答社区](https://www.oschina.net/question) 集中搜索并专门回答过关于 libcurl for android 的问题；
 * 在 [SegmentFault](https://segmentfault.com/) 上搜索过相关的 topic，然而没有找到；
 * 在 [Stack Overflow](https://stackoverflow.com/) 上搜索并回答过关于 libcurl for android 的问题，然而，可能回答中推广自己项目的目的过于露骨，回答被社区删除，好尴尬；
 * 在 [知乎](http://www.zhihu.com/) 搜索并回答过关于 flatbuffers 和 QUIC 协议相关的问题。

# 在一些项目集合中添加自己的项目

目前有一些开源的项目，或网站专门收集各种开源项目，分门别类，以方便开发者使用。目前已知比较流行的这类项目或网站有如下这些：

**[awesome-android-ui](https://github.com/wasabeef/awesome-android-ui)**：大名鼎鼎的 awesome-android-ui 项目，汇集了非常多的 Android 开源 UI 控件项目，以方便开发者使用。

**[codekk](http://p.codekk.com/)**：在大牛 [Trinea 的博客](http://www.trinea.cn/dev-tools/development-tools/features-and-versions/) 看到由它维护的站点。前面列的几个项目，都有加进这个站点。

**[码库CTOLib](http://www.ctolib.com/)**：这个 Site 与 **codekk**类似。这个似乎主要是自己分析 GitHub 项目的活跃度然后主动收录。自己可以提交自己的项目，不过入口略有点隐蔽。注册了帐号之后，在 “个人主页”，页面的下方有“联系我们”，然后可以点击“加入我们”，这个地方让人觉得是给他们投简历呢，但实际上是提交项目的入口。然后点击“我要分享”，填写项目相关的信息分享项目。随后站点会做一个审核，但感觉这个站点做的体验比较差一点，做审核也不给出任何提示，审核成功之后，可以通过这个站点的入口，看到项目的相关信息。

**[开源中国-开源项目](https://www.oschina.net/project/zh)**：开源中国的开源项目频道，感觉是开源项目集合中做的体验最好的一个，可以自己提交项目，但需要审核。前面列出的几个项目都提交了，但其中有两个由于文档太少，被拒收。

# 通过个人博客推广

我本人的[个人博客](https://www.wolfcstech.com/)，有把一些项目的介绍性文档，作为博文放进去。

# 互推

想到这一点，主要是看到 [stormzhang](http://stormzhang.com/)，[hukai](http://hukai.me/)，及 [Trinea](http://www.trinea.cn/) 他们几个的博客，相互为对方在自己的站点内提供外部链接。
