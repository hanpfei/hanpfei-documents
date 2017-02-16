---
title: UDT协议实现分析——连接的建立
date: 2015-09-12 16:05:49
categories: 网络协议
tags:
- 源码分析
- 网络协议
- UDT
---

UDT Server在执行UDT::listen()之后，就可以接受其它节点的连接请求了。这里我们研究一下UDT连接建立的过程。
<!--more-->
# 连接的发起

来看连接的发起方。如前面我们看到的那样，UDT Client创建一个Socket，可以将该Socket绑定到某个端口，也可以不绑定，然后就可以调用UDT::connect()将这个Socket连接到UDT Server了。来看UDT::connect()的定义(src/api.cpp)：
```
int CUDTUnited::connect(const UDTSOCKET u, const sockaddr* name, int namelen) {
    CUDTSocket* s = locate(u);
    if (NULL == s)
        throw CUDTException(5, 4, 0);

    CGuard cg(s->m_ControlLock);

    // check the size of SOCKADDR structure
    if (AF_INET == s->m_iIPversion) {
        if (namelen != sizeof(sockaddr_in))
            throw CUDTException(5, 3, 0);
    } else {
        if (namelen != sizeof(sockaddr_in6))
            throw CUDTException(5, 3, 0);
    }

    // a socket can "connect" only if it is in INIT or OPENED status
    if (INIT == s->m_Status) {
        if (!s->m_pUDT->m_bRendezvous) {
            s->m_pUDT->open();
            updateMux(s);
            s->m_Status = OPENED;
        } else
            throw CUDTException(5, 8, 0);
    } else if (OPENED != s->m_Status)
        throw CUDTException(5, 2, 0);

    // connect_complete() may be called before connect() returns.
    // So we need to update the status before connect() is called,
    // otherwise the status may be overwritten with wrong value (CONNECTED vs. CONNECTING).
    s->m_Status = CONNECTING;
    try {
        s->m_pUDT->connect(name);
    } catch (CUDTException &e) {
        s->m_Status = OPENED;
        throw e;
    }

    // record peer address
    delete s->m_pPeerAddr;
    if (AF_INET == s->m_iIPversion) {
        s->m_pPeerAddr = (sockaddr*) (new sockaddr_in);
        memcpy(s->m_pPeerAddr, name, sizeof(sockaddr_in));
    } else {
        s->m_pPeerAddr = (sockaddr*) (new sockaddr_in6);
        memcpy(s->m_pPeerAddr, name, sizeof(sockaddr_in6));
    }

    return 0;
}

int CUDT::connect(UDTSOCKET u, const sockaddr* name, int namelen) {
    try {
        return s_UDTUnited.connect(u, name, namelen);
    } catch (CUDTException &e) {
        s_UDTUnited.setError(new CUDTException(e));
        return ERROR;
    } catch (bad_alloc&) {
        s_UDTUnited.setError(new CUDTException(3, 2, 0));
        return ERROR;
    } catch (...) {
        s_UDTUnited.setError(new CUDTException(-1, 0, 0));
        return ERROR;
    }
}


int connect(UDTSOCKET u, const struct sockaddr* name, int namelen) {
    return CUDT::connect(u, name, namelen);
}
```
UDT::connect() API实现的结构跟其它的API没有太大的区别，不再赘述，直接来分析CUDTUnited::connect()：

1. 调用CUDTUnited::locate()，查找UDT Socket对应的CUDTSocket结构。若找不到，则抛出异常直接返回；否则，继续执行。

2. 根据UDT Socket的IP版本，检查目标地址的有效性。若无效，则退出，否则继续执行。

3. 检查UDT Socket的状态。确保只有处于INIT或OPENED状态的UDT Socket才可以执行connect()操作。新创建的UDT Socket处于INIT状态，bind之后UDT Socket处于OPENED状态。如果UDT Socket处于INIT状态，且不是Rendezvous模式，还会执行s->m_pUDT->open()，将UDT Socket与多路复用器CMultiplexer，然后将状态置为OPENED。
前面我们在bind的执行过程中有看到[将UDT Socket与多路复用器](http://my.oschina.net/wolfcs/blog/503959)[CMultiplexer关联](http://my.oschina.net/wolfcs/blog/503959)的过程CUDTUnited::updateMux()。但这里执行的updateMux()的不同之处在于，它既没有传递有效的系统UDP socket，也没有传递有效的本地端口地址。回想updateMux()的实现，这两个参数主要决定了CMultiplexer的CChannel将与哪个端口关联。来看两个CChannel::open()的实现(src/channel.cpp)：

```
void CChannel::open(const sockaddr* addr) {
    // construct an socket
    m_iSocket = ::socket(m_iIPversion, SOCK_DGRAM, 0);

#ifdef WIN32
    if (INVALID_SOCKET == m_iSocket)
#else
    if (m_iSocket < 0)
#endif
        throw CUDTException(1, 0, NET_ERROR);

    if (NULL != addr) {
        socklen_t namelen = m_iSockAddrSize;

        if (0 != ::bind(m_iSocket, addr, namelen))
            throw CUDTException(1, 3, NET_ERROR);
    } else {
        //sendto or WSASendTo will also automatically bind the socket
        addrinfo hints;
        addrinfo* res;

        memset(&hints, 0, sizeof(struct addrinfo));

        hints.ai_flags = AI_PASSIVE;
        hints.ai_family = m_iIPversion;
        hints.ai_socktype = SOCK_DGRAM;

        if (0 != ::getaddrinfo(NULL, "0", &hints, &res))
            throw CUDTException(1, 3, NET_ERROR);

        if (0 != ::bind(m_iSocket, res->ai_addr, res->ai_addrlen))
            throw CUDTException(1, 3, NET_ERROR);

        ::freeaddrinfo(res);
    }

    setUDPSockOpt();
}

void CChannel::open(UDPSOCKET udpsock) {
    m_iSocket = udpsock;
    setUDPSockOpt();
}

void CChannel::setUDPSockOpt() {
#if defined(BSD) || defined(OSX)
    // BSD system will fail setsockopt if the requested buffer size exceeds system maximum value
    int maxsize = 64000;
    if (0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_RCVBUF, (char*)&m_iRcvBufSize, sizeof(int)))
    ::setsockopt(m_iSocket, SOL_SOCKET, SO_RCVBUF, (char*)&maxsize, sizeof(int));
    if (0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_SNDBUF, (char*)&m_iSndBufSize, sizeof(int)))
    ::setsockopt(m_iSocket, SOL_SOCKET, SO_SNDBUF, (char*)&maxsize, sizeof(int));
#else
    // for other systems, if requested is greated than maximum, the maximum value will be automactally used
    if ((0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_RCVBUF, (char*) &m_iRcvBufSize, sizeof(int)))
            || (0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_SNDBUF, (char*) &m_iSndBufSize, sizeof(int))))
        throw CUDTException(1, 3, NET_ERROR);
#endif

    timeval tv;
    tv.tv_sec = 0;
#if defined (BSD) || defined (OSX)
    // Known BSD bug as the day I wrote this code.
    // A small time out value will cause the socket to block forever.
    tv.tv_usec = 10000;
#else
    tv.tv_usec = 100;
#endif

#ifdef UNIX
    // Set non-blocking I/O
    // UNIX does not support SO_RCVTIMEO
    int opts = ::fcntl(m_iSocket, F_GETFL);
    if (-1 == ::fcntl(m_iSocket, F_SETFL, opts | O_NONBLOCK))
    throw CUDTException(1, 3, NET_ERROR);
#elif WIN32
    DWORD ot = 1;  //milliseconds
    if (0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_RCVTIMEO, (char *)&ot, sizeof(DWORD)))
    throw CUDTException(1, 3, NET_ERROR);
#else
    // Set receiving time-out value
    if (0 != ::setsockopt(m_iSocket, SOL_SOCKET, SO_RCVTIMEO, (char *) &tv, sizeof(timeval)))
        throw CUDTException(1, 3, NET_ERROR);
#endif
}
```
可以看到CChannel::open()主要是把UDT的CChannel与一个系统的UDP socket关联起来，它们总共处理了3中情况，一是调用者已经创建并绑定到了目标端口的系统UDP socket，这种最简单，直接将传递进来的UDPSOCKET赋值给CChannel的m_iSocket，然后设置系统UDP socket的选项；二是传递进来了一个有效的本地端口地址，此时CChannel会自己先创建一个系统UDP socket，并将该socket绑定到传进来的目标端口地址，一、二两种情况正是UDT的两个bind API的情况；三是既没有有效的系统UDP socket，又没有有效的本地端口地址传进来，则会在创建了系统UDP socket之后，先再找一个可用的端口地址，然后将该socket绑定到找到的端口地址，这也就是UDT Socket没有bind，直接connect的情况。

4. 将UDT Socket的状态置为CONNECTING。

5. 执行s->m_pUDT->connect(name)，连接UDT Server。如果连接失败，有异常抛出，UDT Socket的状态会退回到OPENED状态，然后返回。在这个函数中会完成建立连接整个的网络消息交互过程。

6. 将连接的目标地址复制到UDT Socket的Peer Address。然后返回0表示成功结束。

在仔细地分析连接建立过程中的数据包交互之前，可以先粗略地看一下这个过程收发了几个包，及各个包收发的顺序。我们知道在UDT中，所有数据包的收发都是通过CChannel完成的，我们可以在CChannel::sendto()和CChannel::recvfrom()中加log来track这一过程。通过UDT提供的demo程序appserver和appclient(在app/目录下)来研究。先在一个终端下执行appserver：
```
xxxxxx@ThundeRobot:/media/data/downloads/hudt/app$ ./appserver 
server is ready at port: 9000
```
改造appclient，使得它只发送一个比较小的数据包就结束，编译后在另一个终端下执行，可以看到有如下的logs吐出来：
```
xxxxxx@ThundeRobot:/media/data/downloads/hudt/app$ ./appclient 127.0.0.1 9000
To connect
CRcvQueue::registerConnector
Send packet 0


Receive packet 364855723
unit->m_Packet.m_iID 364855723
Send packet 0


Receive packet 364855723
unit->m_Packet.m_iID 364855723

To send data.
send 10 bytes
Send packet 1020108693


Receive packet 364855723
unit->m_Packet.m_iID 364855723
Send packet 1020108693


Receive packet 364855723
unit->m_Packet.m_iID 364855723
Send packet 1020108693


Receive packet 364855723
unit->m_Packet.m_iID 364855723
Send packet 1020108693
```
在appclient运行的这段时间，在运行appserver的终端下的可以看到有如下的logs输出：
```
xxxxxx@ThundeRobot:/media/data/downloads/hudt/app$ ./appserver 
server is ready at port: 9000

Receive packet 0
unit->m_Packet.m_iID 0
Send packet 364855723


Receive packet 0
unit->m_Packet.m_iID 0
new CUDTSocket SocketID is 1020108693 PeerID 364855723
Send packet 364855723

new connection: 127.0.0.1:59847


Receive packet 1020108693
unit->m_Packet.m_iID 1020108693
Send packet 364855723

Send packet 364855723

Send packet 364855723


Receive packet 1020108693
unit->m_Packet.m_iID 1020108693

Receive packet 1020108693
unit->m_Packet.m_iID 1020108693

Receive packet 1020108693
unit->m_Packet.m_iID 1020108693
recv:Connection was broken.
```
可以看到，UDT Client端先发送了一个消息MSG1给UDT Server；UDT Server端收到消息MSG1之后，回了一个消息MSG2给UDT Client；UDT Client收到消息MSG2，又回了一个消息MSG3给UDT Server；UDT Server收到消息MSG3后又回了一个消息MSG4给UDT Client，然后从UDT::accept()返回，自此UDT Server认为一个连接已经成功建立；UDT Client则在收到消息MSG4后，从UDT::connect()返回，并自此认为连接已成功建立，可以进行数据的收发了。用一幅图来描述这个过程：

![150954_myfS_919237.png](https://www.wolfcstech.com/images/1315506-eadbc50fbe607bfa.png)

至于MSG1、2、3、4的具体格式及内容，则留待我们后面来具体分析了。

接着来看连接建立过程消息交互具体的实现，也就是CUDT::connect()函数：

```
void CUDT::connect(const sockaddr* serv_addr) {
    CGuard cg(m_ConnectionLock);

    if (!m_bOpened)
        throw CUDTException(5, 0, 0);

    if (m_bListening)
        throw CUDTException(5, 2, 0);

    if (m_bConnecting || m_bConnected)
        throw CUDTException(5, 2, 0);

    // record peer/server address
    delete m_pPeerAddr;
    m_pPeerAddr = (AF_INET == m_iIPversion) ? (sockaddr*) new sockaddr_in : (sockaddr*) new sockaddr_in6;
    memcpy(m_pPeerAddr, serv_addr, (AF_INET == m_iIPversion) ? sizeof(sockaddr_in) : sizeof(sockaddr_in6));

    // register this socket in the rendezvous queue
    // RendezevousQueue is used to temporarily store incoming handshake, non-rendezvous connections also require this function
    uint64_t ttl = 3000000;
    if (m_bRendezvous)
        ttl *= 10;
    ttl += CTimer::getTime();
    m_pRcvQueue->registerConnector(m_SocketID, this, m_iIPversion, serv_addr, ttl);

    // This is my current configurations
    m_ConnReq.m_iVersion = m_iVersion;
    m_ConnReq.m_iType = m_iSockType;
    m_ConnReq.m_iMSS = m_iMSS;
    m_ConnReq.m_iFlightFlagSize = (m_iRcvBufSize < m_iFlightFlagSize) ? m_iRcvBufSize : m_iFlightFlagSize;
    m_ConnReq.m_iReqType = (!m_bRendezvous) ? 1 : 0;
    m_ConnReq.m_iID = m_SocketID;
    CIPAddress::ntop(serv_addr, m_ConnReq.m_piPeerIP, m_iIPversion);

    // Random Initial Sequence Number
    srand((unsigned int) CTimer::getTime());
    m_iISN = m_ConnReq.m_iISN = (int32_t) (CSeqNo::m_iMaxSeqNo * (double(rand()) / RAND_MAX));

    m_iLastDecSeq = m_iISN - 1;
    m_iSndLastAck = m_iISN;
    m_iSndLastDataAck = m_iISN;
    m_iSndCurrSeqNo = m_iISN - 1;
    m_iSndLastAck2 = m_iISN;
    m_ullSndLastAck2Time = CTimer::getTime();

    // Inform the server my configurations.
    CPacket request;
    char* reqdata = new char[m_iPayloadSize];
    request.pack(0, NULL, reqdata, m_iPayloadSize);
    // ID = 0, connection request
    request.m_iID = 0;

    int hs_size = m_iPayloadSize;
    m_ConnReq.serialize(reqdata, hs_size);
    request.setLength(hs_size);
    m_pSndQueue->sendto(serv_addr, request);
    m_llLastReqTime = CTimer::getTime();

    m_bConnecting = true;

    // asynchronous connect, return immediately
    if (!m_bSynRecving) {
        delete[] reqdata;
        return;
    }

    // Wait for the negotiated configurations from the peer side.
    CPacket response;
    char* resdata = new char[m_iPayloadSize];
    response.pack(0, NULL, resdata, m_iPayloadSize);

    CUDTException e(0, 0);

    while (!m_bClosing) {
        // avoid sending too many requests, at most 1 request per 250ms
        if (CTimer::getTime() - m_llLastReqTime > 250000) {
            m_ConnReq.serialize(reqdata, hs_size);
            request.setLength(hs_size);
            if (m_bRendezvous)
                request.m_iID = m_ConnRes.m_iID;
            m_pSndQueue->sendto(serv_addr, request);
            m_llLastReqTime = CTimer::getTime();
        }

        response.setLength(m_iPayloadSize);
        if (m_pRcvQueue->recvfrom(m_SocketID, response) > 0) {
            if (connect(response) <= 0)
                break;

            // new request/response should be sent out immediately on receving a response
            m_llLastReqTime = 0;
        }

        if (CTimer::getTime() > ttl) {
            // timeout
            e = CUDTException(1, 1, 0);
            break;
        }
    }

    delete[] reqdata;
    delete[] resdata;

    if (e.getErrorCode() == 0) {
        if (m_bClosing)                                                 // if the socket is closed before connection...
            e = CUDTException(1);
        else if (1002 == m_ConnRes.m_iReqType)                          // connection request rejected
            e = CUDTException(1, 2, 0);
        else if ((!m_bRendezvous) && (m_iISN != m_ConnRes.m_iISN))      // secuity check
            e = CUDTException(1, 4, 0);
    }

    if (e.getErrorCode() != 0)
        throw e;
}
```
可以看到，在这个函数中主要完成了如下的这样一些事情：

1. 检查CUDT的状态。确保只有已经与多路复用器关联，即处于OPENED状态的UDT Socket才能执行CUDT::connect()操作。如前面看到的，bind操作可以使UDT Socket进入OPENED状态。对于没有进行过bind的UDT Socket，CUDTUnited::connect()会做这样的保证。

2. 拷贝目标网络地址为UDT Socket的PeerAddr。

3. 执行m_pRcvQueue->registerConnector()向接收队列注册Connector。来看这个函数的执行过程(src/queue.cpp)：

```
void CRendezvousQueue::insert(const UDTSOCKET& id, CUDT* u, int ipv, const sockaddr* addr, uint64_t ttl) {
    CGuard vg(m_RIDVectorLock);

    CRL r;
    r.m_iID = id;
    r.m_pUDT = u;
    r.m_iIPversion = ipv;
    r.m_pPeerAddr = (AF_INET == ipv) ? (sockaddr*) new sockaddr_in : (sockaddr*) new sockaddr_in6;
    memcpy(r.m_pPeerAddr, addr, (AF_INET == ipv) ? sizeof(sockaddr_in) : sizeof(sockaddr_in6));
    r.m_ullTTL = ttl;

    m_lRendezvousID.push_back(r);
}

void CRcvQueue::registerConnector(const UDTSOCKET& id, CUDT* u, int ipv, const sockaddr* addr, uint64_t ttl) {
    m_pRendezvousQueue->insert(id, u, ipv, addr, ttl);
}
```

可以看到，在这个函数中，主要是向接收队列CRcvQueue的CRendezvousQueue m_pRendezvousQueue中插入了一个CRL结构。那CRendezvousQueue又是个什么东西呢？来看它的定义(src/queue.h)：

```
class CRendezvousQueue {
 public:
    CRendezvousQueue();
    ~CRendezvousQueue();

 public:
    void insert(const UDTSOCKET& id, CUDT* u, int ipv, const sockaddr* addr, uint64_t ttl);
    void remove(const UDTSOCKET& id);
    CUDT* retrieve(const sockaddr* addr, UDTSOCKET& id);

    void updateConnStatus();

 private:
    struct CRL {
        UDTSOCKET m_iID;			// UDT socket ID (self)
        CUDT* m_pUDT;			// UDT instance
        int m_iIPversion;                 // IP version
        sockaddr* m_pPeerAddr;		// UDT sonnection peer address
        uint64_t m_ullTTL;			// the time that this request expires
    };
    std::list<CRL> m_lRendezvousID;      // The sockets currently in rendezvous mode

    pthread_mutex_t m_RIDVectorLock;
};
```

可以看到，它就是一个简单的容器，提供的操作也是常规的插入、移除及检索等操作：

```
void CRendezvousQueue::remove(const UDTSOCKET& id) {
    CGuard vg(m_RIDVectorLock);

    for (list<CRL>::iterator i = m_lRendezvousID.begin(); i != m_lRendezvousID.end(); ++i) {
        if (i->m_iID == id) {
            if (AF_INET == i->m_iIPversion)
                delete (sockaddr_in*) i->m_pPeerAddr;
            else
                delete (sockaddr_in6*) i->m_pPeerAddr;

            m_lRendezvousID.erase(i);

            return;
        }
    }
}

CUDT* CRendezvousQueue::retrieve(const sockaddr* addr, UDTSOCKET& id) {
    CGuard vg(m_RIDVectorLock);

    // TODO: optimize search
    for (list<CRL>::iterator i = m_lRendezvousID.begin(); i != m_lRendezvousID.end(); ++i) {
        if (CIPAddress::ipcmp(addr, i->m_pPeerAddr, i->m_iIPversion) && ((0 == id) || (id == i->m_iID))) {
            id = i->m_iID;
            return i->m_pUDT;
        }
    }

    return NULL;
}
```
那接收队列CRcvQueue是用这个队列来做什么的呢？这主要与接收队列CRcvQueue的消息dispatch机制有关。在接收队列CRcvQueue的worker线程中，接收到一条消息之后，它会根据消息的目标SocketID，及发送端的地址等信息，将消息以不同的方式进行dispatch，m_pRendezvousQueue中的CUDT是其中的一类dispatch目标。后面我们在研究消息接收时，会再来仔细研究接收队列CRcvQueue的worker线程及m_pRendezvousQueue。

4. 构造 连接请求 消息CHandShake m_ConnReq。可以看一下CHandShake的定义(src/packet.h)：

```
class CHandShake {
 public:
    CHandShake();

    int serialize(char* buf, int& size);
    int deserialize(const char* buf, int size);

 public:
    static const int m_iContentSize;  // Size of hand shake data

 public:
    int32_t m_iVersion;          // UDT version
    int32_t m_iType;             // UDT socket type
    int32_t m_iISN;              // random initial sequence number
    int32_t m_iMSS;              // maximum segment size
    int32_t m_iFlightFlagSize;   // flow control window size
    int32_t m_iReqType;  // connection request type: 1: regular connection request, 0: rendezvous connection request, -1/-2: response
    int32_t m_iID;		// socket ID
    int32_t m_iCookie;		// cookie
    uint32_t m_piPeerIP[4];	// The IP address that the peer's UDP port is bound to
};
```

CHandShake的m_iID为发起端UDT Socket的SocketID，请求类型m_iReqType将被设置为了1，还设置了m_iMSS用于协商MSS值。CHandShake的构造函数会初始化所有的字段(src/packet.cpp)：
```
CHandShake::CHandShake()
        : m_iVersion(0),
          m_iType(0),
          m_iISN(0),
          m_iMSS(0),
          m_iFlightFlagSize(0),
          m_iReqType(0),
          m_iID(0),
          m_iCookie_iCookie(0) {
    for (int i = 0; i < 4; ++i)
        m_piPeerIP[i] = 0;
}
```

可以看到m_iCookie被初始化为了0。但注意在这里，CHandShake m_ConnReq的构造过程中，m_iCookie并没有被赋予新值。

5. 随机初始化序列号Sequence Number。

6. 创建一个CPacket结构request，为它创建大小为m_iPayloadSize的缓冲区，将该缓冲区pack进CPacket结构，并专门把request.m_iID，也就是这个包发送的目的UDT SocketID，设置为0。

m_iPayloadSize的值根据UDT Socket创建者的不同，在不同的地方设置。由应用程序创建的UDT Socket在CUDT::open()中设置，比如Listening的UDT Socket在bind时会执行CUDT::open()，或者连接UDT Server但没有执行过bind操作的UDT Socket会在CUDTUnited::connect()中执行CUDT::open()；UDT Server中由Listening的UDT Socket收到连接请求时创建的UDT Socket，在CUDT::connect(const sockaddr* peer, CHandShake* hs)中初设置；发起连接的UDT Socket还会在CUDT::connect(const CPacket& response)中再次更新这个值。但这个值总是被设置为m_iPktSize - CPacket::m_iPktHdrSize，CPacket::m_iPktHdrSize为固定的UDT Packet Header大小16。

m_iPktSize总是与m_iPayloadSize在相同的地方设置，被设置为m_iMSS - 28。m_iMSS，MSS（Maximum Segment Size，最大报文长度），这里是UDT协议定义的一个选项，用于在UDT连接建立时，收发双方协商通信时每一个报文段所能承载的最大数据长度。在CUDT对象创建时被初始化为1500，但可以通过UDT::setsockopt()进行设置。

这里先来看一下CPacket的结构(src/packet.h)：

```
class CPacket {
    friend class CChannel;
    friend class CSndQueue;
    friend class CRcvQueue;

 public:
    int32_t& m_iSeqNo;                   // alias: sequence number
    int32_t& m_iMsgNo;                   // alias: message number
    int32_t& m_iTimeStamp;               // alias: timestamp
    int32_t& m_iID;			// alias: socket ID
    char*& m_pcData;                     // alias: data/control information

    static const int m_iPktHdrSize;	// packet header size

 public:
    CPacket();
    ~CPacket();

    // Functionality:
    //    Get the payload or the control information field length.
    // Parameters:
    //    None.
    // Returned value:
    //    the payload or the control information field length.

    int getLength() const;

    // Functionality:
    //    Set the payload or the control information field length.
    // Parameters:
    //    0) [in] len: the payload or the control information field length.
    // Returned value:
    //    None.

    void setLength(int len);

    // Functionality:
    //    Pack a Control packet.
    // Parameters:
    //    0) [in] pkttype: packet type filed.
    //    1) [in] lparam: pointer to the first data structure, explained by the packet type.
    //    2) [in] rparam: pointer to the second data structure, explained by the packet type.
    //    3) [in] size: size of rparam, in number of bytes;
    // Returned value:
    //    None.

    void pack(int pkttype, void* lparam = NULL, void* rparam = NULL, int size = 0);

    // Functionality:
    //    Read the packet vector.
    // Parameters:
    //    None.
    // Returned value:
    //    Pointer to the packet vector.

    iovec* getPacketVector();

    // Functionality:
    //    Read the packet flag.
    // Parameters:
    //    None.
    // Returned value:
    //    packet flag (0 or 1).

    int getFlag() const;

    // Functionality:
    //    Read the packet type.
    // Parameters:
    //    None.
    // Returned value:
    //    packet type filed (000 ~ 111).

    int getType() const;

    // Functionality:
    //    Read the extended packet type.
    // Parameters:
    //    None.
    // Returned value:
    //    extended packet type filed (0x000 ~ 0xFFF).

    int getExtendedType() const;

    // Functionality:
    //    Read the ACK-2 seq. no.
    // Parameters:
    //    None.
    // Returned value:
    //    packet header field (bit 16~31).

    int32_t getAckSeqNo() const;

    // Functionality:
    //    Read the message boundary flag bit.
    // Parameters:
    //    None.
    // Returned value:
    //    packet header field [1] (bit 0~1).

    int getMsgBoundary() const;

    // Functionality:
    //    Read the message inorder delivery flag bit.
    // Parameters:
    //    None.
    // Returned value:
    //    packet header field [1] (bit 2).

    bool getMsgOrderFlag() const;

    // Functionality:
    //    Read the message sequence number.
    // Parameters:
    //    None.
    // Returned value:
    //    packet header field [1] (bit 3~31).

    int32_t getMsgSeq() const;

    // Functionality:
    //    Clone this packet.
    // Parameters:
    //    None.
    // Returned value:
    //    Pointer to the new packet.

    CPacket* clone() const;

 protected:
    uint32_t m_nHeader[4];               // The 128-bit header field
    iovec m_PacketVector[2];             // The 2-demension vector of UDT packet [header, data]

    int32_t __pad;

 protected:
    CPacket& operator=(const CPacket&);
};
```

它的数据成员是有4个uint32_t元素的数组m_nHeader，描述UDT Packet的Header，和有两个元素的iovec数组m_PacketVector。另外的几个引用则主要是为了方便对这些数据成员的访问，看下CPacket的构造函数就一目了然了(src/packet.cpp)：

```
// Set up the aliases in the constructure
CPacket::CPacket()
        : m_iSeqNo((int32_t&) (m_nHeader[0])),
          m_iMsgNo((int32_t&) (m_nHeader[1])),
          m_iTimeStamp((int32_t&) (m_nHeader[2])),
          m_iID((int32_t&) (m_nHeader[3])),
          m_pcData((char*&) (m_PacketVector[1].iov_base)),
          __pad() {
    for (int i = 0; i < 4; ++i)
        m_nHeader[i] = 0;
    m_PacketVector[0].iov_base = (char *) m_nHeader;
    m_PacketVector[0].iov_len = CPacket::m_iPktHdrSize;
    m_PacketVector[1].iov_base = NULL;
    m_PacketVector[1].iov_len = 0;
}
```

注意m_PacketVector的第一个元素指向了m_nHeader。

在CPacket::pack()中：

```
void CPacket::pack(int pkttype, void* lparam, void* rparam, int size) {
    // Set (bit-0 = 1) and (bit-1~15 = type)
    m_nHeader[0] = 0x80000000 | (pkttype << 16);

    // Set additional information and control information field
    switch (pkttype) {
        case 2:  //0010 - Acknowledgement (ACK)
            // ACK packet seq. no.
            if (NULL != lparam)
                m_nHeader[1] = *(int32_t *) lparam;

            // data ACK seq. no.
            // optional: RTT (microsends), RTT variance (microseconds) advertised flow window size (packets), and estimated link capacity (packets per second)
            m_PacketVector[1].iov_base = (char *) rparam;
            m_PacketVector[1].iov_len = size;

            break;

        case 6:  //0110 - Acknowledgement of Acknowledgement (ACK-2)
            // ACK packet seq. no.
            m_nHeader[1] = *(int32_t *) lparam;

            // control info field should be none
            // but "writev" does not allow this
            m_PacketVector[1].iov_base = (char *) &__pad;  //NULL;
            m_PacketVector[1].iov_len = 4;  //0;

            break;

        case 3:  //0011 - Loss Report (NAK)
            // loss list
            m_PacketVector[1].iov_base = (char *) rparam;
            m_PacketVector[1].iov_len = size;

            break;

        case 4:  //0100 - Congestion Warning
            // control info field should be none
            // but "writev" does not allow this
            m_PacketVector[1].iov_base = (char *) &__pad;  //NULL;
            m_PacketVector[1].iov_len = 4;  //0;

            break;

        case 1:  //0001 - Keep-alive
            // control info field should be none
            // but "writev" does not allow this
            m_PacketVector[1].iov_base = (char *) &__pad;  //NULL;
            m_PacketVector[1].iov_len = 4;  //0;

            break;

        case 0:  //0000 - Handshake
            // control info filed is handshake info
            m_PacketVector[1].iov_base = (char *) rparam;
            m_PacketVector[1].iov_len = size;  //sizeof(CHandShake);

            break;

        case 5:  //0101 - Shutdown
            // control info field should be none
            // but "writev" does not allow this
            m_PacketVector[1].iov_base = (char *) &__pad;  //NULL;
            m_PacketVector[1].iov_len = 4;  //0;

            break;

        case 7:  //0111 - Message Drop Request
            // msg id
            m_nHeader[1] = *(int32_t *) lparam;

            //first seq no, last seq no
            m_PacketVector[1].iov_base = (char *) rparam;
            m_PacketVector[1].iov_len = size;

            break;

        case 8:  //1000 - Error Signal from the Peer Side
            // Error type
            m_nHeader[1] = *(int32_t *) lparam;

            // control info field should be none
            // but "writev" does not allow this
            m_PacketVector[1].iov_base = (char *) &__pad;  //NULL;
            m_PacketVector[1].iov_len = 4;  //0;

            break;

        case 32767:  //0x7FFF - Reserved for user defined control packets
            // for extended control packet
            // "lparam" contains the extended type information for bit 16 - 31
            // "rparam" is the control information
            m_nHeader[0] |= *(int32_t *) lparam;

            if (NULL != rparam) {
                m_PacketVector[1].iov_base = (char *) rparam;
                m_PacketVector[1].iov_len = size;
            } else {
                m_PacketVector[1].iov_base = (char *) &__pad;
                m_PacketVector[1].iov_len = 4;
            }

            break;

        default:
            break;
    }
}
```

在CPacket::pack()中，首先将m_nHeader[0]，也就是m_iSeqNo的bit-0设为1表示这是一个控制包，将bit-1～15设置为消息的类型，然后根据消息的不同类型进行不同的处理。对于Handshake消息，其pkttype为0，这里主要关注pkttype为0的case。可见它就是让m_PacketVector[1]指向前面创建的缓冲区。

7. 将Handshake消息m_ConnReq序列化进前面创建的缓冲区，并正确地设置CPacket request的长度：

```
void CPacket::setLength(int len) {
    m_PacketVector[1].iov_len = len;
}


int CHandShake::serialize(char* buf, int& size) {
    if (size < m_iContentSize)
        return -1;

    int32_t* p = (int32_t*) buf;
    *p++ = m_iVersion;
    *p++ = m_iType;
    *p++ = m_iISN;
    *p++ = m_iMSS;
    *p++ = m_iFlightFlagSize;
    *p++ = m_iReqType;
    *p++ = m_iID;
    *p++ = m_iCookie;
    for (int i = 0; i < 4; ++i)
        *p++ = m_piPeerIP[i];

    size = m_iContentSize;

    return 0;
}
```

序列化时，会将Handshake消息m_ConnReq全部的内容拷贝进缓冲区。略感奇怪，这个地方竟然完全没有顾及字节序的问题。

8. 调用发送队列的sendto()函数，向目标地址发送消息：

```
int CSndQueue::sendto(const sockaddr* addr, CPacket& packet) {
    // send out the packet immediately (high priority), this is a control packet
    m_pChannel->sendto(addr, packet);
    return packet.getLength();
}
```

CSndQueue的sendto()函数直接调用了CChannel::sendto()：

```
int CChannel::sendto(const sockaddr* addr, CPacket& packet) const {
    cout << "CChannel send packet " << packet.m_iID << endl << endl;

    // convert control information into network order
    if (packet.getFlag())
        for (int i = 0, n = packet.getLength() / 4; i < n; ++i)
            *((uint32_t *) packet.m_pcData + i) = htonl(*((uint32_t *) packet.m_pcData + i));

    // convert packet header into network order
    //for (int j = 0; j < 4; ++ j)
    //   packet.m_nHeader[j] = htonl(packet.m_nHeader[j]);
    uint32_t* p = packet.m_nHeader;
    for (int j = 0; j < 4; ++j) {
        *p = htonl(*p);
        ++p;
    }

#ifndef WIN32
    msghdr mh;
    mh.msg_name = (sockaddr*) addr;
    mh.msg_namelen = m_iSockAddrSize;
    mh.msg_iov = (iovec*) packet.m_PacketVector;
    mh.msg_iovlen = 2;
    mh.msg_control = NULL;
    mh.msg_controllen = 0;
    mh.msg_flags = 0;

    int res = ::sendmsg(m_iSocket, &mh, 0);
#else
    DWORD size = CPacket::m_iPktHdrSize + packet.getLength();
    int addrsize = m_iSockAddrSize;
    int res = ::WSASendTo(m_iSocket, (LPWSABUF)packet.m_PacketVector, 2, &size, 0, addr, addrsize, NULL, NULL);
    res = (0 == res) ? size : -1;
#endif

    // convert back into local host order
    //for (int k = 0; k < 4; ++ k)
    //   packet.m_nHeader[k] = ntohl(packet.m_nHeader[k]);
    p = packet.m_nHeader;
    for (int k = 0; k < 4; ++k) {
        *p = ntohl(*p);
        ++p;
    }

    if (packet.getFlag()) {
        for (int l = 0, n = packet.getLength() / 4; l < n; ++l)
            *((uint32_t *) packet.m_pcData + l) = ntohl(*((uint32_t *) packet.m_pcData + l));
    }

    return res;
}
```
在CChannel::sendto()中会处理Header的字节序问题。

这里总结一下，UDT Client向UDT Server发送的连接建立请求消息的内容：消息主要分为两个部分一个是消息的Header，一个是消息的Content。Header为4个uint32_t类型变量，从前到后这4个变量的含义分别为sequence number，message number，timestamp和目标SocketID。就Handshake而言，sequence number的最高位，也就是bit-0为1，表示这是一个控制消息，bit-1~15为pkttype 0，其它位为0；message number及timestamp均为0，目标SocketID为0。

Content部分，总共48个字节，主要用于进行连接的协商，如MSS等，具体可以看CHandShake。

9. 检查是否是同步接收模式。如果不是的话，则delete掉前面为request CPacket的CHandShake创建的缓冲区并退出。后面与UDT Server端进一步的消息交互会有接收队列等帮忙异步地推动。否则继续执行。值得一提的是，CUDT在其构造函数中，会将m_bSynRecving置为true，但在拷贝构造函数中，则会继承传入的值。但这个值如同MSS值一样，也可以通过UDT::setOpt()设置。也就是说由应用程序创建的UDT Socket默认处于同步接收模式，比如Listening的UDT Socket和发起连接的UDT Socket，但可以自行设置，由Listening的UDT Socket在接收到连接建立请求时创建的UDT Socket，则会继承Listening UDT Socket的对应值。

我们暂时先看SynRecving模式，也就是默认模式下的UDT Socket的行为。

10. 创建一个CPacket response，同样为它创建一个大小为m_iPayloadSize的缓冲区以存放数据，并将缓冲区pack进response中。这个CPacket response会被用来存放从UDT Server发回的相应的信息。

11. 进入一个循环执行后续的握手动作，及消息的超时重传等动作。可以将这个循环看做由3个部分组成。

循环开始的地方是一段发送消息的代码，在这段代码中，其实做了两个事情，或者说可能会发送两种类型的消息，一是第一个握手消息的超时重传，二是第二个握手消息的发送及超时重传。看上去发送的都是CHandShake m_ConnReq，但在接收到第一个握手消息的响应之后，这个结构的某些成员会根据响应而被修改。注意，发送第一个握手消息之后，首次进入循环，将会跳过这个部分。

之后的第二部分，主要用于接收响应，第一个握手消息的响应及第二个握手消息的响应。来看CRcvQueue::recvfrom()(src/queue.cpp)：

```
int CRcvQueue::recvfrom(int32_t id, CPacket& packet) {
    CGuard bufferlock(m_PassLock);

    map<int32_t, std::queue<CPacket*> >::iterator i = m_mBuffer.find(id);

    if (i == m_mBuffer.end()) {
#ifndef WIN32
        uint64_t now = CTimer::getTime();
        timespec timeout;

        timeout.tv_sec = now / 1000000 + 1;
        timeout.tv_nsec = (now % 1000000) * 1000;

        pthread_cond_timedwait(&m_PassCond, &m_PassLock, &timeout);
#else
        ReleaseMutex(m_PassLock);
        WaitForSingleObject(m_PassCond, 1000);
        WaitForSingleObject(m_PassLock, INFINITE);
#endif

        i = m_mBuffer.find(id);
        if (i == m_mBuffer.end()) {
            packet.setLength(-1);
            return -1;
        }
    }

    // retrieve the earliest packet
    CPacket* newpkt = i->second.front();

    if (packet.getLength() < newpkt->getLength()) {
        packet.setLength(-1);
        return -1;
    }

    // copy packet content
    memcpy(packet.m_nHeader, newpkt->m_nHeader, CPacket::m_iPktHdrSize);
    memcpy(packet.m_pcData, newpkt->m_pcData, newpkt->getLength());
    packet.setLength(newpkt->getLength());

    delete[] newpkt->m_pcData;
    delete newpkt;

    // remove this message from queue,
    // if no more messages left for this socket, release its data structure
    i->second.pop();
    if (i->second.empty())
        m_mBuffer.erase(i);

    return packet.getLength();
}
```

这也是一个生产者-消费者模型，在这里就如同listen的过程一样，也只能看到这个生产与消费的故事的一半，即消费的那一半。生产者也是RcvQueue的worker线程。这个地方会等待着消息的到来，但也不会无限制的等待，可以看到，这里接收消息的等待时间大概为1s。这里是在等待一个CPacket队列的出现，也就是m_mBuffer中目标UDT Socket的CPacket队列。这里会从这个队列中取出第一个packet返回给调用者。如果队列被取空了，会直接将这个队列从m_mBuffer中移除出去。

循环的第三部分是整个连接建立消息交互过程的超时处理，可以看到，非Rendezvous模式下超时时间为3s，Rendezvous模式下，超时时间则会延长十倍。

CUDT::connect()执行到接收第一个握手消息的相应时，连接建立请求的发起也算是基本完成了。下面来看UDT Server端收到这个消息时是如何处理的。

# UDT Server对首个Handshake消息的处理
来看UDT Server端收到这个消息时是如何处理的。如我们前面在 [UDT协议实现分析——bind、listen与accept](http://my.oschina.net/wolfcs/blog/503959) 一文中了解到的，Listening的UDT Socket会在UDT::accept()中等待连接请求进来，那是一个生产者与消费者的故事，UDT::accept()是生产者，接收队列RcvQueue的worker线程是消费者。

我们这就来仔细地看一下RcvQueue的worker线程，当然重点会关注对于Handshake消息，也就是目标SocketID为0，pkttype为0的packet的处理(src/queue.cpp)：

```
#ifndef WIN32
void* CRcvQueue::worker(void* param)
#else
        DWORD WINAPI CRcvQueue::worker(LPVOID param)
#endif
        {
    CRcvQueue* self = (CRcvQueue*) param;

    sockaddr* addr =
            (AF_INET == self->m_UnitQueue.m_iIPversion) ? (sockaddr*) new sockaddr_in : (sockaddr*) new sockaddr_in6;
    CUDT* u = NULL;
    int32_t id;

    while (!self->m_bClosing) {
#ifdef NO_BUSY_WAITING
        self->m_pTimer->tick();
#endif

        // check waiting list, if new socket, insert it to the list
        while (self->ifNewEntry()) {
            CUDT* ne = self->getNewEntry();
            if (NULL != ne) {
                self->m_pRcvUList->insert(ne);
                self->m_pHash->insert(ne->m_SocketID, ne);
            }
        }

        // find next available slot for incoming packet
        CUnit* unit = self->m_UnitQueue.getNextAvailUnit();
        if (NULL == unit) {
            // no space, skip this packet
            CPacket temp;
            temp.m_pcData = new char[self->m_iPayloadSize];
            temp.setLength(self->m_iPayloadSize);
            self->m_pChannel->recvfrom(addr, temp);
            delete[] temp.m_pcData;
            goto TIMER_CHECK;
        }

        unit->m_Packet.setLength(self->m_iPayloadSize);

        // reading next incoming packet, recvfrom returns -1 is nothing has been received
        if (self->m_pChannel->recvfrom(addr, unit->m_Packet) < 0)
            goto TIMER_CHECK;

        id = unit->m_Packet.m_iID;

        // ID 0 is for connection request, which should be passed to the listening socket or rendezvous sockets
        if (0 == id) {
            if (NULL != self->m_pListener)
                self->m_pListener->listen(addr, unit->m_Packet);
            else if (NULL != (u = self->m_pRendezvousQueue->retrieve(addr, id))) {
                // asynchronous connect: call connect here
                // otherwise wait for the UDT socket to retrieve this packet
                if (!u->m_bSynRecving)
                    u->connect(unit->m_Packet);
                else
                    self->storePkt(id, unit->m_Packet.clone());
            }
        } else if (id > 0) {
            if (NULL != (u = self->m_pHash->lookup(id))) {
                if (CIPAddress::ipcmp(addr, u->m_pPeerAddr, u->m_iIPversion)) {
                    if (u->m_bConnected && !u->m_bBroken && !u->m_bClosing) {
                        if (0 == unit->m_Packet.getFlag())
                            u->processData(unit);
                        else
                            u->processCtrl(unit->m_Packet);

                        u->checkTimers();
                        self->m_pRcvUList->update(u);
                    }
                }
            } else if (NULL != (u = self->m_pRendezvousQueue->retrieve(addr, id))) {
                if (!u->m_bSynRecving)
                    u->connect(unit->m_Packet);
                else
                    self->storePkt(id, unit->m_Packet.clone());
            }
        }

        TIMER_CHECK:
        // take care of the timing event for all UDT sockets

        uint64_t currtime;
        CTimer::rdtsc(currtime);

        CRNode* ul = self->m_pRcvUList->m_pUList;
        uint64_t ctime = currtime - 100000 * CTimer::getCPUFrequency();
        while ((NULL != ul) && (ul->m_llTimeStamp < ctime)) {
            CUDT* u = ul->m_pUDT;

            if (u->m_bConnected && !u->m_bBroken && !u->m_bClosing) {
                u->checkTimers();
                self->m_pRcvUList->update(u);
            } else {
                // the socket must be removed from Hash table first, then RcvUList
                self->m_pHash->remove(u->m_SocketID);
                self->m_pRcvUList->remove(u);
                u->m_pRNode->m_bOnList = false;
            }

            ul = self->m_pRcvUList->m_pUList;
        }

        // Check connection requests status for all sockets in the RendezvousQueue.
        self->m_pRendezvousQueue->updateConnStatus();
    }

    if (AF_INET == self->m_UnitQueue.m_iIPversion)
        delete (sockaddr_in*) addr;
    else
        delete (sockaddr_in6*) addr;

#ifndef WIN32
    return NULL;
#else
    SetEvent(self->m_ExitCond);
    return 0;
#endif
}
```

这个函数，首先创建了一个sockaddr，用于保存发送端的地址。

然后就进入了一个循环，不断地接收UDP消息。

循环内的第一行是执行Timer的tick()，这个是UDT自己的定时器Timer机制的一部分。

接下来的这个子循环也主要与RcvQueue的worker线程中消息的dispatch机制有关。

然后是取一个CUnit，用来接收其它端点发送过来的消息。如果取不到，则接收UDP包并丢弃。然后跳过后面消息dispatch的过程。这个地方的m_UnitQueue用来做缓存，也用来防止收到过多的包消耗过多的资源。完整的CUnitQueue机制暂时先不去仔细分析。

然后就是取到了CUnit的情况，则先通过CChannel接收一个包，并根据包的内容进行包的dispatch。不能跑偏了，这里主要关注目标SocketID为0，pkttype为0的包的dispatch。可以看到，在Listener存在的情况下，是dispatch给了listener，也就是Listening的UDT Socket的CUDT的listen()函数，否则会dispatch给通道上处于Rendezvous模式的UDT Socket。（在 [UDT协议实现分析——bind、listen与accept](http://my.oschina.net/wolfcs/blog/503959) 一文中关于listen的部分有具体理过这个listener的设置过程。）可以看到，对于相同的通道CChannel，也就是同一个端口上，Rendezvous模式下的UDT Socket和Listening的UDT Socket不能共存，或者说同时存在时，Rendezvous的行为可能不是预期的，但多个处于Rendezvous模式下的UDT Socket可以共存。

接收队列CRcvQueue的worker()线程做的其它事情，暂时先不去仔细看。这里先来理一下Listening的UDT Socket在接收到Handshake消息的处理过程，也就是CUDT::listen(sockaddr* addr, CPacket& packet)(src/core.cpp)：

```
int CUDT::listen(sockaddr* addr, CPacket& packet) {
    if (m_bClosing)
        return 1002;

    if (packet.getLength() != CHandShake::m_iContentSize)
        return 1004;

    CHandShake hs;
    hs.deserialize(packet.m_pcData, packet.getLength());

    // SYN cookie
    char clienthost[NI_MAXHOST];
    char clientport[NI_MAXSERV];
    getnameinfo(addr, (AF_INET == m_iVersion) ? sizeof(sockaddr_in) : sizeof(sockaddr_in6), clienthost,
                sizeof(clienthost), clientport, sizeof(clientport), NI_NUMERICHOST | NI_NUMERICSERV);
    int64_t timestamp = (CTimer::getTime() - m_StartTime) / 60000000;  // secret changes every one minute
    stringstream cookiestr;
    cookiestr << clienthost << ":" << clientport << ":" << timestamp;
    unsigned char cookie[16];
    CMD5::compute(cookiestr.str().c_str(), cookie);

    if (1 == hs.m_iReqType) {
        hs.m_iCookie = *(int*) cookie;
        packet.m_iID = hs.m_iID;
        int size = packet.getLength();
        hs.serialize(packet.m_pcData, size);
        m_pSndQueue->sendto(addr, packet);
        return 0;
    } else {
        if (hs.m_iCookie != *(int*) cookie) {
            timestamp--;
            cookiestr << clienthost << ":" << clientport << ":" << timestamp;
            CMD5::compute(cookiestr.str().c_str(), cookie);

            if (hs.m_iCookie != *(int*) cookie)
                return -1;
        }
    }

    int32_t id = hs.m_iID;

    // When a peer side connects in...
    if ((1 == packet.getFlag()) && (0 == packet.getType())) {
        if ((hs.m_iVersion != m_iVersion) || (hs.m_iType != m_iSockType)) {
            // mismatch, reject the request
            hs.m_iReqType = 1002;
            int size = CHandShake::m_iContentSize;
            hs.serialize(packet.m_pcData, size);
            packet.m_iID = id;
            m_pSndQueue->sendto(addr, packet);
        } else {
            int result = s_UDTUnited.newConnection(m_SocketID, addr, &hs);
            if (result == -1)
                hs.m_iReqType = 1002;

            // send back a response if connection failed or connection already existed
            // new connection response should be sent in connect()
            if (result != 1) {
                int size = CHandShake::m_iContentSize;
                hs.serialize(packet.m_pcData, size);
                packet.m_iID = id;
                m_pSndQueue->sendto(addr, packet);
            } else {
                // a new connection has been created, enable epoll for write
                s_UDTUnited.m_EPoll.update_events(m_SocketID, m_sPollID, UDT_EPOLL_OUT, true);
            }
        }
    }

    return hs.m_iReqType;
}
```
在这个函数中主要做了这样的一些事情：

1. 检查UDT Socket的状态，如果处于Closing状态下，就返回，否则继续执行。

2. 检查包的数据部分长度。若长度不为CHandShake::m_iContentSize 48字节，则说明这不是一个有效的Handshake，则返回，否则继续执行。

3. 创建一个CHandShake hs，并将传入的packet的数据部分反序列化进这个CHandShake。这里来扫一眼这个CHandShake::deserialize()(src/packet.cpp)：
```
int CHandShake::deserialize(const char* buf, int size) {
    if (size < m_iContentSize)
        return -1;

    int32_t* p = (int32_t*) buf;
    m_iVersion = *p++;
    m_iType = *p++;
    m_iISN = *p++;
    m_iMSS = *p++;
    m_iFlightFlagSize = *p++;
    m_iReqType = *p++;
    m_iID = *p++;
    m_iCookie = *p++;
    for (int i = 0; i < 4; ++i)
        m_piPeerIP[i] = *p++;

    return 0;
}
```

这个函数如同它的反函数serialize()一样没有处理字节序的问题。

4. 计算cookie值。所谓cookie值，即由连接发起端的网络地址(包括IP地址与端口号)及时间戳组成的字符串计算出来的16个字节长度的MD5值。时间戳精确到分钟值。用于计算MD5值的字符串类似127.0.0.1:49033:0。

5. 计算出来cookie值之后的部分，应该被分成两个部分。一部分处理连接发起端发送的地一个握手包，也就是hs.m_iReqType == 1的block，在CUDT::connect()中构造m_ConnReq的部分我们有看到这个值要被设为1的；另一部分则处理连接发起端发送的第二个握手消息。这里我们先来看hs.m_iReqType == 1的block。

它取前一步计算的cookie的前4个字节，直接将其强转为一个int值，赋给前面反序列化的CHandShake的m_iCookie。这个地方竟然顾及字节序的问题，也没有顾及不同平台的差异，即int类型的长度在不同的机器上可能不同，这个地方用int32_t似乎要更安全一点。将CHandShake的m_iID，如我们在CUDT::connect()中构造m_ConnReq的部分我们有看到的，为连接发起端UDT Socket的SocketID，设置给packet的m_iID，也就是包的目标SocketID。再将hs重新序列化进packet。通过发送队列SndQueue发送经过了这一番修改的packet。然后返回。

总结一下UDT Server中Listening的UDT Socket接收到第一个HandShake包时，对于这个包的处理过程：

计算一个cookie值，设置给接收到的HandShake的cookie字段，修改包的目标SocketID字段为发起连接的UDT Socket的SocketID，包的其它部分原封不动，最后将这个包重新发回给连接发起端。

# UDT Client发送第二个HandShake消息

UDT Server接收到第一个HandShake消息，回给UDT Client一个HandShake消息。这样球就又被踢回给了UDT Client端。接着来看在UDT Client端接收到首个HandShake包的响应后会做什么样的处理。

我们知道在CUDT::connect(const sockaddr* serv_addr)中，发送首个HandShake包之后，会调用CRcvQueue::recvfrom()来等着接收UDT Server的响应，消费者焦急地等待着食物的到来。在消息到来时，CUDT::connect()会被生产者，也就是CRcvQueue的worker线程唤醒。这里就来具体看一下这个生产与消费的故事的另一半，生产的故事，也就是CRcvQueue的worker线程的消息dispatch。

在CRcvQueue::worker()中包dispatch的部分可以看到：

```
} else if (id > 0) {
            if (NULL != (u = self->m_pHash->lookup(id))) {
                if (CIPAddress::ipcmp(addr, u->m_pPeerAddr, u->m_iIPversion)) {
                    cout << "Receive packet by m_pHash table" << endl;
                    if (u->m_bConnected && !u->m_bBroken && !u->m_bClosing) {
                        if (0 == unit->m_Packet.getFlag())
                            u->processData(unit);
                        else
                            u->processCtrl(unit->m_Packet);

                        u->checkTimers();
                        self->m_pRcvUList->update(u);
                    }
                }
            } else if (NULL != (u = self->m_pRendezvousQueue->retrieve(addr, id))) {
                cout << "Receive packet by m_pRendezvousQueue, u->m_bSynRecving " << u->m_bSynRecving << endl;
                if (!u->m_bSynRecving)
                    u->connect(unit->m_Packet);
                else
                    self->storePkt(id, unit->m_Packet.clone());
            }
        }
```
我们知道UDT Server回复的消息中是设置了目标SocketID了的。因而会走id > 0的block。

在CUDT::connect( const sockaddr* serv_addr )中有看到调用m_pRcvQueue->registerConnector()将CUDT添加进RcvQueue的m_pRendezvousQueue中，因而这里会执行id > 0 block中下面的那个block。

如果前面对于m_bSynRecving的分析，默认情况为true。因而这个地方会执行CRcvQueue::storePkt()来存储包。来看这个函数的实现：

```
void CRcvQueue::storePkt(int32_t id, CPacket* pkt) {
    CGuard bufferlock(m_PassLock);

    map<int32_t, std::queue<CPacket*> >::iterator i = m_mBuffer.find(id);

    if (i == m_mBuffer.end()) {
        m_mBuffer[id].push(pkt);

#ifndef WIN32
        pthread_cond_signal(&m_PassCond);
#else
        SetEvent(m_PassCond);
#endif
    } else {
        //avoid storing too many packets, in case of malfunction or attack
        if (i->second.size() > 16)
            return;

        i->second.push(pkt);
    }
}
```
在这个函数中会保存接收到的packet，并在必要的时候唤醒等待接收消息的线程。(对应CRcvQueue::recvfrom()的逻辑来看。)

然后来看CUDT::connect(const sockaddr* serv_addr)在收到第一个HandShake消息的响应之后会做什么样的处理，也就是CUDT::connect(const CPacket& response)(src/core.cpp)：
```
int CUDT::connect(const CPacket& response) throw () {
    // this is the 2nd half of a connection request. If the connection is setup successfully this returns 0.
    // returning -1 means there is an error.
    // returning 1 or 2 means the connection is in process and needs more handshake

    if (!m_bConnecting)
        return -1;

    if (m_bRendezvous && ((0 == response.getFlag()) || (1 == response.getType())) && (0 != m_ConnRes.m_iType)) {
        //a data packet or a keep-alive packet comes, which means the peer side is already connected
        // in this situation, the previously recorded response will be used
        goto POST_CONNECT;
    }

    if ((1 != response.getFlag()) || (0 != response.getType()))
        return -1;

    m_ConnRes.deserialize(response.m_pcData, response.getLength());

    if (m_bRendezvous) {
        // regular connect should NOT communicate with rendezvous connect
        // rendezvous connect require 3-way handshake
        if (1 == m_ConnRes.m_iReqType)
            return -1;

        if ((0 == m_ConnReq.m_iReqType) || (0 == m_ConnRes.m_iReqType)) {
            m_ConnReq.m_iReqType = -1;
            // the request time must be updated so that the next handshake can be sent out immediately.
            m_llLastReqTime = 0;
            return 1;
        }
    } else {
        // set cookie
        if (1 == m_ConnRes.m_iReqType) {
            m_ConnReq.m_iReqType = -1;
            m_ConnReq.m_iCookie = m_ConnRes.m_iCookie;
            m_llLastReqTime = 0;
            return 1;
        }
    }
```
这个函数会处理第一个HandShake的响应，也会处理第二个HandShake的响应，这里先来关注第一个HandShake的响应的处理，因而只列出它的一部分的代码。

这个函数先是检查了CUDT的状态，检查了packet的有效性，然后就是将接收到的包的数据部分反序列化至CHandShake m_ConnRes中。我们不关注对于Rendezvous模式的处理。

接着会检查m_ConnRes的m_iReqType，若为1，则设置m_ConnReq.m_iReqType为-1，设置m_ConnReq.m_iCookie为m_ConnRes.m_iCookie用以标识m_ConnReq为一个合法的第二个HandShake packet；同时设置m_llLastReqTime为0，如我们前面对CUDT::connect(const sockaddr* serv_addr)的分析，以便于此刻保存于m_ConnReq中的第二个HandShake能够被发送出去as soon as possible。

这第二个HandShake，与第一个HandShake的差异仅仅在于有了有效的Cookie值，且请求类型ReqType为-1。其它则完全一样。

# UDT Server对第二个HandShake的处理

UDT Client对于m_ConnReq的改变并不足以改变接收队列中worker线程对这个包的dispatch规则，因而直接来看CUDT::listen(sockaddr* addr, CPacket& packet)中对于这第二个HandShake消息的处理。

接着前面对于这个函数的分析，接前面的第4步。

5. 对于这第二个HandShake，它的ReqType自然不再是1了，而是-1。因而在计算完了cookie值之后，它会先验证一下HandShake包中的cookie值是否是有效的，如果无效，则直接返回。根据这个地方的逻辑，可以看到cookie的有效时间最长为2分钟。

6. 检查包的Flag和Type，如果不是HandShake包，则直接返回，否则继续执行。

7. 检查连接发起端IP的版本及Socket类型SockType与本地Listen的UDT Socket是否匹配。若不匹配，则将错误码1002放在发过来的HandShanke的ReqType字段中，设置packet的目标SocketID为发起连接的SocketID，然后将这个包重新发回给UDT Client。

8. 检查之后，发现完全匹配的情况。调用CUDTUnited::newConnection()创建一个新的UDT Socket。若创建过程执行失败，则将错误码1002放在发过来的HandShanke的ReqType字段中。若创建成功，会设置发过来的packet的目标SocketID为适当的值，然后将同一个包再发送回UDT Client。CUDTUnited::newConnection()会适当地修改HandShake packet的一些字段。若失败在执行s_UDTUnited.m_EPoll.update_events()。

9. 返回hs.m_iReqType。

然后来看在CUDTUnited::newConnection()中是如何新建Socket的：
```
int CUDTUnited::newConnection(const UDTSOCKET listen, const sockaddr* peer, CHandShake* hs) {
    CUDTSocket* ns = NULL;
    CUDTSocket* ls = locate(listen);

    if (NULL == ls)
        return -1;

    // if this connection has already been processed
    if (NULL != (ns = locate(peer, hs->m_iID, hs->m_iISN))) {
        if (ns->m_pUDT->m_bBroken) {
            // last connection from the "peer" address has been broken
            ns->m_Status = CLOSED;
            ns->m_TimeStamp = CTimer::getTime();

            CGuard::enterCS(ls->m_AcceptLock);
            ls->m_pQueuedSockets->erase(ns->m_SocketID);
            ls->m_pAcceptSockets->erase(ns->m_SocketID);
            CGuard::leaveCS(ls->m_AcceptLock);
        } else {
            // connection already exist, this is a repeated connection request
            // respond with existing HS information

            hs->m_iISN = ns->m_pUDT->m_iISN;
            hs->m_iMSS = ns->m_pUDT->m_iMSS;
            hs->m_iFlightFlagSize = ns->m_pUDT->m_iFlightFlagSize;
            hs->m_iReqType = -1;
            hs->m_iID = ns->m_SocketID;

            return 0;

            //except for this situation a new connection should be started
        }
    }

    // exceeding backlog, refuse the connection request
    if (ls->m_pQueuedSockets->size() >= ls->m_uiBackLog)
        return -1;

    try {
        ns = new CUDTSocket;
        ns->m_pUDT = new CUDT(*(ls->m_pUDT));
        if (AF_INET == ls->m_iIPversion) {
            ns->m_pSelfAddr = (sockaddr*) (new sockaddr_in);
            ((sockaddr_in*) (ns->m_pSelfAddr))->sin_port = 0;
            ns->m_pPeerAddr = (sockaddr*) (new sockaddr_in);
            memcpy(ns->m_pPeerAddr, peer, sizeof(sockaddr_in));
        } else {
            ns->m_pSelfAddr = (sockaddr*) (new sockaddr_in6);
            ((sockaddr_in6*) (ns->m_pSelfAddr))->sin6_port = 0;
            ns->m_pPeerAddr = (sockaddr*) (new sockaddr_in6);
            memcpy(ns->m_pPeerAddr, peer, sizeof(sockaddr_in6));
        }
    } catch (...) {
        delete ns;
        return -1;
    }

    CGuard::enterCS(m_IDLock);
    ns->m_SocketID = --m_SocketID;
    cout << "new CUDTSocket SocketID is " << ns->m_SocketID << " PeerID " << hs->m_iID << endl;
    CGuard::leaveCS(m_IDLock);

    ns->m_ListenSocket = listen;
    ns->m_iIPversion = ls->m_iIPversion;
    ns->m_pUDT->m_SocketID = ns->m_SocketID;
    ns->m_PeerID = hs->m_iID;
    ns->m_iISN = hs->m_iISN;

    int error = 0;

    try {
        // bind to the same addr of listening socket
        ns->m_pUDT->open();
        updateMux(ns, ls);
        ns->m_pUDT->connect(peer, hs);
    } catch (...) {
        error = 1;
        goto ERR_ROLLBACK;
    }

    ns->m_Status = CONNECTED;

    // copy address information of local node
    ns->m_pUDT->m_pSndQueue->m_pChannel->getSockAddr(ns->m_pSelfAddr);
    CIPAddress::pton(ns->m_pSelfAddr, ns->m_pUDT->m_piSelfIP, ns->m_iIPversion);

    // protect the m_Sockets structure.
    CGuard::enterCS(m_ControlLock);
    try {
        m_Sockets[ns->m_SocketID] = ns;
        m_PeerRec[(ns->m_PeerID << 30) + ns->m_iISN].insert(ns->m_SocketID);
    } catch (...) {
        error = 2;
    }
    CGuard::leaveCS(m_ControlLock);

    CGuard::enterCS(ls->m_AcceptLock);
    try {
        ls->m_pQueuedSockets->insert(ns->m_SocketID);
    } catch (...) {
        error = 3;
    }
    CGuard::leaveCS(ls->m_AcceptLock);

    // acknowledge users waiting for new connections on the listening socket
    m_EPoll.update_events(listen, ls->m_pUDT->m_sPollID, UDT_EPOLL_IN, true);

    CTimer::triggerEvent();

    ERR_ROLLBACK: if (error > 0) {
        ns->m_pUDT->close();
        ns->m_Status = CLOSED;
        ns->m_TimeStamp = CTimer::getTime();

        return -1;
    }

    // wake up a waiting accept() call
#ifndef WIN32
    pthread_mutex_lock(&(ls->m_AcceptLock));
    pthread_cond_signal(&(ls->m_AcceptCond));
    pthread_mutex_unlock(&(ls->m_AcceptLock));
#else
    SetEvent(ls->m_AcceptCond);
#endif

    return 1;
}
```
在这个函数中做了如下这样的一些事情：

1. 找到listening的UDT Socket的CUDTSocket结构，若找不到则直接返回-1。否则继续执行。

2. 检查相同的连接请求是否已经处理过了。在CUDTUnited有一个专门的缓冲区m_PeerRec，用来存放由Listening的Socket创建的UDT Socket，这里主要是通过在这个缓冲区中查找是否已经有connection请求对应的socket来判断：

```
CUDTSocket* CUDTUnited::locate(const sockaddr* peer, const UDTSOCKET id, int32_t isn) {
    CGuard cg(m_ControlLock);

    map<int64_t, set<UDTSOCKET> >::iterator i = m_PeerRec.find((id << 30) + isn);
    if (i == m_PeerRec.end())
        return NULL;

    for (set<UDTSOCKET>::iterator j = i->second.begin(); j != i->second.end(); ++j) {
        map<UDTSOCKET, CUDTSocket*>::iterator k = m_Sockets.find(*j);
        // this socket might have been closed and moved m_ClosedSockets
        if (k == m_Sockets.end())
            continue;

        if (CIPAddress::ipcmp(peer, k->second->m_pPeerAddr, k->second->m_iIPversion))
            return k->second;
    }

    return NULL;
}
```
如果已经为这个connection请求创建了UDT Socket，又分为两种情况：

(1). 为connection请求创建的UDT Socket还是好的，可用的，则根据之前创建的UDT Socket的一些字段设置接收到的HandShake，m_iReqType会被设置为-1，m_iID会被设置为UDT Socket的SocketID。然后返回0。如我们前面在CUDTUnited::newConnection()中看到的，这样返回之后，CUDTUnited::newConnection()会发送一个响应消息给UDT Client。

(2). 为connection请求创建的UDT Socket已经烂掉了，不可用了，此时则主要会将其状态设置为CLOSED，设置时间戳，将其从m_pQueuedSockets和m_pAcceptSockets中移除出去。然后执行后续的新建UDT Socket的流程。

但对于一个由Listening Socket创建的UDT Socket而言，又会是什么原因导致它处于broken状态呢？此处这样的检查是否真有必要呢？后面会再来研究。

3. 检查m_pQueuedSockets的大小是否超出了为Listening的UDT Socket设置的backlog大小，若超出，则返回-1，否则继续执行。

4. 创建一个CUDTSocket对象。创建一个CUDT对象，这里创建的CUDT对象会继承Listening的UDT Socket的许多属性(src/api.cpp)：

```
CUDT::CUDT(const CUDT& ancestor) {
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
    m_iMSS = ancestor.m_iMSS;
    m_bSynSending = ancestor.m_bSynSending;
    m_bSynRecving = ancestor.m_bSynRecving;
    m_iFlightFlagSize = ancestor.m_iFlightFlagSize;
    m_iSndBufSize = ancestor.m_iSndBufSize;
    m_iRcvBufSize = ancestor.m_iRcvBufSize;
    m_Linger = ancestor.m_Linger;
    m_iUDPSndBufSize = ancestor.m_iUDPSndBufSize;
    m_iUDPRcvBufSize = ancestor.m_iUDPRcvBufSize;
    m_iSockType = ancestor.m_iSockType;
    m_iIPversion = ancestor.m_iIPversion;
    m_bRendezvous = ancestor.m_bRendezvous;
    m_iSndTimeOut = ancestor.m_iSndTimeOut;
    m_iRcvTimeOut = ancestor.m_iRcvTimeOut;
    m_bReuseAddr = true;  // this must be true, because all accepted sockets shared the same port with the listener
    m_llMaxBW = ancestor.m_llMaxBW;

    m_pCCFactory = ancestor.m_pCCFactory->clone();
    m_pCC = NULL;
    m_pCache = ancestor.m_pCache;

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
```
为SelfAddr分配内存。

为PeerAddr分配内存。

拷贝发送端地址到PeerAddr。

设置SocketID。等等。

5. 执行ns->m_pUDT->open()完成打开动作。然后执行updateMux(ns, ls)，将新建的这个UDT Socket绑定到Listening的UDT Socket所绑定的多路复用器：

```
void CUDTUnited::updateMux(CUDTSocket* s, const CUDTSocket* ls) {
    CGuard cg(m_ControlLock);

    int port = (AF_INET == ls->m_iIPversion) ?
                    ntohs(((sockaddr_in*) ls->m_pSelfAddr)->sin_port) :
                    ntohs(((sockaddr_in6*) ls->m_pSelfAddr)->sin6_port);

    // find the listener's address
    for (map<int, CMultiplexer>::iterator i = m_mMultiplexer.begin(); i != m_mMultiplexer.end(); ++i) {
        if (i->second.m_iPort == port) {
            // reuse the existing multiplexer
            ++i->second.m_iRefCount;
            s->m_pUDT->m_pSndQueue = i->second.m_pSndQueue;
            s->m_pUDT->m_pRcvQueue = i->second.m_pRcvQueue;
            s->m_iMuxID = i->second.m_iID;
            return;
        }
    }
}
```
6. 执行 ns->m_pUDT->connect(peer, hs)：

```
void CUDT::connect(const sockaddr* peer, CHandShake* hs) {
    CGuard cg(m_ConnectionLock);

    // Uses the smaller MSS between the peers
    if (hs->m_iMSS > m_iMSS)
        hs->m_iMSS = m_iMSS;
    else
        m_iMSS = hs->m_iMSS;

    // exchange info for maximum flow window size
    m_iFlowWindowSize = hs->m_iFlightFlagSize;
    hs->m_iFlightFlagSize = (m_iRcvBufSize < m_iFlightFlagSize) ? m_iRcvBufSize : m_iFlightFlagSize;

    m_iPeerISN = hs->m_iISN;

    m_iRcvLastAck = hs->m_iISN;
    m_iRcvLastAckAck = hs->m_iISN;
    m_iRcvCurrSeqNo = hs->m_iISN - 1;

    m_PeerID = hs->m_iID;
    hs->m_iID = m_SocketID;

    // use peer's ISN and send it back for security check
    m_iISN = hs->m_iISN;

    m_iLastDecSeq = m_iISN - 1;
    m_iSndLastAck = m_iISN;
    m_iSndLastDataAck = m_iISN;
    m_iSndCurrSeqNo = m_iISN - 1;
    m_iSndLastAck2 = m_iISN;
    m_ullSndLastAck2Time = CTimer::getTime();

    // this is a reponse handshake
    hs->m_iReqType = -1;

    // get local IP address and send the peer its IP address (because UDP cannot get local IP address)
    memcpy(m_piSelfIP, hs->m_piPeerIP, 16);
    CIPAddress::ntop(peer, hs->m_piPeerIP, m_iIPversion);

    m_iPktSize = m_iMSS - 28;
    m_iPayloadSize = m_iPktSize - CPacket::m_iPktHdrSize;

    // Prepare all structures
    try {
        m_pSndBuffer = new CSndBuffer(32, m_iPayloadSize);
        m_pRcvBuffer = new CRcvBuffer(&(m_pRcvQueue->m_UnitQueue), m_iRcvBufSize);
        m_pSndLossList = new CSndLossList(m_iFlowWindowSize * 2);
        m_pRcvLossList = new CRcvLossList(m_iFlightFlagSize);
        m_pACKWindow = new CACKWindow(1024);
        m_pRcvTimeWindow = new CPktTimeWindow(16, 64);
        m_pSndTimeWindow = new CPktTimeWindow();
    } catch (...) {
        throw CUDTException(3, 2, 0);
    }

    CInfoBlock ib;
    ib.m_iIPversion = m_iIPversion;
    CInfoBlock::convert(peer, m_iIPversion, ib.m_piIP);
    if (m_pCache->lookup(&ib) >= 0) {
        m_iRTT = ib.m_iRTT;
        m_iBandwidth = ib.m_iBandwidth;
    }

    m_pCC = m_pCCFactory->create();
    m_pCC->m_UDT = m_SocketID;
    m_pCC->setMSS(m_iMSS);
    m_pCC->setMaxCWndSize(m_iFlowWindowSize);
    m_pCC->setSndCurrSeqNo(m_iSndCurrSeqNo);
    m_pCC->setRcvRate(m_iDeliveryRate);
    m_pCC->setRTT(m_iRTT);
    m_pCC->setBandwidth(m_iBandwidth);
    m_pCC->init();

    m_ullInterval = (uint64_t) (m_pCC->m_dPktSndPeriod * m_ullCPUFrequency);
    m_dCongestionWindow = m_pCC->m_dCWndSize;

    m_pPeerAddr = (AF_INET == m_iIPversion) ? (sockaddr*) new sockaddr_in : (sockaddr*) new sockaddr_in6;
    memcpy(m_pPeerAddr, peer, (AF_INET == m_iIPversion) ? sizeof(sockaddr_in) : sizeof(sockaddr_in6));

    // And of course, it is connected.
    m_bConnected = true;

    // register this socket for receiving data packets
    m_pRNode->m_bOnList = true;
    m_pRcvQueue->setNewEntry(this);

    //send the response to the peer, see listen() for more discussions about this
    CPacket response;
    int size = CHandShake::m_iContentSize;
    char* buffer = new char[size];
    hs->serialize(buffer, size);
    response.pack(0, NULL, buffer, size);
    response.m_iID = m_PeerID;
    m_pSndQueue->sendto(peer, response);
    delete[] buffer;
}
```
这个函数里会根据HandShake包设置非常多的成员。但主要来关注m_pRcvQueue->setNewEntry(this)，这个调用也是与RcvQueue的worker线程的消息dispatch机制有关。后面我们会再来仔细地了解这个函数。

这个函数会在最后发送响应给UDT Client。

7. 将UDT Socket的状态置为CONNECTED。拷贝Channel的地址到PeerAddr。

8. 将创建的CUDTSocket放进m_Sockets中，同时放进m_PeerRec中。

9. 将创建的UDT Socket放进m_pQueuedSockets中。这正是Listening UDT Socket accept那个生产-消费故事的另一半，这里是生产者。

10. 将等待在accept()的线程唤醒。至此在UDT Server端，accept()返回一个UDT Socket，UDT Server认为一个连接成功建立。

# UDT Client从UDT::connect()返回

如我们前面看到的，CUDT::connect(const sockaddr* serv_addr)在发送了第二个Handshake消息之后，它就会开是等待UDT Server的第二次响应。UDT Server发送第二个Handshake消息的相应之后，UDT Client端将会返回并处理它。这个消息的dispatch过程与第一个HandShake的响应消息的处理过程一致，这里不再赘述。这里来看这第二个HandShake的响应消息的处理，同样是在CUDT::connect(const CPacket& response)中：

```
} else {
        // set cookie
        if (1 == m_ConnRes.m_iReqType) {
            m_ConnReq.m_iReqType = -1;
            m_ConnReq.m_iCookie = m_ConnRes.m_iCookie;
            m_llLastReqTime = 0;
            return 1;
        }
    }

    POST_CONNECT:
    // Remove from rendezvous queue
    m_pRcvQueue->removeConnector(m_SocketID);

    // Re-configure according to the negotiated values.
    m_iMSS = m_ConnRes.m_iMSS;
    m_iFlowWindowSize = m_ConnRes.m_iFlightFlagSize;
    m_iPktSize = m_iMSS - 28;
    m_iPayloadSize = m_iPktSize - CPacket::m_iPktHdrSize;
    m_iPeerISN = m_ConnRes.m_iISN;
    m_iRcvLastAck = m_ConnRes.m_iISN;
    m_iRcvLastAckAck = m_ConnRes.m_iISN;
    m_iRcvCurrSeqNo = m_ConnRes.m_iISN - 1;
    m_PeerID = m_ConnRes.m_iID;
    memcpy(m_piSelfIP, m_ConnRes.m_piPeerIP, 16);

    // Prepare all data structures
    try {
        m_pSndBuffer = new CSndBuffer(32, m_iPayloadSize);
        m_pRcvBuffer = new CRcvBuffer(&(m_pRcvQueue->m_UnitQueue), m_iRcvBufSize);
        // after introducing lite ACK, the sndlosslist may not be cleared in time, so it requires twice space.
        m_pSndLossList = new CSndLossList(m_iFlowWindowSize * 2);
        m_pRcvLossList = new CRcvLossList(m_iFlightFlagSize);
        m_pACKWindow = new CACKWindow(1024);
        m_pRcvTimeWindow = new CPktTimeWindow(16, 64);
        m_pSndTimeWindow = new CPktTimeWindow();
    } catch (...) {
        throw CUDTException(3, 2, 0);
    }

    CInfoBlock ib;
    ib.m_iIPversion = m_iIPversion;
    CInfoBlock::convert(m_pPeerAddr, m_iIPversion, ib.m_piIP);
    if (m_pCache->lookup(&ib) >= 0) {
        m_iRTT = ib.m_iRTT;
        m_iBandwidth = ib.m_iBandwidth;
    }

    m_pCC = m_pCCFactory->create();
    m_pCC->m_UDT = m_SocketID;
    m_pCC->setMSS(m_iMSS);
    m_pCC->setMaxCWndSize(m_iFlowWindowSize);
    m_pCC->setSndCurrSeqNo(m_iSndCurrSeqNo);
    m_pCC->setRcvRate(m_iDeliveryRate);
    m_pCC->setRTT(m_iRTT);
    m_pCC->setBandwidth(m_iBandwidth);
    m_pCC->init();

    m_ullInterval = (uint64_t) (m_pCC->m_dPktSndPeriod * m_ullCPUFrequency);
    m_dCongestionWindow = m_pCC->m_dCWndSize;

    // And, I am connected too.
    m_bConnecting = false;
    m_bConnected = true;

    // register this socket for receiving data packets
    m_pRNode->m_bOnList = true;
    m_pRcvQueue->setNewEntry(this);

    // acknowledge the management module.
    s_UDTUnited.connect_complete(m_SocketID);

    // acknowledde any waiting epolls to write
    s_UDTUnited.m_EPoll.update_events(m_SocketID, m_sPollID, UDT_EPOLL_OUT, true);

    return 0;
}
```
1. 这里做的第一件事就是调用m_pRcvQueue->removeConnector(m_SocketID)将自己从RevQueue的RendezvousQueue中移除，以表示自己将不再接收Rendezvous消息(src/queue.cpp)：
```
void CRcvQueue::removeConnector(const UDTSOCKET& id) {
    m_pRendezvousQueue->remove(id);

    CGuard bufferlock(m_PassLock);

    map<int32_t, std::queue<CPacket*> >::iterator i = m_mBuffer.find(id);
    if (i != m_mBuffer.end()) {
        while (!i->second.empty()) {
            delete[] i->second.front()->m_pcData;
            delete i->second.front();
            i->second.pop();
        }
        m_mBuffer.erase(i);
    }
}
```

这个函数执行完之后，RcvQueue暂时将无法向UDT Socket dispatch包。

2. 根据协商的值重新做配置。这里我们可以再来看一下UDT的协商指的是什么。纵览连接建立的整个过程，我们并没有看到针对这些需要协商的值UDT本身有什么特殊的算法来计算，因而所谓的协商则主要是UDT Client端和UDT Server端，针对这些选项，不同应用程序层不同设置的同步协调。

3. 准备所有的数据缓冲区。

4. 设置CUDT的状态，m_bConnecting为false，m_bConnected为true。

5. 执行m_pRcvQueue->setNewEntry(this)，注册socket来接收数据包。这里来看一下CRcvQueue::setNewEntry(CUDT* u)：

```
void CRcvQueue::setNewEntry(CUDT* u) {
    CGuard listguard(m_IDLock);
    m_vNewEntry.push_back(u);
}
```
这个操作本身非常简单。但把CUDT结构放进CRcvQueue之后，又会发生什么呢？回忆我们前面看到的CRcvQueue::worker(void* param)函数中循环开始部分的这段代码：
```
// check waiting list, if new socket, insert it to the list
        while (self->ifNewEntry()) {
            CUDT* ne = self->getNewEntry();
            if (NULL != ne) {
                self->m_pRcvUList->insert(ne);
                self->m_pHash->insert(ne->m_SocketID, ne);
            }
        }
```
对照这段代码中用到的几个函数的实现：
```
bool CRcvQueue::ifNewEntry() {
    return !(m_vNewEntry.empty());
}

CUDT* CRcvQueue::getNewEntry() {
    CGuard listguard(m_IDLock);

    if (m_vNewEntry.empty())
        return NULL;

    CUDT* u = (CUDT*) *(m_vNewEntry.begin());
    m_vNewEntry.erase(m_vNewEntry.begin());

    return u;
}
```

可以了解到，在 执行m_pRcvQueue->setNewEntry(this)，注册socket之后，CRcvQueue的worker线程会将这个CUDT结构从它的m_vNewEntry中移到另外的两个容器m_pRcvUList和m_pHash中。那然后呢？在CRcvQueue::worker(void* param)中不是还有下面这段吗：
```
            if (NULL != (u = self->m_pHash->lookup(id))) {
                if (CIPAddress::ipcmp(addr, u->m_pPeerAddr, u->m_iIPversion)) {
                    cout << "Receive packet by m_pHash table" << endl;
                    if (u->m_bConnected && !u->m_bBroken && !u->m_bClosing) {
                        if (0 == unit->m_Packet.getFlag())
                            u->processData(unit);
                        else
                            u->processCtrl(unit->m_Packet);

                        u->checkTimers();
                        self->m_pRcvUList->update(u);
                    }
                }
            } else if (NULL != (u = self->m_pRendezvousQueue->retrieve(addr, id))) {
```

就是这样，可以说，在CUDT::connect(const CPacket& response)中是完成了一次UDT Socket消息接收方式的转变。

6. 执行s_UDTUnited.connect_complete(m_SocketID)结束整个的connect()过程：

```
void CUDTUnited::connect_complete(const UDTSOCKET u) {
    CUDTSocket* s = locate(u);
    if (NULL == s)
        throw CUDTException(5, 4, 0);

    // copy address information of local node
    // the local port must be correctly assigned BEFORE CUDT::connect(),
    // otherwise if connect() fails, the multiplexer cannot be located by garbage collection and will cause leak
    s->m_pUDT->m_pSndQueue->m_pChannel->getSockAddr(s->m_pSelfAddr);
    CIPAddress::pton(s->m_pSelfAddr, s->m_pUDT->m_piSelfIP, s->m_iIPversion);

    s->m_Status = CONNECTED;
}
```

UDT Socket至此进入CONNECTED状态。

Done。
