---
title: live555 源码分析： PLAY 的处理
date: 2017-09-05 17:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

在 `SETUP` 请求之后，客户端会发起 `PLAY` 请求，以请求服务器开始传输音视频数据。在 `PLAY` 请求执行时，一定是已经执行过 `SETUP` 请求，建立好了客户端会话，因而会与其它要求客户端会话已经建立的请求一起，通过 `clientSession->handleCmd_withinSession()` 执行：
```
      } else if (strcmp(cmdName, "TEARDOWN") == 0
          || strcmp(cmdName, "PLAY") == 0 || strcmp(cmdName, "PAUSE") == 0
          || strcmp(cmdName, "GET_PARAMETER") == 0
          || strcmp(cmdName, "SET_PARAMETER") == 0) {
        if (clientSession != NULL) {
          clientSession->handleCmd_withinSession(this, cmdName, urlPreSuffix,
              urlSuffix, (char const*) fRequestBuffer);
        } else {
          handleCmd_sessionNotFound();
        }
      }
```
<!--more-->
`RTSPServer::RTSPClientSession::handleCmd_withinSession()` 的定义如下：
```
void RTSPServer::RTSPClientSession
::handleCmd_withinSession(RTSPServer::RTSPClientConnection* ourClientConnection,
    char const* cmdName,
    char const* urlPreSuffix, char const* urlSuffix,
    char const* fullRequestStr) {
  // This will either be:
  // - a non-aggregated operation, if "urlPreSuffix" is the session (stream)
  //   name and "urlSuffix" is the subsession (track) name, or
  // - an aggregated operation, if "urlSuffix" is the session (stream) name,
  //   or "urlPreSuffix" is the session (stream) name, and "urlSuffix" is empty,
  //   or "urlPreSuffix" and "urlSuffix" are both nonempty, but when concatenated, (with "/") form the session (stream) name.
  // Begin by figuring out which of these it is:
  ServerMediaSubsession* subsession;
  
  if (fOurServerMediaSession == NULL) { // There wasn't a previous SETUP!
    ourClientConnection->handleCmd_notSupported();
    return;
  } else if (urlSuffix[0] != '\0' && strcmp(fOurServerMediaSession->streamName(), urlPreSuffix) == 0) {
    // Non-aggregated operation.
    // Look up the media subsession whose track id is "urlSuffix":
    ServerMediaSubsessionIterator iter(*fOurServerMediaSession);
    while ((subsession = iter.next()) != NULL) {
      if (strcmp(subsession->trackId(), urlSuffix) == 0) break; // success
    }
    if (subsession == NULL) { // no such track!
      ourClientConnection->handleCmd_notFound();
      return;
    }
  } else if (strcmp(fOurServerMediaSession->streamName(), urlSuffix) == 0 ||
      (urlSuffix[0] == '\0' && strcmp(fOurServerMediaSession->streamName(), urlPreSuffix) == 0)) {
    // Aggregated operation
    subsession = NULL;
  } else if (urlPreSuffix[0] != '\0' && urlSuffix[0] != '\0') {
    // Aggregated operation, if <urlPreSuffix>/<urlSuffix> is the session (stream) name:
    unsigned const urlPreSuffixLen = strlen(urlPreSuffix);
    if (strncmp(fOurServerMediaSession->streamName(), urlPreSuffix, urlPreSuffixLen) == 0
        && fOurServerMediaSession->streamName()[urlPreSuffixLen] == '/'
        && strcmp(&(fOurServerMediaSession->streamName())[urlPreSuffixLen + 1], urlSuffix) == 0) {
      subsession = NULL;
    } else {
      ourClientConnection->handleCmd_notFound();
      return;
    }
  } else { // the request doesn't match a known stream and/or track at all!
    ourClientConnection->handleCmd_notFound();
    return;
  }
  
  if (strcmp(cmdName, "TEARDOWN") == 0) {
    handleCmd_TEARDOWN(ourClientConnection, subsession);
  } else if (strcmp(cmdName, "PLAY") == 0) {
    handleCmd_PLAY(ourClientConnection, subsession, fullRequestStr);
  } else if (strcmp(cmdName, "PAUSE") == 0) {
    handleCmd_PAUSE(ourClientConnection, subsession);
  } else if (strcmp(cmdName, "GET_PARAMETER") == 0) {
    handleCmd_GET_PARAMETER(ourClientConnection, subsession, fullRequestStr);
  } else if (strcmp(cmdName, "SET_PARAMETER") == 0) {
    handleCmd_SET_PARAMETER(ourClientConnection, subsession, fullRequestStr);
  }
}
```

在这个函数中，首先会找到请求的 URL 所相应的 `ServerMediaSubsession`，如果查找失败，会直接返回 404 并结束处理；然后即是根据请求的具体类型，通过不同的方法进行处理。

找到的 `ServerMediaSubsession` 存储于 `subsession` 中。`subsession` 值为空并不表示查找失败。这里通过值为 `NULL` 表示操作应用于 `fOurServerMediaSession` 描述的整个流媒体会话。请求的资源的路径可以分为两部分，分别是资源路径前缀 `urlPreSuffix` 和标识具体 track 的后缀 `urlSuffix`。live555 中通过 `ServerMediaSession` 和 `ServerMediaSubsession` 两级对象管理流媒体会话。

根据请求的资源的路径与流媒体会话状态的对应关系的不同，判断操作是否可以支持，以及操作应用的范围。

查找 `ServerMediaSubsession` 之后，`PLAY` 请求通过`RTSPServer::RTSPClientSession::handleCmd_PLAY()` 函数处理：
```
void RTSPServer::RTSPClientSession
::handleCmd_PLAY(RTSPServer::RTSPClientConnection* ourClientConnection,
    ServerMediaSubsession* subsession, char const* fullRequestStr) {
  char* rtspURL = fOurRTSPServer.rtspURL(fOurServerMediaSession,
      ourClientConnection->fClientInputSocket);
  unsigned rtspURLSize = strlen(rtspURL);
  
  // Parse the client's "Scale:" header, if any:
  float scale;
  Boolean sawScaleHeader = parseScaleHeader(fullRequestStr, scale);

  // Try to set the stream's scale factor to this value:
  if (subsession == NULL /*aggregate op*/) {
    fOurServerMediaSession->testScaleFactor(scale);
  } else {
    subsession->testScaleFactor(scale);
  }

  char buf[100];
  char* scaleHeader;
  if (!sawScaleHeader) {
    buf[0] = '\0'; // Because we didn't see a Scale: header, don't send one back
  } else {
    sprintf(buf, "Scale: %f\r\n", scale);
  }
  scaleHeader = strDup(buf);
  
  // Parse the client's "Range:" header, if any:
  float duration = 0.0;
  double rangeStart = 0.0, rangeEnd = 0.0;
  char* absStart = NULL; char* absEnd = NULL;
  Boolean startTimeIsNow;
  Boolean sawRangeHeader
    = parseRangeHeader(fullRequestStr, rangeStart, rangeEnd, absStart, absEnd, startTimeIsNow);
  
  if (sawRangeHeader && absStart == NULL/*not seeking by 'absolute' time*/) {
    // Use this information, plus the stream's duration (if known), to create our own "Range:" header, for the response:
    duration = subsession == NULL /*aggregate op*/
      ? fOurServerMediaSession->duration() : subsession->duration();
    if (duration < 0.0) {
      // We're an aggregate PLAY, but the subsessions have different durations.
      // Use the largest of these durations in our header
      duration = -duration;
    }
    
    // Make sure that "rangeStart" and "rangeEnd" (from the client's "Range:" header)
    // have sane values, before we send back our own "Range:" header in our response:
    if (rangeStart < 0.0) rangeStart = 0.0;
    else if (rangeStart > duration) rangeStart = duration;
    if (rangeEnd < 0.0) rangeEnd = 0.0;
    else if (rangeEnd > duration) rangeEnd = duration;
    if ((scale > 0.0 && rangeStart > rangeEnd && rangeEnd > 0.0)
        || (scale < 0.0 && rangeStart < rangeEnd)) {
      // "rangeStart" and "rangeEnd" were the wrong way around; swap them:
      double tmp = rangeStart;
      rangeStart = rangeEnd;
      rangeEnd = tmp;
    }
  }
  
  // Create a "RTP-Info:" line.  It will get filled in from each subsession's state:
  char const* rtpInfoFmt =
    "%s" // "RTP-Info:", plus any preceding rtpInfo items
    "%s" // comma separator, if needed
    "url=%s/%s"
    ";seq=%d"
    ";rtptime=%u"
    ;
  unsigned rtpInfoFmtSize = strlen(rtpInfoFmt);
  char* rtpInfo = strDup("RTP-Info: ");
  unsigned i, numRTPInfoItems = 0;
  
  // Do any required seeking/scaling on each subsession, before starting streaming.
  // (However, we don't do this if the "PLAY" request was for just a single subsession
  // of a multiple-subsession stream; for such streams, seeking/scaling can be done
  // only with an aggregate "PLAY".)
  for (i = 0; i < fNumStreamStates; ++i) {
    if (subsession == NULL /* means: aggregated operation */ || fNumStreamStates == 1) {
      if (fStreamStates[i].subsession != NULL) {
        if (sawScaleHeader) {
          fStreamStates[i].subsession->setStreamScale(fOurSessionId, fStreamStates[i].streamToken, scale);
        }
        if (absStart != NULL) {
          // Special case handling for seeking by 'absolute' time:

          fStreamStates[i].subsession->seekStream(fOurSessionId, fStreamStates[i].streamToken, absStart, absEnd);
        } else {
          // Seeking by relative (NPT) time:

          u_int64_t numBytes;
          if (!sawRangeHeader || startTimeIsNow) {
            // We're resuming streaming without seeking, so we just do a 'null' seek
            // (to get our NPT, and to specify when to end streaming):
            fStreamStates[i].subsession->nullSeekStream(fOurSessionId, fStreamStates[i].streamToken,
                rangeEnd, numBytes);
          } else {
            // We do a real 'seek':
            double streamDuration = 0.0; // by default; means: stream until the end of the media
            if (rangeEnd > 0.0 && (rangeEnd + 0.001) < duration) {
              // the 0.001 is because we limited the values to 3 decimal places
              // We want the stream to end early.  Set the duration we want:
              streamDuration = rangeEnd - rangeStart;
              if (streamDuration < 0.0) streamDuration = -streamDuration;
              // should happen only if scale < 0.0
            }
            fStreamStates[i].subsession->seekStream(fOurSessionId, fStreamStates[i].streamToken,
                rangeStart, streamDuration, numBytes);
          }
        }
      }
    }
  }
  
  // Create the "Range:" header that we'll send back in our response.
  // (Note that we do this after seeking, in case the seeking operation changed the range start time.)
  if (absStart != NULL) {
    // We're seeking by 'absolute' time:
    if (absEnd == NULL) {
      sprintf(buf, "Range: clock=%s-\r\n", absStart);
    } else {
      sprintf(buf, "Range: clock=%s-%s\r\n", absStart, absEnd);
    }
    delete[] absStart; delete[] absEnd;
  } else {
    // We're seeking by relative (NPT) time:
    if (!sawRangeHeader || startTimeIsNow) {
      // We didn't seek, so in our response, begin the range with the current NPT (normal play time):
      float curNPT = 0.0;
      for (i = 0; i < fNumStreamStates; ++i) {
        if (subsession == NULL /* means: aggregated operation */
        || subsession == fStreamStates[i].subsession) {
          if (fStreamStates[i].subsession == NULL) continue;
          float npt = fStreamStates[i].subsession->getCurrentNPT(fStreamStates[i].streamToken);
          if (npt > curNPT)
            curNPT = npt;
          // Note: If this is an aggregate "PLAY" on a multi-subsession stream,
          // then it's conceivable that the NPTs of each subsession may differ
          // (if there has been a previous seek on just one subsession).
          // In this (unusual) case, we just return the largest NPT; I hope that turns out OK...
        }
      }
      rangeStart = curNPT;
    }

    if (rangeEnd == 0.0 && scale >= 0.0) {
      sprintf(buf, "Range: npt=%.3f-\r\n", rangeStart);
    } else {
      sprintf(buf, "Range: npt=%.3f-%.3f\r\n", rangeStart, rangeEnd);
    }
  }
  char* rangeHeader = strDup(buf);
  
  // Now, start streaming:
  for (i = 0; i < fNumStreamStates; ++i) {
    if (subsession == NULL /* means: aggregated operation */
    || subsession == fStreamStates[i].subsession) {
      unsigned short rtpSeqNum = 0;
      unsigned rtpTimestamp = 0;
      if (fStreamStates[i].subsession == NULL) continue;
      fStreamStates[i].subsession->startStream(fOurSessionId,
          fStreamStates[i].streamToken,
          (TaskFunc*) noteClientLiveness, this,
          rtpSeqNum, rtpTimestamp,
          RTSPServer::RTSPClientConnection::handleAlternativeRequestByte, ourClientConnection);
      const char *urlSuffix = fStreamStates[i].subsession->trackId();
      char* prevRTPInfo = rtpInfo;
      unsigned rtpInfoSize = rtpInfoFmtSize
          + strlen(prevRTPInfo)
          + 1
          + rtspURLSize + strlen(urlSuffix)
          + 5 /*max unsigned short len*/
          + 10 /*max unsigned (32-bit) len*/
          + 2 /*allows for trailing \r\n at final end of string*/;
      rtpInfo = new char[rtpInfoSize];
      sprintf(rtpInfo, rtpInfoFmt, prevRTPInfo,
          numRTPInfoItems++ == 0 ? "" : ",", rtspURL, urlSuffix, rtpSeqNum,
          rtpTimestamp);
      delete[] prevRTPInfo;
    }
  }
  if (numRTPInfoItems == 0) {
    rtpInfo[0] = '\0';
  } else {
    unsigned rtpInfoLen = strlen(rtpInfo);
    rtpInfo[rtpInfoLen] = '\r';
    rtpInfo[rtpInfoLen + 1] = '\n';
    rtpInfo[rtpInfoLen + 2] = '\0';
  }
  
  // Fill in the response:
  snprintf((char*) ourClientConnection->fResponseBuffer,
      sizeof ourClientConnection->fResponseBuffer, "RTSP/1.0 200 OK\r\n"
          "CSeq: %s\r\n"
          "%s"
          "%s"
          "%s"
          "Session: %08X\r\n"
          "%s\r\n",
          ourClientConnection->fCurrentCSeq,
          dateHeader(),
          scaleHeader,
          rangeHeader,
          fOurSessionId,
          rtpInfo);
  delete[] rtpInfo;
  delete[] rangeHeader;
  delete[] scaleHeader;
  delete[] rtspURL;
}
```
在具体分析这个函数之前，先来看一个 `PLAY` 请求/响应的示例。`PLAY` 请求示例：
```
PLAY rtsp://10.240.248.20:8554/video/raw_h264_stream.264/ RTSP/1.0
Range: npt=0.000-
CSeq: 4
User-Agent: Lavf56.40.101
Session: D10C8C71
```

 `PLAY` 响应的示例：
```
RTSP/1.0 200 OK
CSeq: 4
Date: Sat, Sep 02 2017 08:54:03 GMT
Range: npt=0.000-
Session: D10C8C71
RTP-Info: url=rtsp://10.240.248.20:8554/video/raw_h264_stream.264/track1;seq=12647;rtptime=2457491257
```

然后来看 `RTSPServer::RTSPClientSession::handleCmd_PLAY()` 的具体处理过程。

第一步，解析 `Scale:` 头部，为流媒体会话的各个子会话设置 Scale，并构造要返回的 `Scale:` 头部。
```
  // Parse the client's "Scale:" header, if any:
  float scale;
  Boolean sawScaleHeader = parseScaleHeader(fullRequestStr, scale);

  // Try to set the stream's scale factor to this value:
  if (subsession == NULL /*aggregate op*/) {
    fOurServerMediaSession->testScaleFactor(scale);
  } else {
    subsession->testScaleFactor(scale);
  }

  char buf[100];
  char* scaleHeader;
  if (!sawScaleHeader) {
    buf[0] = '\0'; // Because we didn't see a Scale: header, don't send one back
  } else {
    sprintf(buf, "Scale: %f\r\n", scale);
  }
  scaleHeader = strDup(buf);
```

用于解析 `Scale:` 头部的 `parseScaleHeader()` 定义如下：
```
Boolean parseScaleHeader(char const* buf, float& scale) {
  // Initialize the result parameter to a default value:
  scale = 1.0;

  // First, find "Scale:"
  while (1) {
    if (*buf == '\0') return False; // not found
    if (_strncasecmp(buf, "Scale:", 6) == 0) break;
    ++buf;
  }

  char const* fields = buf + 6;
  while (*fields == ' ') ++fields;
  float sc;
  if (sscanf(fields, "%f", &sc) == 1) {
    scale = sc;
  } else {
    return False; // The header is malformed
  }

  return True;
}
```
`scale` 默认情况下会被设置为1.0。

解释一下 scale 值的作用。根据 RTSP 的规范 RFC 2326 [12.34 Scale](https://tools.ietf.org/html/rfc2326#section-12.34)，scale 值是用于控制播放速度的。scale 值为 1，表示以正常的向前观看速率正常播放或记录。如果不是 1，值对应于相对于正常观看速率的比率。比如，值为 2 的比率表示两倍的正常观看速率（“快进”），而值为 0.5 的比率表示一半的观看速率。换句话说，值为 2 的比率播放的时间增长速率是墙上时钟的两倍。对于流逝的每一秒时间，将有 2 秒的内容被发送。负值表示相反的方向。

除非 `Speed` 参数另有要求，否则数据速率不得更改。Scale
 变化的实现取决于服务器和媒体类型。对于视频，服务器可以，比如，传送一个关键帧或选择的关键帧。对于音频，它可以在保持音调的同时缩放音频，或者更不希望地传送音频片段。服务器应尝试提供近似的观看速率，但可以限制其支持的比例值范围。响应要包含服务器选择的实际的 scale。如果请求包含了 `Range` 参数，新的 scale 值将在此时生效。

应用于整个流媒体会话的 `PLAY` 请求，通过 `ServerMediaSession::testScaleFactor(float& scale)` 设置 scale，应用于具体子会话的 `PLAY` 请求，则通过 `ServerMediaSubsession` 的 `testScaleFactor(float& scale)` 设置。

对于我们的 `H264VideoFileServerMediaSubsession`，`testScaleFactor(float& scale)` 的实现还是在 `ServerMediaSubsession` 中：
```
void ServerMediaSubsession::testScaleFactor(float& scale) {
  // default implementation: Support scale = 1 only
  scale = 1;
}
```

`ServerMediaSession::testScaleFactor(float& scale)` 的定义如下：
```
void ServerMediaSession::testScaleFactor(float& scale) {
  // First, try setting all subsessions to the desired scale.
  // If the subsessions' actual scales differ from each other, choose the
  // value that's closest to 1, and then try re-setting all subsessions to that
  // value.  If the subsessions' actual scales still differ, re-set them all to 1.
  float minSSScale = 1.0;
  float maxSSScale = 1.0;
  float bestSSScale = 1.0;
  float bestDistanceTo1 = 0.0;
  ServerMediaSubsession* subsession;
  for (subsession = fSubsessionsHead; subsession != NULL;
       subsession = subsession->fNext) {
    float ssscale = scale;
    subsession->testScaleFactor(ssscale);
    if (subsession == fSubsessionsHead) { // this is the first subsession
      minSSScale = maxSSScale = bestSSScale = ssscale;
      bestDistanceTo1 = (float)fabs(ssscale - 1.0f);
    } else {
      if (ssscale < minSSScale) {
        minSSScale = ssscale;
      } else if (ssscale > maxSSScale) {
        maxSSScale = ssscale;
      }

      float distanceTo1 = (float) fabs(ssscale - 1.0f);
      if (distanceTo1 < bestDistanceTo1) {
        bestSSScale = ssscale;
        bestDistanceTo1 = distanceTo1;
      }
    }
  }
  if (minSSScale == maxSSScale) {
    // All subsessions are at the same scale: minSSScale == bestSSScale == maxSSScale
    scale = minSSScale;
    return;
  }

  // The scales for each subsession differ.  Try to set each one to the value
  // that's closest to 1:
  for (subsession = fSubsessionsHead; subsession != NULL;
       subsession = subsession->fNext) {
    float ssscale = bestSSScale;
    subsession->testScaleFactor(ssscale);
    if (ssscale != bestSSScale) break; // no luck
  }
  if (subsession == NULL) {
    // All subsessions are at the same scale: bestSSScale
    scale = bestSSScale;
    return;
  }

  // Still no luck.  Set each subsession's scale to 1:
  for (subsession = fSubsessionsHead; subsession != NULL;
       subsession = subsession->fNext) {
    float ssscale = 1;
    subsession->testScaleFactor(ssscale);
  }
  scale = 1;
}
```
在这个函数中，执行的步骤如下：
Step 1：尝试将所有子会话的 scale 设置客户端请求的值，并记录下子会话选择的 scale 的最大值，最小值，以及与 1 最接近的值。
Step 2：第 1 步执行过程中，所有子会话选择的 scale 值相同，则将该 scale 返回给调用者。
Step 3：第 1 步执行过程中，所有子会话选择的 scale 值不同，则尝试将与 1 最接近的 scale 值设置给所有的。
Step 4：第 3 步可以成功将与 1 最接近的 scale 值设置给所有子会话，则将该 scale 返回给调用者。
Step 5：第 3 步无法将与 1 最接近的 scale 值设置给所有子会话，则将每个子会话的 scale 值设置为 1。

回到 `RTSPServer::RTSPClientSession::handleCmd_PLAY()`。

第二步，解析客户端的 `Range:` 头部，并根据 scale 值更新 range 值。
```
  // Parse the client's "Range:" header, if any:
  float duration = 0.0;
  double rangeStart = 0.0, rangeEnd = 0.0;
  char* absStart = NULL; char* absEnd = NULL;
  Boolean startTimeIsNow;
  Boolean sawRangeHeader
    = parseRangeHeader(fullRequestStr, rangeStart, rangeEnd, absStart, absEnd, startTimeIsNow);
  
  if (sawRangeHeader && absStart == NULL/*not seeking by 'absolute' time*/) {
    // Use this information, plus the stream's duration (if known), to create our own "Range:" header, for the response:
    duration = subsession == NULL /*aggregate op*/
      ? fOurServerMediaSession->duration() : subsession->duration();
    if (duration < 0.0) {
      // We're an aggregate PLAY, but the subsessions have different durations.
      // Use the largest of these durations in our header
      duration = -duration;
    }
    
    // Make sure that "rangeStart" and "rangeEnd" (from the client's "Range:" header)
    // have sane values, before we send back our own "Range:" header in our response:
    if (rangeStart < 0.0) rangeStart = 0.0;
    else if (rangeStart > duration) rangeStart = duration;
    if (rangeEnd < 0.0) rangeEnd = 0.0;
    else if (rangeEnd > duration) rangeEnd = duration;
    if ((scale > 0.0 && rangeStart > rangeEnd && rangeEnd > 0.0)
        || (scale < 0.0 && rangeStart < rangeEnd)) {
      // "rangeStart" and "rangeEnd" were the wrong way around; swap them:
      double tmp = rangeStart;
      rangeStart = rangeEnd;
      rangeEnd = tmp;
    }
  }
```

第四步，创建 `RTP-Info:` 行：
```
  // Create a "RTP-Info:" line.  It will get filled in from each subsession's state:
  char const* rtpInfoFmt =
    "%s" // "RTP-Info:", plus any preceding rtpInfo items
    "%s" // comma separator, if needed
    "url=%s/%s"
    ";seq=%d"
    ";rtptime=%u"
    ;
  unsigned rtpInfoFmtSize = strlen(rtpInfoFmt);
  char* rtpInfo = strDup("RTP-Info: ");
  unsigned i, numRTPInfoItems = 0;
```

第五步，在开始流式传输之前，对每个子会话进行所需的 seeking/scaling，即设置播放进度和播放速率。
```
  for (i = 0; i < fNumStreamStates; ++i) {
    if (subsession == NULL /* means: aggregated operation */ || fNumStreamStates == 1) {
      if (fStreamStates[i].subsession != NULL) {
        if (sawScaleHeader) {
          fStreamStates[i].subsession->setStreamScale(fOurSessionId, fStreamStates[i].streamToken, scale);
        }
        if (absStart != NULL) {
          // Special case handling for seeking by 'absolute' time:

          fStreamStates[i].subsession->seekStream(fOurSessionId, fStreamStates[i].streamToken, absStart, absEnd);
        } else {
          // Seeking by relative (NPT) time:

          u_int64_t numBytes;
          if (!sawRangeHeader || startTimeIsNow) {
            // We're resuming streaming without seeking, so we just do a 'null' seek
            // (to get our NPT, and to specify when to end streaming):
            fStreamStates[i].subsession->nullSeekStream(fOurSessionId, fStreamStates[i].streamToken,
                rangeEnd, numBytes);
          } else {
            // We do a real 'seek':
            double streamDuration = 0.0; // by default; means: stream until the end of the media
            if (rangeEnd > 0.0 && (rangeEnd + 0.001) < duration) {
              // the 0.001 is because we limited the values to 3 decimal places
              // We want the stream to end early.  Set the duration we want:
              streamDuration = rangeEnd - rangeStart;
              if (streamDuration < 0.0) streamDuration = -streamDuration;
              // should happen only if scale < 0.0
            }
            fStreamStates[i].subsession->seekStream(fOurSessionId, fStreamStates[i].streamToken,
                rangeStart, streamDuration, numBytes);
          }
        }
      }
    }
  }
```

第六步，创建将会在响应中发送的 `Range:` 头部。
```
  if (absStart != NULL) {
    // We're seeking by 'absolute' time:
    if (absEnd == NULL) {
      sprintf(buf, "Range: clock=%s-\r\n", absStart);
    } else {
      sprintf(buf, "Range: clock=%s-%s\r\n", absStart, absEnd);
    }
    delete[] absStart; delete[] absEnd;
  } else {
    // We're seeking by relative (NPT) time:
    if (!sawRangeHeader || startTimeIsNow) {
      // We didn't seek, so in our response, begin the range with the current NPT (normal play time):
      float curNPT = 0.0;
      for (i = 0; i < fNumStreamStates; ++i) {
        if (subsession == NULL /* means: aggregated operation */
        || subsession == fStreamStates[i].subsession) {
          if (fStreamStates[i].subsession == NULL) continue;
          float npt = fStreamStates[i].subsession->getCurrentNPT(fStreamStates[i].streamToken);
          if (npt > curNPT)
            curNPT = npt;
          // Note: If this is an aggregate "PLAY" on a multi-subsession stream,
          // then it's conceivable that the NPTs of each subsession may differ
          // (if there has been a previous seek on just one subsession).
          // In this (unusual) case, we just return the largest NPT; I hope that turns out OK...
        }
      }
      rangeStart = curNPT;
    }

    if (rangeEnd == 0.0 && scale >= 0.0) {
      sprintf(buf, "Range: npt=%.3f-\r\n", rangeStart);
    } else {
      sprintf(buf, "Range: npt=%.3f-%.3f\r\n", rangeStart, rangeEnd);
    }
  }
  char* rangeHeader = strDup(buf);
```

第七步，启动流媒体传输，并产生最终的 `RTP-Info:` 行。
```
  // Now, start streaming:
  for (i = 0; i < fNumStreamStates; ++i) {
    if (subsession == NULL /* means: aggregated operation */
    || subsession == fStreamStates[i].subsession) {
      unsigned short rtpSeqNum = 0;
      unsigned rtpTimestamp = 0;
      if (fStreamStates[i].subsession == NULL) continue;
      fStreamStates[i].subsession->startStream(fOurSessionId,
          fStreamStates[i].streamToken,
          (TaskFunc*) noteClientLiveness, this,
          rtpSeqNum, rtpTimestamp,
          RTSPServer::RTSPClientConnection::handleAlternativeRequestByte, ourClientConnection);
      const char *urlSuffix = fStreamStates[i].subsession->trackId();
      char* prevRTPInfo = rtpInfo;
      unsigned rtpInfoSize = rtpInfoFmtSize
          + strlen(prevRTPInfo)
          + 1
          + rtspURLSize + strlen(urlSuffix)
          + 5 /*max unsigned short len*/
          + 10 /*max unsigned (32-bit) len*/
          + 2 /*allows for trailing \r\n at final end of string*/;
      rtpInfo = new char[rtpInfoSize];
      sprintf(rtpInfo, rtpInfoFmt, prevRTPInfo,
          numRTPInfoItems++ == 0 ? "" : ",", rtspURL, urlSuffix, rtpSeqNum,
          rtpTimestamp);
      delete[] prevRTPInfo;
    }
  }
  if (numRTPInfoItems == 0) {
    rtpInfo[0] = '\0';
  } else {
    unsigned rtpInfoLen = strlen(rtpInfo);
    rtpInfo[rtpInfoLen] = '\r';
    rtpInfo[rtpInfoLen + 1] = '\n';
    rtpInfo[rtpInfoLen + 2] = '\0';
  }
```

第八步，产生最终的响应，并释放临时分配的内存。
```
  // Fill in the response:
  snprintf((char*) ourClientConnection->fResponseBuffer,
      sizeof ourClientConnection->fResponseBuffer, "RTSP/1.0 200 OK\r\n"
          "CSeq: %s\r\n"
          "%s"
          "%s"
          "%s"
          "Session: %08X\r\n"
          "%s\r\n",
          ourClientConnection->fCurrentCSeq,
          dateHeader(),
          scaleHeader,
          rangeHeader,
          fOurSessionId,
          rtpInfo);
  delete[] rtpInfo;
  delete[] rangeHeader;
  delete[] scaleHeader;
  delete[] rtspURL;
```

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# live555 源码分析系列文章
[live555 源码分析：简介](https://www.wolfcstech.com/2017/08/28/live555_src_analysis_introduction/)
[live555 源码分析：基础设施](https://www.wolfcstech.com/2017/08/30/live555_src_analysis_infrasture/)
[live555 源码分析：MediaSever](https://www.wolfcstech.com/2017/08/31/live555_src_analysis_mediaserver/)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](https://www.wolfcstech.com/2017/09/01/live555_src_analysis_rtsp_rtp_rtcp_wireshark/)
[live555 源码分析：RTSPServer](https://www.wolfcstech.com/2017/09/03/live555_src_analysis_rtspserver/)
[live555 源码分析： DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)
[live555 源码分析： SETUP 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_setup/)