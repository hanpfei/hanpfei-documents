---
title: 为 ALSA dapm 创建 codec 到 codec 的链路
date: 2023-04-29 20:17:29
categories: Linux 内核
tags:
- Linux 内核
---

大多数情况下，音频流总是从 CPU 到 codec，所以你的系统看起来像下面这样：
```
   ---------          ---------
  |         |  dai   |         |
      CPU    ------->    codec
  |         |        |         |
   ---------          ---------
```

如果你的系统如下所示：
```
                       ---------
                      |         |
                        codec-2
                      |         |
                      ---------
                           |
                         dai-2
                           |
   ----------          ---------
  |          |  dai-1 |         |
      CPU     ------->  codec-1
  |          |        |         |
   ----------          ---------
                           |
                         dai-3
                           |
                       ---------
                      |         |
                        codec-3
                      |         |
                       ---------
```

假设 codec-2 是蓝牙芯片，codec-3 连接了一个扬声器，你有如下场景：

Codec-2 接收音频数据，用户想要在不涉及 CPU 的情况下通过 codec-3 把这些音频播放出来。上述场景是使用 codec 到 codec 连接的理想情况。

你的机器驱动程序文件中的 dai_link 应该像下面这样：
```
 /*
  * this pcm stream only supports 24 bit, 2 channel and
  * 48k sampling rate.
  */
 static const struct snd_soc_pcm_stream dsp_codec_params = {
        .formats = SNDRV_PCM_FMTBIT_S24_LE,
        .rate_min = 48000,
        .rate_max = 48000,
        .channels_min = 2,
        .channels_max = 2,
 };

 {
    .name = "CPU-DSP",
    .stream_name = "CPU-DSP",
    .cpu_dai_name = "samsung-i2s.0",
    .codec_name = "codec-2,
    .codec_dai_name = "codec-2-dai_name",
    .platform_name = "samsung-i2s.0",
    .dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF
            | SND_SOC_DAIFMT_CBM_CFM,
    .ignore_suspend = 1,
    .params = &dsp_codec_params,
 },
 {
    .name = "DSP-CODEC",
    .stream_name = "DSP-CODEC",
    .cpu_dai_name = "wm0010-sdi2",
    .codec_name = "codec-3,
    .codec_dai_name = "codec-3-dai_name",
    .dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF
            | SND_SOC_DAIFMT_CBM_CFM,
    .ignore_suspend = 1,
    .params = &dsp_codec_params,
 },
```

以上代码片段源于 *sound/soc/samsung/speyside.c*。

注意 "params" 回调，它让 dapm 了解到这个 dai_link 是一个 codec 到 codec 的连接。

在 dapm 核心中，在 cpu_dai 播放部件和 codec_dai 采集部件之间创建一条路由，用于播放路径，对于捕获路径反之亦然。为了触发上述路由，DAPM 需要找到一个有效的端点，它可能是分别对应于播放和采集路径的 sink 或 source 部件。

为了触发这种 dai_link 部件，可以为扬声器放大器创建瘦 codec 驱动程序，如 wm8727.c 文件中演示的那样，它为设备设置适当的约束，即使它不需要控制。

清确保分别为相应的 CPU 和 codec 播放和采集 DAI 命名为以 "Playback" 和 "Capture" 结尾，因为 dapm 核心将根据名称链接和启动这些 DAI。

当链接上的所有 DAI 都属于 codec 组件时，"simple-audio-card" 中的 dai_link 将自动检测为 codec 到 codec。dai_link 将使用链接上所有 DAI 支持的流参数的子集 (通道，格式，采样率) 初始化。由于无法在设备树中提供这些参数，因此这主要用于与简单的固定功能 codec，比如蓝牙控制器或蜂窝调制解调器进行通信。

[原文](linux-kernel/Documentation/sound/soc/codec-to-codec.rst)

Done.
