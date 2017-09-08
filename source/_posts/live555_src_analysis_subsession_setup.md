---
title: live555 源码分析：子会话 SETUP
date: 2017-09-08 11:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

在前面 [live555 源码分析：子会话 SDP 行生成](https://www.wolfcstech.com/2017/09/07/live555_src_analysis_subsession_sdp/) 一文中，我们看了 live555 中子会话 `ServerMediaSubsession` 的生命周期，并较为详细地分析了生成 SDP 行的 `sdpLines()`，本文继续分析这一生命周期。
<!--more-->
# 获取流参数

`ServerMediaSubsession` 的 `getStreamParameters()` 函数在执行 RTSP 的 `SETUP` 请求时调用，该函数的实现如下：
```
void OnDemandServerMediaSubsession
::getStreamParameters(unsigned clientSessionId,
    netAddressBits clientAddress,
    Port const& clientRTPPort,
    Port const& clientRTCPPort,
    int tcpSocketNum,
    unsigned char rtpChannelId,
    unsigned char rtcpChannelId,
    netAddressBits& destinationAddress,
    u_int8_t& /*destinationTTL*/,
    Boolean& isMulticast,
    Port& serverRTPPort,
    Port& serverRTCPPort,
    void*& streamToken) {
  if (destinationAddress == 0) destinationAddress = clientAddress;
  struct in_addr destinationAddr; destinationAddr.s_addr = destinationAddress;
  isMulticast = False;

  if (fLastStreamToken != NULL && fReuseFirstSource) {
    // Special case: Rather than creating a new 'StreamState',
    // we reuse the one that we've already created:
    serverRTPPort = ((StreamState*)fLastStreamToken)->serverRTPPort();
    serverRTCPPort = ((StreamState*)fLastStreamToken)->serverRTCPPort();
    ++((StreamState*)fLastStreamToken)->referenceCount();
    streamToken = fLastStreamToken;
  } else {
    // Normal case: Create a new media source:
    unsigned streamBitrate;
    FramedSource* mediaSource
      = createNewStreamSource(clientSessionId, streamBitrate);

    // Create 'groupsock' and 'sink' objects for the destination,
    // using previously unused server port numbers:
    RTPSink* rtpSink = NULL;
    BasicUDPSink* udpSink = NULL;
    Groupsock* rtpGroupsock = NULL;
    Groupsock* rtcpGroupsock = NULL;

    if (clientRTPPort.num() != 0 || tcpSocketNum >= 0) { // Normal case: Create destinations
      portNumBits serverPortNum;
      if (clientRTCPPort.num() == 0) {
        // We're streaming raw UDP (not RTP). Create a single groupsock:
        NoReuse dummy(envir()); // ensures that we skip over ports that are already in use
        for (serverPortNum = fInitialPortNum;; ++serverPortNum) {
          struct in_addr dummyAddr; dummyAddr.s_addr = 0;

          serverRTPPort = serverPortNum;
          rtpGroupsock = createGroupsock(dummyAddr, serverRTPPort);
          if (rtpGroupsock->socketNum() >= 0) break; // success
        }

        udpSink = BasicUDPSink::createNew(envir(), rtpGroupsock);
      } else {
        // Normal case: We're streaming RTP (over UDP or TCP).  Create a pair of
        // groupsocks (RTP and RTCP), with adjacent port numbers (RTP port number even).
        // (If we're multiplexing RTCP and RTP over the same port number, it can be odd or even.)
        NoReuse dummy(envir()); // ensures that we skip over ports that are already in use
        for (portNumBits serverPortNum = fInitialPortNum;; ++serverPortNum) {
          struct in_addr dummyAddr; dummyAddr.s_addr = 0;

          serverRTPPort = serverPortNum;
          rtpGroupsock = createGroupsock(dummyAddr, serverRTPPort);
          if (rtpGroupsock->socketNum() < 0) {
            delete rtpGroupsock;
            continue; // try again
          }

          if (fMultiplexRTCPWithRTP) {
            // Use the RTP 'groupsock' object for RTCP as well:
            serverRTCPPort = serverRTPPort;
            rtcpGroupsock = rtpGroupsock;
          } else {
            // Create a separate 'groupsock' object (with the next (odd) port number) for RTCP:
            serverRTCPPort = ++serverPortNum;
            rtcpGroupsock = createGroupsock(dummyAddr, serverRTCPPort);
            if (rtcpGroupsock->socketNum() < 0) {
              delete rtpGroupsock;
              delete rtcpGroupsock;
              continue; // try again
            }
          }

          break; // success
        }

        unsigned char rtpPayloadType = 96 + trackNumber() - 1; // if dynamic
        rtpSink = createNewRTPSink(rtpGroupsock, rtpPayloadType, mediaSource);
        if (rtpSink != NULL && rtpSink->estimatedBitrate() > 0) streamBitrate = rtpSink->estimatedBitrate();
      }

      // Turn off the destinations for each groupsock.  They'll get set later
      // (unless TCP is used instead):
      if (rtpGroupsock != NULL) rtpGroupsock->removeAllDestinations();
      if (rtcpGroupsock != NULL) rtcpGroupsock->removeAllDestinations();

      if (rtpGroupsock != NULL) {
        // Try to use a big send buffer for RTP -  at least 0.1 second of
        // specified bandwidth and at least 50 KB
        unsigned rtpBufSize = streamBitrate * 25 / 2; // 1 kbps * 0.1 s = 12.5 bytes
        if (rtpBufSize < 50 * 1024) rtpBufSize = 50 * 1024;
        increaseSendBufferTo(envir(), rtpGroupsock->socketNum(), rtpBufSize);
      }
    }

    // Set up the state of the stream.  The stream will get started later:
    streamToken = fLastStreamToken
      = new StreamState(*this, serverRTPPort, serverRTCPPort, rtpSink, udpSink,
        streamBitrate, mediaSource,
        rtpGroupsock, rtcpGroupsock);
  }

  // Record these destinations as being for this client session id:
  Destinations* destinations;
  if (tcpSocketNum < 0) { // UDP
    destinations = new Destinations(destinationAddr, clientRTPPort, clientRTCPPort);
  } else { // TCP
    destinations = new Destinations(tcpSocketNum, rtpChannelId, rtcpChannelId);
  }
  fDestinationsHashTable->Add((char const*)clientSessionId, destinations);
}
```
这个函数主要用于获取为会话分配的 RTP 端口和 RTCP 端口，以及 stream token。`OnDemandServerMediaSubsession` 用 `StreamState` 来管理底层的输入输出组件，并以 `StreamState` 对象的指针作为 stream token。

当会话的 StreamToken 已经存在时，即 `fLastStreamToken` 非空时，请求的信息直接返回给调用者。当会话的 StreamToken 还不存在时，则需要创建底层用于 I/O 的设施，然后将请求的信息返回。

这里我们主要来看流模式为 RTP 的情况，并忽略 RTP 端口号与 RTCP 端口号复用的情况。对于要创建 `StreamState` 的情况，创建动作主要通过如下一些步骤来完成：

第一步，创建 `FramedSource`。

第二步，尝试创建端口号相邻的一对 groupsocks，分别用于 RTP 和 RTCP，其中 RTP 端口号为偶数，RTCP 端口号为奇数。然后基于 RTP 的 groupsock 创建 `RTPSink`。最后对这些 groupsocks 做一些设置，包括清除 RTP 和 RTCP 的 groupsocks 的目标地址，以及增大 RTP 的 groupsock 的发送缓冲区。
```
      } else {
        // Normal case: We're streaming RTP (over UDP or TCP).  Create a pair of
        // groupsocks (RTP and RTCP), with adjacent port numbers (RTP port number even).
        // (If we're multiplexing RTCP and RTP over the same port number, it can be odd or even.)
        NoReuse dummy(envir()); // ensures that we skip over ports that are already in use
        for (portNumBits serverPortNum = fInitialPortNum;; ++serverPortNum) {
          struct in_addr dummyAddr; dummyAddr.s_addr = 0;

          serverRTPPort = serverPortNum;
          rtpGroupsock = createGroupsock(dummyAddr, serverRTPPort);
          if (rtpGroupsock->socketNum() < 0) {
            delete rtpGroupsock;
            continue; // try again
          }

          if (fMultiplexRTCPWithRTP) {
            // Use the RTP 'groupsock' object for RTCP as well:
            serverRTCPPort = serverRTPPort;
            rtcpGroupsock = rtpGroupsock;
          } else {
            // Create a separate 'groupsock' object (with the next (odd) port number) for RTCP:
            serverRTCPPort = ++serverPortNum;
            rtcpGroupsock = createGroupsock(dummyAddr, serverRTCPPort);
            if (rtcpGroupsock->socketNum() < 0) {
              delete rtpGroupsock;
              delete rtcpGroupsock;
              continue; // try again
            }
          }

          break; // success
        }

        unsigned char rtpPayloadType = 96 + trackNumber() - 1; // if dynamic
        rtpSink = createNewRTPSink(rtpGroupsock, rtpPayloadType, mediaSource);
        if (rtpSink != NULL && rtpSink->estimatedBitrate() > 0) streamBitrate = rtpSink->estimatedBitrate();
      }
```

第三步，基于前面创建的 `FramedSource`、RTP 和 RTCP 的 groupsock 和 `RTPSink` 等创建 `StreamState`，并返回给调用者。
```
    // Set up the state of the stream.  The stream will get started later:
    streamToken = fLastStreamToken
      = new StreamState(*this, serverRTPPort, serverRTCPPort, rtpSink, udpSink,
        streamBitrate, mediaSource,
        rtpGroupsock, rtcpGroupsock);
```

`getStreamParameters()` 最后还会创建 `Destinations`。`Destinations` 封装了客户端的地址，及 RTP 和 RTCP 端口号。

live555 中的某些 get* 和 lookup* 操作暗含着在不存在的情况下创建的语义，如这里的 `getStreamParameters()`，还有 `GenericMediaServer` 的 `lookupServerMediaSession()`。

# 播放前的设置
接下来看一下，在 RTSP 的 `PLAY` 请求处理中，播放流媒体之前做的设置。这主要包括 `setStreamScale()`、`seekStream()`、`nullSeekStream()` 和 `getCurrentNPT()` 等操作。

对于 `H264VideoFileServerMediaSubsession` 而言，这些操作的实现也都在 `OnDemandServerMediaSubsession` 中。

`setStreamScale()` 的实现如下：
```
void OnDemandServerMediaSubsession::setStreamScale(unsigned /*clientSessionId*/,
    void* streamToken, float scale) {
  // Changing the scale factor isn't allowed if multiple clients are receiving data
  // from the same source:
  if (fReuseFirstSource) return;

  StreamState* streamState = (StreamState*)streamToken;
  if (streamState != NULL && streamState->mediaSource() != NULL) {
    setStreamSourceScale(streamState->mediaSource(), scale);
  }
}
. . . . . .
void OnDemandServerMediaSubsession
::setStreamSourceScale(FramedSource* /*inputSource*/, float /*scale*/) {
  // Default implementation: Do nothing
}
```

`setStreamScale()` 用于设置流媒体播放的快慢，它将实际的工作委托给 `setStreamSourceScale()`，后者是 `OnDemandServerMediaSubsession` 新加的虚函数，需要子类实现来提供实际的功能。

对于 `H264VideoFileServerMediaSubsession` 来说，设置播放快慢的操作为空操作。

`seekStream()` 的实现如下：
```
void OnDemandServerMediaSubsession::seekStream(unsigned /*clientSessionId*/,
    void* streamToken, double& seekNPT, double streamDuration, u_int64_t& numBytes) {
  fprintf(stderr, "OnDemandServerMediaSubsession::seekStream().\n");
  numBytes = 0; // by default: unknown

  // Seeking isn't allowed if multiple clients are receiving data from the same source:
  if (fReuseFirstSource) return;

  StreamState* streamState = (StreamState*)streamToken;
  if (streamState != NULL && streamState->mediaSource() != NULL) {
    seekStreamSource(streamState->mediaSource(), seekNPT, streamDuration, numBytes);

    streamState->startNPT() = (float)seekNPT;
    RTPSink* rtpSink = streamState->rtpSink(); // alias
    if (rtpSink != NULL) rtpSink->resetPresentationTimes();
  }
}

void OnDemandServerMediaSubsession::seekStream(unsigned /*clientSessionId*/,
    void* streamToken, char*& absStart, char*& absEnd) {
  fprintf(stderr, "OnDemandServerMediaSubsession::seekStream1().\n");
  // Seeking isn't allowed if multiple clients are receiving data from the same source:
  if (fReuseFirstSource) return;

  StreamState* streamState = (StreamState*)streamToken;
  if (streamState != NULL && streamState->mediaSource() != NULL) {
    seekStreamSource(streamState->mediaSource(), absStart, absEnd);
  }
}
. . . . . .
void OnDemandServerMediaSubsession::seekStreamSource(FramedSource* /*inputSource*/,
    double& /*seekNPT*/, double /*streamDuration*/, u_int64_t& numBytes) {
  fprintf(stderr, "OnDemandServerMediaSubsession::seekStreamSource().\n");
  // Default implementation: Do nothing
  numBytes = 0;
}

void OnDemandServerMediaSubsession::seekStreamSource(FramedSource* /*inputSource*/,
    char*& absStart, char*& absEnd) {
  fprintf(stderr, "OnDemandServerMediaSubsession::seekStreamSource().\n");
  // Default implementation: do nothing (but delete[] and assign "absStart" and "absEnd" to NULL, to show that we don't handle this)
  delete[] absStart; absStart = NULL;
  delete[] absEnd; absEnd = NULL;
}
```
`seekStream()` 用于控制播放进度，它们的实际功能同样需要子类实现新定义的两个虚函数来实现。

`nullSeekStream()` 的实现如下：
```
void OnDemandServerMediaSubsession::nullSeekStream(unsigned /*clientSessionId*/, void* streamToken,
    double streamEndTime, u_int64_t& numBytes) {
  fprintf(stderr, "OnDemandServerMediaSubsession::nullSeekStream().\n");
  numBytes = 0; // by default: unknown

  StreamState* streamState = (StreamState*)streamToken;
  if (streamState != NULL && streamState->mediaSource() != NULL) {
    // Because we're not seeking here, get the current NPT, and remember it as the new 'start' NPT:
    streamState->startNPT() = getCurrentNPT(streamToken);

    double duration = streamEndTime - streamState->startNPT();
    if (duration < 0.0) duration = 0.0;
    setStreamSourceDuration(streamState->mediaSource(), duration, numBytes);

    RTPSink* rtpSink = streamState->rtpSink(); // alias
    if (rtpSink != NULL) rtpSink->resetPresentationTimes();
  }
}
. . . . . .
float OnDemandServerMediaSubsession::getCurrentNPT(void* streamToken) {
  fprintf(stderr, "OnDemandServerMediaSubsession::getCurrentNPT().\n");
  do {
    if (streamToken == NULL) break;

    StreamState* streamState = (StreamState*)streamToken;
    RTPSink* rtpSink = streamState->rtpSink();
    if (rtpSink == NULL) break;

    return streamState->startNPT()
      + (rtpSink->mostRecentPresentationTime().tv_sec - rtpSink->initialPresentationTime().tv_sec)
      + (rtpSink->mostRecentPresentationTime().tv_usec - rtpSink->initialPresentationTime().tv_usec)/1000000.0f;
  } while (0);

  return 0.0;
}
. . . . . .
void OnDemandServerMediaSubsession
::setStreamSourceDuration(FramedSource* /*inputSource*/, double /*streamDuration*/, u_int64_t& numBytes) {
  // Default implementation: Do nothing
  numBytes = 0;
}
```
`nullSeekStream()` 用于设置播放的时长，同样需要子类实现新的虚函数，即 `setStreamSourceDuration()` 来提供实际的功能。

`getCurrentNPT()` 用于获取当前 NPT 时间：
```
float OnDemandServerMediaSubsession::getCurrentNPT(void* streamToken) {
  fprintf(stderr, "OnDemandServerMediaSubsession::getCurrentNPT().\n");
  do {
    if (streamToken == NULL) break;

    StreamState* streamState = (StreamState*)streamToken;
    RTPSink* rtpSink = streamState->rtpSink();
    if (rtpSink == NULL) break;

    return streamState->startNPT()
      + (rtpSink->mostRecentPresentationTime().tv_sec - rtpSink->initialPresentationTime().tv_sec)
      + (rtpSink->mostRecentPresentationTime().tv_usec - rtpSink->initialPresentationTime().tv_usec)/1000000.0f;
  } while (0);

  return 0.0;
}
```
这里通过使用起始的 NPT 时间加上 RTP 会话经过的时间长度来计算获得。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# live555 源码分析系列文章
[live555 源码分析：简介](https://www.wolfcstech.com/2017/08/28/live555_src_analysis_introduction/)
[live555 源码分析：基础设施](https://www.wolfcstech.com/2017/08/30/live555_src_analysis_infrasture/)
[live555 源码分析：MediaSever](https://www.wolfcstech.com/2017/08/31/live555_src_analysis_mediaserver/)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](https://www.wolfcstech.com/2017/09/01/live555_src_analysis_rtsp_rtp_rtcp_wireshark/)
[live555 源码分析：RTSPServer](https://www.wolfcstech.com/2017/09/03/live555_src_analysis_rtspserver/)
[live555 源码分析：DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)
[live555 源码分析：SETUP 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_setup/)
[live555 源码分析：PLAY 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_play/)
[live555 源码分析：RTSPServer 组件结构](https://www.wolfcstech.com/2017/09/06/live555_src_analysis_rtspserver_arch/)
[live555 源码分析：ServerMediaSession](https://www.wolfcstech.com/2017/09/07/live555_src_analysis_servermediasession/)
[live555 源码分析：子会话 SDP 行生成](https://www.wolfcstech.com/2017/09/07/live555_src_analysis_subsession_sdp/)
