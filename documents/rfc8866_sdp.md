---
title: RFC 8866 SDP：会话描述协议
date: 2022-01-30 22:35:49
categories: 网络协议
tags:
- 网络协议
---

# 摘要

本备忘录定义了会话描述协议 (SDP)。SDP 旨在描述多媒体会话，用于会话公告、会话邀请和其它形式的多媒体会话启动。本文档已废弃 RFC 4566。


# 备忘录状态

这是一个互联网标准跟踪文件。

本文档是 IETF (Internet Engineering Task Force) 的产品。它代表了 IETF 社区的共识。它已接受公众审查，并已获互联网工程指导小组 (IESG) 批准出版。有关互联网标准的进一步信息，请参阅 [RFC 7841 的第2节](https://www.rfc-editor.org/rfc/rfc7841#section-2)。

有关此文档的当前状态、任何勘误表以及如何对其提供反馈的信息，可在以下网址获得 [ https://www.rfc-editor.org/info/rfc8866.]( https://www.rfc-editor.org/info/rfc8866.)。

# 版权声明

版权所有 (c) 2021 IETF 信托和确认为文档作者的人。保留所有权利。

本文档受 [BCP 78](https://www.rfc-editor.org/bcp/bcp78) 和自本文档发布之日起生效的 IETF 信托关于 IETF 文档的法律规定 (https://trustee.ietf.org/license-info) 的约束。请仔细审阅这些文件，因为它们描述了您在此文档方面的权利和限制。从本文档中提取的代码组件必须包含信托法律规定的第 4.e 节中描述的简化 BSD 许可证文本，且如简化 BSD 许可证所述，不为这些代码提供任何担保。

本文档可能包含在 2008 年 11 月 10 日前发布或公开的来自 IETF 文档或 IETF 贡献的材料。控制这些材料的某些版权的人可能没有授予 IETF Trust 允许在 IETF 标准过程之外修改这些材料的权利。没有从控制这些材料的版权的人那里获得足够的许可，本文档不得在 IETF 标准过程之外被修改，也不得在 IETF 标准过程之外被创作出其衍生作品，除非为了出版将其格式化为 RFC 或将其翻译成英语以外的语言。

# 1. 简介

在发起多媒体电话会议、IP 语音呼叫、流式视频或其它会话时，需要向参与者传达媒体详细信息、传输地址和其它会话描述元数据。

SDP 为此类信息提供标准表示，而不管该信息如何传输。SDP 纯粹是一种会话描述格式 —— 它不包含传输协议，它旨在酌情使用不同的传输协议，包括会话公告协议 (SAP) [[RFC2974](https://www.rfc-editor.org/rfc/rfc8866#RFC2974)]、会话发起协议 (SIP) [[RFC3261](https://www.rfc-editor.org/rfc/rfc8866#RFC3261)] ]、实时流协议 (RTSP) [[RFC7826](https://www.rfc-editor.org/rfc/rfc8866#RFC7826)]、使用 MIME 扩展 [[RFC2045](https://www.rfc-editor.org/rfc/rfc8866#RFC2045)] 的电子邮件 [[RFC5322](https://www.rfc-editor.org/rfc/rfc8866#RFC5322)] 和超文本传输协议 (HTTP) [[RFC7230](https://www.rfc-editor.org/rfc/rfc8866#RFC7230)]。

SDP 旨在通用，以便可以在广泛的网络环境和应用程序中使用。但是，它并不打算支持会话内容或媒体编码的协商：这被视为超出了会话描述的范围。

本备忘录已废弃 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)]。 本备忘录的 [第 10 节](https://www.rfc-editor.org/rfc/rfc8866#changes) 概述了与 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 相关的更改。

# 2. 专业术语

本文档中使用了以下术语，并在本文档的上下文中具有特定含义。

会话描述：一种定义明确的格式，用于传递足够的信息以发现和参与多媒体会话。

媒体描述：媒体描述包含一方建立与另一方的应用层网络协议连接所需的信息。它以 “m=” 行开始，并在下一个 “m=” 行或会话描述结束时终止。

会话级部分：这是指非媒体描述的部分，而会话描述是指包括会话级部分和媒体描述在内的整个主体。

本文档中使用的术语 “多媒体会议” 和 “多媒体会话” 在 [[RFC7656](https://www.rfc-editor.org/rfc/rfc8866#RFC7656)] 中定义。 术语 “会话” 和 “多媒体会话” 在本文档中可互换使用。

本文档中的关键词 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY” 和 “OPTIONAL” 应按照 [BCP 14](https://www.rfc-editor.org/bcp/bcp14) [[RFC2119](https://www.rfc-editor.org/rfc/rfc2119)] [[RFC8174](https://www.rfc-editor.org/rfc/rfc8174)] 的描述解释，当且仅当它们以大写字母出现时，如此处所示。

# 3. SDP 用法示例

## 3.1. 会话发起

[会话发起协议](https://www.rfc-editor.org/rfc/rfc8866#RFC3261) ([SIP](https://www.rfc-editor.org/rfc/rfc8866#RFC3261)) [[RFC3261](https://www.rfc-editor.org/rfc/rfc8866#RFC3261)] 是一种应用层控制协议，用于创建、修改和终止会话，例如 Internet 多媒体会议、Internet 电话呼叫和多媒体分发。用于创建会话的 SIP 消息携带会话描述，允许参与者就一组兼容的媒体类型达成一致 [[RFC6838](https://www.rfc-editor.org/rfc/rfc8866#RFC6838)]。这些会话描述通常使用 SDP 格式化。当与 SIP 一起使用时，[offer/answer 模型](https://www.rfc-editor.org/rfc/rfc8866#RFC3264) [[RFC3264](https://www.rfc-editor.org/rfc/rfc8866#RFC3264)] 为使用 SDP 的协商提供了一个有限的框架。

## 3.2. 流媒体

[实时流协议 (RTSP)](https://www.rfc-editor.org/rfc/rfc8866#RFC7826) [[RFC7826](https://www.rfc-editor.org/rfc/rfc8866#RFC7826)] 是一种应用层协议，用于控制具有实时属性的数据的传递。RTSP 提供了一个可扩展的框架，以支持实时数据，比如音频和视频，受控的、按需的传递。RTSP 客户端和服务器协商出一组适当的媒体传递参数集，部分使用 SDP 语法来描述这些参数。

## 3.3. 电子邮件和万维网

携带会话描述的其它方式还包括电子邮件和万维网 (WWW)。对于电子邮件和 WWW 分发，都使用媒体类型 “application/sdp”。这使得能够以标准方式从 WWW 客户端或邮件阅读器自动启动应用程序以参与会话。

请注意，仅通过电子邮件或 WWW 发送的多播会话的描述不具有会话描述的接收者必须接收会话的属性，因为多播会话的范围可能会受到限制，并且可以访问 WWW 服务器或接收电子邮件可能在这个范围之外。

## 3.4. 多播会话公告

为了帮助多播多媒体会议和其它多播会话的广告，并将相关会话建立信息传达给预期参与者，可以使用分布式会话目录。这种会话目录的一个实例定期将包含会话描述的数据包发送到众所周知的多播组。这些广告由其它会话目录接收，以便潜在的远程参与者可以使用会话描述来启动参与会话所需的工具。

用于实现这种分布式目录的一种协议是 [SAP](https://www.rfc-editor.org/rfc/rfc8866#RFC2974) [[RFC2974](https://www.rfc-editor.org/rfc/rfc8866#RFC2974)]。 SDP 为此类会话公告提供推荐的会话描述格式。

# 4. 要求和建议

SDP 的目的是在多媒体会话中传达有关媒体流的信息，以允许会话描述的接收者参与会话。SDP 主要用于 Internet 协议，尽管它足够通用，可以描述其它网络环境中的多媒体会议。媒体流可以是多对多的。会话不需要持续活跃。

到目前为止，Internet 上基于多播的会话与许多其它形式的会议的不同之处在于，任何接收流量的人都可以加入会话（除非会话流量被加密）。在这样的环境中，SDP 有两个主要目的。它是一种传达会话存在的方法，它是一种传达足够信息以使加入和参与会话的方法。在单播环境中，只有后一个目的可能是相关的。

SDP 描述包括以下内容：

 * 会话名称和目的
 * 会话处于活跃状态的时间
 * 包含会话的媒体
 * 接收这些媒体所需的信息（地址、端口、格式等）

由于参与会话所需的资源可能是受限的，因此可能还需要一些额外的信息：

 * 会话所使用的带宽的信息
 * 会议负责人的联系信息

一般而言，SDP 必须传达足够的信息以使应用程序能够加入会话（加密密钥可能除外），并向可能需要知道的任何非参与者公告要使用的资源。（当 SDP 与多播会话公告协议一起使用时，后一个功能非常有用。）

## 4.1. 媒体和传输信息

一个 SDP 描述包含如下的媒体信息：

 * 媒体的类型（视频，音频，等等）
 * 媒体传输协议（RTP/UDP/IP，H.320，等等）
 * 媒体的格式 （H.261 视频，MPEG 视频，等等）

除了媒体格式和传输协议，SDP 还传达地址和端口的详细信息。对于 IP 多播会话，这些包括：

 * 媒体的多播组地址
 * 媒体的传输端口

该地址和端口是多播流的目标地址和目标端口，无论是发送、接收还是两者。

对于单播 IP 会话，传达以下内容：

 * 媒体的远程地址
 * 媒体的远程传输端口

地址和端口的语义依赖于上下文。通常，这应该 (SHOULD) 是要发送或接收媒体的远程地址和远程端口。细节可能会有所不同，具体取决于指定的网络类型、地址类型、协议和媒体，以及 SDP 是作为广告分发还是在提议/应答 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8866#RFC3264)] 交换中协商。（例如，某些地址类型或协议可能没有端口的概念。）偏离典型行为应谨慎小心，因为这会使实现（包括必须解析地址以打开网络地址转换 (NAT) 或防火墙针孔的中间设备）复杂化。

## 4.2. 时间信息

会话在时间上可能是有界的或无界的。无论它们是否有界，它们可能仅在特定时间处于活动状态。SDP 可以传达：

 * 限制会话的任意开始和停止时间列表
 * 对于每个界限，重复时间，例如 “每周三上午 10 点，持续一小时”

此时间信息是全球一致的，与本地时区或夏令时无关（请参阅 [第 5.9 节](https://www.rfc-editor.org/rfc/rfc8866#timing)）。

## 4.3. 获取关于会话的更多信息

会话描述可以传达足够的信息来决定是否参与会话。SDP 可以包含统一资源标识符 (URI) [[RFC3986](https://www.rfc-editor.org/rfc/rfc8866#RFC3986)] 形式的附加指针，以获取有关会话的更多信息。（请注意，使用 URI 来指示远程资源受 [[RFC3986](https://www.rfc-editor.org/rfc/rfc8866#RFC3986)] 中的安全注意事项的约束。）

## 4.4. 国际化

SDP 规范建议在 UTF-8 编码 [[RFC3629](https://www.rfc-editor.org/rfc/rfc8866#RFC3629)] 中使用 ISO 10646 字符集，以允许表示多种不同的语言。然而，为了有助于紧凑的表示，SDP 还允许在需要时使用其它字符集，例如 [[ISO.8859-1.1998](https://www.rfc-editor.org/rfc/rfc8866#ISO.8859-1.1998)]。国际化仅适用于自由文本子字段（会话名称和背景信息），而不适用于整个 SDP。

# 5. SDP 规范

SDP 描述由媒体类型 “application/sdp” 表示（参见[第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。

SDP 描述是纯文本的。SDP 字段名和属性名只使用 UTF-8 [[RTC3629](https://www.rfc-editor.org/rfc/rfc8866#RFC3629)] 的 US-ASCII 子集，但文本字段和属性值可以 (MAY) 使用 UTF-8 编码的完整 ISO 10646 字符集，或通过 “a=charset:” 属性（[第 6.10 节](https://www.rfc-editor.org/rfc/rfc8866#charset)）定义的某些其它字符集。永远不会直接比较使用完整 UTF-8 字符集的字段和属性值，因此不需要 UTF-8 规范化。选择文本形式，而不是 ASN.1 或 XDR 等二进制编码，以增强可移植性，支持各种传输方式的使用，并允许使用灵活的基于文本的工具包来生成和处理会话描述。然而，由于 SDP 可以在会话描述的最大允许大小受到限制的环境中使用，因此编码方式是故意设计的比较紧凑的。此外，由于描述可能通过非常不可靠的方式传输，或被中间缓存服务器损坏，因此编码设计有严格的顺序和格式规则，以使大多数错误会导致格式错误的会话描述，这样就可以很容易地检测到并丢弃它。

SDP 描述由以下格式的多行文本组成：
```
   <type>=<value>
```

其中 <type> 恰好是一个区分大小写的字符，而 <value> 是其格式取决于 <type> 的结构化文本。通常，<value> 是由单个空格字符或自由格式字符串分隔的多个子字段，并且除非特定字段另有定义，否则是区分大小写的。“=” 号的两侧都不使用空格

分隔符，但是，值可以包含前导空格作为其语法的一部分，即该空格是值的一部分。

SDP 描述必须符合 [第 9 节](https://www.rfc-editor.org/rfc/rfc8866#abnf) 中定义的语法。以下是该语法的概述。

SDP 描述由一个会话级部分，后跟零个或多个媒体描述组成。会话级部分以 “v=” 行开始，并持续到第一个媒体描述（或整个描述的结尾，以先到者为准）。每个媒体描述都以 “m=” 行开始，并持续到下一个媒体描述，或整个会话描述的结尾，以先到者为准。一般来说，会话级值是所有媒体的默认值，除非被等效的媒体级值覆盖。

每个描述中的某些行是必需的，有些是可选的，但如果存在，它们必须完全按照此处给出的顺序出现。（固定顺序极大地增强了错误检测并允许使用简单的解析器）。在以下概述中，可选项都标有 “*”。
```
   Session description
      v=  (protocol version)
      o=  (originator and session identifier)
      s=  (session name)
      i=* (session information)
      u=* (URI of description)
      e=* (email address)
      p=* (phone number)
      c=* (connection information -- not required if included in
           all media descriptions)
      b=* (zero or more bandwidth information lines)
      One or more time descriptions:
        ("t=", "r=" and "z=" lines; see below)
      k=* (obsolete)
      a=* (zero or more session attribute lines)
      Zero or more media descriptions

   Time description
      t=  (time the session is active)
      r=* (zero or more repeat times)
      z=* (optional time zone offset line)

   Media description, if present
      m=  (media name and transport address)
      i=* (media title)
      c=* (connection information -- optional if included at
           session level)
      b=* (zero or more bandwidth information lines)
      k=* (obsolete)
      a=* (zero or more media attribute lines)
```
类型字母集合故意设计的很小，并且不打算进行扩展 —— SDP 解析器必须 (MUST) 完全忽略或拒绝任何包含它不理解的类型字母的会话描述。属性机制（“a=”，在[第 5.13 节](https://www.rfc-editor.org/rfc/rfc8866#attribspec) 中描述）是扩展 SDP 并将其剪裁用于特定应用程序或媒体的主要手段。一些属性（[第 6 节](https://www.rfc-editor.org/rfc/rfc8866#attrs) 中列出的那些）具有已定义的含义，但其它属性可以在特定于媒体或会话的基础上添加。（除了媒体特有和会话特有范围之外的属性范围也可以在本文档的扩展中定义，例如 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8866#RFC5576)] 和 [[RFC8864](https://www.rfc-editor.org/rfc/rfc8866#RFC8864)]。）SDP 解析器必须 (MUST) 忽略任何它不理解的属性。

SDP 描述可能在 “u=”、“k=” 和 “a=” 行中包含引用外部内容的 URI。在某些情况下，这些 URI 可能会被解引用，使会话描述非自包含。

会话级部分中的连接（“c=”）信息适用于该会话的所有媒体描述，除非被媒体描述中的连接信息覆盖。例如，在下面的示例中，每个音频媒体描述的行为就好像它被赋予了 "c=IN IP4 198.51.100.1"。

一个示例 SDP 描述如下：
```
      v=0
      o=jdoe 3724394400 3724394405 IN IP4 198.51.100.1
      s=Call to John Smith
      i=SDP Offer #1
      u=http://www.jdoe.example.com/home.html
      e=Jane Doe <jane@jdoe.example.com>
      p=+1 617 555-6011
      c=IN IP4 198.51.100.1
      t=0 0
      m=audio 49170 RTP/AVP 0
      m=audio 49180 RTP/AVP 0
      m=video 51372 RTP/AVP 99
      c=IN IP6 2001:db8::2
      a=rtpmap:99 h263-1998/90000
```

包含文本的字段，例如会话名称字段和信息字段是八位字节串，可以包含任何八位字节，但 0x00（Nul）、0x0a（ASCII 换行符）和 0x0d（ASCII 回车符）除外。序列 CRLF (0x0d0a) 用于结束一行，尽管解析器应该 (SHOULD) 容忍并接受以单个换行符结尾的行。如果 “a=charset:” 属性不存在，这些八位字节串必须 (MUST) 被解释为包含 UTF-8 编码的 ISO-10646 字符。当存在 “a=charset:” 属性时，会话名称字段、信息字段和一些属性字段将根据所选字符集进行解释。

会话描述可以在 “o=”、“u=”、“e=”、“c=” 和 “a=” 行中包含域名。SDP 中使用的任何域名必须符合 [[RFC1034](https://www.rfc-editor.org/rfc/rfc8866#RFC1034)] 和 [[RFC1035](https://www.rfc-editor.org/rfc/rfc8866#RFC1035)]。国际化域名 (IDN) 必须 (MUST) 使用 [[RFC5890](https://www.rfc-editor.org/rfc/rfc8866#RFC5890)] 中定义的 ASCII 兼容编码 (ACE) 形式表示，并且不得 (MUST NOT) 直接以 UTF-8 或任何其它编码表示（此要求是为了与 [[RFC2327](https://www.rfc-editor.org/rfc/rfc8866#RFC2327)] 和其它早期的 SDP 相关标准兼容，它们早于国际化域名的开发）。

## 5.1. 协议版本 ("v=")

```
      v=0
```

"v=" 行 (版本字段) 给出了会话描述协议的版本。本备忘录定义版本 0。没有次要版本号。

## 5.2. 源 ("o=")

```
     o=<username> <sess-id> <sess-version> <nettype> <addrtype>
     <unicast-address>
```

"o=" 行 (源字段) 给出了会话的发起者（她的用户名和用户的主机的地址）加上会话标识符和版本号：

<username> 是用户在发起主机上的登录名，如果发起主机不支持用户 ID 的概念，则为“-”。 <username> 不得 (MUST NOT) 包含空格。

<sess-id> 是一个数字串，这样 <username>、<sess-id>、<nettype>、<addrtype> 和 <unicast-address> 的元组构成会话的全局唯一标识符。<sess-id> 分配的方式由创建工具决定，但建议使用时间戳，以 UTC 1900 年 1 月 1 日以来的秒数为单位，以确保唯一性。

<sess-version> 是此会话描述的版本号。它的用法取决于创建工具，只要在修改会话描述时增加 <sess-version> 即可。同样，与 <sess-id> 一样，建议使用时间戳。

<nettype> 是一个给出网络类型的文本字符串。最初，只定义了 “IN” ，其含义为 “Internet”，但将来可以 (MAY) 注册其它值（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。

<addrtype> 是一个文本字符串，给出后面地址的类型。 最初，定义了 “IP4” 和 “IP6”，但将来可以 (MAY) 注册其它值（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。

<unicast-address> 是创建会话的机器的地址。对于地址类型 “IP4”，这是机器的完全限定域名，或机器的 IPv4 地址的点分十进制表示。对于地址类型 “ IP6”，这是机器的完全限定域名，或[[RFC5952](https://www.rfc-editor.org/rfc/rfc8866#RFC5952)] 的[第 4 节](https://www.rfc-editor.org/rfc/rfc5952#section-4) 中指定的机器地址。对于 “IP4” 和 “IP6”，完全限定的域名是应该 (SHOULD) 给出的形式，除非它不可用，在这种情况下，可以 (MAY) 用一个全球唯一的地址代替。

通常，“o=” 行用作此版本的会话描述的全局唯一标识符，并且与除版本之外的子字段，一起标识会话，而与任何修改无关。

出于隐私原因，有时需要混淆会话发起者的用户名和 IP 地址。如果这是一个问题，可以选择任意 <username> 和私有 <unicast-address> 来填充 “o=” 行，前提是选择这些的方式不会影响字段的全局唯一性。

## 5.3. 会话名称 ("s=")
```
   s=<session name>
```

"s=" 行 (session-name-field) 是文本的会话名称。每个会话描述必须 (MUST) 有且只有一个 “s=” 行。"s=" 行不能为空。如果会话没有有意义的名称，则建议 (RECOMMENDED) 使用 “s= ” 或 “s=-”（即，单个空格或破折号作为会话名称）。如果存在会话级 “a=charset:” 属性，则它指定 “s=” 字段中使用的字符集。如果会话级 “a=charset:” 属性不存在，“s=” 字段必须 (MUST) 包含 UTF-8 编码的 ISO 10646 字符。

## 5.4. 会话信息 ("i=")
```
   i=<session information>
```

“i=”行（信息字段）提供有关会话的文本信息。每个会话描述最多可以有一个会话级 “i=” 行，每个媒体描述最多可以有一个 “i=” 行。除非提供了媒体级别的 “i=” 行，否则会话级别的 “i=” 行适用于该媒体描述。如果存在 “a=charset:” 属性，则它指定 "i=" 行中使用的字符集。如果 “a=charset:” 属性不存在，“i=” 行必须 (MUST) 包含 UTF-8 编码的 ISO 10646 字符。

每个媒体描述最多可以使用一个 “i=” 行。在媒体定义中，“i=” 行主要用于标记媒体流。因此，当单个会话具有多个相同媒体类型的不同媒体流时，它们最有可能有用。一个例子是两个不同的白板，一个用于幻灯片，一个用于反馈和问题。

“i=” 行旨在提供对会话或媒体流目的的自由形式的人类可读描述。 不适合自动机解析。

## 5.5. URI ("u=")
```
   u=<uri>
```

“u=” 行（uri-field）提供了一个 URI（统一资源标识符）[[RFC3986](https://www.rfc-editor.org/rfc/rfc8866#RFC3986)]。URI 应该是指向有关会话的其它人类可读信息的指针。此行是可选的 (OPTIONAL)。每个会话描述最多允许一个 “u=” 行。

## 5.6. Email 地址和电话号码 ("e=" 和 "p=")
```
   e=<email-address>
   p=<phone-number>
```

“e=” 行（电子邮件字段）和 “p=” 行（电话字段）指定负责会话的人员的联系信息。 这不一定是创建会话描述的同一个人。

包含电子邮件地址或电话号码是可选的 (OPTIONAL)。

如果存在电子邮件地址或电话号码，则必须 (MUST) 在第一个媒体描述之前指定。 会话描述可以提供多个电子邮件或电话行。

电话号码应该 (SHOULD) 以国际公共电信号码的形式给出（参见 ITU-T 建议 E.164 [[E164](https://www.rfc-editor.org/rfc/rfc8866#E164)]），前面带有 “+”。如果需要，可以使用空格和连字符来分隔电话字段以提高可读性。 例如：
```
   p=+1 617 555-6011
```

电子邮件地址和电话号码都可以有一个可选的 (OPTIONAL) 自由文本字符串与之关联，通常给出可能被联系的人的姓名。如果存在，则必须 (MUST) 将其括在括号中。 例如：
```
   e=j.doe@example.com (Jane Doe)
```

电子邮件地址和电话号码也允许使用替代的 [[RFC5322](https://www.rfc-editor.org/rfc/rfc8866#RFC5322)] 名称引用约定。 例如：
```
   e=Jane Doe <j.doe@example.com>
```

自由文本字符串应该 (SHOULD) 采用 UTF-8 编码的 ISO-10646 字符集，或者如果设置了适当的会话级 “a=charset:” 属性，则采用 ISO-8859-1 或其它编码。

## 5.7. 连接信息 ("c=")
```
   c=<nettype> <addrtype> <connection-address>
```

“c=” 行（连接字段）包含建立网络连接所需的信息。

会话描述必须 (MUST) 在每个媒体描述中至少包含一个 “c=” 行，或者在会话级别包含一个 “c=” 行。它可以 (MAY) 包含单个会话级别 “c=” 行，和附加的每个媒体描述的媒体级别 “c=” 行，在这种情况下，媒体级别值会覆盖相应媒体的会话级别设置 .

第一个子字段 (<nettype>) 是网络类型，它是一个给出网络类型的文本字符串。最初，只定义了 “IN” ，其含义为 “Internet”，但将来可以 (MAY) 注册其它值（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。

第二个子字段 (<addrtype>) 是地址类型。这允许 SDP 用于不基于 IP 的会话。本备忘录定义了 “IP4” 和 “IP6”，但将来可以 (MAY) 注册其它值（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。

第三个子字段（<connection-address>）是连接地址。根据 <addrtype> 子字段的值，可以 (MAY) 在连接地址之后添加额外的子字段。

当 <addrtype> 为 "IP4" 或 "IP6" 时，连接地址定义如下：

 * 如果会话是多播的，则连接地址将是一个 IP 多播组地址。如果会话不是多播的，则连接地址包含由附加属性字段确定的预期数据源、数据中继或数据接收器的单播 IP 地址（[第 5.13 节](https://www.rfc-editor.org/rfc/rfc8866#attribspec)）。不期望单播地址将在由多播公告传达的会话描述中给出，尽管不禁止这样做。

 * 除了多播地址之外，使用 “IP4” 多播连接地址的会话还必须 (MUST) 具有生存时间 (TTL) 值。TTL 和地址一起定义了在这个会话中发送的多播数据包将被发送的范围。TTL 值必须 (MUST) 在 0-255 范围内。 尽管必须 (MUST) 指定 TTL，但用它来确定多播流量的范围的做法被废弃； 应用程序应该 (SHOULD) 使用管理范围的地址。

会话的 TTL 使用斜杠作为分隔符附加到地址。 一个例子是：
```
   c=IN IP4 233.252.0.1/127
```

“IP6” 多播不使用 TTL 范围，因此 “IP6” 多播不得 (MUST NOT) 存在 TTL 值。预计 IPv6 范围地址将用于限制多媒体会议的范围。

分层或分层编码方案是这样的数据流，其来自单个媒体源的编码被分成多个层。接收者可以通过仅订阅这些层的子集来选择所需的质量（以及因此的带宽）。这种分层编码通常在多个多播组中传输，以允许多播修剪。这种技术可以为仅需要特定层次结构级别的站点阻止不需要的流量。对于需要多个多播组的应用程序，我们允许将以下符号用于连接地址：
```
   <base multicast address>[/<ttl>]/<number of addresses>
```

如果没有给出地址数，则假定为一个。 如此分配的多播地址在基地址之上连续分配，例如：
```
   c=IN IP4 233.252.0.1/127/3
```

将声明地址 233.252.0.1、233.252.0.2 和 233.252.0.3 将与 TTL 127 一起使用。这在语义上等同于在媒体描述中包含多个“c=”行：
```
   c=IN IP4 233.252.0.1/127
   c=IN IP4 233.252.0.2/127
   c=IN IP4 233.252.0.3/127
```

同样，IPv6 示例是：
```
   c=IN IP6 ff00::db8:0:101/3
```

这在语义上等同于：
```
   c=IN IP6 ff00::db8:0:101
   c=IN IP6 ff00::db8:0:102
   c=IN IP6 ff00::db8:0:103
```

（请记住，“IP6”多播中不存在 TTL 子字段）。

只有当它们为分层或分层编码方案中的不同层提供多播地址时，才可以 (MAY) 在每个媒体描述的基础上指定多个地址或 “c=” 行。 不得 (MUST NOT ) 在会话级别指定多个地址或 “c=” 行。

上述多个地址的斜线符号不得 (MUST NOT) 用于 IP 单播地址。

## 5.8. 带宽信息 ("b=")
```
   b=<bwtype>:<bandwidth>
```

可选的 (OPTIONAL) “b=”行（带宽字段）表示会话或媒体描述要使用的建议带宽。 <bwtype> 是一个字母数字修饰符，它提供了 <bandwidth> 数字的含义。 本规范中定义了两个值，但将来可能会注册其它值（参见[第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana) 和 [[RFC3556](https://www.rfc-editor.org/rfc/rfc8866#RFC3556)]、[[RFC3890](https://www.rfc-editor.org/rfc/rfc8866#RFC3890)]）：

CT 如果会话的带宽与范围隐含的带宽不同，则应该 (SHOULD) 为会话提供 “b=CT:” 行，给出所用带宽的建议上限（“会议总”带宽）。类似地，如果 “m=” 行中的捆绑媒体流 [[RFC8843](https://www.rfc-editor.org/rfc/rfc8866#RFC8843)] 的带宽与范围隐含的带宽值不同，则应该 (SHOULD) 在媒体级别提供 “b=CT:” 行。这样做的主要目的是大致了解两个或多个会话（或捆绑的媒体流）是否可以同时共存。请注意，“b=CT:”行给出了所有端点上所有媒体的总带宽数字。

“b=CT:” 的 Mux 类别是 NORMAL。 这在 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] 中有讨论。

AS 该带宽的解释是特定于应用程序的（它将是应用程序的最大带宽的概念）。通常，这将与应用程序的 “最大带宽” 控件中设置的内容一致（如果适用）。对于基于 RTP 的应用程序，“b=AS:” 行给出了 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8866#RFC3550)] 的 [第 6.2 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.2) 中定义的 RTP “会话带宽”。请注意，“b=AS:”行给出了单个端点上单个媒体的带宽数字，尽管可能有许多端点在同时发送。

“b=AS:” 的 Mux 类别是 SUM。 这在 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] 中有讨论。

[[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 为 <bwtype> 名称定义了一个 “X-” 前缀。 这仅用于实验目的。 例如：
```
   b=X-YZ:128
```

不推荐 (NOT RECOMMENDED) 使用“X-”前缀。 相反，应该 (SHOULD) 定义新的（非“X-”前缀）<bwtype> 名称，然后必须 (MUST) 在标准名称空间中向 IANA 注册。SDP 解析器必须 (MUST) 忽略 <bwtype> 名称未知的带宽字段。 <bwtype> 名称必须是字母数字，尽管没有给出长度限制，但建议它们是 short。

<bandwidth> 默认被解释为每秒千比特（包括传输层和网络层，但不包括链路层开销）。新 <bwtype> 修饰符的定义可以指定带宽以某种替代单位来解释（本备忘录中定义的 “CT” 和 “AS” 修饰符使用默认单位）。

## 5.9. 活动时间 ("t=")
```
   t=<start-time> <stop-time>
```

“t=” 行（时间字段）开始一个时间描述，它指定会话的开始和停止时间。如果会话在多个不规则的时间段中处于活动状态，则可以 (MAY) 使用多个时间描述；每个额外的时间描述指定了会话将处于活动状态的额外时间段。如果会话在有规律的重复时间处于活动状态，则可以在时间字段之后包含以 “r=” 行（参见 [第 5.10 节](https://www.rfc-editor.org/rfc/rfc8866#repeattime)）开始的重复描述 —— 在这种情况下，时间字段指定整个重复序列的开始和停止时间。

以下示例指定了两个活动间隔：
```
   t=3724394400 3724398000 ; Mon 8-Jan-2018 10:00-11:00 UTC
   t=3724484400 3724488000 ; Tue 9-Jan-2018 11:00-12:00 UTC
```

时间字段的第一个和第二个子字段分别给出了会话的开始和停止时间。这些是自 1900 年 1 月 1 日 UTC 以来时间值的十进制表示，以秒为单位。 要将这些值转换为 Unix 时间 (UTC)，请减去十进制 2208988800。

一些时间表示将在 2036 年结束。由于 SDP 使用任意长度的十进制表示，所以它没有这个问题。 SDP 的实现需要准备好处理这些较大的值。

如果 <stop-time> 设置为零，则会话不受限制，但直到 <start-time> 之后才会变为活动状态。 如果 <start-time> 也为零，则认为会话是永久的。

用户接口应该 (SHOULD) 强烈反对创建无界和永久会话，因为它们没有提供有关会话实际何时终止的信息，因此使调度变得困难。

当向用户显示没有超时值的无界会话时，可以做出一般假设，即无界会话将仅在从当前时间或会话开始时间起半小时之前处于活动状态，以较晚者为准。如果需要除此之外的其它行为，则应该 (SHOULD) 给出一个 <stop-time>，并在有关会话何时真正结束的新信息可用时进行适当的修改。

永久会话可能会向用户显示为从不活动，除非有相关的重复时间准确地说明了会话何时将处于活动状态。

## 5.10. 重复时间 ("r=")
```
   r=<repeat interval> <active duration> <offsets from start-time>
```

“r=” 行（重复字段）指定会话的重复时间。 如果需要表达复杂的时间安排，可以包括多个重复字段。例如，如果会话在每周的周一上午 10 点和周二上午 11 点处于活动状态各一小时，持续三个月，则相应 “t=” 行中的 <start-time> 将表示第一个周一上午 10 点 ，<repeat interval> 为 1 周，<active duration> 为 1 小时，偏移量为 0 和 25 小时。相应的 “t=” 行的停止时间将代表三个月后的最后一个会话的结束。默认情况下，所有子字段都以秒为单位，因此 “r=” 和 “t=” 行可能如下：
```
   t=3724394400 3730536000 ; Mon 8-Jan-2018 10:00-11:00 UTC
                           ; Tues 20-Mar-2018 12:00 UTC
   r=604800 3600 0 90000   ; 1 week, 1 hour, zero, 25 hours
```

为了使描述更简洁，时间也可以以天、小时或分钟为单位给出。它们的语法是一个数字，后面紧跟一个区分大小写的字符。不允许使用小数单位—— 应该使用更小的单位代替。 允许使用以下单位规范字符：

| d | 天 (86400 秒) |
|-|-|
| h | 时 (3600 秒) |
| m | 分 (60 秒) |
| s | 秒 (仅为了完整性) |
*表 1：时间单位规范字符*

这样，上面的重复字段也可以写成这样：
```
   r=7d 1h 0 25h
```

不能用单个 SDP 重复时间直接指定每月和每年重复； 相反，应该使用单独的时间描述来明确列出会话时间。

## 5.11. 时区调整 ("z=")
```
   z=<adjustment time> <offset> <adjustment time> <offset> ....
```

“z=” 行（时区字段）是对其紧随其后的重复字段的可选修饰符。 它不适用于任何其它字段。

要安排从夏令时到标准时间或反之亦然的重复会话，有必要指定与基准时间的偏移量。这是必需的，因为不同的时区在一天中的不同时间更改时间，不同的国家/地区在不同的日期更改为夏令时或从夏令时更改，并且某些国家/地区根本没有夏令时。

因此，为了安排在冬季和夏季进行的同一时间的会话，必须能够明确地指定会话被安排在哪个时区。为了简化接收者的此项任务，我们允许发送者指定时区调整发生的时间（表示为自 1900 年 1 月 1 日 UTC 以来的秒数）以及与首次安排会话时间的偏移量。“z=” 行允许发送者指定这些调整时间的列表以及与基准时间的偏移量。

一个例子可能如下：
```
   t=3724394400 3754123200       ; Mon 8-Jan-2018 10:00 UTC
                                 ; Tues 18-Dec-2018 12:00 UTC
   r=604800 3600 0 90000         ; 1 week, 1 hour, zero, 25 hours
   z=3730928400 -1h 3749680800 0 ; Sun 25-Mar-2018 1:00 UTC,
                                 ; offset 1 hour,
                                 ; Sun 28-Oct-2018 2:00 UTC,
                                 ; no offset
```

这指定在时间 3730928400（2018 年 3 月 25 日星期日 1:00 UTC，英国夏令时开始），计算会话重复时间的时基向后移动 1 小时，并且在时间 3749680800（星期日 2018 年 10 月 28 日 2:00 UTC，英国夏令时结束）恢复会话的原始时基。调整总是相对于指定的开始时间——它们不是累积的。

如果一个会话可能持续数年，则期望会话描述将被定期修改，而不是在一个会话描述中传输数年的调整值。

## 5.12. 加密密钥 ("k=")
```
   k=<method>
   k=<method>:<encryption key>
```

“k=” 行（关键字段）已废弃，不得 (MUST NOT) 使用它。 在本文档中包含它出于遗留原因。 在 SDP 中不得 (MUST NOT) 包含 “k=” 行，如果在 SDP 中接收到，则必须 (MUST) 丢弃它。

## 5.13. 属性 ("a=")
```
   a=<attribute-name>
   a=<attribute-name>:<attribute-value>
```

属性是扩展 SDP 的主要方法。属性可以被定义为用作会话级属性、媒体级属性或两者兼而有之。（除了媒体级和会话级范围之外的属性范围，也可以在本文档的扩展中定义，例如 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8866#RFC5576)] 和 [[RFC8864](https://www.rfc-editor.org/rfc/rfc8866#RFC8864)]。）

媒体描述可以包含任意数量的特定于媒体描述的 “a=”行（属性字段）。 这些被称为媒体级属性并添加有关媒体描述的信息。也可以在第一个媒体描述之前添加属性字段；这些会话级属性传达了适用于整个会话而不是单个媒体描述的附加信息。

属性字段可以有两种形式：

 * 属性属性的形式是 “a=<attribute-name>”。这些是二进制属性，属性的存在表明该属性是会话的属性。一个例子可能是 “a=recvonly”。

 * 值属性的格式为 “a=<attribute-name>:<attribute-value>”。例如，白板可以具有值属性 “a=orient:landscape”。

属性的解释取决于被调用的媒体工具。因此，会话描述的接收者，在它们对会话描述的一般解释，特别是对属性的解释方面，应该是可配置的。

属性名称必须 (MUST) 使用 ISO-10646/UTF-8 的 US-ASCII 子集。

属性值是八位字节串，可以 (MAY) 使用除 0x00 (Nul)、0x0A (LF) 和 0x0D (CR) 之外的任何八位字节值。默认情况下，属性值将被解释为采用 UTF-8 编码的 ISO-10646 字符集。与其它文本字段不同，属性值通常不 (NOT) 受 “a=charset:” 属性的影响，因为这会使与已知值的比较出现问题。然而，当一个属性被定义时，它可以被定义为依赖于字符集，在这种情况下，它的值应该以会话字符集而不是 ISO-10646 来解释。

属性必须 (MUST) 在 IANA 注册（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。 如果接收到一个无法理解的属性，接收者必须 (MUST) 忽略它。

## 5.14. 媒体描述 ("m=")
```
   m=<media> <port> <proto> <fmt> ...
```

一个会话描述可以包含许多媒体描述。 每个媒体描述都以 “m=” 行（媒体字段）开始，并由下一个 “m=” 行或会话描述的结尾终止。 一个媒体字段有几个子字段：

<media> 是媒体类型。 本文档定义了值 “audio”、“video”、“text”、“application” 和 “message”。此列表由其它备忘录扩展，将来可能会通过注册媒体类型的其它备忘录进一步扩展（参见[第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）。例如，[[RFC6466](https://www.rfc-editor.org/rfc/rfc8866#RFC6466)] 定义了 “ image” 媒体类型。

<port> 是媒体流发送到的传输端口。传输端口的含义取决于相关 “c=” 行中指定的正在使用的网络，以及媒体字段的 <proto> 子字段中定义的传输协议。媒体应用程序使用的其它端口（例如 RTP 控制协议 (RTCP) 端口 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8866#RFC3550)]）可以 (MAY) 通过算法从基本媒体端口派生，或者可以 (MAY) 在单独的属性中指定（例如，“a=rtcp:” [[RFC3605](https://www.rfc-editor.org/rfc/rfc8866#RFC3605)]] 中定义的属性）。

如果使用非连续端口或者如果它们不遵循偶数 RTP 端口和奇数 RTCP 端口的奇偶规则，则必须 (MUST) 使用 “a=rtcp:” 属性。 被请求将媒体发送到奇数的 <port>，并且存在 “a=rtcp:” 属性的应用程序不得 (MUST NOT) 从 RTP 端口中减去 1：也就是说，它们必须将 RTP 发送到 <port> 指示的端口，并将 RTCP 发送到 “a=rtcp:” 属性中指示的端口。

对于将分层的编码流发送到单播地址的应用程序，可能需要指定多个传输端口。 这是通过在 “c=” 行中使用一个类似于用于 IP 多播地址的符号来实现的：
```
    m=<media> <port>/<number of ports> <proto> <fmt> ...
```

在这种情况下，使用的端口取决于传输协议。对于 RTP，默认只有偶数端口用于数据，对应的高一的奇数端口用于属于 RTP 会话的 RTCP，<number of ports> 表示 RTP 会话的个数。 例如：
```
       m=video 49170/2 RTP/AVP 31
```

将指定端口 49170 和 49171 形成一个 RTP/RTCP 对，而 49172 和 49173 形成第二个 RTP/RTCP 对。 RTP/AVP 是传输协议，31 是格式（见下面 <fmt> 的描述）。

本文档不包括使用非连续端口声明分层编码流的机制。（目前没有定义可以完成此操作的属性。[[RFC3605](https://www.rfc-editor.org/rfc/rfc8866#RFC3605)] 中定义的“ a=rtcp:” 属性不处理分层编码。）如果需要声明非连续端口，则有必要定义一个新属性来做这些。

如果在 “c=” 行中指定了多个地址，并且在 “m=” 行中指定了多个端口，则意味着从端口到相应地址的一对一映射。 例如：
```
       m=video 49170/2 RTP/AVP 31
       c=IN IP4 233.252.0.1/127/2
```

意味着地址 233.252.0.1 与端口 49170 和 49171 一起使用，地址 233.252.0.2 与端口 49172 和 49173 一起使用。

如果使用多个 “c=” 行指定多个地址，则映射类似。 例如：
```
       m=video 49170/2 RTP/AVP 31
       c=IN IP6 ff00::db8:0:101
       c=IN IP6 ff00::db8:0:102
```

意味着地址 ff00::db8:0:101 与端口 49170 和 49171 一起使用，地址 ff00::db8:0:102 与端口 49172 和 49173 一起使用。

将相同的媒体地址分配给多个媒体描述没有任何意义。这样做不会以任何方式隐含地对这些媒体描述进行分组。应该使用显式分组框架（例如，[[RFC5888](https://www.rfc-editor.org/rfc/rfc8866#RFC5888)]）来表达预期的语义。 例如，参见 [[RFC8843](https://www.rfc-editor.org/rfc/rfc8866#RFC8843)]。

<proto> 是传输协议。传输协议的含义取决于相关 “c=” 行中的地址类型子字段。 因此，地址类型为 “IP4” 的 “c=” 行表示传输协议在 IPv4 上运行。定义了以下传输协议，但可以通过向 IANA 注册新协议来扩展（参见 [第 8 节](https://www.rfc-editor.org/rfc/rfc8866#iana)）：

 * udp：表示数据直接通过 UDP 传输，而没有额外的分帧。
 * RTP/AVP：表示 [RTP](https://www.rfc-editor.org/rfc/rfc8866#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8866#RFC3550)] 在  [RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc8866#RFC3551) [[RFC3551](https://www.rfc-editor.org/rfc/rfc8866#RFC3551)] 下运行在 UDP 上。
 * RTP/SAVP：表示在 UDP 上运行的 [安全实时传输协议](https://www.rfc-editor.org/rfc/rfc8866#RFC3711) [[RFC3711](https://www.rfc-editor.org/rfc/rfc8866#RFC3711)]。
 * RTP/SAVPF：表示在 UDP 上运行的具有 [基于 RTCP 的反馈的扩展 SRTP 配置](https://www.rfc-editor.org/rfc/rfc8866#RFC5124) [[RFC5124](https://www.rfc-editor.org/rfc/rfc8866#RFC5124)]] 的 SRTP。

除媒体格式之外还要指定传输协议的主要原因是，即使网络协议相同，相同的标准媒体格式也可能承载在不同的传输协议上 —— 一个历史上的例子是 vat（MBone 的流行多媒体音频工具） 脉冲编码调制 (PCM) 音频和 RTP PCM 音频；另一个例子可能是 TCP/RTP PCM 音频。此外，传输协议特有但与格式无关的中继和监控工具也是可能的。

<fmt> 是媒体格式描述。第四个及之后的任何子字段描述媒体的格式。媒体格式的解释依赖于 <proto> 子字段的值。

如果 <proto> 子字段是 "RTP/AVP" 或 "RTP/SAVP"，则 <fmt> 子字段包含 RTP 有效负载类型编号。当给出有效负载类型编号列表时，这意味着所有这些有效负载格式都可以 (MAY) 在会话中使用，并且这些有效负载格式按优先顺序列出，列出的第一个格式是首选格式。当列出多种有效载荷格式时，从列表开始的第一个可接受的有效载荷格式应该用于会话。对于动态有效负载类型分配，“a=rtpmap:” 属性（参见[第 6.6 节](https://www.rfc-editor.org/rfc/rfc8866#rtpmap)）应该 (SHOULD) 用于将 RTP 有效载荷类型编号映射到标识有效载荷格式的媒体编码名称。“a=fmtp:” 属性可以 (MAY) 用来指定格式参数（见 [第 6.15 节](https://www.rfc-editor.org/rfc/rfc8866#fmtp)）。

如果 <proto> 子字段是 “udp”，则 <fmt> 子字段必须 (MUST) 引用 “audio”、“video”、“text”、“application” 或 “message” 顶级媒体类型下的描述格式的媒体类型。 媒体类型注册应该 (SHOULD) 定义与 UDP 传输一起使用的的数据包格式。

对于使用其它传输协议的媒体，<fmt> 子字段是特定于协议的。<fmt> 子字段的解释规则必须 (MUST) 在注册新协议时定义（见 [第 8.2.2 节](https://www.rfc-editor.org/rfc/rfc8866#MediaTypes)）。

[[RFC4855](https://www.rfc-editor.org/rfc/rfc8866#RFC4855)] 的 [第 3 节](https://www.rfc-editor.org/rfc/rfc4855#section-3) 指出，RTP 配置中定义的有效负载格式（编码）名称通常以大写形式显示，而媒体子类型名称通常以小写形式显示。它还指出，这两种名称在两个地方都不区分大小写，类似于在媒体类型字符串和到 SDP “a=fmtp:” 属性的默认映射中不区分大小写的参数名称。

# 6. SDP 属性

定义了以下属性。 由于应用程序编写者可以根据需要添加新属性，因此此列表并不详尽。新属性的注册程序在 [第 8.2.4 节]( https://www.rfc-editor.org/rfc/rfc8866#AttributeNames) 中定义。 语法是使用 ABNF [[RFC7405](https://www.rfc-editor.org/rfc/rfc8866#RFC7405)] 提供的，其中一些规则在 [第 9 节](https://www.rfc-editor.org/rfc/rfc8866#abnf) 中进一步定义。

## 6.1. cat （类别）

名称：cat
值：cat-value
使用级别：会话
字符集依赖：否
语法：
```
      cat-value = category
      category = non-ws-string
```

例子：
```
      a=cat:foo.bar
```

此属性给出会话的点分层次类别。 这是为了使接收者能够按类别过滤不需要的会话。没有类别的中心登记处。此属性已废弃，不应该 (SHOULD NOT) 使用。如果收到则应该 (SHOULD) 忽略它。

## 6.2. keywds（关键字）

名称：keywds
值：keywds-value
使用级别：会话
字符集依赖：是

语法：
```
      keywds-value = keywords
      keywords = text
```

例子：
```
      a=keywds:SDP session description protocol
```

与 “a=cat:” 属性一样，该属性旨在帮助识别接收者想要的会话，并允许接收者根据描述会话目的的关键字选择感兴趣的会话；但是，没有中心关键字注册表。如果指定了字符集，则应该以为会话描述指定的字符集来解释它的值，或者默认情况下以 ISO 10646/UTF-8 解释。此属性已废弃，不应该 (SHOULD NOT) 使用。如果收到则应该 (SHOULD) 忽略它。

## 6.3. tool

名称：tool
值：tool-value
使用级别：会话
字符集依赖：否

语法：
```
      tool-value = tool-name-and-version
      tool-name-and-version = text
```

例子：
```
      a=tool:foobar V3.2
```

这个属性给出了用于创建会话描述的工具的名称和版本号。

## 6.4. ptime（数据包时间）

名称：ptime
值：ptime-value
使用级别：媒体
字符集依赖：否

语法：
```
      ptime-value = non-zero-int-or-real
```

例子：
```
      a=ptime:20
```

这给出了由数据包中的媒体表示的时间长度（以毫秒为单位）。这可能只对音频数据有意义，但如果有意义，也可以与其它媒体类型一起使用。知道 “a=ptime:” 不是解码 RTP 或 vat 音频所必须的，它旨在作为音频编码/打包的建议。

## 6.5. maxptime（最大数据包时间）

名称：maxptime
值：maxptime-value
使用级别：媒体
字符集依赖：否

语法：
```
      maxptime-value = non-zero-int-or-real
```

例子：
```
      a=maxptime:20
```

它给出了每个数据包中可以封装的媒体的最大数量，以毫秒时间来表示。时间应 (SHALL) 计算为数据包中存在的媒体所代表的时间的总和。对于基于帧的编解码器，时间应该 (SHOULD) 是帧大小的整数倍。这个属性可能只对音频数据有意义，但如果有意义，也可以与其它媒体类型一起使用。请注意，此属性是在 [[RFC2327](https://www.rfc-editor.org/rfc/rfc8866#RFC2327)] 之后引入的，尚未更新的实现将忽略此属性。

## 6.6. rtpmap

名称：rtpmap
值：rtpmap-value
使用级别：媒体
字符集依赖：无

语法：
```
      rtpmap-value = payload-type SP encoding-name
        "/" clock-rate [ "/" encoding-params ]
      payload-type = zero-based-integer
      encoding-name = token
      clock-rate = integer
      encoding-params = channels
      channels = integer
```

此属性将 RTP 有效负载类型编号（如 “m=” 行中使用的）映射到表示要使用的有效负载格式的编码名称。它还提供有关时钟频率和编码参数的信息。请注意，有效负载类型编号由 7 位字段指示，将值限制在 0 到 127 之间。

尽管 RTP 配置文件可以将有效负载类型编号静态分配给有效负载格式，但更常见的是使用 “a=rtpmap:” 属性动态完成该分配。作为静态有效负载类型的示例，请考虑以 8 kHz 采样的 u-law PCM 编码的单通道音频。 这在 RTP 音频/视频配置文件中完全定义为有效载荷类型 0，因此不需要 “a=rtpmap:” 属性，发送到 UDP 端口 49232 的此类流的媒体可以指定为：
```
          m=audio 49232 RTP/AVP 0
```

动态有效负载类型的一个示例是，以 16 kHz 采样的 16 位线性编码立体声音频。如果我们希望为此流使用动态 RTP/AVP 负载类型 98，则需要附加信息来对其进行解码：
```
          m=audio 49232 RTP/AVP 98
          a=rtpmap:98 L16/16000/2
```

最多可以为每个指定的媒体格式定义一个 “a=rtpmap:” 属性。 因此，我们可能有以下内容：
```
          m=audio 49230 RTP/AVP 96 97 98
          a=rtpmap:96 L8/8000
          a=rtpmap:97 L16/8000
          a=rtpmap:98 L16/11025/2
```

指定使用动态有效负载类型的 RTP 配置文件必须 (MUST) 定义一组有效的编码名称和/或注册编码名称的方法，如果该配置文件要与 SDP 一起使用。“RTP/AVP” 和 “RTP/SAVP” 配置文件在 “m=” 行中表示的顶级媒体类型之下，使用媒体子类型来编码名称。在上面的例子中，媒体类型是 “audio/L8” 和 “audio/L16”。

对于音频流，编码参数表示音频的通道数。此参数是可选的 (OPTIONAL)，如果通道数为 1，则可以省略，前提是不需要其它参数。

对于视频流，目前没有指定编码参数。

将来可以 (MAY) 定义其它编码参数，但不应 (SHOULD NOT) 添加特定于编解码器的参数。 添加到 “a=rtpmap:” 属性的参数应该 (SHOULD) 只是会话目录选择合适的媒体参与会话所需的参数。应在其它属性中添加特定于编解码器的参数（例如，“a=fmtp:”）。

注意：RTP 音频格式通常不包括有关每个数据包的样本数的信息。如果需要非默认（如在 RTP 音频/视频配置文件 [[RFC3551](https://www.rfc-editor.org/rfc/rfc8866#RFC3551)] 中定义的）打包，则使用 [第 6.4 节](https://www.rfc-editor.org/rfc/rfc8866#ptime) 中给出的 “a=ptime:” 属性。


## 6.7. 媒体方向属性

“a=recvonly”、“a=sendrecv”、“a=sendonly” 或 “a=inactive” 最多可以 (MAY) 有一个出现在会话级别，并且每个媒体描述中最多可以 (MAY) 出现一个。

如果其中任何一项出现在媒体描述中，则它适用于该媒体描述。如果没有出现在媒体描述中，则会话级别的（如果有）适用于该媒体描述。

如果在会话级别或媒体级别均不存在任何媒体方向属性，则应假定默认值为 “a=sendrecv”。

在以下的 SDP 示例中，“a=sendrecv” 属性适用于第一个音频媒体，“a=inactive” 属性适用于其它的。
```
      v=0
      o=jdoe 3724395000 3724395001 IN IP6 2001:db8::1
      s=-
      c=IN IP6 2001:db8::1
      t=0 0
      a=inactive
      m=audio 49170 RTP/AVP 0
      a=sendrecv
      m=audio 49180 RTP/AVP 0
      m=video 51372 RTP/AVP 99
      a=rtpmap:99 h263-1998/90000
```

### 6.7.1. recvonly（仅接收）

名称：recvonly
值：
使用级别：会话，媒体
字符集依赖：否

例子：
```
      a=recvonly
```

这指定工具应在适用的情况下以仅接收模式启动。请注意，仅接收模式仅适用于媒体，不适用于任何关联的控制协议。一个只接收模式的基于 RTP 的系统必须仍然发送 RTCP 数据包，如 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8866#RFC3550)] [第 6 节](https://www.rfc-editor.org/rfc/rfc3550#section-6) 中所述。

### 6.7.2. sendrecv（收发）

名称：sendrecv
值：
使用级别：会话，媒体
字符集依赖：否

例子：
```
      a= sendrecv
```

这指定工具应在发送和接收模式下启动。 这对于默认使用仅接收模式的工具的交互式多媒体会议是必要的。

### 6.7.3. sendonly（仅发送）

名称：recvonly
值：
使用级别：会话，媒体
字符集依赖：否

例子：
```
      a= sendonly
```

这指定工具应以仅发送模式启动。一个例子可能是一个不同的单播地址将用于流量目的地而不是流量源。在这种情况下，可以使用两个媒体描述，一种是仅发送模式，一种是仅接收模式。请注意，仅发送模式仅适用于媒体，并且任何关联的控制协议（例如，RTCP）仍应正常接收和处理。

### 6.7.4. inactive

名称：inactive
值：
使用级别：会话，媒体
字符集依赖：否

例子：
```
      a= inactive
```

这指定工具应在非活动模式下启动。这对于用户可以暂停其它用户的交互式多媒体会议是必要的。不通过非活动媒体流发送任何媒体。请注意，即使在非活动模式下启动，基于 RTP 的系统仍必须 (MUST) 发送 RTCP（如果使用 RTCP）。

## 6.8. orient（方向）

名称：orient
值：orient-value
使用级别：媒体
字符集依赖：否

语法：
```
      orient-value = portrait / landscape / seascape
      portrait  = %s"portrait"
      landscape = %s"landscape"
      seascape  = %s"seascape"
         ; NOTE: These names are case-sensitive.
```

例子：
```
      a=orient:portrait
```

通常这仅用于白板或演示工具。它指定屏幕上工作区的方向。允许的值为 “portrait”、“landscape” 和 “seascape”（倒置景观）。

## 6.9. type（会议类型）

名称：type
值：type-value
使用级别：会话
字符集依赖：否

语法：
```
      type-value = conference-type
      conference-type = broadcast / meeting / moderated / test /
                        H332
      broadcast = %s"broadcast"
      meeting   = %s"meeting"
      moderated = %s"moderated"
      test      = %s"test"
      H332      = %s"H332"
         ; NOTE: These names are case-sensitive.
```

例子：
```
      a=type:moderated
```

它指定了多媒体会议的类型。允许的值为 "broadcast"、"meeting"、"moderated"、"test" 和 "H332"。 这些值对可能合适的其它选项有影响：

 * 当指定 “a=type:broadcast” 时，可能 “a=recvonly” 适合于那些连接。
 * 当指定 “a=type:meeting” 时，“a=sendrecv” 可能比较合适。
 * “a=type:moderated” 暗示使用发言权控制工具并启动媒体工具以使加入多媒体会议的新站点静音。
 * 指定 “a=type:H332” 表示该松耦合会话是 ITU H.332 规范 [[ITU.H332.1998](https://www.rfc-editor.org/rfc/rfc8866#ITU.H332.1998)] 中定义的 H.332 会话的一部分。媒体工具应以 “a=recvonly” 启动。
 * 建议指定 “a=type:test” 作为提示，除非另有明确要求，否则接收者可以安全地避免向用户显示此会话描述。


## 6.10. charset（字符集）

名称：charset
值：charset-value
使用级别：会话
字符集依赖：否

语法：
```
      charset-value = <defined in [RFC2978]>
```

这指定了用于显示会话名称和信息数据的字符集。默认情况下，使用 UTF-8 编码的 ISO-10646 字符集。如果需要更紧凑的表示，则可以使用其它字符集。例如，使用下面的 SDP 属性指定 ISO 8859-1：
```
      a=charset:ISO-8859-1
```

指定的字符集必须是在 IANA 字符集注册表 (http://www.iana.org/assignments/character-sets) 中注册的字符集之一，例如 ISO-8859-1。字符集标识符是一个字符串，必须 (MUST) 使用不区分大小写的比较，与注册表的 “名称” 或 “首选 MIME 名称” 字段中的标识符进行比较。如果标识符未被识别或不被支持，所有受其影响的字符串都应该 (SHOULD) 被视为八位字节串。

与字符集相关的字段必须仅包含根据所选字符集的定义有效的字节序列。 此外，与字符集相关的字段不得包含字节 0x00 (Nul)、0x0A (LF) 和 0x0d (CR)。

## 6.11. sdplang（SDP 语言）

名称：sdplang
值：sdplang-value
使用级别：会话，媒体
字符集依赖：否

语法：
```
      sdplang-value = Language-Tag
      ; Language-Tag defined in RFC 5646
```

例子：
```
      a=sdplang:fr
```

如果会话描述或媒体使用多种语言，则可以在会话或媒体级别提供多个 “a=sdplang:” 属性。

作为会话级属性，它指定会话描述的语言（而不是媒体的语言）。作为媒体级属性，它指定与该媒体关联的任何媒体级 SDP 信息字段的语言（同样不是媒体的语言），覆盖在会话级别指定的任何 “a=sdplang:” 属性。

通常，不鼓励发送包含多种语言的会话描述。相反，应该 (SHOULD) 发送多个会话描述来描述会话，每种语言一个。但是，这对于所有传输机制都是不可能的，因此允许多个 “a=sdplang:” 属性，尽管不推荐 (NOT RECOMMENDED) 。

“a=sdplang:” 属性值必须是单一语言标签 [[RFC5646](https://www.rfc-editor.org/rfc/rfc8866#RFC5646)]。当会话以足够的范围分发以跨越地理边界时，应指定 “a=sdplang：” 属性，其中无法假定接收者的语言，或者会话使用与本地假定规范不同的语言。

## 6.12. lang（语言）

名称：lang
值：lang-value
使用级别：会话，媒体
字符集依赖：否

语法：
```
      lang-value = Language-Tag
      ; Language-Tag defined in RFC 5646
```

例子：
```
      a=lang:de
```

如果会话或媒体具有不止一种语言的能力，则可以在会话或媒体级别提供多个 “a = lang：” 属性，在这种情况下，属性的顺序指示会话或媒体中各种语言的优先顺序，从最高优先级到最低优先级。

作为会话级属性，“a=lang:” 指定了正在描述的会话的语言能力。作为媒体级属性，它指定该媒体的语言能力，覆盖任何指定的会话级语言。

“a=lang:” 属性值必须是单个 [[RFC5646](https://www.rfc-editor.org/rfc/rfc8866#RFC5646)] 语言标记。当会话的范围足够跨越地理边界时，不能假定参与者的语言，或者会话具有与本地假定规范不同的语言能力时，应该 (SHOULD) 指定 “a=lang:” 属性。

“a=lang:”属性应该用于设置会话中使用的初始语言。会议期间的事件可能会影响所使用的语言，并且并不严格限制参与者只能使用声明的语言。

大多数实时用例从只使用一种语言开始，而其它用例涉及一系列语言，例如，解释的会话或字幕的会话。当指定了多个 “a=lang:” 属性时， “a=lang:” 属性本身不提供有关在会话期间打算使用的多种语言的任何信息，或者如果打算只选择一种语言。如果需要，可以定义一个新属性并将其用于指示此类意图。如果没有这种语义，则假定对于协商的会话，将选择并使用一种声明的语言。

## 6.13. framerate（帧率）

名称：framerate
值：framerate-value
使用级别：媒体
字符集依赖：无

语法：
```
      framerate-value = non-zero-int-or-real
```

例子：
```
      a=framerate:60
```

这给出了以帧/秒为单位的最大视频帧率。它旨在作为视频数据编码的建议。 小数值的十进制表示是允许的。它仅针对视频媒体定义。

## 6.14. quality

名称：quality
值：quality-value
使用级别：媒体
字符集依赖：无

语法：
```
      quality-value = zero-based-integer
```

例子：
```
      a=quality:10
```

它以整数值为编码质量提供了建议。视频质量属性的目的是指定帧率和静止图像质量之间的非默认折衷。对于视频，取值范围为 0 到 10，建议含义如下：

| 10 | 压缩方法可以给出的最好静态图像质量。 |
|--|--|
| 5 | 没有质量建议的默认行为。 |
| 0 | 编解码器设计师认为仍然可用的最差静止图像质量。 |
*表 2：编码质量值*

## 6.15. fmtp（格式参数）

名称：fmtp
值：fmtp-value
使用级别：媒体
字符集依赖：否

语法：
```
      fmtp-value = fmt SP format-specific-params
      format-specific-params = byte-string
        ; Notes:
        ; - The format parameters are media type parameters and
        ;   need to reflect their syntax.
```

例子：
```
      a=fmtp:96 profile-level-id=42e016;max-mbps=108000;max-fs=3600
```

此属性允许特定于特定格式的参数以 SDP 不必理解的方式传达。格式必须是为媒体指定的格式之一。格式特定的参数（分号分隔）可以是任何需要由 SDP 传送的参数集，并且未更改地提供给将使用此格式的媒体工具。每种格式最多允许该属性的一个实例。

“a=fmtp:” 属性可用于指定任何协议的参数和定义此类参数使用的格式。

[原文](https://www.rfc-editor.org/rfc/rfc8866)
