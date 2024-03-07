---
title: SOF项目简介
date: 2024-03-05 21:53:37
categories: Linux 内核
tags:
- Linux 内核
---

拓扑接口允许开发者以文本文件格式定义 DSP 拓扑，并将文本格式的拓扑转换为内核可以理解的二进制拓扑表示。目前的拓扑核心可以识别的对象类型如下：

 * Controls (mixer，enumerated 和 byte)，包括 TLV 数据
 * PCM (前端 DAI & DAI 链接)
 * DAPM widgets
 * DAPM 图元素
 * 物理 DAI & DAI 链接
 * 各个对象类型的私有数据
 * 清单 Manifest (包含各个对象类型的个数)

本文介绍的是以文本格式定义 ALSA 拓扑的基础 alsaconf 语法，其它以文本格式定义 ALSA 拓扑的方法为 [SoF 的 M4 宏语言](https://thesofproject.github.io/latest/developer_guides/topology/topology.html) 和 [拓扑 2.0 语法](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html) 。用 [M4 宏语言](https://thesofproject.github.io/latest/developer_guides/topology/topology.html) 和 [拓扑 2.0 语法](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html) 编写的 ALSA 拓扑配置文件，会首先被转换为本文说明的语法格式的配置文件，进而转换为二进制拓扑文件。

基础 alsaconf 语法定义的 ALSA 拓扑配置文件中的各个对象和结构，与 Linux 内核拓扑驱动程序代码的对应关系比较直接，同时它也是 alsatplg 编译器解码二进制拓扑配置文件生成的拓扑配置文件所用格式。基础 alsaconf 语法可以理解为 ALSA 拓扑的汇编语言。

本文的内容主要来自 *alsa-lib/include/topology.h* 头文件的注释。

## 拓扑文件格式

拓扑文件格式使用标准的 ALSA 配置文件格式来描述各个拓扑对象类型。这允许拓扑对象包含其它拓扑对象作为其定义的一部分。比如，使用相同 TLV (阈限值，Threshold Limit Value) 数据的多个 control 对象可以共享 TLV 数据对象。

### Controls

拓扑音频 controls 可分为三种类型：

 * Mixer control
 * Enumerated control
 * Byte control

每个 control 类型可以包含 TLV 数据、私有数据、操作，也可以属于 widget 对象。

#### Control 操作

驱动程序的 Kcontrol 回调 info()，get() 和 put() 操作映射到拓扑配置文件中的 CTL ops 部分。ctl ops 部分可以使用标准 kcontrol 类型的标准名称 (如下所列) 来分配操作，或者使用 ID 号 (>256) 映射到定制的驱动程序 controls。

```
ops."ctl" {
    info "volsw"
    get "257"
    put "257"
}
```

这个映射显示 `info()` 使用标准的 “volsw” info 回调，然而 `get()` 和 `put()` 映射到定制的驱动程序回调。

Control 的 `get()`，`put()` 和 `info()` 调用的标准操作名称有：

 * volsw
 * volsw_sx
 * volsw_xr_sx
 * enum
 * bytes
 * enum_value
 * range
 * strobe

这些标准的操作名称会被 alsatplg 编译器编译为一个 ID。在 Linux 内核的 *linux/sound/soc/soc-topology.c* 文件中，有一个表建立了这些 ID 和具体的操作实现函数之间的映射：
```
/* mapping of Kcontrol types and associated operations. */
static const struct snd_soc_tplg_kcontrol_ops io_ops[] = {
	{SND_SOC_TPLG_CTL_VOLSW, snd_soc_get_volsw,
		snd_soc_put_volsw, snd_soc_info_volsw},
	{SND_SOC_TPLG_CTL_VOLSW_SX, snd_soc_get_volsw_sx,
		snd_soc_put_volsw_sx, NULL},
	{SND_SOC_TPLG_CTL_ENUM, snd_soc_get_enum_double,
		snd_soc_put_enum_double, snd_soc_info_enum_double},
	{SND_SOC_TPLG_CTL_ENUM_VALUE, snd_soc_get_enum_double,
		snd_soc_put_enum_double, NULL},
	{SND_SOC_TPLG_CTL_BYTES, snd_soc_bytes_get,
		snd_soc_bytes_put, snd_soc_bytes_info},
	{SND_SOC_TPLG_CTL_RANGE, snd_soc_get_volsw_range,
		snd_soc_put_volsw_range, snd_soc_info_volsw_range},
	{SND_SOC_TPLG_CTL_VOLSW_XR_SX, snd_soc_get_xr_sx,
		snd_soc_put_xr_sx, snd_soc_info_xr_sx},
	{SND_SOC_TPLG_CTL_STROBE, snd_soc_get_strobe,
		snd_soc_put_strobe, NULL},
	{SND_SOC_TPLG_DAPM_CTL_VOLSW, snd_soc_dapm_get_volsw,
		snd_soc_dapm_put_volsw, snd_soc_info_volsw},
	{SND_SOC_TPLG_DAPM_CTL_ENUM_DOUBLE, snd_soc_dapm_get_enum_double,
		snd_soc_dapm_put_enum_double, snd_soc_info_enum_double},
	{SND_SOC_TPLG_DAPM_CTL_ENUM_VIRT, snd_soc_dapm_get_enum_double,
		snd_soc_dapm_put_enum_double, NULL},
	{SND_SOC_TPLG_DAPM_CTL_ENUM_VALUE, snd_soc_dapm_get_enum_double,
		snd_soc_dapm_put_enum_double, NULL},
	{SND_SOC_TPLG_DAPM_CTL_PIN, snd_soc_dapm_get_pin_switch,
		snd_soc_dapm_put_pin_switch, snd_soc_dapm_info_pin_switch},
};
```

如上面的映射中，`info()` 使用标准的 “volsw” info 回调，这个回调具体将是内核的 `snd_soc_info_volsw()` 函数。为拓扑配置文件中定义的 control 创建内核 control 对象并绑定操作函数在 `soc_tplg_kcontrol_elems_load()`、`soc_tplg_dmixer_create()`/`soc_tplg_denum_create()`/`soc_tplg_dbytes_create()` 和 `soc_tplg_kcontrol_bind_io()` 等函数中完成，不同类型的 control 在不同的函数中创建。

#### Control 访问权限

Control 访问权限可以使用 “access” 区指定。如果没有定义 “access” 区，则将为普通的和 TLV controls 设置默认的 RW 访问标记。
```
access [
    read
    write
    tlv_command
]
```

标准的访问标记如下：

 * read
 * write
 * read_write
 * volatile
 * timestamp
 * tlv_read
 * tlv_write
 * tlv_read_write
 * tlv_command
 * inactive
 * lock
 * owner
 * tlv_callback
 * user

Control 的访问标记在各个 control 类型的创建函数里处理。

#### Control TLV Data

Control 也可以使用 TLV 书记来表示 dB 信息。这可以通过定义 TLV 区域，并在 control 中使用 TLV 区域完成。DB Scale 类型的 TLV 数据定义如下：
```
scale {
    min "-9000"
    step "300"
    mute "1"
}
```

其中 min、step 和 mute 的值及其含义与驱动程序代码中定义的完全一致。

只有 mixer control 可能带有 TLV 数据，在 Linux 内核中，control 的 TLV 数据由 *linux/sound/soc/soc-topology.c* 文件中的 `soc_tplg_dmixer_create()`、`soc_tplg_create_tlv()` 和 `soc_tplg_create_tlv_db_scale()` 等函数处理。

#### Control 通道映射

 Control 还可以指定与哪些通道进行映射。这对用户空间很有用，因为它允许应用程序确定左和右等的正确控制通道。通道映射定义如下：
```
channel."name" {
    reg "0"
    shift "0"
}
```

通道映射 reg 是 control 的寄存器偏移量，shift 是通道的寄存器内的位移位，section 名是通道名，可以是以下值之一：
```
* mono        # mono stream
* fl      # front left
* fr      # front right
* rl      # rear left
* rr      # rear right
* fc      # front center
* lfe     # LFE
* sl      # side left
* sr      # side right
* rc      # rear center
* flc     # front left center
* frc     # front right center
* rlc     # rear left center
* rrc     # rear right center
* flw     # front left wide
* frw     # front right wide
* flh     # front left high
* fch     # front center high
* frh     # front right high
* tc      # top center
* tfl     # top front left
* tfr     # top front right
* tfc     # top front center
* trl     # top rear left
* trr     # top rear right
* trc     # top rear center
* tflc        # top front left center
* tfrc        # top front right center
* tsl     # top side left
* tsr     # top side right
* llfe        # left LFE
* rlfe        # right LFE
* bc      # bottom center
* blc     # bottom left center
* brc     # bottom right center
```

#### Control 私有数据

Control 也可以有私有数据。这可以通过定义私有数据区并将该区包含在 control 中来实现。私有数据区定义如下：
```
SectionData."pdata for EQU1" {
   file "/path/to/file"
   bytes "0x12,0x34,0x56,0x78"
   shorts "0x1122,0x3344,0x5566,0x7788"
   words "0xaabbccdd,0x11223344,0x66aa77bb,0xefef1234"
   tuples "section id of the vendor tuples"
};
```

file，bytes，shorts，words 和 tuples 关键字是互斥的，因为私有数据应该只从一个来源获取。私有数据可以从单独的文件中读取，或在拓扑文件中使用 bytes，shorts，words 或 tuples 关键字定义。关键字 tuples 用于定义特定于供应商的元组。请参阅 Vendor Tokens 和 Vendor tuples 区。

#### 如何定义具有私有数据的元素

元素可以引用单个数据区，也可以引用多个数据区。

##### 引用单个数据区

```
Sectionxxx."element name" {
   ...
   data "name of data section"     # optional private data
}
```

##### 引用多个数据区

```
Sectionxxx."element name" {
   ...
   data [                      # optional private data
       "name of 1st data section"
       "name of 2nd data section"
       ...
   ]
}
```

这些 section 的数据将按照它们在列表中的顺序合并，作为元素的私有数据。

##### Vendor Tokens

Vendor token 列表被定义为一个新的 section。每个 token 元素是一对字符串 ID 和整数值。ID 和值都是特定于 vendor 的。
```
SectionVendorTokens."id of the vendor tokens" {
   comment "optional comments"
   VENDOR_TOKEN_ID1 "1"
   VENDOR_TOKEN_ID2 "2"
   VENDOR_TOKEN_ID3 "3"
   ...
}
```

##### Vendor Tuples

Vendor tuples 被定义为一个新的 section。它包含对 vendor token 列表和几个元组数组的引用。所有数组共享一个由 tokens 关键字定义的 vendor token 列表。每个元组数组对应一个特定的类型，由 tuples 关键字后跟字符串定义。支持的类型有：string，uuid，bool，byte，short 和 word。
```
SectionVendorTuples."id of the vendor tuples" {
	tokens "id of the vendor tokens"

	tuples."string" {
		VENDOR_TOKEN_ID1 "character string"
		...
	}

	tuples."uuid" {			# 16 characters separated by commas
		VENDOR_TOKEN_ID2 "0x01,0x02,...,0x0f"
		...
	}

	tuples."bool" {
		VENDOR_TOKEN_ID3 "true/false"
		...
	}

	tuples."byte" {
		VENDOR_TOKEN_ID4 "0x11"
		VENDOR_TOKEN_ID5 "0x22"
		...
	}

	tuples."short" {
		VENDOR_TOKEN_ID6 "0x1122"
		VENDOR_TOKEN_ID7 "0x3344"
		...
	}

	tuples."word" {
		VENDOR_TOKEN_ID8 "0x11223344"
		VENDOR_TOKEN_ID9 "0x55667788"
		...
	}
}
```

要定义相同类型的多个 vendor tuples，请在类型字符串 (“string”，“uuid”，“bool”，“byte”，“short” 或 “word”) 后附加一些字符，以避免 SectionVendorTuples 区域中的重复ID。

解析器将检查 ID 中的前几个字符以获得元组类型。下面是一个例子：
```
SectionVendorTuples."id of the vendor tuples" {
   ...
	tuples."word.module0" {
		VENDOR_TOKEN_PARAM_ID1 "0x00112233"
		VENDOR_TOKEN_PARAM_ID2 "0x44556677"
		...
	}

	tuples."word.module2" {
		VENDOR_TOKEN_PARAM_ID1 "0x11223344"
		VENDOR_TOKEN_PARAM_ID2 "0x55667788"
		...
	}
	...
}
```

Vendor token 和 vendor tuples 不只用作定义 control 的私有数据的方式，其它许多元素也用它们来定义私有数据。Vendor token 和 vendor tuples 也可以像下面这样定义（用 alsatplg 反编译某个拓扑二进制文件获得）：
```
SectionWidget {
	'pipeline.1' {
		type 'scheduler'
		stream_name 'dai.SSP.0.playback'
		no_pm 1
		data 'pipeline.1:tuple0'
	}
	'host-copier-simple.0.playback' {
		type 'aif_in'
		stream_name 'Compr Playback'
		no_pm 1
		data 'host-copier-simple.0.playback:tuple0'
	}
}
 . . . . . .
SectionVendorTokens {
	'pipeline.1' {
		token210 210
		token207 207
		token206 206
		token201 201
		token205 205
		token417 417
		token402 402
	}
	'host-copier-simple.0.playback' {
		token409 409
		token416 416
		token415 415
		token412 412
		token156 156
		token417 417
		token405 405
		token402 402
		token1970 1970
		token1908 1908
		token1907 1907
		token1906 1906
		token1905 1905
		token1904 1904
		token1903 1903
		token1900 1900
		token1901 1901
		token1902 1902
		token1938 1938
		token1971 1971
		token1937 1937
		token1936 1936
		token1935 1935
		token1934 1934
		token1933 1933
		token1930 1930
		token1931 1931
		token1932 1932
	}
}
SectionVendorTuples {
	'pipeline.1:tuple0' {
		tokens 'pipeline.1'
		tuples {
			0_word {
				token210 0
				token207 0
				token206 0
				token201 0
				token205 1
			}
			1_bool.token417 1
			2_string.token402 's16le'
		}
	}
	'host-copier-simple.0.playback:tuple0' {
		tokens 'host-copier-simple.0.playback'
		tuples {
			0_word {
				token409 1
				token416 1
				token415 3
				token412 1
			}
			1_word.token156 0
			2_bool.token417 1
			3_uuid.token405 '1f:1e:4d:02:48:cc:4e:05:ac:33:d1:84:09:69:a7:55'
			4_string.token402 's16le'
			5_word {
				token1970 192
				token1908 1
				token1907 69634
				token1906 0
				token1905 1
				token1904 0xffffff10
				token1903 2
				token1900 48000
				token1901 16
				token1902 16
			}
			6_word {
				token1970 384
				token1908 1
				token1907 71682
				token1906 0
				token1905 1
				token1904 0xffffff10
				token1903 2
				token1900 48000
				token1901 32
				token1902 24
			}
			7_word {
				token1970 384
				token1908 1
				token1907 73730
				token1906 0
				token1905 1
				token1904 0xffffff10
				token1903 2
				token1900 48000
				token1901 32
				token1902 32
			}
			8_word {
				token1938 1
				token1971 384
				token1937 73730
				token1936 0
				token1935 1
				token1934 0xffffff10
				token1933 2
				token1930 48000
				token1931 32
				token1932 32
			}
		}
	}
}
```

某个类型的对象有多个，定义这些对象时，语法也可以为类型名后跟大括号 ({})，大括号内部为各个具体对象名后跟大括号 ({}) 及其各个字段的值。如上面的 `SectionVendorTokens` 和 `SectionVendorTuples`。

上面这段拓扑配置定义中，Widget 对象的 **data** 字段引用 **SectionData** 区的元素，**SectionData** 区的元素描述数据的类型和值，如果数据类型为 bytes、shorts 或 words，直接给出值，否则给出值的引用，如 tuples。**SectionVendorTuples** 区中 tuples 值包含引用的 tokens id，和具体的 tuples 元素值，tuples 元素值按类型用 tokens 描述。不同的 Section 有如下这样的引用关系：

SectionXXX -> SectionData -> SectionVendorTuples -> SectionVendorTokens。

#### Mixer Controls

Mixer control 被定义为一个新的 section，它可以包含通道映射，TLV 数据，回调操作和私有数据。Mixer section 也包含一些其它的配置选项，如：
```
SectionControlMixer."mixer name" {
	comment "optional comments"

	index "1"			# Index number

	channel."name" {		# Channel maps
	   ....
	}

	ops."ctl" {			# Ops callback functions
	   ....
	}

	max "32"			# Max control value
	invert "0"			# Whether control values are inverted

	tlv "tld_data"			# optional TLV data

	data "pdata for mixer1"		# optional private data
}
```

Section 名称用于定义 mixer 名称。index 号可用于标识拓扑对象组。这允许驱动程序对索引号为 N 的对象进行操作，并可用于添加/删除对象的流水线，而其它对象不受影响。

#### Byte Controls

Byte control 被定义为一个新的 section，它可以包含通道映射，TLV 数据，回调操作和私有数据。Byte section 还包含一些其它的配置选项，如下所示：

```
SectionControlBytes."name" {
	comment "optional comments"

	index "1"			# Index number

	channel."name" {		# Channel maps
	   ....
	}

	ops."ctl" {			# Ops callback functions
	   ....
	}

	base "0"			# Register base
	num_regs "16"			# Number of registers
	mask "0xff"			# Mask
	max "255"			# Maximum value

	tlv "tld_data"			# optional TLV data

	data "pdata for mixer1"		# optional private data
}
```

内核代码中，实际没有处理 byte control 的 TLV 数据的逻辑。

#### Enumerated Controls

Enumerated control 被定义为一个新的 section (与 mixer 和 byte 一样)，它可以包含通道映射，回调操作，私有数据和表示枚举的 control 选项的文本字符串。

Enumerated control 的文本字符串在单独的 section 中定义如下：
```
SectionText."name" {

		Values [
			"value1"
			"value2"
			"value3"
		]
}
```

所有枚举的文本值都列在 Values 列表中。

Enumerated control 与其它 controls 类似，定义如下：
```
SectionControlMixer."name" {
	comment "optional comments"

	index "1"			# Index number

	texts "EQU1"			# Enumerated text items

	channel."name" {		# Channel maps
	   ....
	}

	ops."ctl" {			# Ops callback functions
	   ....
	}

	data "pdata for mixer1"		# optional private data
}
```

### DAPM 图

可以使用拓扑文件轻松定义 DAPM 图。该格式与 DAPM 图内核格式非常相似：
```
SectionGraph."dsp" {
	index "1"			# Index number

	lines [
		"sink1, control, source1"
		"sink2, , source2"
	]
}
```

图中的线被定义为 sinks、controls 和 sources 的可变大小列表。由于图的某些线没有关联的 controls，control 名称是可选的。Section 名称可用于将图与其它图区分开来，内核 atm 不使用它。

在 Linux 内核中，DAPM 图由 `soc_tplg_dapm_graph_elems_load()` 函数处理，Linux 内核将会基于 DAPM 图创建 `struct snd_soc_dapm_route` 对象。

### DAPM Widgets

DAPM widgets 类似于 controls，因为它们可以包含许多其它对象。Widgets 可以包含私有数据，mixer controls 和 enum controls。

目前支持以下 widget 类型，并与驱动程序的类型匹配：

 * input
 * output
 * mux
 * mixer
 * pga
 * out_drv
 * adc
 * dac
 * switch
 * pre
 * post
 * aif_in
 * aif_out
 * dai_in
 * dai_out
 * dai_link

Widgets 定义如下：
```
SectionWidget."name" {

	index "1"			# Index number

	type "aif_in"			# Widget type - detailed above
	stream_name "name"		# Stream name

	no_pm "true"			# No PM control bit.
	reg "20"			# PM bit register offset
	shift "0"			# PM bit register shift
	invert "1"			# PM bit is inverted
	subseq "8"			# subsequence number

	event_type "1"			# DAPM widget event type
	event_flags "1"			# DAPM widget event flags

	mixer "name"			# Optional Mixer Control
	enum "name"			# Optional Enum Control

	data "name"			# optional private data
}
```

Section 名称是 widget 名称。mixer 和 enum 字段是互斥的，用于将 controls 包含到 widget 中。Widgets 的 index 和数据字段与 controls 的相同，而其它字段则非常接近于驱动程序的 widget 字段。

#### Widget 私有数据

Widget 可以有私有数据。关于私有数据的格式，请参考 Control 私有数据一节。

在 Linux 内核中，DAPM widget 由 `soc_tplg_dapm_widget_elems_load()` 和 `soc_tplg_dapm_widget_create()` 等函数处理，Linux 内核将会基于 DAPM widget 创建 `struct snd_soc_dapm_widget` 对象。

### PCM Capabilities

拓扑也可以定义前端或物理 DAI 的 PCM Capabilities。Capabilities 可以通过如下 section 定义：
```
SectionPCMCapabilities."name" {

	formats "S24_LE,S16_LE"		# Supported formats
	rates "48000"			# Supported rates
	rate_min "48000"		# Max supported sample rate
	rate_max "48000"		# Min supported sample rate
	channels_min "2"		# Min number of channels
	channels_max "2"		# max number of channels
}
```

支持的格式使用与驱动程序宏相同的命名约定。PCM capabilities 名称可以被 PCM 和物理 DAI sections 引用和包含。

对于编译之后的二进制拓扑文件，PCM capabilities 包含在 PCM 或物理 DAI 对象中，具体如下 (位于 *linux_kernel/include/uapi/sound/asoc.h*)：
```
/*
 * Stream Capabilities
 */
struct snd_soc_tplg_stream_caps {
	__le32 size;		/* in bytes of this structure */
	char name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	__le64 formats;	/* supported formats SNDRV_PCM_FMTBIT_* */
	__le32 rates;		/* supported rates SNDRV_PCM_RATE_* */
	__le32 rate_min;	/* min rate */
	__le32 rate_max;	/* max rate */
	__le32 channels_min;	/* min channels */
	__le32 channels_max;	/* max channels */
	__le32 periods_min;	/* min number of periods */
	__le32 periods_max;	/* max number of periods */
	__le32 period_size_min;	/* min period size bytes */
	__le32 period_size_max;	/* max period size bytes */
	__le32 buffer_size_min;	/* min buffer size bytes */
	__le32 buffer_size_max;	/* max buffer size bytes */
	__le32 sig_bits;        /* number of bits of content */
} __attribute__((packed));
 . . . . . .
/*
 * Describes SW/FW specific features of PCM (FE DAI & DAI link).
 *
 * File block representation for PCM :-
 * +-----------------------------------+-----+
 * | struct snd_soc_tplg_hdr           |  1  |
 * +-----------------------------------+-----+
 * | struct snd_soc_tplg_pcm           |  N  |
 * +-----------------------------------+-----+
 */
struct snd_soc_tplg_pcm {
	__le32 size;		/* in bytes of this structure */
	char pcm_name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	char dai_name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	__le32 pcm_id;		/* unique ID - used to match with DAI link */
	__le32 dai_id;		/* unique ID - used to match */
	__le32 playback;	/* supports playback mode */
	__le32 capture;		/* supports capture mode */
	__le32 compress;	/* 1 = compressed; 0 = PCM */
	struct snd_soc_tplg_stream stream[SND_SOC_TPLG_STREAM_CONFIG_MAX]; /* for DAI link */
	__le32 num_streams;	/* number of streams */
	struct snd_soc_tplg_stream_caps caps[2]; /* playback and capture for DAI */
	__le32 flag_mask;       /* bitmask of flags to configure */
	__le32 flags;           /* SND_SOC_TPLG_LNK_FLGBIT_* flag value */
	struct snd_soc_tplg_private priv;
} __attribute__((packed));
 . . . . . .
struct snd_soc_tplg_dai {
	__le32 size;            /* in bytes of this structure */
	char dai_name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN]; /* name - used to match */
	__le32 dai_id;          /* unique ID - used to match */
	__le32 playback;        /* supports playback mode */
	__le32 capture;         /* supports capture mode */
	struct snd_soc_tplg_stream_caps caps[2]; /* playback and capture for DAI */
	__le32 flag_mask;       /* bitmask of flags to configure */
	__le32 flags;           /* SND_SOC_TPLG_DAI_FLGBIT_* */
	struct snd_soc_tplg_private priv;
} __attribute__((packed));
 . . . . . .
/* Stream Capabilities v4 */
struct snd_soc_tplg_stream_caps_v4 {
	__le32 size;		/* in bytes of this structure */
	char name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	__le64 formats;	/* supported formats SNDRV_PCM_FMTBIT_* */
	__le32 rates;		/* supported rates SNDRV_PCM_RATE_* */
	__le32 rate_min;	/* min rate */
	__le32 rate_max;	/* max rate */
	__le32 channels_min;	/* min channels */
	__le32 channels_max;	/* max channels */
	__le32 periods_min;	/* min number of periods */
	__le32 periods_max;	/* max number of periods */
	__le32 period_size_min;	/* min period size bytes */
	__le32 period_size_max;	/* max period size bytes */
	__le32 buffer_size_min;	/* min buffer size bytes */
	__le32 buffer_size_max;	/* max buffer size bytes */
} __packed;

/* PCM v4 */
struct snd_soc_tplg_pcm_v4 {
	__le32 size;		/* in bytes of this structure */
	char pcm_name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	char dai_name[SNDRV_CTL_ELEM_ID_NAME_MAXLEN];
	__le32 pcm_id;		/* unique ID - used to match with DAI link */
	__le32 dai_id;		/* unique ID - used to match */
	__le32 playback;	/* supports playback mode */
	__le32 capture;		/* supports capture mode */
	__le32 compress;	/* 1 = compressed; 0 = PCM */
	struct snd_soc_tplg_stream stream[SND_SOC_TPLG_STREAM_CONFIG_MAX]; /* for DAI link */
	__le32 num_streams;	/* number of streams */
	struct snd_soc_tplg_stream_caps_v4 caps[2]; /* playback and capture for DAI */
} __packed;
```

在 Linux 内核中，PCM 的 PCM capabilities 与 PCM 对象一同处理，由 `soc_tplg_pcm_elems_load()`、`soc_tplg_pcm_create()`、`soc_tplg_dai_create()` 和 `set_stream_info()`等函数处理。

### PCM 配置

可以通过如下的 section 为播放和录制流方向定义 PCM 运行时配置：
```
SectionPCMConfig."name" {

	config."playback" {		# playback config
		format "S16_LE"		# playback format
		rate "48000"		# playback sample rate
		channels "2"		# playback channels
		tdm_slot "0xf"		# playback TDM slot
	}

	config."capture" {		# capture config
		format "S16_LE"		# capture format
		rate "48000"		# capture sample rate
		channels "2"		# capture channels
		tdm_slot "0xf"		# capture TDM slot
	}
}
```

支持的格式使用与驱动程序宏相同的命名约定。PCM 配置名称可以被 PCM 和物理链接 sections 引用和包含。

### PCM（前端 DAI & DAI 链接）

PCM sections 为支持的播放和录制流定义了支持的 capabilities 和配置，为 DAI & DAI 链接定义了名称和标记。拓扑内核驱动程序将使用 PCM 对象创建 FE DAI & DAI 链接对。

```
SectionPCM."name" {

	index "1"			# Index number

	id "0"				# used for binding to the PCM

	dai."name of front-end DAI" {
		id "0"		# used for binding to the front-end DAI
	}

	pcm."playback" {
		capabilities "capabilities1"	# capabilities for playback

		configs [		# supported configs for playback
			"config1"
			"config2"
		]
	}

	pcm."capture" {
		capabilities "capabilities2"	# capabilities for capture

		configs [		# supported configs for capture
			"config1"
			"config2"
			"config3"
		]
	}

	# Optional boolean flags
	symmetric_rates			"true"
	symmetric_channels		"true"
	symmetric_sample_bits		"false"

	data "name"			# optional private data
}
```

### 物理 DAI 链接配置

物理 DAI 链接的运行时配置可以通过 SectionLink 定义。

后端 DAI 链接属于物理链接，可以通过 SectionLink 或 SectionBE 配置，语法相同。但是 SectionBE 已被弃用，因为内部处理实际上是相同的。
```
SectionLink."name" {

	index "1"			# Index number

	id "0"				# used for binding to the link

	stream_name "name"		# used for binding to the link

	hw_configs [	# runtime supported HW configurations, optional
		"config1"
		"config2"
		...
	]

	default_hw_conf_id "1"		# default HW config ID for init

	# Optional boolean flags
	symmetric_rates			"true"
	symmetric_channels		"false"
	symmetric_sample_bits		"true"

	data "name"			# optional private data
}
```

物理链接可以引用多个运行时支持的硬件配置，它们通过 SectionHWConfig 定义。
```
SectionHWConfig."name" {

	id "1"				# used for binding to the config
	format "I2S"			# physical audio format.
	bclk   "codec_provider"		# Codec provides the bit clock
	fsync  "codec_consumer"		# Codec follows the fsync
}
```

### 物理 DAI

物理 DAI（比如 DPCM 的后端 DAI）被定义为一个新的 section，它可以包含唯一的 ID，播放和录制流的 capabilities，可选的标记，和私有数据。

它的 PCM 流 capablities 与 PCM 对象的那些相同，请参考 **PCM Capabilities** 一节。
```
SectionDAI."name" {

	index "1"			# Index number

	id "0"				# used for binding to the Backend DAI

	pcm."playback" {
		capabilities "capabilities1"	# capabilities for playback
	}

	pcm."capture" {
		capabilities "capabilities2"	# capabilities for capture
	}

	symmetric_rates "true"			# optional flags
	symmetric_channels "true"
	symmetric_sample_bits "false"

	data "name"			# optional private data
}
```

### Manifest 私有数据

Manifest 可能有私有数据。用户需要定义一个 manifest section，并为它定义指向 1 个或多个数据 sections 引用。请参考 **如何定义具有私有数据的元素** 一节。

且文本配置文件最多可以有 1 个 manifest section。

Manifest section 定义如下：
```
SectionManifest"name" {

	data "name"			# optional private data
}
```

具有私有数据的 manifest 的实际例子如下：
```
SectionManifest.manifest.data.0 'manifest:data0'
 . . . . . .
SectionData {
	'manifest:data0'.bytes '03:00:1d:00:00:00'
	'pipeline.1:tuple0'.tuples 'pipeline.1:tuple0'
	'host-copier-simple.0.playback:tuple0'.tuples 'host-copier-simple.0.playback:tuple0'
}
```

### 包含其它文件

用户可能通过 alsaconf 语法 <path/to/configuration-file> 在文本配置文件中包含其它文件。这允许用户在单独的文件中定义公共的信息（比如 vendor tokens，tuples），并为不同的平台共享它们，这样就节省了配置文件的总大小。

用户还可以通过 alsaconf 语法 <searchfdir:/relative-path/to/usr/share/alsa> 指定相对于 “/usr/share/alsa/” 的其它配置目录来搜索包含的文件。

比如，文件 A 和文件 B 是平台 X 的两个文本配置文件，它们将被安装到 /usr/share/alsa/topology/platformx。如果我们需要文件 A 包含文件 B，则在文件 A 中，我们可以添加：
```
<searchdir:topology/platformx>

<name-of-file-B>
```

ALSA conf 将以下面的优先级顺序搜索并打开包含的文件：

1. 直接按文件名打开文件；
2. 在 “/usr/share/alsa” 中搜索文件名；
3. 在用户指定的 “/usr/share/alsa” 下的子目录中搜索文件名。

所包含文件的顺序不必与其依赖项相同，因为拓扑库将在解析其依赖项之前加载所有文件。

文件中定义的配置目录只用于搜索该文件包含的文件。

**参考文档：**

[ALSA Topology Interface](https://vovkos.github.io/doxyrest/samples/alsa/page_topology.html)
