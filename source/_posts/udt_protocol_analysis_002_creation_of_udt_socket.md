---
title: UDT协议实现分析——UDT Socket的创建
date: 2015-09-07 16:05:49
categories: 网络协议
tags:
- 源码分析
- 网络协议
- UDT
---

# UDT API的用法
在分析 连接的建立过程 之前，先来看一下UDT API的用法。在UDT网络中，通常要有一个UDT Server监听在某台机器的某个UDP端口上，等待客户端的连接；有一个或多个客户端连接UDT Server；UDT Server接收到来自客户端的连接请求后，创建另外一个单独的UDT Socket用于与该客户端进行通信。
<!--more-->
先来看一下UDT Server的简单的实现，UDT的开发者已经提供了一些demo程序可供参考，位于app/目录下。
```
#include <unistd.h>
#include <cstdlib>
#include <cstring>
#include <netdb.h>

#include <iostream>
#include <udt.h>

using namespace std;

void* recvdata(void*);

struct UDTUpDown {
    UDTUpDown() {
        // use this function to initialize the UDT library
        UDT::startup();
    }
    ~UDTUpDown() {
        // use this function to release the UDT library
        UDT::cleanup();
    }
};

int main(int argc, char* argv[]) {
    if ((1 != argc) && ((2 != argc) || (0 == atoi(argv[1])))) {
        cout << "usage: appserver [server_port]" << endl;
        return 0;
    }

    // Automatically start up and clean up UDT module.
    UDTUpDown _udt_;

    addrinfo hints;
    addrinfo* res;

    memset(&hints, 0, sizeof(struct addrinfo));

    hints.ai_flags = AI_PASSIVE;
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    //hints.ai_socktype = SOCK_DGRAM;

    string service("9000");
    if (2 == argc)
        service = argv[1];

    if (0 != getaddrinfo(NULL, service.c_str(), &hints, &res)) {
        cout << "illegal port number or port is busy.\n" << endl;
        return 0;
    }

    UDTSOCKET serv = UDT::socket(res->ai_family, res->ai_socktype, res->ai_protocol);

    // UDT Options
    //UDT::setsockopt(serv, 0, UDT_CC, new CCCFactory<CUDPBlast>, sizeof(CCCFactory<CUDPBlast>));
    //UDT::setsockopt(serv, 0, UDT_MSS, new int(9000), sizeof(int));
    //UDT::setsockopt(serv, 0, UDT_RCVBUF, new int(10000000), sizeof(int));
    //UDT::setsockopt(serv, 0, UDP_RCVBUF, new int(10000000), sizeof(int));

    if (UDT::ERROR == UDT::bind(serv, res->ai_addr, res->ai_addrlen)) {
        cout << "bind: " << UDT::getlasterror().getErrorMessage() << endl;
        return 0;
    }

    freeaddrinfo(res);

    cout << "server is ready at port: " << service << endl;

    if (UDT::ERROR == UDT::listen(serv, 10)) {
        cout << "listen: " << UDT::getlasterror().getErrorMessage() << endl;
        return 0;
    }

    sockaddr_storage clientaddr;
    int addrlen = sizeof(clientaddr);

    UDTSOCKET recver;

    while (true) {
        if (UDT::INVALID_SOCK == (recver = UDT::accept(serv, (sockaddr*) &clientaddr, &addrlen))) {
            cout << "accept: " << UDT::getlasterror().getErrorMessage() << endl;
            return 0;
        }

        char clienthost[NI_MAXHOST];
        char clientservice[NI_MAXSERV];
        getnameinfo((sockaddr *) &clientaddr, addrlen, clienthost, sizeof(clienthost), clientservice,
                    sizeof(clientservice), NI_NUMERICHOST | NI_NUMERICSERV);
        cout << "new connection: " << clienthost << ":" << clientservice << endl;

        pthread_t rcvthread;
        pthread_create(&rcvthread, NULL, recvdata, new UDTSOCKET(recver));
        pthread_detach(rcvthread);
    }

    UDT::close(serv);

    return 0;
}

void* recvdata(void* usocket) {
    UDTSOCKET recver = *(UDTSOCKET*) usocket;
    delete (UDTSOCKET*) usocket;

    char* data;
    int size = 100000;
    data = new char[size];

    while (true) {
        int rsize = 0;
        int rs;
        while (rsize < size) {
            int rcv_size;
            int var_size = sizeof(int);
            UDT::getsockopt(recver, 0, UDT_RCVDATA, &rcv_size, &var_size);
            if (UDT::ERROR == (rs = UDT::recv(recver, data + rsize, size - rsize, 0))) {
                cout << "recv:" << UDT::getlasterror().getErrorMessage() << endl;
                break;
            }

            rsize += rs;
        }

        if (rsize < size)
            break;
    }

    delete[] data;

    UDT::close(recver);

    return NULL;
}
```
1. 如在  [UDT协议实现分析——UDT初始化和销毁](http://my.oschina.net/wolfcs/blog/501896)  一文中提到的， 在调用任何UDT API之前，需要首先调用UDT::startup()来对库做一个初始化，并在结束之后执行UDT::cleanup()做最后的清理。在这个示例中，是创建了一个helper类UDTUpDown来帮助做这些事情的。利用编程语言本身提供的构造-析够机制来对UDT进行初始化和销毁要比手动调用这些函数要可靠得多。
2. 获得本地UDP端口的网络地址。
3. 调用UDT::socket()函数创建一个UDT Socket。这个Socket是UDT抽象出来的一个逻辑的Socket，我们并不能利用这个Socket本身来收发数据。
4. 调用UDT::bind()将创建的UDT Socket绑定到本地端口的网络地址。这一步将会把UDT的逻辑Socket与能够进行数据收发的系统UDP socket进行关联，自此我们就可以利用UDT Socket进行收发数据了。
5. 调用UDT::listen()告诉UDT，把这个UDT Socket做为这个端口上的Listening Socket。一个UDP端口可以被多个UDT Socket复用，在调用UDT::listen()之后，UDT就知道，在有其它节点连接这个UDP端口时，需要把相关的连接请求消息发送到哪个UDT Socket的接收缓冲区了。
对绑定到相同UDP端口的多个不同UDT Socket调用UDT::listen()时，UDT是如何处理的？我们知道，一个UDP端口上最多只能有一个listening Socket。
6. 调用UDT::accept()函数等待其它节点的连接。其它节点连接时，这个函数返回另外一个单独的UDT Socket，以用于与发起连接的节点进行通信。
UDT::accept()函数返回的UDT Socket会被绑定到另外的一个不同的UDP端口，还是会被绑定到listening Socket所绑定的UDP端口？
7. 使用UDT::accept()返回的UDT Socket，利用UDT::recv()与UDT::send()等函数同发起连接的节点进行数据传输。
8. 在不再需要UDT Socket时，调用UDT::close()函数关掉它，以释放资源。
然后再来看下UDT Client的实现：

```
#include <unistd.h>
#include <cstdlib>
#include <cstring>
#include <netdb.h>

#include <iostream>
#include <udt.h>

using namespace std;

void* monitor(void*);

struct UDTUpDown {
    UDTUpDown() {
        // use this function to initialize the UDT library
        UDT::startup();
    }
    ~UDTUpDown() {
        // use this function to release the UDT library
        UDT::cleanup();
    }
};

int main(int argc, char* argv[]) {
    if ((3 != argc) || (0 == atoi(argv[2]))) {
        cout << "usage: appclient server_ip server_port" << endl;
        return 0;
    }

    // Automatically start up and clean up UDT module.
    UDTUpDown _udt_;

    struct addrinfo hints, *local, *peer;

    memset(&hints, 0, sizeof(struct addrinfo));

    hints.ai_flags = AI_PASSIVE;
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    //hints.ai_socktype = SOCK_DGRAM;

    if (0 != getaddrinfo(NULL, "9000", &hints, &local)) {
        cout << "incorrect network address.\n" << endl;
        return 0;
    }

    UDTSOCKET client = UDT::socket(local->ai_family, local->ai_socktype, local->ai_protocol);

    // UDT Options
    //UDT::setsockopt(client, 0, UDT_CC, new CCCFactory<CUDPBlast>, sizeof(CCCFactory<CUDPBlast>));
    //UDT::setsockopt(client, 0, UDT_MSS, new int(9000), sizeof(int));
    //UDT::setsockopt(client, 0, UDT_SNDBUF, new int(10000000), sizeof(int));
    //UDT::setsockopt(client, 0, UDP_SNDBUF, new int(10000000), sizeof(int));
    //UDT::setsockopt(client, 0, UDT_MAXBW, new int64_t(12500000), sizeof(int));

    // for rendezvous connection, enable the code below
    /*
     UDT::setsockopt(client, 0, UDT_RENDEZVOUS, new bool(true), sizeof(bool));
     if (UDT::ERROR == UDT::bind(client, local->ai_addr, local->ai_addrlen))
     {
     cout << "bind: " << UDT::getlasterror().getErrorMessage() << endl;
     return 0;
     }
     */

    freeaddrinfo(local);

    if (0 != getaddrinfo(argv[1], argv[2], &hints, &peer)) {
        cout << "incorrect server/peer address. " << argv[1] << ":" << argv[2] << endl;
        return 0;
    }

    // connect to the server, implict bind
    if (UDT::ERROR == UDT::connect(client, peer->ai_addr, peer->ai_addrlen)) {
        cout << "connect: " << UDT::getlasterror().getErrorMessage() << endl;
        return 0;
    }

    freeaddrinfo(peer);

    int size = 100000;
    char* data = new char[size];

    pthread_create(new pthread_t, NULL, monitor, &client);

    for (int i = 0; i < 1000000; i++) {
        int ssize = 0;
        int ss;
        while (ssize < size) {
            if (UDT::ERROR == (ss = UDT::send(client, data + ssize, size - ssize, 0))) {
                cout << "send:" << UDT::getlasterror().getErrorMessage() << endl;
                break;
            }

            ssize += ss;
        }

        if (ssize < size)
            break;
    }

    UDT::close(client);
    delete[] data;
    return 0;
}

void* monitor(void* s) {
    UDTSOCKET u = *(UDTSOCKET*) s;
    UDT::TRACEINFO perf;
    cout << "SendRate(Mb/s)\tRTT(ms)\tCWnd\tPktSndPeriod(us)\tRecvACK\tRecvNAK" << endl;

    while (true) {
        sleep(1);
        if (UDT::ERROR == UDT::perfmon(u, &perf)) {
            cout << "perfmon: " << UDT::getlasterror().getErrorMessage() << endl;
            break;
        }

        cout << perf.mbpsSendRate << "\t\t" << perf.msRTT << "\t" << perf.pktCongestionWindow << "\t"
             << perf.usPktSndPeriod << "\t\t\t" << perf.pktRecvACK << "\t" << perf.pktRecvNAK << endl;
    }

    return NULL;
}
```
可以看到UDT Client连接UDT Server并发送数据的过程大体如下：

1. 同样需要在调用任何UDT API之前，先调用UDT::startup()来对库做初始化，并在结束之后执行UDT::cleanup()做最后的清理。这里同样使用helper类UDTUpDown来帮助做这些事情。

2. 获得UDT Server的网络地址。

3. 调用UDT::socket()函数创建一个UDT Socket。这个Socket可以与特定的本地地址绑定，也可以不绑定。如果绑定，则发送数据时的出口地址就是该端口，如果不绑定，出口地址则是一个不确定的值。

4. 调用UDT::connect()连接UDT Server。

5. 使用UDT Socket，利用UDT::recv()与UDT::send()等函数同UDT Server进行数据传输。

6. 在不再需要UDT Socket时，调用UDT::close()函数关掉它，以释放资源。

UDT基本的收发数据的API的用法大体如上面所示。接着我们来看，这些函数是如何实现的。

# UDT Socket的创建
无论是UDT Server要listening，还是UDT client要连接UDT Server，在调用UDT::startup()初始化UDT之后，首先要做的事情都是调用UDT::socket()创建UDT Socket了。我们就来看一下创建UDT Socket的过程：
```
UDTSOCKET CUDTUnited::newSocket(int af, int type) {
    if ((type != SOCK_STREAM) && (type != SOCK_DGRAM))
        throw CUDTException(5, 3, 0);

    CUDTSocket* ns = NULL;

    try {
        ns = new CUDTSocket;
        ns->m_pUDT = new CUDT;
        if (AF_INET == af) {
            ns->m_pSelfAddr = (sockaddr*) (new sockaddr_in);
            ((sockaddr_in*) (ns->m_pSelfAddr))->sin_port = 0;
        } else {
            ns->m_pSelfAddr = (sockaddr*) (new sockaddr_in6);
            ((sockaddr_in6*) (ns->m_pSelfAddr))->sin6_port = 0;
        }
    } catch (...) {
        delete ns;
        throw CUDTException(3, 2, 0);
    }

    CGuard::enterCS(m_IDLock);
    ns->m_SocketID = --m_SocketID;
    CGuard::leaveCS(m_IDLock);

    ns->m_Status = INIT;
    ns->m_ListenSocket = 0;
    ns->m_pUDT->m_SocketID = ns->m_SocketID;
    ns->m_pUDT->m_iSockType = (SOCK_STREAM == type) ? UDT_STREAM : UDT_DGRAM;
    ns->m_pUDT->m_iIPversion = ns->m_iIPversion = af;
    ns->m_pUDT->m_pCache = m_pCache;

    // protect the m_Sockets structure.
    CGuard::enterCS(m_ControlLock);
    try {
        m_Sockets[ns->m_SocketID] = ns;
    } catch (...) {
        //failure and rollback
        CGuard::leaveCS(m_ControlLock);
        delete ns;
        ns = NULL;
    }
    CGuard::leaveCS(m_ControlLock);

    if (NULL == ns)
        throw CUDTException(3, 2, 0);

    return ns->m_SocketID;
}


UDTSOCKET CUDT::socket(int af, int type, int) {
    if (!s_UDTUnited.m_bGCStatus)
        s_UDTUnited.startup();

    try {
        return s_UDTUnited.newSocket(af, type);
    } catch (CUDTException& e) {
        s_UDTUnited.setError(new CUDTException(e));
        return INVALID_SOCK;
    } catch (bad_alloc&) {
        s_UDTUnited.setError(new CUDTException(3, 2, 0));
        return INVALID_SOCK;
    } catch (...) {
        s_UDTUnited.setError(new CUDTException(-1, 0, 0));
        return INVALID_SOCK;
    }
}



UDTSOCKET socket(int af, int type, int protocol) {
    return CUDT::socket(af, type, protocol);
}
```
调用流程为，UDT::socket() -> CUDT::socket() -> CUDTUnited::newSocket()。

在CUDT::socket()中，我们看到，它会首先检查s_UDTUnited.m_bGCStatus，若发现UDT还没有初始化完成的话，则会调用s_UDTUnited.startup()进行初始化。这个地方对UDT状态s_UDTUnited.m_bGCStatus的检查没有问题，但在发现UDT没有初始化完成时调用s_UDTUnited.startup()似乎并不恰当。这个地方调用了s_UDTUnited.startup()，那与这次调用相对应的cleanup()又在哪调用了呢？显然在UDT内部是没有。若是调用cleanup()的职责总是在UDT的使用者，那倒不如在这个地方返回错误给调用者，完全让调用者来管理UDT的生命周期，以尽可能地避免资源泄漏。

创建UDT Socket的工作实际都在CUDTUnited::newSocket()函数中完成。可以看到，在这个函数中主要做了如下的这样一些事情：

1. 创建一个CUDTSocket对象ns，并创建一个CUDT对象被ns->m_pUDT引用。初始化ns对象的Self网络地址，端口会被设置为0。在UDT中使用CUDTSocket和CUDT共同来描述一个Socket。每个UDT Socket都会有其相应的CUDTSocket对象和CUDT对象。

可以看一下CUDTSocket类的定义，来了解它都描述了UDT Socket的哪些属性(src/api.h)：
```
class CUDTSocket {
 public:
    CUDTSocket();
    ~CUDTSocket();

    UDTSTATUS m_Status;                       // current socket state

    uint64_t m_TimeStamp;                     // time when the socket is closed

    int m_iIPversion;                         // IP version
    sockaddr* m_pSelfAddr;                    // pointer to the local address of the socket
    sockaddr* m_pPeerAddr;                    // pointer to the peer address of the socket

    UDTSOCKET m_SocketID;                     // socket ID
    UDTSOCKET m_ListenSocket;                 // ID of the listener socket; 0 means this is an independent socket

    UDTSOCKET m_PeerID;                       // peer socket ID
    int32_t m_iISN;                      // initial sequence number, used to tell different connection from same IP:port

    CUDT* m_pUDT;                             // pointer to the UDT entity

    std::set<UDTSOCKET>* m_pQueuedSockets;    // set of connections waiting for accept()
    std::set<UDTSOCKET>* m_pAcceptSockets;    // set of accept()ed connections

    pthread_cond_t m_AcceptCond;              // used to block "accept" call
    pthread_mutex_t m_AcceptLock;             // mutex associated to m_AcceptCond

    unsigned int m_uiBackLog;                 // maximum number of connections in queue

    int m_iMuxID;                             // multiplexer ID

    pthread_mutex_t m_ControlLock;            // lock this socket exclusively for control APIs: bind/listen/connect

 private:
    CUDTSocket(const CUDTSocket&);
    CUDTSocket& operator=(const CUDTSocket&);
};
```
这个类只提供了构造和析构两个成员函数。还声明了私有的copy构造函数和赋值操作符函数，但没有定义它们，以避免类对象的复制。

可以再来看一下CUDTSocket的构造函数实现(src/api.h)：
```
CUDTSocket::CUDTSocket()
        : m_Status(INIT),
          m_TimeStamp(0),
          m_iIPversion(0),
          m_pSelfAddr(NULL),
          m_pPeerAddr(NULL),
          m_SocketID(0),
          m_ListenSocket(0),
          m_PeerID(0),
          m_iISN(0),
          m_pUDT(NULL),
          m_pQueuedSockets(NULL),
          m_pAcceptSockets(NULL),
          m_AcceptCond(),
          m_AcceptLock(),
          m_uiBackLog(0),
          m_iMuxID(-1) {
#ifndef WIN32
    pthread_mutex_init(&m_AcceptLock, NULL);
    pthread_cond_init(&m_AcceptCond, NULL);
    pthread_mutex_init(&m_ControlLock, NULL);
#else
    m_AcceptLock = CreateMutex(NULL, false, NULL);
    m_AcceptCond = CreateEvent(NULL, false, false, NULL);
    m_ControlLock = CreateMutex(NULL, false, NULL);
#endif
}
```

特别注意m_Status的初始化，该值被初始化为了INIT。从状态机的角度来看CUDTSocket，在它刚被new出来时，它处于INIT状态。

CUDT类有两个主要的职责，一是描述UDT Socket，包括所有的非静态成员变量和非静态成员函数，定义UDT Socket的大部分属性和所能提供操作；二是提供API，包括绝大部分的static成员函数，这些函数将调用者与UDT内部的实现连接起来。CUDT类这样的设计，明显违背了OO的SRP单一职责原则，这多少还是给代码的阅读带来了一定的障碍。再来看一下CUDT的构造函数实现(src/core.cpp)：
```
CUDT::CUDT() {
    m_pSndBuffer = NULL;
    m_pRcvBuffer = NULL;
    m_pSndLossList = NULL;
    m_pRcvLossList = NULL;
    m_pACKWindow = NULL;
    m_pSndTimeWindow = NULL;
    m_pRcvTimeWindow = NULL;

    m_pSndQueue = NULL;
    m_pRcvQueue = NULL;
    m_pPeerAddr = NULL;
    m_pSNode = NULL;
    m_pRNode = NULL;

    // Initilize mutex and condition variables
    initSynch();

    // Default UDT configurations
    m_iMSS = 1500;
    m_bSynSending = true;
    m_bSynRecving = true;
    m_iFlightFlagSize = 25600;
    m_iSndBufSize = 8192;
    m_iRcvBufSize = 8192;  //Rcv buffer MUST NOT be bigger than Flight Flag size
    m_Linger.l_onoff = 1;
    m_Linger.l_linger = 180;
    m_iUDPSndBufSize = 65536;
    m_iUDPRcvBufSize = m_iRcvBufSize * m_iMSS;
    m_iSockType = UDT_STREAM;
    m_iIPversion = AF_INET;
    m_bRendezvous = false;
    m_iSndTimeOut = -1;
    m_iRcvTimeOut = -1;
    m_bReuseAddr = true;
    m_llMaxBW = -1;

    m_pCCFactory = new CCCFactory<CUDTCC>;
    m_pCC = NULL;
    m_pCache = NULL;

    // Initial status
    m_bOpened = false;
    m_bListening = false;
    m_bConnecting = false;
    m_bConnected = false;
    m_bClosing = false;
    m_bShutdown = false;
    m_bBroken = false;
    m_bPeerHealth = true;
    m_ullLingerExpiration = 0;
}

void CUDT::initSynch() {
#ifndef WIN32
    pthread_mutex_init(&m_SendBlockLock, NULL);
    pthread_cond_init(&m_SendBlockCond, NULL);
    pthread_mutex_init(&m_RecvDataLock, NULL);
    pthread_cond_init(&m_RecvDataCond, NULL);
    pthread_mutex_init(&m_SendLock, NULL);
    pthread_mutex_init(&m_RecvLock, NULL);
    pthread_mutex_init(&m_AckLock, NULL);
    pthread_mutex_init(&m_ConnectionLock, NULL);
#else
    m_SendBlockLock = CreateMutex(NULL, false, NULL);
    m_SendBlockCond = CreateEvent(NULL, false, false, NULL);
    m_RecvDataLock = CreateMutex(NULL, false, NULL);
    m_RecvDataCond = CreateEvent(NULL, false, false, NULL);
    m_SendLock = CreateMutex(NULL, false, NULL);
    m_RecvLock = CreateMutex(NULL, false, NULL);
    m_AckLock = CreateMutex(NULL, false, NULL);
    m_ConnectionLock = CreateMutex(NULL, false, NULL);
#endif
}
```
都是成员变量的初始化，后续再来详细了解这些成员变量的作用。
2. 为Socket分配SocketID，其值为CUDTUnited的m_SocketID递减的结果。m_SocketID在CUDTUnited的构造函数中初始化：

```
CUDTUnited::CUDTUnited()
        : m_Sockets(),
          m_ControlLock(),
          m_IDLock(),
          m_SocketID(0),
          m_TLSError(),
          m_mMultiplexer(),
          m_MultiplexerLock(),
          m_pCache(NULL),
          m_bClosing(false),
          m_GCStopLock(),
          m_GCStopCond(),
          m_InitLock(),
          m_iInstanceCount(0),
          m_bGCStatus(false),
          m_GCThread(),
          m_ClosedSockets() {
    // Socket ID MUST start from a random value
    srand((unsigned int) CTimer::getTime());
    m_SocketID = 1 + (int) ((1 << 30) * (double(rand()) / RAND_MAX));

#ifndef WIN32
    pthread_mutex_init(&m_ControlLock, NULL);
    pthread_mutex_init(&m_IDLock, NULL);
    pthread_mutex_init(&m_InitLock, NULL);
#else
    m_ControlLock = CreateMutex(NULL, false, NULL);
    m_IDLock = CreateMutex(NULL, false, NULL);
    m_InitLock = CreateMutex(NULL, false, NULL);
#endif

#ifndef WIN32
    pthread_key_create(&m_TLSError, TLSDestroy);
#else
    m_TLSError = TlsAlloc();
    m_TLSLock = CreateMutex(NULL, false, NULL);
#endif

    m_pCache = new CCache<CInfoBlock>;
}
```
m_SocketID的初始值是一个随机数。

3. 初始化ns及它的CUDT对象的一些成员变量。特别注意ns->m_Status的赋值，这里该值被赋为了INIT。从状态机的角度来看待CUDTSocket，在执行UDT Socket创建结束执行时，它处于INIT状态下。

4. 将ns放在std::map<UDTSOCKET, CUDTSocket*> m_Sockets中。

5. 将UDT socket的SocketID返回给调用者。

这个接口直接返回一个表示UDT Socket的类对象，而不是一个handle，以便于调用者在调用UDT API时无需每次都输入UDT Socket handle参数，这样的API设计相对于UDT的这种API设计，有什么样的优缺点？

# UDT的错误处理机制

在前面UDT Server和UDT Client的demo程序中，我们有看到，所有的UDT API都会在出错时返回一个错误码，比如UDT::bind()、UDT::bind2()、UDT::listen()和UDT::connect()等返回值类型为int的函数，在出错时返回UDT::ERROR，UDT::socket()和UDT::accept()等返回值类型为UDTSOCKET的函数在出错时返回UDT::INVALID_SOCK。

UDT API的调用者在检测到它们返回了错误值时，通过调用UDT::getlasterror()函数来获取关于异常更加详细的信息。这里我们就来看一下这套机制的实现。

注意看CUDT::socket()函数的实现。在这个函数中，会将实际创建UDT Socket的任务委托给s_UDTUnited.newSocket()，但这个调用会被包在一个try-catch块中。在s_UDTUnited.newSocket()函数执行的过程中发生任何的异常，都会先被CUDT::socket()捕获。CUDT::socket()函数在捕获这些异常之后，会根据捕获的异常的类型创建不同的CUDTException对象，并通过调用CUDTUnited::setError()函数将该CUDTException对象设置给s_UDTUnited。

我们来看一下CUDTUnited::setError()函数的实现(src/api.cpp)：
```
void CUDTUnited::setError(CUDTException* e) {
#ifndef WIN32
    delete (CUDTException*) pthread_getspecific(m_TLSError);
    pthread_setspecific(m_TLSError, e);
#else
    CGuard tg(m_TLSLock);
    delete (CUDTException*)TlsGetValue(m_TLSError);
    TlsSetValue(m_TLSError, e);
    m_mTLSRecord[GetCurrentThreadId()] = e;
#endif
}
```
在这个函数中，会首先取出线程局部存储变量m_TLSError中保存的本线程上一次创建的 CUDTException对象，将其delete掉，并将本次创建的CUDTException对象设置进去。CUDTUnited类定义中m_TLSError的声明(src/api.h)：
```
private:
    pthread_key_t m_TLSError;                         // thread local error record (last error)
#ifndef WIN32
    static void TLSDestroy(void* e) {
        if (NULL != e)
            delete (CUDTException*) e;
    }
#else
    std::map<DWORD, CUDTException*> m_mTLSRecord;
    void checkTLSValue();
    pthread_mutex_t m_TLSLock;
#endif
```

在CUDTUnited构造函数中也可以看到对这个对象的初始化。 CUDTUnited::TLSDestroy()函数是m_TLSError的析构函数，在m_TLSError最后被销毁时，这个函数被调用，以便于释放搜有还未释放的资源。

再来看UDT API的调用者获取上一次发生的异常的函数UDT::getlasterror()：
```
CUDTException* CUDTUnited::getError() {
#ifndef WIN32
    if (NULL == pthread_getspecific(m_TLSError))
        pthread_setspecific(m_TLSError, new CUDTException);
    return (CUDTException*) pthread_getspecific(m_TLSError);
#else
    CGuard tg(m_TLSLock);
    if(NULL == TlsGetValue(m_TLSError))
    {
        CUDTException* e = new CUDTException;
        TlsSetValue(m_TLSError, e);
        m_mTLSRecord[GetCurrentThreadId()] = e;
    }
    return (CUDTException*)TlsGetValue(m_TLSError);
#endif
}

CUDTException& CUDT::getlasterror() {
    return *s_UDTUnited.getError();
}

ERRORINFO& getlasterror() {
    return CUDT::getlasterror();
}
```
调用过程为UDT::getlasterror() -> CUDT::getlasterror() -> CUDTUnited::getError()。

主要就是从s_UDTUnited的线程局部存储变量m_TLSError中取出前面设置的本线程上次创建的CUDTException对象返回给调用者。

总结一下，UDT的使用者在调用UDT API时，UDT API会直接调用CUDT类对应的static API函数，在CUDT类的这些static API函数中会将做实际事情的工作委托给s_UDTUnited的相应函数，但这个委托调用会被包在一个try-catch block中。s_UDTUnited的函数在遇到异常情况时抛出异常，CUDT类的static API函数捕获异常，根据捕获到的异常的具体类型，创建不同的CUDTException对象设置给s_UDTUnited的线程局部存储变量m_TLSError中并向UDT API调用者返回错误码，UDT API的调用者检测到错误码后，通过UDT::getlasterror()获取存储在m_TLSError中的异常。

此处可以看到，CUDT提供的这一层API，一个比较重要的作用大概就是做异常处理了。

在UDT中，使用CUDTException来描述所有出现的异常。可以看一下这个类的定义(src/udt.h)：
```
class UDT_API CUDTException {
 public:
    CUDTException(int major = 0, int minor = 0, int err = -1);
    CUDTException(const CUDTException& e);
    virtual ~CUDTException();

    // Functionality:
    //    Get the description of the exception.
    // Parameters:
    //    None.
    // Returned value:
    //    Text message for the exception description.

    virtual const char* getErrorMessage();

    // Functionality:
    //    Get the system errno for the exception.
    // Parameters:
    //    None.
    // Returned value:
    //    errno.

    virtual int getErrorCode() const;

    // Functionality:
    //    Clear the error code.
    // Parameters:
    //    None.
    // Returned value:
    //    None.

    virtual void clear();

 private:
    int m_iMajor;        // major exception categories

// 0: correct condition
// 1: network setup exception
// 2: network connection broken
// 3: memory exception
// 4: file exception
// 5: method not supported
// 6+: undefined error

    int m_iMinor;		// for specific error reasons
    int m_iErrno;		// errno returned by the system if there is any
    std::string m_strMsg;	// text error message

    std::string m_strAPI;	// the name of UDT function that returns the error
    std::string m_strDebug;  // debug information, set to the original place that causes the error

 public:
    // Error Code
    static const int SUCCESS;
    static const int ECONNSETUP;
    static const int ENOSERVER;
    static const int ECONNREJ;
    static const int ESOCKFAIL;
    static const int ESECFAIL;
    static const int ECONNFAIL;
    static const int ECONNLOST;
    static const int ENOCONN;
    static const int ERESOURCE;
    static const int ETHREAD;
    static const int ENOBUF;
    static const int EFILE;
    static const int EINVRDOFF;
    static const int ERDPERM;
    static const int EINVWROFF;
    static const int EWRPERM;
    static const int EINVOP;
    static const int EBOUNDSOCK;
    static const int ECONNSOCK;
    static const int EINVPARAM;
    static const int EINVSOCK;
    static const int EUNBOUNDSOCK;
    static const int ENOLISTEN;
    static const int ERDVNOSERV;
    static const int ERDVUNBOUND;
    static const int ESTREAMILL;
    static const int EDGRAMILL;
    static const int EDUPLISTEN;
    static const int ELARGEMSG;
    static const int EINVPOLLID;
    static const int EASYNCFAIL;
    static const int EASYNCSND;
    static const int EASYNCRCV;
    static const int ETIMEOUT;
    static const int EPEERERR;
    static const int EUNKNOWN;
};
```
这个class主要通过Major错误码和Minor错误码来描述异常情况，如果是调用系统调用出错了，还会用Errno值。

具体看一下这个class的实现，特别是CUDTException::getErrorMessage()函数，来了解每一个错误码所代表的含义：
```
CUDTException::CUDTException(int major, int minor, int err)
        : m_iMajor(major),
          m_iMinor(minor) {
    if (-1 == err)
#ifndef WIN32
        m_iErrno = errno;
#else
        m_iErrno = GetLastError();
#endif
        else
        m_iErrno = err;
}

CUDTException::CUDTException(const CUDTException& e)
        : m_iMajor(e.m_iMajor),
          m_iMinor(e.m_iMinor),
          m_iErrno(e.m_iErrno),
          m_strMsg() {
}

CUDTException::~CUDTException() {
}

const char* CUDTException::getErrorMessage() {
    // translate "Major:Minor" code into text message.

    switch (m_iMajor) {
        case 0:
            m_strMsg = "Success";
            break;

        case 1:
            m_strMsg = "Connection setup failure";

            switch (m_iMinor) {
                case 1:
                    m_strMsg += ": connection time out";
                    break;

                case 2:
                    m_strMsg += ": connection rejected";
                    break;

                case 3:
                    m_strMsg += ": unable to create/configure UDP socket";
                    break;

                case 4:
                    m_strMsg += ": abort for security reasons";
                    break;

                default:
                    break;
            }

            break;

        case 2:
            switch (m_iMinor) {
                case 1:
                    m_strMsg = "Connection was broken";
                    break;

                case 2:
                    m_strMsg = "Connection does not exist";
                    break;

                default:
                    break;
            }

            break;

        case 3:
            m_strMsg = "System resource failure";

            switch (m_iMinor) {
                case 1:
                    m_strMsg += ": unable to create new threads";
                    break;

                case 2:
                    m_strMsg += ": unable to allocate buffers";
                    break;

                default:
                    break;
            }

            break;

        case 4:
            m_strMsg = "File system failure";

            switch (m_iMinor) {
                case 1:
                    m_strMsg += ": cannot seek read position";
                    break;

                case 2:
                    m_strMsg += ": failure in read";
                    break;

                case 3:
                    m_strMsg += ": cannot seek write position";
                    break;

                case 4:
                    m_strMsg += ": failure in write";
                    break;

                default:
                    break;
            }

            break;

        case 5:
            m_strMsg = "Operation not supported";

            switch (m_iMinor) {
                case 1:
                    m_strMsg += ": Cannot do this operation on a BOUND socket";
                    break;

                case 2:
                    m_strMsg += ": Cannot do this operation on a CONNECTED socket";
                    break;

                case 3:
                    m_strMsg += ": Bad parameters";
                    break;

                case 4:
                    m_strMsg += ": Invalid socket ID";
                    break;

                case 5:
                    m_strMsg += ": Cannot do this operation on an UNBOUND socket";
                    break;

                case 6:
                    m_strMsg += ": Socket is not in listening state";
                    break;

                case 7:
                    m_strMsg += ": Listen/accept is not supported in rendezous connection setup";
                    break;

                case 8:
                    m_strMsg += ": Cannot call connect on UNBOUND socket in rendezvous connection setup";
                    break;

                case 9:
                    m_strMsg += ": This operation is not supported in SOCK_STREAM mode";
                    break;

                case 10:
                    m_strMsg += ": This operation is not supported in SOCK_DGRAM mode";
                    break;

                case 11:
                    m_strMsg += ": Another socket is already listening on the same port";
                    break;

                case 12:
                    m_strMsg += ": Message is too large to send (it must be less than the UDT send buffer size)";
                    break;

                case 13:
                    m_strMsg += ": Invalid epoll ID";
                    break;

                default:
                    break;
            }

            break;

        case 6:
            m_strMsg = "Non-blocking call failure";

            switch (m_iMinor) {
                case 1:
                    m_strMsg += ": no buffer available for sending";
                    break;

                case 2:
                    m_strMsg += ": no data available for reading";
                    break;

                default:
                    break;
            }

            break;

        case 7:
            m_strMsg = "The peer side has signalled an error";

            break;

        default:
            m_strMsg = "Unknown error";
    }

    // Adding "errno" information
    if ((0 != m_iMajor) && (0 < m_iErrno)) {
        m_strMsg += ": ";
#ifndef WIN32
        char errmsg[1024];
        if (strerror_r(m_iErrno, errmsg, 1024) == 0)
            m_strMsg += errmsg;
#else
        LPVOID lpMsgBuf;
        FormatMessage(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS, NULL, m_iErrno, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPTSTR)&lpMsgBuf, 0, NULL);
        m_strMsg += (char*)lpMsgBuf;
        LocalFree(lpMsgBuf);
#endif
    }

    // period
#ifndef WIN32
    m_strMsg += ".";
#endif

    return m_strMsg.c_str();
}

int CUDTException::getErrorCode() const {
    return m_iMajor * 1000 + m_iMinor;
}

void CUDTException::clear() {
    m_iMajor = 0;
    m_iMinor = 0;
    m_iErrno = 0;
}

const int CUDTException::SUCCESS = 0;
const int CUDTException::ECONNSETUP = 1000;
const int CUDTException::ENOSERVER = 1001;
const int CUDTException::ECONNREJ = 1002;
const int CUDTException::ESOCKFAIL = 1003;
const int CUDTException::ESECFAIL = 1004;
const int CUDTException::ECONNFAIL = 2000;
const int CUDTException::ECONNLOST = 2001;
const int CUDTException::ENOCONN = 2002;
const int CUDTException::ERESOURCE = 3000;
const int CUDTException::ETHREAD = 3001;
const int CUDTException::ENOBUF = 3002;
const int CUDTException::EFILE = 4000;
const int CUDTException::EINVRDOFF = 4001;
const int CUDTException::ERDPERM = 4002;
const int CUDTException::EINVWROFF = 4003;
const int CUDTException::EWRPERM = 4004;
const int CUDTException::EINVOP = 5000;
const int CUDTException::EBOUNDSOCK = 5001;
const int CUDTException::ECONNSOCK = 5002;
const int CUDTException::EINVPARAM = 5003;
const int CUDTException::EINVSOCK = 5004;
const int CUDTException::EUNBOUNDSOCK = 5005;
const int CUDTException::ENOLISTEN = 5006;
const int CUDTException::ERDVNOSERV = 5007;
const int CUDTException::ERDVUNBOUND = 5008;
const int CUDTException::ESTREAMILL = 5009;
const int CUDTException::EDGRAMILL = 5010;
const int CUDTException::EDUPLISTEN = 5011;
const int CUDTException::ELARGEMSG = 5012;
const int CUDTException::EINVPOLLID = 5013;
const int CUDTException::EASYNCFAIL = 6000;
const int CUDTException::EASYNCSND = 6001;
const int CUDTException::EASYNCRCV = 6002;
const int CUDTException::ETIMEOUT = 6003;
const int CUDTException::EPEERERR = 7000;
const int CUDTException::EUNKNOWN = -1;
```
Done。
