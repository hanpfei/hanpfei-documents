---
title: RFC 8834 WebRTC 中的媒体传输和 RTP 的使用 II
date: 2022-03-08 22:35:49
categories: 网络协议
tags:
- 网络协议
---

# 8. WebRTC 对 RTP 的使用：性能监控

如 [第 4.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp) 所述，实现必需 (**REQUIRED**) 生成与它们发送和接收的 RTP 数据包流相关的 RTCP 发送方报告 (SR) 和接收方报告 (RR) 数据包。这些 RTCP 报告可用于性能监控，因为它们包括基本的丢包和抖动统计。

RTCP Extended Reports (XR) 框架支持大量额外的性能指标；参见 [[RFC3611](https://www.rfc-editor.org/rfc/rfc8834#RFC3611)] 和 [[RFC6792](https://www.rfc-editor.org/rfc/rfc8834#RFC6792)]。在撰写本文时，尚不清楚哪些扩展指标适合在 WebRTC 中使用，因此不要求实现生成 RTCP XR 数据包。然而，可以使用详细的性能监控数据的实现可以 (**MAY**) 适当地生成 RTCP XR 数据包。应该声明 RTCP XR 数据包的使用；实现必须 (**MUST**) 忽略意外或无法理解的 RTCP XR 数据包。

# 9. WebRTC 对 RTP 的使用：未来的扩展

本备忘录中描述的 RTP 协议和 RTP 扩展的核心集合可能不足以满足 WebRTC 的未来需求。在这种情况下，对本备忘录的未来更新必需遵循 ["RTP 有效负载格式规范编写者指南"](https://www.rfc-editor.org/rfc/rfc8834#RFC2736) [[RFC2736](https://www.rfc-editor.org/rfc/rfc8834#RFC2736)]，["如何编写一份 RTP 有效负载格式"](https://www.rfc-editor.org/rfc/rfc8834#RFC8088) [[RFC8088](https://www.rfc-editor.org/rfc/rfc8834#RFC8088)]，和 ["扩展 RTP 控制协议 (RTCP) 指南"](https://www.rfc-editor.org/rfc/rfc8834#RFC5968) [[RFC5968](https://www.rfc-editor.org/rfc/rfc8834#RFC5968)]。他们还应该 (**SHOULD**) 考虑任何未来扩展 RTP 和已开发的相关协议的指南。

敦促未来扩展的作者在推荐扩展时考虑使用 RTP 的广泛环境，因为在某些情况下适用的扩展在其它情况下可能会出现问题。在可能的情况下，WebRTC 框架将采用通用的 RTP 扩展，以便使用 RTP 轻松实现通往其它应用程序的网关，而不是采用狭隘地针对特定 WebRTC 用例的机制。

# 10. 信令注意事项

RTP 基于这样的假设建立：存在外部信令通道，并且可用于配置 RTP 会话及其特性。RTP 会话的基本配置包括以下参数：

RTP 配置文件 (profile)：将在会话中使用的 RTP 配置文件的名称。[RTP/AVP](https://www.rfc-editor.org/rfc/rfc8834#RFC3551) [[RFC3551](https://www.rfc-editor.org/rfc/rfc8834#RFC3551)] 和 [RTP/AVPF](https://www.rfc-editor.org/rfc/rfc8834#RFC4585) [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 配置文件可以在基本级别上互操作，它们的安全变体 [RTP/SAVP](https://www.rfc-editor.org/rfc/rfc8834#RFC3711) [[RFC3711](https://www.rfc-editor.org/rfc/rfc8834#RFC3711)] 和 [RTP/SAVPF](https://www.rfc-editor.org/rfc/rfc8834#RFC5124) [[RFC5124](https://www.rfc-editor.org/rfc/rfc8834#RFC5124)] 也可以互操作。配置文件的安全变体不直接与非安全变体互操作，因为在 SRTP 数据包中存在用于身份验证的附加头部字段和有效负载的加密转换。WebRTC 要求对 RTP/SAVPF 配置文件的使用必须 (**MUST**) 经过声明。互通功能可以通过向 WebRTC 端点指示使用 RTP/SAVPF 并配置 4 秒的 "trr-int" 值，将其转换为旧用例的 RTP/SAVP 配置文件。

传输信息：必须 (**MUST**) 为每个 RTP 会话声明 RTP 和 RTCP 的源和目的 IP 地址和端口。在 WebRTC 中，这些传输地址将通过，通知候选地址并在提名的候选地址对到达的 [交互式连接建立 (ICE)](https://www.rfc-editor.org/rfc/rfc8834#RFC8445) [[RFC8445](https://www.rfc-editor.org/rfc/rfc8834#RFC8445)] 提供。如果要使用 [RTP 和 RTCP 多路复用](https://www.rfc-editor.org/rfc/rfc8834#RFC5761) [[RFC5761](https://www.rfc-editor.org/rfc/rfc8834#RFC5761)]，以便将单个端口 —— 即传输层流 —— 用于 RTP 和 RTCP 流，则必须 (**MUST**) 进行声明（参见 [第 4.5 节](https://www.rfc-editor.org/rfc/rfc8834#sec.rtcp-mux)）。

RTP 有效负载类型、媒体格式和格式参数：必须 (**MUST**) 声明媒体类型名称（以及因此要使用的 RTP 有效负载格式）和 RTP 有效负载类型编号之间的映射关系。每个媒体类型还可以 (**MAY**) 具有大量，也必须 (**MUST**) 声明以用于配置编解码器和 RTP 有效负载格式（来自 SDP 的 "a=fmtp:" 行）的媒体类型参数。本备忘录的 [第 4.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec.codecs) 讨论对有效负载类型唯一性的要求。

RTP 扩展：必须 (**MUST**) 声明使用的任何附加 RTP 头部扩展和 RTCP 数据包类型，包括任何必要的参数。这个信令确保 WebRTC 端点的行为，特别是发送时，是可预测的和一致的。为了与通过网关连接到 WebRTC 会话的非 WebRTC 系统之间的健壮性和兼容性，实现必需 (**REQUIRED**) 忽略未知的 RTCP 数据包和 RTP 头部扩展（参见 [第 4.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp)）。

RTP 带宽：需要支持与端点交换 RTCP 带宽值。如果使用 SDP 或一些语义等价的东西，这应该 (** SHALL**) 按照 ["用于 RTP 控制协议 (RTCP) 带宽的会话描述协议 (SDP) 带宽修饰符"](https://www.rfc-editor.org/rfc/rfc8834#RFC3556) [[RFC3556](https://www.rfc-editor.org/rfc/rfc8834#RFC3556)] 中的描述来完成。这也确保了端点对 RTCP 带宽有一个共同的视图。不同端点之间 RTCP 带宽的共同视图，对于防止 RTCP 数据包时序和超时间隔的差异导致互操作性问题非常重要。

这些参数通常在 要约/应答 交换中传达的 SDP 消息中表示。RTP 不依赖于SDP 或 要约/应答 模型，但确实要求协商所有必要的参数，并提供给 RTP  实现。请注意，在 WebRTC 中，这将取决于，如何配置这些参数的信令模型和 API，但它们需要在 API 中设置，或在对等方之间明确地声明。

# 11. WebRTC API 注意事项

[WebRTC API](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC) [[W3C.WebRTC](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC)] 和 [媒体采集和流 API](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams) [[W3C.WD-mediacapture-streams](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams)] 定义并使用了 MediaStream 的概念，它由零个或多个 MediaStreamTracks 组成。MediaStreamTrack 是来自任何类型的媒体源（比如麦克风或摄像头）的单个媒体流，但概念上的源（例如音频混合或视频合成）也是可能的。MediaStreamTracks 中的 MediaStream 在播放时可能需要同步。

在 RTCPeerConnection 的上下文中，RTP 中的 MediaStreamTrack 实现由一个源数据包流组成，由 SSRC 标识，在作为 RTCPeerConnection 一部分的 RTP 会话中发送。MediaStreamTrack 还可以在同一个 RTP 会话中产生额外的数据包流，从而产生 SSRC。如果使用这样的媒体编码器，这些可以是来自与 MediaStreamTrack 相关联的源流的可伸缩编码的依赖数据包流。它们也可以是冗余数据包流；这些是在对源数据包流应用 [前向纠错](https://www.rfc-editor.org/rfc/rfc8834#sec-FEC)（[第 6.2 节](https://www.rfc-editor.org/rfc/rfc8834#sec-FEC)）或 [RTP 重传](https://www.rfc-editor.org/rfc/rfc8834#sec-rtx)（[第 6.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtx)）时创建的。

需要注意的是，同一个媒体源可以提供多个 MediaStreamTrack。由于可以对 MediaStreamTrack 应用不同的约束集合或其它参数集，因此添加到 RTCPeerConnection 的每个 MediaStreamTrack 实例应生成一个独立的源数据包流，该数据包流具有其自己关联的数据包流，因此具有不同的 SSRC。如果共享相同媒体源的不同 MediaStreamTracks 之间的源流和编码参数是相同的，则取决于应用的约束和参数。如果编码参数和约束相同，则实现可以选择仅使用一个编码流来创建不同的 RTP 数据包流。请注意，此类优化需要考虑到 MediaStreamTrack 中的一个的约束可能随时更改，这意味着编码配置可能不再相同，然后需要两个不同的编码器实例。

同一个 MediaStreamTrack 也可以包含在多个 MediaStream 中；因此，多组 MediaStreams 可能隐式需要使用相同的同步基。为确保这在所有情况下都有效，并且不会强制端点在任何正在进行的数据包流传输期间，通过更改同步基和 CNAME 来中断媒体，来自同一端点的所有 MediaStreamTrack 及其关联的 SSRC 需要使用一个 RTCPeerConnection 中的相同 CNAME 来发送。促使使用 [第 4.9 节](https://www.rfc-editor.org/rfc/rfc8834#sec-cname) 中的单个 CNAME。

> 对源自同一端点的所有 SSRC 使用相同 CNAME 的要求，不要求转发来自多个端点的流量的中间设备仅使用单个 CNAME。

不同的 CNAME 通常需要用于不同的 RTCPeerConnection 实例，如 [第 4.9 节](https://www.rfc-editor.org/rfc/rfc8834#sec-cname) 所述。具有相同 CNAME 的两个通信会话可以跨不同服务跟踪用户或设备（有关详细信息，请参阅 [[RFC8826](https://www.rfc-editor.org/rfc/rfc8834#RFC8826)] 的 [第 4.4.1 节](https://www.rfc-editor.org/rfc/rfc8826#section-4.4.1)）。Web 应用程序可以请求不同 RTCPeerConnection（在同源上下文中）使用的 NAME 相同；这允许跨不同 RTCPeerConnections 同步端点的 RTP 数据包流。

> 注意：这不会导致跟踪问题，因为匹配 CNAME 的创建取决于单个来源中的现有跟踪。

以上将强制 WebRTC 端点在一个 RTCPeerConnection 上接收 MediaStreamTrack 并将其添加为任何 RTCPeerConnection 上的传出 MediaStreamTrack 以执行流的重新同步。由于发送方需要将 CNAME 更改为它使用的 CNAME，这意味着它必须使用本地系统时钟作为同步的时基。因此，需要定义输入流的时基与系统发送的时基的相对关系。这种关系还需要监测时钟漂移和同步的可能调整。发送实体还负责对其发送的流的拥塞控制。在丢包的情况下，还需要处理传入数据的丢失。这导致观察到，最不可能导致传出源数据包流出现问题或中断的方法是完全解码的模型，包括修复，然后将媒体再次编码到传出数据包流中。这种方法的优化显然是可能的并且是特定于实现的。

WebRTC 端点必须 (**MUST**) 支持接收多个 MediaStreamTrack，其中每个不同的 MediaStreamTrack（及其相关数据包流集）使用不同的 CNAME。但是，使用不同 CNAME 接收的 MediaStreamTrack 没有定义同步。（***不需要同步？***）

> 注意：支持接收多个 CNAME 的动机是，在端点中继/转发流时，允许与任何未来实现更有效的流处理的更改的前向兼容。它还确保端点可以与某些类型的多流中间设备或不是 WebRTC 的端点进行互操作。

["JavaScript 会话建立协议 (JSEP)"](https://www.rfc-editor.org/rfc/rfc8834#RFC8829) [[RFC8829](https://www.rfc-editor.org/rfc/rfc8834#RFC8829)] 指定，WebRTC MediaStreams、MediaStreamTracks 和 SSRC 之间的绑定，按照 ["会话描述协议中的 WebRTC MediaStream 标识"](https://www.rfc-editor.org/rfc/rfc8834#RFC8830) [[RFC8830](https://www.rfc-editor.org/rfc/rfc8834#RFC8830)] 的描述完成。[MediaStream 标识 (MSID) 文档](https://www.rfc-editor.org/rfc/rfc8834#RFC8830) [[RFC8830](https://www.rfc-editor.org/rfc/rfc8834#RFC8830)] 还定义了，如何将具有未知 SSRC 的源数据包流映射到 MediaStreamTracks 和 MediaStreams。后者与处理一些遗留的互操作性的情况有关。与正确源数据包流的关联取决于数据包流使用的有效负载格式。

最后，本规范要求 WebRTC API 实现确定 [CSRC 列表](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp)（[第 4.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp)）以及 [混合器到客户端音频电平](https://www.rfc-editor.org/rfc/rfc8834#sec-mixer-to-client)（[第 5.2.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-mixer-to-client)）（如果支持）的方法；对此的基本要求将在 [第 12.2.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-media-stream-id) 中进一步讨论。

# 12. RTP 实现注意事项

以下讨论为本备忘录中描述的 RTP 特性的实现提供了一些指导。重点是 WebRTC 端点实现的角度，虽然提到了一些中间设备的行为，但这不是本备忘录的重点。

## 12.1. RTP 会话的配置和使用

WebRTC 端点将同时参与一个或多个 RTP 会话。每个 RTP 会话可以传送多个媒体源，并包含来自多个端点的媒体数据。下面概述了 WebRTC 端点可以配置和使用 RTP 会话的一些方法。

### 12.1.1. 在 RTP 会话中使用多个媒体源

RTP 是一种组通信协议，每个 RTP 会话都可能包含多个 RTP 数据包流。这可能是可取的有几个原因：

 * 多个媒体类型：
在 WebRTC 之外，通常为每种类型的媒体源使用一个 RTP 会话（例如，一个用于音频源的 RTP 会话和一个用于视频源的 RTP 会话，每个都通过不同的传输层流发送）。但是，为了减少使用的 UDP 端口的数量，WebRTC 中的默认设置是在单个 RTP 会话中发送所有类型的媒体，如 [第 4.4 节](https://www.rfc-editor.org/rfc/rfc8834#sec.session-mux) 所述，使用 RTP 和 RTCP 多路复用（[第 4.5 节](https://www.rfc-editor.org/rfc/rfc8834#sec.rtcp-mux)）进一步减少需要的 UDP 端口的数量。然后，此 RTP 会话仅使用一个双向传输层流，但将包含多个 RTP 数据包流，每个流包含不同类型的媒体。一个常见的例子可能是一个带有摄像头和麦克风的端点，它将两个 RTP 数据包流（一个视频和一个音频）发送到单个 RTP 会话中。

 * 多个采集设备：
WebRTC 端点可能具有多个摄像头、麦克风或其它媒体采集设备，因此它可能希望生成多个相同媒体类型的 RTP 数据包流。或者，它也可能希望一次以多种不同格式或质量设置，发送来自于单个采集设备的媒体。两者都可能导致单个端点同时将相同媒体类型的多个 RTP 数据包流发送到单个 RTP 会话中。

 * 相关的修复数据：
一个端点可能会发送一个与另一个流相关联的 RTP 数据包流。例如，它可能会发送包含 FEC 或与另一个流相关的重传数据的 RTP 数据包流。一些 RTP 有效负载格式将这种相关的修复数据作为源数据包流的一部分发送，而另一些则将其作为单独的数据包流发送。

 * 分层或多描述编码：
单个 RTP 会话中，端点可以使用分层媒体编解码器 —— 例如，H.264 可扩展视频编码 (SVC) —— 或生成多个 RTP 数据包流的多描述编解码器，每个数据包流具有不同的 RTP SSRC。

 * RTP 混合器、转换器和其它中间设备：
在 WebRTC 上下文中，RTP 会话是端点和其它一些对等设备之间的点对点关联，其中这些设备共享一个公共 SSRC 空间。对等设备可能是另一个 WebRTC 端点，也可能是 RTP 混合器、转换器或其它形式的媒体处理中间设备。在后一种情况下，中间设备可能会发送来自于多个参与者的混合的或中继的 RTP 流，WebRTC 端点将需要渲染这些流。因此，即使 WebRTC 端点可能只是单个 RTP 会话的成员，对等设备也可能会扩展该 RTP 会话以与其它端点合作。WebRTC 是一个组通信环境，端点需要能够同时接收、解码和播放多个 RTP 数据包流，即使在单个 RTP 会话中也是如此。

### 12.1.2. 使用多 RTP 会话

除了在单个 RTP 会话中发送和接收多个 RTP 数据包流，WebRTC 端点还可以参与进多个 RTP 会话。WebRTC 端点可能选择这么做的原因有几个：

 * 与旧设备互操作：
非 WebRTC 世界中的常见实践是在单独的 RTP 会话中发送不同类型的媒体 ——例如，一个 RTP 会话用于音频，使用另一个单独的传输层流的另一个 RTP 会话用于视频。所有 WebRTC 端点都需要支持在不同 RTP 会话上发送不同类型媒体的选项，以便它们可以与此类旧设备互通。这在 [第 4.4 节](https://www.rfc-editor.org/rfc/rfc8834#sec.session-mux) 中有进一步讨论。

 * 提供更高的服务质量：
一些基于网络的服务质量机制在传输层流的粒度上运行。如果希望使用这些机制为某些 RTP 数据包流提供差异化的服务质量，那么这些 RTP 数据包流需要使用不同的传输层流在单独的 RTP 会话中发送，并带有适当的服务质量标记。这将在 [第 12.1.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-differentiated) 中进一步讨论。

 * 将不同用途的媒体分开：
端点可能希望在不同的 RTP 会话上发送具有不同用途的 RTP 数据包流，以便对等方设备能够轻松区分它们。例如，一些集中式多方会议系统以高分辨率显示活跃的发言人，但显示其他参与者的低分辨率 “缩略图”。此类系统可能会将端点配置为使用单独的 RTP 会话同时发送其视频的高分辨率和低分辨率版本，以简化 RTP 中间设备的操作。在 WebRTC 上下文中，目前可以通过在一个（或多个）RTCPeerConnection 中建立多个具有相同媒体源的 WebRTC MediaStreamTracks 来实现。然后将每个 MediaStreamTrack 配置为提供特定的媒体质量和媒体比特率，并且它将生成一个独立编码的版本，其中编解码器参数在该 RTCPeerConnection 的上下文中专门商定。RTP 中间设备可以通过检查，RTP 有效负载、RTP 头部扩展或 RTCP 数据包中包含的，它们的 SSRC、RTP 有效负载类型或一些其它信息，来区分对应于低分辨率和高分辨率流的数据包。但是，如果 RTP 数据包流到达不同传输层流上的不同 RTP 会话，则它们会更容易区分。

 * 直接连接多个对等点：
多方会议不需要使用 RTP 中间设备。相反，可以创建多单播网格，包括几个不同的 RTP 会话，每个参与者通过单独的 RTP 会话（即使用独立的 RTCPeerConnection 对象）向每个其他参与者发送 RTP 流量，如 [图 1](https://www.rfc-editor.org/rfc/rfc8834#fig-mesh) 所示。这种拓扑结构的好处是不需要受信任的 RTP 中间设备节点来访问和管理媒体数据。缺点是它增加了每个发送者的使用带宽，因为它需要为每个参与者提供一份 RTP 数据包流副本，该副本是发送者本身之外的同一会话的一部分。
```
+---+     +---+
| A |<--->| B |
+---+     +---+
  ^         ^
   \       /
    \     /
     v   v
     +---+
     | C |
     +---+
```
*图 1：使用多个 RTP 会话的多单播*

多单播拓扑也可以实现为单个 RTP 会话，跨越多个端到端传输层连接，或者作为在每对对等点之间一个的几个成对的 RTP 会话。为了维护 RTP 会话和 RTCPeerConnection 对象之间关系的一致映射，建议 (**RECOMMENDED**) 将其实现为几个单独的 RTP 会话。唯一的缺点是端点 A 无法了解到 B 和 C 之间发生的任何传输的质量，因为它无法看到 B 和 C 之间 RTP 会话的 RTCP 报告，而如果所有三个参与者都是单个 RTP 会话的一部分它可能希望了解到。使用 Mbone 工具（1990 年代后期基于 RTP 的实验性多播会议工具）的经验表明，接收第三方的 RTCP 质量报告，可以以帮助他们理解不对称的网络问题的方式呈现给用户，但使用单独的 RTP 会话的方法阻止了这种情况。然而，使用单独的 RTP 会话的一个优点是，它能够在不同的对等方之间使用不同的媒体比特率和 RTP 会话配置，因此如果从 A 到 C 的传输存在限制，不会迫使 B 承受与 C 相同的质量下降。相信这些优点超过了调试能力的限制。

 * 间接连接多个对等点：
多方会议中的一个常见场景是使用 RTP 混合器、转换器或其它类型的 RTP 中间设备创建与多个对等方的间接连接。[图 2](https://www.rfc-editor.org/rfc/rfc8834#fig-mixerFirst) 概述了可能用于四人集中式会议的简单拓扑。中间设备的作用是，从某些角度优化 RTP 数据包流的传输，或者通过仅将一些接收到的 RTP 数据包流发送到任何给定的接收方，或者通过从一组贡献流中提供组合的 RTP 数据包流。
```
+---+      +-------------+      +---+
| A |<---->|             |<---->| B |
+---+      | RTP mixer,  |      +---+
           | translator, |
           | or other    |
+---+      | middlebox   |      +---+
| C |<---->|             |<---->| D |
+---+      +-------------+      +---+
```
*图 2：只有单播路径的 RTP 混合器*

中间设备有多种实现方法。如果实现为标准 RTP 混合器或转换器，则单个 RTP 会话将跨越中间设备并包含一个多方会话中的所有端点。其它类型的中间设备可以在每个端点和中间设备之间使用单独的 RTP 会话。一个共同的方面是这些 RTP 中间设备可以使用许多工具来控制 WebRTC 端点提供的媒体编码。这包括像请求中断编码链和让编码器产生所谓的帧内帧等功能。另一个共同的方面是限制流的比特率以更好地匹配混合输出。其它方面是控制最合适的帧率、画面分辨率，以及帧率和空间质量之间的权衡。中间设备负责正确执行拥塞控制、识别源和管理同步，同时为应用程序提供合适的媒体优化。就安全性而言，中间设备也必须是一个受信任的节点，因为它会在将从一个端点接收到的媒体数据发送给其它端点之前，操纵它们的 RTP 包头或媒体本身（或两者）；因此，它们需要能够在发送 RTP 数据包流之前对其进行解密然后重新加密。

混合器被期待不转发有关穿过它们的 RTP 数据包流的 RTCP 报告。这是由于提供给不同端点的 RTP 数据包流之间的差异。原始媒体源在将媒体数据发送到不同的接收器之前缺少有关混合器处理的信息。这种情况还会导致端点的反馈或请求进入混合器。当混合器无法自行处理时，它会被迫去原始媒体源来满足接收者的请求。这对任何 RTP 和 RTCP 流量不一定是明确可见的，但交互和完成它们的时间将表明这种依赖关系。

在多方场景中提供源身份验证是一项挑战。在基于混合器的拓扑中，端点源认证首先基于，通过密码验证来验证媒体是否来自混合器，其次，信任混合器已正确标识朝向端点的任何源。在多个端点对一个端点直接可见的 RTP 会话中，所有端点都将了解彼此的主密钥，因此可以在会话中注入声称来自另一个端点的数据包。任何执行中继的节点，都可以通过阻止转发之前来自其它端点的具有 SSRC 字段的数据包，来执行非加密缓解。对于源的加密验证，SRTP 将需要额外的安全机制 —— 例如，SRTP [[RFC4383](https://www.rfc-editor.org/rfc/rfc8834#RFC4383)] 的 [定时高效流丢失容忍认证 (TESLA)](https://www.rfc-editor.org/rfc/rfc8834#RFC4383) —— 这不是基本 WebRTC 标准的一部分。

 * 在多个对等方之间转发媒体：

有时希望接收到 RTP 数据包流的端点能够将该 RTP 数据包流转发给第三方。支持这一点有一些明显的安全和隐私影响，但也有潜在用途。这在 W3C API 中得到支持，方法是获取接收和解码的媒体，并将其用作重新编码并作为新流传输的媒体源。

在 RTP 层，媒体转发充当背靠背的 RTP 接收器和 RTP 发送器。接收端终止 RTP 会话并对媒体进行解码，而发送端使用完全独立的 RTP 会话重新编码和传输媒体。原始发送者只会看到媒体的单个接收者，并且无法根据 RTP 层信息判断正在发生转发，因为用于发送转发媒体的 RTP 会话未连接到正在进行转发的节点接收媒体所在的 RTP 会话。

执行转发的端点负责产生适合向前传输的 RTP 数据包流。用于发送转发媒体的传出 RTP 会话与接收媒体的 RTP 会话完全分开。这将需要出于拥塞控制目的而进行的媒体转码，以便为传出 RTP 会话生成合适的比特率，从而降低媒体质量并迫使转发端点将资源用于转码。媒体转码确实导致两个不同分支的分离，消除了几乎所有的依赖关系，并允许转发端点优化其媒体转码操作。代价是大大增加了转发节点上的计算复杂度。转发流的接收者将转发设备视为流的发送者，并且无法从 RTP 层得知它们正在接收转发的流，而不是转发设备生成的全新 RTP 数据包流。

### 12.1.3. RTP 流的差异化处理

有需要 RTP 数据包流的差异化处理的用例。这种差异可能发生在系统中的几个地方。首先是发送媒体的端点内的优先级，它控制将发送哪些 RTP 数据包流以及它们在当前可用聚合中的比特率分配，由拥塞控制确定。

预计 [WebRTC API](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC) [[W3C.WebRTC](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC)] 将允许应用程序指示不同 MediaStreamTrack 的相对优先级。然后，这些优先级可用于影响本地 RTP 处理，尤其是在确定如何在 RTP 数据包流之间划分可用带宽以实现拥塞控制时。对于与主 RTP 数据包流相关联的 RTP 数据包流，例如 RTP 重传和 FEC 的冗余流，也需要考虑相对优先级的任何变化。这种冗余 RTP 数据包流的重要性取决于所使用的媒体类型和编解码器，以及该编解码器对数据包丢失的鲁棒性。但是，默认策略可能是对冗余 RTP 数据包流使用与源 RTP 数据包流相同的优先级。

其次，网络可以为传输层流和子流分优先级，包括 RTP 数据包流。通常，差异化处理包括两个步骤，第一是识别 IP 数据包是否属于必须被差异化处理的类别，第二个是确定数据包优先级的实际机制。IP 数据包分类的三种常用方法是：

DiffServ：端点使用 DiffServ 代码点标记数据包，以向网络表明该数据包属于特定类别。

基于流：需要给予特定处理的数据包使用 IP 和端口地址的组合来标识。

深度包检测：网络分类器 (DPI) 检查数据包并尝试确定数据包是否代表要优先处理的特定应用程序和类型。

基于流的区分将对传输层流中的所有数据包提供相同的处理，即，相对优先级是不可能的。此外，如果资源有限，与 WebRTC 会话中使用的所有 RTP 数据包流的尽力服务相比，可能无法提供差异化处理。需要在 WebRTC 系统和网络之间协调使用基于流的差异化。WebRTC 端点需要知道基于流的区分可能用于将 RTP 数据包流分离到不同的 UDP 流上，以便更精细地使用基于流的区分。使用的流，它们的 5 元组和优先级将需要与网络沟通，以便它可以正确识别流以启用优先级。没有特定的协议支持描述这些信息。

DiffServ 假设端点或分类器可以使用适当的 差异化服务编码点 (DSCP)  标记数据包，以便根据该标记处理数据包。如果端点要标记流量，WebRTC 上下文中会出现两个要求：1) WebRTC 端点必须知道要使用哪些 DSCP，并且知道它可以在某些 RTP 数据包流集上使用它们。2) 传输数据包时需要传递给操作系统的信息。此过程的详细信息超出了本备忘录的范围，并在 [“WebRTC QoS 的差分服务代码点 (DSCP) 数据包标记”](https://www.rfc-editor.org/rfc/rfc8834#RFC8837) [[RFC8837](https://www.rfc-editor.org/rfc/rfc8834#RFC8837)] 中进一步讨论。

尽管使用了 SRTP 媒体加密，深度数据包检查器仍然能够对 RTP 流进行分类。原因是 SRTP 未加密 RTP 包头的前 12 个字节。这样就可以使用 SSRC 轻松识别 RTP 流，并为分类器提供有用的信息，这些信息可以相互关联以确定，例如，流的媒体类型。使用数据包大小、接收时间、数据包间距、RTP 时间戳增量和序列号，可以实现相当可靠的分类。

对于基于数据包的标记方案，可以根据 RTP 有效负载的相对优先级对单个 RTP 数据包进行不同的标记。例如，具有 I、P 和 B 图像的视频编解码器，可以优先考虑仅携带较少 B 帧的任何有效载荷，因为这些有效载荷丢失的破坏性较小。但是，依赖于 QoS 机制和应用的标记，这不仅会导致不同的丢包概率，还会导致数据包重新排序；参见 [[RFC8837](https://www.rfc-editor.org/rfc/rfc8834#RFC8837)] 和 [[RFC7657](https://www.rfc-editor.org/rfc/rfc8834#RFC7657)] 的进一步讨论。默认策略是，应该为与一个 RTP 数据包流相关的所有 RTP 数据包提供相同的优先级；每个数据包的优先级不在本备忘录的范围内，但将来可能会在其它地方指定。

考虑如何标记与特定 RTP 数据包流相关的 RTCP 数据包也很重要。带有 发送端报告 (SR) 的 RTCP 复合数据包应该被标记为，与 RTP 数据包流本身相同的优先级，因此基于 RTCP 的往返时间 (RTT) 测量使用与 RTP 数据包流体验相同的传输层流优先级完成。包含 RR 数据包的 RTCP 复合数据包应该以报告的大多数 RTP 数据包流使用的优先级发送。包含时序要求严格的反馈数据包的 RTCP 数据包可以使用更高的优先级，来提高传递此类反馈的及时性和可能性。

## 12.2. 媒体源、RTP 流和参与者标识

### 12.2.1. 媒体源标识

每个 RTP 数据包流由唯一的同步源 (SSRC) 标识符标识。SSRC 标识符包含在构成一个 RTP 数据包流的每个 RTP 数据包中，并且还用于在相应的 RTCP 报告中标识该流。SSRC 的选择如 [第 4.8 节](https://www.rfc-editor.org/rfc/rfc8834#sec-ssrc) 所述。在 WebRTC 端点，对在单个传输层流上接收到的 RTP 和 RTCP 数据包进行解复用的第一阶段是，基于它们的 SSRC 值分离 RTP 数据包流；一旦完成，额外的解复用步骤可以确定如何以及在何处渲染媒体。

RTP 允许混合器或其它 RTP 层中间设备组合来自多个媒体源的编码流，以形成来自新媒体源（混合器）的新编码流。新 RTP 数据包流中的 RTP 数据包可以包含贡献源 (CSRC) 列表，指示哪些原始 SSRC 对组合源流做出了贡献。如 [第 4.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp) 所述，实现需要支持接收包含 CSRC 列表的 RTP 数据包和与 CSRC 列表中存在的源相关的 RTCP 数据包。根据正在执行的混合操作，CSRC 列表可以逐包更改。如果用户界面指示在会话中哪些参与者处于活动状态，则了解哪些媒体源对特定 RTP 数据包有贡献可能很重要。如果应用程序要能够跟踪会话参与的变化，则需要使用某些 API 将数据包中包含的 CSRC 列表中的更改暴露给 WebRTC 应用程序。当它们穿过这些 API 时，最好将它们映射回 WebRTC MediaStream 标识，以避免将 SSRC/CSRC 命名空间暴露给 WebRTC 应用程序。

如果在会话中使用了混音器到客户端的音频电平扩展 [[RFC6465](https://www.rfc-editor.org/rfc/rfc8834#RFC6465)]（参见 [第 5.2.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-mixer-to-client)），则 CSRC 列表中的信息将通过每个贡献源的音频电平信息进行增强。在将 CSRC 值映射到 WebRTC MediaStream 标识之后，最好使用一些 API 将此信息暴露给 WebRTC 应用程序，以便可以在用户界面中公开。

### 12.2.2. SSRC 碰撞检测

RTP 标准要求 RTP 实现支持检测和处理 SSRC 冲突 —— 即，当两个不同的端点使用相同的 SSRC 值时能够解决冲突（参见 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 8.2 节](https://www.rfc-editor.org/rfc/rfc3550#section-8.2)）。这个要求也适用于 WebRTC 端点。有几种情况会发生 SSRC 碰撞：

 * 在每个 SSRC 与两个端点中的任何一个相关联的点对点会话中，承载媒体的主要 SSRC 标识符将在信令信道中公布，由于有关使用的 SSRC 的信息，不太可能发生冲突。如果使用 SDP，则此信息由 [源特定的 SDP 属性](https://www.rfc-editor.org/rfc/rfc8834#RFC5576) [[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 提供。尽管如此，如果两个端点在将新的 SSRC 标识符发送给对等方并在信令消息上收到确认之前就开始使用新的 SSRC 标识符，则可能会发生冲突。[“会话描述协议 (SDP) 中的源特定媒体属性”](https://www.rfc-editor.org/rfc/rfc8834#RFC5576) [[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 包含一种机制来指示端点如何解决 SSRC 冲突。

 * 未发送信号通知的 SSRC 值也可能出现在 RTP 会话中。这比看起来更有可能，因为一些 RTP 功能使用额外的 SSRC 来提供它们的功能。例如，重传数据可能使用单独的 RTP 数据包流传输，该数据包流需要自己的 SSRC，与源 RTP 数据包流 [[RFC4588](https://www.rfc-editor.org/rfc/rfc8834#RFC4588)] 的 SSRC 分开。在这些情况下，端点可以创建一个新的 SSRC，该 SSRC 严格不需要通过信令通道进行通告，以便在 RTP 和 RTCPeerConnection 级别上正常运行。

 * 多方会议中的多个端点可以创建新的源并向 RTP 中间设备发送信号。在 SSRC/CSRC 在 RTP 中间设备的不同端点之间传播的情况下，可能会发生冲突。

 * RTP 中间设备可以将端点的 RTCPeerConnection 连接到来自同一端点的另一个 RTCPeerConnection，从而形成一个环，其中端点将接收自己的流量。虽然它显然被认为是一个缺陷，但重要的是端点能够在它发生时识别和处理这种情况。当涉及媒体混合器等时，这种情况变得更加成问题，其中接收到的流是不同的流，但仍包含该客户端的输入。

这些 SSRC/CSRC 冲突只有在 RTP 中间设备跨多个 RTCPeerConnection 扩展同一个 RTP 会话时才能在 RTP 层处理。为了解决多个 RTCPeerConnection 互连的更一般的情况，需要在这些互连中保留，作为正在跨多个互联的 RTCPeerConnection 传播的 MediaStreamTrack 的一部分的媒体源的标识。

### 12.2.3. 媒体同步上下文

当端点从多个媒体源发送媒体时，它需要考虑是否（以及哪些）这些媒体源要同步。在 RTP/RTCP 中，通过使用相同的 RTCP CNAME 标识符将一组 RTP 数据包流指示为来自相同的同步上下文和逻辑端点来提供同步。

下一个规定是所有媒体源的内部时钟 —— 即驱动 RTP 时间戳的那个 ——可以与以 NTP 格式编码的 RTCP Sender Reports 中提供的系统时钟相关联。通过将所有 RTP 时间戳与所有源的公共系统时钟相关联，可以在接收端处导出不同 RTP 数据包流（也跨多个 RTP 会话）的时序关系，并且如果需要，可以同步流。这要求媒体发送方提供相关信息；信息是否被使用取决于接收者。

# 13. 安全注意事项

WebRTC 的总体安全架构在 [[RFC8827](https://www.rfc-editor.org/rfc/rfc8834#RFC8827)] 中描述，WebRTC 框架的安全注意事项在 [[RFC8826](https://www.rfc-editor.org/rfc/rfc8834#RFC8826)] 中描述。这些注意事项也适用于本备忘录。

RTP 规范、RTP/SAVPF 配置文件以及构成本备忘录中描述的完整协议套件的各种 RTP/RTCP 扩展和 RTP 有效负载格式的安全考虑适用。相信这些不同协议扩展的组合不会产生新的安全注意事项。

[“基于实时传输控制协议 (RTCP) 的反馈 (RTP/SAVPF) 的扩展安全 RTP 配置文件”](https://www.rfc-editor.org/rfc/rfc8834#RFC5124) [[RFC5124](https://www.rfc-editor.org/rfc/rfc8834#RFC5124)] 通过提供机密性、完整性和部分源身份验证来处理基本问题。一个强制实现和使用的媒体安全解决方案是通过结合这个安全的 RTP 配置文件和 [DTLS-SRTP 密钥](https://www.rfc-editor.org/rfc/rfc8834#RFC5764) [[RFC5764](https://www.rfc-editor.org/rfc/rfc8834#RFC5764)] 来创建的，如 [[RFC8827](https://www.rfc-editor.org/rfc/rfc8834#RFC8827)] 的 [第 5.5 节](https://www.rfc-editor.org/rfc/rfc8827#section-5.5) 所定义。

RTCP 数据包传送规范名称 (CNAME) 标识符，用于关联需要跨相关 RTP 会话同步的 RTP 数据包流。不恰当的 CNAME 值选择可能会引起隐私问题，因为可以使用长期持久的 CNAME 标识符来跨多个 WebRTC 呼叫跟踪用户。本备忘录的 [第 4.9 节](https://www.rfc-editor.org/rfc/rfc8834#sec-cname) 要求生成短期固定的 RTCP CNAMES，如 RFC 7022 中所指定的那样，导致无法追踪的 CNAME 值可以减轻这种风险。

如果 RTCP 报告间隔配置为不适当的值，则存在一些潜在的拒绝服务攻击。这可以通过使用 SDP "b=RR:" 或 "b=RS:" 行 [[RFC3556](https://www.rfc-editor.org/rfc/rfc8834#RFC3556)] 或一些类似机制将 RTCP 带宽分数配置为过大或过小值来完成，或者通过选择过大或过小 RTP/AVPF 最小接收者报告间隔的值（如果使用 SDP，这是 "a=rtcp-fb:... trr-int" 参数）来完成 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)]。风险如下：

1. RTCP 带宽可以配置为使定期报告间隔过大，以至于无法维持有效的拥塞控制，可能会由于媒体流量引起的拥塞而导致拒绝服务；

2. RTCP 间隔可以配置为非常小的值，导致端点生成高速 RTCP 流量，可能由于 RTCP 流量没有受到拥塞控制而导致拒绝服务；以及

3. 可以为每个端点配置不同的 RTCP 参数，其中一些端点使用较大的报告间隔，而一些端点使用较小的间隔，由于基于报告间隔的超时期限不匹配而导致参与者过早超时，从而导致拒绝服务。如果端点为 RTP/AVPF 最小接收者报告间隔 (trr-int) [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 使用一个很小但非 0 的值，这是一个特别关注的问题，如 [[RFC8108](https://www.rfc-editor.org/rfc/rfc8834#RFC8108)] 的 [第 6.1 节](https://www.rfc-editor.org/rfc/rfc8108#section-6.1) 所述。

在计算参与者超时时，可以通过使用固定（非减少）最小间隔来避免参与者过早超时（参见本备忘录的 [第 4.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-rtcp) 和 [[RFC8108](https://www.rfc-editor.org/rfc/rfc8834#RFC8108)] 的 [第 7.1.2 节](https://www.rfc-editor.org/rfc/rfc8108#section-7.1.2)）。为了解决其它问题，端点 应该  (**SHOULD**) 忽略将 RTCP 报告间隔配置为明显长于 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 中指定的默认 5 秒间隔的参数（除非媒体数据速率太低以至于较长的报告间隔大致对应于 5 % 的媒体数据速率），或者将 RTCP 报告间隔配置得足够小，以至于 RTCP 带宽将超过媒体带宽的参数。

[[RFC6562](https://www.rfc-editor.org/rfc/rfc8834#RFC6562)] 中的指南适用于使用可变比特率 (VBR) 音频编解码器，例如 Opus（有关必须的音频编解码器的讨论，请参见 [第 4.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec.codecs)）。[[RFC6562](https://www.rfc-editor.org/rfc/rfc8834#RFC6562)] 中的指南也适用于，但重要性较低，当使用客户端到混合器音频电平头部扩展（[第 5.2.2 节](https://www.rfc-editor.org/rfc/rfc8834#sec-client-to-mixer)）或混合器到客户端音频电平头部扩展（[第 5.2.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-mixer-to-client)）时。建议 (**RECOMMENDED**) 使用头部扩展的加密，除非有已知原因，例如 RTP 中间设备执行基于语音活动的源选择或第三方监控，它们将极大地受益于这些信息，这已使用 API 或发信号表达过了。如果有进一步的证据表明音频电平指示的信息泄漏很严重，那么此时需要强制使用加密。

在使用 RTP 中间设备的多方通信场景中，很多信任都放在这些中间设备上，以保护会话的安全性。中间设备需要维护机密性和完整性，并执行源身份验证。如 [第 12.1.1 节](https://www.rfc-editor.org/rfc/rfc8834#sec.multiple-flows) 所述，中间设备可以执行检查以防止任何参与会议的端点冒充另一个端点。关于多方拓扑的一些额外安全注意事项可以在 [[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 中找到。

# 14. IANA 注意事项

本文档没有 IANA 操作。

# 15. 参考引用

## 15.1. 规范性参考 (Normative References)

```
   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <https://www.rfc-editor.org/info/rfc2119>.

   [RFC2736]  Handley, M. and C. Perkins, "Guidelines for Writers of RTP
              Payload Format Specifications", BCP 36, RFC 2736,
              DOI 10.17487/RFC2736, December 1999,
              <https://www.rfc-editor.org/info/rfc2736>.

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
              Jacobson, "RTP: A Transport Protocol for Real-Time
              Applications", STD 64, RFC 3550, DOI 10.17487/RFC3550,
              July 2003, <https://www.rfc-editor.org/info/rfc3550>.

   [RFC3551]  Schulzrinne, H. and S. Casner, "RTP Profile for Audio and
              Video Conferences with Minimal Control", STD 65, RFC 3551,
              DOI 10.17487/RFC3551, July 2003,
              <https://www.rfc-editor.org/info/rfc3551>.

   [RFC3556]  Casner, S., "Session Description Protocol (SDP) Bandwidth
              Modifiers for RTP Control Protocol (RTCP) Bandwidth",
              RFC 3556, DOI 10.17487/RFC3556, July 2003,
              <https://www.rfc-editor.org/info/rfc3556>.

   [RFC3711]  Baugher, M., McGrew, D., Naslund, M., Carrara, E., and K.
              Norrman, "The Secure Real-time Transport Protocol (SRTP)",
              RFC 3711, DOI 10.17487/RFC3711, March 2004,
              <https://www.rfc-editor.org/info/rfc3711>.

   [RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session
              Description Protocol", RFC 4566, DOI 10.17487/RFC4566,
              July 2006, <https://www.rfc-editor.org/info/rfc4566>.

   [RFC4585]  Ott, J., Wenger, S., Sato, N., Burmeister, C., and J. Rey,
              "Extended RTP Profile for Real-time Transport Control
              Protocol (RTCP)-Based Feedback (RTP/AVPF)", RFC 4585,
              DOI 10.17487/RFC4585, July 2006,
              <https://www.rfc-editor.org/info/rfc4585>.

   [RFC4588]  Rey, J., Leon, D., Miyazaki, A., Varsa, V., and R.
              Hakenberg, "RTP Retransmission Payload Format", RFC 4588,
              DOI 10.17487/RFC4588, July 2006,
              <https://www.rfc-editor.org/info/rfc4588>.

   [RFC4961]  Wing, D., "Symmetric RTP / RTP Control Protocol (RTCP)",
              BCP 131, RFC 4961, DOI 10.17487/RFC4961, July 2007,
              <https://www.rfc-editor.org/info/rfc4961>.

   [RFC5104]  Wenger, S., Chandra, U., Westerlund, M., and B. Burman,
              "Codec Control Messages in the RTP Audio-Visual Profile
              with Feedback (AVPF)", RFC 5104, DOI 10.17487/RFC5104,
              February 2008, <https://www.rfc-editor.org/info/rfc5104>.

   [RFC5124]  Ott, J. and E. Carrara, "Extended Secure RTP Profile for
              Real-time Transport Control Protocol (RTCP)-Based Feedback
              (RTP/SAVPF)", RFC 5124, DOI 10.17487/RFC5124, February
              2008, <https://www.rfc-editor.org/info/rfc5124>.

   [RFC5506]  Johansson, I. and M. Westerlund, "Support for Reduced-Size
              Real-Time Transport Control Protocol (RTCP): Opportunities
              and Consequences", RFC 5506, DOI 10.17487/RFC5506, April
              2009, <https://www.rfc-editor.org/info/rfc5506>.

   [RFC5761]  Perkins, C. and M. Westerlund, "Multiplexing RTP Data and
              Control Packets on a Single Port", RFC 5761,
              DOI 10.17487/RFC5761, April 2010,
              <https://www.rfc-editor.org/info/rfc5761>.

   [RFC5764]  McGrew, D. and E. Rescorla, "Datagram Transport Layer
              Security (DTLS) Extension to Establish Keys for the Secure
              Real-time Transport Protocol (SRTP)", RFC 5764,
              DOI 10.17487/RFC5764, May 2010,
              <https://www.rfc-editor.org/info/rfc5764>.

   [RFC6051]  Perkins, C. and T. Schierl, "Rapid Synchronisation of RTP
              Flows", RFC 6051, DOI 10.17487/RFC6051, November 2010,
              <https://www.rfc-editor.org/info/rfc6051>.

   [RFC6464]  Lennox, J., Ed., Ivov, E., and E. Marocco, "A Real-time
              Transport Protocol (RTP) Header Extension for Client-to-
              Mixer Audio Level Indication", RFC 6464,
              DOI 10.17487/RFC6464, December 2011,
              <https://www.rfc-editor.org/info/rfc6464>.

   [RFC6465]  Ivov, E., Ed., Marocco, E., Ed., and J. Lennox, "A Real-
              time Transport Protocol (RTP) Header Extension for Mixer-
              to-Client Audio Level Indication", RFC 6465,
              DOI 10.17487/RFC6465, December 2011,
              <https://www.rfc-editor.org/info/rfc6465>.

   [RFC6562]  Perkins, C. and JM. Valin, "Guidelines for the Use of
              Variable Bit Rate Audio with Secure RTP", RFC 6562,
              DOI 10.17487/RFC6562, March 2012,
              <https://www.rfc-editor.org/info/rfc6562>.

   [RFC6904]  Lennox, J., "Encryption of Header Extensions in the Secure
              Real-time Transport Protocol (SRTP)", RFC 6904,
              DOI 10.17487/RFC6904, April 2013,
              <https://www.rfc-editor.org/info/rfc6904>.

   [RFC7007]  Terriberry, T., "Update to Remove DVI4 from the
              Recommended Codecs for the RTP Profile for Audio and Video
              Conferences with Minimal Control (RTP/AVP)", RFC 7007,
              DOI 10.17487/RFC7007, August 2013,
              <https://www.rfc-editor.org/info/rfc7007>.

   [RFC7022]  Begen, A., Perkins, C., Wing, D., and E. Rescorla,
              "Guidelines for Choosing RTP Control Protocol (RTCP)
              Canonical Names (CNAMEs)", RFC 7022, DOI 10.17487/RFC7022,
              September 2013, <https://www.rfc-editor.org/info/rfc7022>.

   [RFC7160]  Petit-Huguenin, M. and G. Zorn, Ed., "Support for Multiple
              Clock Rates in an RTP Session", RFC 7160,
              DOI 10.17487/RFC7160, April 2014,
              <https://www.rfc-editor.org/info/rfc7160>.

   [RFC7164]  Gross, K. and R. Brandenburg, "RTP and Leap Seconds",
              RFC 7164, DOI 10.17487/RFC7164, March 2014,
              <https://www.rfc-editor.org/info/rfc7164>.

   [RFC7742]  Roach, A.B., "WebRTC Video Processing and Codec
              Requirements", RFC 7742, DOI 10.17487/RFC7742, March 2016,
              <https://www.rfc-editor.org/info/rfc7742>.

   [RFC7874]  Valin, JM. and C. Bran, "WebRTC Audio Codec and Processing
              Requirements", RFC 7874, DOI 10.17487/RFC7874, May 2016,
              <https://www.rfc-editor.org/info/rfc7874>.

   [RFC8083]  Perkins, C. and V. Singh, "Multimedia Congestion Control:
              Circuit Breakers for Unicast RTP Sessions", RFC 8083,
              DOI 10.17487/RFC8083, March 2017,
              <https://www.rfc-editor.org/info/rfc8083>.

   [RFC8108]  Lennox, J., Westerlund, M., Wu, Q., and C. Perkins,
              "Sending Multiple RTP Streams in a Single RTP Session",
              RFC 8108, DOI 10.17487/RFC8108, March 2017,
              <https://www.rfc-editor.org/info/rfc8108>.

   [RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
              2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
              May 2017, <https://www.rfc-editor.org/info/rfc8174>.

   [RFC8285]  Singer, D., Desineni, H., and R. Even, Ed., "A General
              Mechanism for RTP Header Extensions", RFC 8285,
              DOI 10.17487/RFC8285, October 2017,
              <https://www.rfc-editor.org/info/rfc8285>.

   [RFC8825]  Alvestrand, H., "Overview: Real-Time Protocols for
              Browser-Based Applications", RFC 8825,
              DOI 10.17487/RFC8825, January 2021,
              <https://www.rfc-editor.org/info/rfc8825>.

   [RFC8826]  Rescorla, E., "Security Considerations for WebRTC",
              RFC 8826, DOI 10.17487/RFC8826, January 2021,
              <https://www.rfc-editor.org/info/rfc8826>.

   [RFC8827]  Rescorla, E., "WebRTC Security Architecture", RFC 8827,
              DOI 10.17487/RFC8827, January 2021,
              <https://www.rfc-editor.org/info/rfc8827>.

   [RFC8843]  Holmberg, C., Alvestrand, H., and C. Jennings,
              "Negotiating Media Multiplexing Using the Session
              Description Protocol (SDP)", RFC 8843,
              DOI 10.17487/RFC8843, January 2021,
              <https://www.rfc-editor.org/info/rfc8843>.

   [RFC8854]  Uberti, J., "WebRTC Forward Error Correction
              Requirements", RFC 8854, DOI 10.17487/RFC8854, January
              2021, <https://www.rfc-editor.org/info/rfc8854>.

   [RFC8858]  Holmberg, C., "Indicating Exclusive Support of RTP and RTP
              Control Protocol (RTCP) Multiplexing Using the Session
              Description Protocol (SDP)", RFC 8858,
              DOI 10.17487/RFC8858, January 2021,
              <https://www.rfc-editor.org/info/rfc8858>.

   [RFC8860]  Westerlund, M., Perkins, C., and J. Lennox, "Sending
              Multiple Types of Media in a Single RTP Session",
              RFC 8860, DOI 10.17487/RFC8860, January 2021,
              <https://www.rfc-editor.org/info/rfc8860>.

   [RFC8861]  Lennox, J., Westerlund, M., Wu, Q., and C. Perkins,
              "Sending Multiple RTP Streams in a Single RTP Session:
              Grouping RTP Control Protocol (RTCP) Reception Statistics
              and Other Feedback", RFC 8861, DOI 10.17487/RFC8861,
              January 2021, <https://www.rfc-editor.org/info/rfc8861>.

   [W3C.WD-mediacapture-streams]
              Jennings, C., Aboba, B., Bruaroey, J-I., and H. Boström,
              "Media Capture and Streams", W3C Candidate Recommendation,
              <https://www.w3.org/TR/mediacapture-streams/>.

   [W3C.WebRTC]
              Jennings, C., Boström, H., and J-I. Bruaroey, "WebRTC 1.0:
              Real-time Communication Between Browsers", W3C Proposed
              Recommendation, <https://www.w3.org/TR/webrtc/>.
```

## 15.2. 参考资料 (Informative References)

```
   [RFC3611]  Friedman, T., Ed., Caceres, R., Ed., and A. Clark, Ed.,
              "RTP Control Protocol Extended Reports (RTCP XR)",
              RFC 3611, DOI 10.17487/RFC3611, November 2003,
              <https://www.rfc-editor.org/info/rfc3611>.

   [RFC4383]  Baugher, M. and E. Carrara, "The Use of Timed Efficient
              Stream Loss-Tolerant Authentication (TESLA) in the Secure
              Real-time Transport Protocol (SRTP)", RFC 4383,
              DOI 10.17487/RFC4383, February 2006,
              <https://www.rfc-editor.org/info/rfc4383>.

   [RFC5576]  Lennox, J., Ott, J., and T. Schierl, "Source-Specific
              Media Attributes in the Session Description Protocol
              (SDP)", RFC 5576, DOI 10.17487/RFC5576, June 2009,
              <https://www.rfc-editor.org/info/rfc5576>.

   [RFC5968]  Ott, J. and C. Perkins, "Guidelines for Extending the RTP
              Control Protocol (RTCP)", RFC 5968, DOI 10.17487/RFC5968,
              September 2010, <https://www.rfc-editor.org/info/rfc5968>.

   [RFC6263]  Marjou, X. and A. Sollaud, "Application Mechanism for
              Keeping Alive the NAT Mappings Associated with RTP / RTP
              Control Protocol (RTCP) Flows", RFC 6263,
              DOI 10.17487/RFC6263, June 2011,
              <https://www.rfc-editor.org/info/rfc6263>.

   [RFC6792]  Wu, Q., Ed., Hunt, G., and P. Arden, "Guidelines for Use
              of the RTP Monitoring Framework", RFC 6792,
              DOI 10.17487/RFC6792, November 2012,
              <https://www.rfc-editor.org/info/rfc6792>.

   [RFC7478]  Holmberg, C., Hakansson, S., and G. Eriksson, "Web Real-
              Time Communication Use Cases and Requirements", RFC 7478,
              DOI 10.17487/RFC7478, March 2015,
              <https://www.rfc-editor.org/info/rfc7478>.

   [RFC7656]  Lennox, J., Gross, K., Nandakumar, S., Salgueiro, G., and
              B. Burman, Ed., "A Taxonomy of Semantics and Mechanisms
              for Real-Time Transport Protocol (RTP) Sources", RFC 7656,
              DOI 10.17487/RFC7656, November 2015,
              <https://www.rfc-editor.org/info/rfc7656>.

   [RFC7657]  Black, D., Ed. and P. Jones, "Differentiated Services
              (Diffserv) and Real-Time Communication", RFC 7657,
              DOI 10.17487/RFC7657, November 2015,
              <https://www.rfc-editor.org/info/rfc7657>.

   [RFC7667]  Westerlund, M. and S. Wenger, "RTP Topologies", RFC 7667,
              DOI 10.17487/RFC7667, November 2015,
              <https://www.rfc-editor.org/info/rfc7667>.

   [RFC8088]  Westerlund, M., "How to Write an RTP Payload Format",
              RFC 8088, DOI 10.17487/RFC8088, May 2017,
              <https://www.rfc-editor.org/info/rfc8088>.

   [RFC8445]  Keranen, A., Holmberg, C., and J. Rosenberg, "Interactive
              Connectivity Establishment (ICE): A Protocol for Network
              Address Translator (NAT) Traversal", RFC 8445,
              DOI 10.17487/RFC8445, July 2018,
              <https://www.rfc-editor.org/info/rfc8445>.

   [RFC8829]  Uberti, J., Jennings, C., and E. Rescorla, Ed.,
              "JavaScript Session Establishment Protocol (JSEP)",
              RFC 8829, DOI 10.17487/RFC8829, January 2021,
              <https://www.rfc-editor.org/info/rfc8829>.

   [RFC8830]  Alvestrand, H., "WebRTC MediaStream Identification in the
              Session Description Protocol", RFC 8830,
              DOI 10.17487/RFC8830, January 2021,
              <https://www.rfc-editor.org/info/rfc8830>.

   [RFC8836]  Jesup, R. and Z. Sarker, Ed., "Congestion Control
              Requirements for Interactive Real-Time Media", RFC 8836,
              DOI 10.17487/RFC8836, January 2021,
              <https://www.rfc-editor.org/info/rfc8836>.

   [RFC8837]  Jones, P., Dhesikan, S., Jennings, C., and D. Druta,
              "Differentiated Services Code Point (DSCP) Packet Markings
              for WebRTC QoS", RFC 8837, DOI 10.17487/RFC8837, January
              2021, <https://www.rfc-editor.org/info/rfc8837>.

   [RFC8872]  Westerlund, M., Burman, B., Perkins, C., Alvestrand, H.,
              and R. Even, "Guidelines for Using the Multiplexing
              Features of RTP to Support Multiple Media Streams",
              RFC 8872, DOI 10.17487/RFC8872, January 2021,
              <https://www.rfc-editor.org/info/rfc8872>.
```

# 致谢

作者们想要为他们感谢 Bernard Aboba、Harald Alvestrand、Cary Bran、Ben Campbell、Alissa Cooper、Spencer Dawkins、Charles Eckel、Alex Eleftheriadis、Christian Groves、Chris Inacio、Cullen Jennings、Olle Johansson、Suhas Nandakumar、Dan Romascanu、Jim Spring、Martin Thomson 和 IETF RTCWEB 工作组的其他成员提供的宝贵反馈。

# 作者的地址

   **Colin Perkins**

   University of Glasgow

   School of Computing Science

   Glasgow

   G12 8QQ

   United Kingdom

   Email: csp@csperkins.org

   URI:   https://csperkins.org/


   **Magnus Westerlund**

   Ericsson

   Torshamnsgatan 23

   SE-164 80 Kista

   Sweden

   Email: magnus.westerlund@ericsson.com


   **Jörg Ott**

   Technical University Munich

   Department of Informatics

   Chair of Connected Mobility

   Boltzmannstrasse 3

   85748 Garching

   Germany

   Email: ott@in.tum.de

原文：[RFC 8834 Media Transport and Use of RTP in WebRTC](https://www.rfc-editor.org/rfc/rfc8834)
