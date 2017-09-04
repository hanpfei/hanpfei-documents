---
title: live555 源码分析：RTSPServer
date: 2017-09-03 12:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

live555 使用 RTSP/RTP/RTCP 协议来实现流媒体的传输，其中使用 RTSP 来建立流媒体会话，并对流媒体会话进行控制。在 live555 中，通过类 `RTSPServerSupportingHTTPStreaming::RTSPClientConnectionSupportingHTTPStreaming` 来处理 RTSP 请求。客户端发送过来的请求在其父类 `GenericMediaServer::ClientConnection` 的 `incomingRequestHandler(void*, int /*mask*/)` 函数中接收，并在其父类 `RTSPServer::RTSPClientConnection` 的函数 `handleRequestBytes()` 中处理。
<!--more-->
# RTSP 消息格式
在具体的看 `GenericMediaServer::ClientConnection` 的 `incomingRequestHandler(void*, int /*mask*/)` 函数的实现之前，我们先来看一下 RTSP 消息的格式。

RTSP 消息分为请求消息和响应消息，从客户端发向服务器的请求消息的格式如下：
```
       Request      =       Request-Line
                    *(      general-header
                    |       request-header
                    |       entity-header )
                            CRLF
                            [ message-body ]
```

请求消息的第一行是请求行，其中包含了要对资源应用的方法，资源的标识符，也就是 URL，以及使用的协议。请求行的具体格式如下：
```
Request-Line = Method SP Request-URI SP RTSP-Version CRLF
```
上面格式中的 SP 表示空格，即请求行中不同元素以空格分隔，并以 `\n\r`，即 CRLF 结束。RTSP 支持的 Method 主要有如下这些：
```
   Method         =         "DESCRIBE"
                  |         "ANNOUNCE"
                  |         "GET_PARAMETER"
                  |         "OPTIONS"
                  |         "PAUSE"
                  |         "PLAY"
                  |         "RECORD"
                  |         "REDIRECT"
                  |         "SETUP"
                  |         "SET_PARAMETER"
                  |         "TEARDOWN"
                  |         extension-method

  extension-method = token
```
URI 的格式为：
```
Request-URI = "*" | absolute_URI
```
即可以是 "*" 或一个完整的 URI，其中前者表示操作应用于所有资源。

RTSP 版本号的格式为：
```
RTSP-Version = "RTSP" "/" 1*DIGIT "." 1*DIGIT
```
即字符串 "RTSP/" 后面加上以点号 (".") 分隔的两个整数。

请求行之后的是头部。头部分为三类，分别是通用头部、请求头部和实体头部。通用头部主要包含如下这些：
```
      general-header     =     Cache-Control
                         |     Connection
                         |     Date
                         |     Via
```

请求头部有如下这些：
```
  request-header  =          Accept
                  |          Accept-Encoding
                  |          Accept-Language 
                  |          Authorization
                  |          From
                  |          If-Modified-Since
                  |          Range
                  |          Referer
                  |          User-Agent
```

实体头部包括这些：
```
     entity-header       =    Allow
                         |    Content-Base
                         |    Content-Encoding
                         |    Content-Language
                         |    Content-Length
                         |    Content-Location
                         |    Content-Type
                         |    Expires
                         |    Last-Modified
                         |    extension-header
     extension-header    =    message-header
```
每个头部也以 CRLF 结束。在所有头部之后，需要添加另外一个 CRLF 来将头部与消息体隔开。消息体最后也要添加 CRLF 作为结束。

RTSP 的请求消息结构与 HTTP 的非常类似。

`OPTIONS` 请求消息示例：
```
OPTIONS rtsp://10.240.248.20:8554/raw_h264_stream.264 RTSP/1.0
CSeq: 1
User-Agent: Lavf56.40.101

```

`DESCRIBE` 请求消息示例：
```
DESCRIBE rtsp://10.240.248.20:8554/raw_h264_stream.264 RTSP/1.0
Accept: application/sdp
CSeq: 2
User-Agent: Lavf56.40.101

```

`SETUP` 请求消息示例：
```
SETUP rtsp://10.240.248.20:8554/raw_h264_stream.264/track1 RTSP/1.0
Transport: RTP/AVP/UDP;unicast;client_port=27056-27057
CSeq: 3
User-Agent: Lavf56.40.101

```

`PLAY` 请求消息示例：
```
PLAY rtsp://10.240.248.20:8554/raw_h264_stream.264/ RTSP/1.0
Range: npt=0.000-
CSeq: 4
User-Agent: Lavf56.40.101
Session: 92C91EC2

```

响应消息格式如下：
```
     Response    =     Status-Line
                 *(    general-header
                 |     response-header
                 |     entity-header )
                       CRLF
                       [ message-body ]
```

响应消息的第一行是状态行，由协议版本号，数字状态码，和状态码关联的文本形式说明，具体格式定义如下：
```
Status-Line =   RTSP-Version SP Status-Code SP Reason-Phrase CRLF
```
状态行之后的是头部，它与请求消息的头部类似，只是那里的请求头部替换为了响应头部。响应头部主要包括如下这些：
```
   response-header  =     Location
                    |     Proxy-Authenticate
                    |     Public
                    |     Retry-After
                    |     Server
                    |     Vary
                    |     WWW-Authenticate
```
每个头部也以 CRLF 结束。在所有头部之后，需要添加另外一个 CRLF 来将头部与消息体隔开。消息体最后也要添加 CRLF 作为结束。

`OPTIONS` 响应消息示例：
```
RTSP/1.0 200 OK
CSeq: 1
Date: Fri, Sep 01 2017 07:18:16 GMT
Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, GET_PARAMETER, SET_PARAMETER

```

`DESCRIBE` 响应消息示例：
```
RTSP/1.0 200 OK
CSeq: 2
Date: Fri, Sep 01 2017 07:18:16 GMT
Content-Base: rtsp://10.240.248.20:8554/raw_h264_stream.264/
Content-Type: application/sdp
Content-Length: 531

v=0
o=- 1504250296129739 1 IN IP4 10.240.248.20
s=H.264 Video, streamed by the LIVE555 Media Server
i=raw_h264_stream.264
t=0 0
a=tool:LIVE555 Streaming Media v2017.07.18
a=type:broadcast
a=control:*
a=range:npt=0-
a=x-qt-text-nam:H.264 Video, streamed by the LIVE555 Media Server
a=x-qt-text-inf:raw_h264_stream.264
m=video 0 RTP/AVP 96
c=IN IP4 0.0.0.0
b=AS:500
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42802A;sprop-parameter-sets=Z0KAKtoBEA8eXlIKDAoNoUJq,aM4G4g==
a=control:track1
RTSPClientConnection[0x8018c0]::handleRequestBytes() read 163 new bytes:SETUP rtsp://10.240.248.20:8554/raw_h264_stream.264/track1 RTSP/1.0
Transport: RTP/AVP/UDP;unicast;client_port=27056-27057
CSeq: 3
User-Agent: Lavf56.40.101

```
在这个响应中，消息体为 SDP 消息。


`SETUP` 响应消息示例：
```
RTSP/1.0 200 OK
CSeq: 3
Date: Fri, Sep 01 2017 07:18:16 GMT
Transport: RTP/AVP;unicast;destination=10.240.248.20;source=10.240.248.20;client_port=27056-27057;server_port=6970-6971
Session: 92C91EC2;timeout=65

```

`PLAY` 响应消息示例：
```
RTSP/1.0 200 OK
CSeq: 4
Date: Fri, Sep 01 2017 07:18:16 GMT
Range: npt=0.000-
Session: 92C91EC2
RTP-Info: url=rtsp://10.240.248.20:8554/raw_h264_stream.264/track1;seq=32567;rtptime=2735105232

```

RTSP 消息格式基本上就是这样。

# RTSP 请求的处理
有了 RTSP 消息格式的认识之后，来看 live555 `RTSPServer` 中对于 RTSP 消息的处理函数 `RTSPServer::RTSPClientConnection::handleRequestBytes()`。

这个函数的定义有点长，抛去 HTTP 的处理，RTSP 的处理逻辑如下：
```
void RTSPServer::RTSPClientConnection::handleRequestBytes(int newBytesRead) {
  int numBytesRemaining = 0;
  ++fRecursionCount;
  
  do {
    RTSPServer::RTSPClientSession* clientSession = NULL;

    if (newBytesRead < 0 || (unsigned)newBytesRead >= fRequestBufferBytesLeft) {
      // Either the client socket has died, or the request was too big for us.
      // Terminate this connection:
#ifdef DEBUG
      fprintf(stderr, "RTSPClientConnection[%p]::handleRequestBytes() read %d new bytes (of %d); terminating connection!\n", this, newBytesRead, fRequestBufferBytesLeft);
#endif
      fIsActive = False;
      break;
    }
    
    Boolean endOfMsg = False;
    unsigned char* ptr = &fRequestBuffer[fRequestBytesAlreadySeen];
#ifdef DEBUG
    ptr[newBytesRead] = '\0';
    fprintf(stderr, "RTSPClientConnection[%p]::handleRequestBytes() %s %d new bytes:%s\n",
	    this, numBytesRemaining > 0 ? "processing" : "read", newBytesRead, ptr);
#endif

    . . . . . .
    
    unsigned char* tmpPtr = fLastCRLF + 2;
    if (fBase64RemainderCount == 0) { // no more Base-64 bytes remain to be read/decoded
      // Look for the end of the message: <CR><LF><CR><LF>
      if (tmpPtr < fRequestBuffer)
        tmpPtr = fRequestBuffer;
      while (tmpPtr < &ptr[newBytesRead - 1]) {
        if (*tmpPtr == '\r' && *(tmpPtr + 1) == '\n') {
          if (tmpPtr - fLastCRLF == 2) { // This is it:
            endOfMsg = True;
            break;
          }
          fLastCRLF = tmpPtr;
        }
        ++tmpPtr;
      }
    }

    fRequestBufferBytesLeft -= newBytesRead;
    fRequestBytesAlreadySeen += newBytesRead;

    if (!endOfMsg) break; // subsequent reads will be needed to complete the request
    
    // Parse the request string into command name and 'CSeq', then handle the command:
    fRequestBuffer[fRequestBytesAlreadySeen] = '\0';
    char cmdName[RTSP_PARAM_STRING_MAX];
    char urlPreSuffix[RTSP_PARAM_STRING_MAX];
    char urlSuffix[RTSP_PARAM_STRING_MAX];
    char cseq[RTSP_PARAM_STRING_MAX];
    char sessionIdStr[RTSP_PARAM_STRING_MAX];
    unsigned contentLength = 0;
    fLastCRLF[2] = '\0'; // temporarily, for parsing
    Boolean parseSucceeded = parseRTSPRequestString(
        (char*) fRequestBuffer,
        fLastCRLF + 2 - fRequestBuffer,
        cmdName, sizeof cmdName,
        urlPreSuffix, sizeof urlPreSuffix,
        urlSuffix, sizeof urlSuffix,
        cseq, sizeof cseq,
        sessionIdStr, sizeof sessionIdStr,
        contentLength);
    fLastCRLF[2] = '\r'; // restore its value
    Boolean playAfterSetup = False;
    if (parseSucceeded) {
#ifdef DEBUG
      fprintf(stderr, "parseRTSPRequestString() succeeded, returning cmdName \"%s\", urlPreSuffix \"%s\", urlSuffix \"%s\", CSeq \"%s\", Content-Length %u, with %d bytes following the message.\n", cmdName, urlPreSuffix, urlSuffix, cseq, contentLength, ptr + newBytesRead - (tmpPtr + 2));
#endif
      // If there was a "Content-Length:" header, then make sure we've received all of the data that it specified:
      if (ptr + newBytesRead < tmpPtr + 2 + contentLength) break; // we still need more data; subsequent reads will give it to us 
      
      // If the request included a "Session:" id, and it refers to a client session that's
      // current ongoing, then use this command to indicate 'liveness' on that client session:
      Boolean const requestIncludedSessionId = sessionIdStr[0] != '\0';
      if (requestIncludedSessionId) {
        clientSession
          = (RTSPServer::RTSPClientSession*)(fOurRTSPServer.lookupClientSession(sessionIdStr));
        if (clientSession != NULL) clientSession->noteLiveness();
      }
    
      // We now have a complete RTSP request.
      // Handle the specified command (beginning with commands that are session-independent):
      fCurrentCSeq = cseq;
      if (strcmp(cmdName, "OPTIONS") == 0) {
        // If the "OPTIONS" command included a "Session:" id for a session that doesn't exist,
        // then treat this as an error:
        if (requestIncludedSessionId && clientSession == NULL) {
          handleCmd_sessionNotFound();
        } else {
          // Normal case:
          handleCmd_OPTIONS();
        }
      } else if (urlPreSuffix[0] == '\0' && urlSuffix[0] == '*'
          && urlSuffix[1] == '\0') {
        // The special "*" URL means: an operation on the entire server.  This works only for GET_PARAMETER and SET_PARAMETER:
        if (strcmp(cmdName, "GET_PARAMETER") == 0) {
          handleCmd_GET_PARAMETER((char const*) fRequestBuffer);
        } else if (strcmp(cmdName, "SET_PARAMETER") == 0) {
          handleCmd_SET_PARAMETER((char const*) fRequestBuffer);
        } else {
          handleCmd_notSupported();
        }
      } else if (strcmp(cmdName, "DESCRIBE") == 0) {
        handleCmd_DESCRIBE(urlPreSuffix, urlSuffix,
            (char const*) fRequestBuffer);
      } else if (strcmp(cmdName, "SETUP") == 0) {
        Boolean areAuthenticated = True;

        if (!requestIncludedSessionId) {
          // No session id was present in the request.
          // So create a new "RTSPClientSession" object for this request.

          // But first, make sure that we're authenticated to perform this command:
          char urlTotalSuffix[2 * RTSP_PARAM_STRING_MAX];
          // enough space for urlPreSuffix/urlSuffix'\0'
          urlTotalSuffix[0] = '\0';
          if (urlPreSuffix[0] != '\0') {
            strcat(urlTotalSuffix, urlPreSuffix);
            strcat(urlTotalSuffix, "/");
          }
          strcat(urlTotalSuffix, urlSuffix);
          if (authenticationOK("SETUP", urlTotalSuffix,
              (char const*) fRequestBuffer)) {
            clientSession =
                (RTSPServer::RTSPClientSession*) fOurRTSPServer.createNewClientSessionWithId();
          } else {
            areAuthenticated = False;
          }
        }
        if (clientSession != NULL) {
          clientSession->handleCmd_SETUP(this, urlPreSuffix, urlSuffix,
              (char const*) fRequestBuffer);
          playAfterSetup = clientSession->fStreamAfterSETUP;
        } else if (areAuthenticated) {
          handleCmd_sessionNotFound();
        }
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
      } else if (strcmp(cmdName, "REGISTER") == 0 || strcmp(cmdName, "DEREGISTER") == 0) {
        // Because - unlike other commands - an implementation of this command needs
        // the entire URL, we re-parse the command to get it:
        char* url = strDupSize((char*) fRequestBuffer);
        if (sscanf((char*) fRequestBuffer, "%*s %s", url) == 1) {
          // Check for special command-specific parameters in a "Transport:" header:
          Boolean reuseConnection, deliverViaTCP;
          char* proxyURLSuffix;
          parseTransportHeaderForREGISTER((const char*) fRequestBuffer,
              reuseConnection, deliverViaTCP, proxyURLSuffix);

          handleCmd_REGISTER(cmdName, url, urlSuffix,
              (char const*) fRequestBuffer, reuseConnection, deliverViaTCP,
              proxyURLSuffix);
          delete[] proxyURLSuffix;
        } else {
          handleCmd_bad();
        }
        delete[] url;
      } else {
        // The command is one that we don't handle:
        handleCmd_notSupported();
      }
    }
    . . . . . .
    
#ifdef DEBUG
    fprintf(stderr, "sending response: %s", fResponseBuffer);
#endif
    send(fClientOutputSocket, (char const*)fResponseBuffer, strlen((char*)fResponseBuffer), 0);
    
    if (playAfterSetup) {
      // The client has asked for streaming to commence now, rather than after a
      // subsequent "PLAY" command.  So, simulate the effect of a "PLAY" command:
      clientSession->handleCmd_withinSession(this, "PLAY", urlPreSuffix, urlSuffix, (char const*)fRequestBuffer);
    }
    
    // Check whether there are extra bytes remaining in the buffer, after the end of the request (a rare case).
    // If so, move them to the front of our buffer, and keep processing it, because it might be a following, pipelined request.
    unsigned requestSize = (fLastCRLF+4-fRequestBuffer) + contentLength;
    numBytesRemaining = fRequestBytesAlreadySeen - requestSize;
    resetRequestBuffer(); // to prepare for any subsequent request

    if (numBytesRemaining > 0) {
      memmove(fRequestBuffer, &fRequestBuffer[requestSize], numBytesRemaining);
      newBytesRead = numBytesRemaining;
    }
  } while (numBytesRemaining > 0);

  --fRecursionCount;
  if (!fIsActive) {
    if (fRecursionCount > 0) closeSockets(); else delete this;
    // Note: The "fRecursionCount" test is for a pathological situation where we reenter the event loop and get called recursively
    // while handling a command (e.g., while handling a "DESCRIBE", to get a SDP description).
    // In such a case we don't want to actually delete ourself until we leave the outermost call.
  }
}
```

这个函数会通过一个 do-while 循环逐个处理从 socket 中读取的在缓冲区存放的完整的 RTSP 消息。在循环体的开始部分，`RTSPServer::RTSPClientConnection::handleRequestBytes()` 先找到请求消息头部的结束位置，这通过如下这段代码来实现：
```
    unsigned char* tmpPtr = fLastCRLF + 2;
    if (fBase64RemainderCount == 0) { // no more Base-64 bytes remain to be read/decoded
      // Look for the end of the message: <CR><LF><CR><LF>
      if (tmpPtr < fRequestBuffer)
        tmpPtr = fRequestBuffer;
      while (tmpPtr < &ptr[newBytesRead - 1]) {
        if (*tmpPtr == '\r' && *(tmpPtr + 1) == '\n') {
          if (tmpPtr - fLastCRLF == 2) { // This is it:
            endOfMsg = True;
            break;
          }
          fLastCRLF = tmpPtr;
        }
        ++tmpPtr;
      }
    }

    fRequestBufferBytesLeft -= newBytesRead;
    fRequestBytesAlreadySeen += newBytesRead;

    if (!endOfMsg) break; // subsequent reads will be needed to complete the request
```
由前面 RTSP 消息格式的部分，我们知道，RTSP 请求消息头部都以连续的两个 "\n\r"，即 `<CR><LF><CR><LF>` 结束，寻找消息的结束位置即是找到这个序列的位置，`fLastCRLF` 指向这个序列的开始位置。

如果找不到请求消息头部的结束位置的话 ，也就是说从 socket 中读取的数据不包含完整的消息头部，消息有一部分数据还没有读到，此时会跳出循环结束处理。

找到了请求消息头部的结束位置之后，就是从消息头部中解析出不同的元素了。这主要通过 `parseRTSPRequestString()` 函数完成：
```
    // Parse the request string into command name and 'CSeq', then handle the command:
    fRequestBuffer[fRequestBytesAlreadySeen] = '\0';
    char cmdName[RTSP_PARAM_STRING_MAX];
    char urlPreSuffix[RTSP_PARAM_STRING_MAX];
    char urlSuffix[RTSP_PARAM_STRING_MAX];
    char cseq[RTSP_PARAM_STRING_MAX];
    char sessionIdStr[RTSP_PARAM_STRING_MAX];
    unsigned contentLength = 0;
    fLastCRLF[2] = '\0'; // temporarily, for parsing
    Boolean parseSucceeded = parseRTSPRequestString(
        (char*) fRequestBuffer,
        fLastCRLF + 2 - fRequestBuffer,
        cmdName, sizeof cmdName,
        urlPreSuffix, sizeof urlPreSuffix,
        urlSuffix, sizeof urlSuffix,
        cseq, sizeof cseq,
        sessionIdStr, sizeof sessionIdStr,
        contentLength);
    fLastCRLF[2] = '\r'; // restore its value
```

这里并不是解析出 RTSP 消息头部中所有的元素，而是仅仅解析出 RTSP 消息头部中决定后续如何处理的几个元素，包括请求的 Method，URL，CSeq 和 SessionId，消息体长度 ContentLength 等。解析成功说明消息是一个有效的 RTSP 消息，否则是一个 HTTP 消息。这里仅看解析成功情况下的处理。

解析成功之后，会先判断消息体是否完整：此时 (ptr + newBytesRead) 指向缓冲区中读取的数据的结束位置，`tmpPtr` 指向消息头部结束位置出的两个  "\n\r" 序列中后面的那个，即消息头部中的最后一个 `<CR><LF>`，这里通过比较 (ptr + newBytesRead) 和 `(tmpPtr + 2 + contentLength)` 来判断缓冲区中的消息是否完整。

同样的消息不完整时，跳出循环，结束处理。

随后，在消息的处理需要已经建立流媒体会话时，则先查找建立的流媒体会话，并为该会话执行保活操作。

随后即是根据 RTSP 请求消息类型的不同，执行分别的处理。

## OPTIONS 请求的处理
`OPTIONS` 请求的处理如下：
```
      if (strcmp(cmdName, "OPTIONS") == 0) {
        // If the "OPTIONS" command included a "Session:" id for a session that doesn't exist,
        // then treat this as an error:
        if (requestIncludedSessionId && clientSession == NULL) {
          handleCmd_sessionNotFound();
        } else {
          // Normal case:
          handleCmd_OPTIONS();
        }
      }
```
根据规范，`OPTIONS` 请求可以在流媒体会话的任何阶段发起，当请求头部中包含了 SessionId 但又找不到对应的客户端会话结构时，通过 `handleCmd_sessionNotFound()` 来处理：
```
void RTSPServer::RTSPClientConnection::handleCmd_sessionNotFound() {
  setRTSPResponse("454 Session Not Found");
}
. . . . . .
void RTSPServer::RTSPClientConnection::setRTSPResponse(
    char const* responseStr) {
  snprintf((char*) fResponseBuffer, sizeof fResponseBuffer,
      "RTSP/1.0 %s\r\n"
      "CSeq: %s\r\n"
      "%s\r\n",
      responseStr,
      fCurrentCSeq,
      dateHeader());
}
```

其中 `dateHeader()` 返回 RTSP `Date` 头部，其中的时间为 RTSP 头部时间格式的当前时间字符串：
```
char const* dateHeader() {
  static char buf[200];
#if !defined(_WIN32_WCE)
  time_t tt = time(NULL);
  strftime(buf, sizeof buf, "Date: %a, %b %d %Y %H:%M:%S GMT\r\n", gmtime(&tt));
#else
. . . . . .
#endif
  return buf;
}
```

`handleCmd_sessionNotFound()` 的处理即是向 `fResponseBuffer` 填充一个 454 的 RTSP 错误响应消息。

正常情况下，通过 `handleCmd_OPTIONS()` 对请求进行处理：
```
char const* RTSPServer::allowedCommandNames() {
  return "OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, GET_PARAMETER, SET_PARAMETER";
}
. . . . . .
void RTSPServer::RTSPClientConnection::handleCmd_OPTIONS() {
  snprintf((char*) fResponseBuffer, sizeof fResponseBuffer,
      "RTSP/1.0 200 OK\r\nCSeq: %s\r\n%sPublic: %s\r\n\r\n", fCurrentCSeq,
      dateHeader(), fOurRTSPServer.allowedCommandNames());
}
```
这里是向 `fResponseBuffer` 填充一个成功的 RTSP 响应消息，其中的 `Public` 头部包含了 RTSP 服务器支持的所有 RTSP Methods。

## 应用于整个服务器的请求的处理
特殊的 URL "*" 表示，请求的操作应用于整个服务器。按照 RTSP 的规范，只有两种操作可以应用于整个服务器，即 `GET_PARAMETER` 和 `SET_PARAMETER`。

 应用于整个服务器 的 `GET_PARAMETER` 和 `SET_PARAMETER` 请求的处理如下：
```
      } else if (urlPreSuffix[0] == '\0' && urlSuffix[0] == '*'
          && urlSuffix[1] == '\0') {
        // The special "*" URL means: an operation on the entire server.  This works only for GET_PARAMETER and SET_PARAMETER:
        if (strcmp(cmdName, "GET_PARAMETER") == 0) {
          handleCmd_GET_PARAMETER((char const*) fRequestBuffer);
        } else if (strcmp(cmdName, "SET_PARAMETER") == 0) {
          handleCmd_SET_PARAMETER((char const*) fRequestBuffer);
        } else {
          handleCmd_notSupported();
        }
      }
```
这里分别通过调用 `handleCmd_GET_PARAMETER()` 和 `handleCmd_SET_PARAMETER()` 来处理 `GET_PARAMETER` 和 `SET_PARAMETER` 请求：
```
void RTSPServer::RTSPClientConnection
::handleCmd_GET_PARAMETER(char const* /*fullRequestStr*/) {
  // By default, we implement "GET_PARAMETER" (on the entire server) just as a 'no op', and send back a dummy response.
  // (If you want to handle this type of "GET_PARAMETER" differently, you can do so by defining a subclass of "RTSPServer"
  // and "RTSPServer::RTSPClientConnection", and then reimplement this virtual function in your subclass.)
  setRTSPResponse("200 OK", LIVEMEDIA_LIBRARY_VERSION_STRING);
}

void RTSPServer::RTSPClientConnection
::handleCmd_SET_PARAMETER(char const* /*fullRequestStr*/) {
  // By default, we implement "SET_PARAMETER" (on the entire server) just as a 'no op', and send back an empty response.
  // (If you want to handle this type of "SET_PARAMETER" differently, you can do so by defining a subclass of "RTSPServer"
  // and "RTSPServer::RTSPClientConnection", and then reimplement this virtual function in your subclass.)
  setRTSPResponse("200 OK");
}
. . . . . .
void RTSPServer::RTSPClientConnection
::setRTSPResponse(char const* responseStr, char const* contentStr) {
  if (contentStr == NULL) contentStr = "";
  unsigned const contentLen = strlen(contentStr);

  snprintf((char*) fResponseBuffer, sizeof fResponseBuffer,
      "RTSP/1.0 %s\r\n"
      "CSeq: %s\r\n"
      "%s"
      "Content-Length: %d\r\n\r\n"
      "%s",
      responseStr,
      fCurrentCSeq,
      dateHeader(),
      contentLen,
      contentStr);
}
```
live555 liveMedia 库中的 `RTSPServer::RTSPClientConnection` 本身并没有实现 `GET_PARAMETER` 和 `SET_PARAMETER` 请求。如果应用程序需要提供这两个方法的实现，需要继承 `RTSPServer` 和 `RTSPServer::RTSPClientConnection`，并实现相应的虚函数。

默认情况下，在处理 `GET_PARAMETER` 时，`RTSPServer::RTSPClientConnection` 把库的版本返回给客户端，对于 `SET_PARAMETER` 请求，则总是返回成功。

## DESCRIBE 请求的处理
`DESCRIBE` 请求的处理如下：
```
      } else if (strcmp(cmdName, "DESCRIBE") == 0) {
        handleCmd_DESCRIBE(urlPreSuffix, urlSuffix,
            (char const*) fRequestBuffer);
      }
```

这里通过 `handleCmd_DESCRIBE()` 函数处理 `DESCRIBE` 消息。

## SETUP 请求的处理
SETUP 请求的处理如下：
```
      } else if (strcmp(cmdName, "SETUP") == 0) {
        Boolean areAuthenticated = True;

        if (!requestIncludedSessionId) {
          // No session id was present in the request.
          // So create a new "RTSPClientSession" object for this request.

          // But first, make sure that we're authenticated to perform this command:
          char urlTotalSuffix[2 * RTSP_PARAM_STRING_MAX];
          // enough space for urlPreSuffix/urlSuffix'\0'
          urlTotalSuffix[0] = '\0';
          if (urlPreSuffix[0] != '\0') {
            strcat(urlTotalSuffix, urlPreSuffix);
            strcat(urlTotalSuffix, "/");
          }
          strcat(urlTotalSuffix, urlSuffix);
          if (authenticationOK("SETUP", urlTotalSuffix,
              (char const*) fRequestBuffer)) {
            clientSession =
                (RTSPServer::RTSPClientSession*) fOurRTSPServer.createNewClientSessionWithId();
          } else {
            areAuthenticated = False;
          }
        }
        if (clientSession != NULL) {
          clientSession->handleCmd_SETUP(this, urlPreSuffix, urlSuffix,
              (char const*) fRequestBuffer);
          playAfterSetup = clientSession->fStreamAfterSETUP;
        } else if (areAuthenticated) {
          handleCmd_sessionNotFound();
        }
      }
```
如果 `SETUP` 请求中不带有 SessionID，即表示流媒体会话还没有创建，则这里会先尝试为请求创建流媒体会话。创建的过程如下：

1. 首先对请求进行认证。认证需要 URL 的完整路径部分，请求头部解析之后，`urlPreSuffix` 保存请求 URL 的路径部分，最后的路径分量前面的所有部分，而 `urlSuffix` 则保存 URL 路径部分的最后一个分量。比如 URL 为 `rtsp://10.240.248.20:8554/raw_h264_stream.264`，`urlPreSuffix` 为空字符串 `""`，`urlSuffix` 为 `"raw_h264_stream.264"`，URL 为 `rtsp://10.240.248.20:8000/video/raw_h264_stream.264`，`urlPreSuffix` 为空字符串 `"video"`，`urlSuffix` 为 `"raw_h264_stream.264"`。因而在认证之前，会先重新拼出 URL 的完整路径部分，然后再通过 `authenticationOK()` 执行认证。
"LIVE555 Media Server" 不提供认证，认证将总是成功。这里不再详细分析认证过程。

2. 请求认证成功时，则通过`GenericMediaServer::createNewClientSessionWithId()` 创建 `RTSPServer::RTSPClientSession` 对象。`GenericMediaServer` 中 `ClientSession` 相关的几个函数声明如下：
```
  virtual ClientSession* createNewClientSession(u_int32_t sessionId) = 0;

  ClientSession* createNewClientSessionWithId();
      // Generates a new (unused) random session id, and calls the "createNewClientSession()"
      // virtual function with this session id as parameter.

  // Lookup a "ClientSession" object by sessionId (integer, and string):
  ClientSession* lookupClientSession(u_int32_t sessionId);
  ClientSession* lookupClientSession(char const* sessionIdStr);
```
`createNewClientSession()` 为纯虚函数，`createNewClientSessionWithId()` 和两个 `lookupClientSession()` 为非虚函数。

`GenericMediaServer::createNewClientSessionWithId()` 函数的定义如下：
```
GenericMediaServer::ClientSession* GenericMediaServer::createNewClientSessionWithId() {
  u_int32_t sessionId;
  char sessionIdStr[8+1];

  // Choose a random (unused) 32-bit integer for the session id
  // (it will be encoded as a 8-digit hex number).  (We avoid choosing session id 0,
  // because that has a special use by some servers.)
  do {
    sessionId = (u_int32_t)our_random32();
    snprintf(sessionIdStr, sizeof sessionIdStr, "%08X", sessionId);
  } while (sessionId == 0 || lookupClientSession(sessionIdStr) != NULL);

  ClientSession* clientSession = createNewClientSession(sessionId);
  if (clientSession != NULL) fClientSessions->Add(sessionIdStr, clientSession);

  return clientSession;
}

GenericMediaServer::ClientSession*
GenericMediaServer::lookupClientSession(u_int32_t sessionId) {
  char sessionIdStr[8+1];
  snprintf(sessionIdStr, sizeof sessionIdStr, "%08X", sessionId);
  return lookupClientSession(sessionIdStr);
}

GenericMediaServer::ClientSession*
GenericMediaServer::lookupClientSession(char const* sessionIdStr) {
  return (GenericMediaServer::ClientSession*)fClientSessions->Lookup(sessionIdStr);
}
```

创建 `GenericMediaServer::ClientSession` 时，`GenericMediaServer::createNewClientSessionWithId()` 首先找到一个非零，且还没有被占用的随机值作为 SessionID；然后通过 `createNewClientSession()` 函数创建对象；最后将创建的对象缓存起来并返回给调用者。

`createNewClientSession()` 是一个纯虚函数，对于`DynamicRTSPServer` -> `RTSPServerSupportingHTTPStreaming` -> `RTSPServer` -> `GenericMediaServer` 这个继承层次，它是在 `RTSPServer` 中定义的：
```
GenericMediaServer::ClientSession*
RTSPServer::createNewClientSession(u_int32_t sessionId) {
  return new RTSPClientSession(*this, sessionId);
}
```

live555 中 `GenericMediaServer::ClientSession` 的继承层次结构如下：

![](https://www.wolfcstech.com/images/1315506-1ec111f30d81ac1a.png)

有了 `RTSPServer::RTSPClientSession` 之后，则通过它的 `handleCmd_SETUP()` 处理 `SETUP` 请求。同时根据会话的设置，设置 `playAfterSetup`。

当无法创建 `RTSPServer::RTSPClientSession`，但认证成功时，通过 `handleCmd_sessionNotFound()` 返回 454 错误响应给客户端。

***但如果认证失败的话，则似乎不会给客户端任何响应。***

## TEARDOWN、PLAY、PAUSE、GET_PARAMETER、SET_PARAMETER请求的处理
`TEARDOWN`、`PLAY`、`PAUSE`、`GET_PARAMETER`、`SET_PARAMETER` 请求的处理如下：
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
这些请求都需要在流媒体会话创建之后才能发起，因而直接调用客户端会话结构 `RTSPServer::RTSPClientSession` 的 `handleCmd_withinSession()` 来处理。当会话结构不存在时，通过 `handleCmd_sessionNotFound()` 返回错误消息给客户端。

这里的 `GET_PARAMETER` 和 `SET_PARAMETER` 操作仅应用于由传入的 URL 标识的特定资源，在 `RTSPServer::RTSPClientSession` 中对这两个方法的处理，与前面看到的，应用于整个服务器的操作相同：
```
void RTSPServer::RTSPClientSession
::handleCmd_withinSession( RTSPServer::RTSPClientConnection* ourClientConnection,
    char const* cmdName,
    char const* urlPreSuffix, char const* urlSuffix,
    char const* fullRequestStr) {
. . . . . .
  } else if (strcmp(cmdName, "GET_PARAMETER") == 0) {
    handleCmd_GET_PARAMETER(ourClientConnection, subsession, fullRequestStr);
  } else if (strcmp(cmdName, "SET_PARAMETER") == 0) {
    handleCmd_SET_PARAMETER(ourClientConnection, subsession, fullRequestStr);
  }
}
. . . . . .
void RTSPServer::RTSPClientSession
::handleCmd_GET_PARAMETER(RTSPServer::RTSPClientConnection* ourClientConnection,
    ServerMediaSubsession* /*subsession*/, char const* /*fullRequestStr*/) {
  // By default, we implement "GET_PARAMETER" just as a 'keep alive', and send back a dummy response.
  // (If you want to handle "GET_PARAMETER" properly, you can do so by defining a subclass of "RTSPServer"
  // and "RTSPServer::RTSPClientSession", and then reimplement this virtual function in your subclass.)
  setRTSPResponse(ourClientConnection, "200 OK", fOurSessionId, LIVEMEDIA_LIBRARY_VERSION_STRING);
}

void RTSPServer::RTSPClientSession
::handleCmd_SET_PARAMETER(RTSPServer::RTSPClientConnection* ourClientConnection,
			  ServerMediaSubsession* /*subsession*/, char const* /*fullRequestStr*/) {
  // By default, we implement "SET_PARAMETER" just as a 'keep alive', and send back an empty response.
  // (If you want to handle "SET_PARAMETER" properly, you can do so by defining a subclass of "RTSPServer"
  // and "RTSPServer::RTSPClientSession", and then reimplement this virtual function in your subclass.)
  setRTSPResponse(ourClientConnection, "200 OK", fOurSessionId);
}
```
`RTSPServer::RTSPClientSession` 的 `setRTSPResponse()` 操作都是直接委托给 `RTSPServer::RTSPClientConnection` 的对应函数来完成的：
```
  class RTSPClientSession: public GenericMediaServer::ClientSession {
. . . . . .
    // Shortcuts for setting up a RTSP response (prior to sending it):
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr) { ourClientConnection->setRTSPResponse(responseStr); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, u_int32_t sessionId) { ourClientConnection->setRTSPResponse(responseStr, sessionId); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, char const* contentStr) { ourClientConnection->setRTSPResponse(responseStr, contentStr); }
    void setRTSPResponse(RTSPClientConnection* ourClientConnection, char const* responseStr, u_int32_t sessionId, char const* contentStr) { ourClientConnection->setRTSPResponse(responseStr, sessionId, contentStr); }
```

## REGISTER、DEREGISTER 请求的处理
` REGISTER` 和 `DEREGISTER` 不是规范中明确定义的标准 RTSP 方法。这里简单地看一下，live 555 对于它们的处理：
```
 else if (strcmp(cmdName, "REGISTER") == 0 || strcmp(cmdName, "DEREGISTER") == 0) {
        // Because - unlike other commands - an implementation of this command needs
        // the entire URL, we re-parse the command to get it:
        char* url = strDupSize((char*) fRequestBuffer);
        if (sscanf((char*) fRequestBuffer, "%*s %s", url) == 1) {
          // Check for special command-specific parameters in a "Transport:" header:
          Boolean reuseConnection, deliverViaTCP;
          char* proxyURLSuffix;
          parseTransportHeaderForREGISTER((const char*) fRequestBuffer,
              reuseConnection, deliverViaTCP, proxyURLSuffix);

          handleCmd_REGISTER(cmdName, url, urlSuffix,
              (char const*) fRequestBuffer, reuseConnection, deliverViaTCP,
              proxyURLSuffix);
          delete[] proxyURLSuffix;
        } else {
          handleCmd_bad();
        }
        delete[] url;
      }
```

## 错误方法的处理
当 RTSP 消息的方法无法识别时，`RTSPServer::RTSPClientConnection` 通过 `handleCmd_notSupported()` 来处理：
```
void RTSPServer::RTSPClientConnection::handleCmd_notSupported() {
  snprintf((char*) fResponseBuffer, sizeof fResponseBuffer,
      "RTSP/1.0 405 Method Not Allowed\r\nCSeq: %s\r\n%sAllow: %s\r\n\r\n",
      fCurrentCSeq, dateHeader(), fOurRTSPServer.allowedCommandNames());
}
```
这里向 `fResponseBuffer` 填充一个 405 的 RTSP 错误响应消息。

根据 RTSP 请求消息类型的不同，执行分别的处理的过程，都是为对应的请求产生一个 RTSP 响应消息，并放进响应缓冲区中。处理完之后，这个缓冲区中的内容，会被发送给客户端：
```
    send(fClientOutputSocket, (char const*)fResponseBuffer, strlen((char*)fResponseBuffer), 0);
```

如果流媒体会话被配置为在 SETUP 之后立即启动播放的话，则在会立即启动播放：
```
    if (playAfterSetup) {
      // The client has asked for streaming to commence now, rather than after a
      // subsequent "PLAY" command.  So, simulate the effect of a "PLAY" command:
      clientSession->handleCmd_withinSession(this, "PLAY", urlPreSuffix, urlSuffix, (char const*)fRequestBuffer);
    }
```

在循环体的最后，会将已经处理的消息的内容移除出请求缓冲区，并将剩余的还未处理的数据移动到缓冲区的开头部分：
```
    unsigned requestSize = (fLastCRLF+4-fRequestBuffer) + contentLength;
    numBytesRemaining = fRequestBytesAlreadySeen - requestSize;
    resetRequestBuffer(); // to prepare for any subsequent request

    if (numBytesRemaining > 0) {
      memmove(fRequestBuffer, &fRequestBuffer[requestSize], numBytesRemaining);
      newBytesRead = numBytesRemaining;
    }
```
其中 `resetRequestBuffer()` 的定义如下：
```
void RTSPServer::RTSPClientConnection::resetRequestBuffer() {
  ClientConnection::resetRequestBuffer();
  
  fLastCRLF = &fRequestBuffer[-3]; // hack: Ensures that we don't think we have end-of-msg if the data starts with <CR><LF>
  fBase64RemainderCount = 0;
}
```

`ClientConnection::resetRequestBuffer()` 的定义如下：
```
void GenericMediaServer::ClientConnection::resetRequestBuffer() {
  fRequestBytesAlreadySeen = 0;
  fRequestBufferBytesLeft = sizeof fRequestBuffer;
}
```

即在 `resetRequestBuffer()` 中会复位 `fRequestBytesAlreadySeen`、`fRequestBufferBytesLeft`、`fLastCRLF` 和 `fBase64RemainderCount` 等状态。注意，newBytesRead 得到了更新，因而如果在缓冲区还存在未处理的不完整的消息的数据，这些状态会在下一次循环迭代的开始处得到正确的更新。

最后，在 `RTSPServer::RTSPClientConnection::handleRequestBytes()` 函数的最后，根据需要会关闭 socket：
```
  --fRecursionCount;
  if (!fIsActive) {
    if (fRecursionCount > 0) closeSockets(); else delete this;
    // Note: The "fRecursionCount" test is for a pathological situation where we reenter the event loop and get called recursively
    // while handling a command (e.g., while handling a "DESCRIBE", to get a SDP description).
    // In such a case we don't want to actually delete ourself until we leave the outermost call.
  }
```

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# live555 源码分析系列文章
[live555 源码分析：简介](http://www.jianshu.com/p/b08729905a8c)
[live555 源码分析：基础设施](http://www.jianshu.com/p/199048a9c959)
[live555 源码分析：MediaSever](http://www.jianshu.com/p/1281f9d1642b)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](http://www.jianshu.com/p/409f20b7e813)
[live555 源码分析：RTSPServer](http://www.jianshu.com/p/62387ac1fe41)
[live555 源码分析： DESCRIBE 的处理](http://www.jianshu.com/p/7ca981ca4c9d)
