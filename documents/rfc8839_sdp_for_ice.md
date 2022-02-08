---
title: RFC8839 交互式连接建立 (ICE) 的会话描述协议 (SDP) 要约/应答过程
date: 2022-02-05 12:35:49
categories: 网络协议
tags:
- 网络协议
---

# 摘要

本文档描述了用于在代理之间执行交互式连接建立 (ICE) 的会话描述协议 (SDP) 要约/应答过程。

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

本文档描述了交互式连接建立 (ICE) 如何与会话描述协议 (SDP) 要约/应答 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 一起使用。ICE 规范 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 描述了对 ICE 的所有应用通用的过程，而本文档提供了将 ICE 与 SDP 要约/应答一起使用所需的额外细节。

本文档已废弃 RFC 5245 和 6336。

注意：以前通用的 ICE 过程和 SDP 要约/应答的具体细节都在 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)] 中进行了描述。[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 已废弃 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)]，并且从文档中删除了 SDP 要约/应答特有的细节信息。[第 11 节](https://www.rfc-editor.org/rfc/rfc8839#sec.5245) 描述了本文档中详述的 SDP 要约/应答特有的细节信息的更改。

# 2. 约定

本文档中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"NOT RECOMMENDED"、"MAY" 和 "OPTIONAL" 应按照 [BCP 14](https://www.rfc-editor.org/bcp/bcp14) [[RFC2119](https://www.rfc-editor.org/rfc/rfc2119)] [[RFC8174](https://www.rfc-editor.org/rfc/rfc8174)] 的描述解释，当且仅当它们以大写字母出现时，如此处所示。

# 3. 术语

读者应该熟悉 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)]、[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中定义的术语以及以下内容：

默认目的地址/候选地址：数据流的组件的默认目的地址是不知道 ICE 的代理将使用的传输地址。组件的默认候选地址是其传输地址与该组件的默认目的地址相匹配的候选地址。对于 RTP 组件，默认连接地址在 SDP 的 "c=" 行，端口和传输协议在 "m=" 行。对于 RTP 控制协议 (RTCP) 组件，如果存在，则使用 [[RFC3605](https://www.rfc-editor.org/rfc/rfc8839#RFC3605)] 中定义的 "rtcp" 属性指示地址和端口； 否则，RTCP 组件地址与 RTP 组件地址相同，其端口比 RTP 组件端口大一。

# 4. SDP 要约/应答 过程

## 4.1. 简介

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 将 ICE 候选地址交换定义为 ICE 代理（发起者和响应者）交换其在代理处进行 ICE 处理所需的候选地址信息的过程。就本规范而言，候选地址交换过程对应于 Offer/Answer 协议 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)]，术语 "offerer" 和 "answerer" 分别对应于 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中的发起者和响应者角色。

一旦发起代理收集、修剪并确定了其候选地址集的优先级 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]，与对等代理的候选地址交换就开始了。

## 4.2. 一般程序

### 4.2.1. 编码

[第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar) 提供了构建本规范中定义的各种 SDP 属性的详细规则。

#### 4.2.1.1. 数据流

每个数据流 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 由一个 SDP 媒体描述 ("m=" 部分) 表示。

#### 4.2.1.2. 候选地址

在 "m=" 部分中，与数据流相关联的每个候选地址（包括默认候选地址）由 SDP "candidate" 属性表示。

在提名之前，与 "m=" 部分关联的 "c=" 行包含默认候选地址的连接地址，而 "m=" 行包含该 "m=" 部分的默认候选地址的端口和传输协议。

提名后，给定 "m=" 部分的 "c=" 行包含提名的候选地址（提名的候选地址对的本地候选地址）的连接地址，"m=" 行包含与该 "m=" 部分的提名候选地址对应的端口和传输协议。

#### 4.2.1.3. 用户名和密码

ICE 用户名由 SDP "ice-ufrag" 属性表示，ICE 密码由 SDP "ice-pwd" 属性表示。

#### 4.2.1.4. 精简实现

ICE 精简实现 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 必须 (MUST) 包含 SDP "ice-lite" 属性。 完整实现不得 (MUST NOT) 包含该属性。

#### 4.2.1.5. ICE 扩展

代理使用 SDP "ice-options" 属性来指示对 ICE 扩展的支持。

符合本规范的代理必须 (MUST) 包含一个带有 "ice2" 属性值的 SDP "ice-options" 属性 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]。 如果代理收到表明 ICE 支持的 SDP 要约或应答，但不包含具有 "ice2" 属性值的 SDP "ice-options" 属性，则代理可以假设对等方符合 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)]。

#### 4.2.1.6. 非活动和禁用的数据流

如果 "m=" 部分被标记为非活动 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)]，或者带宽值为零[[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)]，代理必须 (MUST) 仍然包含与 ICE 相关的 SDP 属性。

如果按照 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的 [第 8.2 节](https://www.rfc-editor.org/rfc/rfc3264#section-8.2) 中的定义，与 "m=" 部分关联的端口值设置为零（暗示禁用流），则代理不应 (SHOULD NOT) 在该 "m=" 中包含 ICE 相关的 SDP "candidate" 属性，除非使用了另外指定的 SDP 扩展。

### 4.2.2. RTP/RTCP 注意事项

如果代理同时使用 RTP 和 RTCP，并且为 RTP 和 RTCP 使用了单独的端口，则代理必须 (MUST) 包括 RTP 和 RTCP 组件的 SDP "candidate" 属性。

代理遵循 [[RFC3605](https://www.rfc-editor.org/rfc/rfc8839#RFC3605)] 中的过程，包括一个 SDP "rtcp" 属性。 因此，在 RTCP 端口值比 RTP 端口值大 1，并且 RTCP 组件地址与 RTP 组件地址相同的情况下，可以省略 SDP "rtcp" 属性。 

注意：[[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)] 要求代理始终包含 SDP "rtcp" 属性，即使 RTCP 端口值比 RTP 端口值高 1。本规范将 "rtcp" 属性过程与 [[RFC3605](https://www.rfc-editor.org/rfc/rfc8839#RFC3605)] 保持一致。

如果代理不使用 RTCP，它通过包含 [[RFC3556](https://www.rfc-editor.org/rfc/rfc8839#RFC3556)] 中描述的 "RS:0" 和 "RR:0" SDP 属性来表示。

### 4.2.3. 确定角色

要约者充当发起人角色。应答者充当响应代理角色。ICE 角色（控制和受控）是使用 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中的程序确定的。

### 4.2.4. STUN 注意事项

一旦代理在 SDP 要约或应答中向其对等方提供了本地候选地址，代理必须 (MUST) 准备好在那些候选地址上接收 STUN（NAT 会话穿透实用程序，[[RFC5389](https://www.rfc-editor.org/rfc/rfc8839#RFC5389)]）连接检查绑定请求。

### 4.2.5. 验证 ICE 支持的过程

ICE 代理通过在要约或应答中至少包含 SDP "ice-pwd" 和 "ice-ufrag" 属性来表示对 ICE 的支持。符合本规范的 ICE 代理还必须 (MUST) 包含一个带有 "ice2" 属性值的 SDP "ice-options" 属性。

如果对于它收到的 SDP 中的每个数据流，该数据流的每个组件的默认目的地址出现在 "candidate" 属性中，则代理将继续执行 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 和本规范中定义的 ICE 过程。例如，在 RTP 的情况下， "c=" 和 "m=" 行中的连接地址、端口和传输协议分别出现在 "candidate" 属性中，值出现在 "rtcp" 属性中，出现在 "candidate" 属性中。

本规范没有提供关于在不满足上述条件的情况下代理应如何进行的指导，以下几点例外：
  1. 如 [第 8 节](https://www.rfc-editor.org/rfc/rfc8839#sec-alg-sip) 所述，某些应用层网关的存在可能会修改传输地址信息。在这种情况下响应代理的行为取决于实现。非正式地，响应代理可以将不匹配的传输地址信息视为从对等方学习到的似是而非的新候选地址，并继续其包含该传输地址的 ICE 处理。或者，响应代理可以 (MAY) 在其对此数据流的应答中包含 "ice-mismatch" 属性。如果代理选择在其对数据流的应答中包含 "ice-mismatch" 属性，那么它也必须 (MUST) 省略 "candidate" 属性，必须 (MUST) 终止 ICE 程序的使用，并且必须 (MUST) 为该数据流使用 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 程序来代替。

  2. 来自对等方的默认目的地址的传输地址设置为 IPv4/IPv6 地址值 "0.0.0.0"/"::" 和端口值 "9"。这不得 (MUST NOT) 被对等代理视为 ICE 故障，并且 ICE 处理必须 (MUST) 照常继续。

  3. 在某些情况下，控制/发起者代理可能会收到一个 SDP 应答，该应答可能会省略数据流的 "candidate" 属性，而是包含媒体级别的 "ice-mismatch" 属性。这向要约者发出信号，表明应答者支持 ICE，但 ICE 处理并未用于该数据流。在这种情况下，必须 (MUST) 终止该数据流的 ICE 处理，而必须 (MUST) 遵循 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的程序。

  4. 来自对等方的默认目的地址的传输地址是一个 FQDN。无论用于解析 FQDN 的过程或解析结果如何，对等代理都不得 (MUST NOT) 将其视为 ICE 故障，并且 ICE 处理必须 (MUST) 照常继续。

### 4.2.6. SDP 示例

以下是包含 ICE 属性的示例 SDP 消息（为便于阅读而折叠行）：
```
v=0
o=jdoe 2890844526 2890842807 IN IP4 203.0.113.141
s=
c=IN IP4 192.0.2.3
t=0 0
a=ice-options:ice2
a=ice-pacing:50
a=ice-pwd:asd88fgpdd777uzjYhagZg
a=ice-ufrag:8hhY
m=audio 45664 RTP/AVP 0
b=RS:0
b=RR:0
a=rtpmap:0 PCMU/8000
a=candidate:1 1 UDP 2130706431 203.0.113.141 8998 typ host
a=candidate:2 1 UDP 1694498815 192.0.2.3 45664 typ srflx raddr
 203.0.113.141 rport 8998
```

## 4.3. 初始要约/应答交换

### 4.3.1. 发送初始的要约

当要约者生成初始要约时，在每个 "m=" 部分中，它必须 (MUST) 包含与该 "m=" 部分关联的每个可用候选地址的 SDP "candidate" 属性。此外，要约者必须 (MUST) 在要约中包含一个 SDP "ice-ufrag" 属性、一个 SDP "ice-pwd" 属性和一个带有 "ice2" 属性值的 SDP "ice-options" 属性。如果要约者是一个完整的 ICE 实现，它应该 (SHOULD) 在要约中包含一个 "ice-pacing" 属性（如果不包含，将应用默认值）。精简版 ICE 实现不得 (MUST NOT) 在要约中包含 "ice-pacing" 属性（因为它不会执行连接检查）。

要约的 "m=" 行不包含 SDP "candidate" 属性，并将默认目标地址设置为 IP 地址值 "0.0.0.0"/"::"，端口值设置为"9"，是有效的。这意味着要约的代理将仅使用对等自反候选地址或将在后续信令消息中提供额外的候选地址。

注意：在本文档的范围内，“初始要约” 是指为了协商 ICE 的使用而发送的第一个 SDP 要约。它可能是，也可能不是 SDP 会话的初始 SDP 要约。

注意：本文档中的程序仅考虑，与使用 ICE 的数据流关联的 "m=" 部分。

### 4.3.2. 发送初始的应答

当应答者收到一个表明要约者支持 ICE 的初始要约，并且如果应答者接受这个要约和使用 ICE，那么应答者必须 (MUST) 在应答的每个 "m=" 部分中，为与 "m=" 部分关联的每个可用的候选地址包含 SDP "candidate" 属性 。此外，应答者必须 (MUST) 在应答中包含一个 SDP "ice-ufrag" 属性、一个 SDP "ice-pwd" 属性和一个带有 "ice2" 属性值的 SDP "ice-options" 属性。如果应答者是一个完整的 ICE 实现，它应该 (SHOULD) 在应答中包含一个 "ice-pacing" 属性（如果不包含，将应用默认值）。精简版 ICE 实现不得 (MUST NOT) 在应答中包含 "ice-pacing" 属性（因为它不会执行连接检查）。

在每个 "m=" 行中，应答者必须 (MUST) 使用与要约的 "m=" 行中使用的相同的传输协议。如果应答的 "m=" 行中，没有候选地址使用与要约的 "m=" 行中指示的相同的传输协议，那么，为了避免 ICE 不匹配，必须 (MUST) 将默认目标地址设置为 IP 地址值 "0.0.0.0"/"::" 和端口值 "9"。

应答的 "m=" 行不包含 SDP "candidate" 属性，并将默认目标地址设置为 IP 地址值 "0.0.0.0"/"::"，端口值设置为"9"，也是有效的。这意味着应答的代理将仅使用对等自反候选地址或将在后续信令消息中提供额外的候选地址。

一旦应答者发送了应答，它就可以开始对要约中提供的对等候选地址执行连接检查。

如果要约没有表明支持 ICE（[第 4.2.5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-ice-mismatch)），则应答者不得 (MUST NOT) 接受 ICE 的使用。如果应答者仍然接受该要约，则应答者不得 (MUST NOT) 在应答中包含任何与 ICE 相关的 SDP 属性。 相反，应答者将根据正常的要约/应答程序 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 生成应答。

如果应答者检测到 ICE 不匹配的可能性，则遵循 [第 4.2.5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-ice-mismatch) 中描述的程序。

### 4.3.3. 接收初始的应答

当要约者收到一个表明应答者支持 ICE 的初始应答时，它可以开始对应答中提供的对等候选地址执行连接性检查。

如果应答没有表明应答者支持 ICE，或者应答者在应答中包含所有活动数据流的 "ice-mismatch" 属性，则要约者必须 (MUST) 对整个会话终止 ICE 的使用，并且必须 (MUST) 遵循 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的程序。

另一方面，如果应答表明支持 ICE，但在某些活动数据流中包含 "ice-mismatch"，则要约者必须 (MUST) 仅对这些数据流终止使用 ICE 程序，并且必须 (MUST) 使用 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的程序来代替。此外，ICE 程序必须 (MUST) 用于不包含 "ice-mismatch" 属性的数据流。

如果要约者在应答中检测到一个或多个数据流的 ICE 不匹配，如 [第 4.2.5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-ice-mismatch) 所述，要约者必须 (MUST) 终止整个会话的 ICE 使用。 要约者采取的后续行动依赖于实现，并且超出了本规范的范围。

### 4.3.4. 结束 ICE

一旦代理成功提名了一个候选地址对 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]，与该对关联的检查列表的状态设置为已完成 (Completed)。一旦每个检查列表的状态都被设置为了 “已完成 (Completed)” 或 “失败 (Failed)”，对于每个 “已完成 (Completed)” 检查列表，代理都会检查提名的对是否与默认候选地址对匹配。如果有一个或多个对不匹配，并且对等方没有表示支持 'ice2' ice-option，则控制代理必须 (MUST) 生成后续要约，其中与每个数据流关联的 
 "c=" 和 "m=" 行中的连接地址、端口和传输协议，与该数据流的提名候选地址对的对应本地信息匹配（[第 4.4.1.2.2 节](https://www.rfc-editor.org/rfc/rfc8839#sec-send-subsequent-offer-after-nom)）。如果对等方确实表示支持 'ice2' ice-option，则控制代理不需要立即生成更新的要约以将连接地址、端口和协议与提名地址对对齐。然而，在会话的后期，无论何时控制代理确实发送了后续要约时，它必须 (MUST) 按照上述方式进行对齐。

如果有一个或多个检查表的状态被设置为失败 (Failed)，控制代理必须 (MUST) 生成一个后续的要约，以便通过将数据流的端口值设置为零来删除相关的数据流（[第 4.4.1.2.2 节](https://www.rfc-editor.org/rfc/rfc8839#sec-send-subsequent-offer-after-nom)），即使对等体确实表示支持 'ice2' ice-option。如果需要，此类要约用于对齐连接地址、端口和传输协议，如上所述。

如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中所述，一旦控制代理为检查列表提名了一个候选地址对，在 ICE 会话的生命周期内（即，直到 ICE 重新启动），该代理不得 (MUST NOT) 为该检查列表指定另一个候选地址对。

[[RFC8863](https://www.rfc-editor.org/rfc/rfc8839#RFC8863)] 提供了一种机制，允许 ICE 过程运行足够长的时间，以便通过等待潜在的对等自反候选地址来找到工作的候选地址对，即使没有从对等方收到候选地址对，或者与一个检查列表关联的所有当前候选地址对，要么已经失败，要么已经被丢弃。

## 4.4. 后续的要约/应答交换

任一代理都可以 (MAY) 在 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 允许的任何时间生成后续要约。 本节定义了构建后续要约和应答的规则。

如果后续要约失败，ICE 处理将继续进行，就好像后续要约从未提出过一样。

### 4.4.1. 发送后续的要约

#### 4.4.1.1. 所有实现的过程

##### 4.4.1.1.1. ICE 重启

代理可以 (MAY) 为一个现有数据流重新启动 ICE 处理 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]。

管理 ICE 重启的规则意味着将 "c=" 行中的连接地址设置为 "0.0.0.0"（对于 IPv4）/ "::"（对于 IPv6）将导致 ICE 重启。因此，ICE 实现不得 (MUST NOT) 使用这种机制来保持呼叫，而必须 (MUST) 使用 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 中描述的 "inactive" 和 "sendonly"。

要重新启动 ICE，代理必须 (MUST) 同时更改要约中数据流的 "ice-pwd" 和 "ice-ufrag"。 但是，允许在一个要约中使用会话级属性，但在后续要约中的媒体级属性中提供相同的 "ice-pwd" 或 "ice-ufrag"。这不得 (MUST NOT) 被视为 ICE 重启。

代理在此数据流的 SDP 中设置其余与 ICE 相关的字段，就像在此数据流的初始要约中一样（[第 4.2.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-encoding)）。因此，候选地址集可以 (MAY) 包括该数据流先前的一些、不包括或所有候选地址，并且可以 (MAY) 包括全新的候选地址集。代理可以 (MAY) 修改 SDP "ice-options" 和 SDP "ice-pacing" 属性的属性值，并且可以 (MAY) 使用 SDP "ice-lite" 属性改变它的角色。代理不得 (MUST NOT) 在后续要约中修改 SDP "ice-options"、"ice-pacing" 和 "ice-lite" 属性，除非发送要约是为了请求 ICE 重新启动。

##### 4.4.1.1.2. 移除数据流

如果代理通过将其端口设置为零来删除数据流，则它不得 (MUST NOT) 为该数据流包含任何 "candidate" 属性，并且不应 (SHOULD NOT) 为该数据流包含 [第 5 节](https://www.rfc-editor.org/rfc/rfc8839#sec-grammar) 中定义的任何其它 ICE 相关属性。

##### 4.4.1.1.3. 添加数据流

如果代理希望添加一个新的数据流，它会在 SDP 中为该数据流设置字段，就好像这是该数据流的初始要约（[第 4.2.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-encoding)）。这将导致 ICE 开始对该数据流进行处理。

#### 4.4.1.2. 完整实现的过程

本节描述完整实现的附加过程，涵盖现有数据流。

##### 4.4.1.2.1. 提名前

当要约者发送后续要约时；在候选地址对尚未被提名的每个 "m=" 部分中，要约必须 (MUST) 包含与先前要约或应答中要约者包含的相同的一组 ICE 相关信息。代理可以 (MAY) 包含它以前没有提供，但自上次要约/应答交换以来收集的其它候选地址，包括对等自反候选地址。

代理可以 (MAY) 更改媒体的默认目的地址。与初始要约一样，要约中必须 (MUST) 有一组与此默认目的地址匹配的 "candidate" 属性。

##### 4.4.1.2.2. 提名后

一旦为数据流提名了候选地址对，与该数据流相关的每个 "c=" 和 "m=" 行中的连接地址、端口和传输协议，必须 (MUST) 和与该数据流的提名地址对相关联的数据匹配。此外，要约者仅包含代表提名候选地址对的本地候选地址的 SDP "candidate" 属性（每个组件一个）。 要约者不得 (MUST NOT) 在后续要约中包含任何其它 SDP "candidate" 属性。

此外，如果代理是控制代理，它必须 (MUST) 为每个其检查列表处于已完成 (Completed) 状态的数据流包含 "remote-candidates" 属性。该属性包含与该数据流的每个组件的有效列表中的提名地址对相对应的远程候选地址。需要避免这样的竞争条件，即控制代理选择其候选地址对，但更新的要约先于连接性检查到达受控代理，受控代理甚至不知道这些候选地址对是有效的，更不用说被选中了。有关此竞争条件的详细说明，请参见 [附录 B](https://www.rfc-editor.org/rfc/rfc8839#sec-why-remote)。

#### 4.4.1.3. 精简实现的过程

如果 ICE 状态为运行中 (Running)，则精简实现必须在任何后续要约的 "candidate" 属性中包含它每个数据流的每个组件的所有候选地址。 候选地址的形成与初始要约的过程相同。

精简实现不得 (MUST NOT) 在后续要约中添加额外的主机候选地址，并且不得 (MUST NOT) 修改用户名片段和密码。如果代理需要提供额外的候选地址，或修改用户名片段和密码，它必须 (MUST) 为该数据流请求 ICE 重启（[第 4.4.1.1.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-suboffer-restarts)）。

如果 ICE 已经完成了一个数据流，并且如果代理受控代理，则该数据流的默认目的地址，必须 (MUST) 设置为有效列表中该组件的候选地址对的远程候选地址。对于精简实现，对于数据流的每个组件，有效列表中始终只有一个候选地址对。此外，代理必须 (MUST) 为每个默认目的地址包含一个 "candidate" 属性。

如果 ICE 状态是已完成 (Completed)，并且如果代理是控制代理（仅当两个代理都是精简实现时发生），代理必须 (MUST) 为每个数据流包含 "remote-candidates" 属性。该属性包含来自有效列表中候选地址对的远程候选地址（每个数据流的每个组件都有一对）。

### 4.4.2. 发送后续的应答

如果数据流的 ICE 为已完成 (Completed)，并且该数据流的要约缺少 "remote-candidates" 属性，则构建应答的规则与要约者的规则相同，除了应答者不得 (MUST NOT) 在应答中包含 "remote-candidates" 属性外。

当对等端已经结束对数据流的 ICE 处理时，受控代理将收到具有数据流的 "remote-candidates" 属性的要约。该属性出现在要约中，以处理收到要约和收到绑定响应之间的竞争条件，该绑定响应告诉应答者将由 ICE 选择的候选地址。有关此竞争条件的解释，请参见 [附录 B](https://www.rfc-editor.org/rfc/rfc8839#sec-why-remote)。因此，对具有此属性的要约的处理取决于竞争的获胜者。

代理通过以下方式为数据流的每个组件形成一个候选地址对：

 * 将远程候选地址设置为该组件的要约者的默认目的地址（即，RTP 的 "m=" 和 "c=" 行的内容，以及 RTCP 的 "rtcp" 属性）
 * 将本地候选地址设置为要约中的 "remote-candidates" 属性中的该相同组件的传输地址。

然后，代理会查看这些候选地址对中的每一个是否存在于有效列表中。 如果特定候选地址对不在有效列表中，则该检查 “输掉” 竞争。 称这样的对为 “输掉的对 (losing pair)”。

代理在检查列表中找到所有对，其远程候选地址与输掉的对中的远程候选地址相同的：

 * 如果没有一对处于进行中 (In-Progress)，并且至少有一个是失败的 (Failed)，则很可能发生了网络故障，例如发生了网络分区或严重的数据包丢失。代理应该 (SHOULD) 为这个数据流生成一个应答，就好像 "remote-candidates" 属性不存在一样，然后为这个流重新启动 ICE。

 * 如果有至少一对处于进行中 (In-Progress)，代理应该 (SHOULD) 等待这些检查完成，并且当每个检查完成时，重做本节中的处理，直到没有 输掉的对。

一旦没有 输掉的对，代理就可以生成应答。它必须 (MUST) 将媒体的默认目的地址，设置为来自要约的 "remote-candidates" 属性中的候选地址（现在这每个候选地址都将成为有效列表中候选地址对的本地候选地址）。它必须 (MUST) 在应答中为要约中的 "remote-candidates" 属性中的每个候选地址包含一个 "candidate" 属性。

#### 4.4.2.1. ICE 重启

如果在随后的要约中要约者请求数据流的 ICE 重启动（[第 4.4.1.1.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-suboffer-restarts)），并且如果应答者接受要约，则应答者遵循生成初始应答的过程。

对于给定的数据流，应答者可以 (MAY) 包含在前一个 ICE 会话中使用的相同候选地址，但它必须 (MUST) 更改 SDP "ice-pwd" 和 "ice-ufrag" 属性值。

应答者可以 (MAY) 修改 SDP "ice-options" 和 SDP "ice-pacing" 属性的属性值，并且它可以 (MAY) 使用 SDP "ice-lite" 属性改变它的角色。应答者不得 (MUST NOT) 在后续应答中修改 SDP "ice-options"、"ice-pacing" 和 "ice-lite" 属性，除非该应答是针对用于请求 ICE 重启的要约而发送的（[第 4.4.1.1.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-suboffer-restarts)）。如果在随后的要约中修改了任何 SDP 属性，但未用于请求 ICE 重新启动，则应答者必须 (MUST) 拒绝该要约。

#### 4.4.2.2. 精简实现的特有过程

如果收到的要约包含数据流的 "remote-candidates" 属性，则代理通过以下方式为数据流的每个组件形成一个候选地址对：

 * 将远程候选地址设置为该组件的要约者的默认目的地址（即，RTP 的 "m=" 和 "c=" 行的内容，以及 RTCP 的 "rtcp" 属性）
 * 将本地候选地址设置为要约中的 "remote-candidates" 属性中的该相同组件的传输地址。

与该数据流关联的检查列表的状态设置为已完成 (Completed)。

此外，如果代理认为自己是控制代理，但要约包含 "remote-candidates" 属性，则两个代理都认为它们是控制代理。在这种情况下，两者都将在大约同一时间发送更新的要约。

然而，承载要约/应答交换的信令协议将解决这种眩光情况，通过使一个代理的要约总是在它的对等方发送要约之前被收到，使该代理总是成为 “赢家”。获胜者扮演控制角色，因此失败者（本节中认为的应答者）必须 (MUST) 将其角色更改为受控者。

因此，如果代理根据 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] [第 8.2 节](https://www.rfc-editor.org/rfc/rfc8445#section-8.2) 中的规则进行控制，并且要发送更新的要约，则不再需要。

除了潜在的角色变化、有效列表的变化和状态变化之外，应答的构建与要约的构建相同。

### 4.4.3. 收到后续要约的应答

#### 4.4.3.1. 完整实现的过程

在某些情况下，要约者收到的 SDP 应答可能缺少 ICE 候选地址，尽管最初的应答包括它们。这种 “意外” 应答可能会发生的一个示例是，当一个不知道 ICE 的背靠背用户代理 (B2BUA) 在呼叫保持期间，使用第三方呼叫控制程序 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] 引入媒体服务器时。省略有关如何完成的更多细节，这可能会导致在保持 (holding) 用户代理 UA 处收到由 B2BUA 构建的应答。由于 B2BUA 不了解 ICE，因此该应答将不包括 ICE 候选地址。

在这种情况下收到没有 ICE 属性的应答可能出乎意料，但不一定会损害用户体验。

当要约者收到表明支持 ICE 的应答时，要约者将执行以下操作之一：

 * 如果要约是一个重新启动，代理必须 (MUST) 执行 [第 4.4.3.1.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-restart-subsequent) 中描述的 ICE 重新启动过程。
 * 如果要约/应答交换删除了数据流，或者应答拒绝了要约的数据流，则代理必须 (MUST) 刷新该数据流的有效列表。它还必须 (MUST) 终止该数据流正在进行的任何 STUN 事务。
 * 如果要约/应答交换添加了一个新的数据流，代理必须 (MUST) 为它创建一个新的检查列表（当然还有一个空的有效列表来开始），这反过来又会触发候选地址处理程序 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]。
 * 如果与数据流关联的检查列表状态为正在运行 (Running)，则代理会重新计算检查列表。如果新检查列表上的候选地址对也在前一个检查列表上，则复制其候选地址对状态。否则，其候选地址对状态设置为冻结 (Frozen)。如果没有一个检查列表处于活动状态（意味着每个检查列表中的候选地址对状态为冻结 (Frozen)），则执行 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中的适当过程以将候选地址对，移动到等待 (Waiting) 状态以进一步继续 ICE 处理。
 * 如果 ICE 状态为已完成 (Completed)，并且 SDP 应答符合 [第 4.4.2 节](https://www.rfc-editor.org/rfc/rfc8839#sec-subsequent-answer)，则代理必须 (MUST) 保持在已完成 (Completed) ICE 状态。

然而，如果 SDP 应答中没有再指示 ICE 支持，代理必须 (MUST) 退回到 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的过程，并且不应 (SHOULD NOT) 因为缺少 ICE 支持或意外应答而放弃对话。当代理发送新的要约时，它必须 (MUST) 执行 ICE 重启。

##### 4.4.3.1.1. ICE 重启

在重新启动之前，代理必须 (MUST) 记住数据流每个组件的有效列表中的提名的候选地址对，称为 “先前选择的对”。代理将继续使用该对发送媒体，如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 12 节](https://www.rfc-editor.org/rfc/rfc8445#section-12) 所述。一旦注意到这些目的地址，代理必须 (MUST) 刷新有效列表和检查列表，然后重新计算检查列表及其状态，从而触发候选地址处理程序 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)]。

#### 4.4.3.2. 精简实现的过程

如果 ICE 正在为数据流重新启动，代理必须 (MUST) 为该数据流创建一个新的有效列表。它必须 (MUST) 记住数据流每个组件之前的有效列表中的提名的候选地址对，称为 “先前选择的对”，并将继续在那里发送媒体，如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 12 节](https://www.rfc-editor.org/rfc/rfc8445#section-12) 所述。每个数据流的每个检查列表的状态必须 (MUST) 更改为正在运行 (Running)，并且 ICE 状态必须 (MUST) 设置为正在运行 (Running)。

# 5. 语法

本规范定义了八个新的 SDP 属性 —— "candidate"、"remote-candidates"、"ice-lite"、"ice-mismatch"、"ice-ufrag"、"ice-pwd"、"ice-pacing"、 和 "ice-options" 属性。

本节还提供了所定义属性的非规范性示例。

属性的语法遵循 [[RFC5234](https://www.rfc-editor.org/rfc/rfc8839#RFC5234)] 中定义的增强 BNF。

## 5.1. "candidate" 属性

"candidate" 属性只是媒体级的属性。它为可用于连接检查的候选地址包含了一个传输地址。
```
candidate-attribute   = "candidate" ":" foundation SP component-id SP
                        transport SP
                        priority SP
                        connection-address SP     ;from RFC 4566
                        port         ;port from RFC 4566
                        SP cand-type
                        [SP rel-addr]
                        [SP rel-port]
                        *(SP cand-extension)

foundation            = 1*32ice-char
component-id          = 1*3DIGIT
transport             = "UDP" / transport-extension
transport-extension   = token              ; from RFC 3261
priority              = 1*10DIGIT
cand-type             = "typ" SP candidate-types
candidate-types       = "host" / "srflx" / "prflx" / "relay" / token
rel-addr              = "raddr" SP connection-address
rel-port              = "rport" SP port
cand-extension        = extension-att-name SP extension-att-value
extension-att-name    = token
extension-att-value   = *VCHAR
ice-char              = ALPHA / DIGIT / "+" / "/"
```

该语法对候选地址的主要信息进行编码：它的 IP 地址、端口和传输协议，以及它的属性：基础、组件 ID、优先级、类型和相关的传输地址：

`<connection-address>`：取自于 RFC 4566 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)]。它是候选地址的 IP 地址，允许 IPv4 地址、IPv6 地址和完全限定的域名 (FQDNs)。解析此字段时，代理可以通过其值中是否存在冒号来区分 IPv4 地址和 IPv6 地址 - 存在冒号表示 IPv6。生成本地候选地址的代理不得 (MUST NOT) 使用 FQDN 地址。处理远程候选地址的代理，必须 (MUST) 忽略其中包含具有 FQDN 或 IP 地址版本不支持或无法识别的候选地址的 "candidate" 行。生成和处理 FQDN 候选地址的过程，以及代理如何指示对此类过程的支持，需要在扩展规范中指定。

`<port>`：取自于 RFC 4566 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)]。它是候选地址的端口。

`<transport>`：表示候选地址的传输协议。本规范只定义了 UDP。但是，通过扩展 “交互式连接建立 (ICE)” 注册表下的子注册表 “ICE 传输协议”，提供了可扩展性，以允许未来的传输协议与 ICE 一起使用。

`<foundation>`：由 1 至 32 个 `<ice-char>` 组成。它是一个标识符，对于两个具有相同类型、共享相同基并且来自相同 STUN 服务器的候选地址是相等的。基础 foundation 用于优化冻结 (Frozen) 算法中的 ICE 性能，如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中所述。

`<component-id>`：是一个 1 到 256 (包含) 之间的正整数，标识候选地址所属的数据流的特定组件。它必须 (MUST) 从 1 开始，并且必须 (MUST) 为特定候选地址的每个组件增加 1。对于基于 RTP 的数据流，实际 RTP 媒体的候选地址必须 (MUST) 具有为 1 的组件 ID，而 RTCP 的候选地址必须 (MUST) 具有为 2 的组件 ID。有关将 ICE 扩展到新数据流的更多讨论，请参见 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 13 节](https://www.rfc-editor.org/rfc/rfc8445#section-13)。

`<priority>`：是一个 1 到 (2^31 - 1) (包含) 之间的正整数。计算候选地址优先级的过程在 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 5.1.2 节](https://www.rfc-editor.org/rfc/rfc8445#section-5.1.2) 描述。

`<cand-type>`：编码候选地址的类型。本规范分别为主机、服务器自反、对端自反和中继候选地址定义了值 "host"、"srflx"、"prflx" 和 "relay"。新候选地址类型的规范必须 (MUST) 定义 ICE 处理中的各个步骤与本规范定义的步骤有何不同（如果有的话）。

`<rel-addr>` 和 `<rel-port>`：传达与候选地址相关的传输地址，对诊断和其它目的有用。`<rel-addr>` 和 `<rel-port>` 必须出现在服务器自反、对端自反和中继候选地址中。如果候选地址是服务器自反或对等自反，则 `<rel-addr>` 和 `<rel-port>` 等于该服务器自反或对等自反候选地址的基。如果候选地址是中继候选地址，则 `<rel-addr>` 和 `<rel-port>` 等于分配 (Allocate) 响应中为客户端提供中继候选地址的映射地址（参见 [[RFC5766](https://www.rfc-editor.org/rfc/rfc8839#RFC5766)] 的 [第 6.3 节](https://www.rfc-editor.org/rfc/rfc5766#section-6.3)）。如果候选地址是主机候选地址，则必须省略 `<rel-addr>` 和 `<rel-port>`。
在某些情况下，例如出于隐私原因，代理可能不想透露相关的地址和端口。在这种情况下，地址必须设置为 "0.0.0.0"（用于 IPv4 候选地址）或 "::"（用于 IPv6 候选地址），端口必须设置为 "9"。

"candidate" 属性本身可以扩展。该语法允许在属性末尾添加新的名称/值对。此类扩展必须 (MUST) 通过 IETF Review 或 IESG Approval [[RFC8126](https://www.rfc-editor.org/rfc/rfc8839#RFC8126)] 进行，并且分配必须 (MUST) 包含特定扩展，和对定义扩展的用法的文档的引用。

实现必须 (MUST) 忽略它不理解的任何名称/值对。

以下是 RTP 组件的 UDP 服务器自反 "candidate" 属性的示例 SDP 行：
```
a=candidate:2 1 UDP 1694498815 192.0.2.3 45664 typ srflx raddr
203.0.113.141 rport 8998
```

## 5.2. "remote-candidates" 属性

"remote-candidates" 属性的语法使用 [[RFC5234](https://www.rfc-editor.org/rfc/rfc8839#RFC5234)] 中定义的增强 BNF 定义。"remote-candidates" 属性只是媒体级的属性。
```
remote-candidate-att = "remote-candidates:" remote-candidate
                         0*(SP remote-candidate)
remote-candidate = component-id SP connection-address SP port
```

该属性包含每个组件的连接地址和端口。组件的顺序无关紧要。但是，数据流的每个组件都必须 (MUST) 存在一个值。该属性必须 (MUST) 由控制代理包含在已完成 (Completed) 数据流的要约中，并且不得 (MUST NOT) 包含在任何其它情况下。

以下是 RTP 和 RTCP 组件的 "remote-candidates" SDP 行示例：
```
a=remote-candidates:1 192.0.2.3 45664
a=remote-candidates:2 192.0.2.3 45665
```

## 5.3. "ice-lite" 和 "ice-mismatch" 属性

"ice-lite" 和 "ice-mismatch" 属性的语法，它们都是标记，是：
```
ice-lite               = "ice-lite"
ice-mismatch           = "ice-mismatch"
```

"ice-lite" 只是一个会话级属性，表示代理是一个精简的实现。"ice-mismatch" 是一个媒体级属性，且只在应答中报告。它表示要约到达时，带有一个媒体组件的默认目的地址，该媒体组件没有相应的 "candidate" 属性。为给定数据流包含 "ice-mismatch" 属性，意味着即使两个代理都支持 ICE，ICE 程序不得 (MUST NOT) 用于此数据流，而必须 (MUST) 使用 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 的程序。

## 5.4. "ice-ufrag" 和 "ice-pwd" 属性

"ice-ufrag" 和 "ice-pwd" 属性传达了 ICE 用于消息完整性的用户名片段和密码。它们的语法是：
```
ice-pwd-att           = "ice-pwd:" password
ice-ufrag-att         = "ice-ufrag:" ufrag
password              = 22*256ice-char
ufrag                 = 4*256ice-char
```

"ice-ufrag" 和 "ice-pwd" 属性可以出现在会话级或媒体级。当两者中都出现时，媒体级中的值优先。因此，会话级别的值实际上是适用于所有数据流的默认值，除非被媒体级别的值覆盖。无论是在会话级别还是媒体级别，每个数据流都必须 (MUST) 有一个 "ice-pwd" 和 "ice-ufrag" 属性。如果两个数据流具有一致的 "ice-ufrag"，则它们必须 (MUST) 具有一致的 "ice-pwd"。

"ice-ufrag" 和 "ice-pwd" 属性必须 (MUST) 在会话开始时随机地选择（当 ICE 为代理重新启动时同样适用）。

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 要求 "ice-ufrag" 属性至少包含 24 位的随机性，"ice-pwd" 属性至少包含 128 位的随机性。这意味着 "ice-ufrag" 属性将至少是 4 个字符长的，而 "ice-pwd" 至少是 22 个字符长的，因为这些属性的语法允许每个字符包含 6 位信息。属性分别可以 (MAY) 超过 4 和 22 个字符，当然最多 256 个字符。上限允许在实现中调整缓冲区大小。其较大的上限，允许随着时间的推移增加更多的随机性。

为了与 STUN 用户名属性值的 512 个字符限制兼容，以及出于带宽节约考虑，"ice-ufrag" 属性在发送时不得 (MUST NOT) 超过 32 个字符，但一个实现在接收时必须 (MUST) 接受最多 256 个字符。

以下示例显示了样本 "ice-ufrag" 和 "ice-pwd" SDP 行：
```
a=ice-pwd:asd88fgpdd777uzjYhagZg
a=ice-ufrag:8hhY
```

## 5.5. "ice-pacing" 属性

"ice-pacing" 是一个会话级属性，它指示发送者希望使用的，想要的连接检查的步调（Ta 间隔），以毫秒为单位。有关选择 pacing 值的更多信息，请参阅 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 14.2 节](https://www.rfc-editor.org/rfc/rfc8445#section-14.2)。语法是：
```
ice-pacing-att            = "ice-pacing:" pacing-value
pacing-value              = 1*10DIGIT
```

如果在要约或应答中不存在，则该属性的默认值为 50 ms，这是 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中指定的推荐值。

正如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中所定义的，无论为每个代理选择的 Ta 值如何，来自所有代理的所有事务的组合（如果给定的实现运行多个并发的代理）将不会超过每 5 毫秒发送一次的频率。

正如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中所定义的，一旦两个代理都指示了它们想要使用的 pacing 值，两个代理将使用指示值中的较大者。

以下示例显示了一个值为 '50' 的 "ice-pacing" SDP 行：
```
a=ice-pacing:50
```

## 5.6. "ice-options" 属性

"ice-options" 属性是一个会话级和媒体级属性。它包含一系列标识代理支持的选项的标记。它的语法是：
```
ice-options           = "ice-options:" ice-option-tag
                          *(SP ice-option-tag)
ice-option-tag        = 1*ice-char
```

要约中存在 "ice-options" 表示代理支持某个扩展，并且如果对等代理在应答中也包含相同的扩展，则它希望使用它。在确定如何在给定会话中使用扩展的代理之间，可能需要进一步的扩展特定的协商。协商过程的细节，如果存在，必须 (MUST) 由定义扩展的规范定义（[第 10.2 节](https://www.rfc-editor.org/rfc/rfc8839#sec-iana-ice-options)）。

下面的示例显示了一个带有 'ice2' 和 'rtp+ecn' [[RFC6679](https://www.rfc-editor.org/rfc/rfc8839#RFC6679)] 值的 "ice-options" SDP 行。
```
a=ice-options:ice2 rtp+ecn
```

[原文](https://www.rfc-editor.org/rfc/rfc8839)
