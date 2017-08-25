---
title: 使用 ortp 发送原始 H.264 码流
date: 2017-08-24 22:05:49
categories: 音视频开发
tags:
- 音视频开发
---


oRTP 是一个 RTP (Real-time Transport Protocol ([RFC 3550](https://www.ietf.org/rfc/rfc3550.txt))) 协议的库实现，它完全以 C 语言来实现，因此方便应用于各种不同的平台。本文分享用 oRTP 发送，以 Android 的 MediaCodec 编码出来的原始 H.264 码流，又称裸流的方法。
<!--more-->
# H.264 码流

MediaCode 以 H.264 编码格式编码之后的视频，是由一个一个的NALU组成的。他们的结构如下图所示。

![](https://www.wolfcstech.com/images/1315506-73c59eb70085d7bc.jpg)

其中每个 NALU 之间通过 startcode（起始码）进行分隔。起始码分成两种，一种是 0x000001（3Byte），另一种是 0x00000001（4Byte）。NALU 中，起始码之后，是 NALU 的类型字节，它用于描述这个 NALU 中数据的类型，NALU 的重要性等。H.264 视频流的 meta 信息等也被封装为 NALU，并以特定的类型标识 ，如 SPS 和 PPS 等描述视频流分辨率、码率等特性的信息。NALU 类型字节格式如下：

```
      +---------------+
      |0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+
      |F|NRI|  Type   |
      +---------------+
```

NALU 类型字节中各个字段的语义，在 ITU 的 [H.264规范](http://www.itu.int/rec/T-REC-H.264-201704-I/en) 中有清晰地定义，这里给出它们的简要说明：

* F：1 位
forbidden_zero_bit。H.264 规范声明，值为 1 时表示语法违规。也就是数据包损坏。

* NRI：2 位
nal_ref_idc。这个字段用于描述该 NALU 的重要性。值为 00 表示 NALU 的内容不被用于重建图像预测的参考图像。这种 NALU 可以被丢弃而不危及参考图像的完整性。大于 00 的值表示需要解码该 NALU 来维护参考图像的完整性。

* Type：5 位
nal_unit_type。这个组件指定 NALU 载荷的类型，在 H.264 的表 7-1 中定义。具体的类型定义如下：

![NAL 单元类型码](http://upload-images.jianshu.io/upload_images/1315506-475027950d4a524d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于 NALU 的起始码，如果它对应的 Slice 为一帧的开始就用0x00000001，否则就用 0x000001。H.264 码流解析的步骤就是首先从码流中搜索 0x000001 和 0x00000001，分离出 NALU；然后再分析NALU的各个字段。在 MediaCodec API 的输出中，通常都是一帧的图像被编码为一个 NALU。不同类型的 Meta 信息也会被编码为不同的 NALU，如 SPS，PPS 等。

H.264 码流的分辨率大小不同，NALU 的类型不同等因素，导致 NALU 有着各种不同的大小。RTP 协议通常为了更高的实时性，而会选择用 UDP 作为传输层协议。但 UDP 包的大小受限于 IP 的 MTU，也就是 UDP 包加上 IP 头不能超过 IP 层的 MTU 值大小。因此在用 RTP 传输 H.264 码流时，需要适配 RTP 包的大小限制。

NALU 适配 RTP 包大小，需要分为多种情况来处理：
1. NALU 非常小，多个 NALU 可以放在一个 RTP 包中传输，为了不浪费传输能力，通常需要把它们聚合在一个包中传输。
2. NALU 大小与 RTP 包大小限制在同一量级，一个 RTP 包中可以放一个 NALU，但不能放多个。
3. NALU 比较大，分辨率较高的视频，比如 1080P 的视频，编码出来的一帧图像可能在近 10 KB 到一两百 KB 之间，这种就需要把一个 NALU 放进多个 RTP 包中传输。

在用 RTP 传输 H.264 码流时，会为每个载荷加上一到两个字节的 RTP载荷头部，用于区分前面提到的这多种不同的情况。关于具体的 H.264 视频的 RTP 载荷格式，可以参考 [H.264 视频的 RTP 载荷格式](https://www.wolfcstech.com/2017/08/18/h264_on_rtp/) 一文，或者 IETF 的 [RFC6184](https://tools.ietf.org/html/rfc6184)。

然后来看使用 ortp 发送原始 H.264 码流的方法。

# oRTP 源码下载
首先需要下载 oRTP 的源码，下载地址如下：
```
http://www.linphone.org/technical-corner/ortp/downloads
```

可以通过 Git 下载最新版本的源码：
```
git clone git://git.linphone.org/ortp.git
```

还可以下载不同发布版本打包的 `.tar.gz` 包：
```
http://download.savannah.nongnu.org/releases/linphone/ortp/sources/
```

最新版为 [ortp-0.27.0.tar.gz](http://download.savannah.nongnu.org/releases/linphone/ortp/sources/ortp-0.27.0.tar.gz)。

oRTP 这个项目已经针对 Android 做了移植。得到源码之后，可以在
 `ortp/build/android` 目录下找到 `Android.mk` 文件，可以借助于这个文件，将 oRTP 的代码集成进自己的 JNI 代码或 Android 的代码库中。

# 使用 ortp 发送原始 H.264 码流
使用 ortp 发送原始 H.264 码流主要需要两步，首先是初始化 RtpSession：

```
#define Y_PLOAD_TYPE 96
#define DefaultTimestampIncrement 1500 //(90000 / framerate)

static const unsigned char RtpPayloadTypeNaluMin = 1;
static const unsigned char RtpPayloadTypeNaluMax = 23;
static const unsigned char RtpPayloadTypeStapA = 24;
static const unsigned char RtpPayloadTypeStapB = 25;
static const unsigned char RtpPayloadTypeMtap16 = 26;
static const unsigned char RtpPayloadTypeMtap24 = 27;
static const unsigned char RtpPayloadTypeFuA = 28;
static const unsigned char RtpPayloadTypeFuB = 29;

typedef enum {
    NALU_TYPE_SLICE = 1,
    NALU_TYPE_DPA = 2,
    NALU_TYPE_DPB = 3,
    NALU_TYPE_DPC = 4,
    NALU_TYPE_IDR = 5,
    NALU_TYPE_SEI = 6,
    NALU_TYPE_SPS = 7,
    NALU_TYPE_PPS = 8,
    NALU_TYPE_AUD = 9,
    NALU_TYPE_EOSEQ = 10,
    NALU_TYPE_EOSTREAM = 11,
    NALU_TYPE_FILL = 12,
} NaluType;

static const unsigned char FUHeaderMaskStart = 0x80;
static const unsigned char FUHeaderMaskEnd = 0x40;
static const unsigned char FUHeaderMaskType = 0x1F;

int cond = 1;

void stop_handler(int signum) {
    if (cond == 1) {
        cond = 0;
    } else {
        exit(1);
    }
}

void ssrc_cb(RtpSession *session) {
    printf("hey, the ssrc has changed !\n");
}

static RtpSession * init_rtp_session(int rtp_port) {
    RtpSession *session = NULL;
    bool_t adapt = TRUE;
    int jittcomp = 40;
    char  *ssrc;

    ortp_init();
    ortp_scheduler_init();
    ortp_set_log_level_mask(
            ORTP_DEBUG | ORTP_MESSAGE | ORTP_WARNING | ORTP_ERROR);
    session = rtp_session_new(RTP_SESSION_SENDRECV);

    rtp_session_set_scheduling_mode(session, 1);
    rtp_session_set_blocking_mode(session, 1);
    rtp_session_set_remote_addr(session, "10.242.55.30", rtp_port);
    rtp_session_set_connected_mode(session, TRUE);
    rtp_session_set_payload_type(session, Y_PLOAD_TYPE);

    ssrc = getenv("SSRC");
    if (ssrc != NULL) {
        printf("using SSRC=%i.\n", atoi(ssrc));
        // 设置输出流的SSRC。不做此步的话将会给个随机值
        rtp_session_set_ssrc(session, atoi(ssrc));
    }

    return session;
}
```
RTP 的使用模式，通常是接收者先 listen 在特定的端口上，然后发送者向该端口发送数据。`rtp_session_set_remote_addr()` 用于设置码流的接收端地址。

需要特别说明的一点是载荷类型的设置，`rtp_session_set_payload_type(session, Y_PLOAD_TYPE);`，这里传入了 `96`。载荷类型用于描述某种特定载荷的一些特性，如 MimeType、时钟频率、比特率等。在 [RFC3551 RTP Profile for Audio and Video Conferences with Minimal Control](https://tools.ietf.org/html/rfc3551) 中定义了为具体的载荷类型分配的载荷类型编号。

在 oRTP 中，预定义了许多载荷类型的描述，如 H.263，PCMU8000，H.264 等（在文件 `ortp/src/avprofile.c` 中）：
```
PayloadType payload_type_pcmu8000={
	TYPE(PAYLOAD_AUDIO_CONTINUOUS),
	CLOCK_RATE(8000),
	BITS_PER_SAMPLE(8),
	ZERO_PATTERN( &offset127),
	PATTERN_LENGTH(1),
	NORMAL_BITRATE(64000),
	MIME_TYPE("PCMU"),
	CHANNELS(1),
	RECV_FMTP(NULL),
	SEND_FMTP(NULL),
	NO_AVPF,
	FLAGS(0)
};

PayloadType payload_type_h263={
	TYPE(PAYLOAD_VIDEO),
	CLOCK_RATE(90000),
	BITS_PER_SAMPLE(0),
	ZERO_PATTERN(NULL),
	PATTERN_LENGTH(0),
	NORMAL_BITRATE(256000),
	MIME_TYPE("H263"),
	CHANNELS(0),
	RECV_FMTP(NULL),
	SEND_FMTP(NULL),
	NO_AVPF,
	FLAGS(0)
};

PayloadType payload_type_h264={
	TYPE(PAYLOAD_VIDEO),
	CLOCK_RATE(90000),
	BITS_PER_SAMPLE(0),
	ZERO_PATTERN(NULL),
	PATTERN_LENGTH(0),
	NORMAL_BITRATE(256000),
	MIME_TYPE("H264"),
	CHANNELS(0),
	RECV_FMTP(NULL),
	SEND_FMTP(NULL),
	AVPF(PAYLOAD_TYPE_AVPF_FIR | PAYLOAD_TYPE_AVPF_PLI, RTCP_DEFAULT_REPORT_INTERVAL),
	FLAGS(PAYLOAD_TYPE_RTCP_FEEDBACK_ENABLED)
};
```

此外，还定义了一个表，基于 [RFC3551](https://tools.ietf.org/html/rfc3551) 建立了载荷类型编号与载荷类型之间的映射关系，具体是在文件 `ortp/src/avprofile.c` 中的 `av_profile_init()` 函数里：

```
void av_profile_init(RtpProfile *profile)
{
	rtp_profile_clear_all(profile);
	profile->name="AV profile";
	rtp_profile_set_payload(profile,0,&payload_type_pcmu8000);
	rtp_profile_set_payload(profile,1,&payload_type_lpc1016);
	rtp_profile_set_payload(profile,3,&payload_type_gsm);
	rtp_profile_set_payload(profile,7,&payload_type_lpc);
	rtp_profile_set_payload(profile,4,&payload_type_g7231);
	rtp_profile_set_payload(profile,8,&payload_type_pcma8000);
	rtp_profile_set_payload(profile,9,&payload_type_g722);
	rtp_profile_set_payload(profile,10,&payload_type_l16_stereo);
	rtp_profile_set_payload(profile,11,&payload_type_l16_mono);
	rtp_profile_set_payload(profile,13,&payload_type_cn);
	rtp_profile_set_payload(profile,18,&payload_type_g729);
	rtp_profile_set_payload(profile,31,&payload_type_h261);
	rtp_profile_set_payload(profile,32,&payload_type_mpv);
	rtp_profile_set_payload(profile,34,&payload_type_h263);
	rtp_profile_set_payload(profile,96,&payload_type_t140);
	rtp_profile_set_payload(profile,97,&payload_type_t140_red);
}
```

`av_profile_init()` 函数在 ortp 库初始化时会被调用到：
```
void ortp_init()
{
	if (ortp_initialized++) return;

#ifdef _WIN32
	win32_init_sockets();
#endif

	av_profile_init(&av_profile);
	ortp_global_stats_reset();
	init_random_number_generator();

	ortp_message("oRTP-" ORTP_VERSION " initialized.");
}
```

为 RtpSession 设置的载荷类型对数据收发的过程有一定的影响。

通过 oRTP 收发数据时，需要为其传入用户时间戳，oRTP 会根据为 RtpSession 设置的载荷类型找到描述载荷类型的PayloadType，并根据 PayloadType 的时钟频率和用户时间戳，计算出数据收发的时间间隔。以此实现用户对数据收发频率的控制。如（`ortp/src/rtpsession.c`）：
```
/* function used by the scheduler only:*/
uint32_t rtp_session_ts_to_time (RtpSession * session, uint32_t timestamp)
{
	PayloadType *payload;
	payload =
		rtp_profile_get_payload (session->snd.profile,
					 session->snd.pt);
	if (payload == NULL)
	{
		ortp_warning
			("rtp_session_ts_to_t: use of unsupported payload type %d.", session->snd.pt);
		return 0;
	}
	/* the return value is in milisecond */
	return (uint32_t) (1000.0 *
			  ((double) timestamp /
			   (double) payload->clock_rate));
}
```

在 [RFC3551](https://tools.ietf.org/html/rfc3551) 中，载荷类型 96 是动态映射的类型，通常由特定的应用字节决定。如在 oRTP 中，这个类型是被映射为 T140 的，但也常将 96 映射到 H.264。为了让我们前面设置的载荷类型能够正常工作，还需要修改 oRTP 的源码 `ortp/src/avprofile.c` 中的 `av_profile_init()` 函数，把如下这一行
```
	rtp_profile_set_payload(profile,96,&payload_type_t140);
```
改为
```
	rtp_profile_set_payload(profile,96,&payload_type_h264);
```

初始化了 RtpSession 之后，就可以发送 H.264 裸流了：
```
static bool isNalu3Start(unsigned char *buffer) {
    if (buffer[0] != 0 || buffer[1] != 0 || buffer[2] != 1) {
        return false;
    } else {
        return true;
    }
}

static bool isNalu4Start(unsigned char *buffer) {
    if (buffer[0] != 0 || buffer[1] != 0 || buffer[2] != 0 || buffer[3] != 1) {
        return false;
    } else {
        return true;
    }
}

static void forward_frame(RtpSession * session, uint8_t * buffer, int len,
        uint32_t userts) {
    unsigned char NALU = buffer[4];
    uint32_t valid_len = len - 4;

    if (valid_len <= MAX_RTP_PKT_LENGTH) {
        int offset = 0;
        int lastNaluStartPos = -1;
        while (offset < len) {
            if (isNalu4Start(buffer + offset)) {
                if (lastNaluStartPos >= 0) {
                    rtp_session_send_with_ts(session, buffer + lastNaluStartPos,
                            offset - lastNaluStartPos, userts);
                }
                lastNaluStartPos = offset + 4;
                offset += 3;
            } else if (isNalu3Start(buffer + offset)) {
                if (lastNaluStartPos >= 0) {
                    rtp_session_send_with_ts(session, buffer + lastNaluStartPos,
                            offset - lastNaluStartPos, userts);
                }
                lastNaluStartPos = offset + 3;
                offset += 2;
            }
            ++offset;
        }
        rtp_session_send_with_ts(session, buffer + lastNaluStartPos,
                len - lastNaluStartPos, userts);
    } else {
        valid_len -= 1;
        int packetnum = valid_len / MAX_RTP_PKT_LENGTH;
        if (valid_len % MAX_RTP_PKT_LENGTH != 0) {
            packetnum += 1;
        }
        int i = 0;
        int pos = 5;
        while (i < packetnum) {
            if (i < packetnum - 1) {
                buffer[pos - 2] = (NALU & 0x60) | 28;
                buffer[pos - 1] = (NALU & 0x1f);
                if (0 == i) {
                    buffer[pos - 1] |= 0x80;
                }
                rtp_session_send_with_ts(session, &buffer[pos - 2],
                MAX_RTP_PKT_LENGTH + 2, userts);
            } else {
                int iSendLen = len - pos;
                buffer[pos - 2] = (NALU & 0x60) | 28;
                buffer[pos - 1] = (NALU & 0x1f);
                buffer[pos - 1] |= 0x40;
                rtp_session_send_with_ts(session, &buffer[pos - 2],
                        iSendLen + 2, userts);
            }
            pos += MAX_RTP_PKT_LENGTH;
            ++i;
        }
    }
}
```
这里根据 RTP 载荷格式的规范，将 NALU 转为 RTP 的载荷，并发送。需要特别说明的是，从 MediaCodec 拿到的第一个 Buffer，其内容通常像下面这样：

```
00000000   00 00 00 01  67 42 80 2A  DA 01 10 0F  1E 5E 52 0A  ....gB.*.....^R.
00000010   0C 0A 0D A1  42 6A 00 00  00 01 68 CE  06 E2
```

其中包含了类型分别为 SPS 和 PPS 的两个 NALU。要发送这块 Buffer，可以按照 H.264 的 RTP 载荷格式规范中描述的，单时间聚合包的格式来发送，或者拆分为两个 RTP 包来发送。

RTP 是一个用于流媒体传输的协议，而不是流媒体方案。要想使 RTP 在实际的项目中用起来，当然还是有许多其它工作要做的。

# 参考文档
[视音频数据处理入门：H.264视频码流解析](http://blog.csdn.net/leixiaohua1020/article/details/50534369)
[ORTP移植到Hi3518e，h.264封包rtp发送](http://blog.csdn.net/jiaozi07/article/details/41749943)

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。
