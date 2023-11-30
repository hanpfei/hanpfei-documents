---
title: 深入浅出 Linux ALSA Compress-Offload 驱动核心
date: 2023-11-26 21:33:29
categories: Linux 内核
tags:
- Linux 内核
---

ALSA Compress-Offload API 是 Linux ALSA 为 ADSP 之类的带有音频数据编解码能力的设备设计的 API，关于 Linux ALSA Compress-Offload API 的更多详细信息，可以参考 [ALSA Compress-Offload API](https://www.cnblogs.com/wolfcs/p/17825631.html)。本文分析 Linux 内核音频 ALSA 框架 Compress-Offload 设备驱动核心的实现。

## 数据结构

Linux 内核音频 ALSA 框架用 `struct snd_compr` 对象表示处理压缩的音频数据的设备，用 `struct snd_compr_ops` 对象描述处理压缩的音频数据的设备支持的操作，这两个结构体定义 (位于 *include/sound/compress_driver.h*) 如下：
```
/**
 * struct snd_compr_ops: compressed path DSP operations
 * @open: Open the compressed stream
 * This callback is mandatory and shall keep dsp ready to receive the stream
 * parameter
 * @free: Close the compressed stream, mandatory
 * @set_params: Sets the compressed stream parameters, mandatory
 * This can be called in during stream creation only to set codec params
 * and the stream properties
 * @get_params: retrieve the codec parameters, mandatory
 * @set_metadata: Set the metadata values for a stream
 * @get_metadata: retrieves the requested metadata values from stream
 * @trigger: Trigger operations like start, pause, resume, drain, stop.
 * This callback is mandatory
 * @pointer: Retrieve current h/w pointer information. Mandatory
 * @copy: Copy the compressed data to/from userspace, Optional
 * Can't be implemented if DSP supports mmap
 * @mmap: DSP mmap method to mmap DSP memory
 * @ack: Ack for DSP when data is written to audio buffer, Optional
 * Not valid if copy is implemented
 * @get_caps: Retrieve DSP capabilities, mandatory
 * @get_codec_caps: Retrieve capabilities for a specific codec, mandatory
 */
struct snd_compr_ops {
	int (*open)(struct snd_compr_stream *stream);
	int (*free)(struct snd_compr_stream *stream);
	int (*set_params)(struct snd_compr_stream *stream,
			struct snd_compr_params *params);
	int (*get_params)(struct snd_compr_stream *stream,
			struct snd_codec *params);
	int (*set_metadata)(struct snd_compr_stream *stream,
			struct snd_compr_metadata *metadata);
	int (*get_metadata)(struct snd_compr_stream *stream,
			struct snd_compr_metadata *metadata);
	int (*trigger)(struct snd_compr_stream *stream, int cmd);
	int (*pointer)(struct snd_compr_stream *stream,
			struct snd_compr_tstamp *tstamp);
	int (*copy)(struct snd_compr_stream *stream, char __user *buf,
		       size_t count);
	int (*mmap)(struct snd_compr_stream *stream,
			struct vm_area_struct *vma);
	int (*ack)(struct snd_compr_stream *stream, size_t bytes);
	int (*get_caps) (struct snd_compr_stream *stream,
			struct snd_compr_caps *caps);
	int (*get_codec_caps) (struct snd_compr_stream *stream,
			struct snd_compr_codec_caps *codec);
};

/**
 * struct snd_compr: Compressed device
 * @name: DSP device name
 * @dev: associated device instance
 * @ops: pointer to DSP callbacks
 * @private_data: pointer to DSP pvt data
 * @card: sound card pointer
 * @direction: Playback or capture direction
 * @lock: device lock
 * @device: device id
 */
struct snd_compr {
	const char *name;
	struct device dev;
	struct snd_compr_ops *ops;
	void *private_data;
	struct snd_card *card;
	unsigned int direction;
	struct mutex lock;
	int device;
#ifdef CONFIG_SND_VERBOSE_PROCFS
	/* private: */
	char id[64];
	struct snd_info_entry *proc_root;
	struct snd_info_entry *proc_info_entry;
#endif
};
```

`struct snd_compr_ops` 结构体包含的操作如下：

 - **open**：打开压缩的音频数据流，驱动程序必须提供，这个操作应该确保 DSP 硬件设备为接收压缩的音频流参数做好准备。
 - **free**：关闭压缩的音频数据流，驱动程序必须提供。
 - **set_params**：设置压缩的音频数据流的参数，驱动程序必须提供。这个操作可以在音频流创建期间调用，设置编解码参数和音频流属性。
 - **get_params**：获取音频流的编解码参数，驱动程序必须提供。
 - **set_metadata**：为音频流设置 metadata 值。
 - **get_metadata**：从音频流获取请求的 metadata 值。
 - **trigger**：诸如 **start**、**pause**、**resume**、**drain** 和 **stop** 这样的 trigger 操作，驱动程序必须提供。
 - **pointer**：获取音频流当前的 h/w 指针信息，驱动程序必须提供。
 - **copy**：在用户空间和内核驱动程序之间拷贝压缩的音频数据，可选的，如果 DSP 支持 mmap 不能实现它。
 - **mmap**：mmap DSP 内存的 DSP mmap 方法。
 - **ack**：当音频数据被写入音频缓冲区时为 DSP 做确认，可选的。如果实现了 **copy** 操作，则它不生效。
 - **get_caps**：获取 DSP 能力，驱动程序必须提供。
 - **get_codec_caps**：获取特定编解码器的能力，驱动程序必须提供。

`struct snd_compr` 和 `struct snd_compr_ops` 对象由处理压缩的音频数据的设备的 Linux 内核驱动程序创建，并随着它所属的 `struct snd_card` 的注册一起注册进 Compress-Offload 设备驱动核心。

`struct snd_compr_stream` 对象表示打开的压缩音频流，`struct snd_compr_runtime` 对象表示运行时压缩音频流的描述，这两个结构体定义 (位于 *include/sound/compress_driver.h*) 如下：
```
/**
 * struct snd_compr_runtime: runtime stream description
 * @state: stream state
 * @ops: pointer to DSP callbacks
 * @buffer: pointer to kernel buffer, valid only when not in mmap mode or
 *	DSP doesn't implement copy
 * @buffer_size: size of the above buffer
 * @fragment_size: size of buffer fragment in bytes
 * @fragments: number of such fragments
 * @total_bytes_available: cumulative number of bytes made available in
 *	the ring buffer
 * @total_bytes_transferred: cumulative bytes transferred by offload DSP
 * @sleep: poll sleep
 * @private_data: driver private data pointer
 * @dma_area: virtual buffer address
 * @dma_addr: physical buffer address (not accessible from main CPU)
 * @dma_bytes: size of DMA area
 * @dma_buffer_p: runtime dma buffer pointer
 */
struct snd_compr_runtime {
	snd_pcm_state_t state;
	struct snd_compr_ops *ops;
	void *buffer;
	u64 buffer_size;
	u32 fragment_size;
	u32 fragments;
	u64 total_bytes_available;
	u64 total_bytes_transferred;
	wait_queue_head_t sleep;
	void *private_data;

	unsigned char *dma_area;
	dma_addr_t dma_addr;
	size_t dma_bytes;
	struct snd_dma_buffer *dma_buffer_p;
};

/**
 * struct snd_compr_stream: compressed stream
 * @name: device name
 * @ops: pointer to DSP callbacks
 * @runtime: pointer to runtime structure
 * @device: device pointer
 * @error_work: delayed work used when closing the stream due to an error
 * @direction: stream direction, playback/recording
 * @metadata_set: metadata set flag, true when set
 * @next_track: has userspace signal next track transition, true when set
 * @partial_drain: undergoing partial_drain for stream, true when set
 * @private_data: pointer to DSP private data
 * @dma_buffer: allocated buffer if any
 */
struct snd_compr_stream {
	const char *name;
	struct snd_compr_ops *ops;
	struct snd_compr_runtime *runtime;
	struct snd_compr *device;
	struct delayed_work error_work;
	enum snd_compr_direction direction;
	bool metadata_set;
	bool next_track;
	bool partial_drain;
	void *private_data;
	struct snd_dma_buffer dma_buffer;
};
```

`struct snd_compr_stream` 和 `struct snd_compr_runtime` 对象在 compress 音频设备文件打开时创建，它们的对象生命周期从文件打开到文件关闭。`struct snd_compr` 和 `struct snd_compr` 对象的生命周期一般从 compress 设备驱动程序加载到驱动程序卸载。`struct snd_compr` 和 `struct snd_compr` 对象一般为静态对象，`struct snd_compr_stream` 和 `struct snd_compr_runtime` 对象则为运行时动态对象。

用户空间程序通过一些结构与内核 ALSA 框架的 Compress 功能块交换数据，`struct snd_compr_params` 对象表示压缩的音频流的参数，`struct snd_compr_metadata` 对象表示压缩的音频流的 metadata，`struct snd_compr_tstamp` 对象表示时间戳描述符，`struct snd_compr_caps` 对象表示音频流能力描述符，`struct snd_compr_codec_caps` 对象表示音频流 codec 能力描述符，这些结构体定义 (位于 *include/uapi/sound/compress_offload.h*) 如下：
```
/**
 * struct snd_compressed_buffer - compressed buffer
 * @fragment_size: size of buffer fragment in bytes
 * @fragments: number of such fragments
 */
struct snd_compressed_buffer {
	__u32 fragment_size;
	__u32 fragments;
} __attribute__((packed, aligned(4)));

/**
 * struct snd_compr_params - compressed stream params
 * @buffer: buffer description
 * @codec: codec parameters
 * @no_wake_mode: dont wake on fragment elapsed
 */
struct snd_compr_params {
	struct snd_compressed_buffer buffer;
	struct snd_codec codec;
	__u8 no_wake_mode;
} __attribute__((packed, aligned(4)));

/**
 * struct snd_compr_tstamp - timestamp descriptor
 * @byte_offset: Byte offset in ring buffer to DSP
 * @copied_total: Total number of bytes copied from/to ring buffer to/by DSP
 * @pcm_frames: Frames decoded or encoded by DSP. This field will evolve by
 *	large steps and should only be used to monitor encoding/decoding
 *	progress. It shall not be used for timing estimates.
 * @pcm_io_frames: Frames rendered or received by DSP into a mixer or an audio
 * output/input. This field should be used for A/V sync or time estimates.
 * @sampling_rate: sampling rate of audio
 */
struct snd_compr_tstamp {
	__u32 byte_offset;
	__u32 copied_total;
	__u32 pcm_frames;
	__u32 pcm_io_frames;
	__u32 sampling_rate;
} __attribute__((packed, aligned(4)));

/**
 * struct snd_compr_avail - avail descriptor
 * @avail: Number of bytes available in ring buffer for writing/reading
 * @tstamp: timestamp information
 */
struct snd_compr_avail {
	__u64 avail;
	struct snd_compr_tstamp tstamp;
} __attribute__((packed, aligned(4)));

enum snd_compr_direction {
	SND_COMPRESS_PLAYBACK = 0,
	SND_COMPRESS_CAPTURE
};

/**
 * struct snd_compr_caps - caps descriptor
 * @codecs: pointer to array of codecs
 * @direction: direction supported. Of type snd_compr_direction
 * @min_fragment_size: minimum fragment supported by DSP
 * @max_fragment_size: maximum fragment supported by DSP
 * @min_fragments: min fragments supported by DSP
 * @max_fragments: max fragments supported by DSP
 * @num_codecs: number of codecs supported
 * @reserved: reserved field
 */
struct snd_compr_caps {
	__u32 num_codecs;
	__u32 direction;
	__u32 min_fragment_size;
	__u32 max_fragment_size;
	__u32 min_fragments;
	__u32 max_fragments;
	__u32 codecs[MAX_NUM_CODECS];
	__u32 reserved[11];
} __attribute__((packed, aligned(4)));

/**
 * struct snd_compr_codec_caps - query capability of codec
 * @codec: codec for which capability is queried
 * @num_descriptors: number of codec descriptors
 * @descriptor: array of codec capability descriptor
 */
struct snd_compr_codec_caps {
	__u32 codec;
	__u32 num_descriptors;
	struct snd_codec_desc descriptor[MAX_NUM_CODEC_DESCRIPTORS];
} __attribute__((packed, aligned(4)));

/**
 * enum sndrv_compress_encoder
 * @SNDRV_COMPRESS_ENCODER_PADDING: no of samples appended by the encoder at the
 * end of the track
 * @SNDRV_COMPRESS_ENCODER_DELAY: no of samples inserted by the encoder at the
 * beginning of the track
 */
enum sndrv_compress_encoder {
	SNDRV_COMPRESS_ENCODER_PADDING = 1,
	SNDRV_COMPRESS_ENCODER_DELAY = 2,
};

/**
 * struct snd_compr_metadata - compressed stream metadata
 * @key: key id
 * @value: key value
 */
struct snd_compr_metadata {
	 __u32 key;
	 __u32 value[8];
} __attribute__((packed, aligned(4)));
```

`struct snd_codec` 对象表示 codec 信息，`struct snd_codec_desc` 对象为 codec 能力描述，这个结构体定义 (位于 *include/uapi/sound/compress_params.h*) 如下：
```
* Encoder options */

struct snd_enc_wma {
	__u32 super_block_align; /* WMA Type-specific data */
};


/**
 * struct snd_enc_vorbis
 * @quality: Sets encoding quality to n, between -1 (low) and 10 (high).
 * In the default mode of operation, the quality level is 3.
 * Normal quality range is 0 - 10.
 * @managed: Boolean. Set  bitrate  management  mode. This turns off the
 * normal VBR encoding, but allows hard or soft bitrate constraints to be
 * enforced by the encoder. This mode can be slower, and may also be
 * lower quality. It is primarily useful for streaming.
 * @max_bit_rate: Enabled only if managed is TRUE
 * @min_bit_rate: Enabled only if managed is TRUE
 * @downmix: Boolean. Downmix input from stereo to mono (has no effect on
 * non-stereo streams). Useful for lower-bitrate encoding.
 *
 * These options were extracted from the OpenMAX IL spec and Gstreamer vorbisenc
 * properties
 *
 * For best quality users should specify VBR mode and set quality levels.
 */

struct snd_enc_vorbis {
	__s32 quality;
	__u32 managed;
	__u32 max_bit_rate;
	__u32 min_bit_rate;
	__u32 downmix;
} __attribute__((packed, aligned(4)));


/**
 * struct snd_enc_real
 * @quant_bits: number of coupling quantization bits in the stream
 * @start_region: coupling start region in the stream
 * @num_regions: number of regions value
 *
 * These options were extracted from the OpenMAX IL spec
 */

struct snd_enc_real {
	__u32 quant_bits;
	__u32 start_region;
	__u32 num_regions;
} __attribute__((packed, aligned(4)));

/**
 * struct snd_enc_flac
 * @num: serial number, valid only for OGG formats
 *	needs to be set by application
 * @gain: Add replay gain tags
 *
 * These options were extracted from the FLAC online documentation
 * at http://flac.sourceforge.net/documentation_tools_flac.html
 *
 * To make the API simpler, it is assumed that the user will select quality
 * profiles. Additional options that affect encoding quality and speed can
 * be added at a later stage if needed.
 *
 * By default the Subset format is used by encoders.
 *
 * TAGS such as pictures, etc, cannot be handled by an offloaded encoder and are
 * not supported in this API.
 */

struct snd_enc_flac {
	__u32 num;
	__u32 gain;
} __attribute__((packed, aligned(4)));

struct snd_enc_generic {
	__u32 bw;	/* encoder bandwidth */
	__s32 reserved[15];	/* Can be used for SND_AUDIOCODEC_BESPOKE */
} __attribute__((packed, aligned(4)));

struct snd_dec_flac {
	__u16 sample_size;
	__u16 min_blk_size;
	__u16 max_blk_size;
	__u16 min_frame_size;
	__u16 max_frame_size;
	__u16 reserved;
} __attribute__((packed, aligned(4)));

struct snd_dec_wma {
	__u32 encoder_option;
	__u32 adv_encoder_option;
	__u32 adv_encoder_option2;
	__u32 reserved;
} __attribute__((packed, aligned(4)));

struct snd_dec_alac {
	__u32 frame_length;
	__u8 compatible_version;
	__u8 pb;
	__u8 mb;
	__u8 kb;
	__u32 max_run;
	__u32 max_frame_bytes;
} __attribute__((packed, aligned(4)));

struct snd_dec_ape {
	__u16 compatible_version;
	__u16 compression_level;
	__u32 format_flags;
	__u32 blocks_per_frame;
	__u32 final_frame_blocks;
	__u32 total_frames;
	__u32 seek_table_present;
} __attribute__((packed, aligned(4)));

union snd_codec_options {
	struct snd_enc_wma wma;
	struct snd_enc_vorbis vorbis;
	struct snd_enc_real real;
	struct snd_enc_flac flac;
	struct snd_enc_generic generic;
	struct snd_dec_flac flac_d;
	struct snd_dec_wma wma_d;
	struct snd_dec_alac alac_d;
	struct snd_dec_ape ape_d;
} __attribute__((packed, aligned(4)));

/** struct snd_codec_desc - description of codec capabilities
 * @max_ch: Maximum number of audio channels
 * @sample_rates: Sampling rates in Hz, use values like 48000 for this
 * @num_sample_rates: Number of valid values in sample_rates array
 * @bit_rate: Indexed array containing supported bit rates
 * @num_bitrates: Number of valid values in bit_rate array
 * @rate_control: value is specified by SND_RATECONTROLMODE defines.
 * @profiles: Supported profiles. See SND_AUDIOPROFILE defines.
 * @modes: Supported modes. See SND_AUDIOMODE defines
 * @formats: Supported formats. See SND_AUDIOSTREAMFORMAT defines
 * @min_buffer: Minimum buffer size handled by codec implementation
 * @reserved: reserved for future use
 *
 * This structure provides a scalar value for profiles, modes and stream
 * format fields.
 * If an implementation supports multiple combinations, they will be listed as
 * codecs with different descriptors, for example there would be 2 descriptors
 * for AAC-RAW and AAC-ADTS.
 * This entails some redundancy but makes it easier to avoid invalid
 * configurations.
 *
 */

struct snd_codec_desc {
	__u32 max_ch;
	__u32 sample_rates[MAX_NUM_SAMPLE_RATES];
	__u32 num_sample_rates;
	__u32 bit_rate[MAX_NUM_BITRATES];
	__u32 num_bitrates;
	__u32 rate_control;
	__u32 profiles;
	__u32 modes;
	__u32 formats;
	__u32 min_buffer;
	__u32 reserved[15];
} __attribute__((packed, aligned(4)));

/** struct snd_codec
 * @id: Identifies the supported audio encoder/decoder.
 *		See SND_AUDIOCODEC macros.
 * @ch_in: Number of input audio channels
 * @ch_out: Number of output channels. In case of contradiction between
 *		this field and the channelMode field, the channelMode field
 *		overrides.
 * @sample_rate: Audio sample rate of input data in Hz, use values like 48000
 *		for this.
 * @bit_rate: Bitrate of encoded data. May be ignored by decoders
 * @rate_control: Encoding rate control. See SND_RATECONTROLMODE defines.
 *               Encoders may rely on profiles for quality levels.
 *		 May be ignored by decoders.
 * @profile: Mandatory for encoders, can be mandatory for specific
 *		decoders as well. See SND_AUDIOPROFILE defines.
 * @level: Supported level (Only used by WMA at the moment)
 * @ch_mode: Channel mode for encoder. See SND_AUDIOCHANMODE defines
 * @format: Format of encoded bistream. Mandatory when defined.
 *		See SND_AUDIOSTREAMFORMAT defines.
 * @align: Block alignment in bytes of an audio sample.
 *		Only required for PCM or IEC formats.
 * @options: encoder-specific settings
 * @reserved: reserved for future use
 */

struct snd_codec {
	__u32 id;
	__u32 ch_in;
	__u32 ch_out;
	__u32 sample_rate;
	__u32 bit_rate;
	__u32 rate_control;
	__u32 profile;
	__u32 level;
	__u32 ch_mode;
	__u32 format;
	__u32 align;
	union snd_codec_options options;
	__u32 reserved[3];
} __attribute__((packed, aligned(4)));
```

Linux 内核音频 ALSA 框架 Compress-Offload 设备驱动核心通过这些数据结构实现其各个操作、行为和抽象。

### Compress 设备的创建

驱动程序调用 `snd_compress_new()` 函数为 sound card 创建 compress 设备，这个函数定义 (位于 *sound/core/compress_offload.c*) 如下：
```
#ifdef CONFIG_SND_VERBOSE_PROCFS
static void snd_compress_proc_info_read(struct snd_info_entry *entry,
					struct snd_info_buffer *buffer)
{
	struct snd_compr *compr = (struct snd_compr *)entry->private_data;

	snd_iprintf(buffer, "card: %d\n", compr->card->number);
	snd_iprintf(buffer, "device: %d\n", compr->device);
	snd_iprintf(buffer, "stream: %s\n",
			compr->direction == SND_COMPRESS_PLAYBACK
				? "PLAYBACK" : "CAPTURE");
	snd_iprintf(buffer, "id: %s\n", compr->id);
}

static int snd_compress_proc_init(struct snd_compr *compr)
{
	struct snd_info_entry *entry;
	char name[16];

	sprintf(name, "compr%i", compr->device);
	entry = snd_info_create_card_entry(compr->card, name,
					   compr->card->proc_root);
	if (!entry)
		return -ENOMEM;
	entry->mode = S_IFDIR | 0555;
	compr->proc_root = entry;

	entry = snd_info_create_card_entry(compr->card, "info",
					   compr->proc_root);
	if (entry)
		snd_info_set_text_ops(entry, compr,
				      snd_compress_proc_info_read);
	compr->proc_info_entry = entry;

	return 0;
}

static void snd_compress_proc_done(struct snd_compr *compr)
{
	snd_info_free_entry(compr->proc_info_entry);
	compr->proc_info_entry = NULL;
	snd_info_free_entry(compr->proc_root);
	compr->proc_root = NULL;
}

static inline void snd_compress_set_id(struct snd_compr *compr, const char *id)
{
	strlcpy(compr->id, id, sizeof(compr->id));
}
#else
static inline int snd_compress_proc_init(struct snd_compr *compr)
{
	return 0;
}

static inline void snd_compress_proc_done(struct snd_compr *compr)
{
}

static inline void snd_compress_set_id(struct snd_compr *compr, const char *id)
{
}
#endif
 . . . . . .
int snd_compress_new(struct snd_card *card, int device,
			int dirn, const char *id, struct snd_compr *compr)
{
	static const struct snd_device_ops ops = {
		.dev_free = snd_compress_dev_free,
		.dev_register = snd_compress_dev_register,
		.dev_disconnect = snd_compress_dev_disconnect,
	};
	int ret;

	compr->card = card;
	compr->device = device;
	compr->direction = dirn;

	snd_compress_set_id(compr, id);

	snd_device_initialize(&compr->dev, card);
	dev_set_name(&compr->dev, "comprC%iD%i", card->number, device);

	ret = snd_device_new(card, SNDRV_DEV_COMPRESS, compr, &ops);
	if (ret == 0)
		snd_compress_proc_init(compr);

	return ret;
}
EXPORT_SYMBOL_GPL(snd_compress_new);
```

一般来说，需要 compress 设备驱动程序创建 `struct snd_compr_ops` 对象，为 `struct snd_compr` 对象分配内存，并初始化 `struct snd_compr` 对象的 `ops` 字段指向创建的 `struct snd_compr_ops` 对象等。

在 `snd_compress_new()` 函数中，主要做了这样一些事情：

1. 进一步初始化 `struct snd_compr` 对象；
2. 为 `struct snd_compr` 对象设置 ID，用于创建 procfs 虚拟文件系统的 compress 设备相关文件；
3. 调用 `snd_device_initialize()` 函数初始化 `struct snd_compr` compress 音频设备的 `struct device`；
4. 为 compress 设备设置设备名；
5. 调用 `snd_device_new()` 函数为 compress 设备创建 `SNDRV_DEV_COMPRESS` 类型的音频设备 `struct snd_device` 对象；
6. 为音频 compress 设备初始化 procfs 虚拟文件系统文件。

`snd_device_initialize()` 函数定义 (位于 *sound/core/init.c*) 如下：
```
/* the default release callback set in snd_device_initialize() below;
 * this is just NOP for now, as almost all jobs are already done in
 * dev_free callback of snd_device chain instead.
 */
static void default_release(struct device *dev)
{
}

/**
 * snd_device_initialize - Initialize struct device for sound devices
 * @dev: device to initialize
 * @card: card to assign, optional
 */
void snd_device_initialize(struct device *dev, struct snd_card *card)
{
	device_initialize(dev);
	if (card)
		dev->parent = &card->card_dev;
	dev->class = sound_class;
	dev->release = default_release;
}
EXPORT_SYMBOL_GPL(snd_device_initialize);
```

`snd_device_initialize()` 函数设置 `struct device` 的 `class` 为 `sound_class`，这个设置与 compress 设备文件名的设置共同协助 compress 设备在 devtmpfs 虚拟文件系统中设备文件的创建。

`snd_device_new()` 函数定义 (位于 *sound/core/init.c*) 如下：
```
int snd_device_new(struct snd_card *card, enum snd_device_type type,
		   void *device_data, const struct snd_device_ops *ops)
{
	struct snd_device *dev;
	struct list_head *p;

	if (snd_BUG_ON(!card || !device_data || !ops))
		return -ENXIO;
	dev = kzalloc(sizeof(*dev), GFP_KERNEL);
	if (!dev)
		return -ENOMEM;
	INIT_LIST_HEAD(&dev->list);
	dev->card = card;
	dev->type = type;
	dev->state = SNDRV_DEV_BUILD;
	dev->device_data = device_data;
	dev->ops = ops;

	/* insert the entry in an incrementally sorted list */
	list_for_each_prev(p, &card->devices) {
		struct snd_device *pdev = list_entry(p, struct snd_device, list);
		if ((unsigned int)pdev->type <= (unsigned int)type)
			break;
	}

	list_add(&dev->list, p);
	return 0;
}
EXPORT_SYMBOL(snd_device_new);
```

`snd_device_new()` 函数分配 `struct snd_device` 对象，初始化其各个字段，并将其添加到 sound card 的设备列表中。`struct device` 对象为 Linux 内核设备管理核心的抽象设备表示，`struct snd_device` 对象为 Linux 内核 ALSA 音频子系统的抽象音频设备表示，`struct snd_compr` 对象为 Linux 内核 ALSA 音频子系统的抽象 compress 音频设备表示。从面向对象设计的角度来看，加上通常由 compress 设备驱动程序定义并创建的具体 compress 设备表示，这些设备表示大概有如下的关系：

![Device in Linux Kernel Audio Sbusystem](https://upload-images.jianshu.io/upload_images/1315506-b0500e812c3caec1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在实际实现中，compress 音频设备的 `struct snd_device` 对象的 `device_data` 字段指向 `struct snd_compr` 对象。

Compress 设备的创建过程中，在 compress 设备驱动程序中创建了 `struct snd_compr` 对象，在 ALSA compress 核心中创建了 `struct snd_device` 对象，`struct snd_device` 对象建立了 `struct snd_compr` 对象和 `struct snd_card` 之间的连接。

### Compress 设备的注册

Compress 设备随着它所属的 `struct snd_card` 的注册一起注册。在上面创建 compress 设备的 `snd_compress_new()` 函数中，compress 设备的 `struct snd_device` 的 `struct snd_device_ops` 是这样的：
```
	static const struct snd_device_ops ops = {
		.dev_free = snd_compress_dev_free,
		.dev_register = snd_compress_dev_register,
		.dev_disconnect = snd_compress_dev_disconnect,
	};
```

`struct snd_device_ops` 的 `dev_register` 操作 `snd_compress_dev_register()` 在 `struct snd_card` 注册时被调用，调用过程类似于下面这样：
```
[   20.391776]  snd_compress_dev_register+0x30/0xc0
[   20.398034]  snd_device_register_all+0x4c/0x80
[   20.404098]  snd_card_register+0x4c/0x170
[   20.409621]  snd_soc_bind_card+0x8a8/0x924
[   20.415236]  snd_soc_register_card+0xf4/0x10c
[   20.421211]  devm_snd_soc_register_card+0x44/0xa0
```

Compress 设备的 `struct snd_device_ops` 的这些操作定义 (位于 *sound/core/compress_offload.c*) 如下：
```
static int snd_compress_dev_register(struct snd_device *device)
{
	int ret;
	struct snd_compr *compr;

	if (snd_BUG_ON(!device || !device->device_data))
		return -EBADFD;
	compr = device->device_data;

	pr_debug("reg device %s, direction %d\n", compr->name,
			compr->direction);
	/* register compressed device */
	ret = snd_register_device(SNDRV_DEVICE_TYPE_COMPRESS,
				  compr->card, compr->device,
				  &snd_compr_file_ops, compr, &compr->dev);
	if (ret < 0) {
		pr_err("snd_register_device failed %d\n", ret);
		return ret;
	}
	return ret;

}

static int snd_compress_dev_disconnect(struct snd_device *device)
{
	struct snd_compr *compr;

	compr = device->device_data;
	snd_unregister_device(&compr->dev);
	return 0;
}
 . . . . . .
static int snd_compress_dev_free(struct snd_device *device)
{
	struct snd_compr *compr;

	compr = device->device_data;
	snd_compress_proc_done(compr);
	put_device(&compr->dev);
	return 0;
}
```

`struct snd_device_ops` 的 `dev_register` 操作 `snd_compress_dev_register()` 调用 `snd_register_device()` 函数为声卡注册类型为 `SNDRV_DEVICE_TYPE_COMPRESS` 的设备，这会在 devtmpfs 虚拟文件系统中创建 compress 设备文件。

通常情况下，包含多个设备的声卡的设备驱动程序通过 `devm_snd_soc_register_card()`/`snd_soc_register_card()`/`snd_card_register()` 这样的接口注册声卡。Compress-Offload 驱动核心提供了 `snd_compress_register()`/`snd_compress_deregister()` 这样的接口，注册/取消注册以 compress 设备为核心的声卡，这两个接口实现 (位于 *sound/core/compress_offload.c*) 如下：
```
static int snd_compress_add_device(struct snd_compr *device)
{
	int ret;

	if (!device->card)
		return -EINVAL;

	/* register the card */
	ret = snd_card_register(device->card);
	if (ret)
		goto out;
	return 0;

out:
	pr_err("failed with %d\n", ret);
	return ret;

}

static int snd_compress_remove_device(struct snd_compr *device)
{
	return snd_card_free(device->card);
}

/**
 * snd_compress_register - register compressed device
 *
 * @device: compressed device to register
 */
int snd_compress_register(struct snd_compr *device)
{
	int retval;

	if (device->name == NULL || device->ops == NULL)
		return -EINVAL;

	pr_debug("Registering compressed device %s\n", device->name);
	if (snd_BUG_ON(!device->ops->open))
		return -EINVAL;
	if (snd_BUG_ON(!device->ops->free))
		return -EINVAL;
	if (snd_BUG_ON(!device->ops->set_params))
		return -EINVAL;
	if (snd_BUG_ON(!device->ops->trigger))
		return -EINVAL;

	mutex_init(&device->lock);

	/* register a compressed card */
	mutex_lock(&device_mutex);
	retval = snd_compress_add_device(device);
	mutex_unlock(&device_mutex);
	return retval;
}
EXPORT_SYMBOL_GPL(snd_compress_register);

int snd_compress_deregister(struct snd_compr *device)
{
	pr_debug("Removing compressed device %s\n", device->name);
	mutex_lock(&device_mutex);
	snd_compress_remove_device(device);
	mutex_unlock(&device_mutex);
	return 0;
}
EXPORT_SYMBOL_GPL(snd_compress_deregister);
```

`snd_compress_register()` 接口检查 compress 设备驱动程序必须实现的操作，初始化 compress 设备的锁，并注册声卡。`snd_compress_deregister()` 接口则注销声卡。

### Compress 音频流的状态机及设备文件的文件操作

Compress 音频流状态机如下：
```
                                      +----------+
                                      |          |
                                      |   OPEN   |
                                      |          |
                                      +----------+
                                           |
                                           |
                                           | compr_set_params()
                                           |
                                           v
       compr_free()                  +----------+
+------------------------------------|          |
|                                    |   SETUP  |
|          +-------------------------|          |<-------------------------+
|          |       compr_write()     +----------+                          |
|          |                              ^                                |
|          |                              | compr_drain_notify()           |
|          |                              |        or                      |
|          |                              |     compr_stop()               |
|          |                              |                                |
|          |                         +----------+                          |
|          |                         |          |                          |
|          |                         |   DRAIN  |                          |
|          |                         |          |                          |
|          |                         +----------+                          |
|          |                              ^                                |
|          |                              |                                |
|          |                              | compr_drain()                  |
|          |                              |                                |
|          v                              |                                |
|    +----------+                    +----------+                          |
|    |          |    compr_start()   |          |        compr_stop()      |
|    | PREPARE  |------------------->|  RUNNING |--------------------------+
|    |          |                    |          |                          |
|    +----------+                    +----------+                          |
|          |                            |    ^                             |
|          |compr_free()                |    |                             |
|          |              compr_pause() |    | compr_resume()              |
|          |                            |    |                             |
|          v                            v    |                             |
|    +----------+                   +----------+                           |
|    |          |                   |          |         compr_stop()      |
+--->|   FREE   |                   |  PAUSE   |---------------------------+
     |          |                   |          |
     +----------+                   +----------+
```

在 Linux 内核中，compress 音频流状态机由 compress 设备文件的文件操作实现。`struct snd_device_ops` 的 `dev_register` 操作 `snd_compress_dev_register()` 中可以看到 compress 设备文件的文件操作为 `snd_compr_file_ops`，`snd_compr_file_ops` 定义 (位于 *sound/core/compress_offload.c*) 如下：
```
/* support of 32bit userspace on 64bit platforms */
#ifdef CONFIG_COMPAT
static long snd_compr_ioctl_compat(struct file *file, unsigned int cmd,
						unsigned long arg)
{
	return snd_compr_ioctl(file, cmd, (unsigned long)compat_ptr(arg));
}
#endif

static const struct file_operations snd_compr_file_ops = {
		.owner =	THIS_MODULE,
		.open =		snd_compr_open,
		.release =	snd_compr_free,
		.write =	snd_compr_write,
		.read =		snd_compr_read,
		.unlocked_ioctl = snd_compr_ioctl,
#ifdef CONFIG_COMPAT
		.compat_ioctl = snd_compr_ioctl_compat,
#endif
		.mmap =		snd_compr_mmap,
		.poll =		snd_compr_poll,
};
```

compress 音频流的生命周期从 `open` 开始，到 `release` (`free`) 结束，`open` 操作 `snd_compr_open()` 和 `release` 操作 `snd_compr_free()` 定义 (位于 *sound/core/compress_offload.c*) 如下：
```
struct snd_compr_file {
	unsigned long caps;
	struct snd_compr_stream stream;
};

static void error_delayed_work(struct work_struct *work);

/*
 * a note on stream states used:
 * we use following states in the compressed core
 * SNDRV_PCM_STATE_OPEN: When stream has been opened.
 * SNDRV_PCM_STATE_SETUP: When stream has been initialized. This is done by
 *	calling SNDRV_COMPRESS_SET_PARAMS. Running streams will come to this
 *	state at stop by calling SNDRV_COMPRESS_STOP, or at end of drain.
 * SNDRV_PCM_STATE_PREPARED: When a stream has been written to (for
 *	playback only). User after setting up stream writes the data buffer
 *	before starting the stream.
 * SNDRV_PCM_STATE_RUNNING: When stream has been started and is
 *	decoding/encoding and rendering/capturing data.
 * SNDRV_PCM_STATE_DRAINING: When stream is draining current data. This is done
 *	by calling SNDRV_COMPRESS_DRAIN.
 * SNDRV_PCM_STATE_PAUSED: When stream is paused. This is done by calling
 *	SNDRV_COMPRESS_PAUSE. It can be stopped or resumed by calling
 *	SNDRV_COMPRESS_STOP or SNDRV_COMPRESS_RESUME respectively.
 */
static int snd_compr_open(struct inode *inode, struct file *f)
{
	struct snd_compr *compr;
	struct snd_compr_file *data;
	struct snd_compr_runtime *runtime;
	enum snd_compr_direction dirn;
	int maj = imajor(inode);
	int ret;

	if ((f->f_flags & O_ACCMODE) == O_WRONLY)
		dirn = SND_COMPRESS_PLAYBACK;
	else if ((f->f_flags & O_ACCMODE) == O_RDONLY)
		dirn = SND_COMPRESS_CAPTURE;
	else
		return -EINVAL;

	if (maj == snd_major)
		compr = snd_lookup_minor_data(iminor(inode),
					SNDRV_DEVICE_TYPE_COMPRESS);
	else
		return -EBADFD;

	if (compr == NULL) {
		pr_err("no device data!!!\n");
		return -ENODEV;
	}

	if (dirn != compr->direction) {
		pr_err("this device doesn't support this direction\n");
		snd_card_unref(compr->card);
		return -EINVAL;
	}

	data = kzalloc(sizeof(*data), GFP_KERNEL);
	if (!data) {
		snd_card_unref(compr->card);
		return -ENOMEM;
	}

	INIT_DELAYED_WORK(&data->stream.error_work, error_delayed_work);

	data->stream.ops = compr->ops;
	data->stream.direction = dirn;
	data->stream.private_data = compr->private_data;
	data->stream.device = compr;
	runtime = kzalloc(sizeof(*runtime), GFP_KERNEL);
	if (!runtime) {
		kfree(data);
		snd_card_unref(compr->card);
		return -ENOMEM;
	}
	runtime->state = SNDRV_PCM_STATE_OPEN;
	init_waitqueue_head(&runtime->sleep);
	data->stream.runtime = runtime;
	f->private_data = (void *)data;
	mutex_lock(&compr->lock);
	ret = compr->ops->open(&data->stream);
	mutex_unlock(&compr->lock);
	if (ret) {
		kfree(runtime);
		kfree(data);
	}
	snd_card_unref(compr->card);
	return ret;
}

static int snd_compr_free(struct inode *inode, struct file *f)
{
	struct snd_compr_file *data = f->private_data;
	struct snd_compr_runtime *runtime = data->stream.runtime;

	cancel_delayed_work_sync(&data->stream.error_work);

	switch (runtime->state) {
	case SNDRV_PCM_STATE_RUNNING:
	case SNDRV_PCM_STATE_DRAINING:
	case SNDRV_PCM_STATE_PAUSED:
		data->stream.ops->trigger(&data->stream, SNDRV_PCM_TRIGGER_STOP);
		break;
	default:
		break;
	}

	data->stream.ops->free(&data->stream);
	if (!data->stream.runtime->dma_buffer_p)
		kfree(data->stream.runtime->buffer);
	kfree(data->stream.runtime);
	kfree(data);
	return 0;
}
```

`snd_compr_open()` 的执行过程大体如下：

1. 获得设备文件的主设备号；
2. 从打开文件的 flags 的 `O_ACCMODE` 获取要打开的压缩音频流的方向，flags Read-Only 对应于 capture，Write-Only 对应于 playback；
3. 检查设备文件的主设备号是否与音频设备的主设备号匹配，如果匹配，获得设备文件的次设备号，并根据次设备号获得对应的 compress 设备的 `struct snd_compr` 对象，如果不匹配则报错返回；
4. 检查要打开的压缩音频流的方向与 compress 设备的音频数据的方向是否匹配，如果不匹配则报错返回，否则继续执行；
5. 为要打开的压缩音频流创建 `struct snd_compr_stream` 和 `struct snd_compr_runtime` 对象，并将压缩音频流的状态切换到 **OPEN**；
6. 为打开的压缩音频流调用 compress 音频设备驱动程序的 `open` 操作。

`snd_compr_free()` 释放压缩音频流及其持有的资源，它的执行过程大体如下：

1. 取消延迟执行的 error work；
2. 当状态为 **RUNNING**、**DRAINING** 和 **PAUSED** 时，为打开的压缩音频流调用 compress 音频设备驱动程序的 `trigger` 操作的 **SNDRV_PCM_TRIGGER_STOP** 命令，停止底层硬件设备驱动程序的压缩音频流操作；
3. 为打开的压缩音频流调用 compress 音频设备驱动程序的 `free` 操作，释放底层硬件设备驱动程序的压缩音频流相关资源；
4. 如果需要，释放压缩音频流的 DMA 缓冲区；
5. 释放 `struct snd_compr_stream` 和 `struct snd_compr_runtime` 对象，压缩音频流的生命周期结束。

compress 音频流状态机主要由 compress 设备文件文件操作的 `ioctl` 操作实现，compress 设备文件的 `ioctl` 操作 `snd_compr_ioctl()` 支持许多命令来触发状态机状态的转换，它支持的所有命令有如下：

 - **SNDRV_COMPRESS_IOCTL_VERSION**：查询 API 的版本
 - **SNDRV_COMPRESS_GET_CAPS**：查询 DSP 的能力
 - **SNDRV_COMPRESS_GET_CODEC_CAPS**：查询一个 codec 的能力
 - **SNDRV_COMPRESS_SET_PARAMS**：设置 codec 和压缩音频流的参数，只有 codec 参数可以在运行时改变，流参数不可以
 - **SNDRV_COMPRESS_GET_PARAMS**：查询 codec 参数
 - **SNDRV_COMPRESS_SET_METADATA**：为压缩音频流设置 metadata 值
 - **SNDRV_COMPRESS_GET_METADATA**：从压缩音频流获取请求的 metadata 值
 - **SNDRV_COMPRESS_TSTAMP**：获得当前的时间戳值
 - **SNDRV_COMPRESS_AVAIL**：获得当前的缓冲区可用值，它也会查询 tstamp 属性
 - **SNDRV_COMPRESS_PAUSE**：暂停运行的压缩音频流
 - **SNDRV_COMPRESS_RESUME**：恢复暂停的压缩音频流
 - **SNDRV_COMPRESS_START**：启动压缩音频流
 - **SNDRV_COMPRESS_STOP**：停止运行的压缩音频流，丢弃环形缓冲区和其它缓冲区的内容
 - **SNDRV_COMPRESS_DRAIN**：播放到缓冲区的结束并停止
 - **SNDRV_COMPRESS_NEXT_TRACK**：切换到下一个音轨
 - **SNDRV_COMPRESS_PARTIAL_DRAIN**：

`snd_compr_ioctl()` 操作定义 (位于 *sound/core/compress_offload.c*) 如下：
```
static int
snd_compr_get_caps(struct snd_compr_stream *stream, unsigned long arg)
{
	int retval;
	struct snd_compr_caps caps;

	if (!stream->ops->get_caps)
		return -ENXIO;

	memset(&caps, 0, sizeof(caps));
	retval = stream->ops->get_caps(stream, &caps);
	if (retval)
		goto out;
	if (copy_to_user((void __user *)arg, &caps, sizeof(caps)))
		retval = -EFAULT;
out:
	return retval;
}

#ifndef COMPR_CODEC_CAPS_OVERFLOW
static int
snd_compr_get_codec_caps(struct snd_compr_stream *stream, unsigned long arg)
{
	int retval;
	struct snd_compr_codec_caps *caps;

	if (!stream->ops->get_codec_caps)
		return -ENXIO;

	caps = kzalloc(sizeof(*caps), GFP_KERNEL);
	if (!caps)
		return -ENOMEM;

	retval = stream->ops->get_codec_caps(stream, caps);
	if (retval)
		goto out;
	if (copy_to_user((void __user *)arg, caps, sizeof(*caps)))
		retval = -EFAULT;

out:
	kfree(caps);
	return retval;
}
#endif /* !COMPR_CODEC_CAPS_OVERFLOW */
 . . . . . .
/* revisit this with snd_pcm_preallocate_xxx */
static int snd_compr_allocate_buffer(struct snd_compr_stream *stream,
		struct snd_compr_params *params)
{
	unsigned int buffer_size;
	void *buffer = NULL;

	buffer_size = params->buffer.fragment_size * params->buffer.fragments;
	if (stream->ops->copy) {
		buffer = NULL;
		/* if copy is defined the driver will be required to copy
		 * the data from core
		 */
	} else {
		if (stream->runtime->dma_buffer_p) {

			if (buffer_size > stream->runtime->dma_buffer_p->bytes)
				dev_err(&stream->device->dev,
						"Not enough DMA buffer");
			else
				buffer = stream->runtime->dma_buffer_p->area;

		} else {
			buffer = kmalloc(buffer_size, GFP_KERNEL);
		}

		if (!buffer)
			return -ENOMEM;
	}
	stream->runtime->fragment_size = params->buffer.fragment_size;
	stream->runtime->fragments = params->buffer.fragments;
	stream->runtime->buffer = buffer;
	stream->runtime->buffer_size = buffer_size;
	return 0;
}

static int snd_compress_check_input(struct snd_compr_params *params)
{
	/* first let's check the buffer parameter's */
	if (params->buffer.fragment_size == 0 ||
	    params->buffer.fragments > U32_MAX / params->buffer.fragment_size ||
	    params->buffer.fragments == 0)
		return -EINVAL;

	/* now codec parameters */
	if (params->codec.id == 0 || params->codec.id > SND_AUDIOCODEC_MAX)
		return -EINVAL;

	if (params->codec.ch_in == 0 || params->codec.ch_out == 0)
		return -EINVAL;

	return 0;
}

static int
snd_compr_set_params(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_compr_params *params;
	int retval;

	if (stream->runtime->state == SNDRV_PCM_STATE_OPEN) {
		/*
		 * we should allow parameter change only when stream has been
		 * opened not in other cases
		 */
		params = memdup_user((void __user *)arg, sizeof(*params));
		if (IS_ERR(params))
			return PTR_ERR(params);

		retval = snd_compress_check_input(params);
		if (retval)
			goto out;

		retval = snd_compr_allocate_buffer(stream, params);
		if (retval) {
			retval = -ENOMEM;
			goto out;
		}

		retval = stream->ops->set_params(stream, params);
		if (retval)
			goto out;

		stream->metadata_set = false;
		stream->next_track = false;

		stream->runtime->state = SNDRV_PCM_STATE_SETUP;
	} else {
		return -EPERM;
	}
out:
	kfree(params);
	return retval;
}

static int
snd_compr_get_params(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_codec *params;
	int retval;

	if (!stream->ops->get_params)
		return -EBADFD;

	params = kzalloc(sizeof(*params), GFP_KERNEL);
	if (!params)
		return -ENOMEM;
	retval = stream->ops->get_params(stream, params);
	if (retval)
		goto out;
	if (copy_to_user((char __user *)arg, params, sizeof(*params)))
		retval = -EFAULT;

out:
	kfree(params);
	return retval;
}

static int
snd_compr_get_metadata(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_compr_metadata metadata;
	int retval;

	if (!stream->ops->get_metadata)
		return -ENXIO;

	if (copy_from_user(&metadata, (void __user *)arg, sizeof(metadata)))
		return -EFAULT;

	retval = stream->ops->get_metadata(stream, &metadata);
	if (retval != 0)
		return retval;

	if (copy_to_user((void __user *)arg, &metadata, sizeof(metadata)))
		return -EFAULT;

	return 0;
}

static int
snd_compr_set_metadata(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_compr_metadata metadata;
	int retval;

	if (!stream->ops->set_metadata)
		return -ENXIO;
	/*
	* we should allow parameter change only when stream has been
	* opened not in other cases
	*/
	if (copy_from_user(&metadata, (void __user *)arg, sizeof(metadata)))
		return -EFAULT;

	retval = stream->ops->set_metadata(stream, &metadata);
	stream->metadata_set = true;

	return retval;
}

static inline int
snd_compr_tstamp(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_compr_tstamp tstamp = {0};
	int ret;

	ret = snd_compr_update_tstamp(stream, &tstamp);
	if (ret == 0)
		ret = copy_to_user((struct snd_compr_tstamp __user *)arg,
			&tstamp, sizeof(tstamp)) ? -EFAULT : 0;
	return ret;
}

static int snd_compr_pause(struct snd_compr_stream *stream)
{
	int retval;

	if (stream->runtime->state != SNDRV_PCM_STATE_RUNNING)
		return -EPERM;
	retval = stream->ops->trigger(stream, SNDRV_PCM_TRIGGER_PAUSE_PUSH);
	if (!retval)
		stream->runtime->state = SNDRV_PCM_STATE_PAUSED;
	return retval;
}

static int snd_compr_resume(struct snd_compr_stream *stream)
{
	int retval;

	if (stream->runtime->state != SNDRV_PCM_STATE_PAUSED)
		return -EPERM;
	retval = stream->ops->trigger(stream, SNDRV_PCM_TRIGGER_PAUSE_RELEASE);
	if (!retval)
		stream->runtime->state = SNDRV_PCM_STATE_RUNNING;
	return retval;
}

static int snd_compr_start(struct snd_compr_stream *stream)
{
	int retval;

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_SETUP:
		if (stream->direction != SND_COMPRESS_CAPTURE)
			return -EPERM;
		break;
	case SNDRV_PCM_STATE_PREPARED:
		break;
	default:
		return -EPERM;
	}

	retval = stream->ops->trigger(stream, SNDRV_PCM_TRIGGER_START);
	if (!retval)
		stream->runtime->state = SNDRV_PCM_STATE_RUNNING;
	return retval;
}

static int snd_compr_stop(struct snd_compr_stream *stream)
{
	int retval;

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
	case SNDRV_PCM_STATE_SETUP:
	case SNDRV_PCM_STATE_PREPARED:
		return -EPERM;
	default:
		break;
	}

	retval = stream->ops->trigger(stream, SNDRV_PCM_TRIGGER_STOP);
	if (!retval) {
		/* clear flags and stop any drain wait */
		stream->partial_drain = false;
		stream->metadata_set = false;
		snd_compr_drain_notify(stream);
		stream->runtime->total_bytes_available = 0;
		stream->runtime->total_bytes_transferred = 0;
	}
	return retval;
}
 . . . . . .
static int snd_compress_wait_for_drain(struct snd_compr_stream *stream)
{
	int ret;

	/*
	 * We are called with lock held. So drop the lock while we wait for
	 * drain complete notification from the driver
	 *
	 * It is expected that driver will notify the drain completion and then
	 * stream will be moved to SETUP state, even if draining resulted in an
	 * error. We can trigger next track after this.
	 */
	stream->runtime->state = SNDRV_PCM_STATE_DRAINING;
	mutex_unlock(&stream->device->lock);

	/* we wait for drain to complete here, drain can return when
	 * interruption occurred, wait returned error or success.
	 * For the first two cases we don't do anything different here and
	 * return after waking up
	 */

	ret = wait_event_interruptible(stream->runtime->sleep,
			(stream->runtime->state != SNDRV_PCM_STATE_DRAINING));
	if (ret == -ERESTARTSYS)
		pr_debug("wait aborted by a signal\n");
	else if (ret)
		pr_debug("wait for drain failed with %d\n", ret);


	wake_up(&stream->runtime->sleep);
	mutex_lock(&stream->device->lock);

	return ret;
}

static int snd_compr_drain(struct snd_compr_stream *stream)
{
	int retval;

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
	case SNDRV_PCM_STATE_SETUP:
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_PAUSED:
		return -EPERM;
	case SNDRV_PCM_STATE_XRUN:
		return -EPIPE;
	default:
		break;
	}

	retval = stream->ops->trigger(stream, SND_COMPR_TRIGGER_DRAIN);
	if (retval) {
		pr_debug("SND_COMPR_TRIGGER_DRAIN failed %d\n", retval);
		wake_up(&stream->runtime->sleep);
		return retval;
	}

	return snd_compress_wait_for_drain(stream);
}

static int snd_compr_next_track(struct snd_compr_stream *stream)
{
	int retval;

	/* only a running stream can transition to next track */
	if (stream->runtime->state != SNDRV_PCM_STATE_RUNNING)
		return -EPERM;

	/* next track doesn't have any meaning for capture streams */
	if (stream->direction == SND_COMPRESS_CAPTURE)
		return -EPERM;

	/* you can signal next track if this is intended to be a gapless stream
	 * and current track metadata is set
	 */
	if (stream->metadata_set == false)
		return -EPERM;

	retval = stream->ops->trigger(stream, SND_COMPR_TRIGGER_NEXT_TRACK);
	if (retval != 0)
		return retval;
	stream->metadata_set = false;
	stream->next_track = true;
	return 0;
}

static int snd_compr_partial_drain(struct snd_compr_stream *stream)
{
	int retval;

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
	case SNDRV_PCM_STATE_SETUP:
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_PAUSED:
		return -EPERM;
	case SNDRV_PCM_STATE_XRUN:
		return -EPIPE;
	default:
		break;
	}

	/* partial drain doesn't have any meaning for capture streams */
	if (stream->direction == SND_COMPRESS_CAPTURE)
		return -EPERM;

	/* stream can be drained only when next track has been signalled */
	if (stream->next_track == false)
		return -EPERM;

	stream->partial_drain = true;
	retval = stream->ops->trigger(stream, SND_COMPR_TRIGGER_PARTIAL_DRAIN);
	if (retval) {
		pr_debug("Partial drain returned failure\n");
		wake_up(&stream->runtime->sleep);
		return retval;
	}

	stream->next_track = false;
	return snd_compress_wait_for_drain(stream);
}

static long snd_compr_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
	struct snd_compr_file *data = f->private_data;
	struct snd_compr_stream *stream;
	int retval = -ENOTTY;

	if (snd_BUG_ON(!data))
		return -EFAULT;

	stream = &data->stream;

	mutex_lock(&stream->device->lock);
	switch (_IOC_NR(cmd)) {
	case _IOC_NR(SNDRV_COMPRESS_IOCTL_VERSION):
		retval = put_user(SNDRV_COMPRESS_VERSION,
				(int __user *)arg) ? -EFAULT : 0;
		break;
	case _IOC_NR(SNDRV_COMPRESS_GET_CAPS):
		retval = snd_compr_get_caps(stream, arg);
		break;
#ifndef COMPR_CODEC_CAPS_OVERFLOW
	case _IOC_NR(SNDRV_COMPRESS_GET_CODEC_CAPS):
		retval = snd_compr_get_codec_caps(stream, arg);
		break;
#endif
	case _IOC_NR(SNDRV_COMPRESS_SET_PARAMS):
		retval = snd_compr_set_params(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_GET_PARAMS):
		retval = snd_compr_get_params(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_SET_METADATA):
		retval = snd_compr_set_metadata(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_GET_METADATA):
		retval = snd_compr_get_metadata(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_TSTAMP):
		retval = snd_compr_tstamp(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_AVAIL):
		retval = snd_compr_ioctl_avail(stream, arg);
		break;
	case _IOC_NR(SNDRV_COMPRESS_PAUSE):
		retval = snd_compr_pause(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_RESUME):
		retval = snd_compr_resume(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_START):
		retval = snd_compr_start(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_STOP):
		retval = snd_compr_stop(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_DRAIN):
		retval = snd_compr_drain(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_PARTIAL_DRAIN):
		retval = snd_compr_partial_drain(stream);
		break;
	case _IOC_NR(SNDRV_COMPRESS_NEXT_TRACK):
		retval = snd_compr_next_track(stream);
		break;

	}
	mutex_unlock(&stream->device->lock);
	return retval;
}
```

**SNDRV_COMPRESS_IOCTL_VERSION** 命令查询 API 的版本，`snd_compr_ioctl()` 操作对这个命令的处理为，将预定义的版本号常量复制到传入的用户空间缓冲区。

**SNDRV_COMPRESS_GET_CAPS** 和 **SNDRV_COMPRESS_GET_CODEC_CAPS** 命令分别查询 DSP 的能力和一个 codec 的能力，它们查询 DSP 硬件及其驱动程序的信息。`snd_compr_ioctl()` 操作对这两个命令的处理为，在内核中分配一块缓冲区，调用 compress 音频设备驱动程序的对应操作获得需要的信息，这些信息保存在缓冲区中，最后将缓冲区中的信息拷贝到用户空间并返回。对于 **SNDRV_COMPRESS_GET_CAPS** 命令，缓冲区在内核栈上分配；对于 **SNDRV_COMPRESS_GET_CODEC_CAPS** 命令，缓冲区所需的空间稍大一点，为了节省宝贵的内核栈空间，缓冲区动态分配。

**SNDRV_COMPRESS_SET_PARAMS** 命令设置 codec 和压缩音频流的参数，它使得压缩音频流的状态从 **OPEN** 转换到 **SETUP**。这个命令由 `snd_compr_set_params()` 函数处理。这个命令的处理过程如下：

1. 检查压缩音频流的状态是否为 **OPEN**，如果不是，报错退出，否则继续执行。
2. 将传入的 codec 和压缩音频流的参数拷贝到内核的缓冲区中。
3. 检查传入的参数，这包括检查压缩音频流的 `fragment_size` 和 `fragments` 参数，及 codec 的 `id`、输入通道数 `ch_in` 和输出通道数 `ch_out` 等参数，如果检查失败则报错退出，否则继续执行。
4. 调用 `snd_compr_allocate_buffer()` 函数分配音频数据缓冲区并设置运行时 `fragment_size`、`fragments` 和数据缓冲区相关参数。当 compress 音频设备驱动程序没有提供 `copy()` 操作，且在 compress 音频设备驱动程序的 `open()` 操作中没有分配数据缓冲区时，会实际分配内存。compress 音频设备驱动程序提供 `copy()` 操作，意味着它打算完全由自己处理和用户空间的数据传递，包括数据缓冲区的管理。
5. 调用 compress 音频设备驱动程序的 `set_params` 操作，通知底层硬件设备驱动程序，为压缩音频流设置的 codec 和流参数。
6. 初始化压缩音频流的 `metadata_set` 和 `next_track` 状态，这意味着 `SNDRV_COMPRESS_SET_METADATA` 和 `SNDRV_COMPRESS_NEXT_TRACK` 命令应该在 **SNDRV_COMPRESS_SET_PARAMS** 命令之后调用，否则状态会被覆盖。
7. 将压缩音频流的状态切换到 **OPEN**。

**SNDRV_COMPRESS_SET_METADATA** 命令为压缩音频流设置 metadata 值，`snd_compr_ioctl()` 操作对这个命令的处理为，将传入的 metadata 值拷贝到内核的栈上缓冲区，调用 compress 音频设备驱动程序的 `set_metadata` 操作，将栈上缓冲区中的 metadata 值设置给 compress 音频设备驱动程序，并更新压缩音频流的 `metadata_set` 状态。

**SNDRV_COMPRESS_GET_PARAMS** 和 **SNDRV_COMPRESS_GET_METADATA** 命令分别用于查询 codec 参数，和从压缩音频流获取请求的 metadata 值。`snd_compr_ioctl()` 操作对这两个命令的处理，与对 **SNDRV_COMPRESS_GET_CAPS** 和 **SNDRV_COMPRESS_GET_CODEC_CAPS** 命令的处理完全一样。

**SNDRV_COMPRESS_START** 命令启动压缩音频流，这个命令执行之后，对于播放的流，DSP 应该开始读取数据缓冲区中的音频数据并开始播放；对于录制的流，DSP 应该开始启动录制，并将编码之后的音频数据放进数据缓冲区。对于不同方向的压缩音频流，调用这个命令的时机不同，对于播放的流，需要数据缓冲区中已经有了一些压缩的音频数据，压缩的音频流处于 **PREPARED** 状态；对于录制的流，则只需要压缩的音频流处于 **SETUP** 状态即可。Compress 音频设备文件文件操作的 `write()` 操作将要播放的音频数据放进数据缓冲区，并将压缩音频流的状态从 **SETUP** 切换到 **PREPARED** 状态。`snd_compr_ioctl()` 操作对这个命令的处理为，首先检查压缩音频流的状态，如果状态不满足条件，则报错退出，否则继续执行；调用 compress 音频设备驱动程序的 `trigger` 操作的 **SNDRV_PCM_TRIGGER_START** 命令，触发底层 compress 音频设备的启动；将压缩音频流的状态切换到 **RUNNING**。
 
**SNDRV_COMPRESS_PAUSE** 和 **SNDRV_COMPRESS_RESUME** 命令分别暂停运行的压缩音频流，和恢复暂停的压缩音频流。这两个命令在 **RUNNING** 和 **PAUSED** 两个状态间相互转换压缩的音频流的状态，这两个命令的执行互为前提。`snd_compr_ioctl()` 操作对这两个命令的处理过程基本相同，首先检查压缩音频流的状态；调用 compress 音频设备驱动程序的 `trigger` 操作的对应命令，触发底层 compress 音频设备数据传递状态的变化；切换压缩音频流的状态。

**SNDRV_COMPRESS_STOP** 命令停止运行的压缩音频流，停止正在传递音频数据的音频流。`snd_compr_ioctl()` 操作对这两个命令的处理为，首先检查压缩音频流的状态不为 **OPEN**、**SETUP** 和 **PREPARED**；调用 compress 音频设备驱动程序的 `trigger` 操作的 **SNDRV_PCM_TRIGGER_STOP** 命令，触发底层 compress 音频设备的停止；复位压缩音频流的 `partial_drain` 和 `metadata_set` 状态；调用 `snd_compr_drain_notify()` 函数切换压缩音频流的状态并唤醒等待音频数据清空的线程，如果处于 partial_drain 中，切换到 **RUNNING** 状态，否则切换到 **SETUP** 状态；复位压缩音频流运行时的数据缓冲区的读写指针。

`snd_compr_drain_notify()` 函数定义 (位于 *include/sound/compress_driver.h*) 如下：
```
static inline void snd_compr_drain_notify(struct snd_compr_stream *stream)
{
	if (snd_BUG_ON(!stream))
		return;

	/* for partial_drain case we are back to running state on success */
	if (stream->partial_drain) {
		stream->runtime->state = SNDRV_PCM_STATE_RUNNING;
		stream->partial_drain = false; /* clear this flag as well */
	} else {
		stream->runtime->state = SNDRV_PCM_STATE_SETUP;
	}

	wake_up(&stream->runtime->sleep);
}
```

**SNDRV_COMPRESS_DRAIN** 命令播放到缓冲区的结束并停止，这个命令针对正在传递音频数据的压缩音频流。`snd_compr_ioctl()` 操作对这个命令的处理为，首先检查压缩音频流的状态不为 **OPEN**、**SETUP**、**PREPARED** 和 **PAUSED**；调用 compress 音频设备驱动程序的 `trigger` 操作的 **SND_COMPR_TRIGGER_DRAIN** 命令，触发底层 compress 音频设备清空音频数据；将压缩音频流的状态置为 **DRAINING** 并等待完成，这可能需要 DSP 设备驱动程序在传递数据结束之后，调用 `snd_compr_fragment_elapsed()` 函数唤醒它。

**SNDRV_COMPRESS_NEXT_TRACK** 和 **SNDRV_COMPRESS_PARTIAL_DRAIN** 命令用于播放音频流，需要切换音轨的场景，**SNDRV_COMPRESS_NEXT_TRACK** 命令将压缩音频流切换到下一个音轨，**SNDRV_COMPRESS_PARTIAL_DRAIN** 命令则播放完前一个音轨还未播放完的音频数据。

`snd_compr_ioctl()` 操作对 **SNDRV_COMPRESS_NEXT_TRACK** 命令的处理为，首先检查压缩音频流的状态为 **RUNNING**，流的方向为播放，且已经设置了 metada；调用 compress 音频设备驱动程序的 `trigger` 操作的 **SND_COMPR_TRIGGER_NEXT_TRACK** 命令，触发底层 compress 音频设备切换到下一个音轨；更新压缩音频流的 `metadata_set` 和 `next_track` 状态，为 **SNDRV_COMPRESS_PARTIAL_DRAIN** 命令的执行做准备。

`snd_compr_ioctl()` 操作对 **SNDRV_COMPRESS_PARTIAL_DRAIN** 命令的处理为，首先检查压缩音频流的状态不为 **OPEN**、**SETUP**、**PREPARED** 和 **PAUSED**；检查确保流的方向为播放，且前面执行过了 **SNDRV_COMPRESS_NEXT_TRACK** 命令；更新压缩音频流的 `partial_drain` 状态；调用 compress 音频设备驱动程序的 `trigger` 操作的 **SND_COMPR_TRIGGER_PARTIAL_DRAIN** 命令，触发底层 compress 音频设备播放完前一个音轨还未播放完的音频数据；更新压缩音频流的 `next_track` 状态，表示切换音轨完成；将压缩音频流的状态置为 **DRAINING** 并等待完成，这可能需要 DSP 设备驱动程序在传递数据结束之后，调用 `snd_compr_fragment_elapsed()` 函数唤醒它。

Compress 音频流的基本数据传递过程，与 PCM 的有一定的相似性。基本过程大体为：
1. 为数据传递分配一块数据缓冲区；
2. 流的运行时中保存对数据缓冲区的读写指针；
3. 对于播放流，用户空间程序向数据缓冲区中写入数据，并更新写指针，硬件设备驱动程序读取数据，并更新读指针；
4. 对于录制流，硬件设备驱动程序向数据缓冲区中写入数据，并更新写指针，用户空间程序读取数据，并更新读指针。
5. 如此往复，直到流结束。

Compress 音频流具体实现上，在 compress 音频设备设备文件操作 `ioctl()` 的 **SNDRV_COMPRESS_SET_PARAMS** 命令中，由 compress-offload 设备驱动核心分配数据缓冲区，或由设备驱动程序在其 `open()` 或 `set_params()` 等操作中分配数据缓冲区。整个压缩音频数据的传递，需要 compress 音频设备的设备文件操作 `write()`/`read()` 及 `ioctl()` 的部分命令协同完成。

压缩音频流的运行时表示 `struct snd_compr_runtime` 中的数据缓冲区写指针为 `total_bytes_available`，读指针为 `total_bytes_transferred`。它们是流的指针，随着数据的传递单调递增。

compress 音频设备的 `write()` 设备文件操作 `snd_compr_write()` 函数向音频数据缓冲区写入要播放的数据，这个函数定义 (位于 *sound/core/compress_offload.c*) 如下：
```
static int snd_compr_update_tstamp(struct snd_compr_stream *stream,
		struct snd_compr_tstamp *tstamp)
{
	if (!stream->ops->pointer)
		return -ENOTSUPP;
	stream->ops->pointer(stream, tstamp);
	pr_debug("dsp consumed till %d total %d bytes\n",
		tstamp->byte_offset, tstamp->copied_total);
	if (stream->direction == SND_COMPRESS_PLAYBACK)
		stream->runtime->total_bytes_transferred = tstamp->copied_total;
	else
		stream->runtime->total_bytes_available = tstamp->copied_total;
	return 0;
}

static size_t snd_compr_calc_avail(struct snd_compr_stream *stream,
		struct snd_compr_avail *avail)
{
	memset(avail, 0, sizeof(*avail));
	snd_compr_update_tstamp(stream, &avail->tstamp);
	/* Still need to return avail even if tstamp can't be filled in */

	if (stream->runtime->total_bytes_available == 0 &&
			stream->runtime->state == SNDRV_PCM_STATE_SETUP &&
			stream->direction == SND_COMPRESS_PLAYBACK) {
		pr_debug("detected init and someone forgot to do a write\n");
		return stream->runtime->buffer_size;
	}
	pr_debug("app wrote %lld, DSP consumed %lld\n",
			stream->runtime->total_bytes_available,
			stream->runtime->total_bytes_transferred);
	if (stream->runtime->total_bytes_available ==
				stream->runtime->total_bytes_transferred) {
		if (stream->direction == SND_COMPRESS_PLAYBACK) {
			pr_debug("both pointers are same, returning full avail\n");
			return stream->runtime->buffer_size;
		} else {
			pr_debug("both pointers are same, returning no avail\n");
			return 0;
		}
	}

	avail->avail = stream->runtime->total_bytes_available -
			stream->runtime->total_bytes_transferred;
	if (stream->direction == SND_COMPRESS_PLAYBACK)
		avail->avail = stream->runtime->buffer_size - avail->avail;

	pr_debug("ret avail as %lld\n", avail->avail);
	return avail->avail;
}

static inline size_t snd_compr_get_avail(struct snd_compr_stream *stream)
{
	struct snd_compr_avail avail;

	return snd_compr_calc_avail(stream, &avail);
}

static int
snd_compr_ioctl_avail(struct snd_compr_stream *stream, unsigned long arg)
{
	struct snd_compr_avail ioctl_avail;
	size_t avail;

	avail = snd_compr_calc_avail(stream, &ioctl_avail);
	ioctl_avail.avail = avail;

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
		return -EBADFD;
	case SNDRV_PCM_STATE_XRUN:
		return -EPIPE;
	default:
		break;
	}

	if (copy_to_user((__u64 __user *)arg,
				&ioctl_avail, sizeof(ioctl_avail)))
		return -EFAULT;
	return 0;
}

static int snd_compr_write_data(struct snd_compr_stream *stream,
	       const char __user *buf, size_t count)
{
	void *dstn;
	size_t copy;
	struct snd_compr_runtime *runtime = stream->runtime;
	/* 64-bit Modulus */
	u64 app_pointer = div64_u64(runtime->total_bytes_available,
				    runtime->buffer_size);
	app_pointer = runtime->total_bytes_available -
		      (app_pointer * runtime->buffer_size);

	dstn = runtime->buffer + app_pointer;
	pr_debug("copying %ld at %lld\n",
			(unsigned long)count, app_pointer);
	if (count < runtime->buffer_size - app_pointer) {
		if (copy_from_user(dstn, buf, count))
			return -EFAULT;
	} else {
		copy = runtime->buffer_size - app_pointer;
		if (copy_from_user(dstn, buf, copy))
			return -EFAULT;
		if (copy_from_user(runtime->buffer, buf + copy, count - copy))
			return -EFAULT;
	}
	/* if DSP cares, let it know data has been written */
	if (stream->ops->ack)
		stream->ops->ack(stream, count);
	return count;
}

static ssize_t snd_compr_write(struct file *f, const char __user *buf,
		size_t count, loff_t *offset)
{
	struct snd_compr_file *data = f->private_data;
	struct snd_compr_stream *stream;
	size_t avail;
	int retval;

	if (snd_BUG_ON(!data))
		return -EFAULT;

	stream = &data->stream;
	mutex_lock(&stream->device->lock);
	/* write is allowed when stream is running or has been steup */
	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_SETUP:
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_RUNNING:
		break;
	default:
		mutex_unlock(&stream->device->lock);
		return -EBADFD;
	}

	avail = snd_compr_get_avail(stream);
	pr_debug("avail returned %ld\n", (unsigned long)avail);
	/* calculate how much we can write to buffer */
	if (avail > count)
		avail = count;

	if (stream->ops->copy) {
		char __user* cbuf = (char __user*)buf;
		retval = stream->ops->copy(stream, cbuf, avail);
	} else {
		retval = snd_compr_write_data(stream, buf, avail);
	}
	if (retval > 0)
		stream->runtime->total_bytes_available += retval;

	/* while initiating the stream, write should be called before START
	 * call, so in setup move state */
	if (stream->runtime->state == SNDRV_PCM_STATE_SETUP) {
		stream->runtime->state = SNDRV_PCM_STATE_PREPARED;
		pr_debug("stream prepared, Houston we are good to go\n");
	}

	mutex_unlock(&stream->device->lock);
	return retval;
}
```

`snd_compr_write()` 函数的执行过程为：

1. 检查压缩音频流的状态为 **SETUP**、**PREPARED** 或 **RUNNING**。
2. 调用 `snd_compr_get_avail()` 函数计算音频数据缓冲区中可用空间的大小，这个可用空间是从用户空间程序视角来看的，对于播放流，指还可以向数据缓冲区写入的数据量，对于录制流，指可以读取的已经编码好的数据量。对于播放流，大体为 (缓冲区大小 - (写指针 - 读指针))，对于录制流，大体为 (写指针 - 读指针)。在使用数据缓冲区的读写指针计算可用空间大小之前，会先调用 `snd_compr_get_avail()` 函数，通过 compress 音频设备驱动程序的 `pointer` 操作，获得硬件 DSP 当前已经传递的数据量，并更新数据缓冲区的读写指针，对于播放流，是读指针；对于录制流，是写指针。
3. 根据要传递的数据量，和数据缓冲区中可用空间的大小，计算实际需要传递数据量。
4. 将用户空间的音频数据实际拷贝到内核音频数据缓冲区。这分为两种情况：
(1). compress 音频设备驱动程序提供了 `copy()` 操作，意味着它打算完全由自己处理和用户空间的数据传递，则调用 compress 音频设备驱动程序的 `copy()` 操作，由底层硬件设备驱动程序来处理音频数据从用户空间到内核数据缓冲区的拷贝。
(2). compress 音频设备驱动程序没有提供 `copy()` 操作，则调用 `snd_compr_write_data()` 函数拷贝数据。拷贝数据时，首先计算指向数据缓冲区的实际的写指针；随后分两种情况处理，一是写指针的位置到数据缓冲区结尾有足够空间的，则直接拷贝，二是写指针的位置到数据缓冲区结尾处空间不够，需要绕回数据缓冲区开头继续写的，则分两段拷贝；最后调用 compress 音频设备驱动程序的 `ack()` 操作，通知底层硬件设备驱动程序拷贝的数据量。
5. 更新数据缓冲区的写指针。
6. 压缩音频流的状态为 **SETUP** 时，将其状态切换为 **PREPARED**，为后面启动压缩音频流做准备。

compress 音频设备设备文件操作 `ioctl()` 的 **SNDRV_COMPRESS_TSTAMP** 命令给用户空间程序获得数据缓冲区和硬件设备之间已经拷贝的音频数据量，并更新数据缓冲区的读写指针，对于播放流，是读指针；对于录制流，是写指针。**SNDRV_COMPRESS_AVAIL** 命令获得从用户空间程序视角来看的，当前音频数据缓冲区可用空间的大小，它也会查询 tstamp 属性。

compress 音频设备的 `read()` 设备文件操作 `snd_compr_read()` 函数从音频数据缓冲区读出录制的音频数据，这个函数定义 (位于 *sound/core/compress_offload.c*) 如下：
```
static ssize_t snd_compr_read(struct file *f, char __user *buf,
		size_t count, loff_t *offset)
{
	struct snd_compr_file *data = f->private_data;
	struct snd_compr_stream *stream;
	size_t avail;
	int retval;

	if (snd_BUG_ON(!data))
		return -EFAULT;

	stream = &data->stream;
	mutex_lock(&stream->device->lock);

	/* read is allowed when stream is running, paused, draining and setup
	 * (yes setup is state which we transition to after stop, so if user
	 * wants to read data after stop we allow that)
	 */
	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_SUSPENDED:
	case SNDRV_PCM_STATE_DISCONNECTED:
		retval = -EBADFD;
		goto out;
	case SNDRV_PCM_STATE_XRUN:
		retval = -EPIPE;
		goto out;
	}

	avail = snd_compr_get_avail(stream);
	pr_debug("avail returned %ld\n", (unsigned long)avail);
	/* calculate how much we can read from buffer */
	if (avail > count)
		avail = count;

	if (stream->ops->copy) {
		retval = stream->ops->copy(stream, buf, avail);
	} else {
		retval = -ENXIO;
		goto out;
	}
	if (retval > 0)
		stream->runtime->total_bytes_transferred += retval;

out:
	mutex_unlock(&stream->device->lock);
	return retval;
}
```

`snd_compr_read()` 函数的执行过程与 `snd_compr_write()` 函数的基本相同：

1. 检查压缩音频流的状态不为 **OPEN**、**PREPARED**、**SUSPENDED** 和 **DISCONNECTED**。
2. 调用 `snd_compr_get_avail()` 函数计算音频数据缓冲区中可用空间的大小。
3. 根据要传递的数据量，和数据缓冲区中可用空间的大小，计算实际需要传递数据量。
4. 将内核音频数据缓冲区的音频数据实际拷贝到用户空间。这个文件操作强制要求 compress 音频设备驱动程序提供 `copy()` 操作，否则报错。数据拷贝由 compress 音频设备驱动程序的 `copy()` 操作完成。
5. 更新数据缓冲区的读指针。

compress 音频设备提供了空的 `mmap()` 设备文件操作 `snd_compr_mmap()`，并提供了 `poll()` 设备文件操作 `snd_compr_poll()`，这两个操作定义 (位于 *sound/core/compress_offload.c*) 如下：
```
static int snd_compr_mmap(struct file *f, struct vm_area_struct *vma)
{
	return -ENXIO;
}

static __poll_t snd_compr_get_poll(struct snd_compr_stream *stream)
{
	if (stream->direction == SND_COMPRESS_PLAYBACK)
		return EPOLLOUT | EPOLLWRNORM;
	else
		return EPOLLIN | EPOLLRDNORM;
}

static __poll_t snd_compr_poll(struct file *f, poll_table *wait)
{
	struct snd_compr_file *data = f->private_data;
	struct snd_compr_stream *stream;
	size_t avail;
	__poll_t retval = 0;

	if (snd_BUG_ON(!data))
		return EPOLLERR;

	stream = &data->stream;

	mutex_lock(&stream->device->lock);

	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_OPEN:
	case SNDRV_PCM_STATE_XRUN:
		retval = snd_compr_get_poll(stream) | EPOLLERR;
		goto out;
	default:
		break;
	}

	poll_wait(f, &stream->runtime->sleep, wait);

	avail = snd_compr_get_avail(stream);
	pr_debug("avail is %ld\n", (unsigned long)avail);
	/* check if we have at least one fragment to fill */
	switch (stream->runtime->state) {
	case SNDRV_PCM_STATE_DRAINING:
		/* stream has been woken up after drain is complete
		 * draining done so set stream state to stopped
		 */
		retval = snd_compr_get_poll(stream);
		stream->runtime->state = SNDRV_PCM_STATE_SETUP;
		break;
	case SNDRV_PCM_STATE_RUNNING:
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_PAUSED:
		if (avail >= stream->runtime->fragment_size)
			retval = snd_compr_get_poll(stream);
		break;
	default:
		retval = snd_compr_get_poll(stream) | EPOLLERR;
		break;
	}
out:
	mutex_unlock(&stream->device->lock);
	return retval;
}
```

对于 compress 音频设备驱动程序，必须为录制流提供 `copy()` 操作；对于播放流，则 `copy()` 和 `ack()` 操作二选一提供，两者同时存在时，`ack()` 操作不会被用到。

ALSA Compress-Offload API 和 PCM API 类似，但也有一些不同的地方：

1. Compress-Offload API 设计的比较正交，不同 API 间基本上没有功能交叉，不同 API 完成不同的功能。如 PCM API 的 `ioctl()` 设备文件操作提供了传递播放或录制数据的命令，`read()`/`write()` 设备文件操作提供相同的功能；Compress-Offload API 只有 `read()`/`write()` 设备文件操作可以传递音频数据。
2. Compress-Offload API 的数据传递操做不会主动触发压缩音频流的启动，但 PCM API 的会。
3. Compress-Offload 驱动核心不如 PCM 的完善。

ALSA Compress-Offload 设备驱动核心将 compress 音频设备文件的文件操作映射到  compress 音频设备驱动程序的接口。

### SoC 中的 Compress 设备驱动

在用于给 SoC 中的音频设备开发驱动程序的 ASoC 框架中，所有音频设备统一用 `struct snd_soc_dai` 对象表示，音频设备支持的操作用 `struct snd_soc_dai_driver` 及其 `struct snd_soc_dai_ops` 和 `struct snd_soc_cdai_ops` 描述。

ASoC 创建了 component 抽象，通常一个设备驱动程序模块实现为一个 component 驱动。一个 component 可以包含一个或多个由 `struct snd_soc_dai` 表示的音频设备。component 驱动的接口为 `struct snd_soc_component_driver`。`struct snd_soc_component_driver` 除包含支持 PCM 设备的操作外，还有一个类型为 `struct snd_compress_ops` 指针的字段，指向一组压缩音频组件操作。ASoC 的 compress 设备驱动程序，可以为 component 提供 `struct snd_compress_ops` 操作集。

可以实现完整的音频数据通路，并导出 pcm 或 compr 这样的设备文件的，在 ASoC 中由 `struct snd_soc_dai_link` 对象表示。`struct snd_soc_dai_link` 可以包含一个或多个 component 的多个音频设备。驱动程序可以为 `struct snd_soc_dai_link` 提供一个用于支持 compress DAI link 的操作集 `struct snd_soc_compr_ops`。

ASoC 中的声卡由 `struct snd_soc_card` 对象表示，声卡可以包含一个或多个 `struct snd_soc_dai_link`。

ASoC 中的这些抽象大体有下图这样的结构：

![Linux ALSA ASoC Objects](https://upload-images.jianshu.io/upload_images/1315506-012a03be8f6cb3d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

支持音频数据编解码的 DSP 硬件设备的 Linux ALSA ASoC 驱动程序，通常实现为某个 component 驱动程序的一个 `struct snd_soc_dai_driver`，具体来说要做这些事情：
 * 为 DSP 硬件设备创建 `struct snd_soc_dai_driver` 对象。
 * 为 `struct snd_soc_dai_driver` 配置支持的流方向，音频参数，特别地提供 `compress_new` 操作实现用于创建 compress 设备。
 * 为 `struct snd_soc_dai_driver` 配置 `struct snd_soc_cdai_ops` 操作集。
 * 为 DSP 音频设备驱动所属的 component 配置 `struct snd_compress_ops` 操作集。
 * 在 machine 驱动程序中为 compress 设备创建 DAI link。
 * 为 compress 设备的 DAI link 配置 `struct snd_soc_compr_ops` 操作集。

ASoC compress 设备驱动核心将 ALSA compress-offload 设备驱动核心的 compress 设备驱动接口转换到 ASoC compress 设备驱动接口。

`struct snd_soc_cdai_ops` 结构定义 (位于 *include/sound/soc-dai.h*) 如下：
```
struct snd_soc_cdai_ops {
	/*
	 * for compress ops
	 */
	int (*startup)(struct snd_compr_stream *,
			struct snd_soc_dai *);
	int (*shutdown)(struct snd_compr_stream *,
			struct snd_soc_dai *);
	int (*set_params)(struct snd_compr_stream *,
			struct snd_compr_params *, struct snd_soc_dai *);
	int (*get_params)(struct snd_compr_stream *,
			struct snd_codec *, struct snd_soc_dai *);
	int (*set_metadata)(struct snd_compr_stream *,
			struct snd_compr_metadata *, struct snd_soc_dai *);
	int (*get_metadata)(struct snd_compr_stream *,
			struct snd_compr_metadata *, struct snd_soc_dai *);
	int (*trigger)(struct snd_compr_stream *, int,
			struct snd_soc_dai *);
	int (*pointer)(struct snd_compr_stream *,
			struct snd_compr_tstamp *, struct snd_soc_dai *);
	int (*ack)(struct snd_compr_stream *, size_t,
			struct snd_soc_dai *);
};
```

`struct snd_compress_ops` 结构定义 (位于 *include/sound/soc-component.h*) 如下：
```
/* component interface */
struct snd_compress_ops {
	int (*open)(struct snd_soc_component *component,
		    struct snd_compr_stream *stream);
	int (*free)(struct snd_soc_component *component,
		    struct snd_compr_stream *stream);
	int (*set_params)(struct snd_soc_component *component,
			  struct snd_compr_stream *stream,
			  struct snd_compr_params *params);
	int (*get_params)(struct snd_soc_component *component,
			  struct snd_compr_stream *stream,
			  struct snd_codec *params);
	int (*set_metadata)(struct snd_soc_component *component,
			    struct snd_compr_stream *stream,
			    struct snd_compr_metadata *metadata);
	int (*get_metadata)(struct snd_soc_component *component,
			    struct snd_compr_stream *stream,
			    struct snd_compr_metadata *metadata);
	int (*trigger)(struct snd_soc_component *component,
		       struct snd_compr_stream *stream, int cmd);
	int (*pointer)(struct snd_soc_component *component,
		       struct snd_compr_stream *stream,
		       struct snd_compr_tstamp *tstamp);
	int (*copy)(struct snd_soc_component *component,
		    struct snd_compr_stream *stream, char __user *buf,
		    size_t count);
	int (*mmap)(struct snd_soc_component *component,
		    struct snd_compr_stream *stream,
		    struct vm_area_struct *vma);
	int (*ack)(struct snd_soc_component *component,
		   struct snd_compr_stream *stream, size_t bytes);
	int (*get_caps)(struct snd_soc_component *component,
			struct snd_compr_stream *stream,
			struct snd_compr_caps *caps);
	int (*get_codec_caps)(struct snd_soc_component *component,
			      struct snd_compr_stream *stream,
			      struct snd_compr_codec_caps *codec);
};
```

`struct snd_soc_compr_ops` 结构定义 (位于 *include/sound/soc.h*) 如下：
```
struct snd_soc_compr_ops {
	int (*startup)(struct snd_compr_stream *);
	void (*shutdown)(struct snd_compr_stream *);
	int (*set_params)(struct snd_compr_stream *);
	int (*trigger)(struct snd_compr_stream *);
};
```

Component 的 `struct snd_compress_ops` 接口包含的操作和 `struct snd_compr_ops` 包含的完全相同，各个操作的含义也完全相同。相对于 `struct snd_compress_ops`，`struct snd_soc_cdai_ops` 包含的操作少了 `copy`、`mmap`、`get_caps` 和 `get_codec_caps`四个，其它完全相同。

一般情况下，为 DSP ASoC 驱动程序的 `struct snd_soc_dai_driver` 提供的 `compress_new` 操作实现为 ASoC compress 设备驱动核心定义的 `snd_soc_new_compress()` 函数。`snd_soc_new_compress()` 函数定义 (位于 *sound/soc/soc-compress.c*) 如下：
```
/* ASoC Compress operations */
static struct snd_compr_ops soc_compr_ops = {
	.open		= soc_compr_open,
	.free		= soc_compr_free,
	.set_params	= soc_compr_set_params,
	.set_metadata   = soc_compr_set_metadata,
	.get_metadata	= soc_compr_get_metadata,
	.get_params	= soc_compr_get_params,
	.trigger	= soc_compr_trigger,
	.pointer	= soc_compr_pointer,
	.ack		= soc_compr_ack,
	.get_caps	= soc_compr_get_caps,
	.get_codec_caps = soc_compr_get_codec_caps
};

/* ASoC Dynamic Compress operations */
static struct snd_compr_ops soc_compr_dyn_ops = {
	.open		= soc_compr_open_fe,
	.free		= soc_compr_free_fe,
	.set_params	= soc_compr_set_params_fe,
	.get_params	= soc_compr_get_params,
	.set_metadata   = soc_compr_set_metadata,
	.get_metadata	= soc_compr_get_metadata,
	.trigger	= soc_compr_trigger_fe,
	.pointer	= soc_compr_pointer,
	.ack		= soc_compr_ack,
	.get_caps	= soc_compr_get_caps,
	.get_codec_caps = soc_compr_get_codec_caps
};

```
/**
 * snd_soc_new_compress - create a new compress.
 *
 * @rtd: The runtime for which we will create compress
 * @num: the device index number (zero based - shared with normal PCMs)
 *
 * Return: 0 for success, else error.
 */
int snd_soc_new_compress(struct snd_soc_pcm_runtime *rtd, int num)
{
	struct snd_soc_component *component;
	struct snd_soc_dai *codec_dai = asoc_rtd_to_codec(rtd, 0);
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	struct snd_compr *compr;
	struct snd_pcm *be_pcm;
	char new_name[64];
	int ret = 0, direction = 0;
	int playback = 0, capture = 0;
	int i;

	if (rtd->num_cpus > 1 ||
	    rtd->num_codecs > 1) {
		dev_err(rtd->card->dev,
			"Compress ASoC: Multi CPU/Codec not supported\n");
		return -EINVAL;
	}

	if (!codec_dai) {
		dev_err(rtd->card->dev, "Missing codec\n");
		return -EINVAL;
	}

	/* check client and interface hw capabilities */
	if (snd_soc_dai_stream_valid(codec_dai, SNDRV_PCM_STREAM_PLAYBACK) &&
	    snd_soc_dai_stream_valid(cpu_dai,   SNDRV_PCM_STREAM_PLAYBACK))
		playback = 1;
	if (snd_soc_dai_stream_valid(codec_dai, SNDRV_PCM_STREAM_CAPTURE) &&
	    snd_soc_dai_stream_valid(cpu_dai,   SNDRV_PCM_STREAM_CAPTURE))
		capture = 1;

	/*
	 * Compress devices are unidirectional so only one of the directions
	 * should be set, check for that (xor)
	 */
	if (playback + capture != 1) {
		dev_err(rtd->card->dev,
			"Compress ASoC: Invalid direction for P %d, C %d\n",
			playback, capture);
		return -EINVAL;
	}

	if (playback)
		direction = SND_COMPRESS_PLAYBACK;
	else
		direction = SND_COMPRESS_CAPTURE;

	compr = devm_kzalloc(rtd->card->dev, sizeof(*compr), GFP_KERNEL);
	if (!compr)
		return -ENOMEM;

	compr->ops = devm_kzalloc(rtd->card->dev, sizeof(soc_compr_ops),
				  GFP_KERNEL);
	if (!compr->ops)
		return -ENOMEM;

	if (rtd->dai_link->dynamic) {
		snprintf(new_name, sizeof(new_name), "(%s)",
			rtd->dai_link->stream_name);

		ret = snd_pcm_new_internal(rtd->card->snd_card, new_name, num,
				rtd->dai_link->dpcm_playback,
				rtd->dai_link->dpcm_capture, &be_pcm);
		if (ret < 0) {
			dev_err(rtd->card->dev,
				"Compress ASoC: can't create compressed for %s: %d\n",
				rtd->dai_link->name, ret);
			return ret;
		}

		rtd->pcm = be_pcm;
		rtd->fe_compr = 1;
		if (rtd->dai_link->dpcm_playback)
			be_pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream->private_data = rtd;
		else if (rtd->dai_link->dpcm_capture)
			be_pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream->private_data = rtd;
		memcpy(compr->ops, &soc_compr_dyn_ops, sizeof(soc_compr_dyn_ops));
	} else {
		snprintf(new_name, sizeof(new_name), "%s %s-%d",
			rtd->dai_link->stream_name, codec_dai->name, num);

		memcpy(compr->ops, &soc_compr_ops, sizeof(soc_compr_ops));
	}

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->copy)
			continue;

		compr->ops->copy = soc_compr_copy;
		break;
	}

	mutex_init(&compr->lock);
	ret = snd_compress_new(rtd->card->snd_card, num, direction,
				new_name, compr);
	if (ret < 0) {
		component = asoc_rtd_to_codec(rtd, 0)->component;
		dev_err(component->dev,
			"Compress ASoC: can't create compress for codec %s: %d\n",
			component->name, ret);
		return ret;
	}

	/* DAPM dai link stream work */
	rtd->close_delayed_work_func = snd_soc_close_delayed_work;

	rtd->compr = compr;
	compr->private_data = rtd;

	dev_dbg(rtd->card->dev, "Compress ASoC: %s <-> %s mapping ok\n",
		codec_dai->name, cpu_dai->name);

	return 0;
}
EXPORT_SYMBOL_GPL(snd_soc_new_compress);
```

`snd_soc_new_compress()` 函数要求 DSP compress DAI link 同时有且仅有一个 CPU DAI 和 codec DAI，且它们只有相同的一个方向的流。这个函数的执行过程大体如下：

1. 检查 DSP compress DAI link 满足要求；
2. 为 compress 音频设备分配 `struct snd_compr` 对象；
3. 为 compress 音频设备分配 `struct snd_compr_ops` 对象；
4. 为 compress 音频设备选择操作集，如果是动态 PCM，选择 `soc_compr_dyn_ops`，否则，选择 `soc_compr_ops`；
5. 为 compress 音频设备选择操作集的 `copy` 操作，操作集的 `copy` 操作具有逻辑意义，操作集的 `copy` 操作是否存在对音频数据的传递过程有重大影响，这里根据各个 component 驱动程序是否提供 `copy` 操作来为 compress 音频设备选择操作集的 `copy` 操作；
6. 为声卡创建 compress 音频设备。

`snd_soc_new_compress()` 函数被调用的过程大体如下：
```
[   20.282884]  snd_soc_new_compress+0x98/0x34c
[   20.288754]  snd_soc_dai_compress_new+0x30/0x84
[   20.294948]  snd_soc_bind_card+0x7d0/0x924
[   20.300542]  snd_soc_register_card+0xf4/0x10c
[   20.306516]  devm_snd_soc_register_card+0x44/0xa0
```

ASoC compress 设备驱动核心定义了一组 `struct snd_soc_compr_ops` 操作的包装函数，这些函数定义 (位于 *sound/soc/soc-link.c*) 如下：
```
int snd_soc_link_compr_startup(struct snd_compr_stream *cstream)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	int ret = 0;

	if (rtd->dai_link->compr_ops &&
	    rtd->dai_link->compr_ops->startup)
		ret = rtd->dai_link->compr_ops->startup(cstream);

	return soc_link_ret(rtd, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_link_compr_startup);

void snd_soc_link_compr_shutdown(struct snd_compr_stream *cstream)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;

	if (rtd->dai_link->compr_ops &&
	    rtd->dai_link->compr_ops->shutdown)
		rtd->dai_link->compr_ops->shutdown(cstream);
}
EXPORT_SYMBOL_GPL(snd_soc_link_compr_shutdown);

int snd_soc_link_compr_set_params(struct snd_compr_stream *cstream)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	int ret = 0;

	if (rtd->dai_link->compr_ops &&
	    rtd->dai_link->compr_ops->set_params)
		ret = rtd->dai_link->compr_ops->set_params(cstream);

	return soc_link_ret(rtd, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_link_compr_set_params);
```

`struct snd_soc_compr_ops` 操作集的四个操作中的 `trigger()` 操作外，每个操作都有一个包装函数。

ASoC compress 设备驱动核心定义了一组 `struct snd_soc_cdai_ops` 操作的包装函数，这些函数定义 (位于 *sound/soc/soc-link.c*) 如下：
```
int snd_soc_dai_compr_startup(struct snd_soc_dai *dai,
			      struct snd_compr_stream *cstream)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->startup)
		ret = dai->driver->cops->startup(cstream, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_startup);

void snd_soc_dai_compr_shutdown(struct snd_soc_dai *dai,
				struct snd_compr_stream *cstream)
{
	if (dai->driver->cops &&
	    dai->driver->cops->shutdown)
		dai->driver->cops->shutdown(cstream, dai);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_shutdown);

int snd_soc_dai_compr_trigger(struct snd_soc_dai *dai,
			      struct snd_compr_stream *cstream, int cmd)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->trigger)
		ret = dai->driver->cops->trigger(cstream, cmd, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_trigger);

int snd_soc_dai_compr_set_params(struct snd_soc_dai *dai,
				 struct snd_compr_stream *cstream,
				 struct snd_compr_params *params)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->set_params)
		ret = dai->driver->cops->set_params(cstream, params, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_set_params);

int snd_soc_dai_compr_get_params(struct snd_soc_dai *dai,
				 struct snd_compr_stream *cstream,
				 struct snd_codec *params)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->get_params)
		ret = dai->driver->cops->get_params(cstream, params, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_get_params);

int snd_soc_dai_compr_ack(struct snd_soc_dai *dai,
			  struct snd_compr_stream *cstream,
			  size_t bytes)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->ack)
		ret = dai->driver->cops->ack(cstream, bytes, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_ack);

int snd_soc_dai_compr_pointer(struct snd_soc_dai *dai,
			      struct snd_compr_stream *cstream,
			      struct snd_compr_tstamp *tstamp)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->pointer)
		ret = dai->driver->cops->pointer(cstream, tstamp, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_pointer);

int snd_soc_dai_compr_set_metadata(struct snd_soc_dai *dai,
				   struct snd_compr_stream *cstream,
				   struct snd_compr_metadata *metadata)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->set_metadata)
		ret = dai->driver->cops->set_metadata(cstream, metadata, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_set_metadata);

int snd_soc_dai_compr_get_metadata(struct snd_soc_dai *dai,
				   struct snd_compr_stream *cstream,
				   struct snd_compr_metadata *metadata)
{
	int ret = 0;

	if (dai->driver->cops &&
	    dai->driver->cops->get_metadata)
		ret = dai->driver->cops->get_metadata(cstream, metadata, dai);

	return soc_dai_ret(dai, ret);
}
EXPORT_SYMBOL_GPL(snd_soc_dai_compr_get_metadata);
```

`struct snd_soc_cdai_ops` 操作集的九个操作，每个操作一个包装函数。

这里主要关注非动态 PCM 的情况，compress 音频设备的 `struct snd_compr_ops` 操作集各操作定义 (位于 *sound/soc/soc-compress.c*) 如下：
```
static int soc_compr_components_open(struct snd_compr_stream *cstream,
				     struct snd_soc_component **last)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->open)
			continue;

		ret = component->driver->compress_ops->open(component, cstream);
		if (ret < 0) {
			dev_err(component->dev,
				"Compress ASoC: can't open platform %s: %d\n",
				component->name, ret);

			*last = component;
			return ret;
		}
	}

	*last = NULL;
	return 0;
}

static int soc_compr_components_free(struct snd_compr_stream *cstream,
				     struct snd_soc_component *last)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i;

	for_each_rtd_components(rtd, i, component) {
		if (component == last)
			break;

		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->free)
			continue;

		component->driver->compress_ops->free(component, cstream);
	}

	return 0;
}

static int soc_compr_open(struct snd_compr_stream *cstream)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component = NULL;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int ret;

	ret = snd_soc_pcm_component_pm_runtime_get(rtd, cstream);
	if (ret < 0)
		goto pm_err;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	ret = snd_soc_dai_compr_startup(cpu_dai, cstream);
	if (ret < 0)
		goto out;

	ret = soc_compr_components_open(cstream, &component);
	if (ret < 0)
		goto machine_err;

	ret = snd_soc_link_compr_startup(cstream);
	if (ret < 0)
		goto machine_err;

	snd_soc_runtime_activate(rtd, cstream->direction);

	mutex_unlock(&rtd->card->pcm_mutex);

	return 0;

machine_err:
	soc_compr_components_free(cstream, component);

	snd_soc_dai_compr_shutdown(cpu_dai, cstream);
out:
	mutex_unlock(&rtd->card->pcm_mutex);
pm_err:
	snd_soc_pcm_component_pm_runtime_put(rtd, cstream, 1);

	return ret;
}
 . . . . . .
static int soc_compr_free(struct snd_compr_stream *cstream)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	struct snd_soc_dai *codec_dai = asoc_rtd_to_codec(rtd, 0);
	int stream;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	if (cstream->direction == SND_COMPRESS_PLAYBACK)
		stream = SNDRV_PCM_STREAM_PLAYBACK;
	else
		stream = SNDRV_PCM_STREAM_CAPTURE;

	snd_soc_runtime_deactivate(rtd, stream);

	snd_soc_dai_digital_mute(codec_dai, 1, cstream->direction);

	if (!snd_soc_dai_active(cpu_dai))
		cpu_dai->rate = 0;

	if (!snd_soc_dai_active(codec_dai))
		codec_dai->rate = 0;

	snd_soc_link_compr_shutdown(cstream);

	soc_compr_components_free(cstream, NULL);

	snd_soc_dai_compr_shutdown(cpu_dai, cstream);

	snd_soc_dapm_stream_stop(rtd, stream);

	mutex_unlock(&rtd->card->pcm_mutex);

	snd_soc_pcm_component_pm_runtime_put(rtd, cstream, 0);

	return 0;
}
 . . . . . .
static int soc_compr_components_trigger(struct snd_compr_stream *cstream,
					int cmd)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->trigger)
			continue;

		ret = component->driver->compress_ops->trigger(
			component, cstream, cmd);
		if (ret < 0)
			return ret;
	}

	return 0;
}

static int soc_compr_trigger(struct snd_compr_stream *cstream, int cmd)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_dai *codec_dai = asoc_rtd_to_codec(rtd, 0);
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int ret;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	ret = soc_compr_components_trigger(cstream, cmd);
	if (ret < 0)
		goto out;

	ret = snd_soc_dai_compr_trigger(cpu_dai, cstream, cmd);
	if (ret < 0)
		goto out;

	switch (cmd) {
	case SNDRV_PCM_TRIGGER_START:
		snd_soc_dai_digital_mute(codec_dai, 0, cstream->direction);
		break;
	case SNDRV_PCM_TRIGGER_STOP:
		snd_soc_dai_digital_mute(codec_dai, 1, cstream->direction);
		break;
	}

out:
	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}
 . . . . . .
static int soc_compr_components_set_params(struct snd_compr_stream *cstream,
					   struct snd_compr_params *params)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->set_params)
			continue;

		ret = component->driver->compress_ops->set_params(
			component, cstream, params);
		if (ret < 0)
			return ret;
	}

	return 0;
}

static int soc_compr_set_params(struct snd_compr_stream *cstream,
				struct snd_compr_params *params)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int ret;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	/*
	 * First we call set_params for the CPU DAI, then the component
	 * driver this should configure the SoC side. If the machine has
	 * compressed ops then we call that as well. The expectation is
	 * that these callbacks will configure everything for this compress
	 * path, like configuring a PCM port for a CODEC.
	 */
	ret = snd_soc_dai_compr_set_params(cpu_dai, cstream, params);
	if (ret < 0)
		goto err;

	ret = soc_compr_components_set_params(cstream, params);
	if (ret < 0)
		goto err;

	ret = snd_soc_link_compr_set_params(cstream);
	if (ret < 0)
		goto err;

	if (cstream->direction == SND_COMPRESS_PLAYBACK)
		snd_soc_dapm_stream_event(rtd, SNDRV_PCM_STREAM_PLAYBACK,
					  SND_SOC_DAPM_STREAM_START);
	else
		snd_soc_dapm_stream_event(rtd, SNDRV_PCM_STREAM_CAPTURE,
					  SND_SOC_DAPM_STREAM_START);

	/* cancel any delayed stream shutdown that is pending */
	rtd->pop_wait = 0;
	mutex_unlock(&rtd->card->pcm_mutex);

	cancel_delayed_work_sync(&rtd->delayed_work);

	return 0;

err:
	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}
 . . . . . .
static int soc_compr_get_params(struct snd_compr_stream *cstream,
				struct snd_codec *params)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int i, ret = 0;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	ret = snd_soc_dai_compr_get_params(cpu_dai, cstream, params);
	if (ret < 0)
		goto err;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->get_params)
			continue;

		ret = component->driver->compress_ops->get_params(
			component, cstream, params);
		break;
	}

err:
	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_get_caps(struct snd_compr_stream *cstream,
			      struct snd_compr_caps *caps)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret = 0;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->get_caps)
			continue;

		ret = component->driver->compress_ops->get_caps(
			component, cstream, caps);
		break;
	}

	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_get_codec_caps(struct snd_compr_stream *cstream,
				    struct snd_compr_codec_caps *codec)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret = 0;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->get_codec_caps)
			continue;

		ret = component->driver->compress_ops->get_codec_caps(
			component, cstream, codec);
		break;
	}

	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_ack(struct snd_compr_stream *cstream, size_t bytes)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int i, ret = 0;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	ret = snd_soc_dai_compr_ack(cpu_dai, cstream, bytes);
	if (ret < 0)
		goto err;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->ack)
			continue;

		ret = component->driver->compress_ops->ack(
			component, cstream, bytes);
		if (ret < 0)
			goto err;
	}

err:
	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_pointer(struct snd_compr_stream *cstream,
			     struct snd_compr_tstamp *tstamp)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret = 0;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	ret = snd_soc_dai_compr_pointer(cpu_dai, cstream, tstamp);
	if (ret < 0)
		goto out;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->pointer)
			continue;

		ret = component->driver->compress_ops->pointer(
			component, cstream, tstamp);
		break;
	}
out:
	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_copy(struct snd_compr_stream *cstream,
			  char __user *buf, size_t count)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	int i, ret = 0;

	mutex_lock_nested(&rtd->card->pcm_mutex, rtd->card->pcm_subclass);

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->copy)
			continue;

		ret = component->driver->compress_ops->copy(
			component, cstream, buf, count);
		break;
	}

	mutex_unlock(&rtd->card->pcm_mutex);
	return ret;
}

static int soc_compr_set_metadata(struct snd_compr_stream *cstream,
				  struct snd_compr_metadata *metadata)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int i, ret;

	ret = snd_soc_dai_compr_set_metadata(cpu_dai, cstream, metadata);
	if (ret < 0)
		return ret;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->set_metadata)
			continue;

		ret = component->driver->compress_ops->set_metadata(
			component, cstream, metadata);
		if (ret < 0)
			return ret;
	}

	return 0;
}

static int soc_compr_get_metadata(struct snd_compr_stream *cstream,
				  struct snd_compr_metadata *metadata)
{
	struct snd_soc_pcm_runtime *rtd = cstream->private_data;
	struct snd_soc_component *component;
	struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
	int i, ret;

	ret = snd_soc_dai_compr_get_metadata(cpu_dai, cstream, metadata);
	if (ret < 0)
		return ret;

	for_each_rtd_components(rtd, i, component) {
		if (!component->driver->compress_ops ||
		    !component->driver->compress_ops->get_metadata)
			continue;

		return component->driver->compress_ops->get_metadata(
			component, cstream, metadata);
	}

	return 0;
}
```

除 `open`、`free`、`set_params` 和 `trigger` 四个操作外，`struct snd_compr_ops` 操作集其它各个操作的实现情况如下：

 * `set_metadata` 操作 `soc_compr_set_metadata()` 函数：第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的对应操作，和 component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `get_metadata` 操作 `soc_compr_get_metadata()` 函数：第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的对应操作，和 component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `get_params` 操作 `soc_compr_get_params()` 函数：第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的对应操作，和 component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `pointer` 操作 `soc_compr_pointer()` 函数：第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的对应操作，和 component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `copy` 操作 `soc_compr_copy()` 函数：component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `ack` 操作 `soc_compr_ack()` 函数：第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的对应操作，和 component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `get_caps` 操作 `soc_compr_get_caps()` 函数：component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `get_codec_caps` 操作 `soc_compr_get_codec_caps()` 函数：component 驱动程序的 `struct snd_compress_ops` 操作集对应操作的包装函数，它调用每一个 component 驱动程序的对应操作。
 * `open` 操作 `soc_compr_open()` 函数：
     1. 增加设备的使用计数并恢复它，在电源管理系统中为设备上电；
     2. 分别调用第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的 `startup` 操作包装函数，component 驱动程序的 `struct snd_compress_ops` 操作集的 `open` 操作包装函数，和 DAI link 的 `struct snd_soc_compr_ops` 操作集的 `startup` 操作包装函数；
     3. 增加 PCM 运行时组件的激活计数。
 * `free` 操作 `soc_compr_free()` 函数，基本上执行了 `open` 操作的逆动作：
     1. 减少 PCM 运行时组件的激活计数，**比较奇怪的是，传入的 stream 参数，没有像 `open` 操作中的那样，不对压缩音频流的 stream 状态做类型转换直接传入，而是先做了类型转换，这两个地方一定有一个地方写的不合适**；
     2. 数字静音 codec DAI，**即使是对于压缩音频流，为它的 codec DAI 驱动提供`struct snd_soc_dai_ops` 操作集的 `mute_stream` 操作也是有意义的**；
     3. 分别调用 DAI link 的 `struct snd_soc_compr_ops` 操作集的 `shutdown` 操作包装函数，component 驱动程序的 `struct snd_compress_ops` 操作集的 `free` 操作包装函数，和第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集的 `shutdown` 操作包装函数；
     4. 发送流停止事件给 dapm 核心；
     5. 降低设备的使用计数并挂起它，在电源管理系统中为设备下电。
 * `set_params` 操作 `soc_compr_set_params()` 函数：
     1. 分别调用第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集，component 驱动程序的 `struct snd_compress_ops` 操作集和 DAI link 的 `struct snd_soc_compr_ops` 操作集的对应操作包装函数；
     2. 发送流启动事件给 dapm 核心；
     3. 取消延迟执行的任务。
 * `trigger` 操作 `soc_compr_trigger()` 函数：
     1. 分别调用 component 驱动程序的 `struct snd_compress_ops` 操作集，和第一个 CPU DAI 的 `struct snd_soc_cdai_ops` 操作集对应操作的包装函数；
     2. 如果是流启动命令，发送流启动事件给 dapm 核心，如果是流停止命令，发送流停止事件。

ASoC compress 设备驱动核心将 ALSA compress 设备驱动核心的设备驱动接口转换到 ASoC compress 设备驱动核心的设备驱动接口。

Done.
