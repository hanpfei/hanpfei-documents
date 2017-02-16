---
title: HTTP/2规范：2. HTTP/2 协议总览
date: 2016-10-29 12:27:49
categories: 网络协议
tags:
- 网络协议
- HTTP2
---

HTTP/2为HTTP语义提供了一个优化的传输方式。HTTP/2支持HTTP/1.1所有的核心功能，但在一些方面又更加高效。

HTTP/2中基本的协议单元是一个帧([Section4.1](http://httpwg.org/specs/rfc7540.html#FrameHeader))。每一个帧类型服务于一个不通的目的。比如，[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)和[DATA](http://httpwg.org/specs/rfc7540.html#DATA)帧构成了HTTP请求和响应的基础([Section8.1](http://httpwg.org/specs/rfc7540.html#HttpSequence))；其它的帧类型，比如[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)，[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，和[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)被用于支持其它的HTTP/2功能。

<!--more-->

请求的多路复用通过使得每个HTTP请求/响应交换关联它自己的流([Section5](http://httpwg.org/specs/rfc7540.html#StreamsLayer))来实现。流在很大程度上是相互独立的，因而一个阻塞或止步不前的请求或响应不会阻止其它流的进程。

Flow控制和优先级确保高效地使用多路复用的流是可能的。Flow控制([Section5.2](http://httpwg.org/specs/rfc7540.html#FlowControl))帮助确保只有能被接收者使用的数据才被传输。优先级([Section5.3](http://httpwg.org/specs/rfc7540.html#StreamPriority))确保有限的资源可以被首先导到最终要的流。

HTTP/2添加了一种新的交互模式，服务器可以借以向一个客户端推送响应([Section8.2](http://httpwg.org/specs/rfc7540.html#PushResources))。服务器推送允许一个服务器推测性地发送服务器预期客户端可能需要的数据给一个客户端，权衡比较网络流量的消耗和一个潜在延迟的增加。服务器通过合成一个请求来做到这一点，该请求以一个[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)来发送。然后服务器可以为合成的请求在另一个流上发送一个响应

由于一个连接中HTTP首部字段可能包含大量冗余数据，包含它们的帧被压缩([Section4.3](http://httpwg.org/specs/rfc7540.html#HeaderBlock))。这对于常见情况下的请求大小具有特别有利的影响，它允许多个请求被压缩为一个包。

## [2.1](http://httpwg.org/specs/rfc7540.html#rfc.section.2.1)文档组织
HTTP/2规范被分为了四个部分：
* 启动HTTP/2 ([Section3](http://httpwg.org/specs/rfc7540.html#starting))描述了一个HTTP/2连接是如何初始化的。 
* 帧([Section4](http://httpwg.org/specs/rfc7540.html#FramingLayer))和流([Section5](http://httpwg.org/specs/rfc7540.html#StreamsLayer))层描述了HTTP/2帧被结构化并构成多路复用的流的方式。
* 帧([Section6](http://httpwg.org/specs/rfc7540.html#FrameTypes))和错误([Section7](http://httpwg.org/specs/rfc7540.html#ErrorCodes))定义包括了HTTP/2中使用的帧类型和错误类型的细节。
* HTTP映射([Section8](http://httpwg.org/specs/rfc7540.html#HTTPLayer))和额外的需求([Section9](http://httpwg.org/specs/rfc7540.html#HttpExtra))描述了HTTP语义如何使用帧和流来表述。

尽管帧和流层的一些概念与HTTP是隔离的，但这份规范不定义一个完整的通用帧层。帧和流层被裁剪为满足HTTP协议和服务器推送的需要。

## [2.2](http://httpwg.org/specs/rfc7540.html#rfc.section.2.2)约定和术语
这份文档中关键字"MUST"，"MUST NOT"，"REQUIRED"，"SHALL"，"SHALL NOT"，"SHOULD"，"SHOULD NOT"，"RECOMMENDED"，"MAY"，和"OPTIONAL"的解释在[RFC 2119](http://httpwg.org/specs/rfc7540.html#RFC2119)[RFC2119]中描述。

所有的数字值以网络子节序。值是无符号的除非特别指明。字面值以十进制或在适当的时候以16进制提供。十六进制字面值以0x为前缀以与十进制字面值进行区分。

使用了下面的术语：
**客户端(client)**：初始化一个HTTP/2连接的终端。客户端发送HTTP请求并接收HTTP响应。
**连接(connection)**：两个终端之间的一个传输层连接。
**连接错误(connection error)**：影响整个HTTP/2连接的一个错误。
**终端(endpoint)**：连接中的客户端或服务器。
**帧(frame)**：一个HTTP/2连接内最小的通信单元，由一个首部和根据帧类型组织的一个可变长度的字节序列组成。
**对端(peer)**：一个终端。当讨论一个特定的终端时，"对端(peer)"指所讨论的主要科目的远端的终端。
**接收者(receiver)**：接收帧的终端。
**发送者(sender)**：传送帧的终端。
**服务器(server)**：接受一个HTTP/2连接的终端。服务器接收HTTP请求并发送HTTP响应。
**流(stream)**：HTTP/2连接内一个双向的帧流。
**流错误(stream error)**：独立的HTTP/2流上的一个错误。

最后，术语"网关(gateway)"，"intermediary"，"代理(proxy)"，和"隧道(tunnel)"在[[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 2.3](http://httpwg.org/specs/rfc7230.html#intermediaries)中定义。Intermediaries在不同的时间客户端和服务器的角色都扮演。

术语"载荷体(payload body)"在[[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 3.3](http://httpwg.org/specs/rfc7230.html#message.body)中定义。

