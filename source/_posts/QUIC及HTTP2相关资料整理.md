---
title: QUIC及HTTP2相关资料整理
date: 2017-01-24 17:43:49
tags:
- 网络
---

# 网络基础技术
 [The Transport Layer Security (TLS) Protocol Version 1.2](https://tools.ietf.org/html/rfc5246)
 [Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing](https://tools.ietf.org/html/rfc7230)
 [TLS v1.3规范](https://tlswg.github.io/tls13-spec/#implementation-notes)
<!--more-->
# QUIC
 [QUIC主页](https://www.chromium.org/quic)
 [QUIC概要设计文档](https://docs.google.com/document/d/1RNHkx_VvKWyWg6Lr8SZ-saqsQx7rFV-ev2jRFUoVD34/edit)

 [QUIC传输格式的详细设计文档](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit#)

 [Google的QUIC协议:可以将web的TCP转为UDP](http://bobao.360.cn/news/detail/3399.html)
 [Caddy – 最简单的支持 HTTP/2 的网页服务器](http://www.appinn.com/caddy-server/)
 [Caddy](https://github.com/mholt/caddy)
 [Caddy FAQ](https://caddyserver.com/docs/faq)
 [Caddy QUIC WiKi](https://github.com/mholt/caddy/wiki/QUIC)
 [怎样让网站支持QUIC](https://amon.org/quic.html)
 [现在怎么为自己网站开启 QUIC？](https://www.v2ex.com/t/300309)

# HTTP/2

 [HTTP/2规范](https://http2.github.io/http2-spec/)
 [HPACK规范](https://tools.ietf.org/html/rfc7541)
 [ALPN规范](https://tools.ietf.org/html/rfc7301)
 [NPN规范](https://tools.ietf.org/html/draft-agl-tls-nextprotoneg-04)
 [OkHttp对与NPN协议协商的支持](http://stackoverflow.com/questions/32492699/can-i-send-http-2-request-with-okhttp-over-npn-negotiation)

 [SPDY主页](https://www.chromium.org/spdy)
 [HTTP/2 GitHub主页](https://github.com/http2)
 [Known implementations of HTTP/2](https://github.com/http2/http2-spec/wiki/Implementations)
 [Netty支持HTTP/2 over cleartext TCP](http://netty.io/downloads.html)
 [HTTP/2 资料汇总 | JerryQu 的小站](https://imququ.com/)
 [HTTP 2.0的那些事](http://mrpeak.cn/blog/http2/)

 [未来已到——HTTP/2](https://segmentfault.com/a/1190000007637735)

 [https://www.google.com](https://www.google.com/)
 [https://www.aliyun.com](https://www.aliyun.com/)
 [https://www.youtube.com](https://www.youtube.com/)
 [https://www.facebook.com](https://www.facebook.com/)
 [https://twitter.com](https://twitter.com/)

## [When will nginx add support for QUIC?](https://www.quora.com/When-will-nginx-add-support-for-QUIC)

[Jaco Toledo](https://www.quora.com/profile/Jaco-Toledo), I have experience with HTML, CSS, PHP and MYSQL1.2k Views

Don't think nginx will support quic anytime soon.If you're interested in playing with QUIC today you'll need to build the test QUIC server and client that are part of the Chromium project, get Google Chrome Canary and enable QUIC in chrome://flags.
[Written 8 Feb](https://www.quora.com/When-will-nginx-add-support-for-QUIC/answer/Jaco-Toledo)

[Zhifeng Mi](https://www.quora.com/profile/Zhifeng-Mi)1.1k Views

nginx is working on TCP level 7
and quic is based on UDP
so I don't see any chance nginx will support quic.
And actually, you just need to develop a udp to tcp proxy to pass the data to nginx.
Now in Chrome if you enable quic, Chrome will start a proxy to send data to the
web server so if you don't want to modify the whole architect in your server side.
develop a upd to tcp proxy to support quic will be enough.
[Written 27 Dec 2015](https://www.quora.com/When-will-nginx-add-support-for-QUIC/answer/Zhifeng-Mi) · [View Upvotes](https://www.quora.com/api/mobile_expanded_voter_list?type=answer&key=XPaYjhf3yFS)

总结：
不太可能期待nginx去支持QUIC协议，在nginx中应用QUIC协议的方法是开发UDP到TCP的代理，由代理作为nginx服务器和终端进行通信时的桥梁。

## [What REST API Services Can Expect from HTTP/2](https://www.api2cart.com/blog/rest-api-services-can-expect-http2/)

One of the hottest topics for discussion today is the new HTTP/2 that is the most substantial protocol update since 1999, the year when the HTTP/1.1 version was released.

Among the** key improvements brought by HTTP/2** are multiplexed streams, header compression, server push, and a binary protocol instead of textual one. These and other positive changes allowed to achieve good web pages loading results, including those having lots of additional files attached to them (e.g. styles, scripts, images, fonts, etc.). But what effect is this going to have upon REST APIs and their work?

The first thing to talk about is the **format of data transmission**. As it has been mentioned above, a binary format has substituted a textual one which is good news for servers but not for developers. The latter one is convenient for program debugging but the binary one is still a good innovation, as it is much easier for a computer to process such data. Consequently, the binary format will contribute to lightening some of the server processor load, which would be helpful for heavily loaded REST APIs.

The protocol is compatible with the HTTP/1.1 version. Status codes, methods and headers did not overcome any changes. The only thing that did change is the way of transmitting data via TCP connection.

HTTP/2 is much more effective when it comes to** processing a large number of requests.** One of the biggest HTTP/1.1 problems is that separate requests are expensive. The reason for this is that many TCP connections are created, and these are established due to the “3-way handshake” method. Then some time has to pass for [“slow start”](http://en.wikipedia.org/wiki/Slow-start) algorithm to make the connection effectively fast. HTTP/2 is devoid of such a drawback thanks to multiplexed streams. This allows to make several HTTP requests as if they were made concurrently. Therefore, all the styles, scripts and images will load faster and users will certainly notice the speed increase. Although in REST APIs there is no need to load a few files simultaneously, multiplexing will let to perform a number of concurrent server requests more effectively due to one TCP connection being used for all these requests. This will also lighten the server load, as there will be no necessity to allocate resources to maintain more than one TCP connection, which will save some memory though the work of processor will be a little bit more intensive.

One more improvement that came with HTTP/2 is **header compression**. API requests usually have much less headers than ordinary web pages. That is why the benefits it may bring are most likely to be minor.

**“Server push” technology** made it possible for server to initiate data transmission. Here is an example. A client requests a web page. The web server processes the request and, as a result, gets an idea of what resources the client needs additionally and thus can send them right after the response with the document. This way, the client does not need to send additional request to get the additional resources mentioned above. Unfortunately, it is difficult to find a RESTful API service where this technology can be truly useful.

Though **many changes brought with HTTP/2** were created to speed web browsing up, some of them**can be used in the world of REST APIs as well**. It is possible to achieve greater efficiency at a lower cost if the HTTP/2 strong points are taken into consideration when designing an application.

总结：
HTTP/2的主要特性：
1. 多路复用流。降低多个文件传输时建立TCP连接及TCP`慢启动`的开销，降低服务器维护TCP连接的开销从而降低服务器负载，节省内存。
2. 首部压缩。
3. 服务器推送。
4. 二进制的协议。方便计算机对数据的处理，降低服务器CPU的负载。但对开发者调试不利。
5. 请求优先级。
6. 其它。

HTTP/2能够给Web页加载带来好处的特性：
几乎全部特性都能给Web页加载带来性能提升。

HTTP/2特性给REST API服务带来的影响：
1. **二进制的数据传输格式**。对于负载较重的REST API服务比较有用。
2. **多路复用流**。对于一次性发送大量请求的Web加载比较有意义。对于REST API服务，依然可以带来一定的好处，降低服务器负载，节省内存。
3. 首部压缩。对于REST API服务而言，通常首部的数据量都不是特别大，此项特性带来的好处并不大。
4. 服务器推送对REST API服务的意义并不大。

## [Chrome不再支持SPDY](http://blog.chromium.org/2016/02/transitioning-from-spdy-to-http2.html)

[Transitioning from SPDY to HTTP/2](http://blog.chromium.org/2016/02/transitioning-from-spdy-to-http2.html)
Thursday, February 11, 2016

Last year we [announced](https://blog.chromium.org/2015/02/hello-http2-goodbye-spdy.html) our intent to end support for the experimental protocol SPDY in favor of the standardized version, [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2). HTTP/2 is the next-generation protocol for transferring information on the web, improving upon HTTP/1.1 with more features leading to better performance. Since then we've seen huge adoption of HTTP/2 from both [web](http://isthewebhttp2yet.com/measurements/adoption.html) [servers](https://github.com/http2/http2-spec/wiki/Implementations) and [browsers](http://caniuse.com/#search=HTTP%2F2), with most now supporting HTTP/2. Over 25% of resources in Chrome are currently served over HTTP/2, compared to less than 5% over SPDY. Based on such strong adoption, starting on May 15th — the anniversary of the[HTTP/2 RFC](https://tools.ietf.org/html/rfc7540) — Chrome will no longer support SPDY. Servers that do not support HTTP/2 by that time will serve Chrome requests over HTTP/1.1, providing the exact same features to users without the enhanced performance of HTTP/2.

At the same time, Chrome will stop supporting the TLS protocol extension [NPN](https://tools.ietf.org/id/draft-agl-tls-nextprotoneg-04.html), which allows servers to negotiate SPDY and HTTP/2 connections with clients. NPN has been superseded by the TLS extension [ALPN](https://tools.ietf.org/html/rfc7301), published by the IETF in 2014. ALPN is already used 99% of the time to negotiate HTTP/2 with Chrome, and the remaining servers can gain ALPN support by [upgrading their SSL library](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation#Support).

We are looking forward to HTTP/2 continuing to gain adoption, bringing us an even faster web.

Update: To better align with Chrome's release cycle, SPDY and NPN support will be removed with the release of Chrome 51.

Posted by Bence Béky, Network Protocol Engineer and HTTP/2 Enthusiast

总结：
采用HTTP/2的Web服务器和浏览器非常多。Chrome访问的资源中有25%使用HTTP/2传输，而使用SPDY的资源不足5%。

Chrome在5月15日起，也就是正式版HTTP/2 RFC发布一周年的时候，不再支持SPDY了。不支持HTTP/2的服务器将用HTTP/1.1协议访问。
同时也将不再支持HTTP/2和SPDY的NPN协商，而只支持ALPN协商。Chrome中已经有99%的时间在使用ALPN协商。ALPN通常可以通过升级使用的SSL库来实现。
