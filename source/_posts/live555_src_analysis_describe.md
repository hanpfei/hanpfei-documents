---
title: live555 源码分析： DESCRIBE 的处理
date: 2017-09-04 15:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

前面在 [live555 源码分析：RTSPServer](http://www.jianshu.com/p/62387ac1fe41) 中分析了 live555 中处理 RTSP 请求的大体流程，并分析了处理起来没有那么复杂的一些方法，如 `OPTIONS`，`GET_PARAMETER`，`SET_PARAMETER` 等。篇幅所限，没有分析最为重要的 `DESCRIBE`，`SETUP` 和 `PLAY` 这些方法的处理。
<!--more-->
本文继续分析 live555 对 RTSP 请求，分析 `DESCRIBE`，`SETUP` 和 `PLAY` 这些最为重要的方法的处理。

在 `RTSPServer::RTSPClientConnection::handleRequestBytes(int newBytesRead)` 中，通过调用 `handleCmd_DESCRIBE()` 函数处理 `DESCRIBE` 请求，如下所示：
```
      } else if (strcmp(cmdName, "DESCRIBE") == 0) {
        handleCmd_DESCRIBE(urlPreSuffix, urlSuffix,
            (char const*) fRequestBuffer);
      }
```

`handleCmd_DESCRIBE()` 函数的定义是这样的：
```
void RTSPServer::RTSPClientConnection
::handleCmd_DESCRIBE(char const* urlPreSuffix, char const* urlSuffix, char const* fullRequestStr) {
  ServerMediaSession* session = NULL;
  char* sdpDescription = NULL;
  char* rtspURL = NULL;
  do {
    char urlTotalSuffix[2*RTSP_PARAM_STRING_MAX];
        // enough space for urlPreSuffix/urlSuffix'\0'
    urlTotalSuffix[0] = '\0';
    if (urlPreSuffix[0] != '\0') {
      strcat(urlTotalSuffix, urlPreSuffix);
      strcat(urlTotalSuffix, "/");
    }
    strcat(urlTotalSuffix, urlSuffix);
    
    if (!authenticationOK("DESCRIBE", urlTotalSuffix, fullRequestStr)) break;
    
    // We should really check that the request contains an "Accept:" #####
    // for "application/sdp", because that's what we're sending back #####
    
    // Begin by looking up the "ServerMediaSession" object for the specified "urlTotalSuffix":
    session = fOurServer.lookupServerMediaSession(urlTotalSuffix);
    if (session == NULL) {
      handleCmd_notFound();
      break;
    }
    
    // Increment the "ServerMediaSession" object's reference count, in case someone removes it
    // while we're using it:
    session->incrementReferenceCount();

    // Then, assemble a SDP description for this session:
    sdpDescription = session->generateSDPDescription();
    if (sdpDescription == NULL) {
      // This usually means that a file name that was specified for a
      // "ServerMediaSubsession" does not exist.
      setRTSPResponse("404 File Not Found, Or In Incorrect Format");
      break;
    }
    unsigned sdpDescriptionSize = strlen(sdpDescription);
    
    // Also, generate our RTSP URL, for the "Content-Base:" header
    // (which is necessary to ensure that the correct URL gets used in subsequent "SETUP" requests).
    rtspURL = fOurRTSPServer.rtspURL(session, fClientInputSocket);

    snprintf((char*) fResponseBuffer, sizeof fResponseBuffer,
        "RTSP/1.0 200 OK\r\nCSeq: %s\r\n"
        "%s"
        "Content-Base: %s/\r\n"
        "Content-Type: application/sdp\r\n"
        "Content-Length: %d\r\n\r\n"
        "%s",
        fCurrentCSeq,
        dateHeader(),
        rtspURL,
        sdpDescriptionSize,
        sdpDescription);
  } while (0);
  
  if (session != NULL) {
    // Decrement its reference count, now that we're done using it:
    session->decrementReferenceCount();
    if (session->referenceCount() == 0 && session->deleteWhenUnreferenced()) {
      fOurServer.removeServerMediaSession(session);
    }
  }

  delete[] sdpDescription;
  delete[] rtspURL;
}
```

`handleCmd_DESCRIBE()` 函数通过一个 do-while(0) 结构实现对 `DESCRIBE` 方法的处理。do-while(0) 结构的好处大概是，在出错时，即无需直接返回，搞乱控制流，又可以在不用 goto 语句的情况下，跳到函数的结尾处吧。

这个函数中 `DESCRIBE` 操作整体的执行流程为：
1.首先对在 URL 上执行 `DESCRIBE` 操作的权限的认证，“LIVE555 Media Server” 不提供认证，认证将总是成功。
2. 查找或创建一个 `ServerMediaSession` 结构，用于操作媒体流的元信息。查找根据 URL 中资源的路径进行，对于 “LIVE555 Media Server”，也是对应的文件相对于服务器运行的目录的相对路径。函数执行失败时，会直接返回 404 失败给客户端：
```
void RTSPServer::RTSPClientConnection::handleCmd_notFound() {
  setRTSPResponse("404 Stream Not Found");
}
```
3. 生成 SDP 描述。失败时，也会返回 404 失败给客户端。
4. 获得 RTSP URL。RTSP URL 由服务器的 IP 地址和 URL 路径拼接而成：
```
char* RTSPServer
::rtspURL(ServerMediaSession const* serverMediaSession, int clientSocket) const {
  char* urlPrefix = rtspURLPrefix(clientSocket);
  char const* sessionName = serverMediaSession->streamName();
  
  char* resultURL = new char[strlen(urlPrefix) + strlen(sessionName) + 1];
  sprintf(resultURL, "%s%s", urlPrefix, sessionName);
  
  delete[] urlPrefix;
  return resultURL;
}

char* RTSPServer::rtspURLPrefix(int clientSocket) const {
  struct sockaddr_in ourAddress;
  if (clientSocket < 0) {
    // Use our default IP address in the URL:
    ourAddress.sin_addr.s_addr = ReceivingInterfaceAddr != 0
      ? ReceivingInterfaceAddr
      : ourIPAddress(envir()); // hack
  } else {
    SOCKLEN_T namelen = sizeof ourAddress;
    getsockname(clientSocket, (struct sockaddr*)&ourAddress, &namelen);
  }
  
  char urlBuffer[100]; // more than big enough for "rtsp://<ip-address>:<port>/"
  
  portNumBits portNumHostOrder = ntohs(fServerPort.num());
  if (portNumHostOrder == 554 /* the default port number */) {
    sprintf(urlBuffer, "rtsp://%s/", AddressString(ourAddress).val());
  } else {
    sprintf(urlBuffer, "rtsp://%s:%hu/",
        AddressString(ourAddress).val(), portNumHostOrder);
  }

  return strDup(urlBuffer);
}
```
5. 生成响应消息。

在 `DESCRIBE` 请求中，客户端可以通过 `Accept` 向服务器表明，支持哪种媒体流会话的描述方式，但在 live555 中，似乎是认定了客户端只会请求 SDP，因而整个处理过程都按照产生媒体流 SDP 的方式进行。SDP 消息是放在响应消息的消息体中的。

# 创建/查找 ServerMediaSession
`handleCmd_DESCRIBE()` 函数通过 `lookupServerMediaSession()` 查找或创建一个 `ServerMediaSession` 结构，这个函数在 `GenericMediaServer` 类中声明：
```
  virtual ServerMediaSession*
  lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession = True);
```

它是一个虚函数，调用的实际的函数实现位于继承层次中实现了该函数的最底层的类中。对于 `DynamicRTSPServer` -> `RTSPServerSupportingHTTPStreaming` -> `RTSPServer` -> `GenericMediaServer` 这个继承层次，实现了这个方法的类有 `DynamicRTSPServer` 和 `GenericMediaServer`。这里来看 `DynamicRTSPServer` 类中的实现：
```
ServerMediaSession* DynamicRTSPServer
::lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession) {
  // First, check whether the specified "streamName" exists as a local file:
  FILE* fid = fopen(streamName, "rb");
  Boolean fileExists = fid != NULL;

  // Next, check whether we already have a "ServerMediaSession" for this file:
  ServerMediaSession* sms = RTSPServer::lookupServerMediaSession(streamName);
  Boolean smsExists = sms != NULL;

  // Handle the four possibilities for "fileExists" and "smsExists":
  if (!fileExists) {
    if (smsExists) {
      // "sms" was created for a file that no longer exists. Remove it:
      removeServerMediaSession(sms);
      sms = NULL;
    }

    return NULL;
  } else {
    if (smsExists && isFirstLookupInSession) { 
      // Remove the existing "ServerMediaSession" and create a new one, in case the underlying
      // file has changed in some way:
      removeServerMediaSession(sms); 
      sms = NULL;
    } 

    if (sms == NULL) {
      sms = createNewSMS(envir(), streamName, fid); 
      addServerMediaSession(sms);
    }

    fclose(fid);
    return sms;
  }
}
```
`DynamicRTSPServer::lookupServerMediaSession()` 首先检查对应的文件是否存在，然后通过父类的方法查找文件对应的 `ServerMediaSession` 结构是否已存在。父类方法在 `GenericMediaServer` 类中的定义如下：
```
ServerMediaSession* GenericMediaServer
::lookupServerMediaSession(char const* streamName, Boolean /*isFirstLookupInSession*/) {
  // Default implementation:
  return (ServerMediaSession*)(fServerMediaSessions->Lookup(streamName));
}
```

然后根据检查和查找的结果分为几种情况来处理：
1. 文件不存在，对应的 `ServerMediaSession` 结构已存在 -> 移除 `ServerMediaSession` 结构，返回 `NULL` 给调用者。
2. 文件不存在，对应的 `ServerMediaSession` 结构不存在 -> 返回 `NULL` 给调用者。
3. 文件存在，`ServerMediaSession` 结构存在，且是流媒体会话中的第一次查找 -> 移除 `ServerMediaSession` 结构，然后创建新的结构，保存起来，并把它返回给调用者。
4. 文件存在，(`ServerMediaSession` 结构存在，但不是流媒体会话中的第一次查找) 或者 (`ServerMediaSession` 结构不存在) -> 创建新的
 `ServerMediaSession` 结构，保存起来，并返回给调用者。

对于 `DESCRIBE` 方法而言，此时流媒体会话通常都还没有建立，因而总是会执行第一次查找的流程。

创建新的 `ServerMediaSession` 被交给 `createNewSMS()` 来完成：
```
#define NEW_SMS(description) do {\
char const* descStr = description\
    ", streamed by the LIVE555 Media Server";\
sms = ServerMediaSession::createNew(env, fileName, fileName, descStr);\
} while(0)

static ServerMediaSession* createNewSMS(UsageEnvironment& env,
					char const* fileName, FILE* /*fid*/) {
  // Use the file name extension to determine the type of "ServerMediaSession":
  char const* extension = strrchr(fileName, '.');
  if (extension == NULL) return NULL;

  ServerMediaSession* sms = NULL;
  Boolean const reuseSource = False;
. . . . . .
  } else if (strcmp(extension, ".264") == 0) {
    // Assumed to be a H.264 Video Elementary Stream file:
    NEW_SMS("H.264 Video");
    OutPacketBuffer::maxSize = 100000; // allow for some possibly large H.264 frames
    sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  } else if (strcmp(extension, ".265") == 0) {
. . . . . .
  return sms;
}
```

`createNewSMS()` 根据文件的后缀名创建 `ServerMediaSession` 结构。我们只看 H.264 格式视频的 `ServerMediaSession` 结构的创建。

`createNewSMS()` 用宏创建 `ServerMediaSession` 结构，在宏中，通过 do-while(0) 中的 `ServerMediaSession::createNew()` 创建；将输出缓冲区设置为 100000 字节；创建类型为 `H264VideoFileServerMediaSubsession` 的 `ServerMediaSubsession` 并设置给 `ServerMediaSession` 结构；然后将 `ServerMediaSession` 结构返回给调用者。

***注意，输出缓冲区对视频流中一帧的大小做了限制，对于一些分辨率比较高的视频流，这个大小可能无法满足要求，比如 1080P 的视频流，某些帧大小可能超过 100000，达到 150000，甚至更多。***

`ServerMediaSession::createNew()` 定义如下：
```
ServerMediaSession* ServerMediaSession
::createNew(UsageEnvironment& env,
	    char const* streamName, char const* info,
	    char const* description, Boolean isSSM, char const* miscSDPLines) {
  return new ServerMediaSession(env, streamName, info, description,
				isSSM, miscSDPLines);
}
. . . . . .
ServerMediaSession::ServerMediaSession(UsageEnvironment& env,
				       char const* streamName,
				       char const* info,
				       char const* description,
				       Boolean isSSM, char const* miscSDPLines)
  : Medium(env), fIsSSM(isSSM), fSubsessionsHead(NULL),
    fSubsessionsTail(NULL), fSubsessionCounter(0),
    fReferenceCount(0), fDeleteWhenUnreferenced(False) {
  fStreamName = strDup(streamName == NULL ? "" : streamName);

  char* libNamePlusVersionStr = NULL; // by default
  if (info == NULL || description == NULL) {
    libNamePlusVersionStr = new char[strlen(libNameStr) + strlen(libVersionStr) + 1];
    sprintf(libNamePlusVersionStr, "%s%s", libNameStr, libVersionStr);
  }
  fInfoSDPString = strDup(info == NULL ? libNamePlusVersionStr : info);
  fDescriptionSDPString = strDup(description == NULL ? libNamePlusVersionStr : description);
  delete[] libNamePlusVersionStr;

  fMiscSDPLines = strDup(miscSDPLines == NULL ? "" : miscSDPLines);

  gettimeofday(&fCreationTime, NULL);
}
```

`ServerMediaSession` 中 `ServerMediaSubsession` 被组织为一个单向链表。
```
Boolean
ServerMediaSession::addSubsession(ServerMediaSubsession* subsession) {
  if (subsession->fParentSession != NULL) return False; // it's already used

  if (fSubsessionsTail == NULL) {
    fSubsessionsHead = subsession;
  } else {
    fSubsessionsTail->fNext = subsession;
  }
  fSubsessionsTail = subsession;

  subsession->fParentSession = this;
  subsession->fTrackNumber = ++fSubsessionCounter;
  return True;
}
```
新加的 `ServerMediaSubsession` 总是会被放在链表的尾部。

# 生成 SDP 消息
SDP 消息由 `ServerMediaSession` 生成：
```
float ServerMediaSession::duration() const {
  float minSubsessionDuration = 0.0;
  float maxSubsessionDuration = 0.0;
  for (ServerMediaSubsession* subsession = fSubsessionsHead; subsession != NULL;
       subsession = subsession->fNext) {
    // Hack: If any subsession supports seeking by 'absolute' time, then return a negative value, to indicate that only subsessions
    // will have a "a=range:" attribute:
    char* absStartTime = NULL; char* absEndTime = NULL;
    subsession->getAbsoluteTimeRange(absStartTime, absEndTime);
    if (absStartTime != NULL) return -1.0f;

    float ssduration = subsession->duration();
    if (subsession == fSubsessionsHead) { // this is the first subsession
      minSubsessionDuration = maxSubsessionDuration = ssduration;
    } else if (ssduration < minSubsessionDuration) {
      minSubsessionDuration = ssduration;
    } else if (ssduration > maxSubsessionDuration) {
      maxSubsessionDuration = ssduration;
    }
  }

  if (maxSubsessionDuration != minSubsessionDuration) {
    return -maxSubsessionDuration; // because subsession durations differ
  } else {
    return maxSubsessionDuration; // all subsession durations are the same
  }
}
. . . . . .
char* ServerMediaSession::generateSDPDescription() {
  AddressString ipAddressStr(ourIPAddress(envir()));
  unsigned ipAddressStrSize = strlen(ipAddressStr.val());

  // For a SSM sessions, we need a "a=source-filter: incl ..." line also:
  char* sourceFilterLine;
  if (fIsSSM) {
    char const* const sourceFilterFmt =
      "a=source-filter: incl IN IP4 * %s\r\n"
      "a=rtcp-unicast: reflection\r\n";
    unsigned const sourceFilterFmtSize = strlen(sourceFilterFmt) + ipAddressStrSize + 1;

    sourceFilterLine = new char[sourceFilterFmtSize];
    sprintf(sourceFilterLine, sourceFilterFmt, ipAddressStr.val());
  } else {
    sourceFilterLine = strDup("");
  }

  char* rangeLine = NULL; // for now
  char* sdp = NULL; // for now

  do {
    // Count the lengths of each subsession's media-level SDP lines.
    // (We do this first, because the call to "subsession->sdpLines()"
    // causes correct subsession 'duration()'s to be calculated later.)
    unsigned sdpLength = 0;
    ServerMediaSubsession* subsession;
    for (subsession = fSubsessionsHead; subsession != NULL; subsession = subsession->fNext) {
      char const* sdpLines = subsession->sdpLines();
      if (sdpLines == NULL) continue; // the media's not available
      sdpLength += strlen(sdpLines);
    }
    if (sdpLength == 0) break; // the session has no usable subsessions

    // Unless subsessions have differing durations, we also have a "a=range:" line:
    float dur = duration();
    if (dur == 0.0) {
      rangeLine = strDup("a=range:npt=0-\r\n");
    } else if (dur > 0.0) {
      char buf[100];
      sprintf(buf, "a=range:npt=0-%.3f\r\n", dur);
      rangeLine = strDup(buf);
    } else { // subsessions have differing durations, so "a=range:" lines go there
      rangeLine = strDup("");
    }

    char const* const sdpPrefixFmt =
      "v=0\r\n"
      "o=- %ld%06ld %d IN IP4 %s\r\n"
      "s=%s\r\n"
      "i=%s\r\n"
      "t=0 0\r\n"
      "a=tool:%s%s\r\n"
      "a=type:broadcast\r\n"
      "a=control:*\r\n"
      "%s"
      "%s"
      "a=x-qt-text-nam:%s\r\n"
      "a=x-qt-text-inf:%s\r\n"
      "%s";
    sdpLength += strlen(sdpPrefixFmt)
      + 20 + 6 + 20 + ipAddressStrSize
      + strlen(fDescriptionSDPString)
      + strlen(fInfoSDPString)
      + strlen(libNameStr) + strlen(libVersionStr)
      + strlen(sourceFilterLine)
      + strlen(rangeLine)
      + strlen(fDescriptionSDPString)
      + strlen(fInfoSDPString)
      + strlen(fMiscSDPLines);
    sdpLength += 1000; // in case the length of the "subsession->sdpLines()" calls below change
    sdp = new char[sdpLength];
    if (sdp == NULL) break;

    // Generate the SDP prefix (session-level lines):
    snprintf(sdp, sdpLength, sdpPrefixFmt, fCreationTime.tv_sec,
        fCreationTime.tv_usec, // o= <session id>
        1, // o= <version> // (needs to change if params are modified)
        ipAddressStr.val(), // o= <address>
        fDescriptionSDPString, // s= <description>
        fInfoSDPString, // i= <info>
        libNameStr, libVersionStr, // a=tool:
        sourceFilterLine, // a=source-filter: incl (if a SSM session)
        rangeLine, // a=range: line
        fDescriptionSDPString, // a=x-qt-text-nam: line
        fInfoSDPString, // a=x-qt-text-inf: line
        fMiscSDPLines); // miscellaneous session SDP lines (if any)

    // Then, add the (media-level) lines for each subsession:
    char* mediaSDP = sdp;
    for (subsession = fSubsessionsHead; subsession != NULL; subsession =
        subsession->fNext) {
      unsigned mediaSDPLength = strlen(mediaSDP);
      mediaSDP += mediaSDPLength;
      sdpLength -= mediaSDPLength;
      if (sdpLength <= 1)
        break; // the SDP has somehow become too long

      char const* sdpLines = subsession->sdpLines();
      if (sdpLines != NULL)
        snprintf(mediaSDP, sdpLength, "%s", sdpLines);
    }
  } while (0);

  delete[] rangeLine; delete[] sourceFilterLine;
  return sdp;
}
```
SDP 消息中主要包括两块内容，一块是通用的 SDP 消息内容，这主要包括时间戳，服务器 IP 地址，持续时间等；另一块是流媒体会话中的子会话特有的信息。

对于如下的 SDP 消息：
```
v=0
o=- 1504342443358944 1 IN IP4 10.240.248.20
s=H.264 Video, streamed by the LIVE555 Media Server
i=video/raw_h264_stream.264
t=0 0
a=tool:LIVE555 Streaming Media v2017.07.18
a=type:broadcast
a=control:*
a=range:npt=0-
a=x-qt-text-nam:H.264 Video, streamed by the LIVE555 Media Server
a=x-qt-text-inf:video/raw_h264_stream.264
m=video 0 RTP/AVP 96
c=IN IP4 0.0.0.0
b=AS:500
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42802A;sprop-parameter-sets=Z0KAKtoBEA8eXlIKDAoNoUJq,aM4G4g==
a=control:track1
```
其中通用的 SDP 消息内容为：
```
v=0
o=- 1504342443358944 1 IN IP4 10.240.248.20
s=H.264 Video, streamed by the LIVE555 Media Server
i=video/raw_h264_stream.264
t=0 0
a=tool:LIVE555 Streaming Media v2017.07.18
a=type:broadcast
a=control:*
a=range:npt=0-
a=x-qt-text-nam:H.264 Video, streamed by the LIVE555 Media Server
a=x-qt-text-inf:video/raw_h264_stream.264
```

由 H.264 流媒体文件产生的内容为：
```
m=video 0 RTP/AVP 96
c=IN IP4 0.0.0.0
b=AS:500
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42802A;sprop-parameter-sets=Z0KAKtoBEA8eXlIKDAoNoUJq,aM4G4g==
a=control:track1
```

子会话的 SDP 消息来自于 `ServerMediaSubsession::sdpLines()`，它被声明为纯虚函数：
```
class ServerMediaSubsession: public Medium {
public:
  unsigned trackNumber() const { return fTrackNumber; }
  char const* trackId();
  virtual char const* sdpLines() = 0;
```

对于我们由文件创建的 H.264 流媒体子会话而言，`ServerMediaSubsession` 为 `H264VideoFileServerMediaSubsession`，它有着如下图所示的继承体系：

![](https://www.wolfcstech.com/images/1315506-912929536d7c724f.png)

由图可知，H.264 流媒体子会话所特有的 SDP 内容将来自于 `OnDemandServerMediaSubsession::sdpLines()`：
```
char const*
OnDemandServerMediaSubsession::sdpLines() {
  if (fSDPLines == NULL) {
    // We need to construct a set of SDP lines that describe this
    // subsession (as a unicast stream).  To do so, we first create
    // dummy (unused) source and "RTPSink" objects,
    // whose parameters we use for the SDP lines:
    unsigned estBitrate;
    FramedSource* inputSource = createNewStreamSource(0, estBitrate);
    if (inputSource == NULL) return NULL; // file not found

    struct in_addr dummyAddr;
    dummyAddr.s_addr = 0;
    Groupsock* dummyGroupsock = createGroupsock(dummyAddr, 0);
    unsigned char rtpPayloadType = 96 + trackNumber()-1; // if dynamic
    RTPSink* dummyRTPSink = createNewRTPSink(dummyGroupsock, rtpPayloadType, inputSource);
    if (dummyRTPSink != NULL && dummyRTPSink->estimatedBitrate() > 0) estBitrate = dummyRTPSink->estimatedBitrate();

    setSDPLinesFromRTPSink(dummyRTPSink, inputSource, estBitrate);
    Medium::close(dummyRTPSink);
    delete dummyGroupsock;
    closeStreamSource(inputSource);
  }

  return fSDPLines;
}
```
在这个函数中，为了构造 SDP 行以描述子会话，它首先创建一个临时的 `FramedSource` 和 `RTPSink`，然后从这些结构中获得信息以构造 SDP 行，并在最后销毁临时的 `FramedSource` 和 `RTPSink`。

构造 SDP 行过程如下：
```
float ServerMediaSubsession::duration() const {
  // default implementation: assume an unbounded session:
  return 0.0;
}

void ServerMediaSubsession::getAbsoluteTimeRange(char*& absStartTime, char*& absEndTime) const {
  // default implementation: We don't support seeking by 'absolute' time, so indicate this by setting both parameters to NULL:
  absStartTime = absEndTime = NULL;
}

void ServerMediaSubsession::setServerAddressAndPortForSDP(netAddressBits addressBits,
							  portNumBits portBits) {
  fServerAddressForSDP = addressBits;
  fPortNumForSDP = portBits;
}

void OnDemandServerMediaSubsession
::setSDPLinesFromRTPSink(RTPSink* rtpSink, FramedSource* inputSource, unsigned estBitrate) {
  if (rtpSink == NULL) return;

  char const* mediaType = rtpSink->sdpMediaType();
  unsigned char rtpPayloadType = rtpSink->rtpPayloadType();
  AddressString ipAddressStr(fServerAddressForSDP);
  char* rtpmapLine = rtpSink->rtpmapLine();
  char const* rtcpmuxLine = fMultiplexRTCPWithRTP ? "a=rtcp-mux\r\n" : "";
  char const* rangeLine = rangeSDPLine();
  char const* auxSDPLine = getAuxSDPLine(rtpSink, inputSource);
  if (auxSDPLine == NULL) auxSDPLine = "";

  char const* const sdpFmt =
    "m=%s %u RTP/AVP %d\r\n"
    "c=IN IP4 %s\r\n"
    "b=AS:%u\r\n"
    "%s"
    "%s"
    "%s"
    "%s"
    "a=control:%s\r\n";
  unsigned sdpFmtSize = strlen(sdpFmt)
    + strlen(mediaType) + 5 /* max short len */ + 3 /* max char len */
    + strlen(ipAddressStr.val())
    + 20 /* max int len */
    + strlen(rtpmapLine)
    + strlen(rtcpmuxLine)
    + strlen(rangeLine)
    + strlen(auxSDPLine)
    + strlen(trackId());
  char* sdpLines = new char[sdpFmtSize];
  sprintf(sdpLines, sdpFmt, mediaType, // m= <media>
      fPortNumForSDP, // m= <port>
      rtpPayloadType, // m= <fmt list>
      ipAddressStr.val(), // c= address
      estBitrate, // b=AS:<bandwidth>
      rtpmapLine, // a=rtpmap:... (if present)
      rtcpmuxLine, // a=rtcp-mux:... (if present)
      rangeLine, // a=range:... (if present)
      auxSDPLine, // optional extra SDP line
      trackId()); // a=control:<track-id>
  delete[] (char*) rangeLine;
  delete[] rtpmapLine;

  fSDPLines = strDup(sdpLines);
  delete[] sdpLines;
}
```

创建临时的 `FramedSource` 和 `RTPSink` 的函数都是纯虚函数：
```
  virtual FramedSource* createNewStreamSource(unsigned clientSessionId,
      unsigned& estBitrate) = 0;
  // "estBitrate" is the stream's estimated bitrate, in kbps
  virtual RTPSink* createNewRTPSink(Groupsock* rtpGroupsock,
      unsigned char rtpPayloadTypeIfDynamic,
      FramedSource* inputSource) = 0;
```

对于 `H264VideoFileServerMediaSubsession` 的类继承层次结构，这两个函数的实现都在 `H264VideoFileServerMediaSubsession` 类中：
```
FramedSource* H264VideoFileServerMediaSubsession::createNewStreamSource(unsigned /*clientSessionId*/, unsigned& estBitrate) {
  estBitrate = 500; // kbps, estimate

  // Create the video source:
  ByteStreamFileSource* fileSource = ByteStreamFileSource::createNew(envir(), fFileName);
  if (fileSource == NULL) return NULL;
  fFileSize = fileSource->fileSize();

  // Create a framer for the Video Elementary Stream:
  return H264VideoStreamFramer::createNew(envir(), fileSource);
}

RTPSink* H264VideoFileServerMediaSubsession::createNewRTPSink(
    Groupsock* rtpGroupsock,
    unsigned char rtpPayloadTypeIfDynamic,
    FramedSource* /*inputSource*/) {
  return H264VideoRTPSink::createNew(envir(), rtpGroupsock, rtpPayloadTypeIfDynamic);
}
```
创建的实际 `FramedSource` 和 `RTPSink` 类型分别为 `H264VideoStreamFramer` 和 `H264VideoRTPSink`。

### [打赏](https://www.wolfcstech.com/about/donate.html)

# live555 源码分析系列文章
[live555 源码分析：简介](https://www.wolfcstech.com/2017/08/28/live555_src_analysis_introduction/)
[live555 源码分析：基础设施](https://www.wolfcstech.com/2017/08/30/live555_src_analysis_infrasture/)
[live555 源码分析：MediaSever](https://www.wolfcstech.com/2017/08/31/live555_src_analysis_mediaserver/)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](https://www.wolfcstech.com/2017/09/01/live555_src_analysis_rtsp_rtp_rtcp_wireshark/)
[live555 源码分析：RTSPServer](https://www.wolfcstech.com/2017/09/03/live555_src_analysis_rtspserver/)
[live555 源码分析： DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)
