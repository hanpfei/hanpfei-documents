---
title: ASoC 平台驱动程序
date: 2023-04-29 16:47:29
categories: Linux 内核
tags:
- Linux 内核
---

一个 ASoC 平台类驱动程序可以被分拆为音频 DMA 驱动程序，SoC DAI 驱动程序和 DSP 驱动程序。平台驱动程序仅针对 SoC CPU，并且必须没有特定于板子的代码。

## 音频 DMA

平台 DMA 驱动程序可选地支持下列 ALSA 操作：
```
  /* SoC audio ops */
  struct snd_soc_ops {
	int (*startup)(struct snd_pcm_substream *);
	void (*shutdown)(struct snd_pcm_substream *);
	int (*hw_params)(struct snd_pcm_substream *, struct snd_pcm_hw_params *);
	int (*hw_free)(struct snd_pcm_substream *);
	int (*prepare)(struct snd_pcm_substream *);
	int (*trigger)(struct snd_pcm_substream *, int);
  };
```

平台驱动程序通过 `struct snd_soc_component_driver` 导出它的 DMA 功能：
```
  struct snd_soc_component_driver {
	const char *name;

	...
	int (*probe)(struct snd_soc_component *);
	void (*remove)(struct snd_soc_component *);
	int (*suspend)(struct snd_soc_component *);
	int (*resume)(struct snd_soc_component *);

	/* pcm creation and destruction */
	int (*pcm_new)(struct snd_soc_pcm_runtime *);
	void (*pcm_free)(struct snd_pcm *);

	...
	const struct snd_pcm_ops *ops;
	const struct snd_compr_ops *compr_ops;
	...
  };
```

请参考 ALSA 驱动程序文档了解音频 DMA 的详情。http://www.alsa-project.org/~iwai/writing-an-alsa-driver/。

一个 DMA 驱动程序的示例为 soc/pxa/pxa2xx-pcm.c。

## SoC DAI 驱动程序

每个 SoC DAI 驱动程序必须提供以下功能：

1. 数字音频接口 (Digital audio interface，DAI) 描述
2. 数字音频接口配置
3. PCM 描述
4. SYSCLK 配置
5. 挂起和恢复（可选的）

请参考 codec.rst 来了解第 1 - 4 项的描述。

## SoC DSP 驱动程序

每个 SoC DSP 驱动程序通常提供以下功能：

1. DAPM 图
2. 混音器控制
3. DMA IO 到/从 DSP 缓冲区（如果适用的话）
4. DSP 前端 (FE) PCM 设备的定义

请参考 DPCM.txt 来了解第 4 项的描述。

[原文](linux-kernel/Documentation/sound/soc/platform.rst)

Done.
