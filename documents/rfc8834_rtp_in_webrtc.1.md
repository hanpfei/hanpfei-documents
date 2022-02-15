---
title: RFC 8834 WebRTC 中的媒体传输和 RTP 的使用
date: 2022-02-13 12:35:49
categories: 网络协议
tags:
- 网络协议
---

# 摘要

Web 实时通信 (WebRTC) 框架为两个对等体 Web 浏览器之间，使用音频、视频、文本、协作、游戏等的直接交互式富通信提供了支持。本备忘录描述了 WebRTC 框架的媒体传输方面。它指定了在 WebRTC 的上下文中，如何使用实时传输协议 (RTP)，并给出了哪些 RTP 特性、配置文件和扩展需要支持的要求。

# 本备忘录状态

这是一份互联网标准跟踪文件。

本文档是 IETF (Internet Engineering Task Force) 的一个产品。它代表了 IETF 社区的共识。它已接受公众审查，并已获互联网工程指导小组 (IESG) 批准出版。有关互联网标准的进一步信息，请参阅 RFC 7841 的第2节。

有关此文档的当前状态、任何勘误表，以及如何提供关于它的反馈的信息，可在以下网址获得 [https://www.rfc-editor.org/info/rfc8834](https://www.rfc-editor.org/info/rfc8834)。

# 版权声明

版权所有 (c) 2021 IETF 信托和确认为文档作者的人。保留所有权利。

本文档受 BCP 78 和自本文档发布之日起生效的 IETF 信托关于 IETF 文档 ([https://trustee.ietf.org/license-info](https://trustee.ietf.org/license-info)) 的法律规定 (https://trustee.ietf.org/license-info) 的约束。请仔细审阅这些文件，因为它们描述了您在此文档方面的权利和限制。从本文档中提取的代码组件，必须包含信托法律规定的第 4.e 节中描述的简化 BSD 许可证文本，且如简化 BSD 许可证所述，不为这些代码提供任何担保。

# 1. 简介

[实时传输协议 (RTP)](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 为音频和视频电话会议数据和其它实时媒体应用的交付提供了一个框架。之前的工作已经定义了 RTP 协议，以及大量的配置文件、载荷格式、和其它扩展。当与适当的信令系统相结合时，这些构成了许多电话会议系统的基础。

Web 实时通信 (WebRTC) 框架提供了许多协议构建块，来支持两个对等体 Web 浏览器之间，使用音频、视频、协作、游戏等的直接的、交互式的实时通信。本备忘录描述了 RTP 框架是如何用于 WebRTC 上下文的。它提出了一组要由所有 WebRTC 端点实现的 RTP 特性的基线，以及用于功能增强的建议的扩展。

本备忘录指定了一个用于 WebRTC 框架内的协议，但它不限于 WebRTC 的上下文。WebRTC 框架的概述可以参考 [[RFC8825](https://www.rfc-editor.org/rfc/rfc8834#RFC8825)]。

本备忘录的结构如下。[第 2 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rationale) 概述了我们准备这份备忘录和选择这些 RTP 特性的逻辑依据。 [第 3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-terminology) 定义了术语。核心 RTP 协议的要求在 [第 4 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-core) 中描述，建议的 RTP 扩展在 [第 5 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-extn) 中描述。[第 6 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-robust) 概述了可以增加对网络问题的健壮性的机制，而 [第 7 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rate-control) 描述了拥塞控制和速率适应机制。[第 8 节](https://www.rfc-editor.org/rfc/rfc8834#sec-perf) 包含了对强制的 RTP 机制的讨论，以及对性能监控和网络管理工具的审查。[第 9 节](https://www.rfc-editor.org/rfc/rfc8834#sec-extn) 为将来将其它 RTP 和 RTP 控制协议 (RTCP) 扩展合并到这个框架提供了指南。[第 10 节](https://www.rfc-editor.org/rfc/rfc8834#sec-signalling) 描述了对信令通道的要求。[第 11 节](https://www.rfc-editor.org/rfc/rfc8834#sec-webrtc-api) 讨论了 RTP 框架的特性和 WebRTC 应用编程接口 (API) 之间的关系，[第 12 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-func) 讨论了 RTP 实现注意事项。本备忘录以 [安全注意事项](https://www.rfc-editor.org/rfc/rfc8834#sec-security)（[第 13 节](https://www.rfc-editor.org/rfc/rfc8834#sec-security)）和 [IANA 注意事项](https://www.rfc-editor.org/rfc/rfc8834#sec-iana)（[第 14 节](https://www.rfc-editor.org/rfc/rfc8834#sec-iana)）结束。

# 2. 逻辑依据

RTP 框架包含 RTP 数据传输协议、RTP 控制协议、大量的 RTP 载荷格式、配置文件和扩展。这一系列的附加组件使 RTP 可以满足各种各样的甚至是最初的协议设计者都没有预想到的需求，还可以支持许多新的媒体编码方式，但它带来的问题是，新实现要支持哪些扩展。WebRTC 框架的发展提供了一个审视这些可用的 RTP 特性和扩展的机会，并为所有的 WebRTC 端点定义了一个通用的基线 RTP 特性集。这构建在过去 20 年 RTP 发展的基础之上，以强制使用已显示出得到广泛使用的扩展，同时仍尽可能与广泛安装的 RTP 实现兼容。

本文档中没有讨论的 RTP 和 RTCP 扩展如果对新用例有益，则可以由 WebRTC 端点实现。

也可以由 WebRTC 端点实现，如果它们能在新的使用场景下收受益的话。但它们不是解决 [[RFC7478](https://www.rfc-editor.org/rfc/rfc8834#RFC7478)] 中确定的 WebRTC 的用例和要求所必需的。

尽管在本备忘录中定义的 RTP 特性和扩展的基线集合是针对 WebRTC 框架的要求的，但它有望广泛用于 RTP 的其它电话会议相关场景。特别是，这组 RTP 特性和扩展很可能适用于其它桌面或移动视频会议系统，或基于房间的高质量远程呈现应用程序。

# 3. 术语

本文档中的关键字 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"NOT RECOMMENDED"、"MAY" 和 "OPTIONAL"，当且仅当它们以大写字母出现，如此处所示时，应按照 [BCP 14](https://www.rfc-editor.org/bcp/bcp14) [[RFC2119](https://www.rfc-editor.org/rfc/rfc2119)] [[RFC8174](https://www.rfc-editor.org/rfc/rfc8174)] 的描述解释。在本备忘录中，这些关键字的小写或混合使用不应被解释为具有特殊意义。

我们定义了以下附加术语：

WebRTC MediaStream：MediaStream 的概念在  [WebRTC API](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams) [[W3C.WD-mediacapture-streams](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams)] 中由 W3C 定义。一个 MediaStream 由 0 个或多个 MediaStreamTrack 组成。

MediaStreamTrack：W3C 在  [WebRTC API](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams) [[W3C.WD-mediacapture-streams](https://www.rfc-editor.org/rfc/rfc8834#W3C.WD-mediacapture-streams)] 中定义的 MediaStream 概念的一部分。MediaStreamTrack 是一个独立的媒体流，它可以来自于任何类型的媒体源，如麦克风或摄像头等，但概念上的源也可能是诸如混音的音频或视频合成之类的。

传输层流：传输数据包的单向流，由一个特定的源 IP 地址、源端口、目标 IP 地址、目标端口和传输协议 5 元组标识。 

双向传输层流：双向传输层流是对称的传输层流。即，反方向传输层流具有一个 5 元组，其源和目的地址和端口相对于正向路径传输层流被交换过来，而传输协议相同。

本文档使用了来自于 [[RFC7656](https://www.rfc-editor.org/rfc/rfc8834#RFC7656)] 和 [[RFC8825](https://www.rfc-editor.org/rfc/rfc8834#RFC8825)] 的术语。使用的其它术语根据它们来自于 [RTP 规范](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的定义解释。特别要注意以下常用术语：RTP 流、RTP 会话和端点。

# 4. WebRTC 对 RTP 的使用：核心协议

以下部分描述了需要实现的 RTP 和 RTCP 核心特性，以及强制的 RTP 配置文件。还描述了所有 WebRTC 端点为了在今日的网路上有效运行需要实现的，提供了必要特性的核心扩展。

## 4.1. RTP 和 RTCP

必需 (**REQUIRED**) 实现 [实时传输协议 (RTP)](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 作为 WebRTC 的媒体传输协议。RTP 本身包含两部分：RTP 数据传输协议和 RTP 控制协议 (RTCP)。RTCP 是 RTP 的基本组成部分，必须 (**MUST**) 在所有 WebRTC 端点中实现和使用。

在一些功能受限的 RTP 实现中，有时会省略以下 RTP 和 RTCP 特性，但在所有 WebRTC 端点中它们是必需 (**REQUIRED**) 的：

 * 支持在单个 RTP 会话中使用多个并发的同步源 (SSRC) 值，包括支持并发地发送多个 SSRC 值的 RTP 端点，遵循 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 和 [[RFC8108](https://www.rfc-editor.org/rfc/rfc8834#RFC8108)]。可以 (**MAY**) 支持 [[RFC8861](https://www.rfc-editor.org/rfc/rfc8834#RFC8861)] 中定义的多 SSRC 会话的 RTCP 优化；如果支持，则必须 (**MUST**) 发出信号以通知这种使用。

 * 加入会话时 SSRC 的随机选择；SSRC 值的碰撞检测和解决（另见 [第 4.8 节](https://www.rfc-editor.org/rfc/rfc8834#sec-ssrc)）。

 * 支持接收由 RTP 混合器生成的包含贡献源 (CSRC) 列表的 RTP 数据包，以及与 CSRC 相关的 RTCP 数据包。

 * 在 RTCP 发送端报告中发送正确的同步信息，以允许接收者实现音画同步；关于对快速 RTP 同步扩展的支持，请参见 [第 5.2.1 节](https://www.rfc-editor.org/rfc/rfc8834#rapid-sync)。

 * 支持多个同步上下文。发送多个并发 RTP 数据包流的参与者应该  (**SHOULD**) 这样做，作为单个同步上下文的一部分，对所有流使用单个 RTCP CNAME，并允许接收者以同步的方式将流播放出来。为了与本规范的潜在未来版本兼容，或为了通过网关与非 WebRTC 设备的互操作性，接收者必须支持在 RTP 会话中由多个 RTCP CNAME 的使用表示的多个同步上下文。在某些情况下，本规范强制在发送 RTP 流时使用单个 CNAME；参见 [第 4.9 节](https://www.rfc-editor.org/rfc/rfc8834#sec-cname)。

 * 支持发送和接收 RTCP 发送者报告 (SR)，接收者报告 (RR)，源描述 (SDES)，和 BYE 数据包类型。注意，对其它 RTCP 数据包类型的支持是可选的  (**OPTIONAL**) ，除非本规范的其它部分有强制要求。注意 [RTP/SAVPF 配置文件](https://www.rfc-editor.org/rfc/rfc8834#sec-profile) ([第 4.2 节](https://www.rfc-editor.org/rfc/rfc8834#sec-profile)) 和其它 [RTCP 扩展](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-extn) ([第 5 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-extn)) 使用了其它 RTCP 数据包类型。实现了绘画描述协议 (SDP) 捆绑协商扩展的 WebRTC 端点将使用 SDP 分组框架 "mid" 属性来识别媒体流。这样的端点必须 (**MUST**) 实现 [[RFC8843](https://www.rfc-editor.org/rfc/rfc8834#RFC8843)] 中描述的 RTCP SDES 媒体标识项。

 * 支持在单个会话中包含多个端点，并支持根据会话中参与者的个数放缩 RTCP 传输间隔；支持随机化的 RTCP 传输间隔以避免 RTCP 报告的同步；支持 RTCP 定时器重新考虑 ([[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 6.3.6 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.3.6)) 和反方向重新考虑 ([[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 6.3.4 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.3.4)) 。

 * 支持将 RTCP 带宽配置为媒体带宽的一部分，以及配置分配给发送者的 RTCP 带宽的部分 —— 例如，使用 SDP "b=" 行 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8834#RFC4566)] [[RFC3556](https://www.rfc-editor.org/rfc/rfc8834#RFC3556)]。

 * 支持 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 6.2 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.2) 描述的减少的最小 RTCP 报告间隔。在使用减少的最小 RTCP 报告间隔时，当计算参与者的超时间隔时 (参见 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的第 [6.2](https://www.rfc-editor.org/rfc/rfc3550#section-6.2) 和 [6.3.5](https://www.rfc-editor.org/rfc/rfc3550#section-6.3.5) 小节)，必须 (**MUST**) 使用固定的（非减少的）最小间隔。发送初始复合 RTCP 数据包之前的延迟可以设置为 0（参见 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 6.2 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.2)，由 [[RFC8108](https://www.rfc-editor.org/rfc/rfc8834#RFC8108)] 更新）。

 * 支持不连续的传输。RTP 允许端点在任何时刻暂停和恢复传输。当恢复时，RTP 序列号将像平常一样增加 1，而 RTP 时间戳值的增加将取决于暂停持续的时长。不连续的传输最常用于某些音频载荷格式，但它不是特定于音频的，它可用于任何 RTP 载荷格式。

 * 忽略未知的 RTCP 数据包类型和 RTP 头部扩展。这是为了确保，对可能导致接收到未发送信号通知的 RTP 头部扩展或 RTCP 数据包类型的未来扩展、中间设备行为等的稳健处理。如果收到了复合 RTCP 数据包，它包含已知和未知 RTCP 数据包类型的混合，则已知数据包类型照常处理，仅丢弃未知数据包类型。

众所周知，大量的传统 RTP 实现，尤其是那些针对仅具有 IP 语音 (VoIP) 能力的系统的实现，不支持上述所有特性，甚至在某些情况下根本不支持 RTCP。建议实现在与传统的实现互操作时考虑优雅降级的要求。

其它的实现注意事项在 [第 12 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rtp-func) 中讨论。

## 4.2. RTP 配置文件的选择

特定应用领域的完整 RTP 规范需要选择 RTP 配置文件。对于 WebRTC 使用，必须 (**MUST**) 实现由 [[RFC7007](https://www.rfc-editor.org/rfc/rfc8834#RFC7007)] 扩展的，[基于 RTCP 反馈的扩展安全 RTP 配置文件 (RTP/SAVPF)](https://www.rfc-editor.org/rfc/rfc8834#RFC5124) [[RFC5124](https://www.rfc-editor.org/rfc/rfc8834#RFC5124)]。RTP/SAVPF 配置文件是基本的 [RTP/AVP 配置文件](https://www.rfc-editor.org/rfc/rfc8834#RFC3551) [[RFC3551](https://www.rfc-editor.org/rfc/rfc8834#RFC3551)]、[基于 RTCP 反馈的 RTP 配置文件 (RTP/AVPF)](https://www.rfc-editor.org/rfc/rfc8834#RFC4585) [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 和 [安全 RTP 配置文件 (RTP/SAVP)](https://www.rfc-editor.org/rfc/rfc8834#RFC3711) [[RFC3711](https://www.rfc-editor.org/rfc/rfc8834#RFC3711)] 的组合。

改进的 RTCP 定时器模型需要基于 RTCP 反馈的扩展 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)]。这允许在对事件的响应中，更灵活地传输 RTCP 数据包，而不是严格地根据带宽，这对于能够报告拥塞信号和媒体事件至关重要。这些扩展还可以节省 RTCP 带宽，如果有许多需要反馈的事件，端点通常只会使用完整的 RTCP 分配带宽。为了充分利用 [第 5.1 节](https://www.rfc-editor.org/rfc/rfc8834#conf-ext) 讨论的 RTP 会议扩展，还需要定时器规则。

> 注意：在 RTP/AVPF 配置文件中定义的增强 RTCP 定时器模型，与只实现了 RTP/AVP 或 RTP/SAVP 配置文件的遗留系统向后兼容，给定参数配置的一些限制，例如 RTCP 带宽值和 "trr‑int"。通过网关与 RTP/(S)AVP 端点互通的最终要因素是将 "trr-int" 参数设置为表示 4 秒的值；参见 [[RFC8108](https://www.rfc-editor.org/rfc/rfc8834#RFC8108)] 的 [第 7.1.3 节](https://www.rfc-editor.org/rfc/rfc8108#section-7.1.3)。

提供媒体加密、完整性保护、重放保护和有限形式的源身份验证，需要安全 RTP 配置文件扩展 [[RFC3711](https://www.rfc-editor.org/rfc/rfc8834#RFC3711)]。WebRTC 端点不得 (**MUST NOT**) 发送使用基础 RTP/AVP 配置文件或 RTP/AVPF 配置文件的数据包；它们必须 (**MUST**) 使用完整 RTP/SAVPF 配置文件来保护生成的所有 RTP 和 RTCP 数据包。换句话说，实现必须 (**MUST**) 使用 SRTP 和安全 RTCP (SRTCP)。RTP/SAVPF 配置文件必须 (**MUST**) 使用加密套件、DTLS-SRTP 保护配置文件、密钥机制和 [[RFC8827](https://www.rfc-editor.org/rfc/rfc8834#RFC8827)] 中描述的其它参数进行配置。

## 4.3. RTP 载荷格式的选择

WebRTC 端点强制实现的音频编解码器和 RTP 载荷格式在 [[RFC7874](https://www.rfc-editor.org/rfc/rfc8834#RFC7874)] 中定义。WebRTC 端点强制实现的视频编解码器和 RTP 载荷格式在 [[RFC7742](https://www.rfc-editor.org/rfc/rfc8834#RFC7742)] 中定义。WebRTC 端点可以额外实现任何其它的，已经定义了 RTP 载荷格式和相关联的信令的编解码器。

WebRTC 端点不能假设 RTP 会话中的其它参与者理解任何 RTP 载荷格式，无论有多常见。RTP 载荷类型编号和特定 RTP 载荷格式的特定配置之间的映射，必须 (**MUST**) 在可以使用那些载荷类型/格式之前达成共识。在 SDP 上下文中，这可以通过使用与 "m=" 行关联的 "a=rtpmap:" 和 "a=fmtp:" 属性，以及任何其它配置 RTP 载荷格式所需的 SDP 属性来完成。

端点可以声明支持多个 RTP 载荷格式，或单个 RTP 载荷格式的多个配置，只要每个唯一的 RTP 载荷格式配置使用不同的 RTP 载荷类型编号即可。正如 [第 4.8 节](https://www.rfc-editor.org/rfc/rfc8834#sec-ssrc) 所述，RTP 载荷类型编号有时用于将 RTP 数据包流与信令上下文关联起来。如果在每个上下文中都使用唯一的 RTP 载荷类型编号，则这种关联是可能的。比如，RTP 数据包流可以通过比较 RTP 数据包流所使用的 RTP 载荷类型编号与 SDP 的媒体部分中的 "a=rtpmap:" 行中的载荷类型，与一个 SDP "m=" 行关联。

这导致以下考虑：

> 如果基于 RTP 载荷类型将 RTP 数据包流与信令上下文关联起来，则 RTP 载荷类型编号的分配必须 (**MUST**) 是跨信令上下文唯一的。
> 
> 如果相同的 RTP 载荷格式配置用于多个上下文，则每个上下文中必须分配不同的 RTP 载荷类型编号以确保唯一性。
> 
> 如果没有把 RTP 载荷类型编号用来将 RTP 数据包流与信令上下文关联起来，则相同 RTP 载荷类型编号可以被用于表示多个上下文中完全相同的 RTP 载荷格式配置。

在单个 RTP 会话内，单个 RTP 载荷类型编号不得 (**MUST NOT**) 被分配给不同的 RTP 载荷格式，或者相同 RTP 载荷格式的不同配置（清注意，[SDP BUNDLE 组](https://www.rfc-editor.org/rfc/rfc8834#RFC8843) [[RFC8843](https://www.rfc-editor.org/rfc/rfc8834#RFC8843)] 中的 "m=" 行形成单个 RTP 会话）。

已经声明支持多个 RTP 载荷格式的端点必须 (**MUST**) 能够在任何时间接受任何那些载荷格式的数据，除非它之前已经声明关于它的解码能力的限制。如果在同一个会话中发送多种类型的媒体（例如，音频和视频），则此要求受到限制。在这种情况下，源 (SSRC) 仅限于在为该源发送的媒体类型声明的 RTP 载荷格式之间切换；参见 [第 4.4 节](https://www.rfc-editor.org/rfc/rfc8834#sec.session-mux)。为了通过更改编解码器来支持快速速率适应，RTP 不需要提前通知，会话建立期间声明的单个 SSRC 所使用的 RTP 载荷格式之间的切换。

如果执行了使用不同 RTP 时钟频率的 RTP 载荷类型之间的更改，则 RTP 发送者必须 (**MUST**) 遵循 [[RFC7160](https://www.rfc-editor.org/rfc/rfc8834#RFC7160)] 的 [第 4.1 节](https://www.rfc-editor.org/rfc/rfc7160#section-4.1) 中的建议。RTP 接收者必须 (**MUST**) 遵循 [[RFC7160](https://www.rfc-editor.org/rfc/rfc8834#RFC7160)] 的 [第 4.3 节](https://www.rfc-editor.org/rfc/rfc7160#section-4.3) 中的建议，以支持 RTP 会话中在时钟频率间切换的源。这些对接收者的建议，与发送者仅使用单一时钟频率的情况向后兼容。

## 4.4. RTP 会话的使用

使用 RTP 进行通信的一组端点之间的关联称为 RTP 会话 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)]。一个端点可以同时参与多个 RTP 会话。在多媒体会话中，每种类型的媒体通常都由单独的 RTP 会话中承载（例如，对音频使用一个 RTP 会话，同时对视频使用利用了不同的传输层流的单独 RTP 会话）。WebRTC 端点必需 (**REQUIRED**) 以这种方式实现对多媒体会话的支持，每个分离的 RTP 会话使用不同的传输层流，以与遗留系统兼容（这有时称为会话多路复用）。

然而，在现代网络中，随着网络地址/端口转换器 (NAT/NAPT) 和防火墙的广泛使用，希望减少 RTP 应用程序使用的传输层流的数量。这可以通过在单个 RTP 会话中发送所有 RTP 数据包流来完成，该会话将包含单个传输层流。这将阻止使用某些服务质量机制，如 [第 12.1.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec-differentiated) 所述。因此，实现还必需根据 [[RFC8860](https://www.rfc-editor.org/rfc/rfc8834#RFC8860)] 支持在使用单个传输层流的单个 RTP 会话中传输所有 RTP 数据包流，与媒体类型无关（有时称为 SSRC 多路复用）。如果要在单个 RTP 会话中使用多种类型的媒体，则该 RTP 会话中的所有参与者都必须就这种用法达成一致。在一个 SDP 上下文中，[[RFC8843](https://www.rfc-editor.org/rfc/rfc8834#RFC8843)] 中所描述的机制可以被用于声明捆绑 RTP 数据包流构成单个 RTP 会话。

关于不同 RTP 会话结构和多路复用方法对于不同场景的适用性的进一步讨论，可以参考 [[RFC8872](https://www.rfc-editor.org/rfc/rfc8834#RFC8872)]。

## 4.5. RTP 和 RTCP 多路复用

从历史上看，RTP 和 RTCP 一直在不同的传输层流上运行（比如，每个 RTP 会话两个端口，一个用于 RTP，一个用于 RTCP）。随着网络地址/端口转换 (NAT/NAPT) 使用的增加，这已成为问题，因为维护多个 NAT 绑定可能会很昂贵。它还使防火墙管理复杂化，因为需要打开多个端口以允许 RTP 流量通过。为了降低这些开销和会话建立时间，实现必需 (**REQUIRED**) 支持单个传输层流之上的 RTP 数据包和 RTCP 控制数据包的多路复用 [[RFC5761](https://www.rfc-editor.org/rfc/rfc8834#RFC5761)]。这样的 RTP 和 RTCP 多路复用必须 (**MUST**) 在使用之前在信令通道中协商完成。如果信令使用了 SDP，则这种协商必须 (**MUST**) 使用 [[RFC5761](https://www.rfc-editor.org/rfc/rfc8834#RFC5761)] 中定义的机制。实现也可以支持在不同的传输层流上发送 RTP 和 RTCP，但这对于实现可选的 (**OPTIONAL**) 。如果实现不支持在不同的传输层流上发送 RTP 和 RTCP，则它必 (**MUST**) 须使用 [[RFC8858](https://www.rfc-editor.org/rfc/rfc8834#RFC8858)] 中定义的机制表明这一点。

请注意，在单个传输层流上多路复用 RTP 和 RTCP 的用法，可确保在该端口上偶尔发送流量，即使没有活动的媒体流量也是如此。这对于保持 NAT 绑定有效 [[RFC6263](https://www.rfc-editor.org/rfc/rfc8834#RFC6263)] 很有用。

## 4.6. 减小大小的 RTCP

RTCP 数据包通常作为复合 RTCP 数据包发送，[[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 要求这些复合数据包以 SR 或 RR 数据包开头。当在 RTP/AVPF 配置文件 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 下使用频繁的 RTCP 反馈消息时，不是每个数据包都需要这些统计信息的，它们会不必要地增加平均 RTCP 数据包大小。这可能会限制在 RTCP 带宽份额内，RTCP 数据包可以被发送的频率。

为了避免这个问题，[[RFC5506](https://www.rfc-editor.org/rfc/rfc8834#RFC5506)] 指定了如何减小平均 RTCP 消息大小的方法，并允许更频繁的反馈。频繁的反馈，反过来，对于使实时应用程序快速意识到网络条件的改变，并允许它们适配它们的传输和编码行为是必要的。实现必须 (**MUST**) 支持发送和接收非复合的 RTCP 反馈数据包 [[RFC5506](https://www.rfc-editor.org/rfc/rfc8834#RFC5506)]。对非复合 RTCP 数据包的使用必须 (**MUST**) 使用信令通道进行协商。如果 SDP 用于信令，则协商必须 (**MUST**) 使用 [[RFC5506](https://www.rfc-editor.org/rfc/rfc8834#RFC5506)] 中定义的属性。为了向后兼容，如果远程端点在信令交换中不同意使用非复合 RTCP，则实现也必需 (**REQUIRED**) 支持使用复合 RTCP 反馈数据包。

## 4.7. 对称 RTP/RTCP

为了简化 NAT 和防火墙设备穿透，实现必需 (**REQUIRED**) 实现并使用 [对称 RTP](https://www.rfc-editor.org/rfc/rfc8834#RFC4961) [[RFC4961](https://www.rfc-editor.org/rfc/rfc8834#RFC4961)]。使用对称 RTP 的原因主要是，通过确保发送和接收 RTP 的数据包流，以及 RTCP，实际上是双向传输层流，来避免 NAT 和 防火墙的问题。这将使 NAT 和防火墙绑定保持活动状态，并有助于表明接收方向是预期接收者实际想要的传输层流。此外，它还节省资源，特别是端点上的端口，但也节省了网络中的资源，因为 NAT 映射或防火墙状态不会不必要地膨胀。保持在网络中的每流 QoS 状态的数量也减少了。

## 4.8. RTP 同步源 (SSRC) 的选择

实现必需 (**REQUIRED**) 支持声明的 RTP 同步源 (SSRC) 标识符。如果使用了 SDP，则这必须 (**MUST**) 使用在 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 的第 [4.1](https://www.rfc-editor.org/rfc/rfc5576#section-4.1) 和 [5](https://www.rfc-editor.org/rfc/rfc5576#section-5) 小节中定义的 "a=ssrc:" SDP 属性和 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 的 [第 6.2 节](https://www.rfc-editor.org/rfc/rfc5576#section-6.2) 中定义 "previous-ssrc" 源属性来完成；[[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 中定义的其它每 SSRC 属性也可以 (**MAY**) 支持。

尽管支持声明 SSRC 标识符是强制的，但它们在 RTP 会话中的使用是可选的 (** OPTIONAL**) 。实现必须 (**MUST**) 准备好接受使用 SSRC 的 RTP 和 RTCP 数据包，这些 SSRC 还没有提前明确声明。实现必须 (**MUST**) 根据 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)]，支持随机 SSRC 值，并且必须 (**MUST**) 支持 SSRC 冲突检测和解决。当使用声明的 SSRC 值时，冲突检测必须按照 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8834#RFC5576)] 的 [第 5 节中](https://www.rfc-editor.org/rfc/rfc5576#section-5) 的描述进行。

经常希望将 RTP 数据包流与非 RTP 上下文关联。对于 WebRTC API 的用户，SSRC 和 MediaStreamTrack 之间的映射由 [第 11 节](https://www.rfc-editor.org/rfc/rfc8834#sec-webrtc-api) 提供。对于网关或其它使用，将 RTP 数据包流与使用 SDP 格式化的会话描述中的 "m=" 行关联起来是可能的。如果声明了 SSRC，这很简单（在 SDP 中，"a=ssrc:" 行将处于媒体级别，允许与 "m=" 行直接关联）。如果没有声明 SSRC，RTP 数据包流中使用的 RTP 载荷类型编号，对于将该数据包流与信令上下文关联起来常常是足够的。例如，如果按照本备忘录 [第 4.3 节](https://www.rfc-editor.org/rfc/rfc8834#sec.codecs) 所述分配 RTP 有效载荷类型编号，则 RTP 数据包流使用的 RTP 有效载荷类型可以与 SDP "a=rtpmap:" 行中的值进行比较，这些值在 SDP 中位于媒体级别，因此映射到 "m=" 行。

## 4.9. 生成 RTCP 规范名称 (CNAME)

RTCP 规范名称 (CNAME) 为 RTP 端点提供持久的传输级标识符。如果检测到冲突或重新启动 RTP 应用程序，RTP 端点的 SSRC 标识符可能会更改，但其 RTCP CNAME 在 RTCPeerConnection [W3C.WebRTC] 期间保持不变，因此在一系列相关的 RTP 会话内，该 RTP 端点可以被唯一地标识，并与它们的 RTP 数据包流关联。

每个 RTP 端点必须 (**MUST**) 至少有一个 RTCP CNAME，并且该 RTCP CNAME 在 RTCPeerConnection 中必须 (**MUST**) 是唯一的。RTCP CNAME 标识特定的同步上下文 —— 即，与单个 RTCP CNAME 关联的所有 SSRC 共享一个公共参考时钟。如果一个端点的 SSRCs 与几个不同步的参考时钟相关联，并因此有不同的同步上下文，则它需要使用多个 RTCP CNAME，每个同步上下文一个。

考虑到 [第 11 节](https://www.rfc-editor.org/rfc/rfc8834#sec-webrtc-api) 中的讨论，WebRTC 端点不得 (**MUST NOT**) 在属于单个 RTCPeerConnection 的 RTP 会话中使用多个 RTCP CNAME（即，一个 RTCPeerConnection 形成一个同步上下文）。RTP 中间设备可以 (**MAY**) 生成与多个 RTCP CNAME 相关联的 RTP 数据包流，以允许它们避免必须重新同步来自多个不同端点的媒体，这些端点是多方 RTP 会话的一部分。

[RTP 规范](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 包括选择唯一 RTP CNAME 的指南，但在存在 NAT 设备的情况下这些指南是不够的。此外，从[隐私的角度来看](https://www.rfc-editor.org/rfc/rfc8834#sec-security)（[第 13 节](https://www.rfc-editor.org/rfc/rfc8834#sec-security)），长期持久的标识符可能是有问题的。因此，WebRTC 端点必须 (**MUST**) 按照 [[RFC7022](https://www.rfc-editor.org/rfc/rfc8834#RFC7022)] 为每个 RTCPeerConnection 生成一个新的、唯一的、短期的，持久的 RTCP CNAME，但有一个例外；如果在创建时明确请求，则在其公共同源上下文中，RTCPeerConnection 可以 (**MAY**) 使用与现有 RTCPeerConnection 相同的 CNAME。

WebRTC 端点必须 (**MUST**) 支持接收与 [RTP 规范](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 指定的语法限制相匹配的任何 CNAME，并且不能假定将根据上述建议的形式选择任何 CNAME。

## 4.10. 闰秒处理

应该 (**SHOULD**) 遵循 [[RFC7164](https://www.rfc-editor.org/rfc/rfc8834#RFC7164)] 中给出的关于闰秒的处理的指南，以限制它们对 RTP 媒体播放和同步的影响。

# 5. WebRTC 对 RTP 的使用：扩展

在 WebRTC 上下文中，有许多 RTP 扩展对于获得完整的功能很有必要，或者对提高基线性能非常有用。其中一组扩展与会议有关，而其它扩展在本质上更为通用。以下小节描述了在 WebRTC 中强制或建议使用的各种 RTP 扩展。

## 5.1. 会议扩展和拓扑

RTP 是一种固有地支持组通信的协议。组可以通过让每个端点将其 RTP 数据包流发送到重新分发流量的 RTP 中间设备来实现，通过在端点之间使用单播 RTP 数据包流的网格，或通过使用 IP 多播组来分发 RTP 数据包流。这些拓扑可以通过 [[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 中讨论的多种方式实现。

虽然 IP 多播组的使用在 IPTV 系统中很流行，但基于 RTP 中间设备的拓扑在交互式视频会议环境中占主导地位。迄今为止，基于单播传输层流网格创建通用 RTP 会话的拓扑尚未得到广泛部署。因此，WebRTC 端点不应支持基于 IP 多播组的拓扑或基于网格的拓扑，例如配置为点对多点网格的单个 RTP 会话（[[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 术语中的 “拓扑网格” ("Topo-Mesh")）。然而，使用多个 RTP 会话构建的点对多点网格，在 WebRTC 中使用独立的 [RTCPeerConnections](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC) [[W3C.WebRTC](https://www.rfc-editor.org/rfc/rfc8834#W3C.WebRTC)] 实现，可以预期将用在 WebRTC 中并且需要得到支持。

根据本备忘录实现的 WebRTC 端点预期将支持 [[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 中描述的所有拓扑，其中 RTP 端点向和从某些对等设备发送和接收单播 RTP 数据包流，前提是对等方可以参与对 RTP 数据包执行拥塞控制。对等方设备可以是另一个 RTP 端点，或者它可以是重新分法 RTP 数据包流给其它 RTP 端点的 RTP 中间设备。这种限制意味着一些基于 RTP 中间设备的拓扑不合适在 WebRTC 中使用。具体来说：

 * 不应 (**SHOULD NOT**) 使用视频切换多点控制单元 (MCU) (Topo-Video-switch-MCU)，因为它们使用 RTCP 来解决拥塞控制和服务质量报告的问题（参见 [[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 的 [第 3.8 节](https://www.rfc-editor.org/rfc/rfc7667#section-3.8)）。

 * 不应 (**SHOULD NOT**) 使用中继传输转换器（Topo-PtM-Trn-Translator）拓扑，因为它的安全使用需要拥塞控制算法或处理点对多点的 RTP 断路器，这尚未标准化。

可以使用如下拓扑，然而它还有一些值得注意的问题：

 * 可以 (**MAY**) 使用带有 RTCP 终止的内容修改 MCU (Topo-RTCP-terminating-MCU) 。请注意，在此 RTP 拓扑中，RTP 环路检测和活跃发送者的识别是 WebRTC 应用程序的责任；由于客户端在 RTP 层彼此隔离，因此 RTP 无法协助这些功能（参见 [[RFC7667](https://www.rfc-editor.org/rfc/rfc8834#RFC7667)] 的 [第 3.9 节](https://www.rfc-editor.org/rfc/rfc7667#section-3.9)）。

第 [5.1.1](https://www.rfc-editor.org/rfc/rfc8834#sec-fir) 到 [5.1.6](https://www.rfc-editor.org/rfc/rfc8834#sec.tmmbr) 节中描述的 RTP 扩展设计用于集中式会议，其中 RTP 中间设备（例如，会议桥）接收参与者的 RTP 数据包流并将它们分发给其它参与者。这些扩展对于互操作性来说不是必需的；未实现这些扩展的 RTP 端点将正常工作，但可能会提供较差的性能。对列出的扩展的支持将大大提高体验质量；为了提供合理的基线质量，WebRTC 端点必须支持其中的一些扩展。

RTCP 会议扩展在 “[基于实时传输控制协议 (RTCP) 的反馈 (RTP/AVPF) 的扩展 RTP 配置文件](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)” [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 和 “[带反馈的 RTP 视听配置文件 (AVPF) 中的编解码器控制消息](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)” [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)] 中定义；[此配置文件的安全变体 (RTP/SAVPF)](https://www.rfc-editor.org/rfc/rfc8834#RFC5124) [[RFC5124](https://www.rfc-editor.org/rfc/rfc8834#RFC5124)] 完全可以使用它们。

### 5.1.1. 完整的 Intra Request (FIR)

完整的 Intra Request 消息在 [编解码器控制消息](https://www.rfc-editor.org/rfc/rfc8834#RFC5104) [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)] 的第 [3.5.1](https://www.rfc-editor.org/rfc/rfc5104#section-3.5.1) 和 [4.3.1](https://www.rfc-editor.org/rfc/rfc5104#section-4.3.1) 节定义。它用于使混合器向会话中的参与者请求一个新的 Intra 图片。这在源之间切换时使用，以确保接收器可以解码具有长预测链的视频或其它预测媒体编码。发送媒体的 WebRTC 端点必须 (**MUST**) 理解并响应它们收到的 FIR 反馈消息，因为这极大地改善了使用基于混合器的集中式会议的用户体验。对于发送 FIR 消息的支持是可选的 (** OPTIONAL**) 。

### 5.1.2. 图片丢失指示 (PLI)

图片丢失指示消息在 RTP/AVPF 配置文件 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 的 [第 6.3.1 节](https://www.rfc-editor.org/rfc/rfc4585#section-6.3.1) 中定义。接收者使用它来告诉发送编码器，它丢失了解码器上下文并希望以某种方式对其进行修复。这在语义上与上面的完整 Intra Request 不同，因为可以有多种方式来完成请求。发送媒体的 WebRTC 端点必须 (**MUST**) 理解 PLI 反馈消息并对其做出反应，这是一种丢失处理机制。接收者 (**MAY**) 可以发送 PLI 消息。

### 5.1.3. 切片丢失指示 (SLI)

切片丢失指示在 RTP/AVPF 配置文件 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 的 [第 6.3.2 节](https://www.rfc-editor.org/rfc/rfc4585#section-6.3.2) 中定义。接收者使用它来告诉编码器，它已探测到一个或多个连续宏块的丢失或损坏，并希望以某种方式修复这些宏块。建议 (**RECOMMENDED**) 在使用支持宏块概念的编解码器时，如果切片丢失，接收者生成 SLI 反馈消息。收到 SLI 反馈消息的发送者应该 (**SHOULD**) 尝试修复丢失的切片。

### 5.1.4. 参考图片选择指示 (RPSI)

参考图片选择指示在 RTP/AVPF 配置文件 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 的 [第 6.3.3 节](https://www.rfc-editor.org/rfc/rfc4585#section-6.3.3) 中定义。一些视频编码标准允许使用更旧的参考图片，而不是最新的那个进行预测编码。如果使用了这样的编解码器，并且如果编码器已经知道编码器-解码器同步已经丢失，那么可以将已知为正确的参考图片用作未来编码的基础。RPSI 消息允许通知这种情况。如果正在使用的编解码器支持参考图片选择，探测到编码器-解码器同步已经丢失的接收者应该 (**SHOULD**) 生成一个 RPSI 反馈消息。如果可以在可用带宽限制内，和使用适当编解码器的情况下，接收此类 RPSI 消息的 RTP 数据包流发送者应该 (**SHOULD**) 对该消息采取行动以更改参考图片。

### 5.1.5. 时空权衡请求 (TSTR)

时空权衡请求和通知在 [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)] 的第 [3.5.2](https://www.rfc-editor.org/rfc/rfc5104#section-3.5.2) 和 [4.3.2](https://www.rfc-editor.org/rfc/rfc5104#section-4.3.2) 节中定义。此请求可用于要求视频编码器更改它在时间和空间分辨率之间做出的权衡 —— 比如，高空间图像质量优先但低帧率。支持 TSTR 请求和通知是可选的 (** OPTIONAL**) 。

### 5.1.6. 临时最大媒体流比特率请求 (TMMBR)

临时最大媒体流比特率请求 (TMMBR) 反馈消息在 [编解码器控制消息](https://www.rfc-editor.org/rfc/rfc8834#RFC5104) [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)] 的第 [3.5.4](https://www.rfc-editor.org/rfc/rfc5104#section-3.5.4) 和 [4.2.1](https://www.rfc-editor.org/rfc/rfc5104#section-4.2.1) 节中定义。体接收者使用这个请求及其相应的临时最大媒体流比特率通知 (TMMBN) 消息 [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)] 通知发送方，该接收器可用的带宽量当前存在限制。这可能有多种原因：例如，RTP 混合器可以使用此消息来限制混合器转发的发送方的媒体速率（不进行媒体转码），以适应其它会话参与者存在的瓶颈。发送媒体的 WebRTC 端点必需 (**REQUIRED**) 实现对 TMMBR 消息的支持，并且必须 (**MUST**) 为其 SSRC 遵循接收到的 TMMBR 消息设置的带宽限制。

## 5.2. 头部扩展

[RTP 规范](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 提供了包含 *包含带内数据的 RTP 头部扩展* 的能力，但扩展的格式和语义没有明确规定。在 WebRTC 中使用头部扩展是可选的 (** OPTIONAL**) ，但如果使用它们，它们必须 (**MUST**) 遵循 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中定义的 RTP 头部扩展的通用机制进行格式化和声明，因为这给 RTP 头部扩展提供了明确定义的语义。

正如 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中所指出的，RTP 规范中关于头部扩展的要求是 “设计为可以忽略头部扩展” [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)]。具体来说，头部扩展必须 (**MUST**) 只用于接收方可以安全忽略而不影响互操作性的数据，并且当扩展的存在以某种方式改变了数据包其余部分的形式或性质，而与流声明的方式不兼容时，不得 (**MUST NOT**) 使用（例如，由有效负载类型定义）。RTP 头部扩展的有效示例可能包括通常 RTP 信息之外的附加元数据，但可以在不影响互操作性的情况下安全地忽略这些元数据。

### 5.2.1. 快速同步

许多 RTP 会话需要在音频、视频和其它内容之间进行同步。这种同步由接收者执行，使用包含在 RTCP SR 数据包中的信息，如 [RTP 规范](https://www.rfc-editor.org/rfc/rfc8834#RFC3550) [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 中所描述的。但这种基本的机制可能比较慢，因而建议 (**RECOMMENDED**) 在基于 RTCP SR 的同步之外，实现 [[RFC6051](https://www.rfc-editor.org/rfc/rfc8834#RFC6051)] 中描述的快速 RTP 同步扩展。

这个头部扩展使用 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中描述的通用头部扩展框架，因而在使用之前需要进行协商。

### 5.2.2. 客户端到混合器的音频电平

端点使用 [客户端到混合器的音频电平扩展](https://www.rfc-editor.org/rfc/rfc8834#RFC6464) [[RFC6464](https://www.rfc-editor.org/rfc/rfc8834#RFC6464)] RTP 头部扩展，通知混合器头部所附加到的数据包中的音频活动电平。这使 RTP 中间设备无需解码或详细检查有效载荷即可做出混合或选择决策，从而降低了某些类型的混合器的复杂性。它还可以节省接收者中的解码资源，接收者可以根据音频活动电平选择仅解码最相关的 RTP 数据包流。

[客户端到混合器的音频电平扩展](https://www.rfc-editor.org/rfc/rfc8834#RFC6464) [[RFC6464](https://www.rfc-editor.org/rfc/rfc8834#RFC6464)] 必须 (**MUST**) 被实现。实现必需 (**REQUIRED**) 能够根据 [[RFC6904](https://www.rfc-editor.org/rfc/rfc8834#RFC6904)] 加密头部扩展，因为这些头部扩展中包含的信息可能被认为是敏感的。建议 (**RECOMMENDED**) 使用这种加密；然而，对加密的使用可以通过 API 或声明被明确禁止。

这个头部扩展使用 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中描述的通用头部扩展框架，因而在使用之前需要进行协商。

### 5.2.3. 混合器到客户端的音频电平

[混合器到客户端的音频电平头部扩展](https://www.rfc-editor.org/rfc/rfc8834#RFC6465) [[RFC6465](https://www.rfc-editor.org/rfc/rfc8834#RFC6465)] 为一个端点提供了由 RTP 混合器混入一个公共的音频源流的不同源的音频电平。这使得用户界面可以指示每个会话参与者的相对活动等级，而不仅仅是基于 CSRC 字段是否包含。这是对非关键功能的纯粹优化，因此对实现是可选的 (** OPTIONAL**) 。如果实现了这个头部扩展，则实现必需 (**REQUIRED**) 能够根据 [[RFC6904](https://www.rfc-editor.org/rfc/rfc8834#RFC6904)] 加密头部扩展，因为这些头部扩展中包含的信息可能被认为是敏感的。进一步建议 (**RECOMMENDED**) 使用这种加密，除非加密已经通过 API 或声明被明确禁止。

这个头部扩展使用 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中描述的通用头部扩展框架，因而在使用之前需要进行协商。

### 5.2.4. 媒体流识别

实现 SDP 捆绑协商扩展的 WebRTC 端点将使用 SDP 分组框架 "mid" 属性来识别媒体流。这样的端点必须 (**MUST**) 实现 [[RFC8843](https://www.rfc-editor.org/rfc/rfc8834#RFC8843)] 中描述的 RTP MID 头部扩展。

这个头部扩展使用 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中描述的通用头部扩展框架，因而在使用之前需要进行协商。

### 5.2.5. 视频方向的协调

发送或接收视频的 WebRTC 端点必须 (**MUST**) 实现视频方向协调 (CVO) RTP 头部扩展，如 [[RFC7742](https://www.rfc-editor.org/rfc/rfc8834#RFC7742)] 的 [第 4 节](https://www.rfc-editor.org/rfc/rfc7742#section-4) 所述。

这个头部扩展使用 [[RFC8285](https://www.rfc-editor.org/rfc/rfc8834#RFC8285)] 中描述的通用头部扩展框架，因而在使用之前需要进行协商。

# 6. WebRTC 对 RTP 的使用：提高传输健壮性

有一些工具可以使 RTP 数据包流对数据包丢失具有健壮性，并减少丢失对媒体质量的影响。但是，与非健壮的流相比，它们通常会增加一些开销。这种开销需要考虑，并且必须对总比特率进行速率控制以避免导致网络拥塞（参见 [第 7 节](https://www.rfc-editor.org/rfc/rfc8834#sec-rate-control)）。因此，提高健壮性可能需要较低的基本编码质量，但有可能以更少的错误提供该质量。以下小节中描述的机制可用于提高对丢包的容忍度。

## 6.1. 否定确认和 RTP 重传

作为支持 RTP/SAVPF 配置文件的结果，实现可以为 RTP 数据包发送否定确认 (NACKs) [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)]。此反馈可用于通知发送方特定 RTP 数据包的丢失，但受 RTCP 反馈通道的容量限制。发送者可以使用此信息，通过调整媒体编码来补偿已知丢失的数据包，来优化用户体验。

RTP 数据包流的发送方必需 (**REQUIRED**) 能够理解 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 的 [第 6.2.1 节](https://www.rfc-editor.org/rfc/rfc4585#section-6.2.1) 中定义的通用 NACK 消息，但它们可以 (**MAY**) 选择忽略部分或全部这些反馈（遵循 [[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] [第 4.2 节](https://www.rfc-editor.org/rfc/rfc4585#section-4.2)）。接收方可以 (**MAY**) 为丢失的 RTP 数据包发送 NACK。[[RFC4585](https://www.rfc-editor.org/rfc/rfc8834#RFC4585)] 提供了关于何时发送 NACK 的指南。不期望接收者会为每个丢失的 RTP 数据包发送 NACK；相反，它需要考虑发送 NACK 反馈的成本和丢失数据包的重要性，以便就是否值得将丢包事件告知发送方做出明智的决定。

[RTP 重传有效负载格式](https://www.rfc-editor.org/rfc/rfc8834#RFC4588) [[RFC4588](https://www.rfc-editor.org/rfc/rfc8834#RFC4588)] 提供了基于 NACK 反馈重传丢失的数据包的能力。在交互式实时应用程序中需要谨慎使用重传，以确保重传的数据包及时到达能够派上用场，但它在网络 RTT 相对较低的环境中可能有效。（RTP 发送方可以使用 RTCP SR 和 RR 数据包中的信息估计到达接收方的 RTT，如 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8834#RFC3550)] 的 [第 6.4.1 节](https://www.rfc-editor.org/rfc/rfc3550#section-6.4.1) 末尾所述）。使用重传还可能增加转发 RTP 的带宽，如果原始数据包丢失是由网络拥塞引起的的话，则可能会导致数据包丢失增加。但是请注意，重传重要的丢失数据包以修复解码器状态，可以比发送完整的 intra 帧具有更低的成本。盲目地重传 RTP 数据包以响应 NACK 是不合适的。在使用 RTP 重传之前，需要考虑丢失数据包的重要性以及它们及时到达有用的可能性。

接收方必需 (**REQUIRED**) 实现对使用 SSRC 多路复用发送的 RTP 重传数据包 [[RFC4588](https://www.rfc-editor.org/rfc/rfc8834#RFC4588)] 的支持，也可以 (**MAY**) 支持使用会话多路复用发送的 RTP 重传数据包。如果已经协商了对 RTP 重传有效载荷格式的支持，并且发送者认为发送 NACK 中引用的包的重传是有用的，则发送者可以 (**MAY**) 发送 RTP 重传包以响应 NACK。发送者不需要重新传输每个 NACK 引用的数据包。

## 6.2. 前向纠错 (FEC)

使用前向纠错 (FEC) 可以有效地防止某种程度的数据包丢失，但代价是稳定的带宽开销。有几种 FEC 方案被定义为与 RTP 一起使用。其中一些方案特定于特定的 RTP 有效负载格式，而其它方案则跨 RTP 数据包运行，并且可以与任何有效负载格式一起使用。请注意，使用冗余编码或 FEC 会导致播放延迟增加，在选择 FEC 方案及其参数时需要考虑这一点。

WebRTC 端点必须 (**MUST**) 遵循 [[RFC8854](https://www.rfc-editor.org/rfc/rfc8834#RFC8854)] 中给出的 FEC 使用建议。WebRTC 端点可以 (**MAY**) 支持其它类型的 FEC，但这些必须 (**MUST**) 在使用之前进行协商。

# 7. WebRTC 对 RTP 的使用：速率控制和媒体适配

WebRTC 将用于使用各种链接技术（包括有线和无线链接）的异构网络环境中，以互连世界各地潜在的大量用户。因此，用户之间的网络路径可能具有广泛变化的单向延迟、可用比特率、负载水平和流量混合。各个端点可能向每个参与者发送一个或多个 RTP 数据包流，并且可能有多个参与者。这些 RTP 数据包流中的每一个都可以包含不同类型的媒体，并且媒体类型、比特率和 RTP 数据包流的数量以及传输层流可能是高度不对称的。非 RTP 流量可能与 RTP 传输层流共享网络路径。由于网络环境不可预测或不稳定，WebRTC 端点必须 (**MUST**) 确保它们生成的 RTP 流量能够适应可用网络容量的变化。

WebRTC 用户的体验质量很大程度上取决于媒体对网络限制的有效适应。端点的设计必须使它们传输的数据不会明显超过网络路径可以支持的数据，除非是非常短的时间段；否则，将出现高水平的网络丢包或延迟峰值，从而导致媒体质量下降。网络路径容量的限制因素可能是链路带宽，也可能是与链路上其它流量的竞争（这可能是非 WebRTC 流量、来自其它 WebRTC 流的流量，甚至是与相同会话中其它 WebRTC 流的竞争）。

因此，有效的媒体拥塞控制算法是 WebRTC 框架的重要组成部分。 但是，在撰写本文时，还没有标准的拥塞控制算法可用于交互式媒体应用程序，例如 WebRTC 的流。

[[RFC8836](https://www.rfc-editor.org/rfc/rfc8834#RFC8836)] 中讨论了 RTCPeerConnections 的拥塞控制算法的一些要求。如果将来开发出满足这些要求的标准化拥塞控制算法，则需要更新该备忘录以强制使用它。

## 7.1. 边界条件和断路器

WebRTC 端点必须 (**MUST**) 实现 [[RFC8083](https://www.rfc-editor.org/rfc/rfc8834#RFC8083)] 中描述的 RTP 断路器算法。RTP 断路器旨在使应用程序能够识别极端网络拥塞的情况并做出反应。然而，由于 RTP 断路器可能在拥塞变得极端之前不会被触发，因此它不能被视为拥塞控制的替代品，并且应用程序还必须 (**MUST**) 实现拥塞控制以允许它们适应网络容量的变化。在标准化的拥塞控制算法可用之前，拥塞控制算法不得不是专有的。任何未来的 RTP 拥塞控制算法都应该在断路器允许的范围内运行。

会话建立信令也必须建立媒体比特率符合的边界。媒体编解码器的选择提供了应用程序可以用来提供有用质量的支持的比特率的上限和下限，以及存在的打包 (packetization) 选择。此外，信令通道可以建立使用的最大媒体比特率边界，比如 SDP 的 "b=AS:" 或 "b=CT:" 行，和 RTP/AVPF TMMBR 消息（参见本备忘录的 [第 5.1.6 节](https://www.rfc-editor.org/rfc/rfc8834#sec.tmmbr)）。声明的带宽限制，比如从对端收到的 SDP "b=AS:" 或 "b=CT:" 行，在发送 RTP 数据包流时必须 (**MUST**) 遵循。接收媒体的 WebRTC 端点应该 (**SHOULD**) 声明它的带宽限制。这些限制必须基于已知的带宽限制，比如边缘链路的容量。

## 7.2. 拥塞控制互操作性和遗留系统

所有希望与 WebRTC 互通的端点必须 (**MUST**) 实现 RTCP 并通过定义的 RTCP 报告机制提供拥塞反馈。

当使用 [RTP/AVP 配置文件](https://www.rfc-editor.org/rfc/rfc8834#RFC3551) [[RFC3551](https://www.rfc-editor.org/rfc/rfc8834#RFC3551)] 与支持 RTCP 的遗留系统互通时，每隔几秒在 RTCP RR 数据包中提供拥塞反馈。必须与这样的端点互通的实现必须 (**MUST**) 确保，它们保持在 [RTP 断路器](https://www.rfc-editor.org/rfc/rfc8834#RFC8083) [[RFC8083](https://www.rfc-editor.org/rfc/rfc8834#RFC8083)] 约束范围内，以限制它们可能导致的拥塞。

如果遗留端点支持 RTP/AVPF，这可以协商用于频繁报告的重要参数，例如 "trr-int" 参数，以及端点支持某些用于拥塞控制目的的有用反馈格式的可能性，例如 [TMMBR](https://www.rfc-editor.org/rfc/rfc8834#RFC5104) [[RFC5104](https://www.rfc-editor.org/rfc/rfc8834#RFC5104)]。必须与这样的端点互通的实现必须 (**MUST**) 确保，它们保持在 [RTP 断路器](https://www.rfc-editor.org/rfc/rfc8834#RFC8083) [[RFC8083](https://www.rfc-editor.org/rfc/rfc8834#RFC8083)] 约束范围内，以限制它们可能导致的拥塞，但是它们可能会发现它们可以根据可用的反馈量实现更好的拥塞响应。

使用专有的拥塞控制算法，当不同的算法和实现在通信会话中交互时可能会出现问题。如果不同的实现在适配类型方面做出了不同的选择，例如一个基于发送方和一个基于接收方，那么最终可能会出现一个方向被双重控制而另一个方向不受控制的情况。本备忘录不强制要求专有拥塞控制算法的行为，但使用此类算法的实现应该意识到这个问题，并尝试确保为双向媒体流协商有效的拥塞控制。如果 IETF 未来要为 WebRTC 流量标准化基于发送方和接收方的拥塞控制算法，则还需要考虑互操作性、控制以及确保媒体流的两个方向都受到拥塞控制的问题。

原文：[RFC 8834 Media Transport and Use of RTP in WebRTC](https://www.rfc-editor.org/rfc/rfc8834)
