---
title: 应用层协议协商（ALPN）规范（RFC7301）
date: 2016-10-29 13:44:49
categories: 网络协议
tags:
- 网络协议
- HTTP2
- 翻译
---

# 摘要

这份文档描述了一种传输层安全(TLS)扩展，即TLS握手消息里的应用层协议协商。比如在一些在相同的TCP或UDP端口上支持多种应用层协议的应用中，这个扩展允许应用层协商将在TLS连接内使用的协议。

<!--more-->

# Status of This Memo

This is an Internet Standards Track document.

This document is a product of the Internet Engineering Task Force(IETF).  It represents the consensus of the IETF community.  It has received public review and has been approved for publication by the Internet Engineering Steering Group (IESG).  Further information on Internet Standards is available in Section 2 of RFC 5741.

Information about the current status of this document, any errata, and how to provide feedback on it may be obtained at http://www.rfc-editor.org/info/rfc7301.

# Copyright Notice

Copyright (c) 2014 IETF Trust and the persons identified as the document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (http://trustee.ietf.org/license-info) in effect on the date of publication of this document.  Please review these documents carefully, as they describe your rights and restrictions with respect to this document.  Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

# 目录

## 1.  简介
## 2.  需求语言
## 3.  应用层协议协商
### 3.1.  应用层协议协商扩展
### 3.2.  协议选择
## 4.  设计注意事项
## 5.  安全注意事项
## 6.  IANA注意事项
## 7.  致谢
## 8.  参考文献
### 8.1.  规范性参考文献
### 8.2.  参考资料

# 1. 简介

越来越多的应用层协议被封装在了TLS协议[[RFC 5246](https://tools.ietf.org/html/rfc5246)]中。这种封装使得应用程序可以使用已经存在的，几乎遍布全球的IP基础设施的已经出现在端口443上的安全的通信链路。

当在单个服务器端端口号上支持多个应用协议时，比如端口443,客户端和服务器需要协商用于每个连接的应用协议。希望能在不增加客户端与服务器之间的网络往返的情况下完成这种协商，因为每次往返都将降低终端用户的体验。此外，最好能允许证书选择基于协商的应用协议。

本文档描述了一种TLS扩展，它允许应用层在TLS握手中协商协议。这项工作是HTTPbis工作组请求的，用以解决基于TLS的 HTTP/2 ([HTTP2]) 的协商；然而，ALPN便于任意应用层协议的协商。

使用ALPN，客户端发送支持的应用协议的列表作为TLS `ClientHello`消息的一部分。服务器选择一个协议，并将所选择的协议作为TLS `ServerHello`消息的一部分发送。因此，如果需要，可以在TLS握手期间完成应用层协议协商，而不增加网络往返，且允许服务器为每种应用协议关联不同的证书。

# 2. 需求语言

本文档中的关键字"MUST"，"MUST NOT"，"REQUIRED"，"SHALL"，"SHALL NOT"，"SHOULD"，"SHOULD NOT"，"RECOMMENDED"，"MAY"，和 "OPTIONAL"依照[[RFC2119](https://tools.ietf.org/html/rfc2119)]的描述进行解释。

# 3. 应用层协议协商

## 3.1. 应用层协议协商扩展

定义了新的扩展类型 ("application_layer_protocol_negotiation(16)")，并且 **可以(MAY)** 被客户端包含在其 "ClientHello" 消息中。

```
   enum {
       application_layer_protocol_negotiation(16), (65535)
   } ExtensionType;
```
`("application_layer_protocol_negotiation(16)")` 扩展的 `"extension_data"` 字段 **应该(SHALL)** 包含一个`"ProtocolNameList"`值。

```

opaque ProtocolName<1..2^8-1>;

struct {
    ProtocolName protocol_name_list<2..2^16-1>
} ProtocolNameList;
```

`"ProtocolNameList"`包含了由客户端发布的协议的列表，按优先级降序排列。协议按照IANA注册的，不透明的非空字节字符串命名，如在本文档的第6节 (“IANA注意事项”) 中所述。 **一定不能(MUST NOT)** 包含空字符串，字节串 **一定不能(MUST NOT)** 被截断。

接收到了包含`"application_layer_protocol_negotiation"`扩展的`ClientHello`消息的服务器 **可以(MAY)** 向客户端返回适当的协议选择响应。服务器将忽略任何它无法识别的协议名。 **可以(MAY)** 在扩展的`ServerHello`消息中向客户端返回新的`ServerHello`扩展类型`("application_layer_protocol_negotiation(16)")`。除了`"ProtocolNameList"` **必须(MUST)** 恰好包含一个 `"ProtocolName"`外，`("application_layer_protocol_negotiation(16)")`扩展的`"extension_data"`字段的结构与上述客户端`"extension_data"`一样。

因而，ClientHello和ServerHello消息中包含了`"application_layer_protocol_negotiation"`扩展的完整的握手的流程如下 (对比 [RFC5246] 的 7.3节)：


```
   Client                                              Server

   ClientHello                     -------->       ServerHello
     (ALPN extension &                               (ALPN extension &
      list of protocols)                              selected protocol)
                                                   Certificate*
                                                   ServerKeyExchange*
                                                   CertificateRequest*
                                   <--------       ServerHelloDone
   Certificate*
   ClientKeyExchange
   CertificateVerify*
   [ChangeCipherSpec]
   Finished                        -------->
                                                   [ChangeCipherSpec]
                                   <--------       Finished
   Application Data                <------->       Application Data

                                 Figure 1

```

* 表示 不总是发送的可选的或情境相关的消息。

包含了`"application_layer_protocol_negotiation"`扩展的简短的握手过程如下：

```
   Client                                              Server

   ClientHello                     -------->       ServerHello
     (ALPN extension &                               (ALPN extension &
      list of protocols)                              selected protocol)
                                                   [ChangeCipherSpec]
                                   <--------       Finished
   [ChangeCipherSpec]
   Finished                        -------->
   Application Data                <------->       Application Data
```

图 2

与许多其它TLS扩展不同，这个扩展不建立会话属性，而只建立连接的。当使用会话恢复或会话tickets [RFC5077] 时，此扩展之前的内容是不相关的，而只考虑新握手消息中的值。

## 3.2. 协议选择

预计服务器将有一个它支持的协议的列表，以优先级排序，而将只选择一个客户端支持的协议。在这种情况下，服务器 **应该(SHOULD)** 选择它支持且也由客户端发布的最优选的一个。如果服务器不支持客户端发布的协议，则服务器 **应该(SHALL)** 用 致命的`"no_application_protocol"`警告进行响应。

```
enum {
    no_application_protocol(120),
    (255)
} AlertDescription;
```
在`ServerHello`的`"application_layer_protocol_negotiation"`扩展类型中标识的协议对于连接 **应该(SHALL)** 确定的，直到重新协商为止。服务器 **不应该(SHALL NOT)** 以一个选择的协议进行响应，而后续却为应用数据交换使用一个不同的协议。

# 4. 设计注意事项

ALPN扩展旨在遵循TLS协议扩展的典型设计。具体地，协商完全是在 客户端/服务器 交换hello的过程中执行的，而与已建立的TLS架构保持一致。"application_layer_protocol_negotiation" ServerHello扩展意在对于连接是确定的 (直到连接被重协商)，且以明文发送，以允许当连接中使用的应用层协议的TCP或UDP端口号不确定的时候，网络元素为连接提供不同的服务。通过将协议选择权放在服务器，ALPN便于证书选择或连接重路由可以基于已协商的协议的情况。

最后，通过以作为握手消息的一部分的明文管理协议选择，ALPN避免了引入关于 在建立连接之前隐藏协商的协议 的能力 的虚假信任。如果需要隐藏协议，则在连接建立之后重协商，这将提供真正的TLS安全性保证，将是一个优选的方法。

# 5. 安全注意事项

ALPN扩展不影响TLS会话建立或数据交换的安全性。ALPN服务于为与TLS连接关联的应用层协议提供一个外部可见的标记。历史上，与一个连接关联的应用层协议，可以从使用的TCP或UDP端口号来确定。

想要通过添加新的协议标识符来扩展协议标识符注册表的实现者和文档编辑者应该考虑到在TLS版本1.2及更低的情况下，客户端是明文发送这些标识符的。他们还应该考虑到，最少在下一个十年内，预计浏览器将正常地在初始的ClientHello中使用这些较早的TLS版本。

必须注意这些标识符可能泄漏个人的可识别的信息，或当这种泄漏可能导致分析或泄漏敏感信息时。如果这些情况中的任何一个适用于新的协议标识符，则标识符 **不应该(SHOULD NOT)** 被用在TLS配置中，TLS配置将是明文可见的，在描述这种协议标识符的文档中 **应该(SHOULD)** 反对这种不安全的使用。

# 6. IANA注意事项

IANA已经更新了它的 "ExtensionType Values" 注册表来包含如下的项：

```
      16 application_layer_protocol_negotiation
```

该文档在现有的“传输层安全（TLS）扩展”标题下建立了名为“应用层协议协商（ALPN）协议ID”的协议标识符的注册表。

注册表中的项需要下面的这些字段：

* 协议：协议的名字。
* 标识序列：标识协议的精确的字节值集合。这可以是UTF-8编码 [RFC3629] 的协议名字。
* 参考文献：对一份定义了这个协议的规范的引用。

这个注册表依照 [RFC5226] 中定义的"Expert Review" 策略操作。建议指定的专家鼓励包含对一个永久的现成的规范的引用，以使能够创建标识的协议的可互操作的实现。

这个注册表的初始注册集合如下：

```
协议：HTTP/1.1
标识序列：0x68 0x74 0x74 0x70 0x2f 0x31 0x2e 0x31 ("http/1.1")
参考文献：[RFC7230]

协议：SPDY/1
标识序列：0x73 0x70 0x64 0x79 0x2f 0x31 ("spdy/1")
参考文献：http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft1

协议：SPDY/2
标识序列：0x73 0x70 0x64 0x79 0x2f 0x32 ("spdy/2")
参考文献：http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft2

协议：SPDY/3
标识序列：0x73 0x70 0x64 0x79 0x2f 0x33 ("spdy/3")
参考文献：http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3
```
# 7. 致谢

这份文档特别受益于由Adam Langley撰写的 下一个协议协商 (NPN) 扩展 文档，及与Cisco的Tom Wesselman和Cullen Jennings的讨论。

# 8.  参考文献

## 8.1.  规范性参考文献

```
[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC3629]  Yergeau, F., "UTF-8, a transformation format of ISO
           10646", STD 63, RFC 3629, November 2003.

[RFC5226]  Narten, T. and H. Alvestrand, "Guidelines for Writing an
           IANA Considerations Section in RFCs", BCP 26, RFC 5226,
              May 2008.

[RFC5246]  Dierks, T. and E. Rescorla, "The Transport Layer Security
           (TLS) Protocol Version 1.2", RFC 5246, August 2008.

[RFC7230]  Fielding, R. and J. Reschke, "Hypertext Transfer Protocol
           (HTTP/1.1): Message Syntax and Routing", RFC 7230, June 2014.
```

## 8.2.  参考资料

```
[HTTP2]    Belshe, M., Peon, R., and M. Thomson,  "Hypertext Transfer Protocol 
           version 2", Work in Progress, June 2014.

[RFC5077]  Salowey, J., Zhou, H., Eronen, P., and H. Tschofenig, "Transport Layer 
           Security (TLS) Session Resumption without  Server-Side State", RFC 5077, January 2008.
```

# 作者的地址

```
Stephan Friedl
Cisco Systems, Inc.
170 West Tasman Drive
San Jose, CA  95134
USA

Phone: (720)562-6785
EMail: sfriedl@cisco.com
```
```
   Andrei Popov
   Microsoft Corp.
   One Microsoft Way
   Redmond, WA  98052
   USA

   EMail: andreipo@microsoft.com
```

```
Adam Langley
Google Inc.
USA

EMail: agl@google.com
```

```
Emile Stephan
Orange
2 avenue Pierre Marzin
Lannion  F-22307
France

EMail: emile.stephan@orange.com
```
[原文](https://tools.ietf.org/html/rfc7301)
