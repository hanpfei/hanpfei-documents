---
title: live555 源码分析：MediaSever
date: 2017-08-31 15:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

位于 live555 项目 mediaServer 目录下的是 "LIVE555 Media Server"，它是一个完整的 RTSP 服务器应用程序。它可以把多种媒体文件转为流，提供给请求者。
<!--more-->
这里来看一下 "LIVE555 Media Server" 的实现。抛开其中向终端输出应用程序信息的代码， "LIVE555 Media Server" 主程序的代码像下面这样：
```
#include <BasicUsageEnvironment.hh>
#include "DynamicRTSPServer.hh"
#include "version.hh"

int main(int argc, char** argv) {
  // Begin by setting up our usage environment:
  TaskScheduler* scheduler = BasicTaskScheduler::createNew();
  UsageEnvironment* env = BasicUsageEnvironment::createNew(*scheduler);

  UserAuthenticationDatabase* authDB = NULL;
#ifdef ACCESS_CONTROL
  // To implement client access control to the RTSP server, do the following:
  authDB = new UserAuthenticationDatabase;
  authDB->addUserRecord("username1", "password1"); // replace these with real strings
  // Repeat the above with each <username>, <password> that you wish to allow
  // access to the server.
#endif

  // Create the RTSP server.  Try first with the default port number (554),
  // and then with the alternative port number (8554):
  RTSPServer* rtspServer;
  portNumBits rtspServerPortNum = 554;
  rtspServer = DynamicRTSPServer::createNew(*env, rtspServerPortNum, authDB);
  if (rtspServer == NULL) {
    rtspServerPortNum = 8554;
    rtspServer = DynamicRTSPServer::createNew(*env, rtspServerPortNum, authDB);
  }
  if (rtspServer == NULL) {
    *env << "Failed to create RTSP server: " << env->getResultMsg() << "\n";
    exit(1);
  }
  . . . . . .
  if (rtspServer->setUpTunnelingOverHTTP(80) || rtspServer->setUpTunnelingOverHTTP(8000) || rtspServer->setUpTunnelingOverHTTP(8080)) {
    *env << "(We use port " << rtspServer->httpServerPortNum() << " for optional RTSP-over-HTTP tunneling, or for HTTP live streaming (for indexed Transport Stream files only).)\n";
  } else {
    *env << "(RTSP-over-HTTP tunneling is not available.)\n";
  }

  env->taskScheduler().doEventLoop(); // does not return

  return 0; // only to prevent compiler warning
}
```

"LIVE555 Media Server" 的主程序非常简单，做的事情仅仅如下面这样：
1. 创建 `TaskScheduler` 用于执行任务调度。关于 `TaskScheduler` 的更详细信息可以参考 [live555 源码分析：基础设施](http://www.jianshu.com/p/199048a9c959)。
2. 创建 `UsageEnvironment` 用于日志输出等 I/O 操作。关于 `UsageEnvironment` 的更详细信息可以参考 [live555 源码分析：基础设施](http://www.jianshu.com/p/199048a9c959)。
3. 创建 `RTSPServer` 用于接受连接、处理请求等。这里会首先尝试使用 554 端口，如果失败，就尝试使用 8554 端口，如果两个端口都没法用，就失败退出程序。
4. 为 `RTSPServer` 设置 HTTP 隧道端口。这里会依次尝试使用 80、8000 和 8080 端口。
5. 执行事件循环。

`TaskScheduler` 用于执行任务调度，但要监听的 socket，以及 socket 上的 I/O 事件的处理程序，则需要 `RTSPServer`，也就是 `DynamicRTSPServer` 提供。`DynamicRTSPServer` 是 "LIVE555 Media Server" 应用程序的核心，其定义如下：
```
#ifndef _RTSP_SERVER_SUPPORTING_HTTP_STREAMING_HH
#include "RTSPServerSupportingHTTPStreaming.hh"
#endif

class DynamicRTSPServer: public RTSPServerSupportingHTTPStreaming {
public:
  static DynamicRTSPServer* createNew(UsageEnvironment& env, Port ourPort,
				      UserAuthenticationDatabase* authDatabase,
				      unsigned reclamationTestSeconds = 65);

protected:
  DynamicRTSPServer(UsageEnvironment& env, int ourSocket, Port ourPort,
		    UserAuthenticationDatabase* authDatabase, unsigned reclamationTestSeconds);
  // called only by createNew();
  virtual ~DynamicRTSPServer();

protected: // redefined virtual functions
  virtual ServerMediaSession*
  lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession);
};
```

`DynamicRTSPServer` 的类层次结构如下图：
![](https://www.wolfcstech.com/images/1315506-791dcd5f2574c420.png)

要监听的 socket 以及 socket 上的 I/O 事件的处理程序是在 `DynamicRTSPServer` 创建的过程中注册给 `TaskScheduler` 的。`DynamicRTSPServer` 需要通过其静态函数 `createNew()` 创建，该函数定义如下：
```
DynamicRTSPServer*
DynamicRTSPServer::createNew(UsageEnvironment& env, Port ourPort,
			     UserAuthenticationDatabase* authDatabase,
			     unsigned reclamationTestSeconds) {
  int ourSocket = setUpOurSocket(env, ourPort);
  if (ourSocket == -1) return NULL;

  return new DynamicRTSPServer(env, ourSocket, ourPort, authDatabase, reclamationTestSeconds);
}

DynamicRTSPServer::DynamicRTSPServer(UsageEnvironment& env, int ourSocket,
				     Port ourPort,
				     UserAuthenticationDatabase* authDatabase, unsigned reclamationTestSeconds)
  : RTSPServerSupportingHTTPStreaming(env, ourSocket, ourPort, authDatabase, reclamationTestSeconds) {
}
```
在 `DynamicRTSPServer` 的 `createNew()` 中，首先创建一个 socket，然后用这个 socket 创建 `DynamicRTSPServer` 对象。构造函数顺着类继承层次一级级执行上去。

`RTSPServerSupportingHTTPStreaming` 构造函数定义如下：
```
RTSPServerSupportingHTTPStreaming
::RTSPServerSupportingHTTPStreaming(UsageEnvironment& env, int ourSocket, Port rtspPort,
				    UserAuthenticationDatabase* authDatabase, unsigned reclamationTestSeconds)
  : RTSPServer(env, ourSocket, rtspPort, authDatabase, reclamationTestSeconds) {
}
```

`RTSPServer` 构造函数定义如下：
```
RTSPServer::RTSPServer(UsageEnvironment& env,
		       int ourSocket, Port ourPort,
		       UserAuthenticationDatabase* authDatabase,
		       unsigned reclamationSeconds)
  : GenericMediaServer(env, ourSocket, ourPort, reclamationSeconds),
    fHTTPServerSocket(-1), fHTTPServerPort(0),
    fClientConnectionsForHTTPTunneling(NULL), // will get created if needed
    fTCPStreamingDatabase(HashTable::create(ONE_WORD_HASH_KEYS)),
    fPendingRegisterOrDeregisterRequests(HashTable::create(ONE_WORD_HASH_KEYS)),
    fRegisterOrDeregisterRequestCounter(0), fAuthDB(authDatabase), fAllowStreamingRTPOverTCP(True) {
}
```

`GenericMediaServer` 构造函数定义如下：
```
GenericMediaServer
::GenericMediaServer(UsageEnvironment& env, int ourSocket, Port ourPort,
		     unsigned reclamationSeconds)
  : Medium(env),
    fServerSocket(ourSocket), fServerPort(ourPort), fReclamationSeconds(reclamationSeconds),
    fServerMediaSessions(HashTable::create(STRING_HASH_KEYS)),
    fClientConnections(HashTable::create(ONE_WORD_HASH_KEYS)),
    fClientSessions(HashTable::create(STRING_HASH_KEYS)) {
  ignoreSigPipeOnSocket(fServerSocket); // so that clients on the same host that are killed don't also kill us
  
  // Arrange to handle connections from others:
  env.taskScheduler().turnOnBackgroundReadHandling(fServerSocket, incomingConnectionHandler, this);
}
```
**在 `GenericMediaServer` 的构造函数中，要监听的 Server socket 及该 socket 上的 I/O 事件处理程序，被注册给任务调度器。事件循环中检测到 socket 上出现 I/O 事件时，该处理程序会被调用到。注册的事件处理程序为 `GenericMediaServer::incomingConnectionHandler()` 。**

`Medium` 构造函数定义如下：
```
Medium::Medium(UsageEnvironment& env)
	: fEnviron(env), fNextTask(NULL) {
  // First generate a name for the new medium:
  MediaLookupTable::ourMedia(env)->generateNewName(fMediumName, mediumNameMaxLen);
  env.setResultMsg(fMediumName);

  // Then add it to our table:
  MediaLookupTable::ourMedia(env)->addNew(this, fMediumName);
}
```

在 `Medium` 中，主要通过 `MediaLookupTable` 维护一个 medium name 到 `Medium` 对象的映射表。`MediaLookupTable` 可以看作是 `BasicHashTable` 对值类型为 `Medium` 对象指针的特化，该类定义如下：
```
class MediaLookupTable {
public:
  static MediaLookupTable* ourMedia(UsageEnvironment& env);
  HashTable const& getTable() { return *fTable; }

protected:
  MediaLookupTable(UsageEnvironment& env);
  virtual ~MediaLookupTable();

private:
  friend class Medium;

  Medium* lookup(char const* name) const;
  // Returns NULL if none already exists

  void addNew(Medium* medium, char* mediumName);
  void remove(char const* name);

  void generateNewName(char* mediumName, unsigned maxLen);

private:
  UsageEnvironment& fEnv;
  HashTable* fTable;
  unsigned fNameGenerator;
};
```

`MediaLookupTable` 的引用实际由 `UsageEnvironment` 持有：
```
_Tables* _Tables::getOurTables(UsageEnvironment& env, Boolean createIfNotPresent) {
  if (env.liveMediaPriv == NULL && createIfNotPresent) {
    env.liveMediaPriv = new _Tables(env);
  }
  return (_Tables*)(env.liveMediaPriv);
}
. . . . . . 
MediaLookupTable* MediaLookupTable::ourMedia(UsageEnvironment& env) {
  _Tables* ourTables = _Tables::getOurTables(env);
  if (ourTables->mediaTable == NULL) {
    // Create a new table to record the media that are to be created in
    // this environment:
    ourTables->mediaTable = new MediaLookupTable(env);
  }
  return ourTables->mediaTable;
}
. . . . . . 
void MediaLookupTable::generateNewName(char* mediumName,
				       unsigned /*maxLen*/) {
  // We should really use snprintf() here, but not all systems have it
  sprintf(mediumName, "liveMedia%d", fNameGenerator++);
}

MediaLookupTable::MediaLookupTable(UsageEnvironment& env)
  : fEnv(env), fTable(HashTable::create(STRING_HASH_KEYS)), fNameGenerator(0) {
}
```

medium name 在 `Medium` 构造过程中分配，创建的 `Medium` 也在此时被加入 `MediaLookupTable` 中：
```
void MediaLookupTable::addNew(Medium* medium, char* mediumName) {
  fTable->Add(mediumName, (void*)medium);
}
. . . . . . 
void MediaLookupTable::generateNewName(char* mediumName,
				       unsigned /*maxLen*/) {
  // We should really use snprintf() here, but not all systems have it
  sprintf(mediumName, "liveMedia%d", fNameGenerator++);
}
```

# Server socket 的创建及连接建立
在 `DynamicRTSPServer` 的 `createNew()`，创建 `DynamicRTSPServer` 对象之前，会先通过 `setUpOurSocket()` 创建一个 socket，`setUpOurSocket()` 是 `GenericMediaServer` 的一个静态函数，其定义为：
```
int GenericMediaServer::setUpOurSocket(UsageEnvironment& env, Port& ourPort) {
  int ourSocket = -1;
  
  do {
    // The following statement is enabled by default.
    // Don't disable it (by defining ALLOW_SERVER_PORT_REUSE) unless you know what you're doing.
#if !defined(ALLOW_SERVER_PORT_REUSE) && !defined(ALLOW_RTSP_SERVER_PORT_REUSE)
    // ALLOW_RTSP_SERVER_PORT_REUSE is for backwards-compatibility #####
    NoReuse dummy(env); // Don't use this socket if there's already a local server using it
#endif
    
    ourSocket = setupStreamSocket(env, ourPort);
    if (ourSocket < 0) break;
    
    // Make sure we have a big send buffer:
    if (!increaseSendBufferTo(env, ourSocket, 50*1024)) break;
    
    // Allow multiple simultaneous connections:
    if (listen(ourSocket, LISTEN_BACKLOG_SIZE) < 0) {
      env.setResultErrMsg("listen() failed: ");
      break;
    }
    
    if (ourPort.num() == 0) {
      // bind() will have chosen a port for us; return it also:
      if (!getSourcePort(env, ourSocket, ourPort)) break;
    }
    
    return ourSocket;
  } while (0);
  
  if (ourSocket != -1) ::closeSocket(ourSocket);
  return -1;
}
```

`GenericMediaServer::setUpOurSocket()` 主要即是，通过 `setupStreamSocket()` 函数创建一个 TCP socket，增大该 socket 的发送缓冲区，并 listen 该 socket。

`setupStreamSocket()` 函数在 groupsock 模块中定义：
```
_groupsockPriv* groupsockPriv(UsageEnvironment& env) {
  if (env.groupsockPriv == NULL) { // We need to create it
    _groupsockPriv* result = new _groupsockPriv;
    result->socketTable = NULL;
    result->reuseFlag = 1; // default value => allow reuse of socket numbers
    env.groupsockPriv = result;
  }
  return (_groupsockPriv*)(env.groupsockPriv);
}

void reclaimGroupsockPriv(UsageEnvironment& env) {
  _groupsockPriv* priv = (_groupsockPriv*)(env.groupsockPriv);
  if (priv->socketTable == NULL && priv->reuseFlag == 1/*default value*/) {
    // We can delete the structure (to save space); it will get created again, if needed:
    delete priv;
    env.groupsockPriv = NULL;
  }
}

static int createSocket(int type) {
  // Call "socket()" to create a (IPv4) socket of the specified type.
  // But also set it to have the 'close on exec' property (if we can)
  int sock;

#ifdef SOCK_CLOEXEC
  sock = socket(AF_INET, type|SOCK_CLOEXEC, 0);
  if (sock != -1 || errno != EINVAL) return sock;
  // An "errno" of EINVAL likely means that the system wasn't happy with the SOCK_CLOEXEC; fall through and try again without it:
#endif

  sock = socket(AF_INET, type, 0);
#ifdef FD_CLOEXEC
  if (sock != -1) fcntl(sock, F_SETFD, FD_CLOEXEC);
#endif
  return sock;
}
. . . . . .
int setupStreamSocket(UsageEnvironment& env,
                      Port port, Boolean makeNonBlocking) {
  if (!initializeWinsockIfNecessary()) {
    socketErr(env, "Failed to initialize 'winsock': ");
    return -1;
  }

  int newSocket = createSocket(SOCK_STREAM);
  if (newSocket < 0) {
    socketErr(env, "unable to create stream socket: ");
    return newSocket;
  }

  int reuseFlag = groupsockPriv(env)->reuseFlag;
  reclaimGroupsockPriv(env);
  if (setsockopt(newSocket, SOL_SOCKET, SO_REUSEADDR,
		 (const char*)&reuseFlag, sizeof reuseFlag) < 0) {
    socketErr(env, "setsockopt(SO_REUSEADDR) error: ");
    closeSocket(newSocket);
    return -1;
  }

  // SO_REUSEPORT doesn't really make sense for TCP sockets, so we
  // normally don't set them.  However, if you really want to do this
  // #define REUSE_FOR_TCP
#ifdef REUSE_FOR_TCP
#if defined(__WIN32__) || defined(_WIN32)
  // Windoze doesn't properly handle SO_REUSEPORT
#else
#ifdef SO_REUSEPORT
  if (setsockopt(newSocket, SOL_SOCKET, SO_REUSEPORT,
		 (const char*)&reuseFlag, sizeof reuseFlag) < 0) {
    socketErr(env, "setsockopt(SO_REUSEPORT) error: ");
    closeSocket(newSocket);
    return -1;
  }
#endif
#endif
#endif

  // Note: Windoze requires binding, even if the port number is 0
#if defined(__WIN32__) || defined(_WIN32)
#else
  if (port.num() != 0 || ReceivingInterfaceAddr != INADDR_ANY) {
#endif
    MAKE_SOCKADDR_IN(name, ReceivingInterfaceAddr, port.num());
    if (bind(newSocket, (struct sockaddr*)&name, sizeof name) != 0) {
      char tmpBuffer[100];
      sprintf(tmpBuffer, "bind() error (port number: %d): ",
	      ntohs(port.num()));
      socketErr(env, tmpBuffer);
      closeSocket(newSocket);
      return -1;
    }
#if defined(__WIN32__) || defined(_WIN32)
#else
  }
#endif

  if (makeNonBlocking) {
    if (!makeSocketNonBlocking(newSocket)) {
      socketErr(env, "failed to make non-blocking: ");
      closeSocket(newSocket);
      return -1;
    }
  }

  return newSocket;
}
```
在这里基本上就是创建 TCP socket，为 socket 设置 RESUE 选项，将 socket 绑定到目标端口，并根据需要设置 socket 为非阻塞的。

接着来看，在 live555 中是如何启动监听 server socket 上的 I/O 事件，并处理这些事件的。

我们看到，在 `GenericMediaServer` 的构造函数中，用于接收客户端发起的连接的 server socket，及该 socket 上的 I/O 事件处理程序，被注册给任务调度器。当有客户端发起了到 "LIVE555 Media Server" 的连接时，该处理程序会被调用到。事件处理程序为 `GenericMediaServer::incomingConnectionHandler()` ，相关的几个函数声明为：
```
class GenericMediaServer: public Medium {
. . . . . .
protected:
. . . . . .
  static void incomingConnectionHandler(void*, int /*mask*/);
  void incomingConnectionHandler();
  void incomingConnectionHandlerOnSocket(int serverSocket);
. . . . . .
protected:
  virtual ClientConnection* createNewClientConnection(int clientSocket, struct sockaddr_in clientAddr) = 0;
```

`GenericMediaServer::incomingConnectionHandler()`为静态函数，`incomingConnectionHandler()` 及 `incomingConnectionHandlerOnSocket(int serverSocket)` 为非虚函数，它们的定义为：
```
void GenericMediaServer::incomingConnectionHandler(void* instance, int /*mask*/) {
  GenericMediaServer* server = (GenericMediaServer*)instance;
  server->incomingConnectionHandler();
}
void GenericMediaServer::incomingConnectionHandler() {
  incomingConnectionHandlerOnSocket(fServerSocket);
}

void GenericMediaServer::incomingConnectionHandlerOnSocket(int serverSocket) {
  struct sockaddr_in clientAddr;
  SOCKLEN_T clientAddrLen = sizeof clientAddr;
  int clientSocket = accept(serverSocket, (struct sockaddr*)&clientAddr, &clientAddrLen);
  if (clientSocket < 0) {
    int err = envir().getErrno();
    if (err != EWOULDBLOCK) {
      envir().setResultErrMsg("accept() failed: ");
    }
    return;
  }
  ignoreSigPipeOnSocket(clientSocket); // so that clients on the same host that are killed don't also kill us
  makeSocketNonBlocking(clientSocket);
  increaseSendBufferTo(envir(), clientSocket, 50*1024);
  
#ifdef DEBUG
  envir() << "accept()ed connection from " << AddressString(clientAddr).val() << "\n";
#endif
  
  // Create a new object for handling this connection:
  (void)createNewClientConnection(clientSocket, clientAddr);
}
```
可以看到，当监听的 server socket 上发现有客户端发起的连接请求时，处理过程大体为：
1. 通过 `accept()` 获得新建立的连接的 socket。
2. 设置新 socket 的选项，使它称为非阻塞 socket，并增大该 socket 的发送缓冲区。可以看一下在 live555 中设置 socket 选项的这些函数，它们在 groupsock 模块中定义：
```
Boolean makeSocketNonBlocking(int sock) {
. . . . . .
  int curFlags = fcntl(sock, F_GETFL, 0);
  return fcntl(sock, F_SETFL, curFlags|O_NONBLOCK) >= 0;
#endif
}
. . . . . .
void ignoreSigPipeOnSocket(int socketNum) {
. . . . . .
  signal(SIGPIPE, SIG_IGN);
  #endif
  #endif
}

static unsigned getBufferSize(UsageEnvironment& env, int bufOptName,
			      int socket) {
  unsigned curSize;
  SOCKLEN_T sizeSize = sizeof curSize;
  if (getsockopt(socket, SOL_SOCKET, bufOptName,
		 (char*)&curSize, &sizeSize) < 0) {
    socketErr(env, "getBufferSize() error: ");
    return 0;
  }

  return curSize;
}
. . . . . .
static unsigned increaseBufferTo(UsageEnvironment& env, int bufOptName,
				 int socket, unsigned requestedSize) {
  // First, get the current buffer size.  If it's already at least
  // as big as what we're requesting, do nothing.
  unsigned curSize = getBufferSize(env, bufOptName, socket);

  // Next, try to increase the buffer to the requested size,
  // or to some smaller size, if that's not possible:
  while (requestedSize > curSize) {
    SOCKLEN_T sizeSize = sizeof requestedSize;
    if (setsockopt(socket, SOL_SOCKET, bufOptName,
		   (char*)&requestedSize, sizeSize) >= 0) {
      // success
      return requestedSize;
    }
    requestedSize = (requestedSize+curSize)/2;
  }

  return getBufferSize(env, bufOptName, socket);
}
unsigned increaseSendBufferTo(UsageEnvironment& env,
			      int socket, unsigned requestedSize) {
  return increaseBufferTo(env, SO_SNDBUF, socket, requestedSize);
}
```
3. 通过 `createNewClientConnection()`，创建一个新的客户端连接 `ClientConnection`。`createNewClientConnection()` 是一个纯虚函数，它的实现将在类继承层次中实现了该函数的类中层次最低的那个类里，对于
 `DynamicRTSPServer` -> `RTSPServerSupportingHTTPStreaming` -> `RTSPServer` -> `GenericMediaServer` 这个继承层次，从 `DynamicRTSPServer` 类开始，逐级向上找，可以发现 `createNewClientConnection()` 函数的实现在 `RTSPServerSupportingHTTPStreaming` 类中。

`RTSPServerSupportingHTTPStreaming` 类的
 `createNewClientConnection()` 函数定义如下：
```
GenericMediaServer::ClientConnection*
RTSPServerSupportingHTTPStreaming::createNewClientConnection(int clientSocket, struct sockaddr_in clientAddr) {
  return new RTSPClientConnectionSupportingHTTPStreaming(*this, clientSocket, clientAddr);
}

RTSPServerSupportingHTTPStreaming::RTSPClientConnectionSupportingHTTPStreaming
::RTSPClientConnectionSupportingHTTPStreaming(RTSPServer& ourServer, int clientSocket, struct sockaddr_in clientAddr)
  : RTSPClientConnection(ourServer, clientSocket, clientAddr),
    fClientSessionId(0), fStreamSource(NULL), fPlaylistSource(NULL), fTCPSink(NULL) {
}
```

这里简单地创建一个 `RTSPClientConnectionSupportingHTTPStreaming` 类对象。在 live555 中，用 `GenericMediaServer` 的内部类 `GenericMediaServer::ClientConnection` 表示一个客户端连接。对于  "LIVE555 Media Server" 而言，这个类的继承层次体系结构如下图所示：

![](https://www.wolfcstech.com/images/1315506-25767662578513fe.png)

`RTSPClientConnectionSupportingHTTPStreaming` 类对象构造函数调用其父类 `RTSPServer::RTSPClientConnection` 的构造函数，后者的实现为：
```
RTSPServer::RTSPClientConnection
::RTSPClientConnection(RTSPServer& ourServer, int clientSocket, struct sockaddr_in clientAddr)
  : GenericMediaServer::ClientConnection(ourServer, clientSocket, clientAddr),
    fOurRTSPServer(ourServer), fClientInputSocket(fOurSocket), fClientOutputSocket(fOurSocket),
    fIsActive(True), fRecursionCount(0), fOurSessionCookie(NULL) {
  resetRequestBuffer();
}
```

`RTSPServer::RTSPClientConnection` 的构造函数继续调用其父类 `GenericMediaServer::ClientConnection` 的构造函数：
```
GenericMediaServer::ClientConnection
::ClientConnection(GenericMediaServer& ourServer, int clientSocket, struct sockaddr_in clientAddr)
  : fOurServer(ourServer), fOurSocket(clientSocket), fClientAddr(clientAddr) {
  // Add ourself to our 'client connections' table:
  fOurServer.fClientConnections->Add((char const*)this, this);
  
  // Arrange to handle incoming requests:
  resetRequestBuffer();
  envir().taskScheduler()
    .setBackgroundHandling(fOurSocket, SOCKET_READABLE|SOCKET_EXCEPTION, incomingRequestHandler, this);
}
```

在 live555 中，`GenericMediaServer` 用于执行通用的媒体服务器相关的管理，包括管理所有的客户端连接。`GenericMediaServer::ClientConnection` 的构造函数中会将其自身加入 `GenericMediaServer` 的哈希表 `fClientConnections` 中。更重要的是，将客户端连接的 socket 及该 socket 上的 I/O 事件处理程序注册给了任务调度器，其中 I/O 事件处理程序为 `GenericMediaServer::ClientConnection::incomingRequestHandler()` 函数。

也就是说，"LIVE555 Media Server" 中与客户端的交互将由 `GenericMediaServer::ClientConnection::incomingRequestHandler()` 函数完成，这个函数定义如下：
```
void GenericMediaServer::ClientConnection::incomingRequestHandler(void* instance, int /*mask*/) {
  ClientConnection* connection = (ClientConnection*)instance;
  connection->incomingRequestHandler();
}

void GenericMediaServer::ClientConnection::incomingRequestHandler() {
  struct sockaddr_in dummy; // 'from' address, meaningless in this case
  
  int bytesRead = readSocket(envir(), fOurSocket, &fRequestBuffer[fRequestBytesAlreadySeen], fRequestBufferBytesLeft, dummy);
  handleRequestBytes(bytesRead);
}
```
在 `GenericMediaServer::ClientConnection` 的 `incomingRequestHandler() ` 中，会从客户端连接的 socket 中读取数据，然后调用 `handleRequestBytes()` 函数做进一步的处理。

读取数据的动作由 groupsock 模块中的 `readSocket()` 完成：
```
int readSocket(UsageEnvironment& env,
	       int socket, unsigned char* buffer, unsigned bufferSize,
	       struct sockaddr_in& fromAddress) {
  SOCKLEN_T addressSize = sizeof fromAddress;
  int bytesRead = recvfrom(socket, (char*)buffer, bufferSize, 0,
			   (struct sockaddr*)&fromAddress,
			   &addressSize);
  if (bytesRead < 0) {
    //##### HACK to work around bugs in Linux and Windows:
    int err = env.getErrno();
    if (err == 111 /*ECONNREFUSED (Linux)*/
. . . . . .
	|| err == EAGAIN
#endif
	|| err == 113 /*EHOSTUNREACH (Linux)*/) { // Why does Linux return this for datagram sock?
      fromAddress.sin_addr.s_addr = 0;
      return 0;
    }
    //##### END HACK
    socketErr(env, "recvfrom() error: ");
  } else if (bytesRead == 0) {
    // "recvfrom()" on a stream socket can return 0 if the remote end has closed the connection.  Treat this as an error:
    return -1;
  }

  return bytesRead;
}
```
这里通过 `recvfrom()` 函数从 socket 中读取数据。

 `GenericMediaServer::ClientConnection` 中，处理客户端请求的这几个函数声明如下：
```
    static void incomingRequestHandler(void*, int /*mask*/);
    void incomingRequestHandler();
    virtual void handleRequestBytes(int newBytesRead) = 0;
```

`handleRequestBytes()` 为纯虚函数，需要由子类实现。对于 `RTSPServerSupportingHTTPStreaming::RTSPClientConnectionSupportingHTTPStreaming` -> `RTSPServer::RTSPClientConnection` -> `GenericMediaServer::ClientConnection` 这个继承层次体系，该函数的实现实际上位于 `RTSPServer::RTSPClientConnection` 中。

从客户端接收到的请求，将由 `RTSPServer::RTSPClientConnection::handleRequestBytes()` 处理。

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
