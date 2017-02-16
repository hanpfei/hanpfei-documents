---
title: 传输层安全(TLS)下一个协议协商(NPN)扩展
date: 2016-10-29 13:48:49
categories: 网络协议
tags:
- 网络协议
- HTTP2
---


# 摘要

本文档描述了一种用于应用层协议协商的 传输层安全 (TLS) 扩展。这允许应用层协商在安全连接上应该执行的协议。

<!--more-->

# Status of this Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF).  Note that other groups may also distribute working documents as Internet-Drafts.  The list of current Internet-Drafts is at http://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time.  It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on October 31, 2012.

# Copyright Notice

Copyright (c) 2012 IETF Trust and the persons identified as the document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (http://trustee.ietf.org/ license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document.  Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

# 目录

## 1. 简介

## 2. 要求表示法

## 3. 下一个协议协商 扩展

## 4. 协议选择

## 5. 设计讨论

## 6. 安全性注意事项

## 7. IANA注意事项

## 8. 致谢

## 9. 参考文献

### 9.1. 规范性参考文献

### 9.2. 参考资料

## 作者地址

# 1. 简介

下一个协议协商 (NPN) 扩展当前被用于对443端口上的应用层协议SPDY [spdy]的使用的协商，并执行SPDY版本协商。然而，它也不是只能用在SPDY中。

新的应用层协议的设计者面临着一个问题：没有一种好的选项来为新协议建立一个干净的传输，并协商对它的使用。端口80上的协商将与拦截代理相冲突。80和443之外的其它端口可能被防火墙屏蔽，没有任何快速的检测方法，也不太可能通过CONNECT穿越HTTP代理。443端口上的协商是可能的，但可能与MITM相冲突，而且也要在建立TLS连接的网络往返的基础上为协商而使用一个往返。在那一层的协商还要依赖于应用层协议，比如，服务器对HTTP Upgrade请求的真实世界容忍

下一个协议协商 允许在没有额外的网络往返的情况下协商应用层协议，并且在不支持的MITM代理的情况下具有干净的回退。

# 2.  要求表示法

本文档中的关键词 "MUST"，"MUST NOT"，"REQUIRED"，"SHALL"，"SHALL NOT"，"SHOULD"，"SHOULD NOT"，"RECOMMENDED"，"MAY"，和"OPTIONAL" 如RFC 2119 [RFC2119] 中所述那样解释。

# 3. 下一个协议协商扩展

一个新的扩展类型 `("next_protocol_negotiation(TBD)")` 被定义出来了，它 **可以(MAY)** 被客户端包含在它的`"ClientHello"`消息中。当且仅当，服务器在 `"ClientHello"` 中看到了这个扩展，它 **可以(MAY)** 在它的 `"ServerHello"` 中选择回显该扩展。

```
enum {
   next_protocol_negotiation(TBD), (65535)
} ExtensionType;
```

`"ClientHello"` 中 `"next_protocol_negotiation"` 扩展的 `"extension_data"` 字段 **必须(MUST)** 是空的。

`"ServerHello"` 中 `"next_protocol_negotiation"` 扩展的 `"extension_data"` 字段包含一个可选的由服务器发布的协议的列表。协议的名字是不透明的，非空的字节串，

The "extension_data" field of a "next_protocol_negotiation" extension in a "ServerHello" contains an optional list of protocols advertised by the server.  Protocols are named by opaque, non-empty byte strings and the list of protocols is serialized as a concatenation of 8-bit, length prefixed byte strings.  Implementations MUST ensure that the empty string is not included and that no byte strings are truncated.

A new handshake message type ("encrypted_extensions(TBD)") is defined.  If the server included a "next_protocol_negotiation" extension in its "ServerHello" message, the client MUST send a "EncryptedExtensions" message after its "ChangeCipherSpec" and before its "Finished" message.

   enum {
     encrypted_extensions(TBD), (65535)
   } HandshakeType;

Therefore a full handshake with "EncryptedExtensions" has the following flow (contrast with section 7.3 of RFC 5246 [RFC5246]):

```
   Client                                               Server

   ClientHello (NPN extension)   -------->
                                                   ServerHello
                                                     (NPN extension &
                                                      list of protocols)
                                                  Certificate*
                                            ServerKeyExchange*
                                           CertificateRequest*
                                <--------      ServerHelloDone
   Certificate*
   ClientKeyExchange
   CertificateVerify*
   [ChangeCipherSpec]
   EncryptedExtensions
   Finished                     -------->
                                            [ChangeCipherSpec]
                                <--------             Finished
   Application Data             <------->     Application Data
```

An abbreviated handshake with "EncryptedExtensions" has the following flow:

```
   Client                                                Server

   ClientHello (NPN extension)    -------->
                                                   ServerHello
                                                     (NPN extension &
                                                      list of protocols)
                                            [ChangeCipherSpec]
                                 <--------            Finished
   [ChangeCipherSpec]
   EncryptedExtensions
   Finished                      -------->
   Application Data              <------->    Application Data
```

The "EncryptedExtensions" message contains a series of "Extension" structures (see section 7.4.1.4 of RFC 5246 [RFC5246]

If the server included a "next_protocol_negotiation" extension in its "ServerHello" message, the client MUST include an "Extension" with "extension_type" equal to "next_protocol_negotiation(TBD)".  The "extension_data" of which has the following format:

```
struct {
   opaque selected_protocol<0..255>;
   opaque padding<0..255>;
} NextProtocolNegotiationEncryptedExtension;
```

The contents of "selected_protocol" are an opaque protocol string, but need not have been advertised by the server.  The length of "padding" SHOULD be 32 - ((len(selected_protocol) + 2) % 32). Note that len(selected_protocol) does not include its length prefix.

Unlike many other TLS extensions, this extension does not establish properties of the session, only of the connection.  When session resumption or session tickets [RFC5077] are used, the previous contents of this extension are irrelevant and only the values in the new handshake messages are considered.

For the same reasons, after a handshake has been performed for a given connection, renegotiations on the same connection MUST NOT include the "next_protocol_negotiation" extension.

# 4. 协议选择

It's expected that a client will have a list of protocols that it supports, in preference order, and will only select a protocol if the server supports it.  In that case, the client SHOULD select the first protocol advertised by the server that it also supports.  In the event that the client doesn't support any of server's protocols, or the server doesn't advertise any, it SHOULD select the first protocol that it supports.

There may be cases where the client knows, via other means, that a server supports an unadvertised protocol.  In these cases the client can simply select that protocol.

# 5. 设计讨论

NPN is an outlier from TLS in several respects: firstly that it introduces a handshake message between the "ChangeCipherSpec" and "Finished" message, that the handshake message is padded, and that the negotiation isn't done purely with the hello messages.  All these aspects of the protocol are intended to prevent middle-ware discrimination based on the negotiated protocol and follow the general principle that anything that can be encrypted, should be encrypted.  The server's list of advertised protocols is in the clear as a compromise between performance and robustness.

# 6. 安全性注意事项

通过这个扩展，服务器支持的协议的列表依然是明文发布的。

The server's list of supported protocols is still advertised in the clear with this extension.  This may be undesirable for certain protocols (such as Tor [tor]) where one could imagine that hostile networks would terminate any TLS connection with a server that advertised such a capability.  In this case, clients may wish to opportunistically select a protocol that wasn't advertised by the server.  However, the workings of such a scheme are outside the scope of this document.

# 7. IANA注意事项

本文档需要IANA更新它的 TLS扩展 注册表，以分配此处称为 `"next_protocol_negotiation"` 的条目。

本文档还要求IANA更新它的TLS 握手类型注册表，以分配此处称为 `"encrypted_extensions"` 的条目。

本文档还要求IANA以先到先得的方式创建一个TLS 下一个协议协商 协议字符串 注册表，其中初始包含下面的条目：

* "http/1.1"：HTTP/1.1 [RFC2616]
* "spdy/1"：(废弃的) SPDY 版本 1
* "spdy/2"：SPDY 版本 2
* "spdy/3"：SPDY 版本 3

# 8. 鸣谢

This document benefited specifically from discussions with Wan-Teh Chang and Nagendra Modadugu.

# 9. 参考文献

## 9.1. 规范性参考文献

```
[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC5246]  Dierks, T. and E. Rescorla, "The Transport Layer Security
           (TLS) Protocol Version 1.2", RFC 5246, August 2008.
```

## 9.2. 参考资料


```
[RFC2616]  Fielding, R., Gettys, J., Mogul, J., Frystyk, H.,
           Masinter, L., Leach, P. and T. Berners-Lee, "Hypertext
           Transfer Protocol -- HTTP/1.1", RFC 2616, June 1999.

[RFC5077]  Salowey, J., Zhou, H., Eronen, P. and H. Tschofenig,
           "Transport Layer Security (TLS) Session Resumption without
           Server-Side State", RFC 5077, January 2008.

[tor]      Dingledine, R., Matthewson, N. and P. Syverson, "Tor: The
           Second-Generation Onion Router", August 2004.

[spdy]     Belshe, M. and R. Peon, "SPDY Protocol (Internet Draft)",
           Feb 2012.
```

# 作者地址

```
Adam Langley
Google Inc

Email: agl@google.com
```

[原文](https://tools.ietf.org/html/draft-agl-tls-nextprotoneg-04)
