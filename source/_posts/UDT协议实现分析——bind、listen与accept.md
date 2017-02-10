---
title: UDT协议实现分析——bind、listen与accept
date: 2015-09-09 16:05:49
tags:
- 网络
---

UDT Server启动之后，基于UDT协议的UDP数据可靠传输才成为可能，因而接下来分析与UDT Server有关的几个主要API的实现，来了解下UDT Server是如何listening在特定UDP端口上的。主要有UDT::bind()，UDT::listen()和UDT::accept()等几个函数。
<!--more-->
# bind过程

通常UDT Server在创建UDT Socket之后，首先就要调用UDT::bind()，与一个特定的本地UDP端口地址进行绑定，以便可以在希望的端口上监听。这里来看一下UDT::bind()的实现：
```
int CUDTUnited::bind(const UDTSOCKET u, const sockaddr* name, int namelen) {
    CUDTSocket* s = locate(u);
    if (NULL == s)
        throw CUDTException(5, 4, 0);

    CGuard cg(s->m_ControlLock);

    // cannot bind a socket more than once
    if (INIT != s->m_Status)
        throw CUDTException(5, 0, 0);

    // check the size of SOCKADDR structure
    if (AF_INET == s->m_iIPversion) {
        if (namelen != sizeof(sockaddr_in))
            throw CUDTException(5, 3, 0);
    } else {
        if (namelen != sizeof(sockaddr_in6))
            throw CUDTException(5, 3, 0);
    }

    s->m_pUDT->open();
    updateMux(s, name);
    s->m_Status = OPENED;

    // copy address information of local node
    s->m_pUDT->m_pSndQueue->m_pChannel->getSockAddr(s->m_pSelfAddr);

    return 0;
}

int CUDTUnited::bind(UDTSOCKET u, UDPSOCKET udpsock) {
    CUDTSocket* s = locate(u);
    if (NULL == s)
        throw CUDTException(5, 4, 0);

    CGuard cg(s->m_ControlLock);

    // cannot bind a socket more than once
    if (INIT != s->m_Status)
        throw CUDTException(5, 0, 0);

    sockaddr_in name4;
    sockaddr_in6 name6;
    sockaddr* name;
    socklen_t namelen;

    if (AF_INET == s->m_iIPversion) {
        namelen = sizeof(sockaddr_in);
        name = (sockaddr*) &name4;
    } else {
        namelen = sizeof(sockaddr_in6);
        name = (sockaddr*) &name6;
    }

    if (-1 == ::getsockname(udpsock, name, &namelen))
        throw CUDTException(5, 3);

    s->m_pUDT->open();
    updateMux(s, name, &udpsock);
    s->m_Status = OPENED;

    // copy address information of local node
    s->m_pUDT->m_pSndQueue->m_pChannel->getSockAddr(s->m_pSelfAddr);

    return 0;
}



int CUDT::bind(UDTSOCKET u, const sockaddr* name, int namelen) {
    try {
        return s_UDTUnited.bind(u, name, namelen);
    } catch (CUDTException& e) {
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

int CUDT::bind(UDTSOCKET u, UDPSOCKET udpsock) {
    try {
        return s_UDTUnited.bind(u, udpsock);
    } catch (CUDTException& e) {
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


int bind(UDTSOCKET u, const struct sockaddr* name, int namelen) {
    return CUDT::bind(u, name, namelen);
}

int bind2(UDTSOCKET u, UDPSOCKET udpsock) {
    return CUDT::bind(u, udpsock);
}
```
UDT主要提供了两个bind接口，分别是UDT::bind()和，UDT::bind2()。UDT::bind()将一个UDT Socket与一个struct sockaddr对象描述的地址进行绑定，这需要UDT自己先创建相应的系统UDP socket，并将该系统UDP socket绑定到地址，然后把UDT Socket绑定到该系统UDP socket；UDT::bind2()则将一个UDT Socket直接与一个已经创建好的系统UDP socket进行绑定。

这两个API的实现结构与[UDT::socket()的实现结构](http://my.oschina.net/wolfcs/blog/502509)基本一致，一样是分为3层：UDT命名空间中提供了给应用程序调用的接口，可称为UDT API或User API；User API调用CUDT API，这一层主要用来做错误处理，也就是捕获动作实际执行过程中抛出的异常并保存起来，然后给应用程序使用；CUDT API调用CUDTUnited中API的实现。

这里主要来看CUDTUnited中bind()函数的实现。先来看CUDTUnited::bind(const UDTSOCKET u, const sockaddr* name, int namelen)函数的实现：

1. 调用CUDTUnited::locate()，根据SocketID，也就是UDT Socket handle在CUDTUnited的std::map<UDTSOCKET, CUDTSocket*> m_Sockets中找到对应的CUDTSocket结构(src/api.cpp)：
```
CUDTSocket* CUDTUnited::locate(const UDTSOCKET u) {
    CGuard cg(m_ControlLock);

    map<UDTSOCKET, CUDTSocket*>::iterator i = m_Sockets.find(u);

    if ((i == m_Sockets.end()) || (i->second->m_Status == CLOSED))
        return NULL;

    return i->second;
}
```
若找不到，则直接返回；否则，继续执行。

2. 检查CUDTSocket对象的状态，如果当前的状态不为INIT，直接抛异常退出；否则，继续执行。

3. 根据本地IP地址的版本，检查绑定到的目标地址的长度的有效性。IP版本是在UDT Socket创建时指定的。如果无效，则直接抛异常退出；否则，继续执行。

4. 执行相应的CUDT的open()操作(src/core.cpp)：
```
void CUDT::open() {
    CGuard cg(m_ConnectionLock);

    // Initial sequence number, loss, acknowledgement, etc.
    m_iPktSize = m_iMSS - 28;
    m_iPayloadSize = m_iPktSize - CPacket::m_iPktHdrSize;

    m_iEXPCount = 1;
    m_iBandwidth = 1;
    m_iDeliveryRate = 16;
    m_iAckSeqNo = 0;
    m_ullLastAckTime = 0;

    // trace information
    m_StartTime = CTimer::getTime();
    m_llSentTotal = m_llRecvTotal = m_iSndLossTotal = m_iRcvLossTotal = m_iRetransTotal = m_iSentACKTotal =
            m_iRecvACKTotal = m_iSentNAKTotal = m_iRecvNAKTotal = 0;
    m_LastSampleTime = CTimer::getTime();
    m_llTraceSent = m_llTraceRecv = m_iTraceSndLoss = m_iTraceRcvLoss = m_iTraceRetrans = m_iSentACK = m_iRecvACK =
            m_iSentNAK = m_iRecvNAK = 0;
    m_llSndDuration = m_llSndDurationTotal = 0;

    // structures for queue
    if (NULL == m_pSNode)
        m_pSNode = new CSNode;
    m_pSNode->m_pUDT = this;
    m_pSNode->m_llTimeStamp = 1;
    m_pSNode->m_iHeapLoc = -1;

    if (NULL == m_pRNode)
        m_pRNode = new CRNode;
    m_pRNode->m_pUDT = this;
    m_pRNode->m_llTimeStamp = 1;
    m_pRNode->m_pPrev = m_pRNode->m_pNext = NULL;
    m_pRNode->m_bOnList = false;

    m_iRTT = 10 * m_iSYNInterval;
    m_iRTTVar = m_iRTT >> 1;
    m_ullCPUFrequency = CTimer::getCPUFrequency();

    // set up the timers
    m_ullSYNInt = m_iSYNInterval * m_ullCPUFrequency;

    // set minimum NAK and EXP timeout to 100ms
    m_ullMinNakInt = 300000 * m_ullCPUFrequency;
    m_ullMinExpInt = 300000 * m_ullCPUFrequency;

    m_ullACKInt = m_ullSYNInt;
    m_ullNAKInt = m_ullMinNakInt;

    uint64_t currtime;
    CTimer::rdtsc(currtime);
    m_ullLastRspTime = currtime;
    m_ullNextACKTime = currtime + m_ullSYNInt;
    m_ullNextNAKTime = currtime + m_ullNAKInt;

    m_iPktCount = 0;
    m_iLightACKCount = 1;

    m_ullTargetTime = 0;
    m_ullTimeDiff = 0;

    // Now UDT is opened.
    m_bOpened = true;
}
```
在这个函数中，主要还是对变量的初始化，后面会再结合UDT可靠传输的具体机制，来说明这些变量的具体含义。

5. 执行updateMux()函数更新UDT Socket的多路复用器的相关信息，后面我们会再来详细了解这个更新操作。

6. 将CUDTSocket对象的状态更新为OPENED。

7. 将发送队列的Channel的地址信息拷贝到本节点的s->m_pSelfAddr，m_pSelfAddrde对象的内存空间是在创建UDT Socket的CUDTUnited::newSocket()函数中分配的。
后面会再来解释UDT中Channel和多路复用器Multipexer的含义。

8. 返回0给调用者表示成功结束。

再来看CUDTUnited::bind(UDTSOCKET u, UDPSOCKET udpsock)函数将UDT Socket绑定到一个已经创建好的系统UDP socket的过程：

1. 调用CUDTUnited::locate()，根据SocketID，也就是UDT Socket handle在CUDTUnited的std::map<UDTSOCKET, CUDTSocket*> m_Sockets中找到对应的CUDTSocket结构。若找不到，则直接返回；否则，继续执行。

2. 检查CUDTSocket对象的状态，如果当前的状态不为INIT，直接抛异常退出；否则，继续执行。

3. 获取系统UDP socket的网络地址(含端口信息)。若获取失败则抛异常推出；否则，继续执行。

4. 执行相应的CUDT的open()操作对一些变量进行初始化。

5. 执行updateMux()函数更新UDT Socket的多路复用器的相关信息，后面我们会再来详细了解这个更新操作。

6. 将CUDTSocket对象的状态更新为OPENED。

7. 将发送队列的Channel的地址信息拷贝到本节点的s->m_pSelfAddr，m_pSelfAddrde对象的内存空间是在创建UDT Socket的CUDTUnited::newSocket()函数中分配的。
后面会再来解释UDT中Channel和多路复用器Multipexer的含义。

8. 返回0给调用者表示成功结束。m_MultiplexerLock

总体来说，bind操作使的UDT Socket状态机的状态由INIT状态，转换到了OPENED状态。

CUDTUnited的这两个bind()函数有如此多的重复逻辑，总让人觉得，是有方法做进一步的抽象，以消除重复的逻辑，并使这两个函数的实现都更加精简的。

# UDT Socket与多路复用器的关联

bind()操作所做的最最重要的事大概就是将UDT Socket与多路复用器关联，也就是CUDTUnited::updateMux()函数的执行了。为了后面能够更清晰地说明更新多路复用器的操作过程，这里先说明一下UDT的多路复用器CMultiplexer、通道CChannel、发送队列CSndQueue和接收队列CRcvQueue的含义。

UDT中的通道CChannel是系统UDP socket的一个封装，它主要封装了系统UDP socket handle，IP版本号，socket地址的长度，发送缓冲区的大小及接收缓冲区的大小等信息，并提供了用于操作 系统UDP socket进行数据收发或属性设置等动作的函数。我们可以看一下这个class的定义(src/channel.h)：
```
class CChannel {
 public:
    CChannel();
    CChannel(int version);
    ~CChannel();

    // Functionality:
    //    Open a UDP channel.
    // Parameters:
    //    0) [in] addr: The local address that UDP will use.
    // Returned value:
    //    None.
    void open(const sockaddr* addr = NULL);

    // Functionality:
    //    Open a UDP channel based on an existing UDP socket.
    // Parameters:
    //    0) [in] udpsock: UDP socket descriptor.
    // Returned value:
    //    None.
    void open(UDPSOCKET udpsock);

    // Functionality:
    //    Disconnect and close the UDP entity.
    // Parameters:
    //    None.
    // Returned value:
    //    None.
    void close() const;

    // Functionality:
    //    Get the UDP sending buffer size.
    // Parameters:
    //    None.
    // Returned value:
    //    Current UDP sending buffer size.
    int getSndBufSize();

    // Functionality:
    //    Get the UDP receiving buffer size.
    // Parameters:
    //    None.
    // Returned value:
    //    Current UDP receiving buffer size.
    int getRcvBufSize();

    // Functionality:
    //    Set the UDP sending buffer size.
    // Parameters:
    //    0) [in] size: expected UDP sending buffer size.
    // Returned value:
    //    None.
    void setSndBufSize(int size);

    // Functionality:
    //    Set the UDP receiving buffer size.
    // Parameters:
    //    0) [in] size: expected UDP receiving buffer size.
    // Returned value:
    //    None.
    void setRcvBufSize(int size);

    // Functionality:
    //    Query the socket address that the channel is using.
    // Parameters:
    //    0) [out] addr: pointer to store the returned socket address.
    // Returned value:
    //    None.
    void getSockAddr(sockaddr* addr) const;

    // Functionality:
    //    Send a packet to the given address.
    // Parameters:
    //    0) [in] addr: pointer to the destination address.
    //    1) [in] packet: reference to a CPacket entity.
    // Returned value:
    //    Actual size of data sent.
    int sendto(const sockaddr* addr, CPacket& packet) const;

    // Functionality:
    //    Receive a packet from the channel and record the source address.
    // Parameters:
    //    0) [in] addr: pointer to the source address.
    //    1) [in] packet: reference to a CPacket entity.
    // Returned value:
    //    Actual size of data received.
    int recvfrom(sockaddr* addr, CPacket& packet) const;

 private:
    void setUDPSockOpt();

 private:
    int m_iIPversion;                    // IP version
    int m_iSockAddrSize;                 // socket address structure size (pre-defined to avoid run-time test)

    UDPSOCKET m_iSocket;                 // socket descriptor

    int m_iSndBufSize;                   // UDP sending buffer size
    int m_iRcvBufSize;                   // UDP receiving buffer size
};
```
接收队列CRcvQueue在初始化时会起一个线程，该线程在被停掉前，会不断地由CChannel接收其它节点发送过来的UDP消息，可以将这个线程看做是listening在系统UDP 端口上的一个UDP Server。在接收到消息之后，该线程会根据消息的类型及目标 SocketID，把消息dispatch给不同的UDT Socket的CUDT对象。比如对于Handshake类型的消息就会dispatch给listening的UDT Socket的CUDT对象。后面我们研究具体的消息收发的时候再来仔细看这个类的设计。

发送队列CSndQueue，主要用于同步地向特定的目标发送一个UDT的Packet，或者在适当的时机异步地发送一些消息，它同样会在初始化是起一个线程，用来执行异步地发送任务。这个class是UDT做可靠传输的一个比较关键的class，后面我们研究具体的消息收发的时候再来仔细看这个类的设计。

UDT的多路复用器结构CMultiplexer将所有这些与特定的系统UDP socket相关联的CChannel，CRcvQueue，CSndQueue包在一起，并描述了这个系统UDP socket收发的数据的一些公有属性，有UDP 端口号，IP版本号，最大的包大小，引用计数，是否可复用，及用做哈希索引的ID等。可以看一下这个class的定义：
```
struct CMultiplexer {
    CSndQueue* m_pSndQueue;  // The sending queue
    CRcvQueue* m_pRcvQueue;  // The receiving queue
    CChannel* m_pChannel;    // The UDP channel for sending and receiving
    CTimer* m_pTimer;        // The timer

    int m_iPort;             // The UDP port number of this multiplexer
    int m_iIPversion;        // IP version
    int m_iMSS;              // Maximum Segment Size
    int m_iRefCount;         // number of UDT instances that are associated with this multiplexer
    bool m_bReusable;        // if this one can be shared with others

    int m_iID;               // multiplexer ID

    CMultiplexer()
            : m_pSndQueue(NULL),
              m_pRcvQueue(NULL),
              m_pChannel(NULL),
              m_pTimer(NULL),
              m_iPort(0),
              m_iIPversion(0),
              m_iMSS(0),
              m_iRefCount(0),
              m_bReusable(true),
              m_iID(0) {
    }
};
```
接着来看CUDTUnited::updateMux()函数的定义(src/api.cpp)：
```
void CUDTUnited::updateMux(CUDTSocket* s, const sockaddr* addr, const UDPSOCKET* udpsock) {
    CGuard cg(m_ControlLock);

    CMultiplexer m;
    if ((s->m_pUDT->m_bReuseAddr) && (NULL != addr)) {
        int port = (AF_INET == s->m_pUDT->m_iIPversion) ?
                ntohs(((sockaddr_in*) addr)->sin_port) : ntohs(((sockaddr_in6*) addr)->sin6_port);

        // find a reusable address
        for (map<int, CMultiplexer>::iterator i = m_mMultiplexer.begin(); i != m_mMultiplexer.end(); ++i) {
            if ((i->second.m_iIPversion == s->m_pUDT->m_iIPversion) && (i->second.m_iMSS == s->m_pUDT->m_iMSS)
                    && i->second.m_bReusable) {
                if (i->second.m_iPort == port) {
                    // reuse the existing multiplexer
                    m = i->second;
                    break;
                }
            }
        }
    }

    // a new multiplexer is needed
    if (m.m_iID == 0) {
        m.m_iMSS = s->m_pUDT->m_iMSS;
        m.m_iIPversion = s->m_pUDT->m_iIPversion;
        m.m_bReusable = s->m_pUDT->m_bReuseAddr;
        m.m_iID = s->m_SocketID;

        m.m_pChannel = new CChannel(s->m_pUDT->m_iIPversion);
        m.m_pChannel->setSndBufSize(s->m_pUDT->m_iUDPSndBufSize);
        m.m_pChannel->setRcvBufSize(s->m_pUDT->m_iUDPRcvBufSize);

        try {
            if (NULL != udpsock)
                m.m_pChannel->open(*udpsock);
            else
                m.m_pChannel->open(addr);
        } catch (CUDTException& e) {
            m.m_pChannel->close();
            delete m.m_pChannel;
            throw e;
        }

        sockaddr* sa =
                (AF_INET == s->m_pUDT->m_iIPversion) ? (sockaddr*) new sockaddr_in : (sockaddr*) new sockaddr_in6;
        m.m_pChannel->getSockAddr(sa);
        m.m_iPort = (AF_INET == s->m_pUDT->m_iIPversion) ?
                        ntohs(((sockaddr_in*) sa)->sin_port) : ntohs(((sockaddr_in6*) sa)->sin6_port);
        if (AF_INET == s->m_pUDT->m_iIPversion)
            delete (sockaddr_in*) sa;
        else
            delete (sockaddr_in6*) sa;

        m.m_pTimer = new CTimer;

        m.m_pSndQueue = new CSndQueue;
        m.m_pSndQueue->init(m.m_pChannel, m.m_pTimer);
        m.m_pRcvQueue = new CRcvQueue;
        m.m_pRcvQueue->init(32, s->m_pUDT->m_iPayloadSize, m.m_iIPversion, 1024, m.m_pChannel, m.m_pTimer);

        m_mMultiplexer[m.m_iID] = m;
    }

    ++m.m_iRefCount;
    s->m_pUDT->m_pSndQueue = m.m_pSndQueue;
    s->m_pUDT->m_pRcvQueue = m.m_pRcvQueue;
    s->m_iMuxID = m.m_iID;
}
```
1. 这个函数首先会在已经创建的多路复用器的map中查找，看看是否存在 要与多路复用器关联的UDT Socket可用的多路复用器存在。对于一个UDT Socket来说，UDT Socket本身网络地址可复用，且某个多路复用器同时满足它的CChannel的UDP端口号与UDT Socket要bind的目标UDP端口号匹配，它的CChannel的IP地址版本及MSS与UDT Socket的IP地址版本及MSS匹配，它本身可复用，则该多路复用器就是该UDT Socket可用的多路复用器。

2. 若在前面的步骤中，没有找到可用的多路复用器，则创建一个。
根据UDT Socket的MSS值，IP版本号，及地址的可复用性来初始化CMultiplexer的对应值。设置CMultiplexer的ID为UDT Socket的SocketID。也就是说，某个CMultiplexer的ID就是与它关联的首个UDT Socket的SocketID。
创建CChannel，设置系统UDP socket发送缓冲区及接收缓冲区的大小。并执行CChannel的open()操作。在CChannel::open()中如果不是绑定的已经创建好的系统UDP socket的话，它会自行创建系统UDP socket，并绑定到目标端口上。
获取CChannel实际绑定的UDP端口号，赋值给m.m_iPort。
创建CTimer。
创建并初始化CSndQueue。
创建并初始化CRcvQueue。
将新建的CMultiplexer放进std::map<int, CMultiplexer> m_mMultiplexer中。
在CUDTUnited类定义中可以看到如下几行：
```
private:
    std::map<int, CMultiplexer> m_mMultiplexer;		// UDP multiplexer
    pthread_mutex_t m_MultiplexerLock;
```
原本设计似乎是要用m_MultiplexerLock来保证对m_mMultiplexer多线程的互斥访问的，但却没有一个地方有用到这个m_MultiplexerLock。不知是发现保护全无必要，还是有所遗漏？

3. 将UDT Socket与多路复用器关联起来，不管是找到的现成可用的，还是完全新创建的。这里可以看到所谓的将UDT Socket与多路复用器关联的含义，即是让CUDTSocket的CUDT对象m_pUDT的发送队列和接收队列指向CMultiplexer的发送队列和接收队列，设置CUDTSocket的多路复用器ID为CMultiplexer的ID m_iID，这样后面CUDTSocket和CUDT就可以使用发送队列CSndQueue和接收队列CRcvQueue进行数据的收发，并可在需要的时候找到相关的CMultiplexer对象了。
自此之后，CUDTSocket就有了可以用来收发数据的设施了。

总结一下UDT bind的主要过程。UDT bind过程中，做的最主要的事情就是，根据一个已经创建好的UDT Socket的一些信息及要绑定的本地UDP端口，找到或创建一个多路复用器CMultiplexer，将UDT Socket与该CMultiplexer关联，即设置CUDTSocket的多路复用器ID m_iMuxID为该CMultiplexer的ID，UDT Socket的发送队列指针和接收队列指针指向该CMultiplexer的发送队列和接收队列。后续UDT Socket就可以通过发送队列/接收队列及它们的CChannel进行数据的收发了。在这个过程中，UDT Socket状态机完成了状态由INIT到OPENED的转变。

# listen过程

在UDT Server端，对UDT Socket执行了bind操作之后，就可以执行listen来等待其它节点的连接了。这里来看下UDT listen的过程(src/api.cpp)：
```
int CUDTUnited::listen(const UDTSOCKET u, int backlog) {
    CUDTSocket* s = locate(u);
    if (NULL == s)
        throw CUDTException(5, 4, 0);

    CGuard cg(s->m_ControlLock);

    // do nothing if the socket is already listening
    if (LISTENING == s->m_Status)
        return 0;

    // a socket can listen only if is in OPENED status
    if (OPENED != s->m_Status)
        throw CUDTException(5, 5, 0);

    // listen is not supported in rendezvous connection setup
    if (s->m_pUDT->m_bRendezvous)
        throw CUDTException(5, 7, 0);

    if (backlog <= 0)
        throw CUDTException(5, 3, 0);

    s->m_uiBackLog = backlog;

    try {
        s->m_pQueuedSockets = new set<UDTSOCKET>;
        s->m_pAcceptSockets = new set<UDTSOCKET>;
    } catch (...) {
        delete s->m_pQueuedSockets;
        delete s->m_pAcceptSockets;
        throw CUDTException(3, 2, 0);
    }

    s->m_pUDT->listen();

    s->m_Status = LISTENING;

    return 0;
}


int CUDT::listen(UDTSOCKET u, int backlog) {
    try {
        return s_UDTUnited.listen(u, backlog);
    } catch (CUDTException& e) {
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


int listen(UDTSOCKET u, int backlog) {
    return CUDT::listen(u, backlog);
}
```
这个API的实现同样分为3层，UDT命名空间提供的直接给应用程序调用User API层，CUDT API层用于做异常处理，CUDTUnited具体实现API的功能。这里直接来分析CUDTUnited::listen()函数：

1. 调用CUDTUnited::locate()，查找UDT Socket对应的CUDTSocket结构。若找不到，则抛出异常直接返回；否则，继续执行。

2. 检查CUDTSocket对象的状态，如果当前的状态为LISTENING，则说明UDT Socket已经处于监听状态了，直接返回；若当前状态不为OPENED，直接抛异常退出，否则，继续执行。这就限制了只有经过了bind操作的UDT Socket才能监听，也就是UDT Socket的状态只能由OPENED转为LISTENING。

3. 检查是否是rendezvous的UDT Socket，若是则抛出异常推出。这确保在监听的UDT Socket不能为rendezvous的。

4. 检查传入的backlog参数并进行设置。backlog参数用于指定Listening的UDT Socket同一时刻能够处理的最大的等待连接的请求数。Listening的UDT Socket在收到连接请求的Handshake消息后，经过几次来回确认，会创建新的UDT Socket以便于通过UDT::accept()函数返回给应用程序，用于与请求连接的发起方进行通信。backlog值用于限定，还没有通过accept()返回的新创建的UDT Socket的个数。

5. 创建两个UDTSOCKET的集合m_pQueuedSockets和m_pAcceptSockets，前者为Listening的UDT Socket的连接已经成功建立但还未通过UDT::accept()返回给应用程序的UDT Socket的集合；而后者则是已经通过UDT::accept()返回给应用程序的UDT Socket的集合。

6. 执行UDTSocket的CUDT的listen()操作，可以看一下CUDT listen动作的具体含义(src/core.cpp)：
```
void CUDT::listen() {
    CGuard cg(m_ConnectionLock);

    if (!m_bOpened)
        throw CUDTException(5, 0, 0);

    if (m_bConnecting || m_bConnected)
        throw CUDTException(5, 2, 0);

    // listen can be called more than once
    if (m_bListening)
        return;

    // if there is already another socket listening on the same port
    if (m_pRcvQueue->setListener(this) < 0)
        throw CUDTException(5, 11, 0);

    m_bListening = true;
}
```
先是进行状态的合法性检查。
然后执行m_pRcvQueue->setListener(this)，将本CUDT设置为接收队列的listener。
最后设置CUDT的状态m_bListening为true。
这里可以看出CUDTSocket与CUDT是表示UDT Socket的两层状态机，它们的状态之间有关联，但又有各自的描述方法。这样似乎大大增加了这个UDT Socket状态管理的复杂度了。
再来看一下CRcvQueue::setListener()(src/queue.cpp)：
```
int CRcvQueue::setListener(CUDT* u) {
    CGuard lslock(m_LSLock);

    if (NULL != m_pListener)
        return -1;

    m_pListener = u;
    return 0;
}
```
设置接收队列CRcvQueue的Listener。

7. 设置CUDTSocket的状态为LISTENING并返回。
可以看到对于UDT::listen()的调用，促使UDT Socket的状态由OPENED转换为了LISTENING。UDT::listen()主要的作用就是 为与UDT Socket关联的特定端口上的多路复用器CMultiplexer的接收队列CRcvQueue设置listener，这个动作最主要的意义在于消息的dispatch。我们知道CMultiplexer的接收队列CRcvQueue在创建、初始化时会起一个线程，不断地试图从网络接收UDP消息，在收到消息之后，将消息dispatch给不同的UDT Socket处理，其中的Handshake等消息，就会被dispatch给listener CUDT处理。后面在具体研究消息的收发时会再来详细研究这个过程。

# accept过程

UDT Server端在对listening执行了UDT::listen()操作之后，就可以执行UDT::accept()操作来等待其它节点连接自己了。来看一下UDT::accept()的执行过程(src/api.cpp)：
```
UDTSOCKET CUDTUnited::accept(const UDTSOCKET listen, sockaddr* addr, int* addrlen) {
    if ((NULL != addr) && (NULL == addrlen))
        throw CUDTException(5, 3, 0);

    CUDTSocket* ls = locate(listen);

    if (ls == NULL)
        throw CUDTException(5, 4, 0);

    // the "listen" socket must be in LISTENING status
    if (LISTENING != ls->m_Status)
        throw CUDTException(5, 6, 0);

    // no "accept" in rendezvous connection setup
    if (ls->m_pUDT->m_bRendezvous)
        throw CUDTException(5, 7, 0);

    UDTSOCKET u = CUDT::INVALID_SOCK;
    bool accepted = false;

    // !!only one conection can be set up each time!!
#ifndef WIN32
    while (!accepted) {
        pthread_mutex_lock(&(ls->m_AcceptLock));

        if ((LISTENING != ls->m_Status) || ls->m_pUDT->m_bBroken) {
            // This socket has been closed.
            accepted = true;
        } else if (ls->m_pQueuedSockets->size() > 0) {
            u = *(ls->m_pQueuedSockets->begin());
            ls->m_pAcceptSockets->insert(ls->m_pAcceptSockets->end(), u);
            ls->m_pQueuedSockets->erase(ls->m_pQueuedSockets->begin());
            accepted = true;
        } else if (!ls->m_pUDT->m_bSynRecving) {
            accepted = true;
        }

        if (!accepted && (LISTENING == ls->m_Status))
            pthread_cond_wait(&(ls->m_AcceptCond), &(ls->m_AcceptLock));

        if (ls->m_pQueuedSockets->empty())
            m_EPoll.update_events(listen, ls->m_pUDT->m_sPollID, UDT_EPOLL_IN, false);

        pthread_mutex_unlock(&(ls->m_AcceptLock));
    }
#else
    while (!accepted)
    {
        WaitForSingleObject(ls->m_AcceptLock, INFINITE);

        if (ls->m_pQueuedSockets->size() > 0)
        {
            u = *(ls->m_pQueuedSockets->begin());
            ls->m_pAcceptSockets->insert(ls->m_pAcceptSockets->end(), u);
            ls->m_pQueuedSockets->erase(ls->m_pQueuedSockets->begin());

            accepted = true;
        }
        else if (!ls->m_pUDT->m_bSynRecving)
        accepted = true;

        ReleaseMutex(ls->m_AcceptLock);

        if (!accepted & (LISTENING == ls->m_Status))
        WaitForSingleObject(ls->m_AcceptCond, INFINITE);

        if ((LISTENING != ls->m_Status) || ls->m_pUDT->m_bBroken)
        {
            // Send signal to other threads that are waiting to accept.
            SetEvent(ls->m_AcceptCond);
            accepted = true;
        }

        if (ls->m_pQueuedSockets->empty())
        m_EPoll.update_events(listen, ls->m_pUDT->m_sPollID, UDT_EPOLL_IN, false);
    }
#endif

    if (u == CUDT::INVALID_SOCK) {
        // non-blocking receiving, no connection available
        if (!ls->m_pUDT->m_bSynRecving)
            throw CUDTException(6, 2, 0);

        // listening socket is closed
        throw CUDTException(5, 6, 0);
    }

    if ((addr != NULL) && (addrlen != NULL)) {
        if (AF_INET == locate(u)->m_iIPversion)
            *addrlen = sizeof(sockaddr_in);
        else
            *addrlen = sizeof(sockaddr_in6);

        // copy address information of peer node
        memcpy(addr, locate(u)->m_pPeerAddr, *addrlen);
    }

    return u;
}



UDTSOCKET CUDT::accept(UDTSOCKET u, sockaddr* addr, int* addrlen) {
    try {
        return s_UDTUnited.accept(u, addr, addrlen);
    } catch (CUDTException& e) {
        s_UDTUnited.setError(new CUDTException(e));
        return INVALID_SOCK;
    } catch (...) {
        s_UDTUnited.setError(new CUDTException(-1, 0, 0));
        return INVALID_SOCK;
    }
}


UDTSOCKET accept(UDTSOCKET u, struct sockaddr* addr, int* addrlen) {
    return CUDT::accept(u, addr, addrlen);
}
```
这个API实现的3层结构与UDT::bind()，UDT::listen()一样，不再赘述。来看CUDTUnited::accept()的实现：

1. 调用CUDTUnited::locate()，查找UDT Socket对应的CUDTSocket结构。若找不到，则抛出异常直接返回；否则，继续执行。

2. 检查CUDTSocket对象的状态。可见CUDTUnited::accept()操作要求相应的UDT Socket必须处于LISTENING状态，且不能为Rendezvous模式。这个地方对于ls->m_pUDT->m_bRendezvous的检查似乎有些多余了，在CUDTUnited::listen()中可以看到，如果UDT Socket处于Rendezvous模式的话，根本就不可能完成状态由OPENED到LISTENING的转换，因而对于UDT Socket LISTENING状态的检查已经足够了。

3. 通过一个循环来等待其它节点的连接。这指的是，等待ls->m_pQueuedSockets中被放入为新的连接创建的UDT Socket。有新的连接时，CUDTUnited::accept()线程被唤醒，它会将UDT Socket从ls->m_pQueuedSockets中移到ls->m_pAcceptSockets，并准备将UDT Socket返回给调用者。当然CUDTUnited::accept()的等待过程结束的条件不只是有新连接进来，在Listening的UDT Socket被closed掉时，ls->m_pUDT->m_bBroken会被设置，UDT Socket的状态也可能会发生变化，此时等待过程会结束；或者UDT Socket处于同步接收状态，则无论是否有新连接，等待过程都会尽快结束。

4. 等待连接的过程意外退出，也就是在没有等到新连接进来的情况下等待过程就退出了的情况下，抛出异常退出。如果UDT Socket处于同步接收状态，抛出某个类型的异常，否则抛出另外一种类型的异常来表示UDT Socket被关闭了。这个地方的逻辑，向调用者展示的异常信息可能具有误导性，比如一个同步接收的UDT Socket被关闭了，向调用者展示的信息似乎仍然表明，UDT Socket是由于同步接收的问题而没有等到新连接进来才退出的。

5. 等到了新连接进来的情况下，将发起端的网络地址拷贝给调用者。

6. 将新UDT Socket的SocketID返回给调用者。
可以看到，UDT::accept()这个地方是一个典型的生产者-消费者模型。UDT::accept()是消费者，消费的对象是ls->m_pQueuedSockets中的UDT Socket。我们分析UDT::accept()函数的实现，只能看到这个关于生产-消费的故事的一半，另一半关于生产的故事则需要通过更仔细地分析CRcvQueue::worker()的执行来了解了。

总结一下这几个操作与Listening Socket状态变化之间的关系，如下图所示：

![](https://www.wolfcstech.com/images/1315506-2b8081c3aea0670d.png)

Done.
