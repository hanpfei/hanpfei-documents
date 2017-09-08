---
title: live555 源码分析： SETUP 的处理
date: 2017-09-05 11:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

`SETUP` 请求在 RTSP 的整个工作流程中，用于建立流媒体会话。本文分析 live555 对 `SETUP` 请求的处理。
<!--more-->
在 `RTSPServer::RTSPClientConnection::handleRequestBytes(int newBytesRead)` 中，通过 `RTSPServer::RTSPClientSession` 的 `handleCmd_SETUP()` 函数处理 `SETUP` 请求，如下所示：
```
void RTSPServer::RTSPClientSession
::handleCmd_SETUP(RTSPServer::RTSPClientConnection* ourClientConnection,
    char const* urlPreSuffix, char const* urlSuffix, char const* fullRequestStr) {
  // Normally, "urlPreSuffix" should be the session (stream) name, and "urlSuffix" should be the subsession (track) name.
  // However (being "liberal in what we accept"), we also handle 'aggregate' SETUP requests (i.e., without a track name),
  // in the special case where we have only a single track.  I.e., in this case, we also handle:
  //    "urlPreSuffix" is empty and "urlSuffix" is the session (stream) name, or
  //    "urlPreSuffix" concatenated with "urlSuffix" (with "/" inbetween) is the session (stream) name.
  char const* streamName = urlPreSuffix; // in the normal case
  char const* trackId = urlSuffix; // in the normal case
  char* concatenatedStreamName = NULL; // in the normal case
  
  do {
    // First, make sure the specified stream name exists:
    ServerMediaSession* sms
      = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL);
    if (sms == NULL) {
      // Check for the special case (noted above), before we give up:
      if (urlPreSuffix[0] == '\0') {
        streamName = urlSuffix;
      } else {
        concatenatedStreamName = new char[strlen(urlPreSuffix)
            + strlen(urlSuffix) + 2]; // allow for the "/" and the trailing '\0'
        sprintf(concatenatedStreamName, "%s/%s", urlPreSuffix, urlSuffix);
        streamName = concatenatedStreamName;
      }
      trackId = NULL;
      
      // Check again:
      sms = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL);
    }
    if (sms == NULL) {
      if (fOurServerMediaSession == NULL) {
        // The client asked for a stream that doesn't exist (and this session descriptor has not been used before):
        ourClientConnection->handleCmd_notFound();
      } else {
        // The client asked for a stream that doesn't exist, but using a stream id for a stream that does exist. Bad request:
        ourClientConnection->handleCmd_bad();
      }
      break;
    } else {
      if (fOurServerMediaSession == NULL) {
        // We're accessing the "ServerMediaSession" for the first time.
        fOurServerMediaSession = sms;
        fOurServerMediaSession->incrementReferenceCount();
      } else if (sms != fOurServerMediaSession) {
        // The client asked for a stream that's different from the one originally requested for this stream id.  Bad request:
        ourClientConnection->handleCmd_bad();
        break;
      }
    }
    
    if (fStreamStates == NULL) {
      // This is the first "SETUP" for this session.  Set up our array of states for all of this session's subsessions (tracks):
      fNumStreamStates = fOurServerMediaSession->numSubsessions();
      fStreamStates = new struct streamState[fNumStreamStates];
      
      ServerMediaSubsessionIterator iter(*fOurServerMediaSession);
      ServerMediaSubsession* subsession;
      for (unsigned i = 0; i < fNumStreamStates; ++i) {
        subsession = iter.next();
        fStreamStates[i].subsession = subsession;
        fStreamStates[i].tcpSocketNum = -1; // for now; may get set for RTP-over-TCP streaming
        fStreamStates[i].streamToken = NULL; // for now; it may be changed by the "getStreamParameters()" call that comes later
      }
    }
    
    // Look up information for the specified subsession (track):
    ServerMediaSubsession* subsession = NULL;
    unsigned trackNum;
    if (trackId != NULL && trackId[0] != '\0') { // normal case
      for (trackNum = 0; trackNum < fNumStreamStates; ++trackNum) {
        subsession = fStreamStates[trackNum].subsession;
        if (subsession != NULL && strcmp(trackId, subsession->trackId()) == 0) break;
      }
      if (trackNum >= fNumStreamStates) {
        // The specified track id doesn't exist, so this request fails:
        ourClientConnection->handleCmd_notFound();
        break;
      }
    } else {
      // Weird case: there was no track id in the URL.
      // This works only if we have only one subsession:
      if (fNumStreamStates != 1 || fStreamStates[0].subsession == NULL) {
        ourClientConnection->handleCmd_bad();
        break;
      }
      trackNum = 0;
      subsession = fStreamStates[trackNum].subsession;
    }
    // ASSERT: subsession != NULL
    
    void*& token = fStreamStates[trackNum].streamToken; // alias
    if (token != NULL) {
      // We already handled a "SETUP" for this track (to the same client),
      // so stop any existing streaming of it, before we set it up again:
      subsession->pauseStream(fOurSessionId, token);
      fOurRTSPServer.unnoteTCPStreamingOnSocket(fStreamStates[trackNum].tcpSocketNum, this, trackNum);
      subsession->deleteStream(fOurSessionId, token);
    }

    // Look for a "Transport:" header in the request string, to extract client parameters:
    StreamingMode streamingMode;
    char* streamingModeString = NULL; // set when RAW_UDP streaming is specified
    char* clientsDestinationAddressStr;
    u_int8_t clientsDestinationTTL;
    portNumBits clientRTPPortNum, clientRTCPPortNum;
    unsigned char rtpChannelId, rtcpChannelId;
    parseTransportHeader(fullRequestStr, streamingMode, streamingModeString,
        clientsDestinationAddressStr, clientsDestinationTTL,
        clientRTPPortNum, clientRTCPPortNum,
        rtpChannelId, rtcpChannelId);
    if ((streamingMode == RTP_TCP && rtpChannelId == 0xFF)
        || (streamingMode != RTP_TCP && ourClientConnection->fClientOutputSocket != ourClientConnection->fClientInputSocket)) {
      // An anomolous situation, caused by a buggy client.  Either:
      //     1/ TCP streaming was requested, but with no "interleaving=" fields.  (QuickTime Player sometimes does this.), or
      //     2/ TCP streaming was not requested, but we're doing RTSP-over-HTTP tunneling (which implies TCP streaming).
      // In either case, we assume TCP streaming, and set the RTP and RTCP channel ids to proper values:
      streamingMode = RTP_TCP;
      rtpChannelId = fTCPStreamIdCount; rtcpChannelId = fTCPStreamIdCount+1;
    }
    if (streamingMode == RTP_TCP) fTCPStreamIdCount += 2;
    
    Port clientRTPPort(clientRTPPortNum);
    Port clientRTCPPort(clientRTCPPortNum);
    
    // Next, check whether a "Range:" or "x-playNow:" header is present in the request.
    // This isn't legal, but some clients do this to combine "SETUP" and "PLAY":
    double rangeStart = 0.0, rangeEnd = 0.0;
    char* absStart = NULL; char* absEnd = NULL;
    Boolean startTimeIsNow;
    if (parseRangeHeader(fullRequestStr, rangeStart, rangeEnd, absStart, absEnd, startTimeIsNow)) {
      delete[] absStart; delete[] absEnd;
      fStreamAfterSETUP = True;
    } else if (parsePlayNowHeader(fullRequestStr)) {
      fStreamAfterSETUP = True;
    } else {
      fStreamAfterSETUP = False;
    }
    
    // Then, get server parameters from the 'subsession':
    if (streamingMode == RTP_TCP) {
      // Note that we'll be streaming over the RTSP TCP connection:
      fStreamStates[trackNum].tcpSocketNum = ourClientConnection->fClientOutputSocket;
      fOurRTSPServer.noteTCPStreamingOnSocket(fStreamStates[trackNum].tcpSocketNum, this, trackNum);
    }
    netAddressBits destinationAddress = 0;
    u_int8_t destinationTTL = 255;
#ifdef RTSP_ALLOW_CLIENT_DESTINATION_SETTING
    if (clientsDestinationAddressStr != NULL) {
      // Use the client-provided "destination" address.
      // Note: This potentially allows the server to be used in denial-of-service
      // attacks, so don't enable this code unless you're sure that clients are
      // trusted.
      destinationAddress = our_inet_addr(clientsDestinationAddressStr);
    }
    // Also use the client-provided TTL.
    destinationTTL = clientsDestinationTTL;
#endif
    delete[] clientsDestinationAddressStr;
    Port serverRTPPort(0);
    Port serverRTCPPort(0);
    
    // Make sure that we transmit on the same interface that's used by the client (in case we're a multi-homed server):
    struct sockaddr_in sourceAddr; SOCKLEN_T namelen = sizeof sourceAddr;
    getsockname(ourClientConnection->fClientInputSocket, (struct sockaddr*)&sourceAddr, &namelen);
    netAddressBits origSendingInterfaceAddr = SendingInterfaceAddr;
    netAddressBits origReceivingInterfaceAddr = ReceivingInterfaceAddr;
    // NOTE: The following might not work properly, so we ifdef it out for now:
#ifdef HACK_FOR_MULTIHOMED_SERVERS
    ReceivingInterfaceAddr = SendingInterfaceAddr = sourceAddr.sin_addr.s_addr;
#endif
    
    subsession->getStreamParameters(fOurSessionId, ourClientConnection->fClientAddr.sin_addr.s_addr,
        clientRTPPort, clientRTCPPort,
        fStreamStates[trackNum].tcpSocketNum, rtpChannelId, rtcpChannelId,
        destinationAddress, destinationTTL, fIsMulticast,
        serverRTPPort, serverRTCPPort,
        fStreamStates[trackNum].streamToken);
    SendingInterfaceAddr = origSendingInterfaceAddr;
    ReceivingInterfaceAddr = origReceivingInterfaceAddr;

    AddressString destAddrStr(destinationAddress);
    AddressString sourceAddrStr(sourceAddr);
    char timeoutParameterString[100];
    if (fOurRTSPServer.fReclamationSeconds > 0) {
      sprintf(timeoutParameterString, ";timeout=%u",
          fOurRTSPServer.fReclamationSeconds);
    } else {
      timeoutParameterString[0] = '\0';
    }
    if (fIsMulticast) {
      switch (streamingMode) {
      case RTP_UDP: {
        snprintf((char*) ourClientConnection->fResponseBuffer, sizeof ourClientConnection->fResponseBuffer,
            "RTSP/1.0 200 OK\r\n"
            "CSeq: %s\r\n"
            "%s"
            "Transport: RTP/AVP;multicast;destination=%s;source=%s;port=%d-%d;ttl=%d\r\n"
            "Session: %08X%s\r\n\r\n",
            ourClientConnection->fCurrentCSeq,
            dateHeader(),
            destAddrStr.val(), sourceAddrStr.val(),
            ntohs(serverRTPPort.num()), ntohs(serverRTCPPort.num()), destinationTTL,
            fOurSessionId, timeoutParameterString);
        break;
      }
      case RTP_TCP: {
        // multicast streams can't be sent via TCP
        ourClientConnection->handleCmd_unsupportedTransport();
        break;
      }
      case RAW_UDP: {
        snprintf((char*) ourClientConnection->fResponseBuffer, sizeof ourClientConnection->fResponseBuffer,
            "RTSP/1.0 200 OK\r\n"
            "CSeq: %s\r\n"
            "%s"
            "Transport: %s;multicast;destination=%s;source=%s;port=%d;ttl=%d\r\n"
            "Session: %08X%s\r\n\r\n",
            ourClientConnection->fCurrentCSeq,
            dateHeader(),
            streamingModeString, destAddrStr.val(), sourceAddrStr.val(), ntohs(serverRTPPort.num()), destinationTTL,
            fOurSessionId, timeoutParameterString);
        break;
      }
      }
    } else {
      switch (streamingMode) {
      case RTP_UDP: {
        snprintf((char*) ourClientConnection->fResponseBuffer, sizeof ourClientConnection->fResponseBuffer,
            "RTSP/1.0 200 OK\r\n"
            "CSeq: %s\r\n"
            "%s"
            "Transport: RTP/AVP;unicast;destination=%s;source=%s;client_port=%d-%d;server_port=%d-%d\r\n"
            "Session: %08X%s\r\n\r\n",
            ourClientConnection->fCurrentCSeq,
            dateHeader(),
            destAddrStr.val(), sourceAddrStr.val(), ntohs(clientRTPPort.num()), ntohs(clientRTCPPort.num()), ntohs(serverRTPPort.num()), ntohs(serverRTCPPort.num()),
            fOurSessionId, timeoutParameterString);
        break;
      }
      case RTP_TCP: {
        if (!fOurRTSPServer.fAllowStreamingRTPOverTCP) {
          ourClientConnection->handleCmd_unsupportedTransport();
        } else {
          snprintf((char*) ourClientConnection->fResponseBuffer, sizeof ourClientConnection->fResponseBuffer,
              "RTSP/1.0 200 OK\r\n"
              "CSeq: %s\r\n"
              "%s"
              "Transport: RTP/AVP/TCP;unicast;destination=%s;source=%s;interleaved=%d-%d\r\n"
              "Session: %08X%s\r\n\r\n",
              ourClientConnection->fCurrentCSeq,
              dateHeader(),
              destAddrStr.val(), sourceAddrStr.val(), rtpChannelId, rtcpChannelId,
              fOurSessionId, timeoutParameterString);
        }
        break;
      }
      case RAW_UDP: {
        snprintf((char*) ourClientConnection->fResponseBuffer,
            sizeof ourClientConnection->fResponseBuffer,
            "RTSP/1.0 200 OK\r\n"
            "CSeq: %s\r\n"
            "%s"
            "Transport: %s;unicast;destination=%s;source=%s;client_port=%d;server_port=%d\r\n"
            "Session: %08X%s\r\n\r\n",
            ourClientConnection->fCurrentCSeq,
            dateHeader(),
            streamingModeString, destAddrStr.val(), sourceAddrStr.val(), ntohs(clientRTPPort.num()), ntohs(serverRTPPort.num()),
            fOurSessionId, timeoutParameterString);
        break;
      }
      }
    }
    delete[] streamingModeString;
  } while (0);

  delete[] concatenatedStreamName;
}
```
第一步，在这个函数中，首先查找资源对应的 `ServerMediaSession`，先尝试以资源路径不包含 track id 的部分查找，如果失败则会以资源全路径查找：
```
    ServerMediaSession* sms
      = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL);
    if (sms == NULL) {
      // Check for the special case (noted above), before we give up:
      if (urlPreSuffix[0] == '\0') {
        streamName = urlSuffix;
      } else {
        concatenatedStreamName = new char[strlen(urlPreSuffix)
            + strlen(urlSuffix) + 2]; // allow for the "/" and the trailing '\0'
        sprintf(concatenatedStreamName, "%s/%s", urlPreSuffix, urlSuffix);
        streamName = concatenatedStreamName;
      }
      trackId = NULL;
      
      // Check again:
      sms = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL);
    }
```
在这里 `lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession)` 的 `isFirstLookupInSession` 参数不再是默认值了，而是根据 `fOurServerMediaSession` 的值来确定。

可见在处理 `DESCRIBE` 请求时创建的 `ServerMediaSession` 将总是会被销毁，并重建。

第二步，根据前一步的查找结果做容错处理或更新状态。

如果查找或创建 `ServerMediaSession` 失败，且 `fOurServerMediaSession` 为空，向客户端返回 404 错误，这种情况比较容易理解，即在 `DESCRIBE` 请求之后，资源被移除了；如果查找或创建 `ServerMediaSession` 失败，且 `fOurServerMediaSession` 为非空，此时则向客户端返回 400 错误，这种场景似乎是，执行了多次 `SETUP` 操作，第一次执行的时候资源在，但后面执行的时候，资源不存在了。

查找或创建 `ServerMediaSession` 失败总是会使函数提前返回。
```
    if (sms == NULL) {
      if (fOurServerMediaSession == NULL) {
        // The client asked for a stream that doesn't exist (and this session descriptor has not been used before):
        ourClientConnection->handleCmd_notFound();
      } else {
        // The client asked for a stream that doesn't exist, but using a stream id for a stream that does exist. Bad request:
        ourClientConnection->handleCmd_bad();
      }
      break;
    } else {
```

对于查找或创建 `ServerMediaSession` 成功的情况，如果此时 `fOurServerMediaSession` 为空，表明这是这个流媒体会话第一次执行 `SETUP` 操作，此时需要更新 `fOurServerMediaSession` 并增加其引用计数；如果 `fOurServerMediaSession` 非空，且与找到的不同，则向客户端返回 400 错误响应消息，***不是很明白这究竟是什么样的场景***。

第三部，根据需要创建流状态。为流媒体的每个子会话创建一个流状态结构 `streamState`。
```
    if (fStreamStates == NULL) {
      // This is the first "SETUP" for this session.  Set up our array of states for all of this session's subsessions (tracks):
      fNumStreamStates = fOurServerMediaSession->numSubsessions();
      fStreamStates = new struct streamState[fNumStreamStates];
      
      ServerMediaSubsessionIterator iter(*fOurServerMediaSession);
      ServerMediaSubsession* subsession;
      for (unsigned i = 0; i < fNumStreamStates; ++i) {
        subsession = iter.next();
        fStreamStates[i].subsession = subsession;
        fStreamStates[i].tcpSocketNum = -1; // for now; may get set for RTP-over-TCP streaming
        fStreamStates[i].streamToken = NULL; // for now; it may be changed by the "getStreamParameters()" call that comes later
      }
    }
```

第四步，查找特定子会话（track）的信息。对于 URL 中携带了 track id 的请求，就根据 track id 查找，否则对于只有一个子会话的情况，就以该会话作为 track 会话。
```
    // Look up information for the specified subsession (track):
    ServerMediaSubsession* subsession = NULL;
    unsigned trackNum;
    if (trackId != NULL && trackId[0] != '\0') { // normal case
      for (trackNum = 0; trackNum < fNumStreamStates; ++trackNum) {
        subsession = fStreamStates[trackNum].subsession;
        if (subsession != NULL && strcmp(trackId, subsession->trackId()) == 0) break;
      }
      if (trackNum >= fNumStreamStates) {
        // The specified track id doesn't exist, so this request fails:
        ourClientConnection->handleCmd_notFound();
        break;
      }
    } else {
      // Weird case: there was no track id in the URL.
      // This works only if we have only one subsession:
      if (fNumStreamStates != 1 || fStreamStates[0].subsession == NULL) {
        ourClientConnection->handleCmd_bad();
        break;
      }
      trackNum = 0;
      subsession = fStreamStates[trackNum].subsession;
    }
```

第五步，如果 track 子会话的 token 非空，说明已经为该 track 处理过 `SETUP` 请求，则在重新设置它之前，先停止其已有的流。
```
    void*& token = fStreamStates[trackNum].streamToken; // alias
    if (token != NULL) {
      // We already handled a "SETUP" for this track (to the same client),
      // so stop any existing streaming of it, before we set it up again:
      subsession->pauseStream(fOurSessionId, token);
      fOurRTSPServer.unnoteTCPStreamingOnSocket(fStreamStates[trackNum].tcpSocketNum, this, trackNum);
      subsession->deleteStream(fOurSessionId, token);
    }
```
会话中流媒体的控制的具体操作过程，暂时先不详细分析。

第六步，从请求字符串中查找 `Transport: ` 头部，并从中提取客户端参数。`SETUP` 请求的 `Transport: ` 头部看起来可能像下面这样：
```
Transport: RTP/AVP/UDP;unicast;client_port=19586-19587
```
在这个头部中，包含有通信的方式，对于 UDP，是单播还是多播，客户端收发 RTP/RTCP 包所用的端口号等。

```
typedef enum StreamingMode {
  RTP_UDP,
  RTP_TCP,
  RAW_UDP
} StreamingMode;

static void parseTransportHeader(char const* buf,
    StreamingMode& streamingMode,
    char*& streamingModeString,
    char*& destinationAddressStr,
    u_int8_t& destinationTTL,
    portNumBits& clientRTPPortNum, // if UDP
    portNumBits& clientRTCPPortNum, // if UDP
    unsigned char& rtpChannelId, // if TCP
    unsigned char& rtcpChannelId // if TCP
    ) {
  // Initialize the result parameters to default values:
  streamingMode = RTP_UDP;
  streamingModeString = NULL;
  destinationAddressStr = NULL;
  destinationTTL = 255;
  clientRTPPortNum = 0;
  clientRTCPPortNum = 1;
  rtpChannelId = rtcpChannelId = 0xFF;
  
  portNumBits p1, p2;
  unsigned ttl, rtpCid, rtcpCid;
  
  // First, find "Transport:"
  while (1) {
    if (*buf == '\0') return; // not found
    if (*buf == '\r' && *(buf+1) == '\n' && *(buf+2) == '\r') return; // end of the headers => not found
    if (_strncasecmp(buf, "Transport:", 10) == 0) break;
    ++buf;
  }
  
  // Then, run through each of the fields, looking for ones we handle:
  char const* fields = buf + 10;
  while (*fields == ' ') ++fields;
  char* field = strDupSize(fields);
  while (sscanf(fields, "%[^;\r\n]", field) == 1) {
    if (strcmp(field, "RTP/AVP/TCP") == 0) {
      streamingMode = RTP_TCP;
    } else if (strcmp(field, "RAW/RAW/UDP") == 0
        || strcmp(field, "MP2T/H2221/UDP") == 0) {
      streamingMode = RAW_UDP;
      streamingModeString = strDup(field);
    } else if (_strncasecmp(field, "destination=", 12) == 0) {
      delete[] destinationAddressStr;
      destinationAddressStr = strDup(field+12);
    } else if (sscanf(field, "ttl%u", &ttl) == 1) {
      destinationTTL = (u_int8_t)ttl;
    } else if (sscanf(field, "client_port=%hu-%hu", &p1, &p2) == 2) {
      clientRTPPortNum = p1;
      clientRTCPPortNum = streamingMode == RAW_UDP ? 0 : p2; // ignore the second port number if the client asked for raw UDP
    } else if (sscanf(field, "client_port=%hu", &p1) == 1) {
      clientRTPPortNum = p1;
      clientRTCPPortNum = streamingMode == RAW_UDP ? 0 : p1 + 1;
    } else if (sscanf(field, "interleaved=%u-%u", &rtpCid, &rtcpCid) == 2) {
      rtpChannelId = (unsigned char)rtpCid;
      rtcpChannelId = (unsigned char)rtcpCid;
    }
    
    fields += strlen(field);
    while (*fields == ';' || *fields == ' ' || *fields == '\t') ++fields; // skip over separating ';' chars or whitespace
    if (*fields == '\0' || *fields == '\r' || *fields == '\n') break;
  }
  delete[] field;
}
. . . . . . 
    // Look for a "Transport:" header in the request string, to extract client parameters:
    StreamingMode streamingMode;
    char* streamingModeString = NULL; // set when RAW_UDP streaming is specified
    char* clientsDestinationAddressStr;
    u_int8_t clientsDestinationTTL;
    portNumBits clientRTPPortNum, clientRTCPPortNum;
    unsigned char rtpChannelId, rtcpChannelId;
    parseTransportHeader(fullRequestStr, streamingMode, streamingModeString,
        clientsDestinationAddressStr, clientsDestinationTTL,
        clientRTPPortNum, clientRTCPPortNum,
        rtpChannelId, rtcpChannelId);
    if ((streamingMode == RTP_TCP && rtpChannelId == 0xFF)
        || (streamingMode != RTP_TCP && ourClientConnection->fClientOutputSocket != ourClientConnection->fClientInputSocket)) {
      // An anomolous situation, caused by a buggy client.  Either:
      //     1/ TCP streaming was requested, but with no "interleaving=" fields.  (QuickTime Player sometimes does this.), or
      //     2/ TCP streaming was not requested, but we're doing RTSP-over-HTTP tunneling (which implies TCP streaming).
      // In either case, we assume TCP streaming, and set the RTP and RTCP channel ids to proper values:
      streamingMode = RTP_TCP;
      rtpChannelId = fTCPStreamIdCount; rtcpChannelId = fTCPStreamIdCount+1;
    }
    if (streamingMode == RTP_TCP) fTCPStreamIdCount += 2;
    
    Port clientRTPPort(clientRTPPortNum);
    Port clientRTCPPort(clientRTCPPortNum);
```
我们前面在示例 `Transport: ` 头部中，可以看到 "unicast"，但在 live555 中，不是根据 `Transport: ` 头部的内容来确定流用单播还是多播的。

第七步，检查请求中是否有 `Range:` 或 `x-playNow:` 头部。这不是标准 RTSP 支持的做法，但一些客户端可能会用这种方法把 `SETUP` 和 `PLAY` 结合起来。
```
Boolean parseRangeParam(char const* paramStr,
			double& rangeStart, double& rangeEnd,
			char*& absStartTime, char*& absEndTime,
			Boolean& startTimeIsNow) {
  delete[] absStartTime; delete[] absEndTime;
  absStartTime = absEndTime = NULL; // by default, unless "paramStr" is a "clock=..." string
  startTimeIsNow = False; // by default
  double start, end;
  int numCharsMatched1 = 0, numCharsMatched2 = 0, numCharsMatched3 = 0, numCharsMatched4 = 0;
  Locale l("C", Numeric);
  if (sscanf(paramStr, "npt = %lf - %lf", &start, &end) == 2) {
    rangeStart = start;
    rangeEnd = end;
  } else if (sscanf(paramStr, "npt = %n%lf -", &numCharsMatched1, &start) == 1) {
    if (paramStr[numCharsMatched1] == '-') {
      // special case for "npt = -<endtime>", which matches here:
      rangeStart = 0.0; startTimeIsNow = True;
      rangeEnd = -start;
    } else {
      rangeStart = start;
      rangeEnd = 0.0;
    }
  } else if (sscanf(paramStr, "npt = now - %lf", &end) == 1) {
      rangeStart = 0.0; startTimeIsNow = True;
      rangeEnd = end;
  } else if (sscanf(paramStr, "npt = now -%n", &numCharsMatched2) == 0 && numCharsMatched2 > 0) {
    rangeStart = 0.0; startTimeIsNow = True;
    rangeEnd = 0.0;
  } else if (sscanf(paramStr, "clock = %n", &numCharsMatched3) == 0 && numCharsMatched3 > 0) {
    rangeStart = rangeEnd = 0.0;

    char const* utcTimes = &paramStr[numCharsMatched3];
    size_t len = strlen(utcTimes) + 1;
    char* as = new char[len];
    char* ae = new char[len];
    int sscanfResult = sscanf(utcTimes, "%[^-]-%[^\r\n]", as, ae);
    if (sscanfResult == 2) {
      absStartTime = as;
      absEndTime = ae;
    } else if (sscanfResult == 1) {
      absStartTime = as;
      delete[] ae;
    } else {
      delete[] as; delete[] ae;
      return False;
    }
  } else if (sscanf(paramStr, "smtpe = %n", &numCharsMatched4) == 0 && numCharsMatched4 > 0) {
    // We accept "smtpe=" parameters, but currently do not interpret them.
  } else {
    return False; // The header is malformed
  }

  return True;
}

Boolean parseRangeHeader(char const* buf,
			 double& rangeStart, double& rangeEnd,
			 char*& absStartTime, char*& absEndTime,
			 Boolean& startTimeIsNow) {
  // First, find "Range:"
  while (1) {
    if (*buf == '\0') return False; // not found
    if (_strncasecmp(buf, "Range: ", 7) == 0) break;
    ++buf;
  }

  char const* fields = buf + 7;
  while (*fields == ' ') ++fields;
  return parseRangeParam(fields, rangeStart, rangeEnd, absStartTime, absEndTime, startTimeIsNow);
}
. . . . . .
static Boolean parsePlayNowHeader(char const* buf) {
  // Find "x-playNow:" header, if present
  while (1) {
    if (*buf == '\0') return False; // not found
    if (_strncasecmp(buf, "x-playNow:", 10) == 0) break;
    ++buf;
  }
  
  return True;
}
. . . . . .
    double rangeStart = 0.0, rangeEnd = 0.0;
    char* absStart = NULL; char* absEnd = NULL;
    Boolean startTimeIsNow;
    if (parseRangeHeader(fullRequestStr, rangeStart, rangeEnd, absStart, absEnd, startTimeIsNow)) {
      delete[] absStart; delete[] absEnd;
      fStreamAfterSETUP = True;
    } else if (parsePlayNowHeader(fullRequestStr)) {
      fStreamAfterSETUP = True;
    } else {
      fStreamAfterSETUP = False;
    }
```

尽管 RTP/RTCP 也支持 TCP 模式，但这种做法不是很主流，后面我们主要来看基于 UDP 单播的模式。

第八步，为会话分配网络资源，如服务器端 RTP 和 RTCP 的端口等。
```
    netAddressBits destinationAddress = 0;
    u_int8_t destinationTTL = 255;
#ifdef RTSP_ALLOW_CLIENT_DESTINATION_SETTING
    if (clientsDestinationAddressStr != NULL) {
      // Use the client-provided "destination" address.
      // Note: This potentially allows the server to be used in denial-of-service
      // attacks, so don't enable this code unless you're sure that clients are
      // trusted.
      destinationAddress = our_inet_addr(clientsDestinationAddressStr);
    }
    // Also use the client-provided TTL.
    destinationTTL = clientsDestinationTTL;
#endif
    delete[] clientsDestinationAddressStr;
    Port serverRTPPort(0);
    Port serverRTCPPort(0);
    
    // Make sure that we transmit on the same interface that's used by the client (in case we're a multi-homed server):
    struct sockaddr_in sourceAddr; SOCKLEN_T namelen = sizeof sourceAddr;
    getsockname(ourClientConnection->fClientInputSocket, (struct sockaddr*)&sourceAddr, &namelen);
    netAddressBits origSendingInterfaceAddr = SendingInterfaceAddr;
    netAddressBits origReceivingInterfaceAddr = ReceivingInterfaceAddr;
    // NOTE: The following might not work properly, so we ifdef it out for now:
#ifdef HACK_FOR_MULTIHOMED_SERVERS
    ReceivingInterfaceAddr = SendingInterfaceAddr = sourceAddr.sin_addr.s_addr;
#endif
    
    subsession->getStreamParameters(fOurSessionId, ourClientConnection->fClientAddr.sin_addr.s_addr,
        clientRTPPort, clientRTCPPort,
        fStreamStates[trackNum].tcpSocketNum, rtpChannelId, rtcpChannelId,
        destinationAddress, destinationTTL, fIsMulticast,
        serverRTPPort, serverRTCPPort,
        fStreamStates[trackNum].streamToken);
    SendingInterfaceAddr = origSendingInterfaceAddr;
    ReceivingInterfaceAddr = origReceivingInterfaceAddr;

    AddressString destAddrStr(destinationAddress);
    AddressString sourceAddrStr(sourceAddr);
```
后面我们再来分析 `H264VideoFileServerMediaSubsession` 更详细地处理过程。

第九步，生成超时参数字符串。
```
    if (fOurRTSPServer.fReclamationSeconds > 0) {
      sprintf(timeoutParameterString, ";timeout=%u",
          fOurRTSPServer.fReclamationSeconds);
    } else {
      timeoutParameterString[0] = '\0';
    }
```

第十步，生成响应消息。我们仅来看，RTP 的 UDP 单播模式响应消息的生成。
```
    } else {
      switch (streamingMode) {
      case RTP_UDP: {
        snprintf((char*) ourClientConnection->fResponseBuffer, sizeof ourClientConnection->fResponseBuffer,
            "RTSP/1.0 200 OK\r\n"
            "CSeq: %s\r\n"
            "%s"
            "Transport: RTP/AVP;unicast;destination=%s;source=%s;client_port=%d-%d;server_port=%d-%d\r\n"
            "Session: %08X%s\r\n\r\n",
            ourClientConnection->fCurrentCSeq,
            dateHeader(),
            destAddrStr.val(), sourceAddrStr.val(), ntohs(clientRTPPort.num()), ntohs(clientRTCPPort.num()), ntohs(serverRTPPort.num()), ntohs(serverRTCPPort.num()),
            fOurSessionId, timeoutParameterString);
        break;
      }
```

生成的消息看起来大概就像下面这样：
```
RTSP/1.0 200 OK
CSeq: 3
Date: Sat, Sep 02 2017 08:54:03 GMT
Transport: RTP/AVP;unicast;destination=10.240.248.20;source=10.240.248.20;client_port=19586-19587;server_port=6970-6971
Session: D10C8C71;timeout=65
```

通过 `Transport: ` 将协商的通信参数，如服务器端为会话分配的用于收发 RTP/RTCP 包的 UDP 端口。

抛开容错，总结一下 SETUP 请求的常规处理流程：
1. 为会话创建 `ServerMediaSession`。
2. 解析请求头 `Transport:` 中包含的关于客户端请求的传输方式的内容，如使用 UDP 传输还是用 TCP 传输，客户端为 RTP/RTCP 传输开辟的端口等。
3. 解析请求头中的 `Range:` 和 `x-playNow:` 以支持 `SETUP` 和 `PLAY` 的合并。
4. 为会话分配网络资源，如服务器端的 RTP/RTCP 端口等。
5. 产生响应消息。

在 `ServerMediaSubsession` 中，一些更详细地处理过程，留待后面分析。

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
