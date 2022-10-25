---
title: WebRTC NetEQ
date: 2022-10-24 19:53:47
categories: 音视频开发
tags:
- 音视频开发
---

NetEq 是音频的抖动缓冲区和丢包隐藏器。抖动缓冲区是一种自适应抖动缓冲器，即根据网络条件不断优化缓冲延迟。它的主要目标是确保从网络传入的音频数据包，以较少的音频伪影（对数据包的原始内容的更改）下平稳播放，同时保持尽可能低的延迟。
<!--more-->
## API

在较高的层面上看，NetEq API 有两个主要的函数：[`InsertPacket`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=198;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72) 和 [`GetAudio`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=219;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72)。

### InsertPacket

[`InsertPacket`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=198;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72) 将来自于网络的 RTP 数据包传递给 NetEq，其中将发生以下情况：

1. 如果对于播放来说，它来的太晚，则数据包将被丢弃（例如，如果它被重新排序，乱序到达）。否则它被放进数据包缓冲区保存起来，直到被播放出来。如果缓冲区满了，则丢弃所有已有的数据包（这应该很少发生）。

2. 分析数据包之间的到达时间间隔，并更新统计数据，以用于推导新的目标播放延迟。到达间隔时间以 GetAudio ‘ticks’ 的数量来衡量，因此可以照顾到发送方和接收方之间的时钟漂移。

### GetAudio

[`GetAudio`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=219;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72) 从 NetEq 拉取 10 ms 的音频数据来播放。一个大大简化的决策逻辑如下：

1. 如果 sync buffer 中有 10 ms 的音频数据，就返回它们。

2. 如果下一个数据包（基于 RTP 时间辍）在 packet buffer 中可用，则解码它并把结果添加到 sync buffer。
    1. 将当前延迟估计（过滤的缓冲区水位）与目标延迟进行对比，如果缓冲区水位太高或太低，则在时间上伸缩（加速或减速）sync buffer 的内容。
    2. 从 sync buffer 返回 10 ms 的音频数据。

3. 如果上一次解码的数据包是一个不连续传输（DTX）数据包，则产生舒适噪声。

4. 如果由于下一个数据包还没有到达或已经丢失而没有数据包可用于解码，则通过推断 sync buffer 中剩余的音频数据或通过请求解码器生成，来生成丢包隐藏。

总之，输出是以下操作之一的结果：

 * Normal：从数据包解码出来的音频数据。
 * Acceleration：解码数据包的加速播放。
 * Preemptive expand：解码数据包的减速播放。
 * Expand：由 NetEq 或解码器生成的丢包隐藏。
 * Merge：将丢包隐藏和解码的数据缝合起来以防丢失的音频数据。
 * Comfort noise (CNG)：由于数据包的不连续传输 (DTX)，由 NetEq 或解码器在谈话间断产生的舒适噪声。

## 统计数据

有大量的函数可用于查询 NetEq 的内部状态，关于音频输出类型，和诸如数据包已经在缓冲区中等待的时长这样的延迟指标之类的统计数据。

 * [`NetworkStatistics`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=273;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72)：自上次调用此函数以来持续时间内的瞬时值或统计平均值。
 * [`GetLifetimeStatistics`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=280;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72)：在类的生命周期中持续存在的累积统计信息。
 * [`GetOperationsAndState`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/api/neteq/neteq.h;l=284;drc=4461f059d180fe8c2886d422ebd1cb55b5c83e72)：关于 NetEq 内部状态的信息（仅用于测试和调试）。

## 测试和工具

 * [`neteq_rtpplay`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/modules/audio_coding/neteq/tools/neteq_rtpplay.cc;drc=cee751abff598fc19506f77de08bea7c61b9dcca)：基于 RTP 转储，PCAP 文件或 RTC 事件日志仿真 NetEq 行为。也可以使用替换音频文件代替原始有效负载。输出汇总的统计数据和可选的用以收听的音频文件。

 * [`neteq_speed_test`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/modules/audio_coding/neteq/test/neteq_speed_test.cc;drc=2ab97f6f8e27b47c0d9beeb8b6ca5387bda9f55c)：测量 NetEq 的性能，用于 perf 机器人。

 * 单元测试包括位精确性测试，其中 RTP 文件用作 NetEq 的输入，连接输出并计算校验和并与参考值进行比较。

## 其它职责

双音多频信令（DTMF）：接收电话事件并产生双音波形。

前向纠错（RED 或编解码器带内 FEC）：分割插入的数据包并对有效负载进行优先级排序。

NACK （negative acknowledgement，否定确认）：跟踪丢失的数据包，并为 NACK 生成数据包的列表。

音频/视频同步：NetEq 可以被指示增加延迟以保持音频和视频同步。

[原文](https://chromium.googlesource.com/external/webrtc/+/master/modules/audio_coding/neteq/g3doc/index.md)
