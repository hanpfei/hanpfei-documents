---
title: WebRTC 中收集音视频编解码能力
date: 2022-03-12 20:05:49
categories: 音视频开发
tags:
- 音视频开发
---

在 WebRTC 中，交互的两端在建立连接过程中，需要通过 ICE 协议，交换各自的音视频编解码能力，如编解码器和编解码器的一些参数配置，并协商出一组配置和参数，用于后续的音视频传输过程。
<!--more-->
对于音频，编解码能力的一部分信息从 audio 的 encoder 和 decoder factory 中获取。Audio encoder 和 decoder factory 在创建 `PeerConnectionFactoryInterface` 的时候，由 webrtc 的用户传入，如在 webrtc 示例应用 peerconnection_client 中 (`webrtc/src/examples/peerconnection/client/conductor.cc`)：
```
bool Conductor::InitializePeerConnection() {
  RTC_DCHECK(!peer_connection_factory_);
  RTC_DCHECK(!peer_connection_);

  if (!signaling_thread_.get()) {
    signaling_thread_ = rtc::Thread::CreateWithSocketServer();
    signaling_thread_->Start();
  }
  peer_connection_factory_ = webrtc::CreatePeerConnectionFactory(
      nullptr /* network_thread */, nullptr /* worker_thread */,
      signaling_thread_.get(), nullptr /* default_adm */,
      webrtc::CreateBuiltinAudioEncoderFactory(),
      webrtc::CreateBuiltinAudioDecoderFactory(),
      webrtc::CreateBuiltinVideoEncoderFactory(),
      webrtc::CreateBuiltinVideoDecoderFactory(), nullptr /* audio_mixer */,
      nullptr /* audio_processing */);
```

WebRTC 内部用于创建音频***编码器***工厂的 `CreateBuiltinAudioEncoderFactory()` 函数实现 (`webrtc/src/api/audio_codecs/builtin_audio_encoder_factory.cc`) 如下：
```
rtc::scoped_refptr<AudioEncoderFactory> CreateBuiltinAudioEncoderFactory() {
  return CreateAudioEncoderFactory<

#if WEBRTC_USE_BUILTIN_OPUS
      AudioEncoderOpus, NotAdvertised<AudioEncoderMultiChannelOpus>,
#endif

      AudioEncoderIsac, AudioEncoderG722,

#if WEBRTC_USE_BUILTIN_ILBC
      AudioEncoderIlbc,
#endif

      AudioEncoderG711, NotAdvertised<AudioEncoderL16>>();
}
```

WebRTC 内部用于创建音频***解码器***工厂的 `CreateBuiltinAudioDecoderFactory()` 函数实现 (`webrtc/src/api/audio_codecs/builtin_audio_decoder_factory.cc`) 如下：
```
rtc::scoped_refptr<AudioDecoderFactory> CreateBuiltinAudioDecoderFactory() {
  return CreateAudioDecoderFactory<

#if WEBRTC_USE_BUILTIN_OPUS
      AudioDecoderOpus, NotAdvertised<AudioDecoderMultiChannelOpus>,
#endif

      AudioDecoderIsac, AudioDecoderG722,

#if WEBRTC_USE_BUILTIN_ILBC
      AudioDecoderIlbc,
#endif

      AudioDecoderG711, NotAdvertised<AudioDecoderL16>>();
}
```

一般来说，一个音频编解码器库都会同时支持某种编解码器的编码和解码能力，尽管在 WebRTC 中编码器工厂和解码器工厂是两个类，但它们支持的 codec 集合是完全一样的。

WebRTC 在创建 `cricket::ChannelManager` 时，初始化 `cricket::WebRtcVoiceEngine`，此时会从音频编解码器工厂中获取支持的编解码器：
```
#0  cricket::WebRtcVoiceEngine::Init() (this=0x0) at ../../media/engine/webrtc_voice_engine.cc:341
#1  0x000055555725fd64 in cricket::CompositeMediaEngine::Init() (this=0x5555587c1fb0) at ../../media/base/media_engine.cc:172
#2  0x0000555556b8dff7 in cricket::ChannelManager::Create(std::unique_ptr<cricket::MediaEngineInterface, std::default_delete<cricket::MediaEngineInterface> >, bool, rtc::Thread*, rtc::Thread*)
    (media_engine=std::unique_ptr<cricket::MediaEngineInterface> = {...}, enable_rtx=true, worker_thread=0x7fffe4002cd0, network_thread=0x7fffe4001820)
    at ../../pc/channel_manager.cc:39
#3  0x0000555556c1b413 in webrtc::ConnectionContext::<lambda()>::operator()(void) const (__closure=0x7fffeeffc2c0) at ../../pc/connection_context.cc:132
```

`cricket::WebRtcVoiceEngine::Init()` 获取支持的音频编解码器：
```
void WebRtcVoiceEngine::Init() {
  RTC_DCHECK_RUN_ON(&worker_thread_checker_);
  RTC_LOG(LS_INFO) << "WebRtcVoiceEngine::Init";

  // TaskQueue expects to be created/destroyed on the same thread.
  low_priority_worker_queue_.reset(
      new rtc::TaskQueue(task_queue_factory_->CreateTaskQueue(
          "rtc-low-prio", webrtc::TaskQueueFactory::Priority::LOW)));

  // Load our audio codec lists.
  RTC_LOG(LS_VERBOSE) << "Supported send codecs in order of preference:";
  send_codecs_ = CollectCodecs(encoder_factory_->GetSupportedEncoders());
  for (const AudioCodec& codec : send_codecs_) {
    RTC_LOG(LS_VERBOSE) << ToString(codec);
  }

  RTC_LOG(LS_VERBOSE) << "Supported recv codecs in order of preference:";
  recv_codecs_ = CollectCodecs(decoder_factory_->GetSupportedDecoders());
  for (const AudioCodec& codec : recv_codecs_) {
    RTC_LOG(LS_VERBOSE) << ToString(codec);
  }
```

音频编码器工厂从各个编码器中获得它们的描述，如 OPUS 音频编码器返回自己的描述的过程如下：
```
#0  webrtc::AudioEncoderOpusImpl::AppendSupportedEncoders(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> >*) (specs=0x0)
    at ../../modules/audio_coding/codecs/opus/audio_encoder_opus.cc:208
#1  0x0000555556f91c90 in webrtc::AudioEncoderOpus::AppendSupportedEncoders(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> >*)
    (specs=0x7fffdabf83d0) at ../../api/audio_codecs/opus/audio_encoder_opus.cc:24
#2  0x000055555606d404 in webrtc::audio_encoder_factory_template_impl::Helper<webrtc::AudioEncoderOpus, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioEncoderMultiChannelOpus>, webrtc::AudioEncoderIsacFloat, webrtc::AudioEncoderG722, webrtc::AudioEncoderIlbc, webrtc::AudioEncoderG711, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioEncoderL16> >::AppendSupportedEncoders(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> >*) (specs=0x7fffdabf83d0) at ../../api/audio_codecs/audio_encoder_factory_template.h:49
#3  0x000055555606d340 in webrtc::audio_encoder_factory_template_impl::AudioEncoderFactoryT<webrtc::AudioEncoderOpus, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioEncoderMultiChannelOpus>, webrtc::AudioEncoderIsacFloat, webrtc::AudioEncoderG722, webrtc::AudioEncoderIlbc, webrtc::AudioEncoderG711, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioEncoderL16> >::GetSupportedEncoders() (this=0x555558813800)
    at ../../api/audio_codecs/audio_encoder_factory_template.h:82
#4  0x00005555560da2e4 in cricket::WebRtcVoiceEngine::Init() (this=0x55555843daa0) at ../../media/engine/webrtc_voice_engine.cc:352
#5  0x000055555725fd64 in cricket::CompositeMediaEngine::Init() (this=0x5555587c1fb0) at ../../media/base/media_engine.cc:172
```

OPUS 音频编码器的描述的详细内容可以参考如下这段代码 (`webrtc/src/modules/audio_coding/codecs/opus/audio_encoder_opus.cc`)：
```
AudioCodecInfo AudioEncoderOpusImpl::QueryAudioEncoder(
    const AudioEncoderOpusConfig& config) {
  RTC_DCHECK(config.IsOk());
  AudioCodecInfo info(config.sample_rate_hz, config.num_channels,
                      *config.bitrate_bps,
                      AudioEncoderOpusConfig::kMinBitrateBps,
                      AudioEncoderOpusConfig::kMaxBitrateBps);
  info.allow_comfort_noise = false;
  info.supports_network_adaption = true;
  return info;
}

std::unique_ptr<AudioEncoder> AudioEncoderOpusImpl::MakeAudioEncoder(
    const AudioEncoderOpusConfig& config,
    int payload_type) {
  RTC_DCHECK(config.IsOk());
  return std::make_unique<AudioEncoderOpusImpl>(config, payload_type);
}

absl::optional<AudioEncoderOpusConfig> AudioEncoderOpusImpl::SdpToConfig(
    const SdpAudioFormat& format) {
  if (!absl::EqualsIgnoreCase(format.name, "opus") ||
      format.clockrate_hz != kRtpTimestampRateHz || format.num_channels != 2) {
    return absl::nullopt;
  }

  AudioEncoderOpusConfig config;
  config.num_channels = GetChannelCount(format);
  config.frame_size_ms = GetFrameSizeMs(format);
  config.max_playback_rate_hz = GetMaxPlaybackRate(format);
  config.fec_enabled = (GetFormatParameter(format, "useinbandfec") == "1");
  config.dtx_enabled = (GetFormatParameter(format, "usedtx") == "1");
  config.cbr_enabled = (GetFormatParameter(format, "cbr") == "1");
  config.bitrate_bps =
      CalculateBitrate(config.max_playback_rate_hz, config.num_channels,
                       GetFormatParameter(format, "maxaveragebitrate"));
  config.application = config.num_channels == 1
                           ? AudioEncoderOpusConfig::ApplicationMode::kVoip
                           : AudioEncoderOpusConfig::ApplicationMode::kAudio;

  constexpr int kMinANAFrameLength = kANASupportedFrameLengths[0];
  constexpr int kMaxANAFrameLength =
      kANASupportedFrameLengths[arraysize(kANASupportedFrameLengths) - 1];

  // For now, minptime and maxptime are only used with ANA. If ptime is outside
  // of this range, it will get adjusted once ANA takes hold. Ideally, we'd know
  // if ANA was to be used when setting up the config, and adjust accordingly.
  const int min_frame_length_ms =
      GetFormatParameter<int>(format, "minptime").value_or(kMinANAFrameLength);
  const int max_frame_length_ms =
      GetFormatParameter<int>(format, "maxptime").value_or(kMaxANAFrameLength);

  FindSupportedFrameLengths(min_frame_length_ms, max_frame_length_ms,
                            &config.supported_frame_lengths_ms);
  RTC_DCHECK(config.IsOk());
  return config;
}
```

音频解码器工厂从各个解码器中获得它们的描述，如 OPUS 音频解码器返回自己的描述的过程如下：
```
#0  webrtc::AudioDecoderOpus::AppendSupportedDecoders(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> >*)
    (specs=0x55555843dab0) at ../../api/audio_codecs/opus/audio_decoder_opus.cc:62
#1  0x000055555606b866 in webrtc::audio_decoder_factory_template_impl::Helper<webrtc::AudioDecoderOpus, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioDecoderMultiChannelOpus>, webrtc::AudioDecoderIsacFloat, webrtc::AudioDecoderG722, webrtc::AudioDecoderIlbc, webrtc::AudioDecoderG711, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioDecoderL16> >::AppendSupportedDecoders(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> >*) (specs=0x7fffdabf83d0) at ../../api/audio_codecs/audio_decoder_factory_template.h:45
#2  0x000055555606b7bc in webrtc::audio_decoder_factory_template_impl::AudioDecoderFactoryT<webrtc::AudioDecoderOpus, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioDecoderMultiChannelOpus>, webrtc::AudioDecoderIsacFloat, webrtc::AudioDecoderG722, webrtc::AudioDecoderIlbc, webrtc::AudioDecoderG711, webrtc::(anonymous namespace)::NotAdvertised<webrtc::AudioDecoderL16> >::GetSupportedDecoders() (this=0x55555851a1b0)
    at ../../api/audio_codecs/audio_decoder_factory_template.h:70
#3  0x00005555560da729 in cricket::WebRtcVoiceEngine::Init() (this=0x55555843daa0) at ../../media/engine/webrtc_voice_engine.cc:358
#4  0x000055555725fd64 in cricket::CompositeMediaEngine::Init() (this=0x5555587c1fb0) at ../../media/base/media_engine.cc:172
```

OPUS 音频解码器的描述的详细内容可以参考如下这段代码 (`webrtc/src/api/audio_codecs/opus/audio_decoder_opus.cc`)：
```
void AudioDecoderOpus::AppendSupportedDecoders(
    std::vector<AudioCodecSpec>* specs) {
  AudioCodecInfo opus_info{48000, 1, 64000, 6000, 510000};
  opus_info.allow_comfort_noise = false;
  opus_info.supports_network_adaption = true;
  SdpAudioFormat opus_format(
      {"opus", 48000, 2, {{"minptime", "10"}, {"useinbandfec", "1"}}});
  specs->push_back({std::move(opus_format), opus_info});
}
```

***WebRTC 的这段代码有点奇怪，`AudioCodecInfo` 的描述和 `SdpAudioFormat` 描述的通道数竟然是不一样的。***

`cricket::WebRtcVoiceEngine::Init()` 通过 `WebRtcVoiceEngine` 的 `CollectCodecs()` 函数将上面获取的音频 codec 描述与 payload type 建立关联，完善媒体参数，建立适当的音频 codec 的描述 `AudioCodec`，并保存起来：
```
#0  cricket::WebRtcVoiceEngine::CollectCodecs(std::vector<webrtc::AudioCodecSpec, std::allocator<webrtc::AudioCodecSpec> > const&) const
    (this=0x7fffdabf80c0, specs=std::vector of length 12838888471628541, capacity -1039078572199040 = {...}) at ../../media/engine/webrtc_voice_engine.cc:725
#1  0x00005555560da304 in cricket::WebRtcVoiceEngine::Init() (this=0x555558471e00) at ../../media/engine/webrtc_voice_engine.cc:352
#2  0x000055555725fd64 in cricket::CompositeMediaEngine::Init() (this=0x555558823740) at ../../media/base/media_engine.cc:172
```

Payload type 是媒体数据收发双方的协议和约定，收发双方约定用 payload type 这样一个整数来表示一组媒体参数，如用 111 表示采样率为 48kHz，通道数为 2，编码帧时长为 10ms 这样一组参数。Payload type 后续在编码数据传输时由 RTP 包携带，以帮助接收解码端选择适当的解码器。`WebRtcVoiceEngine::CollectCodecs()` 函数的实现如下：
```
std::vector<AudioCodec> WebRtcVoiceEngine::CollectCodecs(
    const std::vector<webrtc::AudioCodecSpec>& specs) const {
  PayloadTypeMapper mapper;
  std::vector<AudioCodec> out;

  // Only generate CN payload types for these clockrates:
  std::map<int, bool, std::greater<int>> generate_cn = {
      {8000, false}, {16000, false}, {32000, false}};
  // Only generate telephone-event payload types for these clockrates:
  std::map<int, bool, std::greater<int>> generate_dtmf = {
      {8000, false}, {16000, false}, {32000, false}, {48000, false}};

  auto map_format = [&mapper](const webrtc::SdpAudioFormat& format,
                              std::vector<AudioCodec>* out) {
    absl::optional<AudioCodec> opt_codec = mapper.ToAudioCodec(format);
    if (opt_codec) {
      if (out) {
        out->push_back(*opt_codec);
      }
    } else {
      RTC_LOG(LS_ERROR) << "Unable to assign payload type to format: "
                        << rtc::ToString(format);
    }

    return opt_codec;
  };

  for (const auto& spec : specs) {
    // We need to do some extra stuff before adding the main codecs to out.
    absl::optional<AudioCodec> opt_codec = map_format(spec.format, nullptr);
    if (opt_codec) {
      AudioCodec& codec = *opt_codec;
      if (spec.info.supports_network_adaption) {
        codec.AddFeedbackParam(
            FeedbackParam(kRtcpFbParamTransportCc, kParamValueEmpty));
      }

      if (spec.info.allow_comfort_noise) {
        // Generate a CN entry if the decoder allows it and we support the
        // clockrate.
        auto cn = generate_cn.find(spec.format.clockrate_hz);
        if (cn != generate_cn.end()) {
          cn->second = true;
        }
      }

      // Generate a telephone-event entry if we support the clockrate.
      auto dtmf = generate_dtmf.find(spec.format.clockrate_hz);
      if (dtmf != generate_dtmf.end()) {
        dtmf->second = true;
      }

      out.push_back(codec);

      if (codec.name == kOpusCodecName && audio_red_for_opus_enabled_) {
        std::string redFmtp =
            rtc::ToString(codec.id) + "/" + rtc::ToString(codec.id);
        map_format({kRedCodecName, 48000, 2, {{"", redFmtp}}}, &out);
      }
    }
  }

  // Add CN codecs after "proper" audio codecs.
  for (const auto& cn : generate_cn) {
    if (cn.second) {
      map_format({kCnCodecName, cn.first, 1}, &out);
    }
  }

  // Add telephone-event codecs last.
  for (const auto& dtmf : generate_dtmf) {
    if (dtmf.second) {
      map_format({kDtmfCodecName, dtmf.first, 1}, &out);
    }
  }

  return out;
}
```

`PeerConnection` 在创建及初始化的过程中，会创建 `SdpOfferAnswerHandler`/`WebRtcSessionDescriptionFactory`/`MediaSessionDescriptionFactory`，`MediaSessionDescriptionFactory` 在创建时通过 `cricket::ChannelManager` 从 `WebRtcVoiceEngine` 获得音频 codec 及其参数的描述：
```
#0  cricket::WebRtcVoiceEngine::send_codecs() const (this=0x5555587c1fc0) at ../../media/engine/webrtc_voice_engine.cc:654
#1  0x0000555556b8e5f7 in cricket::ChannelManager::GetSupportedAudioSendCodecs(std::vector<cricket::AudioCodec, std::allocator<cricket::AudioCodec> >*) const (this=0x7fffc408bfc0, codecs=0x7fffe40058f8) at ../../pc/channel_manager.cc:68
#2  0x0000555556bb77a2 in cricket::MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(cricket::ChannelManager*, cricket::TransportDescriptionFactory const*, rtc::UniqueRandomIdGenerator*)
    (this=0x7fffe40058f0, channel_manager=0x7fffc408bfc0, transport_desc_factory=0x7fffe40058e0, ssrc_generator=0x7fffe4005530)
    at ../../pc/media_session.cc:1563
#3  0x00005555562ce24d in webrtc::WebRtcSessionDescriptionFactory::WebRtcSessionDescriptionFactory(rtc::Thread*, cricket::ChannelManager*, webrtc::SdpStateProvider const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool, std::unique_ptr<rtc::RTCCertificateGeneratorInterface, std::default_delete<rtc::RTCCertificateGeneratorInterface> >, rtc::scoped_refptr<rtc::RTCCertificate> const&, rtc::UniqueRandomIdGenerator*, std::function<void (rtc::scoped_refptr<rtc::RTCCertificate> const&)>)
    (this=0x7fffe4005830, signaling_thread=0x5555587b4430, channel_manager=0x7fffc408bfc0, sdp_info=0x7fffe40052d0, session_id="4462420193833020990", dtls_enabled=true, cert_generator=std::unique_ptr<rtc::RTCCertificateGeneratorInterface> = {...}, certificate=..., ssrc_generator=0x7fffe4005530, on_certificate_ready=...) at ../../pc/webrtc_session_description_factory.cc:151
#4  0x000055555627963b in std::make_unique<webrtc::WebRtcSessionDescriptionFactory, rtc::Thread*, cricket::ChannelManager*, webrtc::SdpOfferAnswerHandler*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, bool, std::unique_ptr<rtc::RTCCertificateGeneratorInterface, std::default_delete<rtc::RTCCertificateGeneratorInterface> >, rtc::scoped_refptr<rtc::RTCCertificate>&, rtc::UniqueRandomIdGenerator*, webrtc::SdpOfferAnswerHandler::Initialize(const webrtc::PeerConnectionInterface::RTCConfiguration&, webrtc::PeerConnectionDependencies&)::<lambda(const rtc::scoped_refptr<rtc::RTCCertificate>&)> >(void) () at /usr/include/c++/9/bits/unique_ptr.h:857
#5  0x000055555624a7f1 in webrtc::SdpOfferAnswerHandler::Initialize(webrtc::PeerConnectionInterface::RTCConfiguration const&, webrtc::PeerConnectionDependencies&) (this=0x7fffe40052d0, configuration=..., dependencies=...) at ../../pc/sdp_offer_answer.cc:1017
#6  0x000055555624a3e2 in webrtc::SdpOfferAnswerHandler::Create(webrtc::PeerConnection*, webrtc::PeerConnectionInterface::RTCConfiguration const&, webrtc::PeerConnectionDependencies&) (pc=0x7fffe4004530, configuration=..., dependencies=...) at ../../pc/sdp_offer_answer.cc:982
#7  0x00005555561bc600 in webrtc::PeerConnection::Initialize(webrtc::PeerConnectionInterface::RTCConfiguration const&, webrtc::PeerConnectionDependencies)
    (this=0x7fffe4004530, configuration=..., dependencies=...) at ../../pc/peer_connection.cc:624
#8  0x00005555561ba211 in webrtc::PeerConnection::Create(rtc::scoped_refptr<webrtc::ConnectionContext>, webrtc::PeerConnectionFactoryInterface::Options const&, std::unique_ptr<webrtc::RtcEventLog, std::default_delete<webrtc::RtcEventLog> >, std::unique_ptr<webrtc::Call, std::default_delete<webrtc::Call> >, webrtc::PeerConnectionInterface::RTCConfiguration const&, webrtc::PeerConnectionDependencies)
    (context=..., options=..., event_log=std::unique_ptr<webrtc::RtcEventLog> = {...}, call=
    std::unique_ptr<webrtc::Call> = {...}, configuration=..., dependencies=...) at ../../pc/peer_connection.cc:474
#9  0x0000555556157b4c in webrtc::PeerConnectionFactory::CreatePeerConnectionOrError(webrtc::PeerConnectionInterface::RTCConfiguration const&, webrtc::PeerConnectionDependencies) (this=0x7fffe4003e40, configuration=..., dependencies=...) at ../../pc/peer_connection_factory.cc:246
```

在 `MediaSessionDescriptionFactory` 创建过程中还会进一步对获取的音频编码器和解码器信息做一些处理和分类：
```
MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(
    ChannelManager* channel_manager,
    const TransportDescriptionFactory* transport_desc_factory,
    rtc::UniqueRandomIdGenerator* ssrc_generator)
    : MediaSessionDescriptionFactory(transport_desc_factory, ssrc_generator) {
  channel_manager->GetSupportedAudioSendCodecs(&audio_send_codecs_);
  channel_manager->GetSupportedAudioReceiveCodecs(&audio_recv_codecs_);
  channel_manager->GetSupportedVideoSendCodecs(&video_send_codecs_);
  channel_manager->GetSupportedVideoReceiveCodecs(&video_recv_codecs_);
  ComputeAudioCodecsIntersectionAndUnion();
  ComputeVideoCodecsIntersectionAndUnion();
}
 . . . . . .
void MediaSessionDescriptionFactory::ComputeAudioCodecsIntersectionAndUnion() {
  audio_sendrecv_codecs_.clear();
  all_audio_codecs_.clear();
  // Compute the audio codecs union.
  for (const AudioCodec& send : audio_send_codecs_) {
    all_audio_codecs_.push_back(send);
    if (!FindMatchingCodec<AudioCodec>(audio_send_codecs_, audio_recv_codecs_,
                                       send, nullptr)) {
      // It doesn't make sense to have an RTX codec we support sending but not
      // receiving.
      RTC_DCHECK(!IsRtxCodec(send));
    }
  }
  for (const AudioCodec& recv : audio_recv_codecs_) {
    if (!FindMatchingCodec<AudioCodec>(audio_recv_codecs_, audio_send_codecs_,
                                       recv, nullptr)) {
      all_audio_codecs_.push_back(recv);
    }
  }
  // Use NegotiateCodecs to merge our codec lists, since the operation is
  // essentially the same. Put send_codecs as the offered_codecs, which is the
  // order we'd like to follow. The reasoning is that encoding is usually more
  // expensive than decoding, and prioritizing a codec in the send list probably
  // means it's a codec we can handle efficiently.
  NegotiateCodecs(audio_recv_codecs_, audio_send_codecs_,
                  &audio_sendrecv_codecs_, true);
}
```

ICE 连接建立，创建 Offer 消息的时候，获得音频 codec 的信息，并构造 SDP 消息，如：
```
#0  cricket::MediaSessionDescriptionFactory::GetAudioCodecsForOffer(webrtc::RtpTransceiverDirection const&) const
    (this=0x5555560afd88 <std::_Rb_tree<int, int, std::_Identity<int>, std::less<int>, std::allocator<int> >::~_Rb_tree()+58>, direction=@0x7fffeeffb750: -285230240) at ../../pc/media_session.cc:1974
#1  0x0000555556bbb036 in cricket::MediaSessionDescriptionFactory::AddAudioContentForOffer(cricket::MediaDescriptionOptions const&, cricket::MediaSessionOptions const&, cricket::ContentInfo const*, cricket::SessionDescription const*, std::vector<webrtc::RtpExtension, std::allocator<webrtc::RtpExtension> > const&, std::vector<cricket::AudioCodec, std::allocator<cricket::AudioCodec> > const&, std::vector<cricket::StreamParams, std::allocator<cricket::StreamParams> >*, cricket::SessionDescription*, cricket::IceCredentialsIterator*) const
    (this=0x7fffe40058f0, media_description_options=..., session_options=..., current_content=0x0, current_description=0x0, audio_rtp_extensions=std::vector of length 6, capacity 8 = {...}, audio_codecs=
    std::vector of length 15, capacity 16 = {...}, current_streams=0x7fffeeffbae0, desc=0x7fffe401ad70, ice_credentials=0x7fffeeffbb60)
    at ../../pc/media_session.cc:2283
#2  0x0000555556bb808f in cricket::MediaSessionDescriptionFactory::CreateOffer(cricket::MediaSessionOptions const&, cricket::SessionDescription const*) const (this=0x7fffe40058f0, session_options=..., current_description=0x0) at ../../pc/media_session.cc:1684
#3  0x00005555562d0552 in webrtc::WebRtcSessionDescriptionFactory::InternalCreateOffer(webrtc::CreateSessionDescriptionRequest)
    (this=0x7fffe4005830, request=...) at ../../pc/webrtc_session_description_factory.cc:349
#4  0x00005555562cf5a9 in webrtc::WebRtcSessionDescriptionFactory::CreateOffer(webrtc::CreateSessionDescriptionObserver*, webrtc::PeerConnectionInterface::RTCOfferAnswerOptions const&, cricket::MediaSessionOptions const&) (this=0x7fffe4005830, observer=0x5555589b1130, options=..., session_options=...)
    at ../../pc/webrtc_session_description_factory.cc:247
#5  0x0000555556253881 in webrtc::SdpOfferAnswerHandler::DoCreateOffer(webrtc::PeerConnectionInterface::RTCOfferAnswerOptions const&, rtc::scoped_refptr<webrtc::CreateSessionDescriptionObserver>) (this=0x7fffe40052d0, options=..., observer=...) at ../../pc/sdp_offer_answer.cc:2037
#6  0x000055555624b0f7 in webrtc::SdpOfferAnswerHandler::<lambda(std::function<void()>)>::operator()(std::function<void()>) const
    (__closure=0x7fffeeffc320, operations_chain_callback=...) at ../../pc/sdp_offer_answer.cc:1132
#7  0x0000555556282b8b in rtc::rtc_operations_chain_internal::OperationWithFunctor<webrtc::SdpOfferAnswerHandler::CreateOffer(webrtc::CreateSessionDescriptionObserver*, const webrtc::PeerConnectionInterface::RTCOfferAnswerOptions&)::<lambda(std::function<void()>)> >::Run(void) (this=0x7fffe4011260)
    at ../../rtc_base/operations_chain.h:71
#8  0x00005555562798e4 in rtc::OperationsChain::ChainOperation<webrtc::SdpOfferAnswerHandler::CreateOffer(webrtc::CreateSessionDescriptionObserver*, const webrtc::PeerConnectionInterface::RTCOfferAnswerOptions&)::<lambda(std::function<void()>)> >(webrtc::SdpOfferAnswerHandler::<lambda(std::function<void()>)> &&) (this=0x7fffe40055f0, functor=...) at ../../rtc_base/operations_chain.h:154
#9  0x000055555624b37d in webrtc::SdpOfferAnswerHandler::CreateOffer(webrtc::CreateSessionDescriptionObserver*, webrtc::PeerConnectionInterface::RTCOfferAnswerOptions const&) (this=0x7fffe40052d0, observer=0x555558a40468, options=...) at ../../pc/sdp_offer_answer.cc:1115
#10 0x00005555561cc7bd in webrtc::PeerConnection::CreateOffer(webrtc::CreateSessionDescriptionObserver*, webrtc::PeerConnectionInterface::RTCOfferAnswerOptions const&) (this=0x7fffe4004530, observer=0x555558a40468, options=...) at ../../pc/peer_connection.cc:1323
```

视频和音频稍微有一点不一样。

视频的 codec 描述信息，在 `PeerConnection` 创建及初始化的过程中，创建 `SdpOfferAnswerHandler`/`WebRtcSessionDescriptionFactory`/`MediaSessionDescriptionFactory`，`MediaSessionDescriptionFactory` 在创建时通过 `cricket::ChannelManager` 从 `WebRtcVideoEngine` 获得视频 codec 及其参数的描述：
```
#0  cricket::WebRtcVideoEngine::send_codecs() const (this=0x7fffef7fc9c0) at ../../media/engine/webrtc_video_engine.cc:647
#1  0x0000555556b8e6de in cricket::ChannelManager::GetSupportedVideoSendCodecs(std::vector<cricket::VideoCodec, std::allocator<cricket::VideoCodec> >*) const (this=0x7fffc408c160, codecs=0x7fffe0005948) at ../../pc/channel_manager.cc:86
#2  0x0000555556bb77d0 in cricket::MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(cricket::ChannelManager*, cricket::TransportDescriptionFactory const*, rtc::UniqueRandomIdGenerator*)
    (this=0x7fffe00058e0, channel_manager=0x7fffc408c160, transport_desc_factory=0x7fffe00058d0, ssrc_generator=0x7fffe0005520)
    at ../../pc/media_session.cc:1565
#3  0x00005555562ce24d in webrtc::WebRtcSessionDescriptionFactory::WebRtcSessionDescriptionFactory(rtc::Thread*, cricket::ChannelManager*, webrtc::SdpStateProvider const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool, std::unique_ptr<rtc::RTCCertificateGeneratorInterface, std::default_delete<rtc::RTCCertificateGeneratorInterface> >, rtc::scoped_refptr<rtc::RTCCertificate> const&, rtc::UniqueRandomIdGenerator*, std::function<void (rtc::scoped_refptr<rtc::RTCCertificate> const&)>)
    (this=0x7fffe0005820, signaling_thread=0x555558802600, channel_manager=0x7fffc408c160, sdp_info=0x7fffe00052c0, session_id="6661147428887955636", dtls_enabled=true, cert_generator=std::unique_ptr<rtc::RTCCertificateGeneratorInterface> = {...}, certificate=..., ssrc_generator=0x7fffe0005520, on_certificate_ready=...) at ../../pc/webrtc_session_description_factory.cc:151
```

视频的 codec 描述信息并不是预先构造的，而是在获取的时候构造的：
```
// This function will assign dynamic payload types (in the range [96, 127]
// and then [35, 63]) to the input codecs, and also add ULPFEC, RED, FlexFEC,
// and associated RTX codecs for recognized codecs (VP8, VP9, H264, and RED).
// It will also add default feedback params to the codecs.
// is_decoder_factory is needed to keep track of the implict assumption that any
// H264 decoder also supports constrained base line profile.
// Also, is_decoder_factory is used to decide whether FlexFEC video format
// should be advertised as supported.
// TODO(kron): Perhaps it is better to move the implicit knowledge to the place
// where codecs are negotiated.
template <class T>
std::vector<VideoCodec> GetPayloadTypesAndDefaultCodecs(
    const T* factory,
    bool is_decoder_factory,
    const webrtc::WebRtcKeyValueConfig& trials) {
  if (!factory) {
    return {};
  }

  std::vector<webrtc::SdpVideoFormat> supported_formats =
      factory->GetSupportedFormats();
  if (is_decoder_factory) {
    AddH264ConstrainedBaselineProfileToSupportedFormats(&supported_formats);
  }

  if (supported_formats.empty())
    return std::vector<VideoCodec>();

  supported_formats.push_back(webrtc::SdpVideoFormat(kRedCodecName));
  supported_formats.push_back(webrtc::SdpVideoFormat(kUlpfecCodecName));

  // flexfec-03 is supported as
  // - receive codec unless WebRTC-FlexFEC-03-Advertised is disabled
  // - send codec if WebRTC-FlexFEC-03-Advertised is enabled
  if ((is_decoder_factory &&
       !IsDisabled(trials, "WebRTC-FlexFEC-03-Advertised")) ||
      (!is_decoder_factory &&
       IsEnabled(trials, "WebRTC-FlexFEC-03-Advertised"))) {
    webrtc::SdpVideoFormat flexfec_format(kFlexfecCodecName);
    // This value is currently arbitrarily set to 10 seconds. (The unit
    // is microseconds.) This parameter MUST be present in the SDP, but
    // we never use the actual value anywhere in our code however.
    // TODO(brandtr): Consider honouring this value in the sender and receiver.
    flexfec_format.parameters = {{kFlexfecFmtpRepairWindow, "10000000"}};
    supported_formats.push_back(flexfec_format);
  }

  // Due to interoperability issues with old Chrome/WebRTC versions that
  // ignore the [35, 63] range prefer the lower range for new codecs.
  static const int kFirstDynamicPayloadTypeLowerRange = 35;
  static const int kLastDynamicPayloadTypeLowerRange = 63;

  static const int kFirstDynamicPayloadTypeUpperRange = 96;
  static const int kLastDynamicPayloadTypeUpperRange = 127;
  int payload_type_upper = kFirstDynamicPayloadTypeUpperRange;
  int payload_type_lower = kFirstDynamicPayloadTypeLowerRange;

  std::vector<VideoCodec> output_codecs;
  for (const webrtc::SdpVideoFormat& format : supported_formats) {
    VideoCodec codec(format);
    bool isCodecValidForLowerRange =
        absl::EqualsIgnoreCase(codec.name, kFlexfecCodecName) ||
        absl::EqualsIgnoreCase(codec.name, kAv1CodecName) ||
        absl::EqualsIgnoreCase(codec.name, kAv1xCodecName);
    bool isFecCodec = absl::EqualsIgnoreCase(codec.name, kUlpfecCodecName) ||
                      absl::EqualsIgnoreCase(codec.name, kFlexfecCodecName);

    // Check if we ran out of payload types.
    if (payload_type_lower > kLastDynamicPayloadTypeLowerRange) {
      // TODO(https://bugs.chromium.org/p/webrtc/issues/detail?id=12248):
      // return an error.
      RTC_LOG(LS_ERROR) << "Out of dynamic payload types [35,63] after "
                           "fallback from [96, 127], skipping the rest.";
      RTC_DCHECK_EQ(payload_type_upper, kLastDynamicPayloadTypeUpperRange);
      break;
    }

    // Lower range gets used for "new" codecs or when running out of payload
    // types in the upper range.
    if (isCodecValidForLowerRange ||
        payload_type_upper >= kLastDynamicPayloadTypeUpperRange) {
      codec.id = payload_type_lower++;
    } else {
      codec.id = payload_type_upper++;
    }
    AddDefaultFeedbackParams(&codec, trials);
    output_codecs.push_back(codec);

    // Add associated RTX codec for non-FEC codecs.
    if (!isFecCodec) {
      // Check if we ran out of payload types.
      if (payload_type_lower > kLastDynamicPayloadTypeLowerRange) {
        // TODO(https://bugs.chromium.org/p/webrtc/issues/detail?id=12248):
        // return an error.
        RTC_LOG(LS_ERROR) << "Out of dynamic payload types [35,63] after "
                             "fallback from [96, 127], skipping the rest.";
        RTC_DCHECK_EQ(payload_type_upper, kLastDynamicPayloadTypeUpperRange);
        break;
      }
      if (isCodecValidForLowerRange ||
          payload_type_upper >= kLastDynamicPayloadTypeUpperRange) {
        output_codecs.push_back(
            VideoCodec::CreateRtxCodec(payload_type_lower++, codec.id));
      } else {
        output_codecs.push_back(
            VideoCodec::CreateRtxCodec(payload_type_upper++, codec.id));
      }
    }
  }
  return output_codecs;
}
 . . . . . .
std::vector<VideoCodec> WebRtcVideoEngine::send_codecs() const {
  return GetPayloadTypesAndDefaultCodecs(encoder_factory_.get(),
                                         /*is_decoder_factory=*/false, trials_);
}

std::vector<VideoCodec> WebRtcVideoEngine::recv_codecs() const {
  return GetPayloadTypesAndDefaultCodecs(decoder_factory_.get(),
                                         /*is_decoder_factory=*/true, trials_);
}
```

与音频类似，构造视频的 codec 描述时，同样会首先从编码器或者解码器工厂中获得编解码器的信息。视频的编码器工厂和解码器工厂，同样是在创建 `PeerConnectionFactoryInterface` 的时候，由 webrtc 的用户传入。创建内建的视频编码器工厂的 `webrtc::CreateBuiltinVideoEncoderFactory()` 函数实现 (`webrtc/src/api/video_codecs/builtin_video_encoder_factory.cc`) 如下：
```
namespace webrtc {

namespace {

// This class wraps the internal factory and adds simulcast.
class BuiltinVideoEncoderFactory : public VideoEncoderFactory {
 public:
  BuiltinVideoEncoderFactory()
      : internal_encoder_factory_(new InternalEncoderFactory()) {}

  VideoEncoderFactory::CodecInfo QueryVideoEncoder(
      const SdpVideoFormat& format) const override {
    // Format must be one of the internal formats.
    RTC_DCHECK(
        format.IsCodecInList(internal_encoder_factory_->GetSupportedFormats()));
    VideoEncoderFactory::CodecInfo info;
    return info;
  }

  std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) override {
    // Try creating internal encoder.
    std::unique_ptr<VideoEncoder> internal_encoder;
    if (format.IsCodecInList(
            internal_encoder_factory_->GetSupportedFormats())) {
      internal_encoder = std::make_unique<EncoderSimulcastProxy>(
          internal_encoder_factory_.get(), format);
    }

    return internal_encoder;
  }

  std::vector<SdpVideoFormat> GetSupportedFormats() const override {
    return internal_encoder_factory_->GetSupportedFormats();
  }

 private:
  const std::unique_ptr<VideoEncoderFactory> internal_encoder_factory_;
};

}  // namespace

std::unique_ptr<VideoEncoderFactory> CreateBuiltinVideoEncoderFactory() {
  return std::make_unique<BuiltinVideoEncoderFactory>();
}

}  // namespace webrtc
```

创建内建的视频解码器工厂的 `webrtc::CreateBuiltinVideoDecoderFactory()` 函数实现 (`webrtc/src/api/video_codecs/builtin_video_decoder_factory.cc`) 如下：
```
namespace webrtc {

std::unique_ptr<VideoDecoderFactory> CreateBuiltinVideoDecoderFactory() {
  return std::make_unique<InternalDecoderFactory>();
}

}  // namespace webrtc
```

WebRTC 提供的视频编码器工厂和视频解码器工厂实现分别为 `InternalEncoderFactory` 和 `InternalDecoderFactory`。

`WebRtcVideoEngine` 通过 `GetPayloadTypesAndDefaultCodecs()` 函数从这些工厂中获得 codec 描述，如对于视频编码器：
```
#0  webrtc::InternalEncoderFactory::GetSupportedFormats() const (this=0x7fffc4003141) at ../../media/engine/internal_encoder_factory.cc:40
#1  0x000055555606fb02 in webrtc::(anonymous namespace)::BuiltinVideoEncoderFactory::GetSupportedFormats() const (this=0x5555585e9580)
    at ../../api/video_codecs/builtin_video_encoder_factory.cc:58
#2  0x000055555609df41 in cricket::(anonymous namespace)::GetPayloadTypesAndDefaultCodecs<webrtc::VideoEncoderFactory>(webrtc::VideoEncoderFactory const*, bool, webrtc::WebRtcKeyValueConfig const&) (factory=0x5555585e9580, is_decoder_factory=false, trials=...) at ../../media/engine/webrtc_video_engine.cc:132
#3  0x000055555607b5ce in cricket::WebRtcVideoEngine::send_codecs() const (this=0x5555587c1bd0) at ../../media/engine/webrtc_video_engine.cc:649
``` 

详细的视频编码器描述如下：
```
std::vector<SdpVideoFormat> InternalEncoderFactory::SupportedFormats() {
  std::vector<SdpVideoFormat> supported_codecs;
  supported_codecs.push_back(SdpVideoFormat(cricket::kVp8CodecName));
  for (const webrtc::SdpVideoFormat& format : webrtc::SupportedVP9Codecs())
    supported_codecs.push_back(format);
  for (const webrtc::SdpVideoFormat& format : webrtc::SupportedH264Codecs())
    supported_codecs.push_back(format);
  if (kIsLibaomAv1EncoderSupported)
    supported_codecs.push_back(SdpVideoFormat(cricket::kAv1CodecName));
  return supported_codecs;
}

std::vector<SdpVideoFormat> InternalEncoderFactory::GetSupportedFormats()
    const {
  return SupportedFormats();
}
```

对于视频解码器则如：
```
#0  webrtc::InternalDecoderFactory::GetSupportedFormats() const (this=0x7fffef7fc4c0) at ../../media/engine/internal_decoder_factory.cc:28
#1  0x000055555609eedd in cricket::(anonymous namespace)::GetPayloadTypesAndDefaultCodecs<webrtc::VideoDecoderFactory>(webrtc::VideoDecoderFactory const*, bool, webrtc::WebRtcKeyValueConfig const&) (factory=0x55555890dd40, is_decoder_factory=true, trials=...) at ../../media/engine/webrtc_video_engine.cc:132
#2  0x000055555607b61e in cricket::WebRtcVideoEngine::recv_codecs() const (this=0x5555587c1bd0) at ../../media/engine/webrtc_video_engine.cc:654
```

详细的视频解码器描述如下：
```
std::vector<SdpVideoFormat> InternalDecoderFactory::GetSupportedFormats()
    const {
  std::vector<SdpVideoFormat> formats;
  formats.push_back(SdpVideoFormat(cricket::kVp8CodecName));
  for (const SdpVideoFormat& format : SupportedVP9DecoderCodecs())
    formats.push_back(format);
  for (const SdpVideoFormat& h264_format : SupportedH264Codecs())
    formats.push_back(h264_format);
  if (kIsLibaomAv1DecoderSupported)
    formats.push_back(SdpVideoFormat(cricket::kAv1CodecName));
  return formats;
}
```

通过分析 WebRTC 中收集音视频编解码能力的过程，我们大致也可以了解在 WebRTC 中增删改查音视频编解码器的过程。
