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

本文档中的关键字 “MUST”，”MUST NOT”，”REQUIRED”，”SHALL”，”SHALL NOT”，”SHOULD”，”SHOULD NOT”，”RECOMMENDED”，”MAY”，和 “OPTIONAL” 按 [BCP 14](https://tools.ietf.org/html/bcp14)，[RFC 2119](https://tools.ietf.org/html/rfc2119) [[2](https://tools.ietf.org/html/rfc3533#ref-2)] 中的描述解释，当且仅当它们如这里所示全大写出现的时候。

本文档中使用的术语定义如下：

 ACK-only：只包含一个或多个 ACK 帧的任何包。
 In-flight：当非 ACK-only 的包已经被发送，但还没有被确认、宣布丢失或连同旧的秘钥一起丢弃时，被认为是 in-flight 的。
 ACK 诱发帧：除 ACK 和 PADDING 外的所有帧被认为是 ACK 诱发帧。
 ACK 诱发包：包含 ACK 诱发帧的包在最大 ACK 延迟内诱发一个来自于接收者的 ACK，被称为 ACK 诱发包。
 加密包：在初始化或握手包中发送的包含加密数据的包。
 乱序包：没有为它的包号空间的最大接收包包号精确地递增一的包。当更早的包丢失或延迟时包乱序到达。

# 3. QUIC 传输机制的设计

QUIC 中所有的传输发送时都带有一个包级的头部，它指示加密级别并包含一个包序列号（下文称为包号）。加密级别指示包号空间，如 [[QUIC-TRANSPORT]](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html#QUIC-TRANSPORT) 所述。在一个连接的生命周期中的包号空间内包号从不重复。一个空间内包号单调递增，以防止歧义。

这种设计消除了在传输和重传之间的二义性消除的需要，并从 TCP 丢包探测机制的 QUIC 翻译中消除了可观的复杂性。

QUIC 包可以包含不同类型的多个帧。恢复机制确保需要可靠递送的数据和帧被确认或被宣布丢失，并在必要时在新包中发送。包中包含的帧的类型影响恢复和拥塞控制逻辑：

 * 所有的包都被确认，尽管包含非 ACK 诱发帧的包只与 ACK 诱发包一起确认。
 * 包含加密帧的长头部包对于 QUIC 握手的性能非常重要，因此使用更短的确认定时器。
 * 只包含 ACK 帧的包不计入拥塞控制限制，且不被认为是 in-flight 的。
 * PADDING 帧使得包贡献 in flight 的字节数，但不直接导致确认的发送。

## 3.1 QUIC 和 TCP 之间的相关差异

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

（Version 22）

[原文链接](https://quicwg.org/base-drafts/draft-ietf-quic-recovery.html)
