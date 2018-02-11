---
title: 个人项目推广
date: 2018-01-19 11:43:49
categories: 随想杂谈
tags:
- 随想杂谈
---

一时兴起，搞了自己的开源项目或者是个人博客出来，虽然出发点常常并不是要利用这些得到多大的好处，而仅仅是出于一种保存曾经战斗过的地方的习惯，或者随手总结记录，以弥补随着年龄的增长而变得越来越差的记忆的不足，但如果自己搞得这些东西能被更多的人看到，能够帮助许许多多有需要的同学，并和全世界各地的广大开发者一起交流，那么对于写代码、写文档时孤寂的心倒也不失为一种很好的慰藉。将个人项目推广一下，为更多人所用，从个人角度而言，可以从中获得成就感，或者其它一些潜在的收益，从全社会而言，则是知识有了更多的沉淀，更多得人得到了帮助。
<!--more-->

![1.jpg](https://www.wolfcstech.com/images/1315506-15a25944cec8ed71.jpg)


笔者目前维护着多个 hosting 在 GitHub 上的开源项目，和多个个人博客。“不要重复制造轮子” 的声音时时在耳边响起，笔者的大多数个人开源项目都是基于前人已有的一些工作成果来完成的。开天辟地的颠覆式创新是伟大的，而一点一滴的微小改进，对现有事物的微创新也一样是有价值的，说不定从一个个的微创新的积累之中，也能产生不错的作品，所谓由量变到质变，积小流以成江河，积跬步以致千里是也。

笔者目前维护的开源项目主要有如下这些：

* **[ht-candywebcache-android](https://github.com/NEYouFan/ht-candywebcache-android)**：这是公司项目，是一个我们团队从零开始完整地开发出来的项目。它是一个移动端 Web 资源的本地缓存解决方案，能够拦截 WebView 的请求，优先使用本地缓存静态资源进行响应，以此来对 WebView 加载页面性能进行优化。在优化页面加载速度，以及移动端应用 Web 资源访问的流量消耗上面极具价值。

* **[brotli-android](https://github.com/NEYouFan/brotli-android)**：这是 Brotli 压缩算法的 Android 封装库，它由 Google 的 [brotli](https://github.com/google/brotli) 项目的 C 代码移植封装而来。Brotli 是 Web 中一个在各方面表现都比较不错的压缩算法，这个库使我们可以将 brotli 算法应用在我们的 Android 应用中。

* **[chromium-net](https://github.com/hanpfei/chromium-net)**：这个项目基于 Chromium 浏览器的网络模块创建。它主要剥离了 Chromium 的网络库及其依赖的一些库和工具链，对原有的代码做了一点点功能上的优化，并做了简单的封装以用于 Android 端。受益于 Chromium 网络库本身的强大，这个库在网络方面功能强大，它可以很好地支持 HTTP/2 和 QUIC 这样一些比较难找到移动端实现的协议。

* **[libcurl-for-android](https://github.com/hanpfei/libcurl-for-android)**：桌面／服务器网络编程利器 libcurl 的 android 移植封装，已通过包含 OpenSSL 内置提供对 HTTPS 的支持，且通过包含 nghttp2 以提供对 HTTP/2 的支持。这个项目对于已经熟悉了 libcurl，并想把它用在 Android 开发中的同学会比较有价值。

* **[LeakTracer](https://github.com/hanpfei/LeakTracer)**：Android native 层代码内存泄漏问题调试利器。由 Linux 平台 C/C++ 代码内存调试开源项目 LeakTracer 经移植、改进而来。这个工具对于分析、解决 Android 端本地层代码中的内存泄漏非常有价值。

* **[stund](https://github.com/hanpfei/stund)**：stund 是一个 STUN 协议的实现，但它有一个限制，就是必须部署在拥有两个公网 IP 的机器上。这个限制给当前在云主机上的部署造成了极大的困扰。这个项目对 stund 的实现做了一点小小的改动，以方便在多台主机上部署。

笔者目前维护的个人博客主要有：

 * [个人网站](https://www.wolfcstech.com/)，笔者购买的云主机，采用 nginx + hexo + markdown 文章搭建的博客站点。

 * [简书个人主页](https://www.jianshu.com/u/1109fa43aaf6)

 * [CSDN 个人博客主页](http://blog.csdn.net/tq08g2z)

本文记录笔者在个人项目推广方面所做的一些尝试。

# 开源项目的文档要尽可能齐全
为了方便有需要的同学能够更快地上手我们的项目，文档需要尽可能的齐全。

![](https://www.wolfcstech.com/images/1315506-cb07f7d769b83e0a.jpg)

[ht-candywebcache-android](https://github.com/NEYouFan/ht-candywebcache-android) 这个项目，在开源初期，只放了 Readme 文档以介绍项目的接入方法及基本用法。后来考虑到，还是应该帮助用户对我们的项目有更多更深的理解，于是以 Wiki 的形式放了设计文档，SDK 与服务器的通信协议文档，Diff 方式的详细说明文档等。

[chromium-net](https://github.com/hanpfei/chromium-net) 这个项目，最初同样只放了 Readme 文档，后来想起来，又以 Wiki 的形式补充了发在博客中，与这个项目有关的一些文档，比如代码分析，构建工具使用说明，开发环境配置等。

# 在问答社区回答相关的问题

![](https://www.wolfcstech.com/images/1315506-1f2a78658299259e.jpg)

为了使自己的项目得到更多的认识和关注，笔者曾尝试在如下的问答社区搜索或回答与项目有关的问题：
 * 在 [oschina 问答社区](https://www.oschina.net/question) 集中搜索并专门回答过关于 libcurl for android 的问题；
 * 在 [SegmentFault](https://segmentfault.com/) 上搜索过相关的 topic；
 * 在 [Stack Overflow](https://stackoverflow.com/) 上搜索并回答过关于 libcurl for android 的问题，然而，可能回答中推广自己项目的目的过于露骨，回答被社区删除，并被封号，好尴尬；
 * 在 [知乎](http://www.zhihu.com/) 搜索并回答过关于 flatbuffers 和 QUIC 协议相关的问题。

# 在项目集合中添加自己的开源项目

目前有一些开源的项目，或网站专门收集各种开源项目，分门别类，以方便开发者使用。笔者知道的比较流行的这类项目或网站有如下这些：

**[awesome-android-ui](https://github.com/wasabeef/awesome-android-ui)**：大名鼎鼎的 `awesome-android-ui`，汇集了非常多的 Android 开源 UI 控件项目，以方便开发者使用。如果要推广的开源项目正好是 Android UI 方面的，则尝试将项目的相关信息放进 `awesome-android-ui` 中不失为一个不错的注意。

**[codekk](http://p.codekk.com/)**：在 [Trinea](http://www.trinea.cn/dev-tools/development-tools/features-and-versions/) 的博客看到由它维护的站点，用于收集开源项目。前面列的几个项目，都有加进这个站点。

**[码库CTOLib](http://www.ctolib.com/)**：这个网站与 **codekk** 类似。这个似乎主要是自己分析 GitHub 项目的活跃度然后主动收录。自己可以提交自己的项目，不过入口略有点隐蔽。注册了帐号之后，在 **个人主页**，页面的下方有 **联系我们**，然后可以点击 **加入我们**（这个地方让人觉得好像是在给他们投简历一样，但实际上是提交项目的入口）。然后点击 **我要分享**，填写项目相关的信息分享项目，随后站点会做一个审核。感觉这个站点做的体验比较差一点，做审核不给出任何提示。审核成功之后，可以通过这个站点的入口，看到项目的相关信息。

**[开源中国-开源项目](https://www.oschina.net/project/zh)**：开源中国的开源项目频道。感觉它是开源项目集合中做的体验最好的一个。可以自己提交项目，但需要审核。前面列出的几个项目都提交了，但其中有两个由于文档太少，被拒收。

# 通过个人博客推广开源项目

笔者的 [个人博客](https://www.wolfcstech.com/)，有把一些项目的介绍性文档，作为博文放进去，以方便更多人看到并了解我们的开源项目。

经过对上面这些开源项目推广方法的尝试，笔者维护的几个开源项目的 stars 数，在一段时间之内有了比较大的提升，准确地说是从几个，增长到接近 200 个。

# 个人博客推广

对于个人博客的推广，也有一些方法可以尝试。首先是 SEO 优化，主动地将我么网站内容提交给 Google，Baidu，Bing 的访问量比较大的搜索引擎。其它更多 SEO 优化的方法，感兴趣的同学可以自己寻找。

此外，在自己注册的各个网站的个人资料中，写上自己的个人博客主页的地址，笔者的 [简书个人主页](https://www.jianshu.com/u/1109fa43aaf6)，[GitHub 主页](https://github.com/hanpfei)，[知乎主页](https://www.zhihu.com/people/han-peng-fei-49/activities)，这样可以为我们的网站引一些流量。通过百度的网站分析可以看到这一点。

![](https://www.wolfcstech.com/images/1315506-80eeaaf985c80052.png)

将我们的博文链接分享到 QQ 空间、微博、脉脉及加入的一些业内的技术交流 QQ 群也是个不错的注意。笔者感觉，微信朋友圈是一个比较私人的场合，博文的主题主要是技术相关的内容，因而笔者不常在微信朋友圈分享博文。

开发者圈子中，维护自己的博客的朋友很多，而大多又都是比较好的人，乐意共同推动开发者社区的发展，因而尝试与其它开发者互换友链也是不错的注意。笔者的个人主页与 [蚂蚁网](http://www.vants.org/) 互换了友链。

[掘金](https://juejin.im/timeline) 这个网站，当前在 http://alexa.chinaz.com 查到的全球综合排名是 9275 位，日均访问 IP 10W 左右，日均 PV 25W 左右。这个网站本身目前的流量还是比较可观的。掘金为我们提供了一个 **分享链接** 的功能，通过这个功能，我们可以把我们的项目主页、博客主页或者文章链接发布出去。

除了 [掘金](https://juejin.im/timeline) 之外，还有如下这些网站，我们也可以将我们的项目自荐过去：

| 网站名称    | 全球综合排名     | 日均访问 IP | 日均 PV |
|:--------|-------------|-------------|-------------:|
| [极客头条](http://geek.csdn.net/)  | 55 | 950 W | 4500 W |
| [36krNEXT](http://next.36kr.com/posts)  | 2032 | 32 W | 85 W |
| [开发者头条](http://toutiao.io/)  | 25308 | 2.6 W | 6.8 W |
| [Mindstore](http://mindstore.io/)  | 89150 | - | - |
| [掘金](https://juejin.im/timeline)  | 9275 | 10 W | 25 W |

其中 [极客头条](http://geek.csdn.net/) 是 CSDN 的一个频道，[36krNEXT](http://next.36kr.com/posts) 是 36氪 的一个频道，alexa.chinaz.com 能只查到主站的访问量，因而所列出的访问量也是主站的访问量。

[用于内推和招人的开源项目](http://b.codekk.com/detail/Trinea/%E7%94%A8%E4%BA%8E%E5%86%85%E6%8E%A8%E5%92%8C%E6%8B%9B%E4%BA%BA%E7%9A%84%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE) 这个项目，可以为我们发布职位招聘信息提供一些方便。

参考文档：
[个人项目如何推广可以获得大量的受众](https://www.yunyingpai.com/market/147.html)

Done.
