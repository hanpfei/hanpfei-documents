---
title: QUIC 丢包检测和拥塞控制
date: 2019-08-29 21:37:49
categories: 网络协议
tags:
- 网络协议
- QUIC
- 翻译
---

# 摘要

本文档描述 QUIC 的丢包检测和拥塞控制机制。
<!--more-->

# 读者注意

本草案的讨论位于 QUIC 工作组的邮件列表（quic@ietf.org），邮件列表的存档位于 https://mailarchive.ietf.org/arch/search/?email_list=quic [[1](https://tools.ietf.org/html/draft-ietf-quic-recovery-20#ref-1)]。

工作组信息可以 [https://github.com/quicwg](https://github.com/quicwg) [[2](https://tools.ietf.org/html/draft-ietf-quic-recovery-20#ref-2)] 找到；源码及问题清单可以在 [https://github.com/quicwg/base-drafts/labels/-recovery](https://github.com/quicwg/base-drafts/labels/-recovery) [[3](https://tools.ietf.org/html/draft-ietf-quic-recovery-20#ref-3)] 找到。

# 本备忘录的状态

本互联网草案的提交完全符合 [BCP 78](https://tools.ietf.org/html/bcp78) 和 [BCP 79](https://tools.ietf.org/html/bcp79) 的规定。

互联网草案是互联网工程任务组（IETF）的工作文档。请注意，其他组也可能以互联网草案分发工作文档。当前的互联网草案列表位于 [https://datatracker.ietf.org/drafts/current/](https://datatracker.ietf.org/drafts/current/)。

互联网草案是草案文档，它们最长的有效期为 6 个月，且可能在任何时间被其它文档更新，替代，或废弃。使用互联网草案作为参考资料或引用它们，而不是把它们作为 “正在进行的工作” 是不恰当的。

本互联网草案将在 2020 年 2 月 29 日过期。

# 版权声明

版权所有（c）2019 IETF Trust 和被确定为文件作者的人员。保留一切权力。

本文档受 [BCP 78](https://tools.ietf.org/html/bcp78) 和 IETF Trust 有关IETF 文件的法律规定（https://trustee.ietf.org/license-info）的约束，该文件自本文档发布之日起生效。 请仔细阅读这些文档，它们描述了关于本文档，你的权利和限制。 从本文档中提取的代码组件必须包含 Trust Legal Provisions 第 4.e 节所述的简化 BSD 许可文本，且如简化的 BSD 许可证所述，不提供担保。

# 1. 介绍

QUIC 是一个基于 UDP 的新的多路复用且安全的传输协议。QUIC 建立于数十年传输及安全经验的基础之上，并实现了使它成为现代的有吸引力的通用传输协议的机制。QUIC 协议如 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 所述。

QUIC 实现了已有的 TCP 丢包恢复机制的思路和想法，包括各种 RFC 中描述的，多种互联网草案，以及在 Linux TCP 实现中流行的那些。本文档描述 QUIC 拥塞控制和丢包恢复，及适用的地方，属性等效 TCP 的 RFCs，互联网草案，学术论文，和/或 TCP 实现。

# 2. 规约和定义

本文档中的关键字 “MUST”，”MUST NOT”，”REQUIRED”，”SHALL”，”SHALL NOT”，”SHOULD”，”SHOULD NOT”，”RECOMMENDED”，”MAY”，和 “OPTIONAL” 按 [BCP 14](https://tools.ietf.org/html/bcp14)，[RFC 2119](https://tools.ietf.org/html/rfc2119) [[2](https://tools.ietf.org/html/rfc3533#ref-2)] 中的描述解释，当且仅当它们如这里所示全部以大写形式出现的时候。

本文档中使用的术语定义如下：

 ACK-only：只包含一个或多个 ACK 帧的任何包。

 In-flight：当非 ACK-only 的包已经被发送，但还没有被确认、宣布丢失或连同旧的秘钥一起丢弃时，被认为是 in-flight 的。

 ACK 诱发帧：除 ACK 和 PADDING 外的所有帧被认为是 ACK 诱发帧。

 ACK 诱发包：包含 ACK 诱发帧的包在最大 ACK 延迟内诱发一个来自于接收者的 ACK，被称为 ACK 诱发包。

 加密包：在初始化或握手包中发送的包含加密数据的包。

 乱序包：没有为它的包号空间的最大接收包包号精确地递增一的包。当更早的包丢失或延迟时包乱序到达。

# 3. QUIC 传输机制的设计

QUIC 中所有的传输发送时都带有一个包级的头部，它指示加密级别并包含一个包序列号（下文称为包号）。加密级别指示包号空间，如 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 所述。在一个连接的生命周期中的包号空间内包号从不重复。一个空间内包号单调递增，以防止歧义。

这种设计消除了在传输和重传之间的二义性消除的需要，并从 TCP 丢包探测机制的 QUIC 翻译中消除了显著的复杂性。

QUIC 包可以包含不同类型的多个帧。恢复机制确保需要可靠递送的数据和帧被确认或被宣布丢失，并在必要时在新包中发送。包中包含的帧的类型影响恢复和拥塞控制逻辑：

 * 所有的包都被确认，尽管包含非 ACK 诱发帧的包只与 ACK 诱发包一起确认。

 * 包含加密帧的长头部包对于 QUIC 握手的性能非常重要，因此使用更短的确认定时器。

 * 只包含 ACK 帧的包不计入拥塞控制限制，且不被认为是 in-flight 的。

 * PADDING 帧使得包贡献 in flight 的字节数，但不直接导致确认的发送。

## 3.1 QUIC 和 TCP 之间的相关差异

熟悉 TCP 的丢失检测和拥塞控制的读者将发现这里的算法与 TCP 那些众所周知的算法是并行的。然而，QUIC 和 TCP 之间的协议差异导致了算法上的差异。我们将在下面简要地描述这些协议差异。

### 3.1.1 单独的包号空间

QUIC 为每个加密级别使用单独的包号空间，除了 0-RTT 和所有代的 1-RTT 密钥使用相同的包号空间。单独的包号空间确保一个加密级别上发送的包的确认不会导致不同加密级别上发送的包的虚假重传。拥塞控制和往返时间 (RTT) 测量是跨包号空间统一的。

### 3.1.2 单调递增的包号

TCP 将发送端的传输顺序与接收端的交付顺序合并在一起，这会导致携带相同序列号的相同数据的重传，从而导致 “重传歧义”。QUIC将两者分开：QUIC 使用包号来指示传输顺序，并且所有的应用程序数据都在一个或多个流中发送，交付顺序由 STREAM 帧中编码的流偏移确定。

QUIC 的包号在包号空间内严格递增，并直接编码传输顺序。较高的包号表示该数据包是较晚发送的，而较低的包号表示该数据包发送的较早。当检测到包含 ACK 诱发帧的包丢失时，QUIC 会将必要的帧重新打包进具有新包号的新数据包中，则当收到 ACK 时，这种做法可以消除该 ACK 在确认哪个数据包的歧义性。因此，可以进行更精确的 RTT 测量，很容易检查到虚假重传，并且可以仅仅基于包号来统一应用诸如快速重传这样的机制。

这个设计点大大简化了 QUIC 的丢失检测机制。大多数 TCP 机制都会隐式地尝试根据 TCP 序列号推断传输顺序 - 这是一项比较艰难的任务，特别是当 TCP 时间戳不可用时。

### 3.1.3 更清晰的丢失世代

QUIC 在一个包发送后被声明且确认为丢失时结束一个丢失世代。TCP 等待包号空间中的空位被填满，因此如果一段数据在一行中丢失了多次，丢失世代可能在多个往返中都不会结束。由于每个世代中这些情况都应该减小拥塞窗口仅一次，QUIC 可以在每个发生丢包的往返中精确地执行一次，而 TCP 可能只能跨多个往返执行一次。

### 3.1.4 No Reneging

QUIC ACK 包含类似于 TCP SACK 的信息，但 QUIC 不允许任何被确认的包被 reneged，从而极大地简化了双方的实现，并降低了发送方的内存压力。

### 3.1.5 更多 ACK 范围

QUIC 支持多 ACK 范围，与 TCP 的 3 SACK 范围相反。在高丢包环境，这将加速恢复，降低伪重传，并确保发送过程不依赖于超时。

### 3.1.6 延迟的确认的显式修正

当收到一个包且对应的确认被发送时，QUIC 端点测量两端的延迟，这允许对端维护一个更精确的往返时间测量值（参考 [第 4.4 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#host-delay)）。

# 4. 生成确认包

一个确认 SHOULD （应该）在第二个 ack 诱发包收到之后就立即发送。QUIC 算法不假设对端在第二个 ack 诱发包收到之后立即发送 ACK。

为了加速丢包恢复并减少超时，一个端点 SHOULD （应该）在它收到一个乱序的 ACK 诱发包时立即发送一个 ACK 帧。端点 MAY（可以）持续在后续每收到一个包时立即发送 ACK 帧，但端点 SHOULD （应该）在 1/8 x RTT 周期之后返回确认每个其它的包，除非收到了更多的乱序 ACK 诱发包。如果每个后续的 ACK 诱发包到达的都是乱序的，则每次收到 ACK 诱发包都 SHOULD （应该）立即发送一个 ACK 帧。

如果收到的包在其 IP 头中带有 ECN 拥塞经历 (CE) 码点，则端点SHOULD （应该）立即用一个 ACK 帧来响应，即使包是按序到达的。这样做减小对端对拥塞事件的响应时间。

如果一个端点中已经有多个包可用了，则端点 MAY（可以）在响应中发送 ACK 帧之前处理它们。端点可以决定是否应该立即发送确认或延迟到处理了一批包之后。

## 4.1 加密握手数据

为了快速完成握手并避免由于加密重传超时引起的伪重传，加密包SHOULD （应该）使用一个非常短的 ACK 延迟，比如本地计时器粒度。当加密栈指示那个包号空间的所有数据都已经收到时，SHOULD （应该）立即发送 ACK 帧。

## 4.2 ACK 范围

当发送 ACK 帧时，包含一个或多个确认的包的范围。包含更老的包降低了由于前面发送的 ACK 帧丢失导致的伪重传的机会，以更大的 ACK 帧为代价。

ACK 帧 SHOULD （应该）总是确认最近收到的包，乱序的包越多，快速发送一个更新的 ACK 帧越重要，以阻止对端声明一个包丢失及虚假重传它包含的帧。

下面是一个建议的决定在 ACK 帧中包含什么包的方法。

## 4.3 ACK 帧的接收端追踪

当发送包含 ACK 帧的数据包时，可以保存在该帧中确认的最大值。当包含 ACK 帧的数据包被确认时，接收方可以停止确认小于或等于发送的 ACK 帧中确认的最大值的数据包。

在没有 ACK 帧丢失的情况下，算法允许最小 1 个 RTT 的重排序。在 ACK 帧丢失和重排序的情况下，此方法不能保证发送方在每个确认不再包含在 ACK 帧中之前看到它们。数据包可以被乱序接收，且包含它们的所有后续 ACK 帧都可能丢失。在这种情况下，丢失恢复算法可能会导致虚假重传，但发送方将继续发送过程。

## 4.4 测量并报告主机延迟

端点有意地测量当一个 ACK 诱发帧被收到及对应的确认被发送之间引入的延迟。端点在 ACK 帧的 ACK 延迟字段中为最大的被确认包编码这个延迟（参考 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的第 19.3 节）。这允许 ACK 的接收者为任何有意的延迟做调整，这对于延迟的确认很重要，当计算路径 RTT 时。数据包在被处理前可能由主机的 OS 内核或其它什么东西持有。一个端点在给一个 ACK 帧填充其 ACK 延迟字段时 SHOULD NOT（不应该）包含这些无意的延迟。

一个端点 MUST NOT（一定不能）过度地推迟 ACK 诱发包的确认。最大的 ack 延迟在 max_ack_delay 传输参数中沟通；参考 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的第 18.2 节。max_ack_delay 意味着一个显式的约定：一个端点保证从不会延迟一个 ACK 诱发包的确认超过这个字段指示的值。如果做到了这一点，任何多余的都可计入 RTT 估计值并可能导致对端的虚假重传。对于初始化和握手包，使用 0 值 max_ack_delay。

# 4 估计往返时间

在高层，一个端点测量包被发送的时间到它被确认的时间作为往返时间（RTT）样本。端点使用 RTT 样本和对端报告的主机延迟（[第 4.4 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#host-delay)）生成一个连接的 RTT 的统计描述。端点计算如下三个值：连接生命周期中观察到的最小值（min_rtt），指数加权移动平均（smoothed_rtt），观察到的 RTT 样本的变化（rttvar）。

## 4.1 生成 RTT 样本

端点在接收到满足如下两个条件的 ACK 帧时生成一个 RTT 样本：
 - 最大的被确认包号是新确认的，且
 - 至少有一个新确认的包是 ACK 诱发包。

RTT 样本，latest_rtt 以自最大的被确认包被发送之后流逝的时间计：
```
latest_rtt = ack_time - send_time_of_largest_acked
```

RTT 样本只使用接收到的 ACK 帧中的最大被确认包生成。这是因为对端只针对 ACK 帧中的最大被确认包报告主机延迟。尽管报告的主机延迟不被用于 RTT 样本测量，但它被用于在后续计算 [第 5.3 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#smoothed-rtt) 的 smoothed_rtt 和 rttvar 中调整 RTT 样本。

为了避免使用相同的包生成多个 RTT 样本，如果它没有新确认最大已确认包，则ACK 帧 SHOULD NOT（不应该）被用于更新 RTT 估计。

RTT MUST NOT（一定不能）在收到的 ACK 帧没有新确认至少一个 ACK 诱发帧时生成。对端不在收到一个只包含非 ACK 诱发帧的包时发送 ACK 帧，因此后续发送的 ACK 帧可能包含一个任意大的 ACK 延迟字段。忽略这样的 ACK 帧避免后续的 smoothed_rtt 和 rttvar 计算中的并发症。

当一个 RTT 内收到多个 ACK 帧时，发送者可能每 RTT 生成多个 RTT 样本。如 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298) 给出的建议，这样做可能导致 smoothed_rtt 和 rttvar 中不充分的历史。确保那个 RTT 估计保留足够的历史是一个开放的研究课题。

## 4.2 估计 min_rtt

min_rtt 是在连接的整个声明周期中观察到的最小的 RTT。在采集到连接中的第一个样本时设置为 latest_rtt，后续采集到样本时设置为 min_rtt 和 latest_rtt 之中较小的那个。

一个端点在计算 min_rtt 时只使用本地观察到的时间，且不根据对端报告的主机延迟做调整。这样做允许端点完全基于它所观察到的为 smoothed_rtt 设置一个更低的边界（参见 [第 4.3 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#smoothed-rtt)），并限制由于对端错误报告的延迟导致的潜在的错误估计。

## 4.3 估计 smoothed_rtt 和 rttvar

smoothed_rtt 是端点的 RTT 样本的指数加权移动平均，rttvar 是端点估计 RTT 样本的方差。

smoothed_rtt 的计算使用针对主机延时调整之后的路径延迟。对于在 ApplicationData 包号空间中发送的包，对端限制为一个 ACK 诱发包发送的确认的任何延时不能大于它在 max_ack_delay 传输参数中广告的值。因此，当对端报告一个大于它的 max_ack_delay 的 Ack Delay 时，则延迟被归因于失去了对端的控制，比如对端的调度器延迟，或前面的 ACK 帧丢失。因此，任何超出对端的 max_ack_delay 的延迟都被有效地视为路径延迟的一部分，并被纳入 smoothed_rtt 的评估中。

当使用对端报告的确认延迟调整 RTT 样本时，一个端点：

 * 必须（MUST ）忽略初始化和握手包号空间中发送的包的 ACK  帧的 Ack Delay 字段。
 * 必须（MUST ）使用比 ACK 帧的 Ack Delay 字段报告的值和对端的 max_ack_delay 传输参数更小的值。
 * 如果最终的 RTT 样本小于 min_rtt，则必须不（MUST NOT）应用调整。这限制了误报的对端可能导致的对 smoothed_rtt 的低估。

连接中的第一个 RTT 样本到达时，smoothed_rtt  被设置为 latest_rtt。

smoothed_rtt 和 rttvar 按如下的方式计算，类似于 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298)。

连接中的第一个 RTT 样本到达时：
```
smoothed_rtt = latest_rtt
rttvar = latest_rtt / 2
```

在后续的 RTT 样本中：
```
ack_delay = min(Ack Delay in ACK Frame, max_ack_delay)
adjusted_rtt = latest_rtt
if (min_rtt + ack_delay < latest_rtt):
  adjusted_rtt = latest_rtt - ack_delay
smoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
rttvar_sample = abs(smoothed_rtt - adjusted_rtt)
rttvar = 3/4 * rttvar + 1/4 * rttvar_sample
```

# 5. 丢包检测

QUIC 发送端同时使用确认信息和超时来探测丢失的包，这一节将提供这些算法的描述。

如果一个包丢失了，则 QUIC 传输需要从这次丢失中恢复，比如通过重传数据，发送一个更新的帧，或放弃帧。更多信息，请参考  [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的 13.3 节。

## 5.1 基于确认的检测

基于确认的丢包探测实现了 TCP 的快速重传 [[RFC5681]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5681)，Early Retransmit(ER) [[RFC5827]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5827)，FACK [[FACK]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#FACK)，SACK 丢包恢复[[RFC6675]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6675)，和 RACK [[RACK]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RACK) 机制。这一节提供了这些算法在 QUIC 中是如何实现的概述。

如果一个包满足如下的所有条件，则它被声明为丢失：

 * 包未确认，在传输中，以及先于被确认的包发送。
 * 它的包号是 kPacketThreshold，小于一个确认的包([第 5.1.1 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#packet-threshold))，或者它在过去发送了足够长的时间([第 5.1.2 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#time-threshold))。

确认表示后面发送的一个包已经送达，而包和时间阈值为包的重新排序提供了一定的容忍度。虚假地将数据包声明为丢失会导致不必要的重发，并可能由于拥塞控制器在检测到丢失时的动作而导致性能下降。检测到虚假重发并增加包或时间的重排序阈值的实现可以选择从较小的初始重排序阈值开始，以最小化恢复延迟。

### 5.1.1 包阈值

数据包重新排序阈值 (kPacketThreshold) 的推荐（RECOMMENDED ）初值为3，基于TCP丢失检测的最佳实践  [[RFC5681]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5681) [[RFC6675]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6675)。

一些网络可能表现出更高程度的重排序，导致发送方检测到虚假的丢包。实现者可以使用为 TCP 开发的算法，比如 TCP-NCR [[RFC4653]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC4653)，来提高 QUIC 的重新排序弹性。

### 5.1.2 时间阈值

一旦在相同的包号空间内的较晚的包被确认，如果数据包在发送之后经过了一个阈值时间量，端点应该（SHOULD ）声明该包丢失。为了避免过早地声明包丢失，这个时间阈值必须（MUST ）设置为至少是 kGranularity。时间阈值为：
```
kTimeThreshold * max(smoothed_rtt, latest_rtt, kGranularity)
```

如果在最大的已确认的包之前发送的包还不能被声明丢失，那么应该（SHOULD ）为剩余的时间设置一个定时器。

使用 max(smoothed_rtt, latest_rtt) 可以避免以下两种情况：

 * 最新的 RTT 样本低于经过平滑处理的 RTT，可能是由于在确认包遇到较短路径时进行了重新排序
 * 最新的 RTT 样本高于平滑 RTT，这可能是由于实际 RTT 的持续增长，但平滑 RTT 还没有赶上。

建议（RECOMMENDED ）的时间阈值(kTimeThreshold)，表示为往返时间的乘数，是 9/8。

实现可以（MAY ）使用绝对阈值、来自以前连接的阈值、自适应阈值或包括 RTT 方差进行实验。较小的阈值会降低重新排序的弹性并增加虚假重发，较大的阈值会增加丢失检测延迟。

## 5.2 探测超时

当在预期的时间内没有确认 ACK 诱发包或握手尚未完成时，探测超时(PTO)触发发送一个或两个探测数据报。PTO 使连接能够从丢失的尾包或确认中恢复。在 QUIC 中使用的 PTO 算法实现了 Tail Loss Probe [[RACK]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RACK)、RTO [[RFC5681]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5681) 和 TCP 的 F-RTO 算法 [[RFC5682]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5682) 的可靠性功能，超时计算基于 TCP 的重传超时周期 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298)。

### 5.2.1 计算 PTO

当一个 ACK 诱发包被传输时，发送方为 PTO 周期安排一个定时器，如下所示：
```
PTO = smoothed_rtt + max(4*rttvar, kGranularity) + max_ack_delay
```

kGranularity，smoothed_rtt，rttvar, 和 max_ack_delay 在[附录 A.2](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#ld-consts-of-interest) 和[附录 A.3](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#ld-vars-of-interest) 中定义。

PTO 周期是指发送方应该等待接收到发送的报文的确认包的时间。这段时间包括估计的网络往返时间（smoothed_rtt），估计的 (4*rttvar) 的方差，和 max_ack_delay，以充分考虑接收方可能延迟发送确认的最大时间。

必须将 PTO 值至少设置为 kGranularity，以避免计时器立即过期。

当 PTO 计时器到期时，必须（MUST）将 PTO 周期设置为当前值的两倍。送方速率的指数级降低非常重要，因为 PTO 可能由于严重的拥塞而导致数据包丢失或确认。正在经历连续 PTO 的连接的寿命受到端点的空闲超时的限制。

发送方在每次发送 ACK 诱发包时计算其 PTO 计时器。如果发送方知道在短时间内将发送更多 ACK 诱发包，则可以通过设置更少的计时器来优化此操作。

如果设置了时间阈值 [5.1.2 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#time-threshold) 丢失检测计时器，则不设置探测计时器。时间阈值丢失检测计时器预期将比 PTO 更早过期，并且不太可能造成虚假重传。

## 5.3 握手和新路径

新连接或新路径的初始探测超时应设置为初始 RTT的两倍。相同网络上恢复的连接应该（SHOULD ）使用前面的连接最后的经平滑处理的 RTT 值作为恢复的连接的初始 RTT。如果之前没有可用的 RTT，则应（SHOULD）将初始 RTT 设置为 500ms，导致 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298) 中建议的 1 秒初始超时。

连接可以（MAY）使用发送 PATH_CHALLENGE 和接收 PATH_RESPONSE 之间的延迟作为新路径的种子 initial_rtt，但不应（SHOULD NOT）将此延迟视为 RTT 样本。

在服务器验证路径上的客户端地址之前，它可以发送的数据量是有限的，如 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的 8.1 节所描述的。

初始加密时的数据必须在握手数据之前重新传输，而握手加密时的数据必须在任何 ApplicationData 数据之前重新传输。如果没有数据可以发送，那么PTO 警报必须在从客户端接收到数据后才能启动。

由于服务器可能被阻塞，直到从客户端接收到更多的数据包，所以客户端负责发送数据包来解除服务器阻塞，直到服务器完成地址验证（参考 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的第 8 节）。也就是说，如果客户端还没有收到一次握手或 1-RTT 数据包的确认，则客户端必须（MUST）设置探测计时器。

在握手完成之前，如果生成的 RTT 样本很少甚至没有，则可能是由于客户端的 RTT 估计值不正确而导致探测器计时器过期。为了让客户端改进它的 RTT估计，它发送的新包必须是 ACK 诱发包。如果握手密钥对客户端可用，它必须（MUST）发送一个握手包，否则它必须（MUST）在 UDP 数据报中发送一个至少 1200 字节的初始数据包。

初始包和握手包可能永远不会被确认，但是当初始和握手密钥被丢弃时，它们将被从在途字节中删除。

### 5.3.1 发送探测包

当 PTO 计时器过期时，发送方必须发送至少一个 ACK 诱发包作为探测，除非没有可用的数据可供发送。一个端点最多可以（MAY）发送两个完整大小的数据报，其中包含 ACK 诱发包，以避免由于单个丢失的数据报而导致昂贵的连续 PTO 过期。

发送方可能没有新的或以前发送过的数据要发送。比如，考虑下列事件序列：新的应用程序数据在STREAM 帧中发送，被丢失，然后在一个新的数据包中重传，然后最初的传输被确认。在没有任何新的应用程序数据的情况下，PTO 定时器到期现在会发现发送者没有新的或以前发送过的数据要发送。

当没有要发送的数据时，发送方应该（SHOULD）在单个包中发送 PING 或其它 ACK 诱发帧，重新设置 PTO 计时器。

另一种方法是，发送方可以（MAY）将任何仍在传输中的包标记为丢失，而不是发送 ACK 诱发包。这样做可以避免发送额外的数据包，但是会增加由于过于频繁地声明丢失而导致拥塞控制器不必要地降低速率的风险。

连续的 PTO 周期呈指数级增长，因此，随着网络中数据包的不断丢失，连接恢复延迟也呈指数级增长。在 PTO 过期时发送两个数据包可以提高对数据包丢失的弹性，从而降低连续发生 PTO 事件的概率。

PTO 时发送的探测包必须是 ACK 诱发包。如果可能，探测包应该（SHOULD）携带新数据。当新数据不可用时，当流控制不允许发送新数据时，或者为了减少丢失恢复延迟时，探测包可以（MAY）携带重传的未确认数据。实现可以（MAY）使用其他策略来确定探测包的内容，包括根据应用程序的优先级发送新的或重传的数据。

当 PTO 计时器多次过期且无法发送新数据时，实现必须在每次发送相同的有效负载和发送不同的有效负载之间进行选择。发送相同的有效负载可能更简单，并确保优先级最高的帧先到达。每次发送不同的有效载荷可以减少虚假重传的机会。

### 5.3.2 丢失探测

当接收到一个 ACK 帧，该帧新确认一个或多个数据包时，就建立了数据包的传输或丢失。

PTO 定时器过期事件不表示包丢失，并且不能（MUST NOT）导致先前未确认的包被标记为丢失。当接收到新确认了包的确认包时，根据包和时间阈值机制进行丢失检测；参见 [5.1 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#ack-loss-detection)。

## 5.4 重试和版本协商

重试或版本协商包会导致客户端发送另一个初始包，从而有效地重新启动连接进程，并重新设置拥塞控制和丢失恢复状态，包括重新设置任何挂起的计时器。这两个包都表示收到了初始值，但没有进行处理。两个包都不能被当作是对初始包的确认。

但是，客户端可以（MAY ）计算到服务器的 RTT 估计值，为从发送第一个初始值到收到重试或版本协商包的时间内。客户端可以（MAY ）使用此值为 RTT 估计器的种子，用于随后到服务器的连接重试。

## 5.5 丢弃密钥和包状态

当包保护密钥被丢弃时（参考 [[QUIC-TLS]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TLS) 的 4.9 节），所有使用这些密钥发送的数据包都不能再被确认，因为它们的确认不能再被处理。发送方必须（MUST）丢弃与这些包关联的所有恢复状态，并且必须（MUST）从在途字节数中删除它们。

端点一旦开始交换握手包，就停止发送和接收初始包（参考 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 的 17.2.2.1 节）。此时，所有在途的初始包的恢复状态都被丢弃。

当 0-RTT 被拒绝时，所有在途的 0-RTT 数据包的恢复状态将被丢弃。

如果服务器接受了 0-RTT，但没有缓存在初始包之前到达的 0-RTT 包，早期 0-RTT 数据包将被声明丢失，但预计这种情况不会经常发生。

使用密钥加密的数据包被确认或声明丢失后，密钥将被丢弃。然而，一旦握手密钥可用，最初的秘密可能很快就会被销毁（参考 [[QUIC-TLS]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TLS) 的 4.9.1 节）。

## 5.6 讨论

大多数常量来自于 internet 上广泛部署的 TCP 实现中的最佳公共实践。例外如下。

选择较短的延迟 ack 时间为 25ms，因为较长的延迟 ack 可能延迟丢失恢复，并且对于少于每 25ms 发送一次数据包的少量连接，确认每个数据包有利于拥塞控制和丢失恢复。

# 6. 拥塞控制

QUIC 的拥塞控制基于 TCP NewReno [[RFC6582]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6582)。NewReno 是一种基于拥塞窗口的拥塞控制。QUIC 以字节而不是包的形式指定拥塞窗口，这是由于更好的控制和易于进行适当的字节计数 [[RFC3465]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC3465)。

如果要增加 bytes_in_flight (定义在 [附录 B.2](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#vars-of-interest) 中) 将超出可用的拥塞窗口，QUIC 主机一定不能（MUST NOT）发送数据包，除非该数据包是在 PTO 定时器过期后发送的探测数据包，如 [5.2 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#pto) 所述。

实现可以（MAY）使用其它的拥塞控制算法，比如 Cubic [[RFC8312]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC8312)，端点之间可能（MAY）使用不同的算法。QUIC 为拥塞控制提供的信号是通用的，并被设计来支持不同的算法。

## 6.1 显示拥塞通告

如果路径已经被验证支持 ECN，QUIC 将 IP 报头中有拥塞经验的codepoint 视为拥塞的信号。此文档描述当其对端接收到具有拥塞经验的codepoint 的包时端点的响应。如 [[RFC8311]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC8311) 中的讨论，允许端点使用其它响应函数进行实验。

## 6.2 慢启动

QUIC 以慢启动的方式启动每个连接，当丢包或增加进 ECN-CE 计数器中时退出慢启动。当拥塞窗口小于 ssthresh 时，QUIC重新进入慢启动，这只有在声明了持久拥塞之后才会发生。尽管处于慢启动中，QUIC 依然在每个确认被处理时增加拥塞窗口大小被确认的字节数。

## 6.3 拥塞避免

慢启动退出进入到拥塞避免。NewReno 中的拥塞避免使用了一种加法增加乘法减少 (AIMD) 的方法，它在每次拥塞窗口被确认时给拥塞窗口增加一个最大的包大小。当探测到丢包时，NewReno 给拥塞窗口减半，并把慢启动阈值设置为新的拥塞窗口。

## 6.4 恢复周期

恢复是从检测到丢失的包或 ECN-CE计 数器中的增加开始的一段时间。由于 QUIC 不重传数据包，所以它将恢复结束定义为恢复开始后发送的一个数据包被确认。这与 TCP 的恢复的定义稍微有点不同，它在启动了恢复的丢失的数据包被确认时结束。

恢复期将拥塞窗口的减少限制为每往返一次。在恢复期间，无论是新的丢包还是 ECN-CE 计数器增加，拥塞窗口保持不变。

## 6.5 忽略不可解密的包的丢失

在握手期间，某些包保护密钥可能在包到达时不可用。特别是，握手和 0-RTT 包在初始包到达之前无法处理，1-RTT 包在握手完成之前无法处理。端点可以忽略丢失的握手、0-RTT 和 1-RTT包，这些包可能在对等方拥有处理这些包的包保护密钥之前到达。

## 6.6 探测超时

探测数据包不能（MUST NOT）被拥塞控制器阻塞。但是，发送方必须（MUST）将这些包计入正在发送中的包，因为这些包增加了网络负载，但没有造成包丢失。请注意，发送探测数据包可能会导致发送方的在途字节超过拥塞窗口，直到接收到确认，确定数据包丢失或发送。

## 6.7 持续拥塞

当接收到一个 ACK 帧，该帧确定在足够长的时间内发送的所有在途数据包丢失，则认为网络正在经历持久拥塞。通常，这可以由连续的 PTOs 确定，但是由于 PTO 定时器在发送一个新 ACK 诱发包时被重置，所以必须使用一个显式的持续时间来处理那些 PTOs 没有发生或被大量延迟的情况。这个持续时间的计算如下：
```
(smoothed_rtt + 4 * rttvar + max_ack_delay) *
    kPersistentCongestionThreshold
```

比如，假设：
```
smoothed_rtt = 1 rttvar = 0 max_ack_delay = 0 kPersistentCongestionThreshold = 3
```

如果一个 ACK 诱发包在 time = 0 时发送，则下面的场景将演示持续拥塞：

| t=0 | Send Pkt #1 (App Data) |
|--|--|
| t=1 | Send Pkt #2 (PTO 1) |
| t=3 | Send Pkt #3 (PTO 2) |
| t=7 | Send Pkt #4 (PTO 3) |
| t=8 | Recv ACK of Pkt #4 |

当在 t=8 处收到包 4 的 ACK 时，前三个包被确定为丢失。拥塞周期的计算方法是最老的和最新的丢失的包之间的时间：(3 - 0)= 3。持续拥塞的持续时间等于：(1 * kPersistentCongestionThreshold) = 3。由于达到了阈值，并且在最老的和最新的包之间没有一个包得到确认，所以网络被认为经历了持久拥塞。

当确认持久拥塞时，发送方的拥塞窗口必须（MUST）减小到最小拥塞窗口(kMinimumWindow)。这种在持久拥塞时断崖式减小拥塞窗口的响应，在功能上类似于 TCP [[RFC5681]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5681) 中的尾丢失探测(TLP)  [[RACK]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RACK) 后，发送方对重传超时 (RTO) 的响应。

## 6.8 平滑（步调，发送节奏）

本文档没有描述 pacer，但建议（RECOMMENDED）发送方根据拥塞控制器的输入对所有在途数据包进行平滑发送。例如，当与基于窗口的控制器一起使用时，pacer 可以将拥塞窗口分布在经过平滑处理的 RTT 上，而pacer 可以使用基于速率的控制器的速率估计值。

实现应该小心地设计它的拥塞控制器，使其能够很好地与 pacer 一起工作。例如，pacer 可能会封装拥塞控制器并控制拥塞窗口的可用性，或者pacer 可能会把由拥塞控制器传递给它的包平滑发出。ACK 帧的及时送达对于有效的丢失恢复是很重要的。因此，只包含 ACK 帧的包不应该被 paced，以避免延迟它们被发送给对等方。

在网络中不加任何延迟地发送多个数据包会导致数据包爆发，这可能会造成短期的拥塞和丢包。实现必须（MUST）使用 pacing，或者将这种突发限制在最小 10 * kMaxDatagramSize和max(2* kMaxDatagramSize, 14720)，与推荐的初始拥塞窗口相同。

作为一个众所周知且公开可用的流 pacer 实现的例子，实现者可以参考 Linux (3.11 以后) 中的公平队列包调度器 (fq qdisc)。

## 6.9 未充分利用的拥塞窗口

当传输中的字节小于拥塞窗口，且发送没有速度限制时，则拥塞窗口未得到充分利用。当出现这种情况时，无论是慢启动还是拥塞避免，都不应该（SHOULD NOT）增加拥塞窗口。这可能是由于应用程序数据或流控制信用不足造成的。

发送方可以（MAY）使用 [[RFC7661]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC7661) 第 4.3 节中描述的pipeACK 方法来确定拥塞窗口是否被充分利用。

Pace 数据包 (参见[第6.8节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#pacing)) 的发送方，可能会延迟发送包，并且由于这种延迟而不能充分利用拥塞窗口。发送方不应该认为自己的应用程序有限，如果它将充分利用拥塞窗口而没有 pacing 延迟。

发送方可以（MAY）实现替代机制来在利用率不足的时期之后更新其拥塞窗口，例如 [[RFC7661]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC7661) 中为TCP提出的机制。

# 7. 安全注意事项

## 7.1 拥塞信号

拥塞控制本质上涉及使用来自未经身份验证的实体的信号 - 丢包和 ECN codepoints 都包括。路径上的攻击者可能欺骗或修改这些信号。攻击者可能通过丢包导致端点降低它们的速率，或通过修改 ECN codepoints 改变发送速率。

## 7.2 流量分析

只携带 ACK 帧的包可以通过观察包的大小来直观地识别。确认模式可能会暴露有关链接特征或应用程序行为的信息。端点可以使用 PADDING 帧或将确认与其它帧绑定以减少泄漏信息。

## 7.3 误报 ECN 标记

接收方可能误报 ECN 标记来改变发送方的拥塞响应。取消 ECN-CE 标记的报告可能会导致发送方增加其发送速率。这种增加可能导致拥塞和丢包。

发送方可以（MAY）尝试通过标记偶尔与 ECN-CE 一起发送的包来检测报告是否被抑制。如果使用 ECN-CE 标记的包在确认时没有被报告为已被标记，则发送方应（SHOULD）针对该路径禁用 ECN。

报告额外的 ECN-CE 标记将导致发送方降低其发送速率，这在效果上类似于广告减少连接流控制限制，因此这样做没有任何好处。

端点选择它们使用的拥塞控制器。尽管拥塞控制器通常将 ECN-CE 标记的报告视为等同于丢包[[RFC8311]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC8311)，但每个控制器的确切响应可能不同。因此，无法正确响应有关 ECN 标记的信息是很难检测到的。

## 8. IANA 注意事项

本文档没有 IANA 操作。然而。

# 9. 参考文献

## 9.1 引用标准

| [QUIC-TLS] | Thomson, M. and S. Turner, "[Using TLS to Secure QUIC](https://tools.ietf.org/html/draft-ietf-quic-tls)", Internet-Draft draft-ietf-quic-tls, October 2019. |
|--|--|
| [QUIC-TRANSPORT] | Iyengar, J. and M. Thomson, "[QUIC: A UDP-Based Multiplexed and Secure Transport](https://tools.ietf.org/html/draft-ietf-quic-transport)", Internet-Draft draft-ietf-quic-transport, October 2019. |
| [RFC2119] | Bradner, S., "[Key words for use in RFCs to Indicate Requirement Levels](https://tools.ietf.org/html/rfc2119)", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997. |
| [RFC8174] | Leiba, B., "[Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://tools.ietf.org/html/rfc8174)", BCP 14, RFC 8174, DOI 10.17487/RFC8174, May 2017. |
| [RFC8311] | Black, D., "[Relaxing Restrictions on Explicit Congestion Notification (ECN) Experimentation](https://tools.ietf.org/html/rfc8311)", RFC 8311, DOI 10.17487/RFC8311, January 2018. |

## 9.2 参考资料

| [FACK] | Mathis, M. and J. Mahdavi, "Forward Acknowledgement: Refining TCP Congestion Control", ACM SIGCOMM , August 1996. |
|--|--|
| [RACK] | Cheng, Y., Cardwell, N., Dukkipati, N. and P. Jha, "[RACK: a time-based fast loss detection algorithm for TCP](https://tools.ietf.org/html/draft-ietf-tcpm-rack-05)", Internet-Draft draft-ietf-tcpm-rack-05, April 2019. |
| [RFC3465] | Allman, M., "[TCP Congestion Control with Appropriate Byte Counting (ABC)](https://tools.ietf.org/html/rfc3465)", RFC 3465, DOI 10.17487/RFC3465, February 2003. |
| [RFC4653] | Bhandarkar, S., Reddy, A., Allman, M. and E. Blanton, "[Improving the Robustness of TCP to Non-Congestion Events](https://tools.ietf.org/html/rfc4653)", RFC 4653, DOI 10.17487/RFC4653, August 2006. |
| [RFC5681] | Allman, M., Paxson, V. and E. Blanton, "[TCP Congestion Control](https://tools.ietf.org/html/rfc5681)", RFC 5681, DOI 10.17487/RFC5681, September 2009. |
| [RFC5682] | Sarolahti, P., Kojo, M., Yamamoto, K. and M. Hata, "[Forward RTO-Recovery (F-RTO): An Algorithm for Detecting Spurious Retransmission Timeouts with TCP](https://tools.ietf.org/html/rfc5682)", RFC 5682, DOI 10.17487/RFC5682, September 2009. |
| [RFC5827] | Allman, M., Avrachenkov, K., Ayesta, U., Blanton, J. and P. Hurtig, "[Early Retransmit for TCP and Stream Control Transmission Protocol (SCTP)](https://tools.ietf.org/html/rfc5827)", RFC 5827, DOI 10.17487/RFC5827, May 2010. |
| [RFC6298] | Paxson, V., Allman, M., Chu, J. and M. Sargent, "[Computing TCP's Retransmission Timer](https://tools.ietf.org/html/rfc6298)", RFC 6298, DOI 10.17487/RFC6298, June 2011. |
| [RFC6582] | Henderson, T., Floyd, S., Gurtov, A. and Y. Nishida, "[The NewReno Modification to TCP's Fast Recovery Algorithm](https://tools.ietf.org/html/rfc6582)", RFC 6582, DOI 10.17487/RFC6582, April 2012. |
| [RFC6675] | Blanton, E., Allman, M., Wang, L., Jarvinen, I., Kojo, M. and Y. Nishida, "[A Conservative Loss Recovery Algorithm Based on Selective Acknowledgment (SACK) for TCP](https://tools.ietf.org/html/rfc6675)", RFC 6675, DOI 10.17487/RFC6675, August 2012. |
| [RFC6928] | Chu, J., Dukkipati, N., Cheng, Y. and M. Mathis, "[Increasing TCP's Initial Window](https://tools.ietf.org/html/rfc6928)", RFC 6928, DOI 10.17487/RFC6928, April 2013. |
| [RFC7661] | Fairhurst, G., Sathiaseelan, A. and R. Secchi, "[Updating TCP to Support Rate-Limited Traffic](https://tools.ietf.org/html/rfc7661)", RFC 7661, DOI 10.17487/RFC7661, October 2015. |
| [RFC8312] | Rhee, I., Xu, L., Ha, S., Zimmermann, A., Eggert, L. and R. Scheffenegger, "[CUBIC for Fast Long-Distance Networks](https://tools.ietf.org/html/rfc8312)", RFC 8312, DOI 10.17487/RFC8312, February 2018. |

# 附录 A. 丢包恢复伪代码

现在我们描述一个 [第 6 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#loss-detection) 所述的丢包探测机制的示例实现。

## A.1 跟踪发送数据包

### A.1.1 发送数据包字段

 packet_number：发送的数据包的包号。

 ack_eliciting：用于表示一个包是否是 ACK 诱发包的布尔值。如果为真，则预期将收到一个确认，尽管对端可能延迟发送包含它的 ACK 帧直到 MaxAckDelay。

 in_flight：用于表示包是否被计入 in flight 值的布尔值。

 sent_bytes：包中发送的字节数，不包括 UDP 或 IP 头部，但包含 QUIC 帧头部。

 time_sent：发送包的时间。

## A.2 重要的常量

丢包恢复中使用的常量基于 RFCs，论文，和常见实践的结合。一些可能需要改变或协商以便更好地适应各种环境。

 kPacketThreshold：包阈值丢失探测认为一个包丢失前最大的重排序包个数。RECOMMENDED （建议）值取 3。

 kTimeThreshold：时间阈值丢失探测认为一个包丢失前最大的重排序时间。以RTT 系数的方式指定。RECOMMENDED （建议）值取 9/8。

 kGranularity：计时器粒度。这是一个系统相关的值。然而，实现 SHOULD（应该）使用一个不小于 1 ms 的值。

 kInitialRtt：在取得一个 RTT 采样前使用的 RTT。RECOMMENDED （建议）值取 500ms。

 kPacketNumberSpace：列举三种包号空间的枚举。
```
  enum kPacketNumberSpace {
    Initial,
    Handshake,
    ApplicationData,
  }
```

## A.3 重要的变量

实现拥塞控制机制所需的变量在本节描述：

latest_rtt：当接收前一个未确认包的 ack 时执行最新的 RTT 测量。

smoothed_rtt：连接的平滑 RTT，如 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298) 所述的方式计算。

rttvar：RTT 方差，如 [[RFC6298]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6298) 所述的方式计算。

min_rtt：连接看到的最小 RTT，忽略 ack 延时。

max_ack_delay：接收者预期的 ApplicationData 包号空间中的包的延迟确认的最大时间。接收的 ACK 帧中实际的 ack_delay 可能由于 late timers，重排序，或 ACKs 丢失而更大。

loss_detection_timer：用于丢包探测的多模态计时器。

pto_count：在没有收到 ack 的情况下发送 PTO 的次数。

time_of_last_sent_ack_eliciting_packet：最近的 ack 诱发包发送的时间。

largest_acked_packet[kPacketNumberSpace]：目前为止包号空间中已经确认的最大包号。

loss_time[kPacketNumberSpace]：该包号空间中的下一个包被认为丢失的时间，根据时间上超过了重排序窗口。

sent_packets[kPacketNumberSpace]：包号空间中的包号与有关它们的信息的关联。在上面的 [附录 A.1](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#tracking-sent-packets) 中描述。

## A.4 初始化

在连接开始的时候，按如下方式初始化丢包探测变量：
```
   loss_detection_timer.reset()
   pto_count = 0
   latest_rtt = 0
   smoothed_rtt = 0
   rttvar = 0
   min_rtt = 0
   max_ack_delay = 0
   time_of_last_sent_ack_eliciting_packet = 0
   for pn_space in [ Initial, Handshake, ApplicationData ]:
     largest_acked_packet[pn_space] = infinite
     loss_time[pn_space] = 0
```

## A.5 发送一个包

发送一个包之后，存储关于包的信息。OnPacketSent 的参数在上面的 [附录 A.1.1](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#sent-packets-fields) 中详细描述。

OnPacketSent 的伪代码如下：
```
 OnPacketSent(packet_number, pn_space, ack_eliciting,
              in_flight, sent_bytes):
   sent_packets[pn_space][packet_number].packet_number =
                                            packet_number
   sent_packets[pn_space][packet_number].time_sent = now
   sent_packets[pn_space][packet_number].ack_eliciting =
                                            ack_eliciting
   sent_packets[pn_space][packet_number].in_flight = in_flight
   if (in_flight):
     if (ack_eliciting):
       time_of_last_sent_ack_eliciting_packet = now
     OnPacketSentCC(sent_bytes)
     sent_packets[pn_space][packet_number].size = sent_bytes
     SetLossDetectionTimer()
```

## A.6 接收到一个确认

当接收到一个 ACK 帧时，它可以新确认任意数量的包。OnAckReceived 和 UpdateRtt 的伪代码如下：
```
OnAckReceived(ack, pn_space):
  if (largest_acked_packet[pn_space] == infinite):
    largest_acked_packet[pn_space] = ack.largest_acked
  else:
    largest_acked_packet[pn_space] =
        max(largest_acked_packet[pn_space], ack.largest_acked)

  // Nothing to do if there are no newly acked packets.
  newly_acked_packets = DetermineNewlyAckedPackets(ack, pn_space)
  if (newly_acked_packets.empty()):
    return

  // If the largest acknowledged is newly acked and
  // at least one ack-eliciting was newly acked, update the RTT.
  if (sent_packets[pn_space][ack.largest_acked] &&
      IncludesAckEliciting(newly_acked_packets)):
    latest_rtt =
      now - sent_packets[pn_space][ack.largest_acked].time_sent
    ack_delay = 0
    if (pn_space == ApplicationData):
      ack_delay = ack.ack_delay
    UpdateRtt(ack_delay)

  // Process ECN information if present.
  if (ACK frame contains ECN information):
      ProcessECN(ack, pn_space)

  for acked_packet in newly_acked_packets:
    OnPacketAcked(acked_packet.packet_number, pn_space)

  DetectLostPackets(pn_space)

  pto_count = 0

  SetLossDetectionTimer()


UpdateRtt(ack_delay):
  // First RTT sample.
  if (smoothed_rtt == 0):
    min_rtt = latest_rtt
    smoothed_rtt = latest_rtt
    rttvar = latest_rtt / 2
    return

  // min_rtt ignores ack delay.
  min_rtt = min(min_rtt, latest_rtt)
  // Limit ack_delay by max_ack_delay
  ack_delay = min(ack_delay, max_ack_delay)
  // Adjust for ack delay if plausible.
  adjusted_rtt = latest_rtt
  if (latest_rtt > min_rtt + ack_delay):
    adjusted_rtt = latest_rtt - ack_delay

  rttvar = 3/4 * rttvar + 1/4 * abs(smoothed_rtt - adjusted_rtt)
  smoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
```

## A.7 包被确认时

当包第一次被确认时，如下的 OnPacketAcked 函数被调用。注意，单个 ACK 帧可能新确认多个包。OnPacketAcked 必须为这些新确认的包中的每一个都调用一次。

OnPacketAcked 接收两个参数：acked_packet，它是 [附录 A.1.1](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#sent-packets-fields) 中详述的结构体，以及这个 ACK 帧发送到的目标包号空间。

OnPacketAcked 的伪代码如下：
```
   OnPacketAcked(acked_packet, pn_space):
     if (acked_packet.in_flight):
       OnPacketAckedCC(acked_packet)
     sent_packets[pn_space].remove(acked_packet.packet_number)
```

## A.8 设置丢包探测定时器

QUIC 丢包探测为所有的超时丢包探测使用一个单独的定时器。计时器的持续时间基于计时器的模式，该模式在下面的包和计时器事件中设置。下面定义的 SetLossDetectionTimer 函数展示了如何设置单个定时器。

这种算法可能会导致定时器设置过时，特别是如果定时器唤醒晚了。设置过时的定时器 SHOULD 立即启动。

SetLossDetectionTimer 的伪代码如下：
```
// Returns the earliest loss_time and the packet number
// space it's from.  Returns 0 if all times are 0.
GetEarliestLossTime():
  time = loss_time[Initial]
  space = Initial
  for pn_space in [ Handshake, ApplicationData ]:
    if (loss_time[pn_space] != 0 &&
        (time == 0 || loss_time[pn_space] < time)):
      time = loss_time[pn_space];
      space = pn_space
  return time, space

SetLossDetectionTimer():
  loss_time, _ = GetEarliestLossTime()
  if (loss_time != 0):
    // Time threshold loss detection.
    loss_detection_timer.update(loss_time)
    return

  // Don't arm timer if there are no ack-eliciting packets
  // in flight and the handshake is complete.
  if (endpoint is client with 1-RTT keys &&
      no ack-eliciting packets in flight):
    loss_detection_timer.cancel()
    return

  // Use a default timeout if there are no RTT measurements
  if (smoothed_rtt == 0):
    timeout = 2 * kInitialRtt
  else:
    // Calculate PTO duration
    timeout = smoothed_rtt + max(4 * rttvar, kGranularity) +
      max_ack_delay
  timeout = timeout * (2 ^ pto_count)

  loss_detection_timer.update(
    time_of_last_sent_ack_eliciting_packet + timeout)
```

## A.9 超时

当丢包探测定时器超时时，定时器的模式决定了将被执行的行为。

OnLossDetectionTimeout 的伪代码如下：
```
OnLossDetectionTimeout():
  loss_time, pn_space = GetEarliestLossTime()
  if (loss_time != 0):
    // Time threshold loss Detection
    DetectLostPackets(pn_space)
    SetLossDetectionTimer()
    return

  if (endpoint is client without 1-RTT keys):
    // Client sends an anti-deadlock packet: Initial is padded
    // to earn more anti-amplification credit,
    // a Handshake packet proves address ownership.
    if (has Handshake keys):
      SendOneAckElicitingHandshakePacket()
    else:
      SendOneAckElicitingPaddedInitialPacket()
  else:
    // PTO. Send new data if available, else retransmit old data.
    // If neither is available, send a single PING frame.
    SendOneOrTwoAckElicitingPackets()

  pto_count++
  SetLossDetectionTimer()
```

## A.10 探测丢失的包

每次收到 ACK 都调用 DetectLostPackets 并操作该包号空间上的 sent_packets。

DetectLostPackets 的伪代码如下：
```
DetectLostPackets(pn_space):
  assert(largest_acked_packet[pn_space] != infinite)
  loss_time[pn_space] = 0
  lost_packets = {}
  loss_delay = kTimeThreshold * max(latest_rtt, smoothed_rtt)

  // Minimum time of kGranularity before packets are deemed lost.
  loss_delay = max(loss_delay, kGranularity)

  // Packets sent before this time are deemed lost.
  lost_send_time = now() - loss_delay

  foreach unacked in sent_packets[pn_space]:
    if (unacked.packet_number > largest_acked_packet[pn_space]):
      continue

    // Mark packet as lost, or set time when it should be marked.
    if (unacked.time_sent <= lost_send_time ||
        largest_acked_packet[pn_space] >=
          unacked.packet_number + kPacketThreshold):
      sent_packets[pn_space].remove(unacked.packet_number)
      if (unacked.in_flight):
        lost_packets.insert(unacked)
    else:
      if (loss_time[pn_space] == 0):
        loss_time[pn_space] = unacked.time_sent + loss_delay
      else:
        loss_time[pn_space] = min(loss_time[pn_space],
                                  unacked.time_sent + loss_delay)

  // Inform the congestion controller of lost packets and
  // let it decide whether to retransmit immediately.
  if (!lost_packets.empty()):
    OnPacketsLost(lost_packets)
```

# 附录 B. 拥塞控制伪代码

我们现在描述一个 [第 7 节](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#congestion-control) 中描述的拥塞控制器的示例实现。

## B.1 重要的常量

拥塞控制中使用的常量是基于 RFCs，论文，和习惯做法确定的。其中的一些可能需要修改或协商，以更好地适用于各种各样的环境。

kMaxDatagramSize：发送者的最大载荷大小。不包括 UDP 头和 IP 头。最大包大小用于计算初始的和最小的拥塞窗口。RECOMMENDED（建议）值取 1200。

kInitialWindow：初始在途数据量的默认限制，以字节为单位。取自 [[RFC6928]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC6928)，但考虑到 8 字节的 UDP 头和 20 字节的 TCP 头可轻微增加。RECOMMENDED（建议）值取 10 * kMaxDatagramSize 和 max(2* kMaxDatagramSize, 14720)) 两者中小的那个。

kMinimumWindow：最小的拥塞窗口字节数。RECOMMENDED（建议）值取 2 * kMaxDatagramSize。

kLossReductionFactor：当探测到一个新的丢包事件时减小拥塞窗口。RECOMMENDED（建议）值取 0.5。

kPersistentCongestionThreshold：持续拥塞的建立时间，以 PTO 的倍数指定。这个阈值的原理是，让一个发送者使用初始 PTOs 积极地探索，像 TCP 通过 Tail Loss Probe (TLP) [[TLP]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#TLP) [[RACK]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RACK) 做的那样，在建立持久拥塞之前，像 TCP 通过 Retransmission Timeout (RTO) [[RFC5681]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#RFC5681) 做的那样。RECOMMENDED（建议）kPersistentCongestionThreshold 的值取 3，它接近于等于 TCP 中一个 RTO 前有两个 TLPs。

## B.2 重要的变量

实现拥塞控制机制所需的变量在本节描述。

ecn_ce_counters[kPacketNumberSpace]：对端在一个 ACK 帧中报告的包号空间中的 ECN-CE 计数器的最高值。这个值用于探测报告的 ECN-CE 计数的增量。

bytes_in_flight：所有已经发送的包含至少一个 ack 诱发帧或 PADDING 帧，但还没有被确认或声明丢失的包的总字节大小。这个大小不包括 IP 或 UDP 头，但是包括 QUIC 和 AEAD 头。只包含 ACK 帧的包不计入 bytes_in_flight，以确保拥塞控制不妨碍拥塞反馈。

congestion_window：可以发送的在途字节的最大大小。

congestion_recovery_start_time：QUIC 由于丢包或 ECN 首次探测到拥塞的时间，导致它进入拥塞恢复状态。当这个时间之后发送的包被确认时，QUIC 退出拥塞恢复状态。

ssthresh：慢启动阈值的字节数。当拥塞窗口低于 ssthresh 时，模式为慢启动，且窗口随着确认的字节数而增长。

## B.3 初始化

连接开始时，按如下方式初始化拥塞控制变量：
```
   congestion_window = kInitialWindow
   bytes_in_flight = 0
   congestion_recovery_start_time = 0
   ssthresh = infinite
   for pn_space in [ Initial, Handshake, ApplicationData ]:
     ecn_ce_counters[pn_space] = 0
```

## B.4 发送包

无论何时发送一个包，且它包含非 ACK 帧，则包增加 bytes_in_flight。
```
   OnPacketSentCC(bytes_sent):
     bytes_in_flight += bytes_sent
```

## B.5 包确认时

在丢包探测的 OnPacketAcked 中调用，并提供 sent_packets 的 acked_packet。
```
   InCongestionRecovery(sent_time):
     return sent_time <= congestion_recovery_start_time

   OnPacketAckedCC(acked_packet):
     // Remove from bytes_in_flight.
     bytes_in_flight -= acked_packet.size
     if (InCongestionRecovery(acked_packet.time_sent)):
       // Do not increase congestion window in recovery period.
       return
     if (IsAppLimited()):
       // Do not increase congestion_window if application
       // limited.
       return
     if (congestion_window < ssthresh):
       // Slow start.
       congestion_window += acked_packet.size
     else:
       // Congestion avoidance.
       congestion_window += kMaxDatagramSize * acked_packet.size
           / congestion_window
```

## B.6 新拥塞事件发生时

当探测到一个新的拥塞事件时在 ProcessECN 和 OnPacketsLost 中调用。可能启动一个新的恢复周期并减小拥塞窗口。
```
   CongestionEvent(sent_time):
     // Start a new congestion event if packet was sent after the
     // start of the previous congestion recovery period.
     if (!InCongestionRecovery(sent_time)):
       congestion_recovery_start_time = Now()
       congestion_window *= kLossReductionFactor
       congestion_window = max(congestion_window, kMinimumWindow)
       ssthresh = congestion_window
```

## B.7 处理 ECN 信息

当从对端接收到一个带有 ECN 段的 ACK 帧时调用。
```
   ProcessECN(ack, pn_space):
     // If the ECN-CE counter reported by the peer has increased,
     // this could be a new congestion event.
     if (ack.ce_counter > ecn_ce_counters[pn_space]):
       ecn_ce_counters[pn_space] = ack.ce_counter
       CongestionEvent(sent_packets[ack.largest_acked].time_sent)
```

## B.8 On Packets Lost

当包被认为丢失时，在 DetectLostPackets 中调用。
```
   InPersistentCongestion(largest_lost_packet):
     pto = smoothed_rtt + max(4 * rttvar, kGranularity) +
       max_ack_delay
     congestion_period = pto * kPersistentCongestionThreshold
     // Determine if all packets in the time period before the
     // newest lost packet, including the edges, are marked
     // lost
     return AreAllPacketsLost(largest_lost_packet,
                              congestion_period)

   OnPacketsLost(lost_packets):
     // Remove lost packets from bytes_in_flight.
     for (lost_packet : lost_packets):
       bytes_in_flight -= lost_packet.size
     largest_lost_packet = lost_packets.last()
     CongestionEvent(largest_lost_packet.time_sent)

     // Collapse congestion window if persistent congestion
     if (InPersistentCongestion(largest_lost_packet)):
       congestion_window = kMinimumWindow
```

[QUIC Protocol 相关文档中文翻译](https://github.com/quicwg/zh-translations)

[原文链接](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html)
