---
title: RFC8839 交互式连接建立 (ICE) 的会话描述协议 (SDP) 提议/应答过程 II
date: 2022-02-05 12:35:49
categories: 网络协议
tags:
- 网络协议
---

# 6. 保活

所有的 ICE 代理必须 (MUST) 遵循 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 11 节](https://www.rfc-editor.org/rfc/rfc8445#section-11) 中定义的过程发送保活消息。正如 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中所定义的，无论数据流当前是处于 inactive、sendonly、recvonly 还是 sendrecv，并且无论带宽属性是否存在或值为何，都将发送 keepalive。代理可以通过每个媒体会话的 "candidate" 属性的存在来确定其对等方支持 ICE。

# 7. SIP 注意事项

请注意，ICE 不是用于 SIP 信令的 NAT 穿越的，这被假设通过另一种机制 [[RFC5626](https://www.rfc-editor.org/rfc/rfc8839#RFC5626)] 提供。

当 ICE 与 SIP 一起使用时，分叉可能会导致单个要约生成多个应答。在这种情况下，ICE 对每个应答完全并行且独立地进行，将其要约和每个应答的组合视为独立的要约/应答交换，具有自己的一组本地候选地址、候选地址对、检查列表、状态等。

# 7.1. 延迟指南

ICE 需要在端点之间进行一系列基于 STUN 的连接检查。这些检查从应答者生成其应答时开始，并从要约者收到应答时开始。这些检查可能需要一些时间才能完成，因此，选择与要约和应答一起使用的消息可能会影响感知到的用户延迟。两个延迟数据特别令人感兴趣。它们是接听后延迟和拨号后延迟。接听后延迟是指从用户 “接听电话” 到它们说出的任何语音可以传递给呼叫者之间的时间。拨号后延迟是指从用户输入用户的目标地址，到由于已成功开始向被呼叫用户代理发出警报而开始回铃之间的时间。

可以考虑两种情况——一种是要约 (offer) 在初始邀请 (INVITE) 中，另一种是在响应中。

# 7.1.1. 要约 (Offer) 在邀请 (INVITE) 中

为了减少拨号后延迟，建议 (RECOMMENDED) 呼叫者在实际发送其初始 邀请 (INVITE) 之前开始收集候选地址，以便可以在邀请 (INVITE) 中提供候选地址。这可以在呼叫未决的用户界面提示时启动，例如键盘上的活动或电话摘机。

收到要约后，应答者应该 (SHOULD) 在完成候选地址收集后立即在临时响应中生成应答。ICE 要求可靠地传输带有 SDP 的临时响应。这可以通过现有的临时响应确认 (PRACK) 机制 [[RFC3262](https://www.rfc-editor.org/rfc/rfc8839#RFC3262)]，或通过特定于 ICE 的优化来完成，其中，代理使用 [[RFC3262](https://www.rfc-editor.org/rfc/rfc8839#RFC3262)] 中描述的指数退避计时器重传临时响应。这种重传 (MUST) 必须在收到 STUN 绑定请求时停止，该请求的传输地址与在该 SDP 中包含的数据流中的一个的候选地址匹配，或者在传输 2xx 响应中的应答时停止。如果在最后一次重传之前没有收到绑定请求，则代理不认为会话已终止。对于精简 ICE 对等体，代理必须 (MUST) 自没有绑定请求发送后，发送 18x 响应四次之后停止重传它，并且数字四是随意选择的，用来限制 18x 重传的次数。

一旦发送了应答，则代理应该 (SHOULD) 开始它的连接检查。一旦数据流的每个组件的候选地址对进入有效列表，则应答者可以开始在该数据流上发送媒体了。

然而，在此之前，需要发送给呼叫者的任何媒体（例如 SIP 早期媒体 [[RFC3960](https://www.rfc-editor.org/rfc/rfc8839#RFC3960)]）不得 (MUST NOT) 被传输。出于这个原因，实现应该 (SHOULD) 延迟提醒被叫方，直到每个数据流的每个组件的候选地址都进入有效列表。在 PSTN 网关的情况下，这意味着建立 (setup) 消息进入 PSTN 的时刻会延迟到这个时候。这样做会增加拨号后延迟，但具有消除 “魔鬼铃声” 的效果。魔鬼铃声是这样的情况，被呼叫方听到电话铃声，接听，但什么也没听到，听不到的情况。该技术无需支持或使用前提条件 [[RFC3312](https://www.rfc-editor.org/rfc/rfc8839#RFC3312)] 即可工作。它还有一个好处，即保证没有一个媒体包会被剪辑，因此接听后延迟为零。如果代理选择以这种方式延迟本地告警，则它应该 (SHOULD) 在告警开始时生成一个 180 响应。

# 7.1.2. 要约 (Offer) 在响应中

除了在邀请 (INVITE) 中提供要约 (Offer)，在临时响应和/或 200 OK 响应中提供应答的用法之外，ICE 还可以处理要约出现在响应中的情况。在这种情况下，这在第三方呼叫控制 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] 中很常见，ICE 代理应该 (SHOULD) 在可靠的临时响应中生成它们的要约（必须 (MUST) 使用 [[RFC3262](https://www.rfc-editor.org/rfc/rfc8839#RFC3262)]），并且不会在收到邀请 (INVITE) 时提醒用户。应答将出现在 PRACK 中。这允许 ICE 处理在告警之前进行，因此不会出现接听后延迟，但会增加呼叫建立延迟。一旦 ICE 完成，被呼叫者可以提醒用户，然后在它们应答时生成 200 OK。200 OK 将不包含 SDP，因为要约/应答交换已完成。

或者，代理可以 (MAY) 将要约放在 2xx 中（在这种情况下，应答出现在 ACK 中）。发生这种情况时，被叫方会在收到邀请 (INVITE) 时提醒用户，只有在用户应答后才会进行 ICE 交换。这具有减少呼叫建立延迟的效果，但会导致大量的接听后延迟和媒体剪辑。

## 7.2. SIP 选项标签和媒体功能标签

[[RFC5768](https://www.rfc-editor.org/rfc/rfc8839#RFC5768)] 指定了一个 SIP 选项标签和媒体功能标签以供 ICE 使用。使用 SIP 的 ICE 实现应该 (SHOULD) 支持这个规范，它在注册中使用一个功能标签，通过信令中介促进互操作性。

## 7.3. 与分叉的交互

ICE 与分叉的交互非常好。事实上，ICE 修复了一些与分叉相关的问题。 如果没有 ICE，当一个呼叫分叉，并且主叫方收到多个传入数据流时，它无法确定哪个数据流对应于哪个被叫方。

通过 ICE，这个问题被解决了。在媒体传输之前发生的连接检查，携带用户名片段，这些用户名片段又与特定的被叫者相关联。与连接检查相同的候选地址对上到达的后续媒体数据包，将与同一被叫方相关联。因此，呼叫者只要收到应答就可以执行这种关联。

## 7.4. 与前提条件的交互

在 [[RFC3312](https://www.rfc-editor.org/rfc/rfc8839#RFC3312)] 和 [[RFC4032](https://www.rfc-editor.org/rfc/rfc8839#RFC4032)] 中定义的服务质量 (QoS) 前提条件，仅适用于作为要约/应答中媒体的默认目标地址列出的传输地址。如果 ICE 更改接收媒体的传输地址，此更改将反映在更新的要约中，它将更改媒体的默认目的地址以匹配 ICE 的选择。因此，它看起来像任何其它 re-INVITE 一样，并且在 RFC 3312 和 4032 中得到了充分处理，这适用于不考虑由于 “在后台” 发生的 ICE 协商，而导致媒体目的地址发生变化的事实。

实际上，在检查已经完成，且已经选择了媒体要使用的候选地址对之前，一个代理不应该 (SHOULD NOT) 表示 QoS 前提条件已满足。

ICE 还与连接性前提条件 [[RFC5898](https://www.rfc-editor.org/rfc/rfc8839#RFC5898)] 交互。这种交互在那儿描述。请注意，[第 7.1 节](https://www.rfc-editor.org/rfc/rfc8839#sec-latency) 中描述的过程，描述了它们自己的 “先决条件” 类型，尽管其功能比 [[RFC5898](https://www.rfc-editor.org/rfc/rfc8839#RFC5898)] 中明确的先决条件提供的功能要少。

## 7.5. 与第三方呼叫控制的交互

ICE 与 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] 中描述的流程 I、III 和 IV 一起工作。流程 I 在控制器不支持或不了解 ICE 的情况下工作。只要控制器透传 ICE 属性而不更改它，流程 IV 就可以工作。流程 II 从根本上与 ICE 不兼容；每个代理都相信自己是应答者，因此永远不会产生 re-INVITE。

如 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] [第 7 节](https://www.rfc-editor.org/rfc/rfc3725#section-7) 所述，持续操作的流程需要 ICE 实现的额外行为来支持。特别是，如果代理收到一个不包含任何要约的中间对话 re-INVITE，它必须 (MUST) 为每个数据流重新启动 ICE，并完成收集新候选地址的过程。此外，该候选地址列表应该 (SHOULD) 包括当前用于媒体的那些候选地址。

# 8. 与应用层网关和 SIP 的交互

应用层网关 (ALG) 是网络地址转换 (NAT) 设备中存在的功能，它检查数据包的内容并修改它们，以促进应用协议的 NAT 穿越。会话边界控制器 (SBC) 是 ALG 的近亲，但透明度更低，因为它们实际上作为应用层 SIP 中介存在。ICE 与 SBC 和 ALG 有交互。

如果 ALG 知道 SIP 但不知道 ICE，只要 ALG 正确地修改 SDP，ICE 就会通过它。正确的 ALG 实现的行为如下：

 * 如果 "m=" 和 "c=" 行或 "rtcp" 属性包含外部地址，ALG 不会修改它们。
 * 如果 "m=" 和 "c=" 行包含内部地址，则修改依赖于该 ALG 的状态：
     - 如果 ALG 已经建立了一个绑定，将外部端口映射到与 "m=" 和 "c=" 行或 "rtcp" 属性中的值匹配的内部连接地址和端口，则 ALG 使用该绑定而不是创建一个新的。
     - 如果 ALG 还没有绑定，它会创建一个新的绑定，并修改 SDP，重写 "m=" 和 "c=" 行以及 "rtcp" 属性。

不幸的是，众所周知，许多 ALG 在这些极端情况下效果不佳。ICE 不会尝试解决损坏的 ALG，因为这超出了其功能范围。ICE 可以帮助诊断这些情况，这些情况通常表现为候选地址集与 "m=" 和 "c=" 行以及 "rtcp" 属性之间的不匹配。"ice-mismatch" 属性即用于此目的。

当信令通过 TLS 运行时，ICE 通过 ALG 工作得最好。这可以防止 ALG 操纵 SDP 消息并干扰 ICE 操作。预期部署在 ALG 后面的实现 (SHOULD) 应该提供 SDP 的 TLS 传输。

如果 SBC 知道 SIP 但不知道 ICE，结果依赖于 SBC 的行为。如果它充当适当的背靠背用户代理 (B2BUA) 的角色，SBC 将删除它不理解的任何 SDP 属性，包括 ICE 属性。结果，呼叫将出现在两个端点上，就好像对方不支持 ICE。这将导致 ICE 被禁用，如果 SBC 请求，媒体流过 SBC。但是，如果 SBC 未经修改就传递了 ICE 属性，但修改了媒体的默认目标地址（包含在 "m=" 和 "c=" 行以及 "rtcp" 属性中），这将被检测为 ICE 不匹配， 并且该呼叫的 ICE 处理被中止。作为 “解决” SBC 的工具，它超出了 ICE 的范围。如果存在，则不会使用 ICE，SBC 技术优先。

# 9. 安全注意事项

通用 ICE 安全注意事项在 [[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 中定义，通用 SDP 要约/应答安全注意事项在 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 中定义。这些安全注意事项也应用于本文档的实现。

## 9.1. IP 地址隐私

在某些情况下，例如出于隐私原因，代理可能不想暴露相关的地址和端口。 在这种情况下，地址必须 (MUST) 设置为 "0.0.0.0"（用于 IPv4 候选地址）或 "::"（用于 IPv6 候选地址），端口必须设置为 '9'。

## 9.2. 对要约/应答交换的攻击

可以自己修改或破坏要约/应答交换的攻击者，可以很容易地使用 ICE 发起各种攻击。它们可以将媒体引导到 DoS 攻击的目标，它们可以将自己插入数据流中，等等。这些类似于要约/应答交换的一般安全注意事项，并且 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8839#RFC3264)] 中的安全注意事项也适用。这些需要消息完整性技术，以及要约和应答的加密技术，当使用 SIP 时，TLS 机制 [[RFC3261](https://www.rfc-editor.org/rfc/rfc8839#RFC3261)] 可以满足这些要求。因此，建议 (RECOMMENDED) 在 ICE 中使用 TLS。

## 9.3. 语音锤子攻击

语音锤子攻击是一种放大攻击，即使攻击者是会话中经过身份验证的有效参与者也可以触发。在这种攻击中，攻击者向其它代理发起会话，并恶意将 DoS 目标的连接地址和端口作为 SDP 中发出的媒体流量的目的地。这会导致大量的放大；单个要约/应答交换可以创建持续的媒体数据包泛滥，可能以很高的速率（考虑视频源）。ICE 的使用有助于防止这种攻击。

具体来说，如果使用 ICE，接收恶意 SDP 的代理在将媒体发送到那里之前，首先对媒体的目的地址执行连接性检查。如果此目的地址是第三方主机，则检查不会成功，并且永远不会发送媒体。[[RFC7675](https://www.rfc-editor.org/rfc/rfc8839#RFC7675)] 中定义的 ICE 扩展可用于进一步防止语音锤子攻击。

不幸的是，如果不使用 ICE，它就无济于事，在这种情况下，攻击者可以简单地发送没有 ICE 参数的要约。但是，在客户端集合已知，且仅限于支持 ICE 的环境中，服务器可以拒绝任何不表明支持 ICE 的要约或应答。

不愿意接收非 ICE 应答的 SIP 用户代理 (UA) [[RFC3261](https://www.rfc-editor.org/rfc/rfc8839#RFC3261)] 必须 (MUST) 在其要约的 SIP Require 头部字段中包含 "ice" 选项标签 [[RFC5768](https://www.rfc-editor.org/rfc/rfc8839#RFC5768)]。拒绝非 ICE 要约的 UA 通常会使用 421 响应码，以及在响应的 Require 头部字段中包含选项标签 "ice"。

# 10. IANA 注意事项

## 10.1. SDP 属性

最初的 ICE 规范根据 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8839#RFC4566)] 的 [第 8.2.4 节](https://www.rfc-editor.org/rfc/rfc4566#section-8.2.4) 的程序定义了七个新的 SDP 属性。此处包含来自原始规范的注册信息，并进行了修改以包含 Mux 类别 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8839#RFC8859)]，并且还定义了一个新的 SDP 属性 "ice-pacing"。

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

目的：此属性与交互式连接建立 (ICE) 一起使用，并提供要约者希望应答者在其应答中使用的远程候选地址的标识。

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

IANA 根据 “在 RFC 中编写 IANA 注意事项的指南” [[RFC8126](https://www.rfc-editor.org/rfc/rfc8839#RFC8126)] 中定义的 规范要求 政策维护 "ice-options" 标识符的注册表。根据 [第 5.6 节](https://www.rfc-editor.org/rfc/rfc8839#sec-ice-options) 的语法，ICE 选项的长度不受限制；但是，建议 (RECOMMENDED) 它们不超过 20 个字符。这是为了减少消息大小并允许高效的解析。 ICE 选项在会话级别定义。

注册请求必须 (MUST) 包含以下信息：

 * 要注册的 ICE 选项标识符
 * 具有变更控制权的组织或个人的姓名和电子邮件地址
 * 选项相关的 ICE 扩展的简短描述
 * 对定义 ICE 选项和相关扩展的规范的参考引用

## 10.3. 候选地址属性扩展子注册建立

本节在 SDP 参数注册表下创建一个新的子注册表 “候选地址属性扩展”：http://www.iana.org/assignments/sdp-parameters。

子注册表的目的是注册 SDP "candidate" 属性扩展。

当在子注册表中注册 "candidate" 扩展时，它需要满足 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8839#RFC8126)] 中定义的 “规范要求” 政策。

"candidate" 属性扩展必须 (MUST) 遵循 "cand-extension" 语法。 属性扩展名称必须 (MUST) 遵循 'extension-att-name' 语法，属性扩展值必须 (MUST) 遵循 'extension-att-value' 语法。

注册请求必须 (MUST) 包含以下信息：

 * 属性扩展的名称
 * 具有变更控制权的组织或个人的姓名和电子邮件地址
 * 属性扩展的简短描述
 * 对描述属性扩展的语义、用法和可能值的规范的参考引用。

# 11. 自 RFC 5245 以来的变更

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 描述了对通用 SIP 程序所做的更改，包括取消积极提名、修改计算候选地址对状态的程序、调度连接检查以及计时器值的计算。

本文档定义了以下 SDP 要约/应答特有的更改：

 * SDP offer/answer 实现和'ice2' 选项的使用。
 * SDP "ice-pacing" 属性的定义和使用。
 * ICE 代理不得生成具有 FQDN 的候选地址的显式文本，并且如果从对等代理处接收到此类候选地址，则必须丢弃此类候选地址。
 * 放宽包含 SDP "rtcp" 属性的要求。
 * SDP 要约/应答程序的一般说明。
 * ICE 不匹配现在是可选的，并且代理可以选择不触发不匹配，而是将默认候选地址视为附加候选地址。
 * FQDN 和 "0.0.0.0" / "::" IP 地址与端口 "9" 默认候选地址不会触发 ICE 不匹配。

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

"remote-candidates" 属性的存在是为了消除更新的要约，和对将候选地址移动到有效列表中的 STUN 绑定请求的响应之间的竞争条件。这种竞态条件如图 1 所示。收到消息 4 后，代理 L 将候选地址对添加到有效列表中。 如果只有一个包含单个组件的数据流，代理 L 现在可以发送更新的要约。但是，代理 R 的检查尚未收到响应，代理 R 在收到响应（消息 9）之前收到更新的要约（消息 7）。 因此，它还不知道这个特定的对是有效对。为了消除这种情况，由要约者选择的 R 的实际候选地址（远程候选地址）包含在要约本身中，并且应答者延迟其应答，直到这些对验证。

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

当 ICE 在两个对等体之间运行时，一个代理充当受控者角色，另一个充当控制者角色。 规则被定义为实现类型和要约者/应答者的函数，以确定谁在控制和谁被控制。但是，规范中提到，在某些情况下，双方可能都认为自己在控制，或者双方都认为自己受到控制。 这怎么可能发生？

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

该流程是 RFC 3725 [[RFC3725](https://www.rfc-editor.org/rfc/rfc8839#RFC3725)] 流程 III 的变体。 事实上，它比流 III 工作得更好，因为它产生的消息更少。在这个流程中，控制器向代理 A 发送一个无要约的邀请 INVITE，代理 A 以它的要约 SDP1 进行响应。然后，该代理向代理 B 发送无要约邀请 INVITE，代理 B 以它的要约 SDP2 响应该邀请 INVITE。然后控制器使用来自每个代理的要约生成应答。使用此流程时，ICE 将在代理 A 和 B 之间运行，但双方都认为自己处于控制角色。通过角色冲突解决程序，当使用 ICE 时，此流程将正常运行。

目前，没有有记录的流程可能导致两个代理都认为自己受到控制的情况。但是，冲突解决程序允许这种情况，如果出现适合此类别的流程。

# 附录 D. 为什么要发送更新的 Offer？

[[RFC8445](https://www.rfc-editor.org/rfc/rfc8839#RFC8445)] 的 [第 12.1 节](https://www.rfc-editor.org/rfc/rfc8839#RFC8445) 描述了发送媒体的规则。 一旦 ICE 检查完成，两个代理都可以发送媒体，而无需等待更新的 Offer。实际上，更新的 Offer 的唯一目的是 “更正” SDP，以便媒体的默认目的地与根据 ICE 程序发送媒体的目的地相匹配（这将是最高优先级的提名候选地址对）。

这就提出了一个问题——为什么需要更新的 offer/answer 交换？事实上，在纯粹的要约/应答环境中，它不需要。要约者和应答者通过 ICE 就使用的候选地址达成一致意见，然后可以开始使用它们。就代理本身而言，更新的要约/应答没有提供新信息。然而，在实践中，信令路径上的许多组件都会查看 SDP 信息。其中包括执行路径外 QoS 保留的实体、NAT 穿越组件（例如 ALG 和会话边界控制器 (SBC)）以及被动监控网络的诊断工具。为了让这些工具在不改变的情况下继续运行，SDP 的核心属性 —— 即现有的、ICE 之前用于媒体的地址定义 —— "m=" 和 "c=" 行以及 "rtcp" 属性 —— 必须保留。因此，必须发送更新的 offer。

# 致谢

本文档中的大部分文本来自 [[RFC5245](https://www.rfc-editor.org/rfc/rfc8839#RFC5245)]，由 Jonathan Rosenberg 撰写。

本文档中的一些文本摘自 [[RFC6336](https://www.rfc-editor.org/rfc/rfc8839#RFC6336)]，由 Magnus Westerlund 和 Colin Perkins 撰写。

非常感谢 Flemming Andreasen 的牧羊人评论反馈。

感谢以下专家的评论和建设性反馈：Thomas Stach、Adam Roach、Peter Saint-Andre、Roman Danyliw、Alissa Cooper、Benjamin Kaduk、Mirja Kühlewind、Alexey Melnikov、和 Éric Vyncke，为它们详细的评论。 

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
