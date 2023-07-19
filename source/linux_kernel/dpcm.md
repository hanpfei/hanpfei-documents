---
title: 动态 PCM
date: 2023-04-29 18:43:29
categories: Linux 内核
tags:
- Linux 内核
---

## 描述

动态 PCM 允许一个 ALSA PCM 设备在 PCM 流运行时将其 PCM 音频数字地路由到各种各样的数字端点。比如 PCM0 可以将数字音频路由到 I2S DAI0，I2S DAI1 或 PDM DAI2。这对于暴露了多个 ALSA PCM 并且可以路由到多个 DAI 的 SoC DSP 驱动程序非常有用。

DPCM 运行时路由由 ALSA 混音器设置确定，与模拟信号在 ASoC codec 驱动程序中路由的方式相同。DPCM 使用一个 DAPM 图表示 DSP 内部的音频路径，并使用混音器设置决定每个 ALSA PCM 使用的路径。

DPCM 复用所有现有的组件 codec，平台和 DAI 驱动程序，而无需任何修改。

### 包含基于 DSP 的 SoC 的手机音频系统

考虑如下的手机音频子系统。这将在本文档中用于所有示例：
```
  | Front End PCMs    |  SoC DSP  | Back End DAIs | Audio devices |
  
                      *************
  PCM0 <------------> *           * <----DAI0-----> Codec Headset
                      *           *
  PCM1 <------------> *           * <----DAI1-----> Codec Speakers
                      *   DSP     *
  PCM2 <------------> *           * <----DAI2-----> MODEM
                      *           *
  PCM3 <------------> *           * <----DAI3-----> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

这个图展示了一个简单的职能手机音频子系统。它支持蓝牙，FM 数字音频，扬声器，耳机插孔，数字麦克风和蜂窝调制解调器。这个声卡暴露 4 个 DSP 前端 (FE) ALSA PCM 设备，并支持 6 个后端 (BE) DAI。每个 FE PCM 可以将音频数据数字地路由到任何 BE DAI。FE PCM 设备也可以将音频路由到多个 BE DAI。

### 示例 - DPCM 将播放从 DAI0 切换到 DAI1

耳机正在播放音频。过了一会儿，用户取下耳机，音频继续在扬声器上播放。

在 PCM0 上从耳机播放将看起来像这样：
```
                      *************
  PCM0 <============> *           * <====DAI0=====> Codec Headset
                      *           *
  PCM1 <------------> *           * <----DAI1-----> Codec Speakers
                      *   DSP     *
  PCM2 <------------> *           * <----DAI2-----> MODEM
                      *           *
  PCM3 <------------> *           * <----DAI3-----> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

用户将耳机从插孔中拔出，则现在必须使用扬声器：
```
                      *************
  PCM0 <============> *           * <----DAI0-----> Codec Headset
                      *           *
  PCM1 <------------> *           * <====DAI1=====> Codec Speakers
                      *   DSP     *
  PCM2 <------------> *           * <----DAI2-----> MODEM
                      *           *
  PCM3 <------------> *           * <----DAI3-----> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

音频驱动程序处理这种情况的方式如下：

1. 机器驱动程序收到插孔移除事件。

2. 机器驱动程序 **或** 音频 HAL 禁用耳机路径。

3. 由于路径现在被禁用，DPCM 为耳机在 DAI0 上运行 PCM `trigger(stop)`，`hw_free()`，`shutdown()` 操作。

4. 机器驱动程序或音频 HAL 启用扬声器路径。

5. 由于路径被启用，DPCM 为 DAI1 扬声器运行 PCM 操作 `startup()`、`hw_params()`、`prepare()` 和 `trigger(start)`。

在这个例子中，机器驱动程序或用户空间音频 HAL 改变路由，然后 DPCM 将负责管理 DAI PCM 操作，以打开或关闭链路。在此转换过程中，音频播放不停止。

## DPCM 机器驱动程序

启用了 DPCM 的 ASoC 机器驱动程序与一般的机器驱动程序类似，除了我们也必须做如下这些：

1. 定义 FE 和 BE DAI 链路。

2. 定义任何 FE/BE PCM 操作。

3. 定义小部件图连接。

### FE 和 BE DAI 链路

```
  | Front End PCMs    |  SoC DSP  | Back End DAIs | Audio devices |
  
                      *************
  PCM0 <------------> *           * <----DAI0-----> Codec Headset
                      *           *
  PCM1 <------------> *           * <----DAI1-----> Codec Speakers
                      *   DSP     *
  PCM2 <------------> *           * <----DAI2-----> MODEM
                      *           *
  PCM3 <------------> *           * <----DAI3-----> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

对于上面的例子，我们必须定义 4 个 FE DAI 链路和 6 个 BE DAI 链路。FE DAI 链路定义如下：
```
  static struct snd_soc_dai_link machine_dais[] = {
	{
		.name = "PCM0 System",
		.stream_name = "System Playback",
		.cpu_dai_name = "System Pin",
		.platform_name = "dsp-audio",
		.codec_name = "snd-soc-dummy",
		.codec_dai_name = "snd-soc-dummy-dai",
		.dynamic = 1,
		.trigger = {SND_SOC_DPCM_TRIGGER_POST, SND_SOC_DPCM_TRIGGER_POST},
		.dpcm_playback = 1,
	},
	.....< other FE and BE DAI links here >
  };
```

这个 FE DAI 链接与常规 DAI 链接非常相似，只是我们还通过 `dynamic = 1` 把 DAI 链接设置为一个 DPCM FE。支持的 FE 流方向也应该通过 `dpcm_playback` 和 `dpcm_capture` 标记来设置。还有一个选项可以为每个 FE 指定触发调用的顺序。这允许 ASoC 核心在其它组件之前或之后触发 DSP (因为一些 DSP 对 DAI/DSP 的启动和停止顺序有很强的要求)。

上面的 FE DAI 设置 codec 并把 DAI 编码为 dummy 设备，由于 BE 是动态的，并将根据运行时配置而更改。

BE DAI 配置如下：
```
  static struct snd_soc_dai_link machine_dais[] = {
	.....< FE DAI links here >
	{
		.name = "Codec Headset",
		.cpu_dai_name = "ssp-dai.0",
		.platform_name = "snd-soc-dummy",
		.no_pcm = 1,
		.codec_name = "rt5640.0-001c",
		.codec_dai_name = "rt5640-aif1",
		.ignore_suspend = 1,
		.ignore_pmdown_time = 1,
		.be_hw_params_fixup = hswult_ssp0_fixup,
		.ops = &haswell_ops,
		.dpcm_playback = 1,
		.dpcm_capture = 1,
	},
	.....< other BE DAI links here >
  };
```

这个 BE DAI 链接把 DAI0 连接到 codec (在这个例子中是 RT5460 AIF1)。它设置了 `no_pcm` 标志来标记它有一个 BE，并使用上面的 `dpcm_playback` 和 `dpcm_capture` 为支持的流方向设置标记。

BE 还设置了忽略挂起和 PM 停机事件的标志。这允许 BE 在无主机模式下工作，其中主机 CPU 不像 BT 电话呼叫那样传输数据：
```
                      *************
  PCM0 <------------> *           * <----DAI0-----> Codec Headset
                      *           *
  PCM1 <------------> *           * <----DAI1-----> Codec Speakers
                      *   DSP     *
  PCM2 <------------> *           * <====DAI2=====> MODEM
                      *           *
  PCM3 <------------> *           * <====DAI3=====> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

这允许主机 CPU 在 DSP、MODEM DAI 和 BT DAI 仍在运行时休眠。

如果 codec 是外部管理的设备，BE DAI 链接也可以把 codec 设置为 dummy 设备。

同样，如果 CPU DAI 由 DSP 固件管理，则 BE DAI 也可以设置一个 dummy CPU DAI。

### FE/BE PCM 操作

上面的 BE 也导出一些 PCM 操作和一个 `fixup` 回调。机器驱动程序使用 `fixup` 回调来 (重新) 配置基于 FE 硬件参数的 DAI。即 DSP 可以执行从 FE 到 BE 的 SRC 或 ASRC。

例如，DSP 为 DAI0 将所有 FE 硬件参数转换为以固定的 48k 速率, 16 位，立体声运行。这意味着在 DAI0 的机器驱动程序中所有 FE hw_params 必须是固定的，以便 DAI 在所需配置下运行，而无论 FE 配置如何。
```
  static int dai0_fixup(struct snd_soc_pcm_runtime *rtd,
			struct snd_pcm_hw_params *params)
  {
	struct snd_interval *rate = hw_param_interval(params,
			SNDRV_PCM_HW_PARAM_RATE);
	struct snd_interval *channels = hw_param_interval(params,
						SNDRV_PCM_HW_PARAM_CHANNELS);

	/* The DSP will convert the FE rate to 48k, stereo */
	rate->min = rate->max = 48000;
	channels->min = channels->max = 2;

	/* set DAI0 to 16 bit */
	params_set_format(params, SNDRV_PCM_FORMAT_S16_LE);
	return 0;
  }
```

其它 PCM 操作与常规 DAI 链接相同。必要时使用。

### 小部件图连接

BE DAI 链接通常在初始化时由 ASoC DPAM 核心连接到图。然而，如果 BE codec 或 BE DAI 是 dummy 的，则必须在驱动程序中显式设置：
```
  /* BE for codec Headset -  DAI0 is dummy and managed by DSP FW */
  {"DAI0 CODEC IN", NULL, "AIF1 Capture"},
  {"AIF1 Playback", NULL, "DAI0 CODEC OUT"},
```

## 编写 DPCM DSP 驱动程序

DPCM DSP 驱动程序看起来与标准的平台类 ASoC 驱动程序结合 codec 类驱动程序的元素非常相似。DSP 平台驱动程序必须实现：

1. 前端 PCM DAI - 即 struct snd_soc_dai_driver。

2. DAPM 图展示从 FE DAI 到 BE 的 DSP 音频路由。

3. 来自 DSP 图的 DAPM 部件。

4. 增益、路由等混音器。

5. DMA 配置。

6. BE AIF 小部件。

第 6 项对于 DSP 外部的音频路由很重要。需要为每个 BE 和每个流方向定义 AIF。比如，对于上面的 BE DAI0 我们将有：
```
  SND_SOC_DAPM_AIF_IN("DAI0 RX", NULL, 0, SND_SOC_NOPM, 0, 0),
  SND_SOC_DAPM_AIF_OUT("DAI0 TX", NULL, 0, SND_SOC_NOPM, 0, 0),
```

BE AIF 用于将 DSP 图连接到其它组件驱动程序的图 (比如，codec 图)。

## 无主机 PCM 流

无主机 PCM 流是不通过主机 CPU 路由的流。这方面的一个例子是从手机到调制解调器的电话。
```
                      *************
  PCM0 <------------> *           * <----DAI0-----> Codec Headset
                      *           *
  PCM1 <------------> *           * <====DAI1=====> Codec Speakers/Mic
                      *   DSP     *
  PCM2 <------------> *           * <====DAI2=====> MODEM
                      *           *
  PCM3 <------------> *           * <----DAI3-----> BT
                      *           *
                      *           * <----DAI4-----> DMIC
                      *           *
                      *           * <----DAI5-----> FM
                      *************
```

在这种情况下，PCM 数据通过 DSP 路由。在这种用例中，主机 CPU 仅用于控制，且可以在流运行期间休眠。

主机可以通过以下方式控制无主机链路：

1. 把链接配置为 CODEC <-> CODEC 样式的链接。在这种情况下，链接由 DAPM 图的状态启用或禁用。这通常意味着有一个混音器控制，可用于连接或断开两个 DAI 之间的路径。

2. 无主机 FE。这种 FE 与 DAPM 图上的 BE DAI 链接有一个虚拟连接。然后像常规 PCM 操作那样由 FE 进行控制。这种方法提供了对 DAI 链接的更多控制，但需要更多的用户空间代码来控制链接。建议使用 CODEC<->CODEC，除非你的硬件需要更细粒度的 PCM 操作顺序。

### CODEC <-> CODEC 链接

当 DAPM 检测到 DAPM 图中的有效路径时，启用这种 DAI 链接。机器驱动程序给这种 DAI 链接设置一些附加参数，即：
```
  static const struct snd_soc_pcm_stream dai_params = {
	.formats = SNDRV_PCM_FMTBIT_S32_LE,
	.rate_min = 8000,
	.rate_max = 8000,
	.channels_min = 2,
	.channels_max = 2,
  };

  static struct snd_soc_dai_link dais[] = {
	< ... more DAI links above ... >
	{
		.name = "MODEM",
		.stream_name = "MODEM",
		.cpu_dai_name = "dai2",
		.codec_dai_name = "modem-aif1",
		.codec_name = "modem",
		.dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF
				| SND_SOC_DAIFMT_CBM_CFM,
		.params = &dai_params,
	}
	< ... more DAI links here ... >
```

这些参数用于在 DAPM 检测到有效路径并调用 PCM 操作启动链路时配置 DAI `hw_params()`。当路径不再有效时，DAPM 还将调用适当的 PCM 操作来禁用 DAI。

### 无主机 FE

DAI 链接由不读写任何 PCM 数据的 FE 启用。这意味着创建一个与两个 DAI 链接的虚拟路径相连接的新 FE。FE PCM 启动时 DAI 链接启动，FE PCM 停止时 DAI 链接停止。注意，在这种配置中，FE PCM 不能读取或写入数据。

[原文](linux-kernel/Documentation/sound/soc/dpcm.rst)

Done.
