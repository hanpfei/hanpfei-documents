title: RFC8839 交互式连接建立 (ICE) 的会话描述协议 (SDP) 提议/应答过程
date: 2022-02-05 12:35:49
categories: 网络协议
tags:
- 网络协议
---

# 摘要

本文档描述了用于在代理之间执行交互式连接建立 (ICE) 的会话描述协议 (SDP) 提议/应答过程。

本文档以废止 RFC 5245 和 6336。

# 备忘录状态

这是一份互联网标准跟踪文件。

本文档是 IETF (Internet Engineering Task Force) 的产品。它代表了 IETF 社区的共识。它已接受公众审查，并已获互联网工程指导小组 (IESG) 批准出版。有关互联网标准的进一步信息，请参阅 [RFC 7841第2节](https://www.rfc-editor.org/rfc/rfc7841#section-2)。

有关此文档的当前状态、任何勘误表以及如何对其提供反馈的信息，可在以下网址获得 [https://www.rfc-editor.org/info/rfc8839](https://www.rfc-editor.org/info/rfc8839)。

# 版权声明

版权所有 (c) 2021 IETF 信托和确认为文档作者的人。保留所有权利。

本文档受 [BCP 78](https://www.rfc-editor.org/bcp/bcp78) 和自本文档发布之日起生效的 IETF 信托关于 IETF 文档 ([https://trustee.ietf.org/license-info](https://trustee.ietf.org/license-info)) 的法律规定 (https://trustee.ietf.org/license-info) 的约束。请仔细审阅这些文件，因为它们描述了您在此文档方面的权利和限制。从本文档中提取的代码组件必须包含信托法律规定的第 4.e 节中描述的简化 BSD 许可证文本，且如简化 BSD 许可证所述，不为这些代码提供任何担保。

本文档可能包含在 2008 年 11 月 10 日前发布或公开的来自 IETF 文档或 IETF 贡献的材料。控制这些材料的某些版权的人可能没有授予 IETF Trust 允许在 IETF 标准过程之外修改这些材料的权利。没有从控制这些材料的版权的人那里获得足够的许可，本文档不得在 IETF 标准过程之外被修改，也不得在 IETF 标准过程之外被创作出其衍生作品，除非为了出版将其格式化为 RFC 或将其翻译成英语以外的语言。

# 1. 简介

本文档描述了交互式连接建立 (ICE) 如何与会话描述协议 (SDP) 提议/应答 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 一起使用。ICE 规范 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 描述了对 ICE 的所有应用通用的过程，而本文档提供了将 ICE 与 SDP 提议/应答一起使用所需的额外细节。

本文档已废弃 RFC 5245 和 6336。

注意：以前通用的 ICE 过程和 SDP 提议/应答的具体细节都在 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)] 中进行了描述。[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 已废弃 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)]，并且从文档中删除了 SDP 提议/应答特有的细节信息。[第 11 节](https://www.rfc-editor.org/rfc/rfc8839#sec.5245) 描述了本文档中详述的 SDP 提议/应答特有的细节信息的更改。

# 2. 约定

本文档中的关键词 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY” 和 “OPTIONAL” 应按照 [BCP 14](https://www.rfc-editor.org/bcp/bcp14) [[RFC2119](https://www.rfc-editor.org/rfc/rfc2119)] [[RFC8174](https://www.rfc-editor.org/rfc/rfc8174)] 的描述解释，当且仅当它们以大写字母出现时，如此处所示。

# 3. 术语

读者应该熟悉 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)]、[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中定义的术语以及以下内容：


 * 
* *
""
[]()



# 10. IANA 注意事项

## 10.1. SDP 属性

最初的 ICE 规范根据 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)] 的 [第 8.2.4 节](https://www.rfc-editor.org/rfc/rfc4566#section-8.2.4) 的程序定义了七个新的 SDP 属性。此处包含来自原始规范的注册信息，并进行了修改以包含 Mux 类别 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8839#RFC8859)]，并且还定义了一个新的 SDP 属性 “ice-pacing”。

### 10.1.1. "candidate" 属性

属性名称：candidate
属性类型：媒体级 media-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并提供许多可能的通信候选地址之一。这些地址通过使用 NAT 会话遍历实用程序 (STUN) 的端到端连接检查进行验证。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：TRANSPORT

### 10.1.2. "remote-candidates" 属性

属性名称：remote-candidates
属性类型：媒体级 media-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并提供提议者希望应答者在其应答中使用的远程候选地址的标识。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：TRANSPORT

### 10.1.3. "ice-lite" 属性

属性名称：ice-lite
属性类型：会话级 session-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并表示代理具有支持 ICE 与具有完整实现的对等方互操作所需的最低功能。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：NORMAL

### 10.1.4. "ice-mismatch" 属性

属性名称：ice-mismatch
属性类型：媒体级 media-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并表示代理具有 ICE 能力，但由于候选地址与 SDP 中发出的媒体的默认目的地不匹配，而没有继续进行 ICE。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：NORMAL

### 10.1.5. "ice-pwd" 属性

属性名称：ice-pwd
属性类型：会话或媒体级 session- or media-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并提供用于保护 STUN 连接检查的密码。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：TRANSPORT

### 10.1.6. "ice-ufrag" 属性

属性名称：ice-ufrag
属性类型：会话或媒体级 session- or media-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并提供用于构造 STUN 连接检查中的用户名的片段。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：TRANSPORT

### 10.1.7. "ice-options" 属性

属性名称：ice-options
长形式：ice-options
属性类型：会话级 session-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，并指示代理所使用的 ICE 选项或扩展。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：NORMAL

### 10.1.8. "ice-pacing" 属性

本规范还定义了一个新的 SDP 属性，"ice-pacing"，根据如下的数据：

属性名称：ice-pacing
属性类型：会话级 session-level
受制于字符集：否
目的：此属性与交互式连接建立 (ICE) 一起使用，以指示所需要的连接检查 pacing 值。

适当的值：参见 RFC 8839 的 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar)。
联系人姓名：IESG
联系电子邮件：iesg@ietf.org
参考引用：RFC 8839
Mux 类别哦：NORMAL

## 10.2. 交互式连接建立 (ICE) 选项注册表

IANA 根据 “在 RFC 中编写 IANA 注意事项的指南” [[RFC8126](https://www.rfc-editor.org/rfc/rfc8839#RFC8126)] 中定义的 规范要求 政策维护 “ice-options” 标识符的注册表。根据 [第 5.6 节](https://www.rfc-editor.org/rfc/rfc8839#sec-ice-options) 的语法，ICE 选项的长度不受限制；但是，建议 (RECOMMENDED) 它们不超过 20 个字符。这是为了减少消息大小并允许高效的解析。 ICE 选项在会话级别定义。

注册请求必须 (MUST) 包含以下信息：

 * 要注册的 ICE 选项标识符
 * 具有变更控制权的组织或个人的姓名和电子邮件地址
 * 选项相关的 ICE 扩展的简短描述
 * 对定义 ICE 选项和相关扩展的规范的参考引用

## 10.3. 候选地址属性扩展子注册建立

本节在 SDP 参数注册表下创建一个新的子注册表 “候选地址属性扩展”：http://www.iana.org/assignments/sdp-parameters。

子注册表的目的是注册 SDP “candidate” 属性扩展。

当在子注册表中注册 “candidate” 扩展时，它需要满足 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8839#RFC8126)] 中定义的 “规范要求” 政策。

“candidate” 属性扩展必须 (MUST) 遵循 “cand-extension” 语法。 属性扩展名称必须 (MUST) 遵循 'extension-att-name' 语法，属性扩展值必须 (MUST) 遵循 'extension-att-value' 语法。

注册请求必须 (MUST) 包含以下信息：

 * 属性扩展的名称
 * 具有变更控制权的组织或个人的姓名和电子邮件地址
 * 属性扩展的简短描述
 * 对描述属性扩展的语义、用法和可能值的规范的参考引用。

# 11. 自 RFC 5245 以来的变更

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 描述了对通用 SIP 程序所做的更改，包括取消积极提名、修改计算候选地址对状态的程序、调度连接检查以及计时器值的计算。

本文档定义了以下 SDP 提议/应答特有的更改：

 * SDP offer/answer 实现和'ice2' 选项的使用。
 * SDP “ice-pacing” 属性的定义和使用。
 * ICE 代理不得生成具有 FQDN 的候选地址的显式文本，并且如果从对等代理处接收到此类候选地址，则必须丢弃此类候选地址。
 * 放宽包含 SDP “rtcp” 属性的要求。
 * SDP 提议/应答程序的一般说明。
 * ICE 不匹配现在是可选的，并且代理可以选择不触发不匹配，而是将默认候选地址视为附加候选地址。
 * FQDN 和 “0.0.0.0” / “::” IP 地址与端口 “9” 默认候选地址不会触发 ICE 不匹配。

# 12. 参考 (References)

## 12.1. 规范性参考 (Normative References)

```
   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <https://www.rfc-editor.org/info/rfc2119>.

   [RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston,
              A., Peterson, J., Sparks, R., Handley, M., and E.
              Schooler, "SIP: Session Initiation Protocol", RFC 3261,
              DOI 10.17487/RFC3261, June 2002,
              <https://www.rfc-editor.org/info/rfc3261>.

   [RFC3262]  Rosenberg, J. and H. Schulzrinne, "Reliability of
              Provisional Responses in Session Initiation Protocol
              (SIP)", RFC 3262, DOI 10.17487/RFC3262, June 2002,
              <https://www.rfc-editor.org/info/rfc3262>.

   [RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model
              with Session Description Protocol (SDP)", RFC 3264,
              DOI 10.17487/RFC3264, June 2002,
              <https://www.rfc-editor.org/info/rfc3264>.

   [RFC3312]  Camarillo, G., Ed., Marshall, W., Ed., and J. Rosenberg,
              "Integration of Resource Management and Session Initiation
              Protocol (SIP)", RFC 3312, DOI 10.17487/RFC3312, October
              2002, <https://www.rfc-editor.org/info/rfc3312>.

   [RFC3556]  Casner, S., "Session Description Protocol (SDP) Bandwidth
              Modifiers for RTP Control Protocol (RTCP) Bandwidth",
              RFC 3556, DOI 10.17487/RFC3556, July 2003,
              <https://www.rfc-editor.org/info/rfc3556>.

   [RFC3605]  Huitema, C., "Real Time Control Protocol (RTCP) attribute
              in Session Description Protocol (SDP)", RFC 3605,
              DOI 10.17487/RFC3605, October 2003,
              <https://www.rfc-editor.org/info/rfc3605>.

   [RFC4032]  Camarillo, G. and P. Kyzivat, "Update to the Session
              Initiation Protocol (SIP) Preconditions Framework",
              RFC 4032, DOI 10.17487/RFC4032, March 2005,
              <https://www.rfc-editor.org/info/rfc4032>.

   [RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session
              Description Protocol", RFC 4566, DOI 10.17487/RFC4566,
              July 2006, <https://www.rfc-editor.org/info/rfc4566>.

   [RFC5234]  Crocker, D., Ed. and P. Overell, "Augmented BNF for Syntax
              Specifications: ABNF", STD 68, RFC 5234,
              DOI 10.17487/RFC5234, January 2008,
              <https://www.rfc-editor.org/info/rfc5234>.

   [RFC5389]  Rosenberg, J., Mahy, R., Matthews, P., and D. Wing,
              "Session Traversal Utilities for NAT (STUN)", RFC 5389,
              DOI 10.17487/RFC5389, October 2008,
              <https://www.rfc-editor.org/info/rfc5389>.

   [RFC5766]  Mahy, R., Matthews, P., and J. Rosenberg, "Traversal Using
              Relays around NAT (TURN): Relay Extensions to Session
              Traversal Utilities for NAT (STUN)", RFC 5766,
              DOI 10.17487/RFC5766, April 2010,
              <https://www.rfc-editor.org/info/rfc5766>.

   [RFC5768]  Rosenberg, J., "Indicating Support for Interactive
              Connectivity Establishment (ICE) in the Session Initiation
              Protocol (SIP)", RFC 5768, DOI 10.17487/RFC5768, April
              2010, <https://www.rfc-editor.org/info/rfc5768>.

   [RFC6336]  Westerlund, M. and C. Perkins, "IANA Registry for
              Interactive Connectivity Establishment (ICE) Options",
              RFC 6336, DOI 10.17487/RFC6336, July 2011,
              <https://www.rfc-editor.org/info/rfc6336>.

   [RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
              2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
              May 2017, <https://www.rfc-editor.org/info/rfc8174>.

   [RFC8445]  Keranen, A., Holmberg, C., and J. Rosenberg, "Interactive
              Connectivity Establishment (ICE): A Protocol for Network
              Address Translator (NAT) Traversal", RFC 8445,
              DOI 10.17487/RFC8445, July 2018,
              <https://www.rfc-editor.org/info/rfc8445>.
```

## 12.2. 参考资料 (Informative References)

```
   [RFC3725]  Rosenberg, J., Peterson, J., Schulzrinne, H., and G.
              Camarillo, "Best Current Practices for Third Party Call
              Control (3pcc) in the Session Initiation Protocol (SIP)",
              BCP 85, RFC 3725, DOI 10.17487/RFC3725, April 2004,
              <https://www.rfc-editor.org/info/rfc3725>.

   [RFC3960]  Camarillo, G. and H. Schulzrinne, "Early Media and Ringing
              Tone Generation in the Session Initiation Protocol (SIP)",
              RFC 3960, DOI 10.17487/RFC3960, December 2004,
              <https://www.rfc-editor.org/info/rfc3960>.

   [RFC5245]  Rosenberg, J., "Interactive Connectivity Establishment
              (ICE): A Protocol for Network Address Translator (NAT)
              Traversal for Offer/Answer Protocols", RFC 5245,
              DOI 10.17487/RFC5245, April 2010,
              <https://www.rfc-editor.org/info/rfc5245>.

   [RFC5626]  Jennings, C., Ed., Mahy, R., Ed., and F. Audet, Ed.,
              "Managing Client-Initiated Connections in the Session
              Initiation Protocol (SIP)", RFC 5626,
              DOI 10.17487/RFC5626, October 2009,
              <https://www.rfc-editor.org/info/rfc5626>.

   [RFC5898]  Andreasen, F., Camarillo, G., Oran, D., and D. Wing,
              "Connectivity Preconditions for Session Description
              Protocol (SDP) Media Streams", RFC 5898,
              DOI 10.17487/RFC5898, July 2010,
              <https://www.rfc-editor.org/info/rfc5898>.

   [RFC6679]  Westerlund, M., Johansson, I., Perkins, C., O'Hanlon, P.,
              and K. Carlberg, "Explicit Congestion Notification (ECN)
              for RTP over UDP", RFC 6679, DOI 10.17487/RFC6679, August
              2012, <https://www.rfc-editor.org/info/rfc6679>.

   [RFC7675]  Perumal, M., Wing, D., Ravindranath, R., Reddy, T., and M.
              Thomson, "Session Traversal Utilities for NAT (STUN) Usage
              for Consent Freshness", RFC 7675, DOI 10.17487/RFC7675,
              October 2015, <https://www.rfc-editor.org/info/rfc7675>.

   [RFC8126]  Cotton, M., Leiba, B., and T. Narten, "Guidelines for
              Writing an IANA Considerations Section in RFCs", BCP 26,
              RFC 8126, DOI 10.17487/RFC8126, June 2017,
              <https://www.rfc-editor.org/info/rfc8126>.

   [RFC8859]  Nandakumar, S., "A Framework for Session Description
              Protocol (SDP) Attributes When Multiplexing", RFC 8859,
              DOI 10.17487/RFC8859, January 2021,
              <https://www.rfc-editor.org/info/rfc8859>.

   [RFC8863]  Holmberg, C. and J. Uberti, "Interactive Connectivity
              Establishment Patiently Awaiting Connectivity (ICE PAC)",
              RFC 8863, DOI 10.17487/RFC8863, January 2021,
              <https://www.rfc-editor.org/info/rfc8863>.
```

# 附录 A. 示例

对于 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] [第 15 节](https://www.rfc-editor.org/rfc/rfc8445#section-15) 中显示的示例，以 SDP 编码的最终的 offer（消息 5）看起来像（为清晰起见折叠行）：
```
v=0
o=jdoe 2890844526 2890842807 IN IP6 $L-PRIV-1.IP
s=
c=IN IP6 $NAT-PUB-1.IP
t=0 0
a=ice-options:ice2
a=ice-pacing:50
a=ice-pwd:asd88fgpdd777uzjYhagZg
a=ice-ufrag:8hhY
m=audio $NAT-PUB-1.PORT RTP/AVP 0
b=RS:0
b=RR:0
a=rtpmap:0 PCMU/8000
a=candidate:1 1 UDP 2130706431 $L-PRIV-1.IP $L-PRIV-1.PORT typ host
a=candidate:2 1 UDP 1694498815 $NAT-PUB-1.IP $NAT-PUB-1.PORT typ
 srflx raddr $L-PRIV-1.IP rport $L-PRIV-1.PORT
```

该 offer，以它们的值替换变量，将看起来像（为清晰起见折叠行）：
```
v=0
o=jdoe 2890844526 2890842807 IN IP6 fe80::6676:baff:fe9c:ee4a
s=
c=IN IP6 2001:db8:8101:3a55:4858:a2a9:22ff:99b9
t=0 0
a=ice-options:ice2
a=ice-pacing:50
a=ice-pwd:asd88fgpdd777uzjYhagZg
a=ice-ufrag:8hhY
m=audio 45664 RTP/AVP 0
b=RS:0
b=RR:0
a=rtpmap:0 PCMU/8000
a=candidate:1 1 UDP 2130706431 fe80::6676:baff:fe9c:ee4a 8998
 typ host
a=candidate:2 1 UDP 1694498815 2001:db8:8101:3a55:4858:a2a9:22ff:99b9
 45664 typ srflx raddr fe80::6676:baff:fe9c:ee4a rport 8998
```

得到的 answer 如下所示：
```
v=0
o=bob 2808844564 2808844564 IN IP4 $R-PUB-1.IP
s=
c=IN IP4 $R-PUB-1.IP
t=0 0
a=ice-options:ice2
a=ice-pacing:50
a=ice-pwd:YH75Fviy6338Vbrhrlp8Yh
a=ice-ufrag:9uB6
m=audio $R-PUB-1.PORT RTP/AVP 0
b=RS:0
b=RR:0
a=rtpmap:0 PCMU/8000
a=candidate:1 1 UDP 2130706431 $R-PUB-1.IP $R-PUB-1.PORT typ host
```

填入变量之后：
```
v=0
o=bob 2808844564 2808844564 IN IP4 192.0.2.1
s=
c=IN IP4 192.0.2.1
t=0 0
a=ice-options:ice2
a=ice-pacing:50
a=ice-pwd:YH75Fviy6338Vbrhrlp8Yh
a=ice-ufrag:9uB6
m=audio 3478 RTP/AVP 0
b=RS:0
b=RR:0
a=rtpmap:0 PCMU/8000
a=candidate:1 1 UDP 2130706431 192.0.2.1 3478 typ host
```

# 附录 B. "remote-candidates" 属性

“remote-candidates” 属性的存在是为了消除更新的提议，和对将候选地址移动到有效列表中的 STUN 绑定请求的响应之间的竞争条件。这种竞态条件如图 1 所示。收到消息 4 后，代理 L 将候选地址对添加到有效列表中。 如果只有一个包含单个组件的数据流，代理 L 现在可以发送更新的提议。但是，代理 R 的检查尚未收到响应，代理 R 在收到响应（消息 9）之前收到更新的提议（消息 7）。 因此，它还不知道这个特定的对是有效对。为了消除这种情况，由提议者选择的 R 的实际候选地址（远程候选地址）包含在提议本身中，并且应答者延迟其应答，直到这些对验证。

```
Agent L               Network               Agent R
   |(1) Offer            |                     |
   |------------------------------------------>|
   |(2) Answer           |                     |
   |<------------------------------------------|
   |(3) STUN Req.        |                     |
   |------------------------------------------>|
   |(4) STUN Res.        |                     |
   |<------------------------------------------|
   |(5) STUN Req.        |                     |
   |<------------------------------------------|
   |(6) STUN Res.        |                     |
   |-------------------->|                     |
   |                     |Lost                 |
   |(7) Offer            |                     |
   |------------------------------------------>|
   |(8) STUN Req.        |                     |
   |<------------------------------------------|
   |(9) STUN Res.        |                     |
   |------------------------------------------>|
   |(10) Answer          |                     |
   |<------------------------------------------|
```
图 1：竞态条件流程

# 附录 C. 为什么需要冲突解决机制？

当 ICE 在两个对等体之间运行时，一个代理充当受控者角色，另一个充当控制者角色。 规则被定义为实现类型和提议者/应答者的函数，以确定谁在控制和谁被控制。但是，规范中提到，在某些情况下，双方可能都认为自己在控制，或者双方都认为自己受到控制。 这怎么可能发生？

两个代理都认为自己受到控制的情况，出现在第三方呼叫控制案例中。 考虑以下流程：
```
          A         Controller          B
          |(1) INV()     |              |
          |<-------------|              |
          |(2) 200(SDP1) |              |
          |------------->|              |
          |              |(3) INV()     |
          |              |------------->|
          |              |(4) 200(SDP2) |
          |              |<-------------|
          |(5) ACK(SDP2) |              |
          |<-------------|              |
          |              |(6) ACK(SDP1) |
          |              |------------->|
```
*图 2：角色冲突流程*

该流程是 RFC 3725 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] 流程 III 的变体。 事实上，它比流 III 工作得更好，因为它产生的消息更少。在这个流程中，控制器向代理 A 发送一个无提议的邀请 INVITE，代理 A 以它的提议 SDP1 进行响应。然后，该代理向代理 B 发送无提议邀请 INVITE，代理 B 以它的提议 SDP2 响应该邀请 INVITE。然后控制器使用来自每个代理的提议生成应答。使用此流程时，ICE 将在代理 A 和 B 之间运行，但双方都认为自己处于控制角色。通过角色冲突解决程序，当使用 ICE 时，此流程将正常运行。

目前，没有有记录的流程可能导致两个代理都认为自己受到控制的情况。但是，冲突解决程序允许这种情况，如果出现适合此类别的流程。

# 附录 D. 为什么要发送更新的 Offer？

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 12.1 节](https://www.rfc-editor.org/rfc/rfc8839#RFC8445) 描述了发送媒体的规则。 一旦 ICE 检查完成，两个代理都可以发送媒体，而无需等待更新的 Offer。实际上，更新的 Offer 的唯一目的是 “更正” SDP，以便媒体的默认目的地与根据 ICE 程序发送媒体的目的地相匹配（这将是最高优先级的提名候选地址对）。

这就提出了一个问题——为什么需要更新的 offer/answer 交换？事实上，在纯粹的提议/应答环境中，它不需要。提议者和应答者通过 ICE 就使用的候选地址达成一致意见，然后可以开始使用它们。就代理本身而言，更新的提议/应答没有提供新信息。然而，在实践中，信令路径上的许多组件都会查看 SDP 信息。其中包括执行路径外 QoS 保留的实体、NAT 穿越组件（例如 ALG 和会话边界控制器 (SBC)）以及被动监控网络的诊断工具。为了让这些工具在不改变的情况下继续运行，SDP 的核心属性 —— 即现有的、ICE 之前用于媒体的地址定义 —— “m=” 和 “c=” 行以及 “rtcp” 属性 —— 必须保留。因此，必须发送更新的 offer。

# 致谢

本文档中的大部分文本来自 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)]，由 Jonathan Rosenberg 撰写。

本文档中的一些文本摘自 [[RFC6336](https://www.rfc-editor.org/rfc/rfc8839#RFC6336)]，由 Magnus Westerlund 和 Colin Perkins 撰写。

非常感谢 Flemming Andreasen 的牧羊人评论反馈。

感谢以下专家的评论和建设性反馈：Thomas Stach、Adam Roach、Peter Saint-Andre、Roman Danyliw、Alissa Cooper、Benjamin Kaduk、Mirja Kühlewind、Alexey Melnikov、和 Éric Vyncke，为他们详细的评论。 

# 贡献者

以下专家为这项工作做出了文本和结构方面的改进：

**Thomas Stach**
Email: thomass.stach@gmail.com

# 作者的地址

**Marc Petit-Huguenin**
Impedance Mismatch
Email: marc@petit-huguenin.org

**Suhas Nandakumar**
Cisco Systems
707 Tasman Dr
Milpitas, CA 95035
United States of America
Email: snandaku@cisco.com

**Christer Holmberg**
Ericsson
Hirsalantie 11
FI-02420 Jorvas
Finland
Email: christer.holmberg@ericsson.com

**Ari Keränen**
Ericsson
FI-02420 Jorvas
Finland
Email: ari.keranen@ericsson.com

**Roman Shpount**
TurboBridge
4905 Del Ray Avenue, Suite 300
Bethesda, MD 20814
United States of America
Email: rshpount@turbobridge.com

[原文](https://www.rfc-editor.org/rfc/rfc8839)
