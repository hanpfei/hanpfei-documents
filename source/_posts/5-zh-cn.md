---
title: HTTP/2规范：5. 流和多路复用
date: 2016-10-29 12:30:49
categories: HTTP2相关规范
---

“流”是一个HTTP/2连接内在客户端和服务器间独立的双向的交换的帧序列。流具有一些重要的特性：
* 单个的HTTP/2连接可以包含多个并发打开的流，各个终端多个流的帧可以交叉。
* 流可以单方面地建立和使用，或由客户端或服务器共享。
* 流可以被任何一端关闭。
* 流中帧的发送顺序是值得注意的。接收者以它们收到帧的顺序处理。特别的，[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧和[DATA](http://httpwg.org/specs/rfc7540.html#DATA)帧在语义上是非常重要的。
* 流由一个整数标识。流标识符由发起流的一端来赋值。

<!--more-->

## 5.1 流状态

一个流的生命周期如下面的[图 2](http://httpwg.org/specs/rfc7540.html#StreamStatesFigure)所示。

```
            
                             +--------+
                     send PP |        | recv PP
                    ,--------|  idle  |--------.
                   /         |        |         \
                  v          +--------+          v
           +----------+          |           +----------+
           |          |          | send H /  |          |
    ,------| reserved |          | recv H    | reserved |------.
    |      | (local)  |          |           | (remote) |      |
    |      +----------+          v           +----------+      |
    |          |             +--------+             |          |
    |          |     recv ES |        | send ES     |          |
    |   send H |     ,-------|  open  |-------.     | recv H   |
    |          |    /        |        |        \    |          |
    |          v   v         +--------+         v   v          |
    |      +----------+          |           +----------+      |
    |      |   half   |          |           |   half   |      |
    |      |  closed  |          | send R /  |  closed  |      |
    |      | (remote) |          | recv R    | (local)  |      |
    |      +----------+          |           +----------+      |
    |           |                |                 |           |
    |           | send ES /      |       recv ES / |           |
    |           | send R /       v        send R / |           |
    |           | recv R     +--------+   recv R   |           |
    | send R /  `----------->|        |<-----------'  send R / |
    | recv R                 | closed |               recv R   |
    `----------------------->|        |<----------------------'
                             +--------+

       send:   endpoint sends this frame
       recv:   endpoint receives this frame

       H:  HEADERS frame (with implied CONTINUATIONs)
       PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
       ES: END_STREAM flag
       R:  RST_STREAM frame

          
```
图 2：流状态

注意，这个图只显示了流状态的转换及会影响到那些转换的帧和标记。就此而言，[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧不引发这种状态转变，它们是它们跟着的[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)或[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧的有效部分。为了转变状态，END_STREAM标记被作为独立于携带它的帧的事件处理；设置了END_STREAM标记的[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧可能导致两次状态转变。

端点都有一个自己的流状态的主视图，它在帧传输的过程中可能会有不同。终端间不会协调流的创建；终端会单方面地创建流。发送[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)之后，状态上不匹配的负面结果被限制在"closed"状态，在关闭之后的某个时间可能会收到帧。

流具有如下的这些状态：

**idle:**

所有的流以"idle"状态开始。
下面的状态是以这个状态为起始状态的有效目的状态：
* 发送或接收一个[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧使这个流进入"open"状态。选择流标识符的方法如[Section 5.1.1](http://httpwg.org/specs/rfc7540.html#StreamIdentifiers) 中所述。相同的[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧也可能使一个流立即进入"half-closed"状态。
* 在另一个流上发送一个[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧保留被标识的idle流以备后用。保留的流的流状态转变为"reserved (local)"。
* 在另一个流上接收一个[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧保留一个被标识的idle流以备后用。保留的流的流状态转变为"reserved (remote)"。
* 注意[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧不在idle流上发送，而是在Promised Stream ID字段中引用新的被保留的流。

处于这种状态的流上接收到了任何[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)或[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)之外的帧都 **必须(MUST)** 被作为一个类型是[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))处理。

**reserved (local):**

处于"reserved (local)"状态的流都已经通过发送[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)做了保证。[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧通过将流与一个远方的对端节点初始化的已打开的流关联，来保留一个idle流(参见[Section 8.2](http://httpwg.org/specs/rfc7540.html#PushResources))。
这种状态，只有如下的这些转换是可能的：
* 终端可以发送一个[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧。这使得流打开并进入"half-closed (remote)"状态。
* 或者终端可以发送一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧，来使流进入"closed"状态。这将释放流的保留状态。

处于这种状态下的端点 **一定不能(MUST NOT)** 发送[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)，[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)，或[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)之外类型的帧。

这种状态下 **可能会(MAY)**  收到[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)或[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)帧。处于这种状态的流上接收到了任何除[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)，[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)，或[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)之外类型的帧，都 **必须(MUST)** 被作为一个类型[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))处理。

**reserved (remote):**

处于"reserved (remote)"状态的流是已经由远端的端点保留的流。

这种状态下只有如下的状态转换可能发生：
* 接收到[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧使得流进入"half-closed (local)"状态。
* 某一端可以发送一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧使得流进入"closed"状态。这将释放流的保留状态。

这种状态下端点 **可能会(MAY)** 发送一个[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)帧来改变保留的流的优先级。处于这种状态下时，端点 **一定不能(MUST NOT)** 发送除[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)，[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，或[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)之外类型的帧。

处于这种状态的流上接收到了任何除[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)，[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)，或[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY) 之外类型的帧，都 **必须(MUST)** 被作为一个类型[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))处理。

**open:**

两端的端点都可以使用处于"open"状态的流来发送任何类型的帧。在这种状态下，发送端点观察广告的stream-level flow-control限制 ([Section 5.2](http://httpwg.org/specs/rfc7540.html#FlowControl))。

从这种状态开始，每个端点都可以发送一个设置了END_STREAM标记的帧，来使流进入到某种"half-closed"状态。端点发送END_STREAM标记使得流的状态变为"half-closed (local)"；而端点接收END_STREAM标记使得流的状态变为"half-closed (remote)"。

每个端点都可以在这种状态下发送一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧，来使它立即变为"closed"状态。

**half-closed (local):**

处于"half-closed (local)"状态的流不能被用于发送帧，除了[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)，和[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)。

当接收到一个包含END_STREAM标记的帧，或某个端点发送了一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧时，流从这种状态转为"closed"状态。

这种状态下的端点可以接收任何类型的帧。要继续接收flow-controlled帧的话必须先通过[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)帧来提供flow-control credit。在这种状态下，接收者可以忽略[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)帧，其可能在END_STREAM标记被设置了的帧发送后一段时间到达。

这种状态下接收的[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)帧被用于改变依赖于标识的流的流的优先级。

**half-closed (remote):**

处于"half-closed (remote)"状态的流不能再被端点用于发送帧。在这种状态下，端点不再有义务维护接收者flow-control窗口。

对于这种状态的流，如果一个端点收到了额外的帧，除[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)，或[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)外，它 **必须(MUST)** 以一个类型为[STREAM_CLOSED](http://httpwg.org/specs/rfc7540.html#STREAM_CLOSED)的流错误([Section 5.4.2](http://httpwg.org/specs/rfc7540.html#StreamErrorHandler))来响应。

处于"half-closed (remote)"状态的流可被端点用于发送任何类型的帧。在这种状态下，端点将继续观察广告的stream-level flow-control限制 ([Section 5.2](http://httpwg.org/specs/rfc7540.html#FlowControl))。

流可通过发送包含了END_STREAM标记的帧，或者对端发送了一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧时，将状态转为"closed"。

**closed:**

"closed"状态是终止状态。

端点 **一定不能(MUST NOT)** 在一个关闭的流上发送除[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)外的帧。端点在接收到[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)之后接收到了除[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)外的帧， **必须(MUST)** 将它作为一个类型为[STREAM_CLOSED](http://httpwg.org/specs/rfc7540.html#STREAM_CLOSED)的流错误([Section 5.4.2](http://httpwg.org/specs/rfc7540.html#StreamErrorHandler))处理。类似地，终端在收到一个END_STREAM标记被设置的帧之后收到了任何帧，则都必须将其作为类型为 [STREAM_CLOSED](http://httpwg.org/specs/rfc7540.html#STREAM_CLOSED)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))处理，除非帧已经像下面所述被允许了。

在包含了END_STREAM标记的[DATA](http://httpwg.org/specs/rfc7540.html#DATA)或[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧被发送一小段时间之后，这种状态下可以接收[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)或[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧。直到远端端点接收到并处理了[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧或携带有END_STREAM标记的帧，它可能发送这些类型的帧。这种状态下，端点 **必须(MUST)** 忽略接收到的[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)或[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧，尽管端点 **可能(MAY)** 选择将发送END_STREAM之后一段时间到达的帧作为类型为[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))处理。

在关闭的流上可以发送[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)帧来调整依赖于关闭的帧的流的优先级。终端 **应该(SHOULD)** 处理[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)帧，尽管它们可以在流已经被从依赖树中移除时忽略它(参见[Section 5.3.4](http://httpwg.org/specs/rfc7540.html#priority-gc))。

如果是由于发送了一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧而进入的这种状态，接收[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)的终端在流上可能已经发送——或入队即将发送——的帧不能被撤回。在终端已经发送了一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)帧之后，它 **必须(MUST)** 忽略在关闭的流上接收的帧。终端 **可以(MAY)** 选择限制时间，在那段时间之后接收到的帧被忽略及被当做错误来处理。

发送了[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)之后接收到的flow-controlled帧(比如，[DATA](http://httpwg.org/specs/rfc7540.html#DATA))被计入flow-control窗口。即使这些帧可能被忽略，因为它们是在发送者接收到[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)之前发送的，发送者将认为这些帧不利于flow-control窗口。

终端在发送了[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)之后可能收到[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧。[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)使得流进入"reserved"状态，即使关联的流已经被重置。因此，需要一个[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)来关闭一个unwanted promised流。

这份文档中其它地方没有更多特别说明的，实现 **应该(SHOULD)** 将描述中没有明确表示允许的帧的状态的接收作为一个类型为[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section 5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))。注意，任何状态下的流都可以发送和接收[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)。类型未知的帧被忽略。

可以在[Section 8.1](http://httpwg.org/specs/rfc7540.html#HttpSequence)中找到HTTP请求/响应交换过程中状态转变的例子。在Sections [8.2.1](http://httpwg.org/specs/rfc7540.html#PushRequests)和[8.2.2](http://httpwg.org/specs/rfc7540.html#PushResponses)可以找到服务器推送时状态转变的例子。

### 5.1.1 流标识符

流由一个31位无符号整型值标识。由客户端初始化的流 **必须(MUST)** 使用奇数的流标识符；而那些由服务器初始化的则 **必须(MUST)** 使用偶数的流标识符。值为零(0x0)的流标识符被用于连接控制消息；零标识符不能被用于创建一个新的流。

升级到HTTP/2(参见[Section 3.2](https://http2.github.io/http2-spec/#discover-http))的HTTP/1.1请求，使用流标识符一(0x1)来响应。在升级完成后，流0x1对于客户端而言处于"half-closed (local)"状态。因此，流0x1可被从HTTP/1.1升级的客户端选做一个新的流标识符。

一个新建立的流的标识符在数值上 **必须(MUST)** 大于初始化流的端点已经打开或保留的流。这管理了使用[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧打开的流，和使用[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧保留的流。接收到一个意外的流标识符的端点 **必须(MUST)** 回应一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

第一次使用一个新的流标识符隐含地关闭所有可能已经被那个端点以更小的流标识符初始化的处于"idle"状态的流。比如，如果一个客户端在流7上发送了一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧，而不曾在流5上发送帧，则当第一个流7的帧被发送或接收时流5将进入"closed"状态。

流标识符不能被复用。长期存活的连接可能使端点耗尽流标识符的可用范围。客户端在无法创建新的流标识符时可以为新流创建一个新的连接。服务器在无法建立新流时可以发送[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧，以便于强制客户端为新流打开一个新的连接。

### 5.1.2 流的并发
一个端点可以通过[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧的[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS)参数限制同时活跃的并发流的个数。最大的并发流设定特定于每个端点，且只应用于接收设定的对端节点。即，客户端指定服务器可初始化的并发流的最大数量，而服务器指定客户端可初始化的并发流的最大个数。

处于"open"状态或处于某种"half-closed"状态的流被计入一个端点允许打开的流的最大个数。处于这三种状态中任一种的流被计入广告的[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS) 设定的限制。处于某种"reserved"状态的流不被计入这种限制。

终端 **一定不能(MUST NOT)** 超出它们的对端设置的限制。接收到一个导致它广告的并发流限制越界的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧的端点，必须把这作为一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)或[REFUSED_STREAM](https://http2.github.io/http2-spec/#REFUSED_STREAM)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。错误码的选择决定于终端是否想要启用自动的重试(参考[Section 8.1.4](https://http2.github.io/http2-spec/#Reliability)获取更多细节)。

一个想要降低[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS)的值到某个小于当前打开流的个数的值的端点，可以关闭超出新值的流或允许流结束。

## 5.2 Flow Control
在TCP连接之上使用流来实现多路复用引入了一些冲突，这导致阻塞的流。flow-control方案确保相同连接上的流不会破坏性地相互干扰。独立的流和整个连接都可以使用flow control。

HTTP/2通过使用WINDOW_UPDATE帧来提供flow control ([Section 6.9](https://http2.github.io/http2-spec/#WINDOW_UPDATE))。

### 5.2.1 Flow-Control原理
HTTP/2流的flow control旨在允许在不改变协议的情况下使用多个flow-control算法。HTTP/2 flow control具有如下的特性：
1. Flow control是特定于连接的。每种flow control类型都是处于一个single hop的两个端点之间的，而不是整个端到端的路径。
2. Flow control是基于WINDOW_UPDATE帧的。接收者广告它们准备在流中和在整个连接中接收的字节数。这是一种credit-based方案。
3. Flow control的方向是接收者提供的整体控制。一个接收者 **可能(MAY)** 选择为每个流或整个连接设置任何想要的窗口大小。一个发送者 **必须(MUST)** 尊重接收者应用的flow-control限制。作为接收者的客户端，服务器，和intermediaries都独立地广告它们的flow-control限制，同时在作为发送者时要服从它们发送时的对端设置的flow-control限制。
4. 新的流和整个连接的flow-control窗口初始值都是 65,535字节。
5. 帧的类型决定了flow control是否应用于一个帧。这份文档中描述的帧，只有DATA帧受flow control的支配；所有其它类型的帧不消费广播的flow-control窗口中的空间。这确保重要的控制帧不会被flow control阻塞。
6. Flow control不能被禁用。
7. HTTP/2只定义了WINDOW_UPDATE帧([Section 6.9](https://http2.github.io/http2-spec/#WINDOW_UPDATE))的格式和语义。这份文档不规定接收者如何决定何时发送这种帧及发送的值，也不描述发送者如何选择发送包。实现能够选择任何满足它们需要的算法。

实现也要负责管理请求和响应要如何基于优先级来发送，选择如何避免请求的队首阻塞，及管理新流的创建。为此选择的算法可以与任何的flow-control算法交互。

### 5.2.2 对Flow Control的适当使用

Flow control是为保护在资源受限的条件下操作的终端而定义的。比如，一个代理需要在许多连接间共享内存，而且可能具有一个较慢的上行连接和一个较快的下行连接。Flow-control解决了接收者不能处理一个流中的数据，而想要继续处理相同连接上其它流中的数据的问题。

无需这种能力的部署可以广告一个最大的flow-control 窗口大小(2^31-1)，并且可以通过在收到任何数据时发送一个WINDOW_UPDATE帧来维护这个窗口。这将有效地为那个接收者禁用flow control。相反地，发送者要总是受制于接收者广告的flow-control窗口。

资源受限条件下的部署（比如，内存）可以通过flow control来限制一个对端可以消耗的内存数量。然而请注意，如果在不了解带宽－延迟(参见 [[RFC7323]
](https://http2.github.io/http2-spec/#RFC7323))的情况下启用flow control，则可能导致对于可用的网络资源次优的使用。

即使对当前的带宽－延迟有了充分的了解，flow-control的实现也可能是非常困难的。当使用flow control时，接收者 **必须(MUST)** 及时地从TCP的接收缓冲区中读取。当决定性的帧，比如 [WINDOW_UPDATE](https://http2.github.io/http2-spec/#WINDOW_UPDATE)没有被读取并起作用时，这个操作失败的话可能会导致一个死锁。

## 5.3流优先级

客户端可以通过在打开流的HEADERS帧([Section 6.2](https://http2.github.io/http2-spec/#HEADERS))中包含优先级信息来给一个新流分配一个优先级。在其它任何时间，PRIORITY帧([Section 6.3](https://http2.github.io/http2-spec/#PRIORITY))可被用于改变流的优先级。

优先级的目的是，使一个端点在管理多个并发的流时，可以表达它希望连接的对端分配资源的倾向性。最重要的是，当发送能力有限时，优先级可以用于选择传送帧的流。

流的优先级可以通过将它们标记为依赖其它流的完成([Section 5.3.1](https://http2.github.io/http2-spec/#pri-depend))来设置。每个依赖都被分配一个相对的权值，一个用于决定依赖于相同流的流在可用资源分配时的相对占比的数字。

显式的为流设置优先级被输入一个优先级进程。它不保证一个流相对于其它流任何特定的处理或传输顺序。一个终端不能强迫它的对端使用优先级来以一个特定的顺序处理并发的流。因而优先级的表达只是一个建议。

消息中优先级信息可以省略。默认的值先于显式的提供的值([Section 5.3.5](https://http2.github.io/http2-spec/#pri-default))使用。


### 5.3.1 流依赖

每个流都可以显式地被设置为依赖另一个流。包含一个依赖表达了标识的流相对于它所依赖的流分配资源上的倾向。

一个不依赖于任何其它流的流依赖于流0x0。换句话说，不存在的流0是整棵树的根。

依赖于另一个流的流是一个依赖流。被别的流依赖的流是父流。依赖于一个不在树中的流——比如这个流处于"idle"状态——将使那个流具有默认的优先级([Section 5.3.5](https://http2.github.io/http2-spec/#pri-default))。

当依赖于另一个流时，流将作为父流的新的依赖者而添加。依赖于相同流的流相互之间不进行排序。比如，如果流B和C依赖于流A，然后如果创建了流D依赖于流A，则B，C和D可以以任何的依赖顺序依赖A.

```
    A                 A
   / \      ==>      /|\
  B   C             B D C

```
图2：创建默认依赖的例子

一个独占的标记可以插入一个新的依赖层级。独占标记使得流成为它的父流单独的依赖者，这使得其它的依赖变得依赖于独占的流。在前面的例子中，如果流D被创建为独占的依赖于流A，则D将变成B和C依赖的父流。

```
                      A
    A                 |
   / \      ==>       D
  B   C              / \
                    B   C
```
图4：创建独占依赖的例子

在依赖树内部，一个依赖流只有在它所依赖的所有流(直到0x0的父流链)都被关闭了，或者它们不可能产生进展时，才 **应该(SHOULD)** 被分配资源。

流不能依赖于它自己。一个终端必须把这当成是一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。

### 5.3.2 依赖权值

所有的依赖流都被分配一个介于1到256(包含)的整型权值。

具有相同父流的流 **应该(SHOULD)** 依照它们的权值来分配资源的比重。如果流B依赖于流A，权值为4，流C依赖于流A，权值为12，流A无法再取得进展，理想情况下分配给流B的资源应该是分配给流C的三分之一。

### 5.3.3 改变优先级

使用[PRIORITY](https://http2.github.io/http2-spec/#PRIORITY)帧来改变流的优先级。设置一个依赖使得流依赖于所标识的父流。

如果父流的优先级改变了的话，依赖流将与它们的父流一同改变。为一个改变优先级的流设置独占依赖，使得对新的父流的依赖依赖于改变了优先级的流。

如果让一个流依赖于一个原本依赖它的流，则前面的依赖流将首先被改变依赖关系而依赖于被改变了优先级的流的前一个父流。被改变的流的权值保持不变。

比如，考虑一颗依赖树，其最初的状态为B和C依赖于A，D和E依赖于C，而F依赖于D。如果让A依赖于D，则D将取代A的位置。所有的其它依赖关系保持不变，除了F，如果优先级是独占的话它将依赖于A。

```

    x                x                x                 x
    |               / \               |                 |
    A              D   A              D                 D
   / \            /   / \            / \                |
  B   C     ==>  F   B   C   ==>    F   A       OR      A
     / \                 |             / \             /|\
    D   E                E            B   C           B C F
    |                                     |             |
    F                                     E             E
               (intermediate)   (non-exclusive)    (exclusive)
```
图5：依赖重排序的例子

### 5.3.4 优先级状态管理

当一个流被从优先级状态依赖树中移除时，依赖它的流可以依赖于关闭的流的父流。新的依赖的权值被重新计算，关闭的流的依赖权值被等比例地分配给依赖它的流。

从依赖树移除流导致某些优先级信息丢失。具有相同父流的流之间相互共享资源，这意味着如果那个集合中的流关闭了或阻塞了，则分配给那个流的多余资源将被分配给那个流的直接邻居。然而，如果共同的依赖被从树中移除了，则那些流将与下一个最高级别上的流共享资源。

比如，假设流A和B具有相同的父流，而流C和D都依赖于流A。在移除流A之前，如果流A和D无法继续进行，则流C接收流A的所有资源。如果流A被从树中移除了，则流A的权值被分配给C和D。如果流D仍然能够继续进行，则流C接收的资源将响应比例的减少。对于相同的初始权值，C接收三分之一，而不是一半的可用资源。

在创建了对于那个流的一个依赖的优先级信息发生改变时流可以进入"closed"状态。如果一个依赖中标识的流没有关联优先级信息，则依赖流将被分配一个默认优先级([Section 5.3.5](https://http2.github.io/http2-spec/#pri-default))。这潜在地创建次优的优先级，因为流可能被分配一个与它想要的优先级不同的优先级。

要避免这些问题，终端 **应该(SHOULD)** 在流进入closed状态一段时间内保持流的优先级状态。保持状态的时间越长，流被分配了错误的或默认的优先级值的概率就越低。

类似的，处于"idle"状态的流可以被分配优先级，或变成其它流的父流。这可以在依赖树中创建分组节点，这能够更灵活地表达优先级。Idle流的初始优先级为默认优先级([Section 5.3.5](https://http2.github.io/http2-spec/#pri-default))。

对于不被计入[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS)所设置的限制的流的优先级信息的保留可能给一个端点带来巨大的负担。因而，保留的优先级状态 **可能(MAY)** 是有限的。

 一个端点为优先级维护的额外状态的数量可能依赖于负载；在高负载下，优先级状态可被丢弃以限制资源的消耗。在极端情况下，一个端点甚至可以丢弃活跃的或保留的流的优先级状态。如果应用了一个限制，终端 **应该(SHOULD)** 至少维护[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS)设定允许的个数的流的状态。实现也 **应该(SHOULD)** 试着保留优先树中活跃的流的的状态。

如果它为此保留了足够多的状态，则一个端点在收到改变已关闭的流的优先级的[PRIORITY](https://http2.github.io/http2-spec/#PRIORITY)帧时应该改变依赖于它的流的优先级。

### 5.3.5 默认优先级

所有流初始时都会被分配一个非独占的对于流0x0的依赖。推送的流([Section 8.2](https://http2.github.io/http2-spec/#PushResources))初始时依赖于它们关联的流。在这些情况下，流被分配一个值为16的默认权值。

## 5.4 错误处理

HTTP/2分帧允许两类错误：
* 一种错误条件描述了整个连接都是不可用的，这是连接错误。
* 单独的流中的错误是流错误。

在[Section 7](https://http2.github.io/http2-spec/#ErrorCodes)中列出了所有的错误码。

### 5.4.1 连接错误处理

连接错误是阻止了帧层的进一步处理或破坏了连接状态的错误。

遇到了一个连接错误的终端 **应该(SHOULD)** 首先发送一个带有从对端成功接收的最后一个流的流标识符的[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧([Section 6.8](https://http2.github.io/http2-spec/#GOAWAY))。[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧包含了指示连接为什么终止的错误码。在为错误情况发送给了[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧之后，终端 **必须(MUST)** 关闭TCP连接。

[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)可能没有被接收端可靠的接收到([[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 6.6](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#persistent.tear-down)描述了立即的连接关闭是如何导致数据丢失的)。在连接错误的事件中，[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)只是提供了最好的努力尝试与对端就为什么连接被终止进行沟通。

终端可以在任何时候结束一个连接。特别的，一个终端 **可以(MAY)** 选择将一个流错误作为一个连接错误。终端 **应该(SHOULD)** 在结束一个连接时发送一个[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧，提供发生那种状况的信息。

### 5.4.2 流错误处理

流错误是只与特定的流相关而不影响其它流的处理的错误。

探测到流错误发生的端点发送一个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧([Section 6.4](https://http2.github.io/http2-spec/#RST_STREAM))，其中包含了错误发生的流的流标识符。[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧包含指示错误类型的错误码。

[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)是终端在一个流上可以发送的最后的帧。发送了[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧的端点 **必须(MUST)** 准备接收远端的端点为发送准备的已经发送或入队的任何帧。这些帧可以忽略掉，除了改变连接状态的那些(比如为首部压缩([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))或flow control维护的状态)。

通常，一个终端 **不应该(SHOULD NOT)** 为任何流发送多个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧。然而，如果一个端点在一个round-trip时间之后在一个已关闭的流上接收到了帧时， **可以(MAY)** 发送额外的[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧。这个行为被允许用于处理行为不端的实现。

要避免循环，一个终端 **一定不能(MUST NOT)** 给一个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧发送一个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)。

### 5.4.3 连接终止

如果在流处于"open"或"half-closed"状态时，TCP连接被关闭或重置，则影响到的流不能自动地重试(参见[Section 8.1.4](https://http2.github.io/http2-spec/#Reliability)来了解细节)。

## 5.5 扩展HTTP/2

HTTP/2允许扩展协议。在这一节描述的限制之内，协议扩展可以被用于提供额外的服务或改变协议的某些方面。扩展只有在单独的HTTP/2连接的范围内才是有效的。

这应用于这份文档中定义的协议元素。这不影响为扩展HTTP而已有的选项，比如定义新的方法，状态码，或首部字段。

扩展可以使用新的帧类型([Section 4.1](https://http2.github.io/http2-spec/#FrameHeader))，新的设置([Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues))，或新的错误码([Section 7](https://http2.github.io/http2-spec/#ErrorCodes))。注册被建立来管理这些扩展点：帧类型([Section 11.2](https://http2.github.io/http2-spec/#iana-frames))，设置([Section 11.3](https://http2.github.io/http2-spec/#iana-settings))和错误码([Section 11.4](https://http2.github.io/http2-spec/#iana-errors))。

实现 **必须(MUST)** 忽略扩展的协议元素未知或不支持的值。实现 **必须(MUST)** 丢弃具有未知的或不支持的类型的的帧。这意味着任何这种扩展点可以被扩展安全的使用而无需事先的安排或协商。然而，在一个首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))中间出现扩展帧是不允许的；这必须被作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

可能会改变已有的协议组件的语义的扩展 **必须(MUST)** 在使用前进行协商。比如，一个扩展修改了[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧的格式，则在对端给出了一个积极的信号表示接受之前是不能使用它的。在这种情况下，在修改过的格式起作用之前也可能需要做调整。注意，将[DATA](https://http2.github.io/http2-spec/#DATA)帧之外的其它类型帧当做flow controlled帧处理就是这样的在语义上的修改，这只能通过协商完成。

这份文档不要求一个特定的方法协商对于一个扩展的使用，但注意，一个设置([Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues))可被用于那一目的。如果连接的两端都设置了一个值，表明想要使用扩展，则扩展可以使用。如果设置被用于扩展协商，则初始值 **必须(MUST)** 是扩展被禁用。
