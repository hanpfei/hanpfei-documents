---
title: live555 源码分析：RTSPServer 组件结构
date: 2017-09-06 14:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

前面几篇文章分析了 live555 中 RTSP 的处理逻辑，RTSP 处理有关组件的处理逻辑有点复杂，本文就再来梳理一下它们之间的关系。
<!--more-->
live555 中 RTSP 处理有关组件关系如下图：

![](https://www.wolfcstech.com/images/1315506-5ae0f80b706b414b.png)

事件和执行流程的源头在 `TaskScheduler`。`GenericMediaServer` 对象在创建的时候，会向 `TaskScheduler` 注册一个 server socket 及处理该 socket 上的事件的处理程序 `GenericMediaServer::incomingConnectionHandler(void* instance, int /*mask*/)`。

当有客户端连接服务器时，触发 server socket 上的事件处理器执行。此时会基于客户端 socket 创建 `ClientConnection` 对象，及 `RTSPClientConnection`。 `RTSPClientConnection` 对象在创建过程中，会将该客户端 socket 及
 `ClientConnection` 中处理该 socket 上的事件的处理程序 `GenericMediaServer::ClientConnection::incomingRequestHandler(void* instance, int /*mask*/)` 注册给 `TaskScheduler`。

在客户端发送的 RTSP 请求数据到达之后，`GenericMediaServer::ClientConnection` 会读取这些数据，并交给 `RTSPServer::RTSPClientConnection::handleRequestBytes(int newBytesRead)` 处理。

`RTSPServer::RTSPClientConnection` 解析 RTSP 请求，并处理 `OPTIONS`、`DESCRIBE` 和 ``SETUP 等无需流媒体会话建立即可处理的请求。

`RTSPServer::RTSPClientConnection` 在处理 `SETUP` 请求时，会创建流媒体会话的 `RTSPServer::RTSPClientSession`，具体的会话建立过程都会被委托给后者处理。

需要会话建立之后才能处理的请求，也会被交给 `RTSPServer::RTSPClientSession` 处理。

这里来看一下 `RTSPServer::RTSPClientConnection` 的完整定义：
```
class RTSPServer: public GenericMediaServer {
. . . . . .
public: // should be protected, but some old compilers complain otherwise
  // The state of a TCP connection used by a RTSP client:
  class RTSPClientSession; // forward
  class RTSPClientConnection: public GenericMediaServer::ClientConnection {
  public:
    // A data structure that's used to implement the "REGISTER" command:
    class ParamsForREGISTER {
    public:
      ParamsForREGISTER(char const* cmd/*"REGISTER" or "DEREGISTER"*/,
			RTSPClientConnection* ourConnection, char const* url, char const* urlSuffix,
			Boolean reuseConnection, Boolean deliverViaTCP, char const* proxyURLSuffix);
      virtual ~ParamsForREGISTER();
    private:
      friend class RTSPClientConnection;
      char const* fCmd;
      RTSPClientConnection* fOurConnection;
      char* fURL;
      char* fURLSuffix;
      Boolean fReuseConnection, fDeliverViaTCP;
      char* fProxyURLSuffix;
    };
  protected: // redefined virtual functions:
    virtual void handleRequestBytes(int newBytesRead);

  protected:
    RTSPClientConnection(RTSPServer& ourServer, int clientSocket, struct sockaddr_in clientAddr);
    virtual ~RTSPClientConnection();

    friend class RTSPServer;
    friend class RTSPClientSession;

    // Make the handler functions for each command virtual, to allow subclasses to reimplement them, if necessary:
    virtual void handleCmd_OPTIONS();
        // You probably won't need to subclass/reimplement this function; reimplement "RTSPServer::allowedCommandNames()" instead.
    virtual void handleCmd_GET_PARAMETER(char const* fullRequestStr); // when operating on the entire server
    virtual void handleCmd_SET_PARAMETER(char const* fullRequestStr); // when operating on the entire server
    virtual void handleCmd_DESCRIBE(char const* urlPreSuffix, char const* urlSuffix, char const* fullRequestStr);
    virtual void handleCmd_REGISTER(char const* cmd/*"REGISTER" or "DEREGISTER"*/,
				    char const* url, char const* urlSuffix, char const* fullRequestStr,
				    Boolean reuseConnection, Boolean deliverViaTCP, char const* proxyURLSuffix);
        // You probably won't need to subclass/reimplement this function;
        //     reimplement "RTSPServer::weImplementREGISTER()" and "RTSPServer::implementCmd_REGISTER()" instead.
    virtual void handleCmd_bad();
    virtual void handleCmd_notSupported();
    virtual void handleCmd_notFound();
    virtual void handleCmd_sessionNotFound();
    virtual void handleCmd_unsupportedTransport();
    // Support for optional RTSP-over-HTTP tunneling:
    virtual Boolean parseHTTPRequestString(char* resultCmdName, unsigned resultCmdNameMaxSize,
					   char* urlSuffix, unsigned urlSuffixMaxSize,
					   char* sessionCookie, unsigned sessionCookieMaxSize,
					   char* acceptStr, unsigned acceptStrMaxSize);
    virtual void handleHTTPCmd_notSupported();
    virtual void handleHTTPCmd_notFound();
    virtual void handleHTTPCmd_OPTIONS();
    virtual void handleHTTPCmd_TunnelingGET(char const* sessionCookie);
    virtual Boolean handleHTTPCmd_TunnelingPOST(char const* sessionCookie, unsigned char const* extraData, unsigned extraDataSize);
    virtual void handleHTTPCmd_StreamingGET(char const* urlSuffix, char const* fullRequestStr);
  protected:
    void resetRequestBuffer();
    void closeSocketsRTSP();
    static void handleAlternativeRequestByte(void*, u_int8_t requestByte);
    void handleAlternativeRequestByte1(u_int8_t requestByte);
    Boolean authenticationOK(char const* cmdName, char const* urlSuffix, char const* fullRequestStr);
    void changeClientInputSocket(int newSocketNum, unsigned char const* extraData, unsigned extraDataSize);
      // used to implement RTSP-over-HTTP tunneling
    static void continueHandlingREGISTER(ParamsForREGISTER* params);
    virtual void continueHandlingREGISTER1(ParamsForREGISTER* params);

    // Shortcuts for setting up a RTSP response (prior to sending it):
    void setRTSPResponse(char const* responseStr);
    void setRTSPResponse(char const* responseStr, u_int32_t sessionId);
    void setRTSPResponse(char const* responseStr, char const* contentStr);
    void setRTSPResponse(char const* responseStr, u_int32_t sessionId, char const* contentStr);

    RTSPServer& fOurRTSPServer; // same as ::fOurServer
    int& fClientInputSocket; // aliased to ::fOurSocket
    int fClientOutputSocket;
    Boolean fIsActive;
    unsigned char* fLastCRLF;
    unsigned fRecursionCount;
    char const* fCurrentCSeq;
    Authenticator fCurrentAuthenticator; // used if access control is needed
    char* fOurSessionCookie; // used for optional RTSP-over-HTTP tunneling
    unsigned fBase64RemainderCount; // used for optional RTSP-over-HTTP tunneling (possible values: 0,1,2,3)
  };
```

`RTSPServer::RTSPClientConnection` 继承自 `GenericMediaServer::ClientConnection`：
```
class GenericMediaServer: public Medium {
. . . . . .
public: // should be protected, but some old compilers complain otherwise
  // The state of a TCP connection used by a client:
  class ClientConnection {
  protected:
    ClientConnection(GenericMediaServer& ourServer, int clientSocket, struct sockaddr_in clientAddr);
    virtual ~ClientConnection();

    UsageEnvironment& envir() { return fOurServer.envir(); }
    void closeSockets();

    static void incomingRequestHandler(void*, int /*mask*/);
    void incomingRequestHandler();
    virtual void handleRequestBytes(int newBytesRead) = 0;
    void resetRequestBuffer();

  protected:
    friend class GenericMediaServer;
    friend class ClientSession;
    friend class RTSPServer; // needed to make some broken Windows compilers work; remove this in the future when we end support for Windows
    GenericMediaServer& fOurServer;
    int fOurSocket;
    struct sockaddr_in fClientAddr;
    unsigned char fRequestBuffer[REQUEST_BUFFER_SIZE];
    unsigned char fResponseBuffer[RESPONSE_BUFFER_SIZE];
    unsigned fRequestBytesAlreadySeen, fRequestBufferBytesLeft;
  };
```

从它们的定义，不难理解它们的职责主要在于处理网络 I/O，处理 RTSP 请求，并建立会话。

再来看 `RTSPServer::RTSPClientSession` 的定义：
```
class RTSPServer: public GenericMediaServer {
. . . . . .
  // The state of an individual client session (using one or more sequential TCP connections) handled by a RTSP server:
  class RTSPClientSession: public GenericMediaServer::ClientSession {
  protected:
    RTSPClientSession(RTSPServer& ourServer, u_int32_t sessionId);
    virtual ~RTSPClientSession();

    friend class RTSPServer;
    friend class RTSPClientConnection;
    // Make the handler functions for each command virtual, to allow subclasses to redefine them:
    virtual void handleCmd_SETUP(RTSPClientConnection* ourClientConnection,
        char const* urlPreSuffix, char const* urlSuffix, char const* fullRequestStr);
    virtual void handleCmd_withinSession(RTSPClientConnection* ourClientConnection,
        char const* cmdName,
        char const* urlPreSuffix, char const* urlSuffix,
        char const* fullRequestStr);
    virtual void handleCmd_TEARDOWN(RTSPClientConnection* ourClientConnection,
        ServerMediaSubsession* subsession);
    virtual void handleCmd_PLAY(RTSPClientConnection* ourClientConnection,
        ServerMediaSubsession* subsession, char const* fullRequestStr);
    virtual void handleCmd_PAUSE(RTSPClientConnection* ourClientConnection,
        ServerMediaSubsession* subsession);
    virtual void handleCmd_GET_PARAMETER(RTSPClientConnection* ourClientConnection,
        ServerMediaSubsession* subsession, char const* fullRequestStr);
    virtual void handleCmd_SET_PARAMETER(RTSPClientConnection* ourClientConnection,
        ServerMediaSubsession* subsession, char const* fullRequestStr);
  protected:
    void deleteStreamByTrack(unsigned trackNum);
    void reclaimStreamStates();
    Boolean isMulticast() const { return fIsMulticast; }

    // Shortcuts for setting up a RTSP response (prior to sending it):
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr) { ourClientConnection->setRTSPResponse(responseStr); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, u_int32_t sessionId) { ourClientConnection->setRTSPResponse(responseStr, sessionId); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, char const* contentStr) { ourClientConnection->setRTSPResponse(responseStr, contentStr); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, u_int32_t sessionId, char const* contentStr) { ourClientConnection->setRTSPResponse(responseStr, sessionId, contentStr); }

  protected:
    RTSPServer& fOurRTSPServer; // same as ::fOurServer
    Boolean fIsMulticast, fStreamAfterSETUP;
    unsigned char fTCPStreamIdCount; // used for (optional) RTP/TCP
    Boolean usesTCPTransport() const { return fTCPStreamIdCount > 0; }
    unsigned fNumStreamStates;
    struct streamState {
      ServerMediaSubsession* subsession;
      int tcpSocketNum;
      void* streamToken;
    } * fStreamStates;
  };
```

`RTSPServer::RTSPClientSession` 继承自 `GenericMediaServer::ClientSession`：
```
  // The state of an individual client session (using one or more sequential TCP connections) handled by a server:
  class ClientSession {
  protected:
    ClientSession(GenericMediaServer& ourServer, u_int32_t sessionId);
    virtual ~ClientSession();

    UsageEnvironment& envir() { return fOurServer.envir(); }
    void noteLiveness();
    static void noteClientLiveness(ClientSession* clientSession);
    static void livenessTimeoutTask(ClientSession* clientSession);

  protected:
    friend class GenericMediaServer;
    friend class ClientConnection;
    GenericMediaServer& fOurServer;
    u_int32_t fOurSessionId;
    ServerMediaSession* fOurServerMediaSession;
    TaskToken fLivenessCheckTask;
  };
```
不难理解 `RTSPServer::RTSPClientSession` 用于封装整个流媒体会话，处理那些要求流媒体会话已经建立的 RTSP 请求，如 `PLAY` 等。

具体的流媒体数据的交互，如音视频文件/数据的解析，RTP/RTCP 数据的打包及收发等，则依赖于 `ServerMediaSession` 和 `ServerMediaSubsession`。

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
