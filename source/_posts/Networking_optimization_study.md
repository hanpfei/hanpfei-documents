---
title: 网络优化实践探索文章
date: 2018-03-12 13:46:49
categories: 网络调试
tags:
- 网络调试
---

[携程App的网络性能优化实践](http://www.infoq.com/cn/articles/how-ctrip-improves-app-networking-performance)
<!--more-->
[2016年携程App网络服务通道治理和性能优化实践](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112505&idx=1&sn=70b6a46e92c372a09edc7050379bd158&scene=1&srcid=0803zfDgG6KTseJwK3UA16Z5&from=singlemessage&isappinstalled=0#wechat_redirect)

[蘑菇街App Chromium网络栈实践](http://www.infoq.com/cn/articles/mogujie-app-chromium-network-layer)

[手机淘宝性能优化](https://yq.aliyun.com/articles/53)

[无线性能优化：域名收敛](http://taobaofed.org/blog/2015/12/16/h5-performance-optimization-and-domain-convergence/)

[Facebook App对TLS的魔改造：实现0-RTT](http://chuansong.me/n/1553041851528)

[腾讯HTTPS性能优化实践](http://mt.sohu.com/20170220/n481149719.shtml)

[Android微信智能心跳方案](http://mp.weixin.qq.com/s/ghnmC8709DvnhieQhkLJpA)

[微信 Mars](https://github.com/Tencent/mars/wiki)

[Webkit 远程调试协议初探](http://taobaofed.org/blog/2015/11/20/webkit-remote-debug-test/)

[基于TLS1.3的微信安全通信协议mmtls介绍](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286266&idx=1&sn=f5d049033e251cccc22e163532355ddf&mpshare=1&scene=1&srcid=01036d3uXgfesl07utb04Hko&pass_ticket=81sKOxJ6az7s%2Bt9iX%2BimmP9aQMHGKJBFGy9jouP6LK%2FoKPoA34mrlGBUph5RD7LR#rd)

[微信Mars——移动互联网下的高质量网络连接探索](http://www.infoq.com/cn/presentations/wechat-mars-high-quality-network-connection)

[Facebook开源Proxygen HTTP框架和服务器](http://code.csdn.net/news/2822509)

[阿里无线11.11：手机淘宝移动端接入网关基础架构演进之路](http://www.infoq.com/cn/articles/taobao-mobile-terminal-access-gateway-infrastructure)

[Facebook wangle](https://www.slideshare.net/vorfeedchen/facebook-cwangle)

### [从 HTTP2 到 QUIC——QQ 空间 Web 加速实践](http://www.infoq.com/cn/presentations/from-http2-to-quic-qq-space-web-acceleration?utm_campaign=rightbar_v2&utm_source=infoq&utm_medium=presentations_link&utm_content=link_text)

### [QQ空间已在生产环境中使用QUIC协议](http://www.infoq.com/cn/news/2017/10/qzone-quic-practise)

QQ 空间已经在生产环境中应用了 QUIC 协议。在 Server 端，是由腾讯的安全云网关团队提供的支持，但 Server 端 QUIC 协议的具体实现方式，信息不是很明确。在用户端，分 PC 端和移动端两种场景，主要是在 PC 端用了 QUIC，理由是移动端支持 QUIC 的还比较少。这个意思似乎是，在用户端对 QUIC 协议的支持，主要是借助于浏览器实现的，特别是 Chrome/Chromium 浏览器。QQ 空间团队在落地 QUIC 协议的过程中，做了什么呢？主要是在 QUIC 协议性能不好时，切换到其它协议么？

### [QUIC在微博中的落地思考](http://www.infoq.com/cn/news/2018/03/weibo-quic)

微博在移动端应用了 QUIC 协议。从分享中，不难了解到，服务端采用开源的 Caddy Web 服务器来提供支持，但不是很清楚，是否有做一些特别的优化。而客户端，则是基于 Chromium 浏览器的网络库来实现的，分享中讲得很明白，从微博 APK 中 `/lib/armeabi/` 目录下的 `libcronet.so` 也能看出来。

从具体的应用效果来看，QUIC 在弱网环境下相比 TCP 有所提升，针对 CDN、流式传输等一些特殊场景也会有一些提升效果；总体上效果不是那么明显，甚至在一些场景下表现还不如 TCP。

### [An overview of TLS 1.3 and Q&A](https://blog.cloudflare.com/tls-1-3-overview-and-q-and-a/)

### [TLS 1.3 - Enhanced Performance, Hardened Security](https://www.cloudflare.com/learning-resources/tls-1-3/)

### [Netty干货分享：京东京麦的生产级TCP网关技术实践总结](https://www.jianshu.com/p/36308e2caf93?utm_campaign=maleskine&utm_content=note&utm_medium=pc_all_hots&utm_source=recommendation)
