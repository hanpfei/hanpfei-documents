---
title: 便携式设备的动态音频电源管理
date: 2023-04-29 21:59:29
categories: Linux 内核
tags:
- Linux 内核
---

## 描述

动态音频电源管理 (DAPM) 旨在允许便携式 Linux 设备在任何时候在音频子系统内使用最少的电源。它独立于其它内核 PM 系统，因此可以很容易地与其它 PM 系统共存。

DAPM 对所有用户空间应用程序也是完全透明的，因为所有电源切换都在 ASoC 核心中完成。用户空间应用程序不需要修改代码或重新编译。DAPM 根据设备内的任何音频流 (采集/播放) 活动和音频混音器设置进行电源切换决策。

DAPM 横跨整个机器。它涵盖了整个音频子系统中的电源控制，包括内部 codec 电源模块和机器级电源系统。

DAPM 内有 4 个电源域

Codec 偏置域
    VREF，VMID (核心 codec 和音频电源)

  - 通常在 codec 的 probe/remove 和 suspend/resume 时控制，尽管可以在流时设置，如果侧音不需要电源的话，等等。

平台/机器域
    物理连接输入和输出

  - 它是特定于平台/机器和用户行为的，它由机器驱动程序配置，并响应异步事件，比如当 HP 耳机插入时

路径域
     音频子系统信号路径

  - 当用户更改混音器和 mux 设置时自动地设置。比如 alsamixer，amixer。

流域
      DAC 和 ADC

  - 当启动或停止流播放/采集时分别启用或禁用。例如 aplay，arecord。

所有 DAPM 电源切换决策都是通过咨询整个机器的音频路由图自动做出的。该图是特定于每台机器的，由每个音频组件 (包括内部 codec 组件) 之间的互连组成。

所有影响电源的音频组件在下文称为部件。

## DAPM 部件

音频 DAPM 部件分为以下几种类型：

混音器：将多个模拟信号混合为单个模拟信号。

Mux：一种模拟开关，只输出多个输入中的一个。

PGA：可编程增益放大器或衰减部件。

ADC：模拟数字转换器。

DAC：数字模拟转换器。

开关 (Switch)：模拟开关。

输入 (Input)：Codec 输入针脚。

输出 (Output)：Codec 输出针脚。

耳机：耳机 (可选的插孔)。

麦克：麦克 (可选的插孔)。

线 (Line)：线输入/输出 (可选的插孔)。

扬声器：扬声器。

供应 (Supply)：其它部件使用的电源或时钟供应部件。

调整器 (Regulator)：给音频组件供电的外部变压器。

时钟 (Clock)：给音频组件提供时钟的外部时钟。

AIF IN：音频接口输入 (Audio Interface Input，带 TDM 槽掩码)。

AIF OUT：音频接口输入 (Audio Interface Output，带 TDM 槽掩码)。

Siggen：信号发生器。

DAI IN：数字音频接口输入。

DAI OUT：数字音频接口输出。

DAI 链接：两个 DAI 结构之间的 DAI 链接。

Pre：特殊的 PRE 部件 (在所有其它东西之前执行)

Post：特殊的 POST 部件 (在所有其它东西之后执行)

缓冲区：DSP 内的内部部件音频数据缓冲区。

调度器：DSP 内部调度程序，用于调度组件/流水线处理工作。

音效：执行音频处理效果的部件。

SRC：DSP 或 CODEC 内的采样率转换器。

ASRC：DSP 或 CODEC 内的异步采样率转换器。

编码器：将音频数据从一种格式 (通常是 PCM) 编码为另一种通常更压缩的格式的部件。

解码器：将音频数据从压缩格式解码为未压缩格式，如 PCM 的部件。

(部件在 *include/sound/soc-dapm.h* 中定义)

部件可以通过任何组件驱动程序类型添加到声卡中。*soc-dapm.h* 文件中定义了一些方便的宏，可用于快速构建 codec 部件和机器 DAPM 部件的列表。

大多数部件都有名称、寄存器、移位和反转。一些部件有额外的流名称和 kcontrols 参数。

### 流域部件

流域部件与流电源域有关，它仅由 ADC (模数转换器)，DAC (数模转换器)，AIF IN 和 AIF OUT 组成。

流部件有以下格式：
```
  SND_SOC_DAPM_DAC(name, stream name, reg, shift, invert),
  SND_SOC_DAPM_AIF_IN(name, stream, slot, reg, shift, invert)
```

注意：流名称必须与你的 codec snd_soc_codec_dai 中相应的流名称匹配。

比如，HiFi 播放和采集的流部件：
```
  SND_SOC_DAPM_DAC("HiFi DAC", "HiFi Playback", REG, 3, 1),
  SND_SOC_DAPM_ADC("HiFi ADC", "HiFi Capture", REG, 2, 1),
```

AIF 的流部件如下：
```
  SND_SOC_DAPM_AIF_IN("AIF1RX", "AIF1 Playback", 0, SND_SOC_NOPM, 0, 0),
  SND_SOC_DAPM_AIF_OUT("AIF1TX", "AIF1 Capture", 0, SND_SOC_NOPM, 0, 0),
```

### 路径域部件

路径域部件能够控制或影响音频子系统中的音频信号或音频路径。它们具有如下形式：
```
  SND_SOC_DAPM_PGA(name, reg, shift, invert, controls, num_controls)
```

可以使用 controls 和 num_controls 成员设置任何部件 kcontrols。

比如，混音器部件 (首先声明 kcontrols)：
```
  /* Output Mixer */
  static const snd_kcontrol_new_t wm8731_output_mixer_controls[] = {
  SOC_DAPM_SINGLE("Line Bypass Switch", WM8731_APANA, 3, 1, 0),
  SOC_DAPM_SINGLE("Mic Sidetone Switch", WM8731_APANA, 5, 1, 0),
  SOC_DAPM_SINGLE("HiFi Playback Switch", WM8731_APANA, 4, 1, 0),
  };

  SND_SOC_DAPM_MIXER("Output Mixer", WM8731_PWR, 4, 1, wm8731_output_mixer_controls,
	ARRAY_SIZE(wm8731_output_mixer_controls)),
```

如果你不希望混音器元素以混音器部件的名字为前缀，你可以使用 `SND_SOC_DAPM_MIXER_NAMED_CTL` 替代。参数与 `SND_SOC_DAPM_MIXER` 相同。

### 机器域部件

机器部件与 codec 部件不同，因为它们没有与之相关的 codec 寄存器位。可以独立供电的每个机器音频组件(非 codec 或DSP) 分配一个机器部件。比如：

 * 扬声器放大器
 * 麦克风偏置
 * 插孔连接器

机器部件可以有一个可选的回调。

比如，外部麦克风的插孔连接器部件在麦克风插入时启用麦克风偏置：
```
  static int spitz_mic_bias(struct snd_soc_dapm_widget* w, int event)
  {
	gpio_set_value(SPITZ_GPIO_MIC_BIAS, SND_SOC_DAPM_EVENT_ON(event));
	return 0;
  }

  SND_SOC_DAPM_MIC("Mic Jack", spitz_mic_bias),
```

### Codec (BIAS) 域

Codec 偏置电源域没有部件，它由 codec DAPM 事件处理程序处理。当 codec 电源状态被更改为任何流事件，或由内核 PM 事件，调用该处理程序。

### 虚拟部件

有时 codec 或机器音频图中出现的部件没有任何与之对应的软电源控制。在这种情况下，需要创建虚拟部件 - 没有控制位的部件，比如：
```
  SND_SOC_DAPM_MIXER("AC97 Mixer", SND_SOC_DAPM_NOPM, 0, 0, NULL, 0),
```

这可以用于在软件中合并信号路径。

在定义了所有部件之后，可以通过调用 `snd_soc_dapm_new_control()` 将它们单独添加到 DAPM 子系统中。

## Codec/DSP 部件互连

在 codec、平台和机器内，部件通过音频路径互相连接 (称为互连)。必须定义每个互连，以便创建部件间所有音频路径的图。

使用 codec 或 DSP 的图 (以及机器音频子系统的图) 最容易做到这一点，因为它需要通过音频信号路径将部件连接在一起。

比如，来自于 WM8731 输出混音器 (wm8731.c)

WM8731 输出混音器有 3 个输入 (源)

1. 旁路输入
2. DAC (HiFi 播放)
3. 麦克风侧通输入

本例中的每个输入都有一个与之关联的 kcontrol (在上面的示例中定义)，并通过其 kcontrol 名称连接到输出混音器。现在我们可以把目标部件 (音频信号) 和它的源部件连接起来了。
```
	/* output mixer */
	{"Output Mixer", "Line Bypass Switch", "Line Input"},
	{"Output Mixer", "HiFi Playback Switch", "DAC"},
	{"Output Mixer", "Mic Sidetone Switch", "Mic Bias"},
```

我们有：

 * 目标部件  <=== 路径名称 <=== 源部件，或
 * Sink，Path，Source，或
 * `Output Mixer` 通过 `HiFi Playback Switch` 被连接到 `DAC`。

当没有连接部件的路径名称时 (比如直接连接)，我们传递 NULL 作为路径名。

互连通过如下调用创建：
```
  snd_soc_dapm_connect_input(codec, sink, path, source);
```

最后，向核心注册了所有部件和互连之后，必须调用 `snd_soc_dapm_new_widgets(codec)`。这使得核心扫描 codec 和机器，以使内部的 DAPM  状态与机器的物理状态匹配。

### 机器部件互连

机器部件互连的创建方式与 codec 的方式相同，并直接将 codec 针脚连接到机器级部件。

例如，将扬声器输出 codec 针脚连接到内部扬声器。
```
	/* ext speaker connected to codec pins LOUT2, ROUT2  */
	{"Ext Spk", NULL , "ROUT2"},
	{"Ext Spk", NULL , "LOUT2"},
```

这允许 DAPM 分别接通和关闭连接 (和使用) 的引脚和 NC 引脚。

## 端点部件

端点是机器内的音频信号的起始或结束点 (部件)，并包含 codec。例如：

 * 耳机插孔
 * 内部扬声器
 * 内部麦克风
 * 麦克风插孔
 * Codec 针脚

端点被添加到 DAPM 图中，以便确定它们的使用情况，从而节省电力。例如 NC codec 的引脚将被关闭，未连接的插孔也可以被关闭。

## DAPM 部件事件

有些部件可以向 DAPM 核心注册它们感兴趣的 PM 事件。例如，具有放大器的扬声器注册一个部件，以使放大器仅能在扬声器使用时供电。
```
  /* turn speaker amplifier on/off depending on use */
  static int corgi_amp_event(struct snd_soc_dapm_widget *w, int event)
  {
	gpio_set_value(CORGI_GPIO_APM_ON, SND_SOC_DAPM_EVENT_ON(event));
	return 0;
  }

  /* corgi machine dapm widgets */
  static const struct snd_soc_dapm_widget wm8731_dapm_widgets =
	SND_SOC_DAPM_SPK("Ext Spk", corgi_amp_event);
```

请参考 *soc-dapm.h* 了解支持事件的所有其它部件。

### 事件类型

事件部件支持下列事件类型：
```
  /* dapm event types */
  #define SND_SOC_DAPM_PRE_PMU	0x1 	/* before widget power up */
  #define SND_SOC_DAPM_POST_PMU	0x2		/* after widget power up */
  #define SND_SOC_DAPM_PRE_PMD	0x4 	/* before widget power down */
  #define SND_SOC_DAPM_POST_PMD	0x8		/* after widget power down */
  #define SND_SOC_DAPM_PRE_REG	0x10	/* before audio path setup */
  #define SND_SOC_DAPM_POST_REG	0x20	/* after audio path setup */
```

[原文](linux-kernel/Documentation/sound/soc/dapm.rst)

Done.
