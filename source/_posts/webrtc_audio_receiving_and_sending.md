---
title: WebRTC Audio 接收和发送的关键过程
date: 2019-07-20 23:05:49
categories: 音视频开发
tags:
- 音视频开发
---

本文基于 WebRTC 中的示例应用 peerconnection_client 分析 WebRTC Audio 接收和发送的关键过程。首先是发送的过程，然后是接收的过程。
<!--more-->
# 创建 webrtc::AudioState

应用程序择机初始化 `PeerConnectionFactory`：
```
#0  Init () at webrtc/src/pc/channel_manager.cc:121
#1  Initialize () at webrtc/src/pc/peer_connection_factory.cc:139
#6  webrtc::CreateModularPeerConnectionFactory(webrtc::PeerConnectionFactoryDependencies) () at webrtc/src/pc/peer_connection_factory.cc:55
#7  webrtc::CreatePeerConnectionFactory(rtc::Thread*, rtc::Thread*, rtc::Thread*, rtc::scoped_refptr<webrtc::AudioDeviceModule>, rtc::scoped_refptr<webrtc::AudioEncoderFactory>, rtc::scoped_refptr<webrtc::AudioDecoderFactory>, std::__1::unique_ptr<webrtc::VideoEncoderFactory, std::__1::default_delete<webrtc::VideoEncoderFactory> >, std::__1::unique_ptr<webrtc::VideoDecoderFactory, std::__1::default_delete<webrtc::VideoDecoderFactory> >, rtc::scoped_refptr<webrtc::AudioMixer>, rtc::scoped_refptr<webrtc::AudioProcessing>) () at webrtc/src/api/create_peerconnection_factory.cc:65
#8  InitializePeerConnection () at webrtc/src/examples/peerconnection/client/conductor.cc:132
#9  ConnectToPeer () at webrtc/src/examples/peerconnection/client/conductor.cc:422
#10 OnRowActivated () at webrtc/src/examples/peerconnection/client/linux/main_wnd.cc:433
#11 (anonymous namespace)::OnRowActivatedCallback(_GtkTreeView*, _GtkTreePath*, _GtkTreeViewColumn*, void*) () at webrtc/src/examples/peerconnection/client/linux/main_wnd.cc:70
```

由 `Conductor::InitializePeerConnection()` 的代码可知，`PeerConnectionFactory` 竟然是随同 peer connection一起创建的。

在 `webrtc/src/pc/channel_manager.cc` 文件里定义的 `ChannelManager::Init()` 函数中会在另一个线程中起一个 task，调用 `media_engine_->Init()`，完成媒体引擎的初始化：
```
bool ChannelManager::Init() {
  RTC_DCHECK(!initialized_);
  if (initialized_) {
    return false;
  }
  RTC_DCHECK(network_thread_);
  RTC_DCHECK(worker_thread_);
  if (!network_thread_->IsCurrent()) {
    // Do not allow invoking calls to other threads on the network thread.
    network_thread_->Invoke<void>(
        RTC_FROM_HERE, [&] { network_thread_->DisallowBlockingCalls(); });
  }

  if (media_engine_) {
    initialized_ = worker_thread_->Invoke<bool>(
        RTC_FROM_HERE, [&] { return media_engine_->Init(); });
    RTC_DCHECK(initialized_);
  } else {
    initialized_ = true;
  }
  return initialized_;
}
```

媒体引擎初始化过程中，将会创建 AudioState：
```
#0  webrtc::AudioState::Create(webrtc::AudioState::Config const&) () at webrtc/src/audio/audio_state.cc:188
#1  Init () at webrtc/src/media/engine/webrtc_voice_engine.cc:260
#2  cricket::CompositeMediaEngine::Init() () at webrtc/src/media/base/media_engine.cc:155
#3  cricket::ChannelManager::Init()::$_3::operator()() const () at webrtc/src/pc/channel_manager.cc:135
```

AudioState 伴随着 `PeerConnectionFactory` 、`ChannelManager` 和 `MediaEngine` 的创建及初始化一起创建。

# 创建 WebRTC Call

应用程序根据需要创建 peer connection：
```
#0  CreatePeerConnection () at webrtc/src/pc/peer_connection_factory.cc:240
#1  webrtc::PeerConnectionFactory::CreatePeerConnection(webrtc::PeerConnectionInterface::RTCConfiguration const&, std::__1::unique_ptr<cricket::PortAllocator, std::__1::default_delete<cricket::PortAllocator> >, std::__1::unique_ptr<rtc::RTCCertificateGeneratorInterface, std::__1::default_delete<rtc::RTCCertificateGeneratorInterface> >, webrtc::PeerConnectionObserver*) () at webrtc/src/pc/peer_connection_factory.cc:233
#7  CreatePeerConnection () at webrtc/src/examples/peerconnection/client/conductor.cc:184
#8  InitializePeerConnection () at webrtc/src/examples/peerconnection/client/conductor.cc:148
#9  ConnectToPeer () at webrtc/src/examples/peerconnection/client/conductor.cc:422
#10 OnRowActivated () at webrtc/src/examples/peerconnection/client/linux/main_wnd.cc:433
#11 (anonymous namespace)::OnRowActivatedCallback(_GtkTreeView*, _GtkTreePath*, _GtkTreeViewColumn*, void*) () at webrtc/src/examples/peerconnection/client/linux/main_wnd.cc:70
```

`PeerConnectionFactory::CreatePeerConnection()` 在创建 connection 时，会在另一个线程中起一个task 来创建 Call：
```
rtc::scoped_refptr<PeerConnectionInterface>
PeerConnectionFactory::CreatePeerConnection(
    const PeerConnectionInterface::RTCConfiguration& configuration,
    PeerConnectionDependencies dependencies) {
  RTC_DCHECK(signaling_thread_->IsCurrent());

  // Set internal defaults if optional dependencies are not set.
  if (!dependencies.cert_generator) {
    dependencies.cert_generator =
        absl::make_unique<rtc::RTCCertificateGenerator>(signaling_thread_,
                                                        network_thread_);
  }
  if (!dependencies.allocator) {
    network_thread_->Invoke<void>(RTC_FROM_HERE, [this, &configuration,
                                                  &dependencies]() {
      dependencies.allocator = absl::make_unique<cricket::BasicPortAllocator>(
          default_network_manager_.get(), default_socket_factory_.get(),
          configuration.turn_customizer);
    });
  }

  // TODO(zstein): Once chromium injects its own AsyncResolverFactory, set
  // |dependencies.async_resolver_factory| to a new
  // |rtc::BasicAsyncResolverFactory| if no factory is provided.

  network_thread_->Invoke<void>(
      RTC_FROM_HERE,
      rtc::Bind(&cricket::PortAllocator::SetNetworkIgnoreMask,
                dependencies.allocator.get(), options_.network_ignore_mask));

  std::unique_ptr<RtcEventLog> event_log =
      worker_thread_->Invoke<std::unique_ptr<RtcEventLog>>(
          RTC_FROM_HERE,
          rtc::Bind(&PeerConnectionFactory::CreateRtcEventLog_w, this));

  std::unique_ptr<Call> call = worker_thread_->Invoke<std::unique_ptr<Call>>(
      RTC_FROM_HERE,
      rtc::Bind(&PeerConnectionFactory::CreateCall_w, this, event_log.get()));

  rtc::scoped_refptr<PeerConnection> pc(
      new rtc::RefCountedObject<PeerConnection>(this, std::move(event_log),
                                                std::move(call)));
  ActionsBeforeInitializeForTesting(pc);
  if (!pc->Initialize(configuration, std::move(dependencies))) {
    return nullptr;
  }
  return PeerConnectionProxy::Create(signaling_thread(), pc);
}
```

创建 WebRTC Call 的过程如下：
```
#0  webrtc::Call::Create(webrtc::CallConfig const&) () at webrtc/src/call/call.cc:424
#1  webrtc::CallFactory::CreateCall(webrtc::CallConfig const&) () at webrtc/src/call/call_factory.cc:84
#2  CreateCall_w () at webrtc/src/pc/peer_connection_factory.cc:364
```

不难看出，在 WebRTC 中，Call 是 per peer connection 的。

为 WebRTC Call 注入的 AudioState 来自于全局的 MediaEngine 的 VoiceEngine。AudioState 是全局的，而 Call 则是 connection 局部的。

# 创建  WebRtcAudioReceiveStream

WebRTC 应用需要起一个专门的专门的连接，用于接收媒体协商信息。在收到媒体协商信息之后，则将媒体协商信息进行层层传递及处理：
```
#0  cricket::BaseChannel::SetRemoteContent(cricket::MediaContentDescription const*, webrtc::SdpType, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*) () at webrtc/src/pc/channel.cc:299
#1  PushdownMediaDescription () at webrtc/src/pc/peer_connection.cc:5700
#2  UpdateSessionState () at webrtc/src/pc/peer_connection.cc:5668
#3  ApplyRemoteDescription () at webrtc/src/pc/peer_connection.cc:2668
#4  SetRemoteDescription () at webrtc/src/pc/peer_connection.cc:2562
#5  webrtc::PeerConnection::SetRemoteDescription(webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*) () at webrtc/src/pc/peer_connection.cc:2506


#6  void webrtc::ReturnType<void>::Invoke<webrtc::PeerConnectionInterface, void (webrtc::PeerConnectionInterface::*)(webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*), webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*>(webrtc::PeerConnectionInterface*, void (webrtc::PeerConnectionInterface::*)(webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*), webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*) () at webrtc/src/api/proxy.h:131
#7  webrtc::MethodCall2<webrtc::PeerConnectionInterface, void, webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*>::OnMessage(rtc::Message*) () at webrtc/src/api/proxy.h:252
#8  webrtc::internal::SynchronousMethodCall::Invoke(rtc::Location const&, rtc::Thread*) () at webrtc/src/api/proxy.cc:24
#9  webrtc::MethodCall2<webrtc::PeerConnectionInterface, void, webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*>::Marshal(rtc::Location const&, rtc::Thread*) () at webrtc/src/api/proxy.h:246
#10 webrtc::PeerConnectionProxyWithInternal<webrtc::PeerConnectionInterface>::SetRemoteDescription(webrtc::SetSessionDescriptionObserver*, webrtc::SessionDescriptionInterface*) () at webrtc/src/api/peer_connection_proxy.h:101


#11 OnMessageFromPeer () at webrtc/src/examples/peerconnection/client/conductor.cc:351
#12 PeerConnectionClient::OnMessageFromPeer(int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) ()
    at webrtc/src/examples/peerconnection/client/peer_connection_client.cc:250
#13 OnHangingGetRead () at webrtc/src/examples/peerconnection/client/peer_connection_client.cc:403
#18 rtc::SocketDispatcher::OnEvent(unsigned int, int) () at webrtc/src/rtc_base/physical_socket_server.cc:790
#19 rtc::ProcessEvents(rtc::Dispatcher*, bool, bool, bool) () at webrtc/src/rtc_base/physical_socket_server.cc:1379
#20 WaitEpoll () at webrtc/src/rtc_base/physical_socket_server.cc:1620
#21 rtc::PhysicalSocketServer::Wait(int, bool) () at webrtc/src/rtc_base/physical_socket_server.cc:1328
#22 CustomSocketServer::Wait(int, bool) () at webrtc/src/examples/peerconnection/client/linux/main.cc:56
#23 Get () at webrtc/src/rtc_base/message_queue.cc:329
#24 ProcessMessages () at webrtc/src/rtc_base/thread.cc:525
#25 rtc::Thread::Run() () at webrtc/src/rtc_base/thread.cc:351
#26 main () at webrtc/src/examples/peerconnection/client/linux/main.cc:111
```

在 webrtc/src/pc/channel.cc 文件里定义的 `BaseChannel::SetRemoteContent()` 函数中将收到的媒体协商信息，抛给另一个线程进行处理：
```
bool BaseChannel::SetRemoteContent(const MediaContentDescription* content,
                                   SdpType type,
                                   std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetRemoteContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetRemoteContent_w, this, content, type, error_desc));
}
```

`WebRtcAudioReceiveStream` 最终由 webrtc/src/media/engine/webrtc_voice_engine.cc 文件中定义的 `WebRtcVoiceMediaChannel::AddRecvStream()` 创建：
```
#0  AddRecvStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1854
#1  AddRecvStream_w () at webrtc/src/pc/channel.cc:599
#2  UpdateRemoteStreams_w () at webrtc/src/pc/channel.cc:714
#3  SetRemoteContent_w () at webrtc/src/pc/channel.cc:951
```

创建 WebRtcAudioReceiveStream 时，也会一并创建 Call 的 AudioReceiveStream：
```
#0  CreateAudioReceiveStream () at webrtc/src/call/call.cc:779
#1  RecreateAudioReceiveStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1224
#2  WebRtcAudioReceiveStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1090
#3  AddRecvStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1889
#4  AddRecvStream_w () at webrtc/src/pc/channel.cc:599
#5  UpdateRemoteStreams_w () at webrtc/src/pc/channel.cc:714
```

`WebRtcAudioReceiveStream` 创建完成后，随即将其加进 mixer，作为 mixer 的 audio source 之一：
```
#0  AddSource () at webrtc/src/modules/audio_mixer/audio_mixer_impl.cc:160
#1  AddReceivingStream () at webrtc/src/audio/audio_state.cc:60
#2  Start () at webrtc/src/audio/audio_receive_stream.cc:161
#3  SetPlayout () at webrtc/src/media/engine/webrtc_voice_engine.cc:1173
#4  AddRecvStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1899
#5  AddRecvStream_w () at webrtc/src/pc/channel.cc:599
```

# WebRTC 中音频接收处理的关键流程

## 1. 从网络收到 UDP 包
```
#0  OnPacketReceived () at webrtc/src/pc/channel.cc:507
#1  cricket::BaseChannel::OnRtpPacket(webrtc::RtpPacketReceived const&) () at webrtc/src/pc/channel.cc:468
#2  webrtc::RtpDemuxer::OnRtpPacket(webrtc::RtpPacketReceived const&) () at webrtc/src/call/rtp_demuxer.cc:177
#3  DemuxPacket () at webrtc/src/pc/rtp_transport.cc:194
#4  OnRtpPacketReceived () at webrtc/src/pc/srtp_transport.cc:230
#5  OnReadPacket () at webrtc/src/pc/rtp_transport.cc:268
#10 OnReadPacket () at webrtc/src/p2p/base/dtls_transport.cc:600
#15 OnReadPacket () at webrtc/src/p2p/base/p2p_transport_channel.cc:2499
#20 OnReadPacket () at webrtc/src/p2p/base/connection.cc:415
#21 OnReadPacket () at webrtc/src/p2p/base/stun_port.cc:407
#22 cricket::UDPPort::HandleIncomingPacket(rtc::AsyncPacketSocket*, char const*, unsigned long, rtc::SocketAddress const&, long) () at webrtc/src/p2p/base/stun_port.cc:348
#23 OnReadPacket () at webrtc/src/p2p/client/basic_port_allocator.cc:1673
#28 OnReadEvent () at webrtc/src/rtc_base/async_udp_socket.cc:132
#33 rtc::SocketDispatcher::OnEvent(unsigned int, int) () at webrtc/src/rtc_base/physical_socket_server.cc:790
#34 rtc::ProcessEvents(rtc::Dispatcher*, bool, bool, bool) () at webrtc/src/rtc_base/physical_socket_server.cc:1379
#35 WaitEpoll () at webrtc/src/rtc_base/physical_socket_server.cc:1620
#36 rtc::PhysicalSocketServer::Wait(int, bool) () at webrtc/src/rtc_base/physical_socket_server.cc:1328
```

在 webrtc/src/rtc_base/physical_socket_server.cc 文件里 `PhysicalSocketServer` 
 类的 `Wait()` 函数中，通过 epoll 机制等待网络数据包的到来。当数据包到来时，经过层层处理及传递，一直被传到 webrtc/src/pc/channel.cc 文件里 `BaseChannel` 类的 `OnPacketReceived()` 函数中，该函数又将数据包抛进另一个线程进行处理：
```
void BaseChannel::OnPacketReceived(bool rtcp,
                                   const rtc::CopyOnWriteBuffer& packet,
                                   int64_t packet_time_us) {
  if (!has_received_packet_ && !rtcp) {
    has_received_packet_ = true;
    signaling_thread()->Post(RTC_FROM_HERE, this, MSG_FIRSTPACKETRECEIVED);
  }

  if (!srtp_active() && srtp_required_) {
    // Our session description indicates that SRTP is required, but we got a
    // packet before our SRTP filter is active. This means either that
    // a) we got SRTP packets before we received the SDES keys, in which case
    //    we can't decrypt it anyway, or
    // b) we got SRTP packets before DTLS completed on both the RTP and RTCP
    //    transports, so we haven't yet extracted keys, even if DTLS did
    //    complete on the transport that the packets are being sent on. It's
    //    really good practice to wait for both RTP and RTCP to be good to go
    //    before sending  media, to prevent weird failure modes, so it's fine
    //    for us to just eat packets here. This is all sidestepped if RTCP mux
    //    is used anyway.
    RTC_LOG(LS_WARNING)
        << "Can't process incoming "
        << RtpPacketTypeToString(rtcp ? RtpPacketType::kRtcp
                                      : RtpPacketType::kRtp)
        << " packet when SRTP is inactive and crypto is required";
    return;
  }

  invoker_.AsyncInvoke<void>(
      RTC_FROM_HERE, worker_thread_,
      Bind(&BaseChannel::ProcessPacket, this, rtcp, packet, packet_time_us));
}
```

## 2. 媒体引擎对收到的音频包的处理
```
#0  InsertPacketInternal () at webrtc/src/modules/audio_coding/neteq/neteq_impl.cc:467
#1  webrtc::NetEqImpl::InsertPacket(webrtc::RTPHeader const&, rtc::ArrayView<unsigned char const, -4711l>, unsigned int) () at webrtc/src/modules/audio_coding/neteq/neteq_impl.cc:153
#2  InsertPacket () at webrtc/src/modules/audio_coding/acm2/acm_receiver.cc:117
#3  IncomingPacket () at webrtc/src/modules/audio_coding/acm2/audio_coding_module.cc:667
#4  OnReceivedPayloadData () at webrtc/src/audio/channel_receive.cc:283
#5  ReceivePacket () at webrtc/src/audio/channel_receive.cc:669
#6  webrtc::voe::(anonymous namespace)::ChannelReceive::OnRtpPacket(webrtc::RtpPacketReceived const&) () at webrtc/src/audio/channel_receive.cc:622
#7  webrtc::RtpDemuxer::OnRtpPacket(webrtc::RtpPacketReceived const&) () at webrtc/src/call/rtp_demuxer.cc:177
#8  webrtc::RtpStreamReceiverController::OnRtpPacket(webrtc::RtpPacketReceived const&) () at webrtc/src/call/rtp_stream_receiver_controller.cc:54
#9  DeliverRtp () at webrtc/src/call/call.cc:1423
#10 DeliverPacket () at webrtc/src/call/call.cc:1461
#11 OnPacketReceived () at webrtc/src/media/engine/webrtc_voice_engine.cc:2070
#12 ProcessPacket () at webrtc/src/pc/channel.cc:540
```
 webrtc/src/pc/channel.cc 文件里 `BaseChannel` 类的 `ProcessPacket()` 函数将收到的音频包送进媒体引擎进行处理，这一过程包括，根据 RTP 包的 ssrc 派发进不同的 channel，ACM receiver 的处理，一直到最终插入 NetEq 的缓冲区。在 NetEq 中将会完成数据包的重排序，网络对抗，音频的解码等处理操作。

# 音频数据的解码及播放

AudioDevice 组件被初始化时，即会启动一个播放线程，如 (webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc)：
```
AudioDeviceGeneric::InitStatus AudioDeviceLinuxPulse::Init() {
  RTC_DCHECK(thread_checker_.IsCurrent());
  if (_initialized) {
    return InitStatus::OK;
  }

  // Initialize PulseAudio
  if (InitPulseAudio() < 0) {
    RTC_LOG(LS_ERROR) << "failed to initialize PulseAudio";
    if (TerminatePulseAudio() < 0) {
      RTC_LOG(LS_ERROR) << "failed to terminate PulseAudio";
    }
    return InitStatus::OTHER_ERROR;
  }

#if defined(WEBRTC_USE_X11)
  // Get X display handle for typing detection
  _XDisplay = XOpenDisplay(NULL);
  if (!_XDisplay) {
    RTC_LOG(LS_WARNING)
        << "failed to open X display, typing detection will not work";
  }
#endif

  // RECORDING
  _ptrThreadRec.reset(new rtc::PlatformThread(RecThreadFunc, this,
                                              "webrtc_audio_module_rec_thread",
                                              rtc::kRealtimePriority));

  _ptrThreadRec->Start();

  // PLAYOUT
  _ptrThreadPlay.reset(new rtc::PlatformThread(
      PlayThreadFunc, this, "webrtc_audio_module_play_thread",
      rtc::kRealtimePriority));
  _ptrThreadPlay->Start();

  _initialized = true;

  return InitStatus::OK;
}
```

这个线程不断地从媒体引擎中拿数据进行播放，这个过程如下：
```
#0  GetNextAudioInterleaved () at webrtc/src/modules/audio_coding/neteq/sync_buffer.cc:85
#1  GetAudioInternal () at webrtc/src/modules/audio_coding/neteq/neteq_impl.cc:897
#2  GetAudio () at webrtc/src/modules/audio_coding/neteq/neteq_impl.cc:215
#3  GetAudio () at webrtc/src/modules/audio_coding/acm2/acm_receiver.cc:134
#4  PlayoutData10Ms () at webrtc/src/modules/audio_coding/acm2/audio_coding_module.cc:704
#5  GetAudioFrameWithInfo () at webrtc/src/audio/channel_receive.cc:332
#6  webrtc::internal::AudioReceiveStream::GetAudioFrameWithInfo(int, webrtc::AudioFrame*) () at webrtc/src/audio/audio_receive_stream.cc:276
#7  GetAudioFromSources () at webrtc/src/modules/audio_mixer/audio_mixer_impl.cc:186
#8  Mix () at webrtc/src/modules/audio_mixer/audio_mixer_impl.cc:130
#9  NeedMorePlayData () at webrtc/src/audio/audio_transport_impl.cc:193
#10 RequestPlayoutData () at webrtc/src/modules/audio_device/audio_device_buffer.cc:301
#11 PlayThreadProcess () at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:2121
#12 webrtc::AudioDeviceLinuxPulse::PlayThreadFunc(void*) () at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:1984
```

WebRTC 中对于音频，是即解码即播放的，播放和解码在同一个线程中完成，此外从解码到播放，还将完成回声消除，混音等处理。

上面创建 `WebRtcAudioReceiveStream`，并接收到音频数据包，是这里能够从 mixer 拿到数据并播放的基础。

# webrtc::AudioSendStream 的创建
webrtc::AudioSendStream 的创建由应用程序发起：
```
#0  cricket::BaseChannel::SetLocalContent(cricket::MediaContentDescription const*, webrtc::SdpType, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*) () at webrtc/src/pc/channel.cc:290
#1  PushdownMediaDescription () at webrtc/src/pc/peer_connection.cc:5699
#2  UpdateSessionState () at webrtc/src/pc/peer_connection.cc:5668
#3  ApplyLocalDescription () at webrtc/src/pc/peer_connection.cc:2356
#4  SetLocalDescription () at webrtc/src/pc/peer_connection.cc:2187
#10 Conductor::OnSuccess(webrtc::SessionDescriptionInterface*) () at webrtc/src/examples/peerconnection/client/conductor.cc:544
#11 OnMessage () at webrtc/src/pc/webrtc_session_description_factory.cc:299
#12 Dispatch () at webrtc/src/rtc_base/message_queue.cc:513
#13 ProcessMessages () at webrtc/src/rtc_base/thread.cc:527
#14 rtc::Thread::Run() () at webrtc/src/rtc_base/thread.cc:351
#15 main () at webrtc/src/examples/peerconnection/client/linux/main.cc:111
```

`webrtc/src/pc/channel.cc` 文件里的 `BaseChannel::SetLocalContent()` 将 MediaContentDescription 抛进 worker_thread 中进一步处理：
```
bool BaseChannel::SetLocalContent(const MediaContentDescription* content,
                                  SdpType type,
                                  std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetLocalContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetLocalContent_w, this, content, type, error_desc));
}
```

webrtc::AudioSendStream 最终在 Call 中创建：
```
#0  CreateAudioSendStream () at webrtc/src/call/call.cc:707
#1  WebRtcAudioSendStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:735
#2  AddSendStream () at webrtc/src/media/engine/webrtc_voice_engine.cc:1803
#3  UpdateLocalStreams_w () at webrtc/src/pc/channel.cc:671
#4  SetLocalContent_w () at webrtc/src/pc/channel.cc:906
```

# 音频数据的发送

如前面看到的，AudioDevice 组件被初始化时，在启动播放线程的同时，还会启动一个录制线程。录制线程捕获录制的音频数据，一路传递进行处理：
```
#0  ProcessAndEncodeAudio () at webrtc/src/audio/channel_send.cc:1101
#1  SendAudioData () at webrtc/src/audio/audio_send_stream.cc:365
#2  RecordedDataIsAvailable () at webrtc/src/audio/audio_transport_impl.cc:164
#3  DeliverRecordedData () at webrtc/src/modules/audio_device/audio_device_buffer.cc:269
#4  webrtc::AudioDeviceLinuxPulse::ProcessRecordedData(signed char*, unsigned int, unsigned int) ()
    at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:1971
#5  webrtc::AudioDeviceLinuxPulse::ReadRecordedData(void const*, unsigned long) ()
    at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:1918
#6  RecThreadProcess () at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:2229
#7  webrtc::AudioDeviceLinuxPulse::RecThreadFunc(void*) () at webrtc/src/modules/audio_device/linux/audio_device_pulse_linux.cc:1990
```

AudioDevice 将录制的音频数据一直传递到 webrtc/src/audio/channel_send.cc 文件里的 `ChannelSend::ProcessAndEncodeAudio()`：
```
void ChannelSend::ProcessAndEncodeAudio(
    std::unique_ptr<AudioFrame> audio_frame) {
  RTC_DCHECK_RUNS_SERIALIZED(&audio_thread_race_checker_);
  struct ProcessAndEncodeAudio {
    void operator()() {
      RTC_DCHECK_RUN_ON(&channel->encoder_queue_);
      if (!channel->encoder_queue_is_active_) {
        return;
      }
      channel->ProcessAndEncodeAudioOnTaskQueue(audio_frame.get());
    }
    std::unique_ptr<AudioFrame> audio_frame;
    ChannelSend* const channel;
  };
  // Profile time between when the audio frame is added to the task queue and
  // when the task is actually executed.
  audio_frame->UpdateProfileTimeStamp();
  encoder_queue_.PostTask(ProcessAndEncodeAudio{std::move(audio_frame), this});
}
```

在该函数中，录制获得的 AudioFrame 被抛进编码的线程中的 task 进行编码，封装为 RTP 包，并送进 `PacedSender`，`PacedSender` 将 RTP 包放进 queue 中：
```
#0  InsertPacket () at webrtc/src/modules/pacing/paced_sender.cc:200
#1  non-virtual thunk to webrtc::PacedSender::InsertPacket(webrtc::RtpPacketSender::Priority, unsigned int, unsigned short, long, unsigned long, bool) ()
#2  webrtc::voe::(anonymous namespace)::RtpPacketSenderProxy::InsertPacket(webrtc::RtpPacketSender::Priority, unsigned int, unsigned short, long, unsigned long, bool) () at webrtc/src/audio/channel_send.cc:391
#3  SendToNetwork () at webrtc/src/modules/rtp_rtcp/source/rtp_sender.cc:963
#4  webrtc::RTPSenderAudio::LogAndSendToNetwork(std::__1::unique_ptr<webrtc::RtpPacketToSend, std::__1::default_delete<webrtc::RtpPacketToSend> >, webrtc::StorageType) () at webrtc/src/modules/rtp_rtcp/source/rtp_sender_audio.cc:363
#5  SendAudio () at webrtc/src/modules/rtp_rtcp/source/rtp_sender_audio.cc:260
#6  SendRtpAudio () at webrtc/src/audio/channel_send.cc:568
#7  SendData () at webrtc/src/audio/channel_send.cc:497
#8  Encode () at webrtc/src/modules/audio_coding/acm2/audio_coding_module.cc:385
#9  webrtc::(anonymous namespace)::AudioCodingModuleImpl::Add10MsData(webrtc::AudioFrame const&) ()
    at webrtc/src/modules/audio_coding/acm2/audio_coding_module.cc:430
#10 ProcessAndEncodeAudioOnTaskQueue () at webrtc/src/audio/channel_send.cc:1152
```

`PacedSender` 中的另一个线程从 queue 中拿到 RTP 包并发送：
```
#0  SendPacket () at webrtc/src/pc/channel.cc:397
#1  cricket::BaseChannel::SendPacket(rtc::CopyOnWriteBuffer*, rtc::PacketOptions const&) () at webrtc/src/pc/channel.cc:328
#2  cricket::MediaChannel::DoSendPacket(rtc::CopyOnWriteBuffer*, bool, rtc::PacketOptions const&) () at webrtc/src/media/base/media_channel.h:328
#3  cricket::MediaChannel::SendPacket(rtc::CopyOnWriteBuffer*, rtc::PacketOptions const&) () at webrtc/src/media/base/media_channel.h:249
#4  cricket::WebRtcVideoChannel::SendRtp(unsigned char const*, unsigned long, webrtc::PacketOptions const&) () at webrtc/src/media/engine/webrtc_video_engine.cc:1690
#5  SendPacketToNetwork () at webrtc/src/modules/rtp_rtcp/source/rtp_sender.cc:550
#6  PrepareAndSendPacket () at webrtc/src/modules/rtp_rtcp/source/rtp_sender.cc:791
#7  webrtc::RTPSender::TimeToSendPacket(unsigned int, unsigned short, long, bool, webrtc::PacedPacketInfo const&) () at webrtc/src/modules/rtp_rtcp/source/rtp_sender.cc:604
#8  webrtc::ModuleRtpRtcpImpl::TimeToSendPacket(unsigned int, unsigned short, long, bool, webrtc::PacedPacketInfo const&) () at webrtc/src/modules/rtp_rtcp/source/rtp_rtcp_impl.cc:415
#9  webrtc::PacketRouter::TimeToSendPacket(unsigned int, unsigned short, long, bool, webrtc::PacedPacketInfo const&) ()
    at webrtc/src/modules/pacing/packet_router.cc:123
#10 Process () at webrtc/src/modules/pacing/paced_sender.cc:390
```
RTP 包被 `PacedSender` 一直递到 webrtc/src/pc/channel.cc 文件里定义的 `BaseChannel::SendPacket()`。`BaseChannel::SendPacket()` 将包抛进网络发送线程中发送：
```
bool BaseChannel::SendPacket(bool rtcp,
                             rtc::CopyOnWriteBuffer* packet,
                             const rtc::PacketOptions& options) {
  // Until all the code is migrated to use RtpPacketType instead of bool.
  RtpPacketType packet_type = rtcp ? RtpPacketType::kRtcp : RtpPacketType::kRtp;
  // SendPacket gets called from MediaEngine, on a pacer or an encoder thread.
  // If the thread is not our network thread, we will post to our network
  // so that the real work happens on our network. This avoids us having to
  // synchronize access to all the pieces of the send path, including
  // SRTP and the inner workings of the transport channels.
  // The only downside is that we can't return a proper failure code if
  // needed. Since UDP is unreliable anyway, this should be a non-issue.
  if (!network_thread_->IsCurrent()) {
    // Avoid a copy by transferring the ownership of the packet data.
    int message_id = rtcp ? MSG_SEND_RTCP_PACKET : MSG_SEND_RTP_PACKET;
    SendPacketMessageData* data = new SendPacketMessageData;
    data->packet = std::move(*packet);
    data->options = options;
    network_thread_->Post(RTC_FROM_HERE, this, message_id, data);
    return true;
  }
```

网络发送线程将数据包通过系统 socket 发送到网络：
```
#0  rtc::PhysicalSocket::DoSendTo(int, char const*, int, int, sockaddr const*, unsigned int) () at webrtc/src/rtc_base/physical_socket_server.cc:479
#1  SendTo () at webrtc/src/rtc_base/physical_socket_server.cc:344
#2  rtc::AsyncUDPSocket::SendTo(void const*, unsigned long, rtc::SocketAddress const&, rtc::PacketOptions const&) () at webrtc/src/rtc_base/async_udp_socket.cc:84
#3  SendTo () at webrtc/src/p2p/base/stun_port.cc:301
#4  Send () at webrtc/src/p2p/base/connection.cc:1162
#5  SendPacket () at webrtc/src/p2p/base/p2p_transport_channel.cc:1473
#6  SendPacket () at webrtc/src/p2p/base/dtls_transport.cc:409
#7  SendPacket () at webrtc/src/pc/rtp_transport.cc:147
#8  SendRtpPacket () at webrtc/src/pc/srtp_transport.cc:173
#9  SendPacket () at webrtc/src/pc/channel.cc:457
#10 OnMessage () at webrtc/src/pc/channel.cc:757
```

webrtc/src/rtc_base/physical_socket_server.cc 文件里定义的 `PhysicalSocket::DoSendTo()` 函数将数据包通过系统的 socket 发送到网络上：
```
int PhysicalSocket::DoSendTo(SOCKET socket,
                             const char* buf,
                             int len,
                             int flags,
                             const struct sockaddr* dest_addr,
                             socklen_t addrlen) {
  return ::sendto(socket, buf, len, flags, dest_addr, addrlen);
}
```


