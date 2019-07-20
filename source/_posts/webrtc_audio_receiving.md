---
title: WebRTC Audio 接收端关键过程
date: 2019-07-20 23:05:49
categories: 音视频开发
tags:
- 音视频开发
---

本文基于 WebRTC 中的示例应用 peerconnection_client 分析 WebRTC Audio 接收端的关键过程。
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
