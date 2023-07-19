---
title: 音频时钟
date: 2023-04-29 19:07:29
categories: Linux 内核
tags:
- Linux 内核
---

本文介绍 ASoC 和数字音频中的音频时钟术语。注意：音频时钟可能很复杂！

## 主时钟

每个音频子系统都由一个主时钟 (有时称为 MCLK 或 SYSCLK) 驱动。这种音频主时钟可以从许多时钟源 (比如晶振，PLL，CPU 时钟) 派生，并负责产生正确的音频播放和采集采样率。

一些主时钟 (例如 PLL 和基于 CPU 的时钟) 是可配置的，因为软件可以改变他们的速度 (依赖于系统使用和节能)。其它主时钟固定在一个设定的频率 (如晶振)。

## DAI 时钟

数字音频接口 (Digital Audio Interface) 通常由位时钟 (常常被称为 BCLK) 驱动。这种时钟用于驱动数字音频数据通过 codec 和 CPU 之间的链路。

DAI 还有一个帧时钟来通知每个音频帧的开始。这个时钟有时被称为 LRC (左右时钟) 或 FRAME。这个时钟以精确的采样率 (LRC = Rate) 运行。

位时钟可以如下方式生成：

- BCLK = MCLK / x，或
- BCLK = LRC * x，或
- BCLK = LRC * Channels * Word Size

这种关系特别取决于 codec 或 SoC CPU。通常最好将 BCLK 配置为最低的可能速度 (取决于你的采样率，通道数和字大小) 以节省电源。

如果可能的话，最好使用 codec 来驱动 (或控制) 音频时钟，因为它通常可以比 CPU 提供更准确的采样率。

[原文](linux-kernel/Documentation/sound/soc/clocking.rst)

Done.
