---
title: ASoC 机器驱动程序
date: 2023-04-29 16:13:29
categories: Linux 内核
tags:
- Linux 内核
---

ASoC 机器 (或板子) 驱动程序是把所有的组件驱动程序 (比如，codecs，平台和 DAI) 粘在一起的代码。它也描述每个组件之间的关系，包括音频路径、GPIO、中断、时钟、插孔和电压调节器。 

机器驱动程序可以包含特定于 codec 和平台的代码。它将音频子系统注册到内核中，作为平台设备，并由以下结构体表示：
```
  /* SoC machine */
  struct snd_soc_card {
	char *name;

	...

	int (*probe)(struct platform_device *pdev);
	int (*remove)(struct platform_device *pdev);

	/* the pre and post PM functions are used to do any PM work before and
	 * after the codec and DAIs do any PM work. */
	int (*suspend_pre)(struct platform_device *pdev, pm_message_t state);
	int (*suspend_post)(struct platform_device *pdev, pm_message_t state);
	int (*resume_pre)(struct platform_device *pdev);
	int (*resume_post)(struct platform_device *pdev);

	...

	/* CPU <--> Codec DAI links  */
	struct snd_soc_dai_link *dai_link;
	int num_links;

	...
  };
```

## probe()/remove()

probe/remove 是可选的。在这里执行任何特定于机器的探测。

## suspend()/resume()

机器驱动程序具有挂起和恢复的前后版本，以处理在 codec、DAI 和 DMA 被挂起和恢复前或后必须完成的任何机器音频任务。可选。

## 机器 DAI 配置

机器 DAI 配置把所有 codec 和 CPU DAI 粘在一起。它还可以用于设置 DAI 系统时钟和任何与机器相关的 DAI 初始化，例如，机器音频映射可以连接到 codec 音频映射，未连接的 codec 引脚可以这样设置。

`struct snd_soc_dai_link` 用于设置你的机器上的每个 DAI。如：
```
  /* corgi digital audio interface glue - connects codec <--> CPU */
  static struct snd_soc_dai_link corgi_dai = {
	.name = "WM8731",
	.stream_name = "WM8731",
	.cpu_dai_name = "pxa-is2-dai",
	.codec_dai_name = "wm8731-hifi",
	.platform_name = "pxa-pcm-audio",
	.codec_name = "wm8713-codec.0-001a",
	.init = corgi_wm8731_init,
	.ops = &corgi_ops,
  };
```

struct snd_soc_card 然后用它的 DAI 设置机器。如：
```
  /* corgi audio machine driver */
  static struct snd_soc_card snd_soc_corgi = {
	.name = "Corgi",
	.dai_link = &corgi_dai,
	.num_links = 1,
  };
```

## 机器电源图

机器驱动程序可以选择性地扩展 codec 电源图，并成为音频子系统的音频电源图。这允许扬声器/HP 放大器等自动开启/关闭。Codec 针脚可以在机器初始化函数中被连接到机器插孔槽中。

## 机器控制

特定于机器的音频混音器控制可以加进 DAI 初始化函数。

[原文](linux-kernel/Documentation/sound/soc/machine.rst)

Done.
