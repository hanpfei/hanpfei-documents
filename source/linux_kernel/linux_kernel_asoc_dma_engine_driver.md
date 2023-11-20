---
title: Linux 内核 ASoC DMA 引擎驱动程序
date: 2023-07-29 9:23:29
categories: Linux 内核
tags:
- Linux 内核
---

Linux 内核 ASoC 框架，在概念上将嵌入式音频系统拆分为多个可复用的组件驱动程序，包括 Codec 类驱动程序、平台类驱动程序和机器类驱动程序。在实现上，机器类驱动程序用 `struct snd_soc_card` 和 `struct snd_soc_dai_link` 结构描述，属于平台类驱动程序的 DMA 引擎驱动程序由 `struct snd_soc_component_driver` 结构描述，codec 类驱动程序和 I2S 等驱动程序，由 `struct snd_soc_component_driver`、`struct snd_soc_dai_driver` 和 `struct snd_soc_dai_ops` 等结构描述。除平台类驱动程序外的各种驱动程序都通过 component 抽象组织在一起，即这些驱动程序都作为 `struct snd_soc_component_driver` 注册给 Linux 内核 ASoC 框架，Linux 内核 ASoC 框架为它们各自创建 `struct snd_soc_component` 结构对象，并保存在 `sound/soc/soc-core.c` 文件中定义的全局链表 `component_list` 中。

一个 DMA 驱动程序的示例为 `soc/pxa/pxa2xx-pcm.c`：
```
static const struct snd_soc_component_driver pxa2xx_soc_platform = {
	.pcm_construct	= pxa2xx_soc_pcm_new,
	.pcm_destruct	= pxa2xx_soc_pcm_free,
	.open		= pxa2xx_soc_pcm_open,
	.close		= pxa2xx_soc_pcm_close,
	.hw_params	= pxa2xx_soc_pcm_hw_params,
	.hw_free	= pxa2xx_soc_pcm_hw_free,
	.prepare	= pxa2xx_soc_pcm_prepare,
	.trigger	= pxa2xx_soc_pcm_trigger,
	.pointer	= pxa2xx_soc_pcm_pointer,
	.mmap		= pxa2xx_soc_pcm_mmap,
};

static int pxa2xx_soc_platform_probe(struct platform_device *pdev)
{
	return devm_snd_soc_register_component(&pdev->dev, &pxa2xx_soc_platform,
					       NULL, 0);
}

static struct platform_driver pxa_pcm_driver = {
	.driver = {
		.name = "pxa-pcm-audio",
	},

	.probe = pxa2xx_soc_platform_probe,
};

module_platform_driver(pxa_pcm_driver);
```

DMA 引擎驱动程序和 I2S 或 Codec 驱动程序一样，通过 `devm_snd_soc_register_component()` 函数以 `struct snd_soc_component_driver` 的形式注册给 Linux 内核 ASoC 框架，但比较特别的地方在于，它的 dai driver 参数为空。

机器类驱动程序定义的 `struct snd_soc_dai_link` 结构对象通过 `struct snd_soc_dai_link_component` 描述它引用的其它类型的驱动程序，如机器类驱动程序 `sound/soc/pxa/e800_wm9712.c` 有如下的代码片段：
```
SND_SOC_DAILINK_DEFS(ac97,
	DAILINK_COMP_ARRAY(COMP_CPU("pxa2xx-ac97")),
	DAILINK_COMP_ARRAY(COMP_CODEC("wm9712-codec", "wm9712-hifi")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("pxa-pcm-audio")));

SND_SOC_DAILINK_DEFS(ac97_aux,
	DAILINK_COMP_ARRAY(COMP_CPU("pxa2xx-ac97-aux")),
	DAILINK_COMP_ARRAY(COMP_CODEC("wm9712-codec", "wm9712-aux")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("pxa-pcm-audio")));

static struct snd_soc_dai_link e800_dai[] = {
	{
		.name = "AC97",
		.stream_name = "AC97 HiFi",
		SND_SOC_DAILINK_REG(ac97),
	},
	{
		.name = "AC97 Aux",
		.stream_name = "AC97 Aux",
		SND_SOC_DAILINK_REG(ac97_aux),
	},
};

static struct snd_soc_card e800 = {
	.name = "Toshiba e800",
	.owner = THIS_MODULE,
	.dai_link = e800_dai,
	.num_links = ARRAY_SIZE(e800_dai),

	.dapm_widgets = e800_dapm_widgets,
	.num_dapm_widgets = ARRAY_SIZE(e800_dapm_widgets),
	.dapm_routes = audio_map,
	.num_dapm_routes = ARRAY_SIZE(audio_map),
};
```

`SND_SOC_DAILINK_DEFS()` 宏用于为 `struct snd_soc_dai_link` 方便地定义引用的 `cpus`、`codecs` 和 `platforms` 等 `struct snd_soc_dai_link_component` 数组，其中用于定义 `platforms` 的 `DAILINK_COMP_ARRAY(COMP_PLATFORM("pxa-pcm-audio"))` 引用了上面我们看到的 DMA 引擎驱动。

Linux 内核 ASoC 框架提供了一个通用的 DMA 引擎驱动程序，位于文件 `sound/soc/soc-generic-dmaengine-pcm` 中。这个驱动程序本身不会主动向 Linux 内核 ASoC 框架注册自己，需要使用 DMA 引擎在设备和内存之间传数据的驱动程序要在 `probe` 时注册它，如 `sound/soc/rockchip/rockchip_pcm.c`：
```
static const struct snd_pcm_hardware snd_rockchip_hardware = {
	.info			= SNDRV_PCM_INFO_MMAP |
				  SNDRV_PCM_INFO_MMAP_VALID |
				  SNDRV_PCM_INFO_PAUSE |
				  SNDRV_PCM_INFO_RESUME |
				  SNDRV_PCM_INFO_INTERLEAVED,
	.period_bytes_min	= 32,
	.period_bytes_max	= 8192,
	.periods_min		= 1,
	.periods_max		= 52,
	.buffer_bytes_max	= 64 * 1024,
	.fifo_size		= 32,
};

static const struct snd_dmaengine_pcm_config rk_dmaengine_pcm_config = {
	.pcm_hardware = &snd_rockchip_hardware,
	.prepare_slave_config = snd_dmaengine_pcm_prepare_slave_config,
	.prealloc_buffer_size = 32 * 1024,
};

int rockchip_pcm_platform_register(struct device *dev)
{
	return devm_snd_dmaengine_pcm_register(dev, &rk_dmaengine_pcm_config,
		SND_DMAENGINE_PCM_FLAG_COMPAT);
}
EXPORT_SYMBOL_GPL(rockchip_pcm_platform_register);
```

这里的 `rockchip_pcm_platform_register()` 函数在 I2S 驱动程序的 `probe` 操作 (位于 `sound/soc/rockchip/rockchip_i2s.c`) 中调用：
```
static int rockchip_i2s_probe(struct platform_device *pdev)
{
 . . . . . .

	ret = devm_snd_soc_register_component(&pdev->dev,
					      &rockchip_i2s_component,
					      soc_dai, 1);

	if (ret) {
		dev_err(&pdev->dev, "Could not register DAI\n");
		goto err_suspend;
	}

	ret = rockchip_pcm_platform_register(&pdev->dev);
	if (ret) {
		dev_err(&pdev->dev, "Could not register PCM\n");
		goto err_suspend;
	}

	return 0;
 . . . . . .
	return ret;
}
```

`rockchip_pcm_platform_register()` 函数调用 `devm_snd_dmaengine_pcm_register()` 函数注册通用 DMA 引擎驱动程序。`devm_snd_dmaengine_pcm_register()` 函数定义 (位于 `sound/soc/soc-devres.c`) 如下：
```
static void devm_dmaengine_pcm_release(struct device *dev, void *res)
{
	snd_dmaengine_pcm_unregister(*(struct device **)res);
}

/**
 * devm_snd_dmaengine_pcm_register - resource managed dmaengine PCM registration
 * @dev: The parent device for the PCM device
 * @config: Platform specific PCM configuration
 * @flags: Platform specific quirks
 *
 * Register a dmaengine based PCM device with automatic unregistration when the
 * device is unregistered.
 */
int devm_snd_dmaengine_pcm_register(struct device *dev,
	const struct snd_dmaengine_pcm_config *config, unsigned int flags)
{
	struct device **ptr;
	int ret;

	ptr = devres_alloc(devm_dmaengine_pcm_release, sizeof(*ptr), GFP_KERNEL);
	if (!ptr)
		return -ENOMEM;

	ret = snd_dmaengine_pcm_register(dev, config, flags);
	if (ret == 0) {
		*ptr = dev;
		devres_add(dev, ptr);
	} else {
		devres_free(ptr);
	}

	return ret;
}
EXPORT_SYMBOL_GPL(devm_snd_dmaengine_pcm_register);

#endif
```

`devm_snd_dmaengine_pcm_register()` 函数和之前看到的`devm_snd_soc_register_card()` 与 `devm_snd_soc_register_component()` 函数一样，只是它封装的是 `snd_dmaengine_pcm_register()` 函数。`snd_dmaengine_pcm_register()` 函数定义 (位于 `sound/soc/soc-generic-dmaengine-pcm.c`) 如下：
```
static const struct snd_soc_component_driver dmaengine_pcm_component = {
	.name		= SND_DMAENGINE_PCM_DRV_NAME,
	.probe_order	= SND_SOC_COMP_ORDER_LATE,
	.open		= dmaengine_pcm_open,
	.close		= dmaengine_pcm_close,
	.hw_params	= dmaengine_pcm_hw_params,
	.trigger	= dmaengine_pcm_trigger,
	.pointer	= dmaengine_pcm_pointer,
	.pcm_construct	= dmaengine_pcm_new,
};

static const struct snd_soc_component_driver dmaengine_pcm_component_process = {
	.name		= SND_DMAENGINE_PCM_DRV_NAME,
	.probe_order	= SND_SOC_COMP_ORDER_LATE,
	.open		= dmaengine_pcm_open,
	.close		= dmaengine_pcm_close,
	.hw_params	= dmaengine_pcm_hw_params,
	.trigger	= dmaengine_pcm_trigger,
	.pointer	= dmaengine_pcm_pointer,
	.copy_user	= dmaengine_copy_user,
	.pcm_construct	= dmaengine_pcm_new,
};

static const char * const dmaengine_pcm_dma_channel_names[] = {
	[SNDRV_PCM_STREAM_PLAYBACK] = "tx",
	[SNDRV_PCM_STREAM_CAPTURE] = "rx",
};

static int dmaengine_pcm_request_chan_of(struct dmaengine_pcm *pcm,
	struct device *dev, const struct snd_dmaengine_pcm_config *config)
{
	unsigned int i;
	const char *name;
	struct dma_chan *chan;

	if ((pcm->flags & SND_DMAENGINE_PCM_FLAG_NO_DT) || (!dev->of_node &&
	    !(config && config->dma_dev && config->dma_dev->of_node)))
		return 0;

	if (config && config->dma_dev) {
		/*
		 * If this warning is seen, it probably means that your Linux
		 * device structure does not match your HW device structure.
		 * It would be best to refactor the Linux device structure to
		 * correctly match the HW structure.
		 */
		dev_warn(dev, "DMA channels sourced from device %s",
			 dev_name(config->dma_dev));
		dev = config->dma_dev;
	}

	for_each_pcm_streams(i) {
		if (pcm->flags & SND_DMAENGINE_PCM_FLAG_HALF_DUPLEX)
			name = "rx-tx";
		else
			name = dmaengine_pcm_dma_channel_names[i];
		if (config && config->chan_names[i])
			name = config->chan_names[i];
		chan = dma_request_chan(dev, name);
		if (IS_ERR(chan)) {
			/*
			 * Only report probe deferral errors, channels
			 * might not be present for devices that
			 * support only TX or only RX.
			 */
			if (PTR_ERR(chan) == -EPROBE_DEFER)
				return -EPROBE_DEFER;
			pcm->chan[i] = NULL;
		} else {
			pcm->chan[i] = chan;
		}
		if (pcm->flags & SND_DMAENGINE_PCM_FLAG_HALF_DUPLEX)
			break;
	}

	if (pcm->flags & SND_DMAENGINE_PCM_FLAG_HALF_DUPLEX)
		pcm->chan[1] = pcm->chan[0];

	return 0;
}

static void dmaengine_pcm_release_chan(struct dmaengine_pcm *pcm)
{
	unsigned int i;

	for_each_pcm_streams(i) {
		if (!pcm->chan[i])
			continue;
		dma_release_channel(pcm->chan[i]);
		if (pcm->flags & SND_DMAENGINE_PCM_FLAG_HALF_DUPLEX)
			break;
	}
}

/**
 * snd_dmaengine_pcm_register - Register a dmaengine based PCM device
 * @dev: The parent device for the PCM device
 * @config: Platform specific PCM configuration
 * @flags: Platform specific quirks
 */
int snd_dmaengine_pcm_register(struct device *dev,
	const struct snd_dmaengine_pcm_config *config, unsigned int flags)
{
	const struct snd_soc_component_driver *driver;
	struct dmaengine_pcm *pcm;
	int ret;

	pcm = kzalloc(sizeof(*pcm), GFP_KERNEL);
	if (!pcm)
		return -ENOMEM;

#ifdef CONFIG_DEBUG_FS
	pcm->component.debugfs_prefix = "dma";
#endif
	pcm->config = config;
	pcm->flags = flags;

	ret = dmaengine_pcm_request_chan_of(pcm, dev, config);
	if (ret)
		goto err_free_dma;

	if (config && config->process)
		driver = &dmaengine_pcm_component_process;
	else
		driver = &dmaengine_pcm_component;

	ret = snd_soc_component_initialize(&pcm->component, driver, dev);
	if (ret)
		goto err_free_dma;

	ret = snd_soc_add_component(&pcm->component, NULL, 0);
	if (ret)
		goto err_free_dma;

	return 0;

err_free_dma:
	dmaengine_pcm_release_chan(pcm);
	kfree(pcm);
	return ret;
}
EXPORT_SYMBOL_GPL(snd_dmaengine_pcm_register);

/**
 * snd_dmaengine_pcm_unregister - Removes a dmaengine based PCM device
 * @dev: Parent device the PCM was register with
 *
 * Removes a dmaengine based PCM device previously registered with
 * snd_dmaengine_pcm_register.
 */
void snd_dmaengine_pcm_unregister(struct device *dev)
{
	struct snd_soc_component *component;
	struct dmaengine_pcm *pcm;

	component = snd_soc_lookup_component(dev, SND_DMAENGINE_PCM_DRV_NAME);
	if (!component)
		return;

	pcm = soc_component_to_pcm(component);

	snd_soc_unregister_component_by_driver(dev, component->driver);
	dmaengine_pcm_release_chan(pcm);
	kfree(pcm);
}
EXPORT_SYMBOL_GPL(snd_dmaengine_pcm_unregister);

MODULE_LICENSE("GPL");
```

`snd_dmaengine_pcm_register()` 函数的执行过程如下：

1. 动态分配一个 `struct dmaengine_pcm` 结构对象，并初始化其 `config` 和 `flags` 等字段，`struct dmaengine_pcm` 结构定义 (位于 `include/sound/dmaengine_pcm.h`) 如下：
```
struct dmaengine_pcm {
	struct dma_chan *chan[SNDRV_PCM_STREAM_LAST + 1];
	const struct snd_dmaengine_pcm_config *config;
	struct snd_soc_component component;
	unsigned int flags;
};
```
这个结构有一个 `struct snd_soc_component` 结构成员；

2. 为数据的发送和接收申请 DMA 通道，这个需要在设备树的设备节点定义中，指定发送和接收引用的 DMA 通道，像下面这样：
```
	i2s0_8ch: i2s@fe470000 {
 . . . . . .
		dmas = <&dmac0 0>, <&dmac0 1>;
		dma-names = "tx", "rx";
 . . . . . .
	};
```

3. 根据传入的 `config` 参数，选择 `struct snd_soc_component_driver`，`dmaengine_pcm_component_process` 和 `dmaengine_pcm_component` 仅有的区别是，前者多定义了一个 `copy_user` 操作；

4. 初始化并添加 `struct snd_soc_component` 结构对象，有一个函数 (位于 `include/sound/dmaengine_pcm.h`) 可以通过 `struct snd_soc_component` 结构对象获得 `struct dmaengine_pcm` 结构对象：
```
static inline struct dmaengine_pcm *soc_component_to_pcm(struct snd_soc_component *p)
{
	return container_of(p, struct dmaengine_pcm, component);
}
```

通用 DMA 引擎驱动程序支持的操作由 `struct snd_soc_component_driver` 定义。在全局  component 链表中，相应的 `struct snd_soc_component` 由注册它的 dev 标识。

使用了通用 DMA 引擎驱动程序的 ASoC 机器驱动程序，示例 (位于 `sound/soc/rockchip/rockchip_rt5645.c`) 如下：
```
SND_SOC_DAILINK_DEFS(pcm,
	DAILINK_COMP_ARRAY(COMP_EMPTY()),
	DAILINK_COMP_ARRAY(COMP_CODEC(NULL, "rt5645-aif1")),
	DAILINK_COMP_ARRAY(COMP_EMPTY()));

static struct snd_soc_dai_link rk_dailink = {
	.name = "rt5645",
	.stream_name = "rt5645 PCM",
	.init = rk_init,
	.ops = &rk_aif1_ops,
	/* set rt5645 as slave */
	.dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF |
		SND_SOC_DAIFMT_CBS_CFS,
	SND_SOC_DAILINK_REG(pcm),
};

static struct snd_soc_card snd_soc_card_rk = {
	.name = "I2S-RT5650",
	.owner = THIS_MODULE,
	.dai_link = &rk_dailink,
	.num_links = 1,
	.dapm_widgets = rk_dapm_widgets,
	.num_dapm_widgets = ARRAY_SIZE(rk_dapm_widgets),
	.dapm_routes = rk_audio_map,
	.num_dapm_routes = ARRAY_SIZE(rk_audio_map),
	.controls = rk_mc_controls,
	.num_controls = ARRAY_SIZE(rk_mc_controls),
};
 . . . . . .
static int snd_rk_mc_probe(struct platform_device *pdev)
{
 . . . . . .
	rk_dailink.cpus->of_node = of_parse_phandle(np,
			"rockchip,i2s-controller", 0);
	if (!rk_dailink.cpus->of_node) {
		dev_err(&pdev->dev,
			"Property 'rockchip,i2s-controller' missing or invalid\n");
		ret = -EINVAL;
		goto put_codec_of_node;
	}

	rk_dailink.platforms->of_node = rk_dailink.cpus->of_node;
 . . . . . .
}
```

这里为 dai link 定义的 cpus dai 数组和 platforms 数组中都只有一个空元素，但在 `probe` 操作中，根据设备树中设备节点的定义，查找了对应的 `of_node`。尽管 cpus dai 和 platforms 引用了相同的 `of_node`，但在 `snd_soc_add_pcm_runtime()` 函数中，为 CPU DAI 添加 `struct snd_soc_component` 的过程是，先查找对应的 `struct snd_soc_dai`，再从 `struct snd_soc_dai` 获得 `struct snd_soc_component`，这也就意味着，为 CPU DAI 查找  `struct snd_soc_component` 时，不会找到没有 `struct snd_soc_dai` 的通用 DMA 引擎的 `struct snd_soc_component`。为 platforms 添加 `struct snd_soc_component` 的过程，则是直接查找所有匹配的 `struct snd_soc_component` 并添加。

在为 PCM 创建 `struct snd_soc_pcm_runtime` 时，即在机器类驱动程序的 `probe` 操作中，如果为 `snd_soc_dai_link` 做了类似于这里的处理，使 CPU DAI 和 platform 指向相同的 `of_node`，且对应于 `of_node` 的设备驱动程序注册了通用 DMA 引擎驱动程序，通用 DMA 引擎的 `struct snd_soc_component` 将会包含在它的 component 链表中。

通用 DMA 引擎驱动程序的 `struct snd_soc_component_driver` 还有一个特别的地方是，指定了 `probe_order` 为 `SND_SOC_COMP_ORDER_LATE`，这使得它的 `probe` 和 `init` 操作，相对而言，执行的更晚，同时它在 `snd_soc_card` 中的位置也更靠后一点。如在 `soc_probe_link_components()` 函数 (位于 `sound/soc/soc-core.c`) 中：
```
static int soc_probe_component(struct snd_soc_card *card,
			       struct snd_soc_component *component)
{
	struct snd_soc_dapm_context *dapm =
		snd_soc_component_get_dapm(component);
	struct snd_soc_dai *dai;
	int probed = 0;
	int ret;

	if (!strcmp(component->name, "snd-soc-dummy"))
		return 0;

	if (component->card) {
		if (component->card != card) {
			dev_err(component->dev,
				"Trying to bind component to card \"%s\" but is already bound to card \"%s\"\n",
				card->name, component->card->name);
			return -ENODEV;
		}
		return 0;
	}

	ret = snd_soc_component_module_get_when_probe(component);
	if (ret < 0)
		return ret;

	component->card = card;
	soc_set_name_prefix(card, component);

	soc_init_component_debugfs(component);

	snd_soc_dapm_init(dapm, card, component);

	ret = snd_soc_dapm_new_controls(dapm,
					component->driver->dapm_widgets,
					component->driver->num_dapm_widgets);

	if (ret != 0) {
		dev_err(component->dev,
			"Failed to create new controls %d\n", ret);
		goto err_probe;
	}

	for_each_component_dais(component, dai) {
		ret = snd_soc_dapm_new_dai_widgets(dapm, dai);
		if (ret != 0) {
			dev_err(component->dev,
				"Failed to create DAI widgets %d\n", ret);
			goto err_probe;
		}
	}

	ret = snd_soc_component_probe(component);
	if (ret < 0) {
		dev_err(component->dev,
			"ASoC: failed to probe component %d\n", ret);
		goto err_probe;
	}
	WARN(dapm->idle_bias_off &&
	     dapm->bias_level != SND_SOC_BIAS_OFF,
	     "codec %s can not start from non-off bias with idle_bias_off==1\n",
	     component->name);
	probed = 1;

	/*
	 * machine specific init
	 * see
	 *	snd_soc_component_set_aux()
	 */
	ret = snd_soc_component_init(component);
	if (ret < 0)
		goto err_probe;

	ret = snd_soc_add_component_controls(component,
					     component->driver->controls,
					     component->driver->num_controls);
	if (ret < 0)
		goto err_probe;

	ret = snd_soc_dapm_add_routes(dapm,
				      component->driver->dapm_routes,
				      component->driver->num_dapm_routes);
	if (ret < 0) {
		if (card->disable_route_checks) {
			dev_info(card->dev,
				 "%s: disable_route_checks set, ignoring errors on add_routes\n",
				 __func__);
		} else {
			dev_err(card->dev,
				"%s: snd_soc_dapm_add_routes failed: %d\n",
				__func__, ret);
			goto err_probe;
		}
	}

	/* see for_each_card_components */
	list_add(&component->card_list, &card->component_dev_list);

err_probe:
	if (ret < 0)
		soc_remove_component(component, probed);

	return ret;
}
 . . . . . .
static int soc_probe_link_components(struct snd_soc_card *card)
{
	struct snd_soc_component *component;
	struct snd_soc_pcm_runtime *rtd;
	int i, ret, order;

	for_each_comp_order(order) {
		for_each_card_rtds(card, rtd) {
			for_each_rtd_components(rtd, i, component) {
				if (component->driver->probe_order != order)
					continue;

				ret = soc_probe_component(card, component);
				if (ret < 0)
					return ret;
			}
		}
	}

	return 0;
}
```

这里用到的 `for_each_comp_order()` 宏定义 (位于 `include/sound/soc-component.h`) 如下：
```
#define SND_SOC_COMP_ORDER_FIRST	-2
#define SND_SOC_COMP_ORDER_EARLY	-1
#define SND_SOC_COMP_ORDER_NORMAL	 0
#define SND_SOC_COMP_ORDER_LATE		 1
#define SND_SOC_COMP_ORDER_LAST		 2

#define for_each_comp_order(order)		\
	for (order  = SND_SOC_COMP_ORDER_FIRST;	\
	     order <= SND_SOC_COMP_ORDER_LAST;	\
	     order++)
```

## Linux 内核 ASoC 通用 DMA 引擎驱动程序的操作

通用 DMA 引擎驱动程序提供的操作如下：
```
static const struct snd_soc_component_driver dmaengine_pcm_component = {
	.name		= SND_DMAENGINE_PCM_DRV_NAME,
	.probe_order	= SND_SOC_COMP_ORDER_LATE,
	.open		= dmaengine_pcm_open,
	.close		= dmaengine_pcm_close,
	.hw_params	= dmaengine_pcm_hw_params,
	.trigger	= dmaengine_pcm_trigger,
	.pointer	= dmaengine_pcm_pointer,
	.pcm_construct	= dmaengine_pcm_new,
};

static const struct snd_soc_component_driver dmaengine_pcm_component_process = {
	.name		= SND_DMAENGINE_PCM_DRV_NAME,
	.probe_order	= SND_SOC_COMP_ORDER_LATE,
	.open		= dmaengine_pcm_open,
	.close		= dmaengine_pcm_close,
	.hw_params	= dmaengine_pcm_hw_params,
	.trigger	= dmaengine_pcm_trigger,
	.pointer	= dmaengine_pcm_pointer,
	.copy_user	= dmaengine_copy_user,
	.pcm_construct	= dmaengine_pcm_new,
};
```

这些操作中，最早被调用的是 `pcm_construct` 操作，也就是 `dmaengine_pcm_new()` 函数。在机器类驱动程序中，调用 `devm_snd_soc_register_card()` 函数注册声卡，在这个函数中有如下调用过程，`snd_soc_register_card()` -> `snd_soc_bind_card()` -> `soc_init_pcm_runtime()` -> `soc_new_pcm()` -> `snd_soc_pcm_component_new()`，`snd_soc_pcm_component_new()` 函数执行各个 `struct snd_soc_component_driver` 的 `pcm_construct` 操作。`snd_soc_pcm_component_new()` 函数定义 (位于 `sound/soc/soc-component.c`) 如下：
```
int snd_soc_pcm_component_new(struct snd_soc_pcm_runtime *rtd)
{
	struct snd_soc_component *component;
	int ret;
	int i;

	for_each_rtd_components(rtd, i, component) {
		if (component->driver->pcm_construct) {
			ret = component->driver->pcm_construct(component, rtd);
			if (ret < 0)
				return soc_component_ret(component, ret);
		}
	}

	return 0;
}
```

回到通用 DMA 引擎驱动程序的 `pcm_construct` 操作 `dmaengine_pcm_new()` 函数，这个函数定义 (位于 `sound/soc/soc-generic-dmaengine-pcm.c`) 如下：
```
static int dmaengine_pcm_new(struct snd_soc_component *component,
			     struct snd_soc_pcm_runtime *rtd)
{
	struct dmaengine_pcm *pcm = soc_component_to_pcm(component);
	const struct snd_dmaengine_pcm_config *config = pcm->config;
	struct device *dev = component->dev;
	struct snd_pcm_substream *substream;
	size_t prealloc_buffer_size;
	size_t max_buffer_size;
	unsigned int i;

	if (config && config->prealloc_buffer_size) {
		prealloc_buffer_size = config->prealloc_buffer_size;
		max_buffer_size = config->pcm_hardware->buffer_bytes_max;
	} else {
		prealloc_buffer_size = 512 * 1024;
		max_buffer_size = SIZE_MAX;
	}

	for_each_pcm_streams(i) {
		substream = rtd->pcm->streams[i].substream;
		if (!substream)
			continue;

		if (!pcm->chan[i] && config && config->chan_names[i])
			pcm->chan[i] = dma_request_slave_channel(dev,
				config->chan_names[i]);

		if (!pcm->chan[i] && (pcm->flags & SND_DMAENGINE_PCM_FLAG_COMPAT)) {
			pcm->chan[i] = dmaengine_pcm_compat_request_channel(
				component, rtd, substream);
		}

		if (!pcm->chan[i]) {
			dev_err(component->dev,
				"Missing dma channel for stream: %d\n", i);
			return -EINVAL;
		}

		snd_pcm_set_managed_buffer(substream,
				SNDRV_DMA_TYPE_DEV_IRAM,
				dmaengine_dma_dev(pcm, substream),
				prealloc_buffer_size,
				max_buffer_size);

		if (!dmaengine_pcm_can_report_residue(dev, pcm->chan[i]))
			pcm->flags |= SND_DMAENGINE_PCM_FLAG_NO_RESIDUE;

		if (rtd->pcm->streams[i].pcm->name[0] == '\0') {
			strscpy_pad(rtd->pcm->streams[i].pcm->name,
				    rtd->pcm->streams[i].pcm->id,
				    sizeof(rtd->pcm->streams[i].pcm->name));
		}
	}

	return 0;
}
```

这个函数分别为播放和录制申请 DMA 通道，并分配 DMA buffer 用于用户空间应用程序、内核 ALSA/ASoc 框架和硬件设备之间的数据交换。`snd_pcm_set_managed_buffer()` 函数分配 DMA 缓冲区。`snd_pcm_set_managed_buffer()` 函数定义 (位于 `sound/core/pcm_memory.c`) 如下：
```
static void preallocate_pages(struct snd_pcm_substream *substream,
			      int type, struct device *data,
			      size_t size, size_t max, bool managed)
{
	if (snd_BUG_ON(substream->dma_buffer.dev.type))
		return;

	substream->dma_buffer.dev.type = type;
	substream->dma_buffer.dev.dev = data;

	if (size > 0 && preallocate_dma && substream->number < maximum_substreams)
		preallocate_pcm_pages(substream, size);

	if (substream->dma_buffer.bytes > 0)
		substream->buffer_bytes_max = substream->dma_buffer.bytes;
	substream->dma_max = max;
	if (max > 0)
		preallocate_info_init(substream);
	if (managed)
		substream->managed_buffer_alloc = 1;
}
 . . . . . .
void snd_pcm_set_managed_buffer(struct snd_pcm_substream *substream, int type,
				struct device *data, size_t size, size_t max)
{
	preallocate_pages(substream, type, data, size, max, true);
}
EXPORT_SYMBOL(snd_pcm_set_managed_buffer);
```

`open` 操作 `dmaengine_pcm_open()` 函数完成数据传输之前的准备工作，这个函数定义如下：
```
static int
dmaengine_pcm_set_runtime_hwparams(struct snd_soc_component *component,
				   struct snd_pcm_substream *substream)
{
	struct snd_soc_pcm_runtime *rtd = asoc_substream_to_rtd(substream);
	struct dmaengine_pcm *pcm = soc_component_to_pcm(component);
	struct device *dma_dev = dmaengine_dma_dev(pcm, substream);
	struct dma_chan *chan = pcm->chan[substream->stream];
	struct snd_dmaengine_dai_dma_data *dma_data;
	struct snd_pcm_hardware hw;

	if (rtd->num_cpus > 1) {
		dev_err(rtd->dev,
			"%s doesn't support Multi CPU yet\n", __func__);
		return -EINVAL;
	}

	if (pcm->config && pcm->config->pcm_hardware)
		return snd_soc_set_runtime_hwparams(substream,
				pcm->config->pcm_hardware);

	dma_data = snd_soc_dai_get_dma_data(asoc_rtd_to_cpu(rtd, 0), substream);

	memset(&hw, 0, sizeof(hw));
	hw.info = SNDRV_PCM_INFO_MMAP | SNDRV_PCM_INFO_MMAP_VALID |
			SNDRV_PCM_INFO_INTERLEAVED;
	hw.periods_min = 2;
	hw.periods_max = UINT_MAX;
	hw.period_bytes_min = 256;
	hw.period_bytes_max = dma_get_max_seg_size(dma_dev);
	hw.buffer_bytes_max = SIZE_MAX;
	hw.fifo_size = dma_data->fifo_size;

	if (pcm->flags & SND_DMAENGINE_PCM_FLAG_NO_RESIDUE)
		hw.info |= SNDRV_PCM_INFO_BATCH;

	/**
	 * FIXME: Remove the return value check to align with the code
	 * before adding snd_dmaengine_pcm_refine_runtime_hwparams
	 * function.
	 */
	snd_dmaengine_pcm_refine_runtime_hwparams(substream,
						  dma_data,
						  &hw,
						  chan);

	return snd_soc_set_runtime_hwparams(substream, &hw);
}

static int dmaengine_pcm_open(struct snd_soc_component *component,
			      struct snd_pcm_substream *substream)
{
	struct dmaengine_pcm *pcm = soc_component_to_pcm(component);
	struct dma_chan *chan = pcm->chan[substream->stream];
	int ret;

	ret = dmaengine_pcm_set_runtime_hwparams(component, substream);
	if (ret)
		return ret;

	return snd_dmaengine_pcm_open(substream, chan);
}
```

这里调用 `snd_soc_dai_get_dma_data()` 函数获得类型为 `struct snd_dmaengine_dai_dma_data` 的 DMA 数据。这是通用 DMA 引擎驱动程序和 I2S 等 DAI 驱动程序的一个约定，即需要在 DAI 驱动程序的 `probe` 操作中设置 DMA 数据，类似于下面这样：
```
struct i2s_dev {
	struct device *dev;
 . . . . . .
	struct snd_dmaengine_dai_dma_data capture_dma_data;
	struct snd_dmaengine_dai_dma_data playback_dma_data;
 . . . . . .
}
 . . . . . .
static int i2s_dai_probe(struct snd_soc_dai *dai)
{
	struct i2s_dev *i2s = snd_soc_dai_get_drvdata(dai);

	snd_soc_dai_init_dma_data(dai,
		i2s->has_playback ? &i2s->playback_dma_data : NULL,
		i2s->has_capture  ? &i2s->capture_dma_data  : NULL);

	return 0;
}
 . . . . . .
static int i2s_init_dai(struct i2s_dev *i2s, struct resource *res,
								 struct snd_soc_dai_driver **dp)
{
 . . . . . .
		i2s->playback_dma_data.addr = res->start + I2S_TXFIFODATA;
		i2s->playback_dma_data.addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES;
		i2s->playback_dma_data.maxburst = 8;

 . . . . . .
		i2s->capture_dma_data.addr = res->start + I2S_RXFIFODATA;
		i2s->capture_dma_data.addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES;
		i2s->capture_dma_data.maxburst = 8;
 . . . . . .
}
```

`dmaengine_pcm_open()` 函数主要是为流设置了运行时硬件参数。

对于其它操作，暂时不做太多说明。

Done.
