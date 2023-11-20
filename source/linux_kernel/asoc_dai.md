---
title: ASoC 数字音频接口 (DAI)
date: 2023-04-29 17:37:29
categories: Linux 内核
tags:
- Linux 内核
---

ASoC 当前主要支持关于 SoC 控制器和便携式音频 CODEC 的三种数字音频接口 (DAI)，即 AC97，I2S 和 PCM。

## AC97

AC97 是许多 PC 声卡上常见的五线接口。它现在在许多便携式设备上也很流行。这种 DAI 有一条复位线，并在它的 SDATA_OUT (播放) 和 SDATA_IN (采集) 线上对其数据进行分时复用。位时钟 (BCLK) 总是由 CODEC 驱动 (通常为 12.288MHz)，且帧 (FRAME) (通常为 48kHz) 总是由控制器驱动。每个 AC97 帧长 21uS，并分为 13 个时隙。

AC97 规范可在以下网址找到：https://www.intel.com/p/en_US/business/design 。

## I2S

I2S 是一种用于 HiFI，STB 和便携式设备的常见的 4 线 DAI。其 Tx 和 Rx 线用于音频传输，而位时钟 (BCLK) 和左/右时钟 (LRC) 同步链路。I2S 比较灵活，无论是控制器还是 CODEC 都可以驱动 (主) BCLK 和 LRC 时钟线。位时钟通常取决于采样率和主系统时钟 (SYSCLK)。一些设备支持单独的 ADC 和 DAC LRCLK，这允许以不同的采样率同时采集和播放。

I2S 有几种不同的工作模式：

I2S
  MSB 在 LRC 转换后的第一个 BCLK 下降沿上传输。

向左对齐 (Left Justified)
  MSB 在 LRC 转换时传输。

向右对齐 (Right Justified)
  MSB 在 LRC 转换前的采样大小个 BCLK 传输。

## PCM

PCM 是另一个 4 线接口，它与 I2S 非常相似，但它可以支持更加灵活的协议。它具有位时钟 (BCLK) 和同步 (SYNC) 线用于同步链路，而 Tx 和 Rx 线用于传输和接收音频数据。位时钟通常取决于采样率，而同步运行在采样率。PCM 还支持时分多路复用 (TDM)，多个设备可以同时使用总线 (这有时称为网络模式)。

常见的 PCM 操作模式：

模式 A
  MSB 在 FRAME/SYNC 之后的第一个 BCLK 下降沿传输。

模式 B
  MSB 在 FRAME/SYNC 的上升沿传输。

[原文](linux-kernel/Documentation/sound/soc/dai.rst)

Done.
