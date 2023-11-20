---
title: ASoC Codec 类驱动程序
date: 2023-04-29 18:13:29
categories: Linux 内核
tags:
- Linux 内核
---

ASoC codec 类驱动程序是通用的与硬件无关的代码，它配置 codec、FM、MODEM、BT 或外部 DSP 来提供音频采集和播放。它不应该包含特定于目标平台或机器的代码。所有特定于平台或机器的代码应该分别加进平台和机器驱动程序中。

每个 codec 类驱动程序**必须**提供如下功能：

1. Codec DAI 和 PCM 配置
2. Codec 控制 IO - 使用 RegMap API
3. 混音器和音频控制
4. Codec 音频操作
5. DAPM 描述
6. DAPM 事件处理器

Codec 驱动程序也可以选择性地提供：

7. DAC 数字静音控制

最好结合 *sound/soc/codecs/* 中现有的 codec 驱动程序代码来使用本指南。

## ASoC Codec 驱动程序分解

### Codec DAI 和 PCM 配置

每个 codec 驱动程序必须有一个 `struct snd_soc_dai_driver` 对象来定义它的 DAI 和 PCM 能力及操作。Codec 驱动程序导出该结构体，以便你的机器驱动程序可以将其注册到内核中。

如：
```
  static struct snd_soc_dai_ops wm8731_dai_ops = {
	.prepare	= wm8731_pcm_prepare,
	.hw_params	= wm8731_hw_params,
	.shutdown	= wm8731_shutdown,
	.digital_mute	= wm8731_mute,
	.set_sysclk	= wm8731_set_dai_sysclk,
	.set_fmt	= wm8731_set_dai_fmt,
  };
  
  struct snd_soc_dai_driver wm8731_dai = {
	.name = "wm8731-hifi",
	.playback = {
		.stream_name = "Playback",
		.channels_min = 1,
		.channels_max = 2,
		.rates = WM8731_RATES,
		.formats = WM8731_FORMATS,},
	.capture = {
		.stream_name = "Capture",
		.channels_min = 1,
		.channels_max = 2,
		.rates = WM8731_RATES,
		.formats = WM8731_FORMATS,},
	.ops = &wm8731_dai_ops,
	.symmetric_rates = 1,
  };
```

### Codec 控制 IO

Codec 通常可以通过 I2C 或 SPI 风格的接口来控制 (AC97 在 DAI 中将控制与数据结合在一起)。Codec 驱动程序应该为所有 codec IO 使用 Regmap API。请参阅 *include/linux/regmap.h* 和现有的 codec 驱动程序的例子，来了解 regmap 的用法。

### 混音器和音频控制

所有的 codec 混音器和音频控制都可以使用 *soc.h* 中定义的方便的宏来定义。

```
    #define SOC_SINGLE(xname, reg, shift, mask, invert)
```
定义单个控制的方法如下所示：
  - xname = 控制名称，如 "Playback Volume"
  - reg = codec 寄存器
  - shift = 寄存器中的控制位偏移
  - mask = 控制位大小，比如掩码 7 = 3 位
  - invert = 控制是反向的

其它的宏包括：
```
    #define SOC_DOUBLE(xname, reg, shift_left, shift_right, mask, invert)
```

一个立体声控制：
```
    #define SOC_DOUBLE_R(xname, reg_left, reg_right, shift, mask, invert)
```

跨越 2 个寄存器的立体声控制：
```
    #define SOC_ENUM_SINGLE(xreg, xshift, xmask, xtexts)
```

定义单个枚举控制的方法如下：
```
   xreg = 寄存器
   xshift = 控制位在寄存器中的偏移
   xmask = 控制位大小
   xtexts = 指向描述每个设置的字符串数组

   #define SOC_ENUM_DOUBLE(xreg, xshift_l, xshift_r, xmask, xtexts)
```

定义一个立体声枚举控制。

### Codec 音频操作

Codec 驱动程序也支持如下的 ALSA PCM 操作：
```
  /* SoC audio ops */
  struct snd_soc_ops {
	int (*startup)(struct snd_pcm_substream *);
	void (*shutdown)(struct snd_pcm_substream *);
	int (*hw_params)(struct snd_pcm_substream *, struct snd_pcm_hw_params *);
	int (*hw_free)(struct snd_pcm_substream *);
	int (*prepare)(struct snd_pcm_substream *);
  };
```

请参考 ALSA 驱动程序 PCM 文档来了解详情：http://www.alsa-project.org/~iwai/writing-an-alsa-driver/

### DAPM 描述

动态音频电源管理描述描述了 codec 电源组件和它们的关系，并注册给 ASoC 核心。请阅读 dapm.rst 来了解构建描述的细节。

也请参考其它 codec 驱动程序的例子。

### DAPM 事件处理程序

这个函数是一个回调，它处理 codec 域 PM 调用和系统域 PM 调用 (比如，挂起和恢复)。它用于使 codec 在不使用时进入休眠状态。

电源状态：
```
	SNDRV_CTL_POWER_D0: /* full On */
	/* vref/mid, clk and osc on, active */

	SNDRV_CTL_POWER_D1: /* partial On */
	SNDRV_CTL_POWER_D2: /* partial On */

	SNDRV_CTL_POWER_D3hot: /* Off, with power */
	/* everything off except vref/vmid, inactive */

	SNDRV_CTL_POWER_D3cold: /* Everything Off, without power */
```

### Codec DAC 数字静音控制

大多数 codec 在 DAC 之前都有一个数字静音，它可以用来最小化任何系统噪声。静音阻止任何数字数据进入 DAC。

可以创建一个回调，在静音被应用或释放时，核心为每个 codec DAI 调用它。如：
```
  static int wm8974_mute(struct snd_soc_dai *dai, int mute)
  {
	struct snd_soc_component *component = dai->component;
	u16 mute_reg = snd_soc_component_read32(component, WM8974_DAC) & 0xffbf;

	if (mute)
		snd_soc_component_write(component, WM8974_DAC, mute_reg | 0x40);
	else
		snd_soc_component_write(component, WM8974_DAC, mute_reg);
	return 0;
  }
```

[原文](linux-kernel/Documentation/sound/soc/codec.rst)

Done.
