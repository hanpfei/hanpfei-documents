---
title: SOF 拓扑
date: 2024-02-25 11:08:29
categories: Linux 内核
tags:
- Linux 内核
---

拓扑定义了固件使用的音频处理流水线。在 SOF 中，拓扑使用 [M4](http://www.gnu.org/software/m4/m4.html) 宏语言定义，它将转换为基于文本的 ALSA 拓扑文件（基于 alsaconf 语法）。该拓扑文件使用 alsatplg 进一步处理为二进制，可以发送到固件。

有关 ALSA 拓扑的更多详细信息请参见 [此处](https://www.alsa-project.org/main/index.php/ALSA_topology)。

## 1. 拓扑要素

拓扑通常包含以下的项：

1. Widgets
2. Tokens
3. Kcontrols
4. 流水线
5. 后端 DAI’s
6. DAI 链接配置

以下部分将描述如何使用 M4 定义它们。

### 1.1 Widgets

Widgets 定义组成音频处理流水线的各个独立音频处理组件。Widgets 的示例有音量、采样率转换器、音调发生器、主机、dai、缓冲区等。Widgets 是使用宏来定义的，这些宏扩展为包含**包含了音频处理流水线信息、widget 类型和 widget 特定数据等详细信息的相应部分**。Widget 特定数据包含在初始化期间配置 widget 所需的信息。这些是使用 vendor 元组定义的，可以是字符串、单词或短类型。

### 1.2 Tokens/Vendor 元组

Tokens 或 vendor 元组允许我们添加特定于 widget 或特定于平台的数据。Widgets 可以具有所有 widget 类型通用的 tokens，以及配置特定 widget 类型所需的定制 tokens。例如，在音量组件的例子中，配置数据可以包括特定于 pga 类型 widget 的音量步长类型和以毫秒为单位的音量斜坡步长。这些是在 tokens.m4 文件的 sof_volume_tokens 数组中预定义的，如下所示：
```
SectionVendorTokens."sof_volume_tokens" {
    SOF_TKN_VOLUME_RAMP_STEP_TYPE           "250"
    SOF_TKN_VOLUME_RAMP_STEP_MS             "251"
}
```

pga widget 宏示例如下：
```
W_PGA(name, format, periods_sink, periods_source, preload, kcontrol0. kcontrol1...etc)
```

这个宏接受如下参数：
**name**：Widget 的名称
**format**：音频格式
**periods_sink**：Sink 周期数
**periods_source**：Source 周期数
**preload**：预加载计数
**kcontrol0…kcontroln**：与 widget 关联的 kcontrols 列表

W_PGA 宏扩展为包括以下部分：

**SectionWidget**：这个部分包括流水线信息、widget 类型、数据部分以及与 widget 关联的混音器。
**SectionData**：这个部分包括 widget 的配置数据。注意有两个数据部分。第一个用于包含通用的 widget tokens，例如 sink 周期数，source 周期数和预加载计数，它们是 “word” tokens。第二个用于包含组件的音频格式，其类型为 “string”。请注意，即使 tokens 属于相同的 sof_comp_tokens 数组定义，也需要在不同的部分中指定不同的类型。
**SectionVendorTuples**：这个部分包含 tokens 及其各自的值。
**Mixer**：Mixer 部分包含与 widget 关联的 kcontrols 列表。Kcontrols 根据类型使用不同的宏来定义。更多详细信息可以在 Kcontrols 部分找到。
```
define(`W_PGA',
`SectionVendorTuples."'N_PGA($1)`_tuples_w" {'
`       tokens "sof_comp_tokens"'
`       tuples."word" {'
`               SOF_TKN_COMP_PERIOD_SINK_COUNT'         STR($3)
`               SOF_TKN_COMP_PERIOD_SOURCE_COUNT'       STR($4)
`               SOF_TKN_COMP_PRELOAD_COUNT'             STR($5)
`       }'
`}'
`SectionData."'N_PGA($1)`_data_w" {'
`       tuples "'N_PGA($1)`_tuples_w"'
`}'
`SectionVendorTuples."'N_PGA($1)`_tuples_str" {'
`       tokens "sof_comp_tokens"'
`       tuples."string" {'
`               SOF_TKN_COMP_FORMAT'    STR($2)
`       }'
`}'
`SectionData."'N_PGA($1)`_data_str" {'
`       tuples "'N_PGA($1)`_tuples_str"'
`}'
`SectionWidget."'N_PGA($1)`" {'
`       index "'PIPELINE_ID`"'
`       type "pga"'
`       no_pm "true"'
`       data ['
`               "'N_PGA($1)`_data_w"'
`               "'N_PGA($1)`_data_str"'
`       ]'
`       mixer ['
             $6
`       ]'

`}')
```

其它 widget 宏可以在 [SOFT](https://github.com/thesofproject/soft) 存储库的 topology/m4 文件夹下各自的宏文件中找到。

### 1.3 Kcontrols

这些是与 widgets 关联并被导出到用户空间的内核 controls。kcontrol 的示例有音量控制、静音开关等。这些是使用宏定义的，其中包括流水线 ID、IO 处理程序等信息以及其它 control 特定信息（例如音量控制的 tlv 数据）。目前，我们只有用于混音器类型 controls 的预定义宏。未来将添加枚举/字节类型的 controls。

混音器类型 controls 的示例 kcontrol 宏如下：
```
C_CONTROLMIXER(name, index, ops, max, invert, tlv, KCONTROL_CHANNELS)
```

这个宏的参数如下：

**name**：Mixer controls 的名称
**index**：流水线 ID
**ops**：kcontrol IO 处理程序 ID
**max**：最大值
**invert**：指示值是否反转的布尔值
**tlv**：音量的 tlv 数据
**kcontrol_channels**：支持的通道数量和名称

### 1.4 音频数据处理流水线

音频数据处理流水线定义包含以下内容：

1. Widget 描述：这些是构成流水线的 widgets 的详细信息
2. Kcontrol 描述：这些是与流水线中的 widgets 关联的 kcontrol
3. 流水线图：这些指定了流水线中 widgets 之间的连接
4. PCM 能力：这些包含有关流水线支持的格式、采样率、通道等方面的 pcm 能力的详细信息。这个宏的定义如下：
```
PCM_CAPABILITIES(name, formats, rate_min, rate_max, channels_min, channels_max, periods_min, periods_max, period_size_min, period_size_max, buffer_size_min, buffer_size_max)
```

考虑以下音频播放流水线示例（如 pipeline-volume-playback.m4 中所述）
```
host PCM_P --> B0 --> Volume 0 --> B1 --> sink DAI0
```

这个流水线描述包括以下内容：

1. Widgets：4 个 widgets 对应于 Host，音量和 2 个缓冲区实例
2. Kcontrols：1 个与音量组件关联的混音器类型 kcontrol
3. 流水线图：展示了如上所示的 widgets 间的连接
4. PCM 能力：音频播放流水线支持的能力如下：

**supported formats**：S32_LE，S24_LE，S16_LE
**min sample rate**：48000
**max sample rate**：48000
**min number of channels**：2
**max number of channels**：8
**min number of periods**：2
**max number of periods**：16
**min period size**：192
**max period size**：16384
**min buffer size**：65536
**max buffer size**：65536

流水线中的 DAI 组件是使用单独流水线来定义的，与流水线是采集流水线还是播放流水线相对应。请参阅下一节了解更多详细信息。

### 1.5 后端 DAI

这个部分介绍用于播放/采集流水线的 BE（后端）DAI。BE DAI 被定义为一个单独的流水线，这个流水线由 DAI widget和流水线图部分组成，流水线图部分包含 BE DAI 和流水线缓冲区之间的连接。比如，让我们考虑前一个部分展示的播放流水线的情况。流水线图部分将包含 BE DAI 和缓冲区 B1 之间的连接。后端 DAI 使用定义如下的 DAI_ADD 宏添加：
```
DAI_ADD(pipeline, pipe id, dai type, dai_index, dai_be, buffer,
periods, format, frames, deadline, priority, core)
```

**pipeline**：DAI 流水线的名称，例如：播放或采集 dai 流水线如 pipe-dai-playback.m4 或 pipe-dai-capture.m4 中定义
**pipe id**：与 DAI 关联的流水线的 ID
**dai type**：DAI 类型，如 SSP 或 DMIC 或 HDA
**dai_index**：固件中 dai 的索引。请注意，不同类型的 DAI 可以具有相同的 dai_index。dai_index 信息可以通过查看固件中特定于平台的 dai 数组定义来找到。例如，对于 Apollo Lake，这些是在 src/platform/apollolake/dai.c 中定义的。
**dai_be**：CPU DAI 的名称，在平台驱动程序的 DAI 数组中定义。
**buffer**：DAI 连接到的 source/sink 缓冲区。这样就完成了流水线图连接。
**periods**：周期数
**format**：DAI 音频格式
**frames**：每个周期的帧数
**deadline**：流水线以毫秒为单位的截止时间
**priority**：需要为 dai 流水线分配的优先级
**core**：运行流水线的核心数

### 1.6 后端 DAI 链接配置

这个部分介绍音频流水线中 BE DAI 链接的配置详细信息。BE DAI 配置使用如下的宏定义：
```
DAI_CONFIG(type, dai_index, link_id, name, config)
```

其中：

**type**：DAI 的类型，如：SSP 或 DMIC 或 HDA
**dai_index**：固件中定义的 dai 的索引
**link_id**：SOF 驱动程序中定义的链接的 CPU DAI 的 ID。请注意，链接 ID 是从 0 开始线性递增的数字，与 DAI 类型无关
**name**：SOF 驱动程序中定义的 CPU DAI 名称
**config**：配置详细信息，取决于 DAI 的类型。

SSP 的配置参数使用以下宏定义：
```
SSP_CONFIG(format, mclk, bclk, fsync, tdm, ssp_config_data)
```

其中：

**format**：SSP 的格式，如 I2S 或 DSP_A 或 DSP_B 等
**mclk**：provider 时钟（Hz）
**bclk**：以 Hz 为单位的位时钟
**fsync**：帧同步
**TDM**：TDM 信息，包括时隙、宽度、tx 掩码和 rx 掩码
**ssp_config_data**：包括采样有效位和 mclk ID。有些 SoC 暴露了多个 mclk。因此需要指定正确的 mclk ID。如果省略，则默认为 0。

DMIC 的配置参数使用以下宏定义：
```
DMIC_CONFIG(driver_version, clk_min, clk_max, duty_min, duty_max,
sample_rate, fifo word length, type, dai_index, pdm controller
config)
```

其中：

**driver version**：固件中的 dmic 驱动程序版本
**clk_min**：支持的最小时钟
**clk_max**：支持的最大时钟
**duty min/max**：最小和最大占空比
**sample rate**：音频采样率
**fifo word length**：采样字长
**type**：DAI 类型
**dai_index**：固件中定义的 dai 的索引
**pdm controller config**：PDM 控制器配置，指示激活的 PDM 的数量、通道数量等。这些是特定于平台的，可以选择预定义的配置，例如 MONO_PDM0_MICA、STEREO_PDM0、FOUR_CH_PDM0_PDM1 等。

### 1.7 DSP 核心索引

拓扑文件可以指定将在哪个 DSP 核上调度流水线或组件。

要为流水线指定 DSP 核，请使用位于 tools/topology/m4/pipeline.m4 中的 SOF_TKN_SCHED_CORE token：
```
W_PIPELINE(stream, period, priority, core, initiator, platform)
...
`               SOF_TKN_SCHED_CORE'             STR($4)
...
```

然后在流水线定义中指定此“核心”，例如在 tools/topology/sof/pipe-dai-playback.m4 中：
```
W_PIPELINE(N_DAI_OUT, SCHEDULE_PERIOD, SCHEDULE_PRIORITY, SCHEDULE_CORE, SCHEDULE_TIME_DOMAIN, pipe_dai_schedule_plat)
```

要为组件/widget 指定 DSP 核，请使用位于 tools/topology/m4/pga.m4 中的 SOF_TKN_COMP_CORE_ID token：
```
dnl W_PGA(name, format, periods_sink, periods_source, core, kcontrol0. kcontrol1...etc)
...
`               SOF_TKN_COMP_CORE_ID'                   STR($6)
...
```

## 2. 创建一个新的拓扑

以下部分将展示如何定义单流水线和多流水线拓扑。

### 2.1 单流水线拓扑示例

创建新拓扑的最简单方法是使用一个预定义的流水线，例如 pipe-volume-playback，并提供必要的详细信息，例如 BE（后端）/FE（前端）DAI 信息。SOFT 存储库中有一些预定义的流水线，用于使用或不使用音量和 src 组件进行音频播放和音频采集。这部分演示如何使用一个预定义流水线来创建新拓扑。

**第 1 步**：添加预定义流水线

在这一步中，我们使用 PIPELINE_PCM_ADD 宏添加预定义流水线。该宏定义如下：

**pipeline**：预定义流水线的名称
**pipe id**：流水线 ID。这应该是标识流水线的唯一 ID
**pcm**：PCM ID。这将用于绑定到正确的前端 DAI 链接
**max channels**：最大音频通道数
**format**：流水线的音频格式
**frames**：每个周期的帧数
**deadline**：流水线调度的最后期限
**priority**：流水线优先级
**core**：核心 ID

示例：要添加带有音量组件的音频播放流水线：
```
host PCM_P --> B0 --> Volume 0 --> B1 --> sink DAI0
```

其中 deadline 为 1000us，每个周期 48 帧，音频格式为 s32le，PIPELINE_PCM_ADD 宏应该包含如下参数：
```
PIPELINE_PCM_ADD(sof/pipe-volume-playback.m4, 1, 0, 2, s32le, 48, 1000, 0, 0)
```

请注意，上面的定义中流水线 ID 为 1，PCM ID 为 0。稍后将使用它们把 PCM 绑定到流水线。

**第 2 步**：添加 BE DAI

完成流水线定义后，下一步是添加 BE DAI 并将其连接到所需的流水线。这是使用 1.5 节中描述的 DAI_ADD 宏来完成的。

示例：以下定义将 SSP 5 连接到步骤 1 中添加的流水线。
```
DAI_ADD(sof/pipe-dai-playback.m4, 1, SSP, 5, SSP5-Codec,
PIPELINE_SOURCE_1, 2, s24le, 48, 1000, 0, 0)
```

注意：PIPELINE_SOURCE_1 是 SSP 5 连接到的 ID 为 1 的流水线中的端点。 “SSP5-Codec” 是 SOF 驱动程序中定义的 SSP5 CPU DAI 的名称。

**第 3 步**：把 PCM 绑定到流水线

下一步是将流水线与 PCM 或 FE DAI 链路绑定。这是使用宏 PCM_PLAYBACK_ADD、PCM_DUPLEX_ADD 或 PCM_CAPTURE_ADD 完成的，具体取决于流水线所需的能力。

示例：对于步骤 1 和 2 中定义的播放流水线，宏 PCM_PLAYBACK_ADD 用于将 ID 为 1 的流水线与 ID 为 0 的 PCM 绑定，如下所示：
```
PCM_PLAYBACK_ADD(Port5, 0, PIPELINE_PCM_1)
```

其中 “Port5” 为 PCM 的名称，0 为 PCM ID，最后一个参数 PIPEPINE_PCM_1 标识 ID 为 1 的流水线以绑定 PCM。

**第 4 步**：定义 BE DAI 配置

拓扑定义的最后一步包含拓扑中 BE DAI 的配置。

在示例的情况中，有一个 BE DAI (SSP 5)，此步骤使用 DAI_CONFIG 宏定义 SSP 5 的配置，如第 1.6 节中所述。
```
DAI_CONFIG(SSP, 5, 0, SSP5-Codec,
    SSP_CONFIG(I2S, SSP_CLOCK(mclk, 24576000, codec_mclk_in),
    SSP_CLOCK(bclk, 3072000, codec_slave),
    SSP_CLOCK(fsync, 48000, codec_slave),
    SSP_TDM(2, 32, 3, 3),
    SSP_CONFIG_DATA(SSP, 5, 24)))
```

将上述 4 个步骤中的不同部分放在一起，完整的拓扑定义如下所示：
```
# Low Latency playback pipeline 1 on PCM 0 using max 2 channels of s32le.
# Schedule 48 frames per 1000us deadline on core 0 with priority 0
PIPELINE_PCM_ADD(sof/pipe-volume-playback.m4, 1, 0, 2, s32le, 48, 1000, 0, 0)

# playback DAI is SSP5 using 2 periods
# Buffers use s24le format, with 48 frame per 1000us on core 0 with priority 0
DAI_ADD(sof/pipe-dai-playback.m4, 1, SSP, 5, SSP5-Codec,
PIPELINE_SOURCE_1, 2, s24le, 48, 1000, 0, 0)

# PCM Low Latency, id 0
PCM_PLAYBACK_ADD(Port5, 0, PIPELINE_PCM_1)

DAI_CONFIG(SSP, 5, 0, SSP5-Codec,
    SSP_CONFIG(I2S, SSP_CLOCK(mclk, 24576000, codec_mclk_in),
    SSP_CLOCK(bclk, 3072000, codec_slave),
    SSP_CLOCK(fsync, 48000, codec_slave),
    SSP_TDM(2, 32, 3, 3),
    SSP_CONFIG_DATA(SSP, 5, 24)))
```

下图显示了第 2.1 节中定义的拓扑，突出显示了流水线中的组件以及它们之间的连接。下图中的每个节点表示一个组件，如下所示：

**Passthrough Playback 0**：流水线的名称
**PCM0P**：FE DAI
**BUF1.0**：流水线 1 中的缓冲区组件 0
**PGA1.0**：流水线 1 中的音量组件
**BUF1.1**：流水线 1 中的缓冲区组件 1
**SSP5.OUT**：BE DAI 对应于 SSP 5

![tplg1](https://upload-images.jianshu.io/upload_images/1315506-ff4113bb5c024e53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 多流水线拓扑示例

典型的拓扑定义包含流水线的多个实例、每个流水线各自的后端 DAI 以及 DAI 配置。下面给出了一个拓扑定义示例（示例取自 sof-apl-da7219.m4）：

有 4 个流水线，各个流水线分别用于扬声器播放、耳机播放、耳机采集和 DMIC 采集。

**第 1 步**：定义流水线
```
# Low Latency playback pipeline 1 on PCM 0 using max 2 channels of s32le.
# Schedule 48 frames per 1000us deadline on core 0 with priority 0
PIPELINE_PCM_ADD(sof/pipe-volume-playback.m4,
    1, 0, 2, s32le,
    48, 1000, 0, 0)

# Low Latency playback pipeline 2 on PCM 1 using max 2 channels of s32le.
# Schedule 48 frames per 1000us deadline on core 0 with priority 0
PIPELINE_PCM_ADD(sof/pipe-volume-playback.m4,
    2, 1, 2, s32le,
    48, 1000, 0, 0)

# Low Latency capture pipeline 3 on PCM 1 using max 2 channels of s32le.
# Schedule 48 frames per 1000us deadline on core 0 with priority 0
PIPELINE_PCM_ADD(sof/pipe-volume-capture.m4,
    3, 1, 2, s32le,
    48, 1000, 0, 0)

# Low Latency capture pipeline 4 on PCM 0 using max 4 channels of s32le.
# Schedule 48 frames per 1000us deadline on core 0 with priority 0
#PIPELINE_PCM_ADD(sof/pipe-volume-capture.m4,
PIPELINE_PCM_ADD(sof/pipe-passthrough-capture.m4,
    4, 99, 4, s32le,
    48, 1000, 0, 0)
```

**第 2 步**：为各个流水线添加 BE DAI

有 4 个 DAI，步骤一中显示的 4 个流水线各一个：
```
# playback DAI is SSP5 using 2 periods
# Buffers use s16le format, with 48 frame per 1000us on core 0 with priority 0
DAI_ADD(sof/pipe-dai-playback.m4,
    1, SSP, 5, SSP5-Codec,
    PIPELINE_SOURCE_1, 2, s16le,
    48, 1000, 0, 0)

# playback DAI is SSP1 using 2 periods
# Buffers use s16le format, with 48 frame per 1000us on core 0 with priority 0
DAI_ADD(sof/pipe-dai-playback.m4,
    2, SSP, 1, SSP1-Codec,
    PIPELINE_SOURCE_2, 2, s16le,
    48, 1000, 0, 0)

# capture DAI is SSP1 using 2 periods
# Buffers use s16le format, with 48 frame per 1000us on core 0 with priority 0
DAI_ADD(sof/pipe-dai-capture.m4,
    3, SSP, 1, SSP1-Codec,
    PIPELINE_SINK_3, 2, s16le,
    48, 1000, 0, 0)

# capture DAI is DMIC0 using 2 periods
# Buffers use s16le format, with 48 frame per 1000us on core 0 with priority 0
DAI_ADD(sof/pipe-dai-capture.m4,
    4, DMIC, 0, dmic01,
    PIPELINE_SINK_4, 2, s32le,
    48, 1000, 0, 0)
```

**第 3 步**：绑定 PCM 和流水线

接下来的三个宏定义流水线中后端 DAI 的 PCM 部分。请注意，PCM ID 1 是双工 PCM，表明它与流水线 2 和 3 关联。
```
PCM_PLAYBACK_ADD(Speakers, 0, PIPELINE_PCM_1)
PCM_DUPLEX_ADD(Headset, 1, PIPELINE_PCM_2, PIPELINE_PCM_3)
PCM_CAPTURE_ADD(DMIC01, 99, PIPELINE_PCM_4)
```

**第 4 步**：BE DAI 配置

拓扑中的最后一部分定义了 DAI 配置。请注意，只有 3 个 DAI_CONFIG。耳机播放 dai 和采集 DAI 使用相同的配置，因为它们与相同的 SSP1-Codec DAI 关联。
```
#SSP 5 (ID: 0) with 19.2 MHz mclk with MCLK_ID 0 (unused), 1.536 MHz blck
DAI_CONFIG(SSP, 5, 0, SSP5-Codec,
    SSP_CONFIG(I2S, SSP_CLOCK(mclk, 19200000, codec_mclk_in),
        SSP_CLOCK(bclk, 1536000, codec_slave),
        SSP_CLOCK(fsync, 48000, codec_slave),
        SSP_TDM(2, 16, 3, 3),
        SSP_CONFIG_DATA(SSP, 5, 16, 0)))

#SSP 1 (ID: 1) with 19.2 MHz mclk with MCLK_ID 0, 1.92 MHz bclk
DAI_CONFIG(SSP, 1, 1, SSP1-Codec,
    SSP_CONFIG(I2S, SSP_CLOCK(mclk, 19200000, codec_mclk_in),
        SSP_CLOCK(bclk, 1920000, codec_slave),
        SSP_CLOCK(fsync, 48000, codec_slave),
        SSP_TDM(2, 20, 3, 3),
        SSP_CONFIG_DATA(SSP, 1, 16, 0)))

# dmic01 (id: 2)
DAI_CONFIG(DMIC, 0, 2, dmic01,
    DMIC_CONFIG(1, 500000, 4800000, 40, 60, 48000,
    DMIC_WORD_LENGTH(s32le), DMIC, 0,
    # FIXME: what is the right configuration
    # PDM_CONFIG(DMIC, 0, FOUR_CH_PDM0_PDM1)))
    PDM_CONFIG(DMIC, 0, STEREO_PDM0)))
```

下图显示了第 2.2 节中定义的拓扑。

![tplg2](https://upload-images.jianshu.io/upload_images/1315506-6a1db50a3af03db5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3. 调试拓扑

SOF 拓扑文件包含 debug.m4 以及几个用于输出数据的简单宏。这些用于从 dai_add、pcm_add 和流水线图创建阶段提取信息。

调试宏使用 errprint 打印到 stderr，因此你可以区分实际的宏输出和调试消息。要正确打印流水线图，你需要用 DEBUG_START 和 DEBUG_END 包围你的 m4。你可以参考 sof-apl-pcm512x.m4 和 sof-apl-da7219.m4 中的示例。

当前定义了 2 种调试类型：GRAPH 和 INFO。GRAPH 生成描述拓扑图连接的 dot 文件。INFO 生成主要与 dai 索引相关的诊断消息。

你可以像这样调用调试（在拓扑文件夹中）：
```
m4 -I m4 -I common -I platform/common --define=GRAPH sof-apl-da7219.m4 > /dev/null
m4 -I m4 -I common -I platform/common --define=INFO sof-apl-da7219.m4 > /dev/null
```

要生成流水线图的图像：
```
m4 -I m4 -I common -I platform/common --define=GRAPH sof-apl-da7219.m4 2> test.dot
dot test.dot -Tpng -o tplg.png
```

INFO 消息被类似 C 的注释标记包围，因此你实际上可以将这两条消息推送到 dot 文件中：
```
m4 -I m4 -I common -I platform/common --define=GRAPH --define=INFO sof-apl-da7219.m4 2> test.dot
```

## 4. 缩略语

**DAI**：Digital Audio Interface，数字音频接口
**BE**：Back End，后端
**FE**：Front End，前端
**DMIC**：Digital microphone，数字麦克风
**SSP**：Serial Synchronous Port，串行同步端口

[原文](https://thesofproject.github.io/latest/developer_guides/topology/topology.html)。

Done.
