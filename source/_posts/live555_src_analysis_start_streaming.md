---
title: live555 源码分析：播放启动
date: 2017-09-08 19:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

本文分析 live555 中，流媒体播放启动，数据开始通过 RTP/RTCP 传输的过程。
<!--more-->
如我们在 [live555 源码分析：子会话 SETUP](http://www.jianshu.com/p/5b15810b7577) 中看到的，一个流媒体子会话的播放启动，由 `StreamState::startPlaying` 完成：
```
void OnDemandServerMediaSubsession::startStream(unsigned clientSessionId,
    void* streamToken,
    TaskFunc* rtcpRRHandler,
    void* rtcpRRHandlerClientData,
    unsigned short& rtpSeqNum,
    unsigned& rtpTimestamp,
    ServerRequestAlternativeByteHandler* serverRequestAlternativeByteHandler,
    void* serverRequestAlternativeByteHandlerClientData) {
  StreamState* streamState = (StreamState*)streamToken;
  Destinations* destinations
    = (Destinations*)(fDestinationsHashTable->Lookup((char const*)clientSessionId));
  if (streamState != NULL) {
    streamState->startPlaying(destinations, clientSessionId,
        rtcpRRHandler, rtcpRRHandlerClientData,
        serverRequestAlternativeByteHandler, serverRequestAlternativeByteHandlerClientData);
    RTPSink* rtpSink = streamState->rtpSink(); // alias
    if (rtpSink != NULL) {
      rtpSeqNum = rtpSink->currentSeqNo();
      rtpTimestamp = rtpSink->presetNextTimestamp();
    }
  }
}
```

在这个函数中，首先找到子会话的目标地址，也就是客户端的 IP 地址，和用于接收 RTP/RTCP 的端口号，然后通过 `StreamState::startPlaying()` 启动播放，最后将 RTP 包的初始序列号和初始时间戳返回给调用者，也就是 `RTSPServer`，并由后者返回给客户端，以用于客户端的播放同步。

`StreamState::startPlaying()` 的实现是这样的：
```
void StreamState
::startPlaying(Destinations* dests, unsigned clientSessionId,
    TaskFunc* rtcpRRHandler, void* rtcpRRHandlerClientData,
    ServerRequestAlternativeByteHandler* serverRequestAlternativeByteHandler,
    void* serverRequestAlternativeByteHandlerClientData) {
  if (dests == NULL) return;

  if (fRTCPInstance == NULL && fRTPSink != NULL) {
    // Create (and start) a 'RTCP instance' for this RTP sink:
    fRTCPInstance = fMaster.createRTCP(fRTCPgs, fTotalBW, (unsigned char*)fMaster.fCNAME, fRTPSink);
        // Note: This starts RTCP running automatically
    fRTCPInstance->setAppHandler(fMaster.fAppHandlerTask, fMaster.fAppHandlerClientData);
  }

  if (dests->isTCP) {
    // Change RTP and RTCP to use the TCP socket instead of UDP:
    if (fRTPSink != NULL) {
      fRTPSink->addStreamSocket(dests->tcpSocketNum, dests->rtpChannelId);
      RTPInterface::setServerRequestAlternativeByteHandler(fRTPSink->envir(), dests->tcpSocketNum,
        serverRequestAlternativeByteHandler, serverRequestAlternativeByteHandlerClientData);
        // So that we continue to handle RTSP commands from the client
    }
    if (fRTCPInstance != NULL) {
      fRTCPInstance->addStreamSocket(dests->tcpSocketNum, dests->rtcpChannelId);
      fRTCPInstance->setSpecificRRHandler(dests->tcpSocketNum, dests->rtcpChannelId,
          rtcpRRHandler, rtcpRRHandlerClientData);
    }
  } else {
    // Tell the RTP and RTCP 'groupsocks' about this destination
    // (in case they don't already have it):
    if (fRTPgs != NULL) fRTPgs->addDestination(dests->addr, dests->rtpPort, clientSessionId);
    if (fRTCPgs != NULL && !(fRTCPgs == fRTPgs && dests->rtcpPort.num() == dests->rtpPort.num())) {
      fRTCPgs->addDestination(dests->addr, dests->rtcpPort, clientSessionId);
    }
    if (fRTCPInstance != NULL) {
      fRTCPInstance->setSpecificRRHandler(dests->addr.s_addr, dests->rtcpPort,
          rtcpRRHandler, rtcpRRHandlerClientData);
    }
  }

  if (fRTCPInstance != NULL) {
    // Hack: Send an initial RTCP "SR" packet, before the initial RTP packet, so that receivers will (likely) be able to
    // get RTCP-synchronized presentation times immediately:
    fRTCPInstance->sendReport();
  }

  if (!fAreCurrentlyPlaying && fMediaSource != NULL) {
    if (fRTPSink != NULL) {
      fRTPSink->startPlaying(*fMediaSource, afterPlayingStreamState, this);
      fAreCurrentlyPlaying = True;
    } else if (fUDPSink != NULL) {
      fUDPSink->startPlaying(*fMediaSource, afterPlayingStreamState, this);
      fAreCurrentlyPlaying = True;
    }
  }
}
```

在这个函数中，首先在 RTCPInstance 还没有创建时去创建它：
```
RTCPInstance* OnDemandServerMediaSubsession
::createRTCP(Groupsock* RTCPgs, unsigned totSessionBW, /* in kbps */
    unsigned char const* cname, RTPSink* sink) {
  // Default implementation; may be redefined by subclasses:
  return RTCPInstance::createNew(envir(), RTCPgs, totSessionBW, cname, sink, NULL/*we're a server*/);
}
```

忽略 RTP/RTCP 包走 TCP 的情况。随后 `StreamState::startPlaying()` 对 RTP 和 RTCP 的 groupsock 做一些设置，即为它们添加目标地址，并为 RTCPInstance 做了一些设置：
```
  } else {
    // Tell the RTP and RTCP 'groupsocks' about this destination
    // (in case they don't already have it):
    if (fRTPgs != NULL) fRTPgs->addDestination(dests->addr, dests->rtpPort, clientSessionId);
    if (fRTCPgs != NULL && !(fRTCPgs == fRTPgs && dests->rtcpPort.num() == dests->rtpPort.num())) {
      fRTCPgs->addDestination(dests->addr, dests->rtcpPort, clientSessionId);
    }
    if (fRTCPInstance != NULL) {
      fRTCPInstance->setSpecificRRHandler(dests->addr.s_addr, dests->rtcpPort,
          rtcpRRHandler, rtcpRRHandlerClientData);
    }
  }
```

之后 `StreamState::startPlaying()` 发出一个 RTCP 包。
```
  if (fRTCPInstance != NULL) {
    // Hack: Send an initial RTCP "SR" packet, before the initial RTP packet, so that receivers will (likely) be able to
    // get RTCP-synchronized presentation times immediately:
    fRTCPInstance->sendReport();
  }
```

`fUDPSink` 用于流模式为 RAW UDP 的情况，忽略这种流模式的情况。最后执行 `MediaSink::startPlaying()`，并设置标记 `fAreCurrentlyPlaying`，表示流播放已经启动。

# RTP 包的发送
下面具体来看 RTP 包是怎么被发送出去的。`MediaSink::startPlaying()` 函数的定义如下：
```
Boolean MediaSink::startPlaying(MediaSource& source,
    afterPlayingFunc* afterFunc,
    void* afterClientData) {
  // Make sure we're not already being played:
  if (fSource != NULL) {
    envir().setResultMsg("This sink is already being played");
    return False;
  }

  // Make sure our source is compatible:
  if (!sourceIsCompatibleWithUs(source)) {
    envir().setResultMsg("MediaSink::startPlaying(): source is not compatible!");
    return False;
  }
  fSource = (FramedSource*)&source;

  fAfterFunc = afterFunc;
  fAfterClientData = afterClientData;
  return continuePlaying();
}
```

在这个函数中，保存了传入的回调及回调的参数，然后执行 `continuePlaying()`，`continuePlaying()` 是一个纯虚函数，其实现由 `MediaSink` 的子类 `H264or5VideoRTPSink` 实现：
```
Boolean H264or5VideoRTPSink::continuePlaying() {
  // First, check whether we have a 'fragmenter' class set up yet.
  // If not, create it now:
  if (fOurFragmenter == NULL) {
    fOurFragmenter = new H264or5Fragmenter(fHNumber, envir(), fSource, OutPacketBuffer::maxSize,
        ourMaxPacketSize() - 12/*RTP hdr size*/);
  } else {
    fOurFragmenter->reassignInputSource(fSource);
  }
  fSource = fOurFragmenter;

  // Then call the parent class's implementation:
  return MultiFramedRTPSink::continuePlaying();
}
```
在这个类中，主要是为 `H264or5Fragmenter` 设置了流媒体数据源，并将 `fSource` 设置为 `H264or5Fragmenter`。在这里，`MultiFramedRTPSink` 持有的流媒体数据源 `FramedSource` 由最初在 `H264VideoFileServerMediaSubsession` 中创建的 `H264VideoStreamFramer` 变为了 `H264or5Fragmenter`，而 `H264or5Fragmenter` 则封装了 `H264VideoStreamFramer`。

随后 `H264or5VideoRTPSink::continuePlaying()` 执行 `MultiFramedRTPSink::continuePlaying()` 做进一步的处理。
```
Boolean MultiFramedRTPSink::continuePlaying() {
  // Send the first packet.
  // (This will also schedule any future sends.)
  buildAndSendPacket(True);
  return True;
}
. . . . . .
void MultiFramedRTPSink::buildAndSendPacket(Boolean isFirstPacket) {
  nextTask() = NULL;
  fIsFirstPacket = isFirstPacket;

  // Set up the RTP header:
  unsigned rtpHdr = 0x80000000; // RTP version 2; marker ('M') bit not set (by default; it can be set later)
  rtpHdr |= (fRTPPayloadType<<16);
  rtpHdr |= fSeqNo; // sequence number
  fOutBuf->enqueueWord(rtpHdr);

  // Note where the RTP timestamp will go.
  // (We can't fill this in until we start packing payload frames.)
  fTimestampPosition = fOutBuf->curPacketSize();
  fOutBuf->skipBytes(4); // leave a hole for the timestamp

  fOutBuf->enqueueWord(SSRC());

  // Allow for a special, payload-format-specific header following the
  // RTP header:
  fSpecialHeaderPosition = fOutBuf->curPacketSize();
  fSpecialHeaderSize = specialHeaderSize();
  fOutBuf->skipBytes(fSpecialHeaderSize);

  // Begin packing as many (complete) frames into the packet as we can:
  fTotalFrameSpecificHeaderSizes = 0;
  fNoFramesLeft = False;
  fNumFramesUsedSoFar = 0;
  packFrame();
}
```

`MultiFramedRTPSink::continuePlaying()` 执行 `MultiFramedRTPSink::buildAndSendPacket()`。而 `MultiFramedRTPSink::buildAndSendPacket()` 则是在输出缓冲区构造了 RTP 头部，对于其中暂时无法准确获得的头部字段，还预留了空间。随后调用了 `MultiFramedRTPSink::packFrame()`。
```
void MultiFramedRTPSink::packFrame() {
  // Get the next frame.

  // First, skip over the space we'll use for any frame-specific header:
  fCurFrameSpecificHeaderPosition = fOutBuf->curPacketSize();
  fCurFrameSpecificHeaderSize = frameSpecificHeaderSize();
  fOutBuf->skipBytes(fCurFrameSpecificHeaderSize);
  fTotalFrameSpecificHeaderSizes += fCurFrameSpecificHeaderSize;

  // See if we have an overflow frame that was too big for the last pkt
  if (fOutBuf->haveOverflowData()) {
    // Use this frame before reading a new one from the source
    unsigned frameSize = fOutBuf->overflowDataSize();
    struct timeval presentationTime = fOutBuf->overflowPresentationTime();
    unsigned durationInMicroseconds = fOutBuf->overflowDurationInMicroseconds();
    fOutBuf->useOverflowData();

    afterGettingFrame1(frameSize, 0, presentationTime, durationInMicroseconds);
  } else {
    // Normal case: we need to read a new frame from the source
    if (fSource == NULL) return;
    fSource->getNextFrame(fOutBuf->curPtr(), fOutBuf->totalBytesAvailable(),
        afterGettingFrame, this, ourHandleClosure, this);
  }
}
```

`MultiFramedRTPSink::packFrame()` 由 `FramedSource` 的 `getNextFrame()` 获得帧数据，并在获得帧数据之后得到通知。
```
void FramedSource::getNextFrame(unsigned char* to, unsigned maxSize,
    afterGettingFunc* afterGettingFunc,
    void* afterGettingClientData,
    onCloseFunc* onCloseFunc,
    void* onCloseClientData) {
  // Make sure we're not already being read:
  if (fIsCurrentlyAwaitingData) {
    envir() << "FramedSource[" << this << "]::getNextFrame(): attempting to read more than once at the same time!\n";
    envir().internalError();
  }

  fTo = to;
  fMaxSize = maxSize;
  fNumTruncatedBytes = 0; // by default; could be changed by doGetNextFrame()
  fDurationInMicroseconds = 0; // by default; could be changed by doGetNextFrame()
  fAfterGettingFunc = afterGettingFunc;
  fAfterGettingClientData = afterGettingClientData;
  fOnCloseFunc = onCloseFunc;
  fOnCloseClientData = onCloseClientData;
  fIsCurrentlyAwaitingData = True;

  doGetNextFrame();
}
```
这个函数主要用于为 `FramedSource` 设置媒体流数据要读到哪里，可以读多少自己，以及回调函数的地址。并最终执行 `doGetNextFrame()` 读取数据。

最终数据将由 `ByteStreamFileSource` 的 `doGetNextFrame()` 执行读取任务的调度，并从文件中读取。
```
#0  ByteStreamFileSource::doGetNextFrame (this=0x6d8f10) at ByteStreamFileSource.cpp:96
#1  0x000000000043004c in FramedSource::getNextFrame (this=0x6d8f10, to=0x6da9c0 "(\243\203\367\377\177", maxSize=150000, 
    afterGettingFunc=0x46f6c8 <StreamParser::afterGettingBytes(void*, unsigned int, unsigned int, timeval, unsigned int)>, 
    afterGettingClientData=0x6d91b0, onCloseFunc=0x46f852 <StreamParser::onInputClosure(void*)>, onCloseClientData=0x6d91b0) at FramedSource.cpp:78

-------------------------------------------------------------------------------------------------------------------------------------

#2  0x000000000046f69c in StreamParser::ensureValidBytes1 (this=0x6d91b0, numBytesNeeded=4) at StreamParser.cpp:159
#3  0x00000000004343e5 in StreamParser::ensureValidBytes (this=0x6d91b0, numBytesNeeded=4) at StreamParser.hh:118
#4  0x0000000000434179 in StreamParser::test4Bytes (this=0x6d91b0) at StreamParser.hh:54
#5  0x0000000000471b85 in H264or5VideoStreamParser::parse (this=0x6d91b0) at H264or5VideoStreamFramer.cpp:951
#6  0x000000000043510f in MPEGVideoStreamFramer::continueReadProcessing (this=0x6d9000) at MPEGVideoStreamFramer.cpp:159
#7  0x0000000000435077 in MPEGVideoStreamFramer::doGetNextFrame (this=0x6d9000) at MPEGVideoStreamFramer.cpp:142
#8  0x000000000043004c in FramedSource::getNextFrame (this=0x6d9000, to=0x748d61 "", maxSize=100000, 
    afterGettingFunc=0x474cd2 <H264or5Fragmenter::afterGettingFrame(void*, unsigned int, unsigned int, timeval, unsigned int)>, 
    afterGettingClientData=0x700300, onCloseFunc=0x4300c6 <FramedSource::handleClosure(void*)>, onCloseClientData=0x700300) at FramedSource.cpp:78

-------------------------------------------------------------------------------------------------------------------------------------

#9  0x000000000047480a in H264or5Fragmenter::doGetNextFrame (this=0x700300) at H264or5VideoRTPSink.cpp:181
#10 0x000000000043004c in FramedSource::getNextFrame (this=0x700300, to=0x7304ec "", maxSize=100452, 
    afterGettingFunc=0x45af82 <MultiFramedRTPSink::afterGettingFrame(void*, unsigned int, unsigned int, timeval, unsigned int)>, 
    afterGettingClientData=0x6d92e0, onCloseFunc=0x45b96c <MultiFramedRTPSink::ourHandleClosure(void*)>, onCloseClientData=0x6d92e0) at FramedSource.cpp:78

-------------------------------------------------------------------------------------------------------------------------------------

#11 0x000000000045af61 in MultiFramedRTPSink::packFrame (this=0x6d92e0) at MultiFramedRTPSink.cpp:224
#12 0x000000000045adae in MultiFramedRTPSink::buildAndSendPacket (this=0x6d92e0, isFirstPacket=1 '\001') at MultiFramedRTPSink.cpp:199
#13 0x000000000045abed in MultiFramedRTPSink::continuePlaying (this=0x6d92e0) at MultiFramedRTPSink.cpp:159

-------------------------------------------------------------------------------------------------------------------------------------

#14 0x000000000047452a in H264or5VideoRTPSink::continuePlaying (this=0x6d92e0) at H264or5VideoRTPSink.cpp:127
#15 0x0000000000405d2a in MediaSink::startPlaying (this=0x6d92e0, source=..., afterFunc=0x4621f4 <afterPlayingStreamState(void*)>, 
    afterClientData=0x6d95b0) at MediaSink.cpp:78
#16 0x00000000004626ea in StreamState::startPlaying (this=0x6d95b0, dests=0x6d9620, clientSessionId=1584618840, 
    rtcpRRHandler=0x407280 <GenericMediaServer::ClientSession::noteClientLiveness(GenericMediaServer::ClientSession*)>, rtcpRRHandlerClientData=0x70ba40, 
    serverRequestAlternativeByteHandler=0x4093a6 <RTSPServer::RTSPClientConnection::handleAlternativeRequestByte(void*, unsigned char)>, 
    serverRequestAlternativeByteHandlerClientData=0x6ce910) at OnDemandServerMediaSubsession.cpp:576
#17 0x000000000046138d in OnDemandServerMediaSubsession::startStream (this=0x6d8710, clientSessionId=1584618840, streamToken=0x6d95b0, 
    rtcpRRHandler=0x407280 <GenericMediaServer::ClientSession::noteClientLiveness(GenericMediaServer::ClientSession*)>, rtcpRRHandlerClientData=0x70ba40, 
    rtpSeqNum=@0x7fffffffcd76: 0, rtpTimestamp=@0x7fffffffcdc0: 0, 
    serverRequestAlternativeByteHandler=0x4093a6 <RTSPServer::RTSPClientConnection::handleAlternativeRequestByte(void*, unsigned char)>, 
    serverRequestAlternativeByteHandlerClientData=0x6ce910) at OnDemandServerMediaSubsession.cpp:223
```

这个调用栈比较深。看起来可能会让人感觉比较费解。实际上 live555 中采用装饰器模式来设计 `FramedSource`，一个 `FramedSource` 可以包装另一个 `FramedSource`，并额外提供一些功能，或为了性能优化，或为了数据解析等。

live555 中众多的 `FramedSource` 类之间的关系大概如下图所示：

![](https://www.wolfcstech.com/images/1315506-0b28491d52695852.png)

上面的调用栈，也主要根据 `FramedSource` 的包装关系，由虚线分割为几个不同的阶段。

在 `ByteStreamFileSource` 的 `doGetNextFrame()` 中，调度读取任务：
```
void ByteStreamFileSource::doGetNextFrame() {
  if (feof(fFid) || ferror(fFid) || (fLimitNumBytesToStream && fNumBytesToStream == 0)) {
    handleClosure();
    return;
  }

#ifdef READ_FROM_FILES_SYNCHRONOUSLY
  doReadFromFile();
#else
  if (!fHaveStartedReading) {
    // Await readable data from the file:
    envir().taskScheduler().turnOnBackgroundReadHandling(fileno(fFid),
	       (TaskScheduler::BackgroundHandlerProc*)&fileReadableHandler, this);
    fHaveStartedReading = True;
  }
#endif
}
```

`ByteStreamFileSource::fileReadableHandler()` 读取流媒体内容，并通知调用者：
```
void FramedSource::afterGetting(FramedSource* source) {
  source->nextTask() = NULL;
  source->fIsCurrentlyAwaitingData = False;
  // indicates that we can be read again
  // Note that this needs to be done here, in case the "fAfterFunc"
  // called below tries to read another frame (which it usually will)

  if (source->fAfterGettingFunc != NULL) {
    (*(source->fAfterGettingFunc))(source->fAfterGettingClientData,
        source->fFrameSize, source->fNumTruncatedBytes,
        source->fPresentationTime,
        source->fDurationInMicroseconds);
  }
}
. . . . . .
void ByteStreamFileSource::fileReadableHandler(ByteStreamFileSource* source, int /*mask*/) {
  if (!source->isCurrentlyAwaitingData()) {
    source->doStopGettingFrames(); // we're not ready for the data yet
    return;
  }
  source->doReadFromFile();
}

void ByteStreamFileSource::doReadFromFile() {
  // Try to read as many bytes as will fit in the buffer provided (or "fPreferredFrameSize" if less)
  if (fLimitNumBytesToStream && fNumBytesToStream < (u_int64_t)fMaxSize) {
    fMaxSize = (unsigned)fNumBytesToStream;
  }
  if (fPreferredFrameSize > 0 && fPreferredFrameSize < fMaxSize) {
    fMaxSize = fPreferredFrameSize;
  }
#ifdef READ_FROM_FILES_SYNCHRONOUSLY
  fFrameSize = fread(fTo, 1, fMaxSize, fFid);
#else
  if (fFidIsSeekable) {
    fFrameSize = fread(fTo, 1, fMaxSize, fFid);
  } else {
    // For non-seekable files (e.g., pipes), call "read()" rather than "fread()", to ensure that the read doesn't block:
    fFrameSize = read(fileno(fFid), fTo, fMaxSize);
  }
#endif
  if (fFrameSize == 0) {
    handleClosure();
    return;
  }
  fNumBytesToStream -= fFrameSize;

  // Set the 'presentation time':
  if (fPlayTimePerFrame > 0 && fPreferredFrameSize > 0) {
    if (fPresentationTime.tv_sec == 0 && fPresentationTime.tv_usec == 0) {
      // This is the first frame, so use the current time:
      gettimeofday(&fPresentationTime, NULL);
    } else {
      // Increment by the play time of the previous data:
      unsigned uSeconds	= fPresentationTime.tv_usec + fLastPlayTime;
      fPresentationTime.tv_sec += uSeconds/1000000;
      fPresentationTime.tv_usec = uSeconds%1000000;
    }

    // Remember the play time of this data:
    fLastPlayTime = (fPlayTimePerFrame*fFrameSize)/fPreferredFrameSize;
    fDurationInMicroseconds = fLastPlayTime;
  } else {
    // We don't know a specific play time duration for this data,
    // so just record the current time as being the 'presentation time':
    gettimeofday(&fPresentationTime, NULL);
  }

  // Inform the reader that he has data:
#ifdef READ_FROM_FILES_SYNCHRONOUSLY
  // To avoid possible infinite recursion, we need to return to the event loop to do this:
  nextTask() = envir().taskScheduler().scheduleDelayedTask(0,
				(TaskFunc*)FramedSource::afterGetting, this);
#else
  // Because the file read was done from the event loop, we can call the
  // 'after getting' function directly, without risk of infinite recursion:
  FramedSource::afterGetting(this);
#endif
}
```

数据读取完成之后，`MultiFramedRTPSink` 将得到通知：
```
#0  MultiFramedRTPSink::afterGettingFrame (clientData=0x6d92e0, numBytesRead=18, numTruncatedBytes=0, presentationTime=..., 
    durationInMicroseconds=0) at MultiFramedRTPSink.cpp:233

---------------------------------------------------------------------------------------------------------------------------

#1  0x00000000004300c2 in FramedSource::afterGetting (source=0x7002c0) at FramedSource.cpp:92
#2  0x0000000000474ca6 in H264or5Fragmenter::doGetNextFrame (this=0x7002c0) at H264or5VideoRTPSink.cpp:263
#3  0x0000000000474dac in H264or5Fragmenter::afterGettingFrame1 (this=0x7002c0, frameSize=18, numTruncatedBytes=0, presentationTime=..., 
    durationInMicroseconds=0) at H264or5VideoRTPSink.cpp:292
#4  0x0000000000474d25 in H264or5Fragmenter::afterGettingFrame (clientData=0x7002c0, frameSize=18, numTruncatedBytes=0, presentationTime=..., 
    durationInMicroseconds=0) at H264or5VideoRTPSink.cpp:279

---------------------------------------------------------------------------------------------------------------------------

#5  0x00000000004300c2 in FramedSource::afterGetting (source=0x6d9000) at FramedSource.cpp:92
#6  0x00000000004351ea in MPEGVideoStreamFramer::continueReadProcessing (this=0x6d9000) at MPEGVideoStreamFramer.cpp:179
#7  0x00000000004350da in MPEGVideoStreamFramer::continueReadProcessing (clientData=0x6d9000) at MPEGVideoStreamFramer.cpp:155
#8  0x000000000046f84f in StreamParser::afterGettingBytes1 (this=0x6d91b0, numBytesRead=150000, presentationTime=...) at StreamParser.cpp:191
#9  0x000000000046f718 in StreamParser::afterGettingBytes (clientData=0x6d91b0, numBytesRead=150000, presentationTime=...)
    at StreamParser.cpp:170

---------------------------------------------------------------------------------------------------------------------------

#10 0x00000000004300c2 in FramedSource::afterGetting (source=0x6d8f10) at FramedSource.cpp:92
#11 0x0000000000430c2c in ByteStreamFileSource::doReadFromFile (this=0x6d8f10) at ByteStreamFileSource.cpp:182
#12 0x00000000004309cb in ByteStreamFileSource::fileReadableHandler (source=0x6d8f10) at ByteStreamFileSource.cpp:126
```

我们同样将回调的调用栈，根据 `FramedSource` 的包装关系，分为几个阶段，不同阶段以虚线分割。

`MultiFramedRTPSink::afterGettingFrame()` 函数定义如下：
```
void MultiFramedRTPSink
::afterGettingFrame(void* clientData, unsigned numBytesRead,
		    unsigned numTruncatedBytes,
		    struct timeval presentationTime,
		    unsigned durationInMicroseconds) {
  MultiFramedRTPSink* sink = (MultiFramedRTPSink*)clientData;
  sink->afterGettingFrame1(numBytesRead, numTruncatedBytes,
			   presentationTime, durationInMicroseconds);
}
```

在这个函数中调用 `afterGettingFrame1()`， `afterGettingFrame1()` 则会根据需要调用 `sendPacketIfNecessary()`。`MultiFramedRTPSink::sendPacketIfNecessary()` 定义如下：
```
void MultiFramedRTPSink::sendPacketIfNecessary() {
  if (fNumFramesUsedSoFar > 0) {
    // Send the packet:
#ifdef TEST_LOSS
    if ((our_random()%10) != 0) // simulate 10% packet loss #####
#endif
    if (!fRTPInterface.sendPacket(fOutBuf->packet(), fOutBuf->curPacketSize())) {
      // if failure handler has been specified, call it
      if (fOnSendErrorFunc != NULL) (*fOnSendErrorFunc)(fOnSendErrorData);
    }
    ++fPacketCount;
    fTotalOctetCount += fOutBuf->curPacketSize();
    fOctetCount += fOutBuf->curPacketSize()
      - rtpHeaderSize - fSpecialHeaderSize - fTotalFrameSpecificHeaderSizes;

    ++fSeqNo; // for next time
  }

  if (fOutBuf->haveOverflowData()
      && fOutBuf->totalBytesAvailable() > fOutBuf->totalBufferSize()/2) {
    // Efficiency hack: Reset the packet start pointer to just in front of
    // the overflow data (allowing for the RTP header and special headers),
    // so that we probably don't have to "memmove()" the overflow data
    // into place when building the next packet:
    unsigned newPacketStart = fOutBuf->curPacketSize()
      - (rtpHeaderSize + fSpecialHeaderSize + frameSpecificHeaderSize());
    fOutBuf->adjustPacketStart(newPacketStart);
  } else {
    // Normal case: Reset the packet start pointer back to the start:
    fOutBuf->resetPacketStart();
  }
  fOutBuf->resetOffset();
  fNumFramesUsedSoFar = 0;

  if (fNoFramesLeft) {
    // We're done:
    onSourceClosure();
  } else {
    // We have more frames left to send.  Figure out when the next frame
    // is due to start playing, then make sure that we wait this long before
    // sending the next packet.
    struct timeval timeNow;
    gettimeofday(&timeNow, NULL);
    int secsDiff = fNextSendTime.tv_sec - timeNow.tv_sec;
    int64_t uSecondsToGo = secsDiff*1000000 + (fNextSendTime.tv_usec - timeNow.tv_usec);
    if (uSecondsToGo < 0 || secsDiff < 0) { // sanity check: Make sure that the time-to-delay is non-negative:
      uSecondsToGo = 0;
    }

    // Delay this amount of time:
    nextTask() = envir().taskScheduler().scheduleDelayedTask(uSecondsToGo, (TaskFunc*)sendNext, this);
  }
}
```

在 `MultiFramedRTPSink::sendPacketIfNecessary()` 中，会发送帧数据。且如果流媒体数据发送没有结束的话，在一帧数据发送完成之后，会调度一个定时器任务 `MultiFramedRTPSink::sendNext()` 再次发送帧数据。

`MultiFramedRTPSink::sendNext()` 执行与 `MultiFramedRTPSink::continuePlaying()` 类似的流程，获取下一帧数据并发送。
```
void MultiFramedRTPSink::sendNext(void* firstArg) {
  MultiFramedRTPSink* sink = (MultiFramedRTPSink*)firstArg;
  sink->buildAndSendPacket(False);
}
```

当然也并不是每一次发送帧数据的时候，都需要直接从流媒体源中去获得数据。在 `StreamParser` 中会做判断，当需要帧数据的时候，它会发起对流媒体文件的读取。若无需从文件中读取流媒体数据，则会直接回调：
```
#0  MultiFramedRTPSink::sendPacketIfNecessary (this=0x702140) at MultiFramedRTPSink.cpp:365
#1  0x000000000045b5a4 in MultiFramedRTPSink::afterGettingFrame1 (this=0x702140, frameSize=1444, numTruncatedBytes=0, presentationTime=..., 
    durationInMicroseconds=40000) at MultiFramedRTPSink.cpp:347
#2  0x000000000045afd5 in MultiFramedRTPSink::afterGettingFrame (clientData=0x702140, numBytesRead=1444, numTruncatedBytes=0, 
    presentationTime=..., durationInMicroseconds=40000) at MultiFramedRTPSink.cpp:235
#3  0x00000000004300c2 in FramedSource::afterGetting (source=0x7036d0) at FramedSource.cpp:92

------------------------------------------------------------------------------------------------------------------------------------

#4  0x0000000000474ca6 in H264or5Fragmenter::doGetNextFrame (this=0x7036d0) at H264or5VideoRTPSink.cpp:263
#5  0x0000000000474dac in H264or5Fragmenter::afterGettingFrame1 (this=0x7036d0, frameSize=53527, numTruncatedBytes=0, presentationTime=..., 
    durationInMicroseconds=40000) at H264or5VideoRTPSink.cpp:292
#6  0x0000000000474d25 in H264or5Fragmenter::afterGettingFrame (clientData=0x7036d0, frameSize=53527, numTruncatedBytes=0, 
    presentationTime=..., durationInMicroseconds=40000) at H264or5VideoRTPSink.cpp:279
#7  0x00000000004300c2 in FramedSource::afterGetting (source=0x701e20) at FramedSource.cpp:92

------------------------------------------------------------------------------------------------------------------------------------

#8  0x00000000004351ea in MPEGVideoStreamFramer::continueReadProcessing (this=0x701e20) at MPEGVideoStreamFramer.cpp:179
#9  0x0000000000435077 in MPEGVideoStreamFramer::doGetNextFrame (this=0x701e20) at MPEGVideoStreamFramer.cpp:142

------------------------------------------------------------------------------------------------------------------------------------

#10 0x000000000043004c in FramedSource::getNextFrame (this=0x701e20, to=0x7c3091 "\205\270@\367\017\204?\017", <incomplete sequence \340>, 
    maxSize=100000, 
    afterGettingFunc=0x474cd2 <H264or5Fragmenter::afterGettingFrame(void*, unsigned int, unsigned int, timeval, unsigned int)>, 
    afterGettingClientData=0x7036d0, onCloseFunc=0x4300c6 <FramedSource::handleClosure(void*)>, onCloseClientData=0x7036d0)
    at FramedSource.cpp:78
#11 0x000000000047480a in H264or5Fragmenter::doGetNextFrame (this=0x7036d0) at H264or5VideoRTPSink.cpp:181

------------------------------------------------------------------------------------------------------------------------------------

#12 0x000000000043004c in FramedSource::getNextFrame (this=0x7036d0, to=0x7aa81c "|\205\270@\367\017\204?\017", <incomplete sequence \340>, 
    maxSize=100452, 
    afterGettingFunc=0x45af82 <MultiFramedRTPSink::afterGettingFrame(void*, unsigned int, unsigned int, timeval, unsigned int)>, 
    afterGettingClientData=0x702140, onCloseFunc=0x45b96c <MultiFramedRTPSink::ourHandleClosure(void*)>, onCloseClientData=0x702140)
    at FramedSource.cpp:78
#13 0x000000000045af61 in MultiFramedRTPSink::packFrame (this=0x702140) at MultiFramedRTPSink.cpp:224
#14 0x000000000045adae in MultiFramedRTPSink::buildAndSendPacket (this=0x702140, isFirstPacket=0 '\000') at MultiFramedRTPSink.cpp:199
#15 0x000000000045b969 in MultiFramedRTPSink::sendNext (firstArg=0x702140) at MultiFramedRTPSink.cpp:422
#16 0x000000000047f165 in AlarmHandler::handleTimeout (this=0x7038a0) at BasicTaskScheduler0.cpp:34
#17 0x000000000047d268 in DelayQueue::handleAlarm (this=0x6cdc28) at DelayQueue.cpp:187
#18 0x000000000047c196 in BasicTaskScheduler::SingleStep (this=0x6cdc20, maxDelayTime=0) at BasicTaskScheduler.cpp:212
```

总结一下 RTP 数据包的发送过程：

1. `OnDemandServerMediaSubsession` 中执行 `startStream()` 时，将发起一个对流媒体文件进行读取的任务，读取文件的工作由 `ByteStreamFileSource` 的 `doReadFromFile()` 执行。
2. 在文件读取了一些数据之后，`MultiFramedRTPSink` 得到回调 `afterGetting()`，在这个回调中，发送帧数据。
3. `MultiFramedRTPSink` 的回调中，如果流媒体数据还没有读完的话，则调度一个定时器任务，一段时间之后再次发起获取帧数据的动作。
4. 重复 2 和 3 两步，直到所有的数据都发送完。

# RTCP 包的接收
`StreamState::startPlaying()` 通过 `OnDemandServerMediaSubsession::createRTCP()` 创建 `RTCPInstance`：
```
RTCPInstance* OnDemandServerMediaSubsession
::createRTCP(Groupsock* RTCPgs, unsigned totSessionBW, /* in kbps */
    unsigned char const* cname, RTPSink* sink) {
  fprintf(stderr, "OnDemandServerMediaSubsession::createRTCP().\n");
  // Default implementation; may be redefined by subclasses:
  return RTCPInstance::createNew(envir(), RTCPgs, totSessionBW, cname, sink, NULL/*we're a server*/);
}
```

`OnDemandServerMediaSubsession::createRTCP()` 则通过 `RTCPInstance::createNew()` 创建：
```
RTCPInstance::RTCPInstance(UsageEnvironment& env, Groupsock* RTCPgs,
			   unsigned totSessionBW,
			   unsigned char const* cname,
			   RTPSink* sink, RTPSource* source,
			   Boolean isSSMSource)
  : Medium(env), fRTCPInterface(this, RTCPgs), fTotSessionBW(totSessionBW),
    fSink(sink), fSource(source), fIsSSMSource(isSSMSource),
    fCNAME(RTCP_SDES_CNAME, cname), fOutgoingReportCount(1),
    fAveRTCPSize(0), fIsInitial(1), fPrevNumMembers(0),
    fLastSentSize(0), fLastReceivedSize(0), fLastReceivedSSRC(0),
    fTypeOfEvent(EVENT_UNKNOWN), fTypeOfPacket(PACKET_UNKNOWN_TYPE),
    fHaveJustSentPacket(False), fLastPacketSentSize(0),
    fByeHandlerTask(NULL), fByeHandlerClientData(NULL),
    fSRHandlerTask(NULL), fSRHandlerClientData(NULL),
    fRRHandlerTask(NULL), fRRHandlerClientData(NULL),
    fSpecificRRHandlerTable(NULL),
    fAppHandlerTask(NULL), fAppHandlerClientData(NULL) {
#ifdef DEBUG
  fprintf(stderr, "RTCPInstance[%p]::RTCPInstance()\n", this);
#endif
  if (fTotSessionBW == 0) { // not allowed!
    env << "RTCPInstance::RTCPInstance error: totSessionBW parameter should not be zero!\n";
    fTotSessionBW = 1;
  }

  if (isSSMSource) RTCPgs->multicastSendOnly(); // don't receive multicast

  double timeNow = dTimeNow();
  fPrevReportTime = fNextReportTime = timeNow;

  fKnownMembers = new RTCPMemberDatabase(*this);
  fInBuf = new unsigned char[maxRTCPPacketSize];
  if (fKnownMembers == NULL || fInBuf == NULL) return;
  fNumBytesAlreadyRead = 0;

  fOutBuf = new OutPacketBuffer(preferredRTCPPacketSize, maxRTCPPacketSize, maxRTCPPacketSize);
  if (fOutBuf == NULL) return;

  if (fSource != NULL && fSource->RTPgs() == RTCPgs) {
    // We're receiving RTCP reports that are multiplexed with RTP, so ask the RTP source
    // to give them to us:
    fSource->registerForMultiplexedRTCPPackets(this);
  } else {
    // Arrange to handle incoming reports from the network:
    TaskScheduler::BackgroundHandlerProc* handler
      = (TaskScheduler::BackgroundHandlerProc*)&incomingReportHandler;
    fRTCPInterface.startNetworkReading(handler);
  }

  // Send our first report.
  fTypeOfEvent = EVENT_REPORT;
  onExpire(this);
}
. . . . . .
RTCPInstance* RTCPInstance::createNew(UsageEnvironment& env, Groupsock* RTCPgs,
				      unsigned totSessionBW,
				      unsigned char const* cname,
				      RTPSink* sink, RTPSource* source,
				      Boolean isSSMSource) {
  return new RTCPInstance(env, RTCPgs, totSessionBW, cname, sink, source,
			  isSSMSource);
}
```

可以看到，在 `RTCPInstance` 的构造函数中，调用 `RTPInterface::startNetworkReading()` 注册了一个回调：
```
void RTPInterface
::startNetworkReading(TaskScheduler::BackgroundHandlerProc* handlerProc) {
  // Normal case: Arrange to read UDP packets:
  envir().taskScheduler().
    turnOnBackgroundReadHandling(fGS->socketNum(), handlerProc, fOwner);

  // Also, receive RTP over TCP, on each of our TCP connections:
  fReadHandlerProc = handlerProc;
  for (tcpStreamRecord* streams = fTCPStreams; streams != NULL;
       streams = streams->fNext) {
    // Get a socket descriptor for "streams->fStreamSocketNum":
    SocketDescriptor* socketDescriptor = lookupSocketDescriptor(envir(), streams->fStreamSocketNum);

    // Tell it about our subChannel:
    socketDescriptor->registerRTPInterface(streams->fStreamChannelId, this);
  }
}
```

在 `RTPInterface::startNetworkReading()` 中则会向 TaskScheduler 注册 RTCP 的 socket 及该 socket 上的事件的处理程序。live555 中正是通过这种方式，在有 RTCP 包到来时得到通知，并通过 `RTCPInstance::incomingReportHandler()` 来处理 RTCP 包的。

# RTCP 包的发送

RTCP 包根据需要，由 `RTCPInstance::sendReport()` 等函数发送：
```
void RTCPInstance::sendReport() {
#ifdef DEBUG
  fprintf(stderr, "sending REPORT\n");
#endif
  // Begin by including a SR and/or RR report:
  if (!addReport()) return;

  // Then, include a SDES:
  addSDES();

  // Send the report:
  sendBuiltPacket();

  // Periodically clean out old members from our SSRC membership database:
  const unsigned membershipReapPeriod = 5;
  if ((++fOutgoingReportCount) % membershipReapPeriod == 0) {
    unsigned threshold = fOutgoingReportCount - membershipReapPeriod;
    fKnownMembers->reapOldMembers(threshold);
  }
}

void RTCPInstance::sendBYE() {
#ifdef DEBUG
  fprintf(stderr, "sending BYE\n");
#endif
  // The packet must begin with a SR and/or RR report:
  (void)addReport(True);

  addBYE();
  sendBuiltPacket();
}

void RTCPInstance::sendBuiltPacket() {
#ifdef DEBUG
  fprintf(stderr, "sending RTCP packet\n");
  unsigned char* p = fOutBuf->packet();
  for (unsigned i = 0; i < fOutBuf->curPacketSize(); ++i) {
    if (i%4 == 0) fprintf(stderr," ");
    fprintf(stderr, "%02x", p[i]);
  }
  fprintf(stderr, "\n");
#endif
  unsigned reportSize = fOutBuf->curPacketSize();
  fRTCPInterface.sendPacket(fOutBuf->packet(), reportSize);
  fOutBuf->resetOffset();

  fLastSentSize = IP_UDP_HDR_SIZE + reportSize;
  fHaveJustSentPacket = True;
  fLastPacketSentSize = reportSize;
}
```

就像在 `StreamState::startPlaying()` 中看到的那样。

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
[live555 源码分析：子会话 SETUP](https://www.wolfcstech.com/2017/09/08/live555_src_analysis_subsession_setup/)
[live555 源码分析：播放启动](https://www.wolfcstech.com/2017/09/08/live555_src_analysis_start_streaming/)
