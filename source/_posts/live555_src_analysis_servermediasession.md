---
title: live555 源码分析：ServerMediaSession
date: 2017-09-07 12:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

在 live555 中，用一个 `ServerMediaSession` 表示流媒体会话，它连接了 `RTSPServer` 和下层流媒体传输逻辑。`ServerMediaSession` 和 `ServerMediaSubsession` 共同用于执行底层流媒体传输和状态维护。而 `ServerMediaSession` 则是在 `GenericMediaServer` 中，通过 `HashTable` 来维护的。
<!--more-->
在分析 live555 中处理 `DESCRIBE` 请求的代码（ [live555 源码分析：DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)）时，我们曾看到 `RTSPServer::RTSPClientConnection` 通过它来产生 SDP 消息。本文更详细地分析这个类的定义和实现。

# ServerMediaSession
首先看一下 `ServerMediaSession` 的定义：
```
class ServerMediaSubsession; // forward

class ServerMediaSession: public Medium {
public:
  static ServerMediaSession* createNew(UsageEnvironment& env,
      char const* streamName = NULL,
      char const* info = NULL,
      char const* description = NULL,
      Boolean isSSM = False,
      char const* miscSDPLines = NULL);

  static Boolean lookupByName(UsageEnvironment& env,
      char const* mediumName,
      ServerMediaSession*& resultSession);

  char* generateSDPDescription(); // based on the entire session
  // Note: The caller is responsible for freeing the returned string

  char const* streamName() const { return fStreamName; }

  Boolean addSubsession(ServerMediaSubsession* subsession);
  unsigned numSubsessions() const { return fSubsessionCounter; }

  void testScaleFactor(float& scale); // sets "scale" to the actual supported scale
  float duration() const;
    // a result == 0 means an unbounded session (the default)
    // a result < 0 means: subsession durations differ; the result is -(the largest).
    // a result > 0 means: this is the duration of a bounded session

  virtual void noteLiveness();
    // called whenever a client - accessing this media - notes liveness.
    // The default implementation does nothing, but subclasses can redefine this - e.g., if you
    // want to remove long-unused "ServerMediaSession"s from the server.

  unsigned referenceCount() const { return fReferenceCount; }
  void incrementReferenceCount() { ++fReferenceCount; }
  void decrementReferenceCount() { if (fReferenceCount > 0) --fReferenceCount; }
  Boolean& deleteWhenUnreferenced() { return fDeleteWhenUnreferenced; }

  void deleteAllSubsessions();
    // Removes and deletes all subsessions added by "addSubsession()", returning us to an 'empty' state
    // Note: If you have already added this "ServerMediaSession" to a "RTSPServer" then, before calling this function,
    //   you must first close any client connections that use it,
    //   by calling "RTSPServer::closeAllClientSessionsForServerMediaSession()".

protected:
  ServerMediaSession(UsageEnvironment& env, char const* streamName,
      char const* info, char const* description,
      Boolean isSSM, char const* miscSDPLines);
  // called only by "createNew()"

  virtual ~ServerMediaSession();

private: // redefined virtual functions
  virtual Boolean isServerMediaSession() const;

private:
  Boolean fIsSSM;

  // Linkage fields:
  friend class ServerMediaSubsessionIterator;
  ServerMediaSubsession* fSubsessionsHead;
  ServerMediaSubsession* fSubsessionsTail;
  unsigned fSubsessionCounter;

  char* fStreamName;
  char* fInfoSDPString;
  char* fDescriptionSDPString;
  char* fMiscSDPLines;
  struct timeval fCreationTime;
  unsigned fReferenceCount;
  Boolean fDeleteWhenUnreferenced;
};
```

由这个定义，不难理解，它主要是 `ServerMediaSubsession` 的容器，并通过一个单向链表来维护它们。并提供了需要作用于整个流媒体会话所有子会话的操作，如产生 SDP 消息的 `generateSDPDescription()` 和设置播放快慢的 `testScaleFactor()`。

`ServerMediaSession` 对象的创建，就像 live555 中许多类的创建那样，通过一个静态的创建函数 `createNew()` 实现，该函数定义如下：
```
ServerMediaSession* ServerMediaSession
::createNew(UsageEnvironment& env,
    char const* streamName, char const* info,
    char const* description, Boolean isSSM, char const* miscSDPLines) {
  return new ServerMediaSession(env, streamName, info, description,
      isSSM, miscSDPLines);
}
. . . . . .
static char const* const libNameStr = "LIVE555 Streaming Media v";
char const* const libVersionStr = LIVEMEDIA_LIBRARY_VERSION_STRING;

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

ServerMediaSession::~ServerMediaSession() {
  deleteAllSubsessions();
  delete[] fStreamName;
  delete[] fInfoSDPString;
  delete[] fDescriptionSDPString;
  delete[] fMiscSDPLines;
}
```

每个 `ServerMediaSession` 都由一个字符串形式的 `streamName` 标识，这个标识也是在 `GenericMediaServer` 中，通过 `HashTable` 来维护时，所用的 key。`streamName` 由调用者在创建时传入。对于 “LIVE555 Media Server” 而言，这个值为资源的路径。

调用者还可以在创建时传入 `info` 和 `description` 提供更多关于这个会话的描述信息。对于 “LIVE555 Media Server” 而言，`info` 同样为资源的路径，但 `description` 为含有流媒体类型的一个字符串，格式为 `"$MediaType, streamed by the LIVE555 Media Server"`，比如对于 H.264 视频为 `"H.264 Video, streamed by the LIVE555 Media Server"`。

创建对象时，主要是用调用者传入的值来初始化状态。

像许多其它 `Medium` 的子类一样，`ServerMediaSession` 也提供了一个对象查找函数：
```
Boolean ServerMediaSession
::lookupByName(UsageEnvironment& env, char const* mediumName,
    ServerMediaSession*& resultSession) {
  resultSession = NULL; // unless we succeed

  Medium* medium;
  if (!Medium::lookupByName(env, mediumName, medium)) return False;

  if (!medium->isServerMediaSession()) {
    env.setResultMsg(mediumName, " is not a 'ServerMediaSession' object");
    return False;
  }

  resultSession = (ServerMediaSession*)medium;
  return True;
}
```

作为 `ServerMediaSubsession` 的容器，`ServerMediaSession` 还提供了对 `ServerMediaSubsession` 的添加、管理等操作：
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
. . . . . .
void ServerMediaSession::noteLiveness() {
  // default implementation: do nothing
}

void ServerMediaSession::deleteAllSubsessions() {
  Medium::close(fSubsessionsHead);
  fSubsessionsHead = fSubsessionsTail = NULL;
  fSubsessionCounter = 0;
}

Boolean ServerMediaSession::isServerMediaSession() const {
  return True;
}
. . . . . .
```
向 `ServerMediaSession` 中添加子会话时，新加入的子会话总是会被放在单向链表的表头。添加子会话的时候，会根据子会话计数器 `fSubsessionCounter` 为子会话分配 track number。

用来产生 SDP 消息的 `generateSDPDescription()` 和 `duration()` 函数，及用来设置播放快慢的 `testScaleFactor(float& scale)` 函数，在 [live555 源码分析：DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/) 和 [live555 源码分析：PLAY 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_play/) 中已经有较为详细地说明了，这里不再赘述。

为了便于访问 `ServerMediaSubsession`，live555 还提供了一个迭代器，该迭代器定义如下：
```
class ServerMediaSubsessionIterator {
public:
  ServerMediaSubsessionIterator(ServerMediaSession& session);
  virtual ~ServerMediaSubsessionIterator();

  ServerMediaSubsession* next(); // NULL if none
  void reset();

private:
  ServerMediaSession& fOurSession;
  ServerMediaSubsession* fNextPtr;
};
```

该迭代器实现如下：
```
ServerMediaSubsessionIterator
::ServerMediaSubsessionIterator(ServerMediaSession& session)
  : fOurSession(session) {
  reset();
}

ServerMediaSubsessionIterator::~ServerMediaSubsessionIterator() {
}

ServerMediaSubsession* ServerMediaSubsessionIterator::next() {
  ServerMediaSubsession* result = fNextPtr;

  if (fNextPtr != NULL) fNextPtr = fNextPtr->fNext;

  return result;
}

void ServerMediaSubsessionIterator::reset() {
  fNextPtr = fOurSession.fSubsessionsHead;
}
```

可以看到 `ServerMediaSession` 中并没有太多的内容。大多数时候，它仅作为获得 `ServerMediaSubsession` 的中介而存在。

# ServerMediaSubsession

在 `RTSPServer::RTSPClientSession` 中处理 `SETUP` 请求的代码中，我们看到，它从  `ServerMediaSession` 获得 `ServerMediaSubsession` 并接管了对它们的管理，并绕过 `ServerMediaSession` 直接对 `ServerMediaSubsession` 进行操作。执行流媒体单个会话的数据操作的入口也都在 `ServerMediaSubsession`。

对于向 “LIVE555 Media Server” 服务器请求  H.264 视频文件的情况，`ServerMediaSubsession` 的实际类型为 `H264VideoFileServerMediaSubsession`，我们以此为例来分析 `ServerMediaSubsession`。

`H264VideoFileServerMediaSubsession` 在 `DynamicRTSPServer` 类的 `lookupServerMediaSession()` 中随 `ServerMediaSession` 的创建一起创建，并直接被添加到 `ServerMediaSession` 中。
```
#define NEW_SMS(description) do {\
char const* descStr = description\
    ", streamed by the LIVE555 Media Server";\
sms = ServerMediaSession::createNew(env, fileName, fileName, descStr);\
} while(0)
. . . . . .
  } else if (strcmp(extension, ".264") == 0) {
    // Assumed to be a H.264 Video Elementary Stream file:
    NEW_SMS("H.264 Video");
    OutPacketBuffer::maxSize = 100000; // allow for some possibly large H.264 frames
    sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  }
```

`H264VideoFileServerMediaSubsession` 有着如下图所示的继承体系：

![](https://www.wolfcstech.com/images/1315506-1e2dfc17512a3909.png)

在这个继承体系中，`ServerMediaSubsession` 定义了可以对单个流执行的操作，类的定义如下：
```
class ServerMediaSubsession: public Medium {
public:
  unsigned trackNumber() const { return fTrackNumber; }
  char const* trackId();
  virtual char const* sdpLines() = 0;
  virtual void getStreamParameters(unsigned clientSessionId, // in
      netAddressBits clientAddress, // in
      Port const& clientRTPPort, // in
      Port const& clientRTCPPort, // in
      int tcpSocketNum, // in (-1 means use UDP, not TCP)
      unsigned char rtpChannelId, // in (used if TCP)
      unsigned char rtcpChannelId, // in (used if TCP)
      netAddressBits& destinationAddress, // in out
      u_int8_t& destinationTTL, // in out
      Boolean& isMulticast, // out
      Port& serverRTPPort, // out
      Port& serverRTCPPort, // out
      void*& streamToken // out
      ) = 0;
  virtual void startStream(unsigned clientSessionId, void* streamToken,
      TaskFunc* rtcpRRHandler,
      void* rtcpRRHandlerClientData,
      unsigned short& rtpSeqNum,
      unsigned& rtpTimestamp,
      ServerRequestAlternativeByteHandler* serverRequestAlternativeByteHandler,
      void* serverRequestAlternativeByteHandlerClientData) = 0;
  virtual void pauseStream(unsigned clientSessionId, void* streamToken);
  virtual void seekStream(unsigned clientSessionId, void* streamToken, double& seekNPT,
      double streamDuration, u_int64_t& numBytes);
     // This routine is used to seek by relative (i.e., NPT) time.
     // "streamDuration", if >0.0, specifies how much data to stream, past "seekNPT".  (If <=0.0, all remaining data is streamed.)
     // "numBytes" returns the size (in bytes) of the data to be streamed, or 0 if unknown or unlimited.
  virtual void seekStream(unsigned clientSessionId, void* streamToken, char*& absStart, char*& absEnd);
     // This routine is used to seek by 'absolute' time.
     // "absStart" should be a string of the form "YYYYMMDDTHHMMSSZ" or "YYYYMMDDTHHMMSS.<frac>Z".
     // "absEnd" should be either NULL (for no end time), or a string of the same form as "absStart".
     // These strings may be modified in-place, or can be reassigned to a newly-allocated value (after delete[]ing the original).
  virtual void nullSeekStream(unsigned clientSessionId, void* streamToken,
      double streamEndTime, u_int64_t& numBytes);
     // Called whenever we're handling a "PLAY" command without a specified start time.
  virtual void setStreamScale(unsigned clientSessionId, void* streamToken, float scale);
  virtual float getCurrentNPT(void* streamToken);
  virtual FramedSource* getStreamSource(void* streamToken);
  virtual void getRTPSinkandRTCP(void* streamToken,
      RTPSink const*& rtpSink, RTCPInstance const*& rtcp) = 0;
     // Returns pointers to the "RTPSink" and "RTCPInstance" objects for "streamToken".
     // (This can be useful if you want to get the associated 'Groupsock' objects, for example.)
     // You must not delete these objects, or start/stop playing them; instead, that is done
     // using the "startStream()" and "deleteStream()" functions.
  virtual void deleteStream(unsigned clientSessionId, void*& streamToken);

  virtual void testScaleFactor(float& scale); // sets "scale" to the actual supported scale
  virtual float duration() const;
    // returns 0 for an unbounded session (the default)
    // returns > 0 for a bounded session
  virtual void getAbsoluteTimeRange(char*& absStartTime, char*& absEndTime) const;
    // Subclasses can reimplement this iff they support seeking by 'absolute' time.

  // The following may be called by (e.g.) SIP servers, for which the
  // address and port number fields in SDP descriptions need to be non-zero:
  void setServerAddressAndPortForSDP(netAddressBits addressBits,
      portNumBits portBits);

protected: // we're a virtual base class
  ServerMediaSubsession(UsageEnvironment& env);
  virtual ~ServerMediaSubsession();

  char const* rangeSDPLine() const;
      // returns a string to be delete[]d

  ServerMediaSession* fParentSession;
  netAddressBits fServerAddressForSDP;
  portNumBits fPortNumForSDP;

private:
  friend class ServerMediaSession;
  friend class ServerMediaSubsessionIterator;
  ServerMediaSubsession* fNext;

  unsigned fTrackNumber; // within an enclosing ServerMediaSession
  char const* fTrackId;
};
```
这些操作可以分为几类，一类是对播放进行控制的操作，包括 `startStream()`、`pauseStream()`、`seekStream()`、`nullSeekStream()`、`setStreamScale()`、`deleteStream()` 和 `testScaleFactor()` 等；另一类是获得用于执行 I/O 操作的 `FramedSource` 和 `RTPSink` 的 `getStreamSource()` 和 `getRTPSinkandRTCP()`。

`ServerMediaSubsession` 类提供了几个子会话的通用操作的实现，这主要包括用于产生字符串形式的 track id 的 `trackId()`，用于设置 SDP 服务器地址和端口号的 `setServerAddressAndPortForSDP()`，以及用于生成子会话的 SDP 行的 `rangeSDPLine()`：
```
ServerMediaSubsession::ServerMediaSubsession(UsageEnvironment& env)
  : Medium(env),
    fParentSession(NULL), fServerAddressForSDP(0), fPortNumForSDP(0),
    fNext(NULL), fTrackNumber(0), fTrackId(NULL) {
}

ServerMediaSubsession::~ServerMediaSubsession() {
  delete[] (char*)fTrackId;
  Medium::close(fNext);
}

char const* ServerMediaSubsession::trackId() {
  if (fTrackNumber == 0) return NULL; // not yet in a ServerMediaSession

  if (fTrackId == NULL) {
    char buf[100];
    sprintf(buf, "track%d", fTrackNumber);
    fTrackId = strDup(buf);
  }
  return fTrackId;
}
. . . . . .
void ServerMediaSubsession::setServerAddressAndPortForSDP(netAddressBits addressBits,
    portNumBits portBits) {
  fServerAddressForSDP = addressBits;
  fPortNumForSDP = portBits;
}

char const*
ServerMediaSubsession::rangeSDPLine() const {
  // First, check for the special case where we support seeking by 'absolute' time:
  char* absStart = NULL; char* absEnd = NULL;
  getAbsoluteTimeRange(absStart, absEnd);
  if (absStart != NULL) {
    char buf[100];

    if (absEnd != NULL) {
      sprintf(buf, "a=range:clock=%s-%s\r\n", absStart, absEnd);
    } else {
      sprintf(buf, "a=range:clock=%s-\r\n", absStart);
    }
    return strDup(buf);
  }

  if (fParentSession == NULL) return NULL;

  // If all of our parent's subsessions have the same duration
  // (as indicated by "fParentSession->duration() >= 0"), there's no "a=range:" line:
  if (fParentSession->duration() >= 0.0) return strDup("");

  // Use our own duration for a "a=range:" line:
  float ourDuration = duration();
  if (ourDuration == 0.0) {
    return strDup("a=range:npt=0-\r\n");
  } else {
    char buf[100];
    sprintf(buf, "a=range:npt=0-%.3f\r\n", ourDuration);
    return strDup(buf);
  }
}
```
这些操作都比较简明，这里不再赘述。

对于其它众多操作单个流的接口，`ServerMediaSubsession` 类都只是提供了一个空的默认实现：
```
void ServerMediaSubsession::pauseStream(unsigned /*clientSessionId*/,
    void* /*streamToken*/) {
  // default implementation: do nothing
}
void ServerMediaSubsession::seekStream(unsigned /*clientSessionId*/,
    void* /*streamToken*/, double& /*seekNPT*/, double /*streamDuration*/, u_int64_t& numBytes) {
  // default implementation: do nothing
  numBytes = 0;
}
void ServerMediaSubsession::seekStream(unsigned /*clientSessionId*/,
    void* /*streamToken*/, char*& absStart, char*& absEnd) {
  // default implementation: do nothing (but delete[] and assign "absStart" and "absEnd" to NULL, to show that we don't handle this)
  delete[] absStart; absStart = NULL;
  delete[] absEnd; absEnd = NULL;
}
void ServerMediaSubsession::nullSeekStream(unsigned /*clientSessionId*/,
    void* /*streamToken*/, double streamEndTime, u_int64_t& numBytes) {
  // default implementation: do nothing
  numBytes = 0;
}
void ServerMediaSubsession::setStreamScale(unsigned /*clientSessionId*/,
    void* /*streamToken*/, float /*scale*/) {
  // default implementation: do nothing
}
float ServerMediaSubsession::getCurrentNPT(void* /*streamToken*/) {
  // default implementation: return 0.0
  return 0.0;
}
FramedSource* ServerMediaSubsession::getStreamSource(void* /*streamToken*/) {
  // default implementation: return NULL
  return NULL;
}
void ServerMediaSubsession::deleteStream(unsigned /*clientSessionId*/,
    void*& /*streamToken*/) {
  // default implementation: do nothing
}

void ServerMediaSubsession::testScaleFactor(float& scale) {
  // default implementation: Support scale = 1 only
  scale = 1;
}

float ServerMediaSubsession::duration() const {
  // default implementation: assume an unbounded session:
  return 0.0;
}

void ServerMediaSubsession::getAbsoluteTimeRange(char*& absStartTime, char*& absEndTime) const {
  // default implementation: We don't support seeking by 'absolute' time, so indicate this by setting both parameters to NULL:
  absStartTime = absEndTime = NULL;
}
```

`OnDemandServerMediaSubsession` 实现由 `ServerMediaSubsession` 定义的流操作接口。为了实现这些操作，需要一些 I/O 操作，如解析流媒体文件，收发 RTP/RTCP 包等。这些 I/O 操作将由于具体的流媒体源类型的不同而不同，因而不会直接在 `OnDemandServerMediaSubsession` 中实现。`OnDemandServerMediaSubsession` 定义了新的虚函数，以便从子类中获得 `FramedSource` 和 `RTPSink` 对象，来执行 I/O 操作。

`OnDemandServerMediaSubsession` 的具体实现，暂时先不详细说明。

在 `H264VideoFileServerMediaSubsession` 的类继承层次结构中， `FileServerMediaSubsession` 用于维护资源的文件名，其定义如下：
```
class FileServerMediaSubsession: public OnDemandServerMediaSubsession {
protected: // we're a virtual base class
  FileServerMediaSubsession(UsageEnvironment& env, char const* fileName,
			    Boolean reuseFirstSource);
  virtual ~FileServerMediaSubsession();

protected:
  char const* fFileName;
  u_int64_t fFileSize; // if known
};
```

这个定义非常简单，其实现也很简单：
```
FileServerMediaSubsession
::FileServerMediaSubsession(UsageEnvironment& env, char const* fileName,
			    Boolean reuseFirstSource)
  : OnDemandServerMediaSubsession(env, reuseFirstSource),
    fFileSize(0) {
  fFileName = strDup(fileName);
}

FileServerMediaSubsession::~FileServerMediaSubsession() {
  delete[] (char*)fFileName;
}
```

`H264VideoFileServerMediaSubsession` 最主要的功能则是提供用于执行 I/O 操作的 `FramedSource` 和 `RTPSink`，其定义如下：
```
class H264VideoFileServerMediaSubsession: public FileServerMediaSubsession {
public:
  static H264VideoFileServerMediaSubsession*
  createNew(UsageEnvironment& env, char const* fileName, Boolean reuseFirstSource);

  // Used to implement "getAuxSDPLine()":
  void checkForAuxSDPLine1();
  void afterPlayingDummy1();

protected:
  H264VideoFileServerMediaSubsession(UsageEnvironment& env,
      char const* fileName, Boolean reuseFirstSource);
  // called only by createNew();
  virtual ~H264VideoFileServerMediaSubsession();

  void setDoneFlag() { fDoneFlag = ~0; }

protected: // redefined virtual functions
  virtual char const* getAuxSDPLine(RTPSink* rtpSink,
      FramedSource* inputSource);
  virtual FramedSource* createNewStreamSource(unsigned clientSessionId,
      unsigned& estBitrate);
  virtual RTPSink* createNewRTPSink(Groupsock* rtpGroupsock,
      unsigned char rtpPayloadTypeIfDynamic,
      FramedSource* inputSource);

private:
  char* fAuxSDPLine;
  char fDoneFlag; // used when setting up "fAuxSDPLine"
  RTPSink* fDummyRTPSink; // ditto
};
```

`H264VideoFileServerMediaSubsession` 通过实现 `createNewStreamSource()` 和 `createNewRTPSink()` 创建并返回 `FramedSource` 和 `RTPSink`：
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

RTPSink* H264VideoFileServerMediaSubsession
::createNewRTPSink(Groupsock* rtpGroupsock,
    unsigned char rtpPayloadTypeIfDynamic,
    FramedSource* /*inputSource*/) {
  return H264VideoRTPSink::createNew(envir(), rtpGroupsock, rtpPayloadTypeIfDynamic);
}
```

这里返回的 `FramedSource` 为 ***`H264VideoStreamFramer`***，返回的 `RTPSink` 为 ***`H264VideoRTPSink`***。

除此之外，`H264VideoFileServerMediaSubsession` 还提供了 AUX SDP line 有关的接口实现：
```
H264VideoFileServerMediaSubsession::H264VideoFileServerMediaSubsession(UsageEnvironment& env,
    char const* fileName, Boolean reuseFirstSource) :
    FileServerMediaSubsession(env, fileName, reuseFirstSource),
    fAuxSDPLine(NULL), fDoneFlag(0), fDummyRTPSink(NULL) {
}

H264VideoFileServerMediaSubsession::~H264VideoFileServerMediaSubsession() {
  delete[] fAuxSDPLine;
}

static void afterPlayingDummy(void* clientData) {
  H264VideoFileServerMediaSubsession* subsess = (H264VideoFileServerMediaSubsession*)clientData;
  subsess->afterPlayingDummy1();
}

void H264VideoFileServerMediaSubsession::afterPlayingDummy1() {
  // Unschedule any pending 'checking' task:
  envir().taskScheduler().unscheduleDelayedTask(nextTask());
  // Signal the event loop that we're done:
  setDoneFlag();
}

static void checkForAuxSDPLine(void* clientData) {
  H264VideoFileServerMediaSubsession* subsess = (H264VideoFileServerMediaSubsession*)clientData;
  subsess->checkForAuxSDPLine1();
}

void H264VideoFileServerMediaSubsession::checkForAuxSDPLine1() {
  nextTask() = NULL;

  char const* dasl;
  if (fAuxSDPLine != NULL) {
    // Signal the event loop that we're done:
    setDoneFlag();
  } else if (fDummyRTPSink != NULL && (dasl = fDummyRTPSink->auxSDPLine()) != NULL) {
    fAuxSDPLine = strDup(dasl);
    fDummyRTPSink = NULL;

    // Signal the event loop that we're done:
    setDoneFlag();
  } else if (!fDoneFlag) {
    // try again after a brief delay:
    int uSecsToDelay = 100000; // 100 ms
    nextTask() = envir().taskScheduler().scheduleDelayedTask(uSecsToDelay,
        (TaskFunc*) checkForAuxSDPLine, this);
  }
}

char const* H264VideoFileServerMediaSubsession::getAuxSDPLine(RTPSink* rtpSink, FramedSource* inputSource) {
  if (fAuxSDPLine != NULL) return fAuxSDPLine; // it's already been set up (for a previous client)

  if (fDummyRTPSink == NULL) { // we're not already setting it up for another, concurrent stream
    // Note: For H264 video files, the 'config' information ("profile-level-id" and "sprop-parameter-sets") isn't known
    // until we start reading the file.  This means that "rtpSink"s "auxSDPLine()" will be NULL initially,
    // and we need to start reading data from our file until this changes.
    fDummyRTPSink = rtpSink;

    // Start reading the file:
    fDummyRTPSink->startPlaying(*inputSource, afterPlayingDummy, this);

    // Check whether the sink's 'auxSDPLine()' is ready:
    checkForAuxSDPLine(this);
  }

  envir().taskScheduler().doEventLoop(&fDoneFlag);

  return fAuxSDPLine;
}
```

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
