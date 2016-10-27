---
# 摘要
这份规范描述了一种 超文本传输协议(HTTP) 语义的优化表达，被称为HTTP 版本 2(HTTP/2)。HTTP/2通过引入首部字段压缩，及允许在相同的连接上并发的进行多个数据交换，而使得对网络资源的使用更高效，且获得可感知的延迟降低。它还引入了服务器向客户端未经请求的推送语义。

这份规范是HTTP/1.1消息语法的替代品，但并不废弃它。HTTP既有的语义保持不变。

# [1. 简介](http://httpwg.org/specs/rfc7540.html#intro)

超文本传输协议(HTTP)是一个取得了广泛成功的协议。然而，HTTP/1.1使用底层传输模块([[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)，[Section 6](http://httpwg.org/specs/rfc7230.html#connection.management))的方式具有一些特性，那对今天的应用的整体性能产生了负面的影响。

特别地，同一时间在一个给定的TCP连接上，HTTP/1.1只允许发起一个请求。虽然HTTP/1.1加入了请求管线，但这只是部分地解决了请求并发执行的问题，且依然会被队头阻塞问题困扰。因此，那些需要发起大量请求的HTTP/1.0和HTTP/1.1客户端会通过与同一个服务器建立多个连接的方式来实现并发从而降低延迟。

此外，HTTP首部字段常常是重复而冗长的，这导致了不必要的网络流量消耗，且导致初始的[TCP](http://httpwg.org/specs/rfc7540.html#TCP)[TCP]拥塞窗口被快速填满。在一个新的TCP连接上发起多个请求时，这导致过多的延迟。

HTTP/2解决了这些问题，它定义了一种优化的HTTP语义到一个底层连接的映射。特别地，它允许同一个连接上请求和响应消息的交叉，并使用了一种HTTP首部字段高效的编码。它也允许请求优先级，使得更重要的请求更快地完成，这进一步提高了性能。

产生的协议相对于HTTP/1.x而言，由于使用了更少的TCP连接而对网络更加友好。这意味着与其它流和存活的更久的连接更少的竞争，这反过来又导致对于可用网络能力更好的利用。

最后，HTTP/2也通过使用二进制格式的消息帧而使得对消息的处理更高效。
# [2. HTTP/2 协议总览](http://httpwg.org/specs/rfc7540.html#Overview)

HTTP/2为HTTP语义提供了一个优化的传输方式。HTTP/2支持HTTP/1.1所有的核心功能，但在一些方面又更加高效。

HTTP/2中基本的协议单元是一个帧([Section4.1](http://httpwg.org/specs/rfc7540.html#FrameHeader))。每一个帧类型服务于一个不通的目的。比如，[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)和[DATA](http://httpwg.org/specs/rfc7540.html#DATA)帧构成了HTTP请求和响应的基础([Section8.1](http://httpwg.org/specs/rfc7540.html#HttpSequence))；其它的帧类型，比如[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)，[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，和[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)被用于支持其它的HTTP/2功能。

请求的多路复用通过使得每个HTTP请求/响应交换关联它自己的流([Section5](http://httpwg.org/specs/rfc7540.html#StreamsLayer))来实现。流在很大程度上是相互独立的，因而一个阻塞或止步不前的请求或响应不会阻止其它流的进程。

Flow控制和优先级确保高效地使用多路复用的流是可能的。Flow控制([Section5.2](http://httpwg.org/specs/rfc7540.html#FlowControl))帮助确保只有能被接收者使用的数据才被传输。优先级([Section5.3](http://httpwg.org/specs/rfc7540.html#StreamPriority))确保有限的资源可以被首先导到最终要的流。

HTTP/2添加了一种新的交互模式，服务器可以借以向一个客户端推送响应([Section8.2](http://httpwg.org/specs/rfc7540.html#PushResources))。服务器推送允许一个服务器推测性地发送服务器预期客户端可能需要的数据给一个客户端，权衡比较网络流量的消耗和一个潜在延迟的增加。服务器通过合成一个请求来做到这一点，该请求以一个[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)来发送。然后服务器可以为合成的请求在另一个流上发送一个响应

由于一个连接中HTTP首部字段可能包含大量冗余数据，包含它们的帧被压缩([Section4.3](http://httpwg.org/specs/rfc7540.html#HeaderBlock))。这对于常见情况下的请求大小具有特别有利的影响，它允许多个请求被压缩为一个包。

## [2.1 文档组织](http://httpwg.org/specs/rfc7540.html#rfc.section.2.1)

HTTP/2规范被分为了四个部分：
* 启动HTTP/2 ([Section3](http://httpwg.org/specs/rfc7540.html#starting))描述了一个HTTP/2连接是如何初始化的。 
* 帧([Section4](http://httpwg.org/specs/rfc7540.html#FramingLayer))和流([Section5](http://httpwg.org/specs/rfc7540.html#StreamsLayer))层描述了HTTP/2帧被结构化并构成多路复用的流的方式。
* 帧([Section6](http://httpwg.org/specs/rfc7540.html#FrameTypes))和错误([Section7](http://httpwg.org/specs/rfc7540.html#ErrorCodes))定义包括了HTTP/2中使用的帧类型和错误类型的细节。
* HTTP映射([Section8](http://httpwg.org/specs/rfc7540.html#HTTPLayer))和额外的需求([Section9](http://httpwg.org/specs/rfc7540.html#HttpExtra))描述了HTTP语义如何使用帧和流来表述。

尽管帧和流层的一些概念与HTTP是隔离的，但这份规范不定义一个完整的通用帧层。帧和流层被裁剪为满足HTTP协议和服务器推送的需要。

## [2.2 约定和术语](http://httpwg.org/specs/rfc7540.html#rfc.section.2.2)
这份文档中关键字"MUST"，"MUST NOT"，"REQUIRED"，"SHALL"，"SHALL NOT"，"SHOULD"，"SHOULD NOT"，"RECOMMENDED"，"MAY"，和"OPTIONAL"的解释在[RFC 2119](http://httpwg.org/specs/rfc7540.html#RFC2119)[RFC2119]中描述。

所有的数字值以网络子节序。值是无符号的除非特别指明。字面值以十进制或在适当的时候以16进制提供。十六进制字面值以0x为前缀以与十进制字面值进行区分。

使用了下面的术语：

**客户端(client)**：初始化一个HTTP/2连接的终端。客户端发送HTTP请求并接收HTTP响应。

**连接(connection)**：两个终端之间的一个传输层连接。

**连接错误(connection error)**：影响整个HTTP/2连接的一个错误。

**终端(endpoint)**：连接中的客户端或服务器。

**帧(frame)**：一个HTTP/2连接内最小的通信单元，由一个首部和根据帧类型组织的一个可变长度的字节序列组成。

**对端(peer)**：一个终端。当讨论一个特定的终端时，"对端(peer)"指所讨论的主要科目的远端的终端。

**接收者(receiver)**：接收帧的终端。

**发送者(sender)**：传送帧的终端。

**服务器(server)**：接受一个HTTP/2连接的终端。服务器接收HTTP请求并发送HTTP响应。

**流(stream)**：HTTP/2连接内一个双向的帧流。

**流错误(stream error)**：关于独立的HTTP/2流的错误。

最后，术语"网关(gateway)"，"intermediary"，"代理(proxy)"，和"隧道(tunnel)"在[[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 2.3](http://httpwg.org/specs/rfc7230.html#intermediaries)中定义。Intermediaries在不同的时间客户端和服务器的角色都扮演。

术语"载荷体(payload body)"在[[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 3.3](http://httpwg.org/specs/rfc7230.html#message.body)中定义。

# [3. 启动 HTTP/2](http://httpwg.org/specs/rfc7540.html#starting)
一个HTTP/2连接是一个运行于一个TCP连接([[TCP]
](http://httpwg.org/specs/rfc7540.html#TCP))之上的应用层协议。客户端是TCP连接的发起者。

HTTP/2使用了与HTTP/1.1所使用的相同的"http"和"https" URI schemes。HTTP/2共享了相同的默认端口号："http" URIs的是80，"https" URIs的是443。作为结果，HTTP/2的实现在为处理诸如 http://example.org/foo 或 https://example.com/bar 这样的URIs的目标资源的请求时，需要首先发现upstream server(客户端希望建立连接的中间对端)是支持HTTP/2的。

对于"http"和"https" URIs，确定是否支持HTTP/2的方法是不同的。确定"http" URIs是否支持HTTP/2的方法在[Section3.2](http://httpwg.org/specs/rfc7540.html#discover-http)中描述。确定"https" URIs是否支持HTTP/2的方法在[Section3.3](http://httpwg.org/specs/rfc7540.html#discover-https)中描述。

## [3.1 HTTP/2 版本识别](http://httpwg.org/specs/rfc7540.html#versioning)

这份文档中定义的协议具有两个标识符。
* 字符串"h2"标识HTTP/2使用了[传输层安全 (TLS)](http://httpwg.org/specs/rfc7540.html#TLS12)[TLS12]的协议。这个标识符在[TLS 应用层协议协商(ALPN)扩展](http://httpwg.org/specs/rfc7540.html#TLS-ALPN)[TLS-ALPN]字段中使用，及任何运行于TLS之上的HTTP/2被标识的地方。
字符串"h2"被序列化为两个字节值序列的ALPN协议标识符：0x68，0x32。
* 字符串"h2c"标识了HTTP/2运行于明文TCP之上的协议。这个标识符被用于HTTP/1.1 Upgrade首部字段，及任何运行于TCP之上的HTTP/2被标识的地方。
字符串"h2c"是从ALPN标识符空间中预分配的，但描述了一个不使用TLS的协议。

协商"h2"或"h2c"暗含了对这份文档中描述的传输方式，安全，分帧，和消息语义的使用。

## [3.2 启动"http" URIs的HTTP/2](http://httpwg.org/specs/rfc7540.html#discover-http)

一个客户端为一个"http" URI创建一个请求，而又没有先验的关于是否支持HTTP/2的知识，则在下一跳使用HTTP Upgrade机制([[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 6.7](http://httpwg.org/specs/rfc7230.html#header.upgrade))。客户端通过创建一个其中包含了值为"h2c" token的Upgrade首部字段的HTTP/1.1请求来做到这一点。这样一个HTTP/1.1请求**必须(MUST)**精确地包含一个HTTP2-Settings([Section3.2.1](http://httpwg.org/specs/rfc7540.html#Http2SettingsHeader))首部字段。

比如：
```
GET / HTTP/1.1
Host: server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
包含一个载荷体的请求**必须(MUST)**在客户端发送HTTP/2帧之前全文发送。这意味着一个大的请求可能会阻塞对连接的使用直到它被发送完。

如果一个初始的请求与后续的请求的并发很重要，一个OPTIONS请求可被用于执行HTTP/2升级，在消耗额外的一个round trip的情况下。

不支持HTTP/2的服务器可以像Upgrade首部字段不存在一样响应请求：
```
HTTP/1.1 200 OK
Content-Length: 243
Content-Type: text/html

...
```
服务器**必须(MUST)**忽略Upgrade首部字段中的"h2" token。出现"h2" token表示HTTP/2是基于TLS的，它需要像[Section3.3](http://httpwg.org/specs/rfc7540.html#discover-https)中描述的那样进行协商。

支持HTTP/2的服务器以一个101(Switching Protocols)响应来接受升级。在终止了101响应的空行之后，服务器可以开始发送HTTP/2帧了。这些帧**必须(MUST)**包含一个对初始化了升级的请求的响应。

比如：
```
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c

[ HTTP/2 connection ...
```
服务器发送的第一个HTTP/2帧**必须(MUST)**是由一个[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧([Section6.5](http://httpwg.org/specs/rfc7540.html#SETTINGS))构成的服务器连接preface([Section3.5](http://httpwg.org/specs/rfc7540.html#ConnectionHeader))。一旦收到了101响应，客户端**必须(MUST)**发送一个连接preface([Section3.5](http://httpwg.org/specs/rfc7540.html#ConnectionHeader))，其包含一个[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧。

先于upgrade发送的HTTP/1.1请求被分配一个为1的流标识符(参考[Section5.1.1](http://httpwg.org/specs/rfc7540.html#StreamIdentifiers))且具有默认的优先级([Section5.3.5](http://httpwg.org/specs/rfc7540.html#pri-default))。自请求作为一个HTTP/1.1请求被完成之后，流1就是从客户端向服务器的隐含地"half-closed"的了。HTTP/2连接启动之后，流1被用于响应。

### [3.2.1 HTTP2-Settings首部字段](http://httpwg.org/specs/rfc7540.html#Http2SettingsHeader)

从HTTP/1.1升级到HTTP/2的请求**必须(MUST)**准确地包含一个`HTTP2-Settings`首部字段。`HTTP2-Settings`首部字段是一个连接特有的首部字段，它包含了控制HTTP/2连接的参数，以接受请求提供升级的服务器的预期来提供。
```
HTTP2-Settings    = token68
```
如果没有这个首部字段，或这个首部字段出现了多次，服务器**必须不(MUST NOT)**升级到HTTP/2连接。服务器**必须不(MUST NOT)**发送这个首部字段。

`HTTP2-Settings`首部字段的内容是[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧([Section6.5](http://httpwg.org/specs/rfc7540.html#SETTINGS))的载荷，它被编码为一个base64url字符串(即, [[RFC4648]
](http://httpwg.org/specs/rfc7540.html#RFC4648)的[Section 5](https://tools.ietf.org/html/rfc4648#section-5)中描述的URL和文件名安全的Base64编码，但省去了尾部的'='字符)。token68的[ABNF](http://httpwg.org/specs/rfc7540.html#RFC5234)[RFC5234]产品在[[RFC7235]
](http://httpwg.org/specs/rfc7540.html#RFC7235)的[Section 2.1](http://httpwg.org/specs/rfc7235.html#challenge.and.response)中定义。

由于upgrade仅仅是为了应用于直接连接，一个客户端发送了`HTTP2-Settings`首部字段**必须(MUST)**也在Connection首部字段中将`HTTP2-Settings`作为一个连接选项发送，以防止它被转发(参见[[RFC7230]
](http://httpwg.org/specs/rfc7540.html#RFC7230)的[Section 6.1](http://httpwg.org/specs/rfc7230.html#header.connection))。

服务器像其它[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧那样解码并解释这些值。不一定要显式地确认这些设置([Section6.5.3](http://httpwg.org/specs/rfc7540.html#SettingsSync))，因为101响应可作为隐式的确认。在upgrade请求中提供这些值，给客户端提供了一个机会，在从服务器接收任何帧之前提供参数。

## [3.3 启动"https" URIs的HTTP/2](http://httpwg.org/specs/rfc7540.html#discover-https)

创建"https" URI的请求的客户端借助于[应用层协议协商(ALPN)扩展](http://httpwg.org/specs/rfc7540.html#TLS-ALPN)[TLS-ALPN]来使用[TLS](http://httpwg.org/specs/rfc7540.html#TLS12)[TLS12]

TLS之上的HTTP/2使用"h2"协议标识符。客户端**必须不(MUST NOT)**发送协议标识符"h2c"，它也不能被服务器选中；"h2c"描述了一个不使用TLS的协议。

一旦TLS协商完成，客户端和服务器都**必须(MUST)**发送一个连接preface ([Section3.5](http://httpwg.org/specs/rfc7540.html#ConnectionHeader))。

## [3.4 以先验知识启动HTTP/2](http://httpwg.org/specs/rfc7540.html#known-http)

客户端可以通过其它的方式来了解特定服务器支持HTTP/2的情况。比如，*[[ALT-SVC]
](http://httpwg.org/specs/rfc7540.html#ALT-SVC)*描述了一种机制来广告这种能力。

客户端**必须(MUST)**发送连接preface ([Section3.5](http://httpwg.org/specs/rfc7540.html#ConnectionHeader))，它还**可以(MAY)**立即向这个服务器发送HTTP/2帧；服务器可以通过连接preface的出现来识别这些连接。这只影响明文TCP之上的HTTP/2连接的建立；支持TLS之上的HTTP/2的实现**必须(MUST)**使用[TLS中的协议协商](http://httpwg.org/specs/rfc7540.html#TLS-ALPN)[TLS-ALPN]。

同样地，服务器**必须(MUST)**发送一个连接preface([Section3.5](http://httpwg.org/specs/rfc7540.html#ConnectionHeader))。

没有额外信息时，给定服务器先前对于HTTP/2的支持不是该服务器在未来的连接中也支持HTTP/2的强信号。比如，服务器的配置可能改变，可能是为了不同集群服务器实例不同的配置，或者为了网络条件的改变。

## [3.5 HTTP/2 连接Preface](http://httpwg.org/specs/rfc7540.html#ConnectionHeader)
在HTTP/2中，每个终端都需要发送一个连接preface作为在使用的协议的一个最终的确认，并为HTTP/2连接建立初始的设定。客户端和服务器相互发送一个不同的连接preface。

客户端的连接preface以一个24字节的序列开始，其十六进制形式为：
```
0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a
```
即，连接preface以字符串PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n)开始。这个序列后面**必须(MUST)**跟着一个[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧([Section6.5](http://httpwg.org/specs/rfc7540.html#SETTINGS))，它**可能(MAY)**是空的。客户端一收到101(Switching Protocols)响应(表示一个成功的升级)就立即发送客户端连接preface，或作为一个TLS连接的第一个数据字节。如果以服务器对协议的支持情况的先验知识来启动一个HTTP/2连接，客户端连接preface在连接建立时立即发送。

**注意：**选中客户端连接preface以使得大部分的HTTP/1.1或HTTP/1.0服务器和intermediaries不会尝试去处理更多的帧。注意这没有解决[[TALKING]
](http://httpwg.org/specs/rfc7540.html#TALKING)中提出的问题。

服务器连接preface由一个潜在的空[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧([Section6.5](http://httpwg.org/specs/rfc7540.html#SETTINGS))组成，它**必须(MUST)**是服务器在HTTP/2连接中发送的第一帧。

作为连接preface的一部分从对端接收的[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧**必须(MUST)**在发送连接preface之后做确认(参见[Section6.5.3](http://httpwg.org/specs/rfc7540.html#SettingsSync))。

为了避免不必要的延迟，客户端被允许在发送完客户端连接preface之后立即向服务器发送额外的帧，而无需等待接收服务器连接preface。然而，有一点很重要，服务器连接preface [SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧可能包含后续客户端要如何与服务器进行通信所必须的参数。一旦收到了[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)帧，客户端需要按照参数建立连接。在某些配置下，服务器可能会在客户端发送额外的帧之前传送[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)，提供了一个机会来避免这个问题。

客户端和服务器**必须(MUST)**将一个无效的连接preface当作一个类型为[PROTOCOL_ERROR](http://httpwg.org/specs/rfc7540.html#PROTOCOL_ERROR)的连接错误([Section5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))。在这种情况下，[GOAWAY](http://httpwg.org/specs/rfc7540.html#GOAWAY)帧([Section6.8](http://httpwg.org/specs/rfc7540.html#GOAWAY))**可能(MAY)**会被省略，因为一个无效的preface表示对端没有在使用HTTP/2。

# [4. HTTP帧](http://httpwg.org/specs/rfc7540.html#FramingLayer)

建立了HTTP/2连接之后，端点间就可以开始进行帧的交换了。

## [4.1 帧格式](http://httpwg.org/specs/rfc7540.html#FrameHeader)
所有的帧都以一个固定的9字节首部开始，其后紧跟一个可变长度的载荷。
```
 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```
图 1：帧格式
帧首部的字段定义如下：
* **长度(Length)**：帧载荷的长度，以一个无符号24位整数表示。**必须不(MUST NOT)**能发送大于2^14(16,384)的值，除非接收者已经为[SETTINGS_MAX_FRAME_SIZE](http://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_FRAME_SIZE)设置了更大的值。
9字节的帧首部不包含在这个值之内。

* **类型(Type)**：8位的帧类型。帧类型决定了帧的格式和语义。HTTP/2实现**必须(MUST)**忽略并丢弃未知类型的帧。

* **标记(Flags)**：一个特定于帧类型的8位boolean标记保留字段。
标记的语义特定于指示的帧类型。一个特定帧类型中没有定义语义的标记**必须(MUST)**被忽略，且**必须(MUST)**在发送时被复位(0x0)。

* **(R)**：一个保留的1位字段。这个位的语义还没有定义，而在发送时这个位**必须(MUST)**保持复位(0x0)状态，在接收时**必须(MUST)**必须被忽略。

* **流标识符(Stream Identifier)**：流标识符(参见[Section5.1.1](http://httpwg.org/specs/rfc7540.html#StreamIdentifiers))被表示为一个31位无符号整型值。保留0x0值，用于那些与整个连接关联的帧，而不是一个独立的流。

帧载荷的结构和内容完全依赖于帧类型。

## [4.2 帧大小](http://httpwg.org/specs/rfc7540.html#FrameSize)
帧载荷的大小由接收者在[SETTINGS_MAX_FRAME_SIZE](http://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_FRAME_SIZE)设定中通告的最大大小限制。这个设定的值的范围为2^14(16,384)至2^24-1 (16,777,215)，包含。

所有的实现要**必须(MUST)**能够接收并最低限度地处理长度最大在2^14字节，外加9个字节的帧首部([Section4.1](http://httpwg.org/specs/rfc7540.html#FrameHeader))的帧。当描述帧大小时，帧首部的大小不会被包含在内。

**注意：**某些帧类型，比如ING ([Section6.7](http://httpwg.org/specs/rfc7540.html#PING))，会对允许的载荷数据的大小强加额外的限制。

如果一个帧超出了[SETTINGS_MAX_FRAME_SIZE](http://httpwg.org/specs/rfc7540.html#SETTINGS_MAX_FRAME_SIZE)定义的大小，超出了为帧类型定义的任何限制，或者太小以至于无法包含必要的帧数句，终端**必须(MUST)**发送一个错误码[FRAME_SIZE_ERROR](http://httpwg.org/specs/rfc7540.html#FRAME_SIZE_ERROR)。可能改变整个连接状态的帧中的帧大小错误**必须(MUST)**被当作一个连接错误([Section5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))；这包括所有携带首部块的帧([Section4.3](http://httpwg.org/specs/rfc7540.html#HeaderBlock)) (即[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)，[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)，和[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION))，[SETTINGS](http://httpwg.org/specs/rfc7540.html#SETTINGS)，和所有流标识符为0的帧。

终端没有义务用完帧中的所有的可用空间。响应能力可以通过使用比允许的最大大小小的帧来提升。在发送时间敏感的帧时发送大帧可能导致延时（比如[RST_STREAM](http://httpwg.org/specs/rfc7540.html#RST_STREAM)，[WINDOW_UPDATE](http://httpwg.org/specs/rfc7540.html#WINDOW_UPDATE)，或[PRIORITY](http://httpwg.org/specs/rfc7540.html#PRIORITY)），如果传输由于一个大的帧而阻塞，则可能会影响性能。


## [4.3 首部压缩和解压](http://httpwg.org/specs/rfc7540.html#HeaderBlock)
如同在HTTP/1中那样，HTTP/2的一个首部字段是一个名字伴随一个或多个关联的值。HTTP请求和响应消息中会使用首部字段，服务器推送操作(参见[Section8.2](http://httpwg.org/specs/rfc7540.html#PushResources))中也会使用。

首部列表是零个或多个首部字段的集合。当在连接上传输时，首部列表会使用[HTTP首部压缩](http://httpwg.org/specs/rfc7540.html#COMPRESSION)*[COMPRESSION]*被序列化为一个首部块。序列化后的首部块被分为一个或多个字节序列，被称为首部块片段，在HEADERS ([Section6.2](http://httpwg.org/specs/rfc7540.html#HEADERS))，PUSH_PROMISE ([Section6.6](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE))，或CONTINUATION ([Section6.10](http://httpwg.org/specs/rfc7540.html#CONTINUATION))帧的载荷中进行传输。

[Cookie首部字段](http://httpwg.org/specs/rfc7540.html#COOKIE)*[COOKIE]*会被HTTP映射特殊对待(参见[Section8.1.2.5](http://httpwg.org/specs/rfc7540.html#CompressCookie))。

接收终端通过连接首部块的片段来重新组装首部块，然后解压缩首部块并重建首部列表。

一个完整的首部块由这些组成：
* 一个单独的[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)或[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧，其中设置了END_HEADERS标记，或
* 一个[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)或[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)帧，其中END_HEADERS被清除，及一个或多个[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧，其中最后一个[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧设置了END_HEADERS标记

首部压缩是有状态的。压缩上下文和解压上下文被用于整个连接。首部块中的解码错误**必须(MUST)**被作为类型[COMPRESSION_ERROR](http://httpwg.org/specs/rfc7540.html#COMPRESSION_ERROR)的连接错误([Section5.4.1](http://httpwg.org/specs/rfc7540.html#ConnectionErrorHandler))。

每个首部块都被当做一个离散的单元处理。首部块**必须(MUST)**被作为一个连续的帧序列传输，而不能插入其它类型或其它流的帧。[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)或[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧序列中的最后帧设置END_HEADERS标记。[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)或[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧序列中的最后帧设置END_HEADERS标记。这使得首部块在逻辑上等价于一个单独的帧。

首部块字段只能作为[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)，[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)，或[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧的载荷来发送，因为这些帧携带的数据可以修改由接收者维护的压缩上下文。一个接收了[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)，[PUSH_PROMISE](http://httpwg.org/specs/rfc7540.html#PUSH_PROMISE)，或[CONTINUATION](http://httpwg.org/specs/rfc7540.html#CONTINUATION)帧的终端需要重新装配首部块，并执行解压缩，即使帧将被丢弃。如果一个接收者不解压缩首部块，而出现了一个类型为[COMPRESSION_ERROR](http://httpwg.org/specs/rfc7540.html#COMPRESSION_ERROR)的连接错误，则它**必须(MUST)**终止连接。

http://httpwg.org/specs/rfc7540.html#rfc.section.2.2

# 5. 流和多路复用

“流”是一个HTTP/2连接内在客户端和服务器间独立的双向的交换的帧序列。流具有一些重要的特性：
* 单个的HTTP/2连接可以包含多个并发打开的流，各个终端多个流的帧可以交叉。
* 流可以单方面地建立和使用，或由客户端或服务器共享。
* 流可以被任何一端关闭。
* 流中帧的发送顺序是值得注意的。接收者以它们收到帧的顺序处理。特别的，[HEADERS](http://httpwg.org/specs/rfc7540.html#HEADERS)帧和[DATA](http://httpwg.org/specs/rfc7540.html#DATA)帧在语义上是非常重要的。
* 流由一个整数标识。流标识符由发起流的一端来赋值。

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

# 6. 帧定义

这份文档定义了多种帧类型，每种都由一个唯一的8位类型码标识。每种帧类型都服务于建立和管理整个连接或独立的流方面的一个不同的目的。

特定帧类型的传输可能改变连接的状态。如果终端不能维护连接状态视图的一致性，连接内成功的通信将是不可能的。因此，终端之间，关于特定帧的使用对状态所产生的影响具有相同的理解就变得非常重要。

## 6.1 DATA

DATA帧(type=0x0)传送与一个流关联的任意的，可变长度的字节序列。一个或多个DATA帧被用于，比如，携带HTTP请求或响应载荷。

DATA帧也 **可以(MAY)** 包含填充字节。填充字节可以被加进DATA帧来掩盖消息的大小。填充字节是一个安全性的功能；参见 [Section 10.7](https://http2.github.io/http2-spec/#padding)。

```
 +---------------+
 |Pad Length? (8)|
 +---------------+-----------------------------------------------+
 |                            Data (*)                         ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```
图 6：DATA帧载荷


DATA帧包含如下的字段：

**填充长度：**

一个8位的字段，包含了以字节为单位的帧的填充的长度。这个字段是有条件的(如图中的"?"所指的)，只有在PADDED标记设置时才出现。

**数据：**

应用数据。数据的大小是帧载荷减去出现的其它字段的长度剩余的大小。

**填充：**

填充字节包含了无应用语义的值。当填充时填充字节 **必须(MUST)** 被设置为0。接收者没有义务去验证填充，而 **可以(MAY)** 将非零的填充当做一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

DATA帧定义了如下的标记：

**END_STREAM (0x1)：**

当设置了这个标记时，位0表示这个帧是终端将为标识的流发送的最后一帧。设置这个标记使得流进入某种"half-closed"状态或"closed"状态([Section 5.1](https://http2.github.io/http2-spec/#StreamStates))。

**PADDED (0x8):**

当设置了这个标记时，位3表示上面描述的 填充长度 字段及填充存在。

DATA帧 **必须(MUST)** 与一个流关联。如果收到了一个流标识符为0x0的DATA帧，接收者 **必须(MUST)** 以一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))来响应。

DATA帧受控于flow control，而且只能在流处于"open"或"half-closed (remote)"状态时发送。整个的DATA帧载荷被包含在flow control中，可能包括 填充长度 和填充字段。如果收到DATA帧的流不处于"open"或"half-closed (remote)"状态，则接收者 **必须(MUST)** 以一个类型为[STREAM_CLOSED](https://http2.github.io/http2-spec/#STREAM_CLOSED)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))来响应。

填充字节的总大小由填充长度字段决定。如果填充的长度是帧载荷的长度或更大，则接收者 **必须(MUST)** 将这作为一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))来处理应。

注意：一个帧可以通过包含一个值为零的填充长度字段来使帧长度只增加一个字节。

## 6.2 HEADERS

HEADERS帧(type=0x1)用于打开一个流([Section 5.1](https://http2.github.io/http2-spec/#StreamStates))，此外还携带一个首部块片段。HEADERS帧可以在一个"idle"，"reserved (local)"，"open"，或"half-closed (remote)"状态的流上发送。

```
 +---------------+
 |Pad Length? (8)|
 +-+-------------+-----------------------------------------------+
 |E|                 Stream Dependency? (31)                     |
 +-+-------------+-----------------------------------------------+
 |  Weight? (8)  |
 +-+-------------+-----------------------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```
图 7: HEADERS帧载荷

HEADERS帧具有如下的字段：

**填充长度：**

一个8位的字段，包含了以字节为单位的帧的填充的长度。只有在PADDED标记设置时这个字段才出现。

**E:**

一个单独的位标记，指示了流依赖是独占的(参见[Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。这个字段只有在PRIORITY标记设置时才会出现。

流依赖：一个31位的标识了这个流依赖的流的流标识符(参见[Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。这个字段只有在PRIORITY标记设置时才会出现。

**权值：**

一个无符号8位整型值，表示流的优先级权值(see [Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。这个值的范围为1到256。这个字段只有在PRIORITY标记设置时才会出现。

**首部块片段：**

一个首部块片段([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))。

**填充：**

填充字节。

HEADERS帧定义了如下的标记：

**END_STREAM (0x1)：**

当设置时，位0表示这个首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))是终端将会为标识的流发送的最后一个块。

HEADERS帧携带了 END_STREAM标记，表明了流的结束。然而，在相同的流上，一个设置了END_STREAM标记的HEADERS帧后面可以跟着[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧。逻辑上来说，着[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧是HEADERS帧的一部分。

**END_HEADERS (0x4):**

当设置时，位2表示这个帧包含了这个首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))，而且后面没有任何的[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧。

相同的流上，一个没有设置END_HEADERS标记的HEADERS帧后面 **必须(MUST)** 跟着一个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧。接收者 **必须(MUST)** 将 接收到其它类型的帧，或在其它流上接收到了帧，当做是类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误。

**PADDED (0x8):**

当设置时，位3表示将会有填充长度和它对应的填充出现。

**PRIORITY (0x20):**

当设置时，位5指明独占标记(E)，流依赖和权值字段将出现；参见[Section 5.3](https://http2.github.io/http2-spec/#StreamPriority)。

HEADERS帧的载荷包含一个首部块片段([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))。一个首部块无法装进一个HEADERS帧的话，将通过CONTINUATION帧来继续发送([Section 6.10](https://http2.github.io/http2-spec/#CONTINUATION))。

HEADERS帧 **必须(MUST)** 与一个流关联。如果接收到了一个流标识符字段为0x0的HEADERS帧，则接收者 **必须(MUST)** 响应一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

HEADERS帧如[Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock)中所述的那样改变连接的状态。

HEADERS帧可以包含填充。填充字段和标记与DATA帧([Section 6.1](https://http2.github.io/http2-spec/#DATA))中定义的一致。填充超出了首部块片段的剩余大小 **必须(MUST)** 被当做一个 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)。

HEADERS帧中包含的优先级信息在逻辑上等于另一个 [PRIORITY](https://http2.github.io/http2-spec/#PRIORITY)帧，但是包含在HEADERS中可以避免在创建新流时流优先级潜在的扰动。一个流中第一个之后的HEADERS帧中的优先级字段改变流的优先级([Section 5.3.3](https://http2.github.io/http2-spec/#reprioritize))。

## 6.3 PRIORITY

PRIORITY帧 (type=0x2) 描述了给发送者建议的一个流的优先级([Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。它可以在任何流状态下发送，包括idle和closed流。

```
 +-+-------------------------------------------------------------+
 |E|                  Stream Dependency (31)                     |
 +-+-------------+-----------------------------------------------+
 |   Weight (8)  |
 +-+-------------+
```
图 8: PRIORITY帧载荷

PRIORITY帧包含如下的字段：

**E:** 一个单独的位标记指明流依赖是独占的(参见[Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。

**流依赖：**
一个31位的流标识符，指明了这个流依赖的流(参见 [Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。

**权值：**
一个无符号8位整型值，表示流的优先级权值(参见 [Section 5.3](https://http2.github.io/http2-spec/#StreamPriority))。该值的取值范围为1到256。

PRIORITY帧不定义任何标记。

PRIORITY帧总是标识一个流。如果接收到了一个流标识符为0x0的PRIORITY帧，则接收者 **必须(MUST)** 响应一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

可以在任何状态的流上发送RIORITY帧，尽管不能在包含了一个单独的首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))的连续两帧之间发送。注意，这个帧可能在处理或帧发送已经完成时到达，这将不对标识的流产生影响。对于处在"half-closed (remote)"或"closed"状态的流，这个帧只影响标识的流和依赖于它的流的处理；它不影响那些流的帧传输。

可以为处于"idle"或"closed"状态的流发送PRIORITY帧。这允许通过改变未使用或已关闭的父流的优先级来改变一组依赖流的优先级。

PRIORITY帧的长度不是5个字节的话， **必须(MUST)** 被当做一个类型为[FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。

## 6.4 RST_STREAM

RST_STREAM帧 (type=0x3)可以用于立即终止一个流。发送RST_STREAM来请求取消一个流，或者指明发生了一个错误状态。

```
 +---------------------------------------------------------------+
 |                        Error Code (32)                        |
 +---------------------------------------------------------------+
```
Figure 9: RST_STREAM帧载荷

RST_STREAM帧包含了一个单独的无符号32位整型值的错误码([Section 7](https://http2.github.io/http2-spec/#ErrorCodes))。错误码指明了为什么要终止流。

RST_STREAM帧不定义任何标记。

RST_STREAM帧完全终止引用的流，并使它进入"closed"状态。在一个流中收到一个RST_STREAM之后，接收者 **一定不能(MUST NOT)** 再为那个流发送额外的帧，[PRIORITY](https://http2.github.io/http2-spec/#PRIORITY)是例外。然而，在发送RST_STREAM之后，发送端 **必须(MUST)** 准备接收和处理额外的，可能由对端在RST_STREAM到达之前在那个流上发送的帧。

RST_STREAM帧 **必须(MUST)** 关联一个流。如果RST_STREAM帧的流标识符是0x0，则接收者 **必须(MUST)** 将其作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))处理。

RST_STREAM帧 **一定不能(MUST NOT)** 为"idle"状态的流而发送。如果接收了一个RST_STREAM帧，而它标识了一个idle流，则接收者 **必须(MUST)** 将其作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))处理。

RST_STREAM帧的长度不是4字节的话， **必须(MUST)** 被作为一个类型是[FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))处理。

## 6.5 SETTINGS

SETTINGS帧 (type=0x4) 携带影响端点间如何通信的配置参数，比如关于对端行为的首选项和限制。SETTINGS帧也用于确认接收到了那些参数。个别地，一个SETTINGS参数也可以被称为一个"setting"。

不协商SETTINGS参数；它们描述了发送端的特性，而由接收端使用。端点之间对于相同参数可以广告不同的值。比如，一个客户端可以设置一个较大的初始flow-control窗口，然而服务器可以设置一个小的值来保留资源。

SETTINGS帧 **必须(MUST)** 在连接开始时，两端都发送，而在连接整个生命期中其它的任何时间点，其中的一个端点 **可以(MAY)** 发送。实现 **必须(MUST)** 支持这份规范定义的所有参数。

SETTINGS帧中的每个参数替换那个参数既有的值。参数以它们出现的顺序处理，SETTINGS帧的接收者不需要维护额外的状态，除了参数的当前值。因此，一个SETTINGS参数的值是接收者收到的最后的值。

SETTINGS参数由接收端作确认。为了启用这一点，SETTINGS帧定义了如下的标记：

**ACK (0x1):** 设置时，位0指示了这个帧用于确认对端的SETTINGS帧的接收和应用。当设置了这个位时，SETTINGS帧的载荷必须是空的。接收到一个设置了ACK标记的SETTINGS帧，而长度字段的值不是0，这 **必须(MUST)** 被当做一个类型为[FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。要获得更多信息，请参考[Section 6.5.3](https://http2.github.io/http2-spec/#SettingsSync) ("[Settings Synchronization](https://http2.github.io/http2-spec/#SettingsSync)")。

SETTINGS帧总是应用于一个连接，而不是一个单独的流。SETTINGS帧的流标识符 **必须(MUST)** 是零(0x0)。如果一个端点收到了一个流标识符字段不是0的SETTINGS帧，则 **必须(MUST)** 以一个类型为 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler)) 来响应。

SETTINGS帧影响连接的状态。一个格式错误或不完整的SETTINGS帧 **必须(MUST)** 被当做一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

SETTINGS帧的长度如果不是6字节的整数倍的话，必须被作为一个类型是FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

### 6.5.1 SETTINGS格式

SETTINGS帧的载荷由零个或多个参数组成，每个参数由一个无符号16位的设置标识符和一个无符号的32位值组成。

```
 +-------------------------------+
 |       Identifier (16)         |
 +-------------------------------+-------------------------------+
 |                        Value (32)                             |
 +---------------------------------------------------------------+
```
图 10: 设置项的格式

### 6.5.2 已定义的SETTINGS参数

已定义了如下的参数：

**SETTINGS_HEADER_TABLE_SIZE (0x1):**
允许发送者通知远端，用于解码首部块的首部压缩表的最大大小，以字节位单位。编码器可以可以选择任何等于或小于这个值的大小，通过使用首部块内信号特有的首部压缩格式(参考[[COMPRESSION]
](https://http2.github.io/http2-spec/#COMPRESSION))。初始值是4,096字节。

**SETTINGS_ENABLE_PUSH (0x2):**
这个设置项可被用于禁用服务端推送([Section 8.2](https://http2.github.io/http2-spec/#PushResources))。一个终端如果收到了这个设置项，且值为0，则它 **一定不能(MUST NOT)** 发送[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧。一个已经将这个参数设置为了0，且已经收到了对这个设置项的确认的终端，则在收到一个[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧时， **必须(MUST)** 将其作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

初始值是1，这表示服务端推送是允许的。任何0或1之外的值 **必须(MUST)** 被作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

**SETTINGS_MAX_CONCURRENT_STREAMS (0x3):**
指明了发送者允许的最大的并发流个数。这个限制是有方向的：它应用于发送者允许接收者创建的流的个数。初始时，这个值没有限制。建议这个值不要小于100，以便于不要不必要地限制了并发性。

SETTINGS_MAX_CONCURRENT_STREAMS被设置为0 **不应该(SHULD NOT)** 被终端特殊对待。值为0确实阻止创建新的流；然而，这也可能发生在活跃的流超出了限制的时候。服务器 **应该(SHOULD)** 只短暂地将这个值设为0；如果一个服务器不希望接受请求，关闭连接更合适。

**SETTINGS_INITIAL_WINDOW_SIZE (0x4):**
指明了发送者stream-level flow control的初始窗口大小(以字节为单位)。初始值为2^16 - 1 (65,535)字节。

这个设置项影响所有流的窗口大小(参见[Section 6.9.2](https://http2.github.io/http2-spec/#InitialWindowSize))。

值大于2^31-1的最大flow-control窗口大小 **必须(MUST)** 被作为一个类型是[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

**SETTINGS_MAX_FRAME_SIZE (0x5):**
指明了发送者期望接收的最大的帧载荷大小，以字节为单位。

初始值是2^14 (16,384)。终端广告的值 **必须(MUST)** 介于初始值和允许的最大帧大小(2^24-1 or 16,777,215 字节)之间，包含。这个范围之外的值 **必须(MUST)** 被作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

**SETTINGS_MAX_HEADER_LIST_SIZE (0x6):**
这个建议性的设置通知对端发送者准备接受的首部列表的最大大小，以字节为单位。这个值是基于首部字段未压缩的大小来计算的，包括名字和值以字节为单位的长度，再为每个首部字段加上32字节。

对于任何给定的请求，**可以(MAY)** 实施小于广告的限制的值。这个设置项的初始值没有限制。

一个终端接收了一个SETTINGS帧，其中未知的或不支持的标识符的设置项 **必须(MUST)** 被忽略。

### 6.5.3 设置同步

SETTINGS中的大多数值受益于或需要知道对端在何时接收并应用改变的参数值。为了提供这种同步时间点，一个ACK标记没有设置的SETTINGS帧的接收者 **必须(MUST)** 一收到帧就尽可能快地应用更新后的参数。

SETTINGS帧中的参数 **必须(MUST)** 以它们出现的顺序处理，值之间没有对其它帧的处理。不支持的参数 **必须(MUST)** 被忽略。一旦处理了所有值，则接收者 **必须(MUST)** 立即发送一个设置了ACK标记的SETTINGS帧。一旦接收到设置了ACK标记的SETTINGS帧，改变了参数的发送者可以依赖于已经应用的设置了。

如果SETTINGS帧的发送者没有在合理的时间内收到确认，它 **可以(MAY)** 产生一个类型为[SETTINGS_TIMEOUT](https://http2.github.io/http2-spec/#SETTINGS_TIMEOUT)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

## 6.6 PUSH_PROMISE

PUSH_PROMISE帧 (type=0x5)用于通知对端发送方想要启动一个流。PUSH_PROMISE帧包含终端计划创建的流的无符号31位标识符，及为流提供额外上下为的一系列的首部。[Section 8.2](https://http2.github.io/http2-spec/#PushResources)包含对于PUSH_PROMISE帧的使用的一个详尽的描述。

```
 +---------------+
 |Pad Length? (8)|
 +-+-------------+-----------------------------------------------+
 |R|                  Promised Stream ID (31)                    |
 +-+-----------------------------+-------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```
图 11: PUSH_PROMISE载荷格式

PUSH_PROMISE帧载荷具有如下的字段：

**填充长度：**
一个8位的字段，包含了以字节为单位的帧填充的长度。只有在PADDED标记设置时这个字段才会出现。

**R:**
一个保留位。

**约定流ID:**
一个无符号31位整数，标识了PUSH_PROMISE保留的流。约定流标识符 **必须(MUST)** 是发送者后续要发送的流的一个有效的流标识符（参见 [Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers)中的"新的流标识符"）。

**首部块片段：**
一个首部块片段([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))包含了请求的首部字段。

**填充：** 填充字节。

PUSH_PROMISE帧定义了如下的标记：

**END_HEADERS (0x4):**
设置时，位2表示这个帧包含一个完整的首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))，而且后面没有[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧。

PUSH_PROMISE帧的END_HEADERS标记没有设置的话，它的后面 **必须(MUST)** 要有相同流的CONTINUATION帧。接收者 **必须(MUST)** 将收到其它类型的帧，或在一个不同的流上收到帧，作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

**PADDED (0x8):**
设置时，位2表示 **填充长度** 字段和它所对应的填充将会出现。

PUSH_PROMISE帧 **必须(MUST)** 只在对端初始化的处于"open"或 "half-closed (remote)"状态的流上发送。PUSH_PROMISE帧的流标识符指明了它关联的流。如果流标识符字段是0x0，则接收者 **必须(MUST)** 响应一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

约定的流不需要以它们约定的顺序使用。PUSH_PROMISE只为后面的使用保留了流标识符。

如果对端将[SETTINGS_ENABLE_PUSH](https://http2.github.io/http2-spec/#SETTINGS_ENABLE_PUSH)设置为了0，则 **一定不能(MUST NOT)** 发送PUSH_PROMISE。已经设置了这个设定项且已经收到了确认的终端，收到了一个PUSH_PROMISE帧，则它 **必须(MUST)** 将这作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

PUSH_PROMISE的接收者可以通过返回一个引用了约定的流标识符的[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)给PUSH_PROMISE的发送者来拒绝约定的流。

PUSH_PROMISE帧以两种方式改变连接状态。首先，它包含的首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))潜在地改变为首部压缩维护的状态。其次，PUSH_PROMISE也为后续的使用保留一个流，这使得约定的流进入"reserved"状态。除非流处于"open"或"half-closed (remote)"状态，否则发送者 **一定不能(MUST NOT)** 在那个流上发送PUSH_PROMISE；发送者 **必须(MUST)** 确保约定的流是一个新的流标识符([Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers))的有效的选择(即，约定的流 **必须(MUST)** 处于"idle"状态)。

由于PUSH_PROMISE保留一个流，则忽略一个PUSH_PROMISE帧导致流的状态变为不确定的。接收者必须将不处于"open"或"half-closed (local)"状态的流上接收到的PUSH_PROMISE作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。然而，已经在关联的流上发送了[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)的终端， **必须(MUST)** 处理可能在RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)帧接收和处理之前创建的PUSH_PROMISE帧。

接收者接收了一个PUSH_PROMISE，但其约定了一个非法的流标识符([Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers))，则接收者必须将这作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。注意，非法的流标识符是当前不处于"idle"状态的流的标识符。

PUSH_PROMISE帧可能包含填充。填充字段和标记与DATA帧一节([Section 6.1](https://http2.github.io/http2-spec/#DATA))中定义的一致。

## 6.7 PING

PING帧 (type=0x6) 是一种测量自发送者开始的最小往返时间的机制，也用于测定一个idle连接是否仍然有效。PING帧可以自连接的任何一端发送。

```
 +---------------------------------------------------------------+
 |                                                               |
 |                      Opaque Data (64)                         |
 |                                                               |
 +---------------------------------------------------------------+
```
图 12: PING 载荷格式

除了帧首部，PING帧 **必须(MUST)** 在载荷中包含8字节的不透明数据。发送者可以包含它选择的任何值，并且可以以任何方式使用它们。

没有包含ACK标记的PING帧的接收者 **必须(MUST)** 必须在响应中发送一个设置了ACK标记的PING帧，且载荷要一致。**应该(SHOULD)** 给PING响应一个相对于其它帧更高的优先级

PING帧定义了如下的标记：

**ACK (0x1):**
当设置时，位0指明了这个PING帧是一个PING响应。终端在PING响应中 **必须(MUST)** 设置这个标记。终端 **一定不能(MUST NOT)** 响应包含了这个标记的PING帧。

PING帧不与任何独立的流关联。如果接收到一个流标识符字段不是0的PING帧，接收者必须以一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))来响应。

接收到一个PING帧，其长度字段的值不是8，则 **必须(MUST)** 被作为一个类型是[FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

## 6.8 GOAWAY

GOAWAY帧(type=0x7)用于初始化一个连接的关闭，或通知严重的错误条件。GOAWAY允许一个终端在处理之前建立的流的同时优雅地停止接收新的流。这使管理操作称为可能，如服务器维护。

在一个终端启动新流和远端发送一个GOAWAY帧之间有内在的竞态条件。要处理这种情况，GOAWAY包含这个连接中，发送端处理或可能会处理的由对端初始化的最后的流的流标识符。比如，如果服务器发送了一个GOAWAY帧，则标识的流是客户端初始化的流标识符最大的流。

GOAWAY帧一旦发送，则发送者将忽略由接收者初始化的，流标识符大于帧中包含的流标识符的流上发送的帧。GOAWAY的接收者 **一定不能(MUST NOT)** 在连接上打开额外的流，尽管可以为新的流创建一个新的连接。

如果GOAWAY的接收者已经在流标识符比GOAWAY帧中指明的更大的流上发送了数据，那些流将不会被处理。GOAWAY帧的接收者可以像它们从来没有创建一样处理，因此允许那些流稍后在一个新的连接上重试。

终端 **应该(SHOULD)** 总是在关闭连接之前发送一个GOAWAY帧，以使远处的对端可以知道一个流是否被部分处理了，还是么有。比如，如果一个HTTP客户端在服务器关闭一个连接的同时发送了一个POST，如果服务器不发送GOAWAY帧指明它已经处理的流的话，客户端无法知道服务器是否启动了对那个POST请求的处理。

一个终端可能选择在不发送GOAWAY的情况下关闭连接来使对端行为不当。

一个GOAWAY帧可以不紧贴着连接的关闭；不再使用连接的GOAWAY的接收者依然 **应该(SHOULD)** 在终止连接之前发送GOAWAY帧。

```
 +-+-------------------------------------------------------------+
 |R|                  Last-Stream-ID (31)                        |
 +-+-------------------------------------------------------------+
 |                      Error Code (32)                          |
 +---------------------------------------------------------------+
 |                  Additional Debug Data (*)                    |
 +---------------------------------------------------------------+
```
图 13: GOAWAY载荷格式

GOAWAY帧不定义任何标记。

GOAWAY帧应用于连接，而不是特定的流。一个终端 **必须(MUST)** 将一个流标识符不是0x0的GOAWAY帧作为一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

GOAWAY帧中的最后的流标识符包含了GOAWAY帧的发送者可能已经处理或可能会处理的最大的流标识符。所有流标识符小于等于标识的流的流可能已经以某种方式被处理了。如果没有流被处理的话，最后的流标识符可以被设置为0.

**注意:** 在这个上下文中，"处理"意味着来自于流的一些数据被传给了一些更高层级的软件，那些软件可能已经做了一些处理。

如果流终止了，而没有GOAWAY帧，则最后的流标识符实际上是最大的可能流标识符。

具有比标识的流的标识符更低或等于，在连接关闭前没有完全关掉的流，重试请求，事务，或任何的协议活动都是不合理的，除了幂等的行为，比如HTTP GET，PUT，或DELETE。任何使用了更大的流标识符的流的协议活动可以安全地使用一个新的连接来重试。

流标识符小于等于最后的流标识符的流上的活动可能依然可以成功完成。GOAWAY帧的发送者可以通过发送一个GOAWAY帧优雅地关闭一个连接，维护处于"open"状态地连接，直到所有进行中的流完成。

如果条件变了的话，一个终端 **可以(MAY)** 发送多个GOAWAY帧。比如，一个终端在优雅的关闭期间发送了一个带有[NO_ERROR](https://http2.github.io/http2-spec/#NO_ERROR)错误码的GOAWAY，后面遇到了一个需要立即关闭连接的条件。收到的最后的GOAWAY帧中的最后流标识符字段指明了可能已经处理的流。终端 **一定不能(MUST NOT)** 增加它们发送的最后流标识符字段的值，因为对端可能已经在另一个连接上重试了未处理的请求。

一个不能重试请求的客户端丢失所有在服务器关闭连接时处于飞行中状态的请求。特别是对于不为使用HTTP/2的客户端服务的intermediaries。一个试图优雅地关闭一个连接的服务器 **应该(SHOULD)** 发送一个初始的GOAWAY帧，其中的最后流标识符被设为2^31-1，并包含[NO_ERROR](https://http2.github.io/http2-spec/#NO_ERROR)错误码。这通知客户端连接即将关闭，禁止初始化更多的请求。在一段为飞行中的流创建准备的时间之后(至少一个来回的时间)，服务器可以发送另一个GOAWAY帧，其中携带了一个更新过的最后流标识符。这确保连接可以被干净地关闭而不会丢失氢气。

发送一个GOAWAY帧之后，发送者可以丢弃由接收者初始化的，流标识符大于最后的流标识的流。然而，任何改变连接状态的帧不能被完全忽略。比如[HEADERS](https://http2.github.io/http2-spec/#HEADERS)，[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)，和[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧必须被最低限度地处理掉以确保为首部压缩维护的状态是一致的(参见[Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))；类似地，DATA帧 **必须(MUST)** 被计入连接的flow-control窗口。处理这些帧失败的话可能导致flow control或首部压缩状态变得不同步。

GOAWAY帧也包含了一个32位的错误码([Section 7](https://http2.github.io/http2-spec/#ErrorCodes))，其中包含了关闭连接的原因。

终端 **可以(MAY)** 给GOAWAY帧的载荷附加不透明的数据。附加的调试数据只用于诊断目的，而不携带语义值。调试信息可能包含安全或隐私数据。记录或持续存储调试信息 **必须(MUST)** 具有充足的保护措施来防止未授权的访问。

## 6.9 WINDOW_UPDATE

WINDOW_UPDATE帧(type=0x8)用于实现flow control；参考[Section 5.2](https://http2.github.io/http2-spec/#FlowControl)来做整体的了解。

Flow control操作于两个层次：在每个单独的流上，和整个连接上。

flow control的两种类型都是逐段的，即，只在两个端点之间。Intermediaries不在依赖的连接之间转发WINDOW_UPDATE帧。然而，任何接收者数据传输的限制可以间接地使得flow-control信息被传播到最初的发送者。

Flow control只应用于被认为是受控于flow control的帧。就这份文档中定义的帧类型而言，这只包括[DATA](https://http2.github.io/http2-spec/#DATA)帧。那些免除flow control的帧 **必须(MUST)** 被接受并处理，除非接收者不能分配资源来处理帧。如果接收者不能接受一个帧，则它 **可以(MAY)** 响应一个类型是[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))或连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

```
 +-+-------------------------------------------------------------+
 |R|              Window Size Increment (31)                     |
 +-+-------------------------------------------------------------+
```
图 14: WINDOW_UPDATE载荷格式

WINDOW_UPDATE帧的载荷由一个保留位外加一个无符号31位的整数组成，其中后者指明了发送者在已有的flow-control窗口之外可以传输的字节数。flow-control窗口合法的增量范围是1到2^31-1 (2,147,483,647)字节。

WINDOW_UPDATE帧不定义任何标记。

WINDOW_UPDATE帧可以应用于特定的流，也可以是整个连接。对于前者，帧的流标识符指明了受影响的流；而后者，"0"值指明了整个连接受控于该帧。

接收者 **必须(MUST)** 将收到flow-control窗口增量为0的WINDOW_UPDATE帧 作为一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))；连接的flow-control窗口的错误flow-control **必须(MUST)** 被作为一个连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

WINDOW_UPDATE可以由已经发送了一个携带END_STREAM标记的帧的对端发送。这意味着接收者可能在一个"half-closed (remote)"或"closed"流上接收到WINDOW_UPDATE帧。接收者 **一定不能(MUST NOT)** 将这当做一个错误(参见 [Section 5.1](https://http2.github.io/http2-spec/#StreamStates)).。

接收到一个flow-controlled帧的接收者 **必须(MUST)** 总是考虑到它对连接flow-control窗口的影响，除非接收者把这当做一个连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。即使帧是错误的这也是必须的。发送者将帧计入flow-control，但如果接收者没有这样做的话，发送者和接收者的flow-control窗口可能变得不同。

长度不是4字节的WINDOW_UPDATE帧 **必须(MUST)** 被作为一个类型是[FRAME_SIZE_ERROR](https://http2.github.io/http2-spec/#FRAME_SIZE_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

### 6.9.1 Flow-Control窗口

HTTP/2中的Flow control使用一个由每个流上的每个发送者持有的窗口实现。flow-control窗口是一个简单的整型值，它指明了发送者允许传输的字节数；同样地，它的大小衡量了接收者的缓存能力。

两种flow-control窗口是适用的：流flow-control窗口和连接flow-control窗口。发送者 **一定不能(MUST NOT)** 发送一个长度超出了由接收者广告的任一种flow-control窗口的可用空间的flow-controlled帧。两种flow-control窗口中没有可用空间时，**可以(MAY)** 发送长度为0且END_STREAM标记被设置的帧(即，一个空的[DATA](https://http2.github.io/http2-spec/#DATA)帧)。

对于flow-control的计算，9字节的帧首部不计入内。

发送一个flow-controlled帧之后，发送者给两个窗口中的可用空间都减小发送的帧的大小。

帧的接收者由于它消耗数据并释放flow-control窗口的空间而发送一个WINDOW_UPDATE帧。独立的WINDOW_UPDATE帧为流级及连接级的flow-control窗口而发送。

接收到一个WINDOW_UPDATE帧的发送者给对应窗口更新帧中描述的数量。

发送者 **一定不能(MUST NOT)** 允许一个flow-control窗口超出2^31-1字节。如果发送者接收了一个WINDOW_UPDATE，它导致flow-control窗口超出了最大值，则它 **必须(MUST)** 酌情终止流或连接。对于流，发送者发送一个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)，其中由一个错误码[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)；对于连接，发送一个[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧，其中包含错误码[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)。

来自于发送者的Flow-controlled帧和来自于接收者的WINDOW_UPDATE帧相互之间是完全异步的。这个属性允许接收者侵略地更新由发送者保存的窗口大小来防止流的失速。

### 6.9.2 初始的Flow-Control窗口大小

当HTTP/2连接首次被建立时，新建立的流的初始flow-control窗口大小为65,535字节。连接的flow-control窗口也是65,535字节。两端可以通过在构成连接preface一部分的[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧中包含一个[SETTINGS_INITIAL_WINDOW_SIZE](https://http2.github.io/http2-spec/#SETTINGS_INITIAL_WINDOW_SIZE)值来为新流调整初始窗口大小。连接的flow-control窗口只能使用WINDOW_UPDATE帧来改变。

在接收到设置了[SETTINGS_INITIAL_WINDOW_SIZE](https://http2.github.io/http2-spec/#SETTINGS_INITIAL_WINDOW_SIZE)值的[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)值之前，一个端点在发送flow-controlled帧时只能使用默认的初始窗口大小。类似地，连接的flow-control窗口在收到WINDOW_UPDATE帧之前，被设置为默认的初始窗口大小。

除了给还不处于活跃状态的流修改flow-control，[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧可以改变具有活跃的flow-control窗口的流(即，处于"open"或"half-closed (remote)"状态的流)的初始flow-control窗口大小。当[SETTINGS_INITIAL_WINDOW_SIZE](https://http2.github.io/http2-spec/#SETTINGS_INITIAL_WINDOW_SIZE)的值改变时，接收者 **必须(MUST)** 根据新值和老值之间的差异调整它维护的所有的flow-control窗口的大小

对于[SETTINGS_INITIAL_WINDOW_SIZE](https://http2.github.io/http2-spec/#SETTINGS_INITIAL_WINDOW_SIZE)的改变，可能导致一个flow-control窗口中的可用空间变为负值。一个发送者 **必须(MUST)** 追踪负的flow-control窗口，且 **一定不能(MUST NOT)** 发送新的flow-controlled帧，直到它收到了使flow-control窗口变为正值的WINDOW_UPDATE帧。

比如，如果连接一建立客户端就立即发送了60 KB，而服务器将初始的窗口大小设置为16 KB，则客户端在收到[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧时将重新计算可用的flow-control窗口，其值为-44 KB。客户端保持负的flow-control窗口直到WINDOW_UPDATE帧将窗口恢复为正值，然后客户端可以恢复发送。

一个[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧不能改变连接的flow-control窗口。

终端必须将导致任何flow-control窗口超出最大大小的[SETTINGS_INITIAL_WINDOW_SIZE](https://http2.github.io/http2-spec/#SETTINGS_INITIAL_WINDOW_SIZE)的修改作为一个类型[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

### 6.9.3 减小流窗口大小

想要使用一个比当前大小更小的flow-control窗口的接收者可以发送一个新的[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧。然而，接收者 **必须(MUST)** 准备接收超出窗口大小的数据，因为发送者可能在处理[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧之前发送数据而超出了更低的限制。

发送了减小初始flow-control窗口大小的SETTINGS

发送了一个减小初始flow-control窗口大小的SETTINGS帧之后，接收者 **可以(MAY)** 继续处理超出flow-control限制的流。允许流继续不允许接收者立即减小它为flow-control窗口保留的空间。这些流中的进程可能失速，由于需要[WINDOW_UPDATE](https://http2.github.io/http2-spec/#WINDOW_UPDATE)帧来允许发送者恢复发送。接收者 **可以(MAY)** 为受到影响的流发送错误码为[FLOW_CONTROL_ERROR](https://http2.github.io/http2-spec/#FLOW_CONTROL_ERROR)的[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)。

## 6.10 CONTINUATION

CONTINUATION帧 (type=0x9) 被用于继续发送首部块片段([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))的序列。只要相同流上的前导帧是没有设置END_HEADERS标记的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)，[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)，或CONTINUATION帧，就可以发送任意数量的CONTINUATION帧。

```
 +---------------------------------------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
```

图 15: CONTINUATION帧载荷

CONTINUATION帧载荷包含一个首部块片段([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))。

CONTINUATION帧定义了如下的标记：

**END_HEADERS (0x4):**

当设置时，位2指明这个帧结束了一个首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))。如果END_HEADERS位没有设置，则这个帧后面 **必须(MUST)** 跟着另一个CONTINUATION帧。如果一个接收者接收到了其它类型的帧，或在一个不同的流上接收到了帧，则必须将这作为类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

CONTINUATION帧如[Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock)中定义的那样改变连接状态。

CONTINUATION帧 **必须(MUST)** 与一个流关联。如果接收到了一个流标识符字段为0x0的CONTINUATION帧，则接收者必须以一个类型为PROTOCOL_ERROR的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))来响应。

CONTINUATION帧的前面必须是END_HEADERS标记没有设置的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)，[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)或CONTINUATION帧。接收者如果发现违背了这个规则， **必须(MUST)** 响应一个类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

# [7. 错误码](https://http2.github.io/http2-spec/#ErrorCodes)

错误码是用在[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)和[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧中的32位字段，来携带流或连接错误的原因的。

错误码共享一个共同的码空间。一些错误码只应用于流或整个连接，而在其它上下文中没有定义语义。


当前定义了如下错误码：

NO_ERROR (0x0):相关的情况不是发生了一个错误的结果。比如，[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)可能包含这个码来指示优雅的关闭一个连接。

PROTOCOL_ERROR (0x1): 终端探测到一个不明确的协议错误。这个错误在没有更明确的错误码可用时使用。

INTERNAL_ERROR (0x2): 终端遇到了一个不预期的内部错误。


FLOW_CONTROL_ERROR (0x3): 终端探测到它的对端节点违反了流控协议。

SETTINGS_TIMEOUT (0x4): 终端发送了一个[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)帧，但没有及时地收到响应。参见[Section 6.5.3](https://http2.github.io/http2-spec/#SettingsSync) ("Settings Synchronization")。

STREAM_CLOSED (0x5): 终端在流被半关闭 (half-closed) 之后接收到了一个帧。

FRAME_SIZE_ERROR (0x6): 终端接收到了一个大小无效的帧。

REFUSED_STREAM (0x7): 终端在执行任何应用处理之前拒绝了流 (参考 [Section 8.1.4](https://http2.github.io/http2-spec/#Reliability) 来了解更多细节)。

CANCEL (0x8): 被终端用来表明不再需要特定的流了。

COMPRESSION_ERROR (0x9): 终端无法为连接维护首部压缩上下文了。

CONNECT_ERROR (0xa): 对一个CONNECT请求 ([Section 8.3](https://http2.github.io/http2-spec/#CONNECT))做出响应而建立的连接被重置或意外的关闭了。

ENHANCE_YOUR_CALM (0xb): 终端探测到它的对端的行为可能产生过载。

INADEQUATE_SECURITY (0xc):
底层传输部分的属性不满足最低的安全需求(参见 [Section 9.2](https://http2.github.io/http2-spec/#TLSUsage))。

HTTP_1_1_REQUIRED (0xd): 终端需要用HTTP/1.1来替换HTTP/2。

未知的或不支持的错误码**必须不(MUST NOT)** 触发任何特别的行为。一个实现 **可以(MAY)** 将这些当作[INTERNAL_ERROR](https://http2.github.io/http2-spec/#INTERNAL_ERROR)一样。

# [8. HTTP消息交换](https://http2.github.io/http2-spec/#HTTPLayer)

HTTP/2被期待着尽可能与当前使用的HTTP兼容。这意味着，从应用程序的视角来看，大部分的协议的功能不能变。为了实现这一点，而保留了所有的请求和响应的语义，尽管携带这些语义的语法已经变了。

因而，HTTP/1.1的规范和要求，Semantics and Content [[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，Conditional Requests [[RFC7232]
](https://http2.github.io/http2-spec/#RFC7232)，Range Requests [[RFC7233]
](https://http2.github.io/http2-spec/#RFC7233)，Caching [[RFC7234]
](https://http2.github.io/http2-spec/#RFC7234)，和Authentication [[RFC7235]
](https://http2.github.io/http2-spec/#RFC7235)依然适用于HTTP/2。选中的HTTP/1.1 Message Syntax and Routing [[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)的部分，比如HTTP和HTTPS URI schemes，也适用于HTTP/2，但是对于这个协议，那些语义的表达则在下面的小节定义。

## [8.1 HTTP 请求/响应 交换](https://http2.github.io/http2-spec/#HttpSequence)

一个客户端在一个新流上发送一个HTTP请求，使用一个之前未使用的流标识符([Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers))。一个服务器在与请求相同的流上发送HTTP响应。

一个HTTP消息 (请求或响应)的组成为：

1. 仅适用于响应，0个或多个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧 (每个后面都跟着0个或多个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧)包含信息性的 (1xx) HTTP响应的消息头(参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#header.fields)和[[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 6.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#status.1xx))。

2. 一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧 (每个后面都跟着0个或多个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧)包含消息首部 (参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#header.fields))

3. 0个或多个[DATA](https://http2.github.io/http2-spec/#DATA)帧包含载荷体 (参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.3](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#message.body))，以及

4. 可选的，一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧 (每个后面都跟着0个或多个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧)包含尾部，如果存在的话(参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 4.1.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#chunked.trailer.part))

序列的最后一帧携带END_STREAM标记，注意一个携带了END_STREAM标记的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧后面可以跟多个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧来携带首部块的其余部分。

[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧和它后面跟着的任何[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧之间 **必须不(MUST NOT)** 能出现任何其它帧(来自于任何流)。

HTTP/2使用DATA帧来携带携带消息载荷。在HTTP/2中 **必须不(MUST NOT)** 能使用在[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)的[Section 4.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#chunked.encoding)定义的分块传输编码。

首部尾字段 (trailing header fields)由一个首部块携带，首部块也会终止流。这样的一个首部块一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧起始，后跟0个或多个[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧，其中[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧携带了END_STREAM标记。第一个首部块之后没有终止流的首部块不是HTTP请求或响应的一部分。

[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧 (及其相关的[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧)只能出现在一个流的开始或结尾处。一个终端在接收到一个最终的 (final) (非信息性的)状态码之后，接收到了一个没有设置END_STREAM的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧，则它 **必须(MUST)** 将对应的请求或响应当作是已损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

一个HTTP请求/响应交换完全消耗一个流。一个请求以一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧开始，而将流放进“打开”状态。请求以一个携带了END_STREAM的帧结束，而使得流对于客户端变为"half-closed (local)"，对于服务器变为"half-closed (remote)"。一个响应以一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧开始，以一个携带了END_STREAM的帧结束，而将流放进"closed"状态。

一个HTTP响应在服务器发送——或客户端收到——一个设置了END_STREAM标记的帧(包含任何完成一个首部块所需的 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧)之后完成。如果响应不依赖于请求的任何还未接收到的部分的话，则它可以在客户端发送完整的请求之前就发送一个完整的响应。如果是这种情况，服务器 **可以(MAY)** 在发送了完整的响应之后，通过发送一个携带了错误码[NO_ERROR](https://http2.github.io/http2-spec/#NO_ERROR)的[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)，无错误地请求客户端停止传输请求 (比如一个设置了END_STREAM标记的帧)。客户端在收到了这样的一个[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)时， **必须不(MUST NOT)** 能丢弃响应，尽管因为其它原因客户端总是有丢弃响应的自由。

### [8.1.1 从HTTP/2升级](https://http2.github.io/http2-spec/#informational-responses)

HTTP/2移除了对101 (Switching Protocols) 信息性状态码([[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 6.2.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#status.101))的支持。

101 (Switching Protocols)的语义不再适用于多路复用的协议。HTTP/2使用的替代协议能够使用相同的语义来协商使用(参见[Section 3](https://http2.github.io/http2-spec/#starting))。

### [8.1.2 HTTP 首部字段](https://http2.github.io/http2-spec/#HttpHeaders)

HTTP首部字段以一系列键-值对的形式携带信息。要获得已注册的HTTP首部的列表，参见维护于<[https://www.iana.org/assignments/message-headers](https://www.iana.org/assignments/message-headers)>的“消息首部字段”注册。

就如同HTTP/1.x中的那样，首部字段名是不区分大小写的ASCII字符串。然而，首部字段名 **必须(MUST)** 在被编码进HTTP/2之前被转换为小写形式。一个请求或响应包含了大写的首部字段名 **必须(MUST)** 被当作是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

#### [8.1.2.1 伪首部字段](https://http2.github.io/http2-spec/#PseudoHeaderFields)

尽管 HTTP/1.x 使用消息起始行(参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#start.line))来携带目标URI，请求的method，和响应的状态码，而HTTP/2则使用以':'字符 (ASCII 0x3a)起始的特殊的伪首部字段来达到这一目的。

伪首部字段不是HTTP首部字段。终端 **一定不能(MUST NOT)** 产生这份文档中定义的之外的伪首部字段。

伪首部字段只在定义它们的上下文有效。为请求定义的伪首部字段 **一定不能(MUST NOT)** 出现在响应中；为响应定义的伪首部字段 **一定不能(MUST NOT)** 出现在请求中。伪首部字段 **一定不能(MUST NOT)** 出现在尾部。终端 **一定要(MUST)** 将包含了未定义或无效的伪首部字段的请求或响应当做是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

所有的伪首部字段 **必须(MUST)** 出现在首部块中普通的首部字段之前。任何的在首部块中包含了位于普通首部字段之后的伪首部字段的请求或响应 **必须(MUST)** 被当做是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

#### [8.1.2.2 连接特有首部字段](https://http2.github.io/http2-spec/#rfc.section.8.1.2.2)

HTTP/2不使用Connection首部字段来指明 连接特有(connection-specific) 首部字段；在这个协议中，连接特有(connection-specific) 元数据有其它方式传送。一个终端 **一定不能(MUST NOT)** 产生包含 连接特有(connection-specific) 首部字段的HTTP/2消息；任何包含了 连接特有(connection-specific) 首部字段的消息  **必须(MUST)** 被当做是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))

这条规则仅有的例外是TE首部字段，它 **可以(MAY)** 出现HTTP/2请求中；当出现时，它 **一定不能(MUST NOT)** 包含除"trailers"之外的任何值。

这意味着一个中继传输一个HTTP/1.x消息给HTTP/2时，将需要与移除Connection首部字段本身一起，移除任何Connection首部字段提名的字段。这样的中继也 **应该(SHOULD)** 移除其它的连接特有首部字段，比如Keep-Alive，Proxy-Connection，Transfer-Encoding，和Upgrade，即使它们没有被Connection首部字段提名。

**注意：** HTTP/2自觉地不支持升级到其它协议。[Section 3](https://http2.github.io/http2-spec/#starting)中描述的握手方法对于协商使用的协议相信足够了。

#### [8.1.2.3 请求的伪首部字段](https://http2.github.io/http2-spec/#HttpRequest)

下面的伪首部字段是为HTTP/2请求定义的：
* `:method` 伪首部字段包含HTTP method([[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 4](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#methods))


* `:scheme` 伪首部字段包含了目标URI ([[RFC3986]
](https://http2.github.io/http2-spec/#RFC3986)，[Section 3.1](https://tools.ietf.org/html/rfc3986#section-3.1)) 的scheme部分。

`:scheme` 不限于http和https schemed URIs。一个代理或网关可以为非HTTP schemes转换请求，以便于使用HTTP来与非HTTP服务交互。

* `:authority` 伪首部字段包含了目标URI ([[RFC3986]
](https://http2.github.io/http2-spec/#RFC3986)，[Section 3.1](https://tools.ietf.org/html/rfc3986#section-3.1)) 的认证部分。`authority`  **一定不能(MUST NOT)** 给http或https schemed URIs包含废弃的`userinfo`子组件。要确保HTTP/1.1请求行可以被精确地重现，当从 一个有着 以origin或asterisk的形式 (参见 [[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 5.3](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#request-target)) 的请求目标的HTTP/1.1请求转换时，这个伪首部字段 **必须(MUST)** 被省略。直接产生HTTP/2请求的客户端 **应该(SHOULD)** 使用`:authority` 伪首部字段，而不是Host首部字段。如果请求中没有Host首部字段的话，则将HTTP/2请求转换为HTTP/1.1的中继 **必须(MUST)** 通过复制`:authority`伪首部字段的值来创建一个。

* `:path` 伪首部字段包含了目标URI (完整的路径和一个可选的'?'字符及其后面接着的query) 的path和query部分(参见[[RFC3986]
](https://http2.github.io/http2-spec/#RFC3986)的 Sections [3.3](https://tools.ietf.org/html/rfc3986#section-3.3)和[3.4](https://tools.ietf.org/html/rfc3986#section-3.4))。星号形式的请求包含了值为'*'的`:path`伪首部字段。对于http和https URI，这个伪首部字段 **一定不能(MUST NOT)** 是空的。不包含一个path组件的http和https URIs **必须(MUST)** 包含一个'/'值。这条规则的例外是不包含path组件的http或https URI的OPTIONS请求；这些**必须(MUST)** 包含一个值为'*'的`:path`伪首部字段(参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 5.3.4](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#asterisk-form))。

所有的HTTP/2请求 **必须(MUST)** 为`:method`，`:scheme`和`:path`伪首部字段包含且只包含一个有效值，除非它是一个CONNECT请求 ([Section 8.3](https://http2.github.io/http2-spec/#CONNECT))。一个省略了必须的伪首部字段的HTTP请求是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

HTTP/2没有定义一种方式来携带版本标识符，如同HTTP/1.1的请求行中包含的那样。

#### [8.1.2.4 响应的伪首部字段](https://http2.github.io/http2-spec/#HttpResponse)

对于HTTP/2响应，`:status`伪首部字段被定义来携带HTTP状态码字段(参见 [[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 6](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#status.codes))。这个伪首部字段 **必须(MUST)** 被包含在所有的响应中；否则，响应是损坏的 ([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

HTTP/2没有定义一种携带HTTP/1.1的状态行中包含的版本或原因描述的方式。

#### [8.1.2.5 压缩Cookie首部字段](https://http2.github.io/http2-spec/#CompressCookie)

[Cookie首部字段](https://http2.github.io/http2-spec/#COOKIE) [COOKIE]使用分号(";")来分割cookie-pairs (或 "crumbs")。这个首部字段不遵守HTTP中 列表结构规则 (参见[[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.2.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#field.order))，那使得cookie-pairs无法被分成不同的名值对。在只有单独的cookie-pairs更新时，这将大大地降低压缩效率

要达到更好的压缩效率，Cookie首部字段 **可以(MAY)** 被分割为分开的首部字段，每个包含一个或多个cookie-pairs。如果解压后有多个Cookie首部字段，则这些首部字段 **必须(MUST)** 在被传递给非HTTP/2上下文，如一个HTTP/1.1连接，或一个普通的HTTP服务应用 之前，使用值为0x3B，0x20的2-字节分隔符 (ASCII字符串"; ") 连接为一个单独字节串。

因此，下面的两个Cookie首部字段列表在语义上是等价的。

```
  cookie: a=b; c=d; e=f

  cookie: a=b
  cookie: c=d
  cookie: e=f
```

#### [8.1.2.6 损坏的请求和响应](https://http2.github.io/http2-spec/#malformed)

一个损坏的请求或响应，是那种其它方面依然是一个有效的HTTP/2序列，但由于出现了无关的帧，禁用的首部字段，缺失了必要的首部字段，或包含了大写的首部字段名而变得无效的请求或响应。

一个包含载荷体的请求或响应可以包含一个content-length首部字段。如果content-length首部字段的值不等于构成了载荷体的[DATA](https://http2.github.io/http2-spec/#DATA)帧载荷的长度和的请求或响应也是损坏的。被定义为不含有载荷的响应，在 [[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 3.3.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#header.content-length) 中描述，可以具有一个非零的content-length首部字段，尽管在 [DATA](https://http2.github.io/http2-spec/#DATA) 帧不包含内容。

处理HTTP请求或响应的中继 (例如，任何不扮演隧道角色的中继) **一定不能(MUST NOT)** 转发损坏的请求或响应。探测到的损坏的请求或响应 **必须(MUST)** 被作为一个类型是 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR) 的流错误 ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))对待。

对于损坏的请求，服务器 **可以(MAY)** 在关闭或重置流之前发送一个HTTP响应。客户端 **一定不能(MUST NOT)** 接受一个损坏的响应。注意，这些要求被用来保护某些常见类型的HTTP攻击；由于许可可能将实现暴露给这些脆弱性，它们有意地比较严格。

### [8.1.3 例子](https://http2.github.io/http2-spec/#rfc.section.8.1.3)

这个小节展示了HTTP/1.1的请求和响应，以及等价的HTTP/2请求和相应的描述。

一个HTTP GET请求包含请求首部字段而没有载荷体，因此由一个单独的 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧，后面跟着0个或多个包含请求首部序列化块的[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧传输。下面的 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧同时设置了END_HEADERS和END_STREAM标记；没有发送[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧。

```
  GET /resource HTTP/1.1           HEADERS
  Host: example.org          ==>     + END_STREAM
  Accept: image/jpeg                 + END_HEADERS
                                       :method = GET
                                       :scheme = https
                                       :path = /resource
                                       host = example.org
                                       accept = image/jpeg
```
类似地，只包含响应首部字段的响应也由一个包含响应首部字段的序列化块的 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧传输 (再一次，后面跟着0个或多个 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧)。

```
  HTTP/1.1 304 Not Modified        HEADERS
  ETag: "xyzzy"              ==>     + END_STREAM
  Expires: Thu, 23 Jan ...           + END_HEADERS
                                       :status = 304
                                       etag = "xyzzy"
                                       expires = Thu, 23 Jan ...
```

一个包含了请求首部字段和载荷数据的HTTP POST请求以一个 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧，后跟包含有请求首部字段的0个或多个 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧，再后面跟着一个或多个[DATA](https://http2.github.io/http2-spec/#DATA)帧来传输，其中最后的[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) (或[HEADERS](https://http2.github.io/http2-spec/#HEADERS))帧设置了END_HEADERS标记，最后的[DATA](https://http2.github.io/http2-spec/#DATA)帧设置了END_STREAM标记：

```
  POST /resource HTTP/1.1          HEADERS
  Host: example.org          ==>     - END_STREAM
  Content-Type: image/jpeg           - END_HEADERS
  Content-Length: 123                  :method = POST
                                       :path = /resource
  {binary data}                        :scheme = https

                                   CONTINUATION
                                     + END_HEADERS
                                       content-type = image/jpeg
                                       host = example.org
                                       content-length = 123

                                   DATA
                                     + END_STREAM
                                   {binary data}
```
注意任何给定首部字段的数据可以在首部块片段之间传播。这个例子中为帧分配的首部字段只是说明性的。

包含了首部字段和载荷数据的响应以一个[HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧，后跟0个或多个 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧，再后面跟着一个或多个[DATA](https://http2.github.io/http2-spec/#DATA)帧来传输，其中序列中最后的[DATA](https://http2.github.io/http2-spec/#DATA)帧设置了END_STREAM标记：

```
  HTTP/1.1 200 OK                  HEADERS
  Content-Type: image/jpeg   ==>     - END_STREAM
  Content-Length: 123                + END_HEADERS
                                       :status = 200
  {binary data}                        content-type = image/jpeg
                                       content-length = 123

                                   DATA
                                     + END_STREAM
                                   {binary data}
```

一个使用非101的1xx状态码的信息性的响应以一个 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧，后跟0个或多个 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧。

在请求或响应的的首部块及所有的[DATA](https://http2.github.io/http2-spec/#DATA)帧都被发送了之后，以一个首部块发送尾部首部字段。[HEADERS](https://http2.github.io/http2-spec/#HEADERS)帧启动尾部首部块，设置了END_STREAM标记。

下面的例子包含一个100 (Continue)状态码，作为对一个在Expect首部字段，和尾部首部字段中包含了 "100-continue" token的请求的响应：

```
  HTTP/1.1 100 Continue            HEADERS
  Extension-Field: bar       ==>     - END_STREAM
                                     + END_HEADERS
                                       :status = 100
                                       extension-field = bar

  HTTP/1.1 200 OK                  HEADERS
  Content-Type: image/jpeg   ==>     - END_STREAM
  Transfer-Encoding: chunked         + END_HEADERS
  Trailer: Foo                         :status = 200
                                       content-length = 123
  123                                  content-type = image/jpeg
  {binary data}                        trailer = Foo
  0
  Foo: bar                         DATA
                                     - END_STREAM
                                   {binary data}

                                   HEADERS
                                     + END_STREAM
                                     + END_HEADERS
                                       foo = bar
```

### [8.1.4 HTTP/2中的请求可靠性机制](https://http2.github.io/http2-spec/#Reliability)

在HTTP/1.1中，一个HTTP客户端不能在发生错误时重试一个非幂等的请求，因为没有方式来确定错误的性质。有可能一些服务器在发生错误之前处理了请求，如果重试请求的话，那可能导致不希望看到的结果。

HTTP/2提供两种机制来为客户端提供保证，保证请求还没有被处理：

* [GOAWAY](https://http2.github.io/http2-spec/#GOAWAY) 帧指示了可能已经被处理的最高的流号。因而，流号更高的流上的请求的重试保证是安全的。

* [REFUSED_STREAM](https://http2.github.io/http2-spec/#REFUSED_STREAM) 错误码可以被包含进一个 [RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM) 帧来表明，流在任何处理发生之前之前被关闭了。在重置的流上发送的任何请求可以被安全地重试。

还没有被处理的请求并没有失败；客户端 **可以(MAY)** 自动地重试它们，即使它们的methods是非幂等的。

除非服务器能作出保证，否则它 **一定不能(MUST NOT)** 表明一个流还没有被处理。对于任何的流，如果流上的帧被传给了应用层，则 **一定不能(MUST NOT)** 再将[REFUSED_STREAM](https://http2.github.io/http2-spec/#REFUSED_STREAM)用于那个流了，[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY) 帧 **必须(MUST)** 包含一个大于等于给定流标识符的流标识符。

除了这些机制，[PING](https://http2.github.io/http2-spec/#PING) 帧给客户端提供了一种方式来简单地测试一个连接。处于闲置状态的连接可能由于一些middleboxes (比如，网络地址转换器，负载均衡器) 安静地废弃了连接绑定而损坏。[PING](https://http2.github.io/http2-spec/#PING) 帧使得客户端可以在不发送请求的情况下安全地测试一个连接是否有效。

## [8.2 服务器推送](https://http2.github.io/http2-spec/#PushResources)

HTTP/2使服务器可以抢先地发送 (或推送) 与客户端之前初始化的请求相关联的响应 (伴随着对应的"promised"请求) 给客户端。当服务器知道客户端将需要那些响应以完全处理最初的请求的响应。

客户端可以请求禁用服务器推送，尽管会为每个独立的hop协商。[SETTINGS_ENABLE_PUSH](https://http2.github.io/http2-spec/#SETTINGS_ENABLE_PUSH)设置项可以被设置为0以表明服务器推送被禁用。

Promised请求 **必须(MUST)** 是可缓存的 (参见[[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 4.2.3](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#cacheable.methods))， **必须(MUST)** 是安全的(参见 [[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 4.2.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#safe.methods))， **一定不能(MUST NOT)** 包含请求体。客户端接收到一个promised请求，但请求不能被缓存，不知道是否安全，或者出现了请求体，必须通过一个类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))来重置promised流。注意如果客户端无法确认一个新定义的method是安全的，则这可能导致promised流被重置。

推送的可缓存的响应 (参见[[RFC7234]
](https://http2.github.io/http2-spec/#RFC7234)，[Section 3](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7234.html#response.cacheability)) 可以被客户端保存，如果它实现了一个HTTP缓存的话。推送的响应被认为在原始的服务器上已经被成功地验证过 (比如，如果出现了"no-cache"缓存响应指示([[RFC7234]
](https://http2.github.io/http2-spec/#RFC7234)，[Section 5.2.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7234.html#cache-response-directive))) 了，尽管由promised流ID标识的流依然是打开的。

不能缓存的推送的响应 **一定不能(MUST NOT)** 被任何HTTP缓存保存。它们 **可以(MAY)** 单独提供给应用。

服务器 **必须(MUST)** 在`:authority`伪首部字段中包含服务器认证授权的值(参见[Section 10.1](https://http2.github.io/http2-spec/#authority))。客户端 **必须(MUST)** 将服务器没有认证的 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 当作类型为[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的流错误([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。

一个中继可从服务器接收推送而选择不把它们转发给客户端。换句话说，如何使用推送的信息由中继自行处理。同样的，服务器在没有采取任何行动的时候，中继也可能选择给客户端创建额外的推送。

客户端不能推送。因此，服务器 **必须(MUST)** 将收到一个 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧作为一个类型是 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler)) 处理。客户端 **必须(MUST)** 拒绝任何试图改变 [SETTINGS_ENABLE_PUSH](https://http2.github.io/http2-spec/#SETTINGS_ENABLE_PUSH) 设置项为0之外的值的行为，并将其作为一个类型是 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

### [8.2.1 推送请求](https://http2.github.io/http2-spec/#PushRequests)

服务器推送在语义上与服务器响应一个请求是等价的；然而，在这种情况下，那个请求也会由服务器发送，以 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧的形式。

[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧包含一个首部块，其中包含了完整的服务器认为属于请求的请求首部字段集合。不可能推送一个响应给包含请求体的请求。

推送的响应总是与一个来自于客户端的显式的请求关联。服务器发送的[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧是在那个显式的请求的流上发送的。[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧还包含一个promised流标识符，从服务器可用的流表示符中选择的 (参见 [Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers))。

[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)中的首部字段，及任何随后的 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧， **必须(MUST)** 是一个有效的且完整的请求首部字段集 ([Section 8.1.2.3](https://http2.github.io/http2-spec/#HttpRequest))。服务器 **必须(MUST)** 在可缓存的且安全的 `:method` 伪首部字段中包含一个method。如果一个客户端收到了一个[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)，但其中不包含一个完整且有效的首部字段集，或 `:method`伪首部字段描述了一个method但它不是安全的，它 **必须(MUST)** 以一个类型为 [PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR) 的流错误 ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler)) 作为响应。

服务器 **应该(SHOULD)** 在发送任何引用了promised响应的帧之前发送[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) ([Section 6.6](https://http2.github.io/http2-spec/#PUSH_PROMISE))帧。这避免了一个竞态，即客户端在接收到任何 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧之前发出了请求。

比如，如果服务器收到了一个对一个document的请求，其中包含了多个图片文件的内嵌的链接，服务器选择把那些图片推送给客户端，它在发送包含了图片的链接的[DATA](https://http2.github.io/http2-spec/#DATA)帧之前发送[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧，以确保客户端能够在发现内嵌的链接之前知道一份资源将被推送。类似地，如果服务器推送了被首部块引用的响应 (比如，在Link首部字段中)，则在发送首部块之前发送一个 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)，以确保客户端没有清求那些资源。

[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧 **一定不能(MUST NOT)** 由客户端发送。

[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧可以由服务器作为对客户端初始化的任何流的响应而发送，但是流对于服务器 **必须(MUST)** 处于"open"或"half-closed (remote)" 状态。[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)穿插在包含响应的帧中，尽管它们不能穿插在包含一个单独的首部块的[HEADERS](https://http2.github.io/http2-spec/#HEADERS)和[CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION)帧之间。

发送一个[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧创建一个新流，并将流置于服务器端的 “reserved (local)” 状态，及客户端的 “reserved (remote)” 状态。

### [8.2.2 推送响应](https://http2.github.io/http2-spec/#PushResponses)

发送了[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)帧之后，服务器可以开始在一个服务器初始化的使用了promised流标识符的流上传送推送的响应作为一个响应([Section 8.1.2.4](https://http2.github.io/http2-spec/#HttpResponse))了。服务器使用这个流来传输HTTP响应，使用如同在 [Section 8.1](https://http2.github.io/http2-spec/#HttpSequence) 中定义的相同的帧序列。在初始的 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧发送之后，对于客户端而言这个流变为了"half-closed" ([Section 5.1](https://http2.github.io/http2-spec/#StreamStates))。

一旦客户端接收到一个 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧并选择接受推送的响应，客户端 **不应该(SHOULD NOT)** 为promised响应发出任何请求直到promised流已经关闭。

如果客户端决定，由于任何原因，它不愿意接收服务器推送的响应，或如果服务器花费了太长时间才开始发送promised响应，客户端可以发送一个 [RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM) 帧，使用 [CANCEL](https://http2.github.io/http2-spec/#CANCEL) 或 [REFUSED_STREAM](https://http2.github.io/http2-spec/#REFUSED_STREAM) 错误码，并引用推送的流的标识符。

一个客户端使用 [SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS) 设置项来限制服务器可以并发地推送的响应的个数。将[SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS)设置为0值通过阻止服务器创建必要的流来禁用服务器推送。这不禁止服务器发送[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧；客户端需要重置任何它不想要的promised流。

客户端接收了一个推送的响应，**必须(MUST)** 验证服务器是否认证 (参见 [Section 10.1](https://http2.github.io/http2-spec/#authority))，或提供推送响应的代理为相应的请求做了配置。比如，一个只为 example.com DNS-ID提供了一个证书的服务器，或者Common Name不被允许为 https://www.example.org/doc 推送一个响应。

[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE)流的响应以一个 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧开始，它立即将服务器端的响应流置于"half-closed (remote)"状态，而将客户端的置于"half-closed (local)"状态，而以一个设置了END_STREAM的帧结束，它将流置于"closed"状态。

**注意：** 客户端从不为服务器推送发送设置了END_STREAM标记的帧。

## [8.3 CONNECT Method](https://http2.github.io/http2-spec/#CONNECT)

在HTTP/1.x，pseudo-method CONNECT ([[RFC7231]](https://http2.github.io/http2-spec/#RFC7231)，[Section 4.3.6](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#CONNECT)) 被用于将一个HTTP连接转换为一个到远程主机的隧道。CONNECT主要与HTTP代理一起使用，为了与https资源进行交互，而与原始服务器建立一个TLS会话。

在HTTP/2中，为了类似的目的，CONNECT method被用于建立一个到远程的主机的基于单独的HTTP/2流的隧道。HTTP首部字段映射如同[Section 8.1.2.3](https://http2.github.io/http2-spec/#HttpRequest) ("[请求的伪首部字段](https://http2.github.io/http2-spec/#HttpRequest)")中定义的那样工作，但略有不同。特别地：`:method` 伪首部字段被设置为CONNECT。而`:scheme` 和 `:path` 伪首部字段 **必须(MUST)** 被省略。`:authority` 伪首部字段包含了要连接的主机和端口 (等价于CONNECT请求(参见 [[RFC7230]](https://http2.github.io/http2-spec/#RFC7230)，[Section 5.3](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#request-target))的request-target的authority-form)

不符合这些限制的CONNECT请求是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。

支持CONNECT的代理建立一个与由`:authority`伪首部字段确定的服务器的[TCP connection](https://http2.github.io/http2-spec/#TCP) [TCP]。一旦这个连接被成功地建立了，代理给客户端发送一个 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧，其中包含了2xx系列状态码，如同 [[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 4.3.6](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#CONNECT) 中定义的那样。

在每个端点发送了初始的 [HEADERS](https://http2.github.io/http2-spec/#HEADERS) 帧之后，数据对应的所有后续 [DATA](https://http2.github.io/http2-spec/#DATA) 帧在那个TCP连接上发送。客户端发送的所有 [DATA](https://http2.github.io/http2-spec/#DATA) 帧的载荷由代理传输给TCP服务器；从TCP服务器接收的数据由代理汇集为 [DATA](https://http2.github.io/http2-spec/#DATA) 帧。 [DATA](https://http2.github.io/http2-spec/#DATA) 和流管理帧([RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)，[WINDOW_UPDATE](https://http2.github.io/http2-spec/#WINDOW_UPDATE)，和 [PRIORITY](https://http2.github.io/http2-spec/#PRIORITY))之外的其它类型帧 **一定不能(MUST NOT)** 在一个已连接的流上发送，而在出现时 **必须(MUST)** 被当作一个流错误 ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。

TCP连接可以由连接的任何一端关闭。一个 [DATA](https://http2.github.io/http2-spec/#DATA) 帧的END_STREAM标记被认为与TCP FIN位一样。客户端在收到一个设置了END_STREAM标记的帧之后，被预期要发送一个设置了END_STREAM标记的 [DATA](https://http2.github.io/http2-spec/#DATA) 帧。接收到一个设置了END_STREAM标记的 [DATA](https://http2.github.io/http2-spec/#DATA) 帧的代理，发送在最后的片段上设置FIN位整合数据。接收到一个设置了FIN位的TCP片段的代理发送一个设置了END_STREAM标记的 [DATA](https://http2.github.io/http2-spec/#DATA) 帧。注意最后的TCP片段或 [DATA](https://http2.github.io/http2-spec/#DATA) 帧不能是空的。

TCP连接错误用 [RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM) 通知。代理将TCP连接中的任何错误，包括接收到一个设置了RST位的TCP片段，作为类型是 [CONNECT_ERROR](https://http2.github.io/http2-spec/#CONNECT_ERROR) 的流错误 ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler))。相应的，如果探测到流或HTTP/2连接的错误，则代理 **必须(MUST)** 发送一个设置了RST位的TCP片段。

# [9. 额外的HTTP要求/注意事项](https://http2.github.io/http2-spec/#HttpExtra)

这一节概括了提升互操作性，降低暴露给已知的安全脆弱性，或降低潜在的实现变异的HTTP协议属性。

## [9.1 连接管理](https://http2.github.io/http2-spec/#rfc.section.9.1)

HTTP/2连接是持久的。为了获得最好的性能，希望客户端不要关闭连接，直到它决定没有与服务器做进一步通信的必要了 (比如，当用户从一个特定的web页离开了)，或服务器关闭了连接。

客户端 **不应该(SHOULD NOT)** 打开到一个给定主机和端口对的多于一个的HTTP/2连接，其中主机来源于一个URI，一个选定的 [备选服务](https://http2.github.io/http2-spec/#ALT-SVC) [ALT-SVC]，或一个配置的代理。

客户端可以创建额外的连接作为替代，替换那些接近于超出可用流标识符空间 ([Section 5.1.1](https://http2.github.io/http2-spec/#StreamIdentifiers)) 的连接，或者为一个TLS连接刷新重要材料，或者替换那些遇到错误的连接([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

客户端 **可以(MAY)** 使用不同的 [Server Name Indication](https://http2.github.io/http2-spec/#TLS-EXT) [TLS-EXT] 值打开多个到相同的IP地址和端口号的连接，以提供不同的TLS客户端证书，但 **应该(SHOULD)** 避免使用相同的配置创建多个连接。

鼓励服务器尽可能长地维护打开的连接，但允许在需要的时候终止空闲的连接。当连接的其中一端选择关闭传输层 TCP连接时，终止连接的一端 **应该(SHOULD)** 首先发送一个 [GOAWAY (Section 6.8)](https://http2.github.io/http2-spec/#GOAWAY) 帧，以使两端可以可靠地确定之前发送的帧是否得到了处理，或者终止任何必须的剩余的任务。

### [9.1.1 连接重用](https://http2.github.io/http2-spec/#reuse)

与原始的服务器之间建立的连接，无论是直连的或者是使用CONNECT方法([Section 8.3](https://http2.github.io/http2-spec/#CONNECT))创建的一个隧道， **可以(MAY)** 被多个具有不同的URI认证组件的请求重用。只要原始的服务器授权([Section 10.1](https://http2.github.io/http2-spec/#authority))，连接可以一直被重用。对于没有TLS的TCP连接，这依赖于主机解析到了相同的IP地址。

对于https资源，连接重用还依赖于具有一个URI中的主机的有效的证书。服务器呈现的证书， **必须(MUST)** 满足客户端在为URI中的主机建立一个新的TLS连接时执行的检查。

一个原始的服务器可能提供一个证书，该证书具有多个subjectAltName属性，或者具有通配符的名字，而其中的一个对于URI的认证是有效的。比如，一个证书的asubjectAltName是*.example.com，则它可以允许为以https://a.example.com/和https://b.example.com/开头的URI的请求使用相同的连接。


在某些部署中，为多个源重用一个连接可能导致请求被定向到错误的源。比如，TLS终止可能

使用 TLS [Server Name Indication (SNI)](https://http2.github.io/http2-spec/#TLS-EXT) [TLS-EXT]扩展来选择一个源服务器的中间沙箱可以执行TLS终止。这意味着客户端可能给服务器发送保密信息，而服务器可能不是希望的请求目标，尽管服务器依然是授权的。

服务器不希望客户端重用连接时，它可以通过在给请求发送的响应中包含421 (Misdirected Request) 状态码来表明那不是授权的请求。

配置了要使用HTTP/2代理的客户端直接通过一个单独的连接来向代理请求。即，通过一个代理发送的所有请求重用到代理的连接。

### [9.1.2 421 (Misdirected Request)状态码](https://http2.github.io/http2-spec/#MisdirectedRequest)

421 (Misdirected Request)状态码表明，请求被定向给了一个不能产生响应的服务器。一个没有配置来为 请求的URI中包含的scheme和authority的结合 产生响应的服务器可以发送这个状态码。

客户端从服务器接收到了一个421 (Misdirected Request) 响应 **可以(MAY)** 在一个不同的连接上重试请求——无论请求的方法是否是幂等的。这对于被复用的连接([Section 9.1.1](https://http2.github.io/http2-spec/#reuse)) 或选择了一个备选服务 [[ALT-SVC]](https://http2.github.io/http2-spec/#ALT-SVC) 的情况是可能的。

这个状态码 **一定不能(MUST NOT)** 由代理产生。

默认情况下421响应是可缓存的，比如，除非方法定义有所指明，或显式的缓存控制 (参见 [[RFC7234]](https://http2.github.io/http2-spec/#RFC7234) 的 [Section 4.2.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7234.html#heuristic.freshness))。

## [9.2 使用TLS特性](https://http2.github.io/http2-spec/#TLSUsage)

HTTP/2的实现 **必须(MUST)** 为HTTP/2 over TLS使用 [TLS version 1.2](https://http2.github.io/http2-spec/#TLS12) [TLS12]或更高的版本。 **应该(SHOULD)** 遵循[[TLSBCP]](https://http2.github.io/http2-spec/#TLSBCP) 中的通用TLS使用指南，同时伴随着特定于HTTP/2的一些额外的限制。

TLS实现  **必须(MUST)** 支持TLS的 [Server Name Indication (SNI)](https://http2.github.io/http2-spec/#TLS-EXT) [TLS-EXT]扩展。HTTP/2客户端 **必须(MUST)** 在协商TLS时指明目标域名。

协商TLS 1.3或更高版本的HTTP/2部署需要只支持并使用SNI扩展；TLS 1.2 的部署受下面的小节的要求支配。鼓励实现提供一致的默认的选项，但是部署为最终的兼容性负责。。

### [9.2.1 TLS 1.2特性](https://http2.github.io/http2-spec/#rfc.section.9.2.1)

这个部分描述了可被用于HTTP/2的TLS 1.2功能集的限制。由于部署的限制，当这些限制不满足时，TLS协商失败是不可能的。一个终端  **可以(MAY)** 以一个类型是 [INADEQUATE_SECURITY](https://http2.github.io/http2-spec/#INADEQUATE_SECURITY) 的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler)) 立即终止一个不满足这些TLS要求的HTTP/2连接

一个基于TLS 1.2的HTTP/2部署 **必须(MUST)** 禁用压缩。TLS压缩可能导致其它情况下不会泄漏的信息暴露 [[RFC3749]](https://http2.github.io/http2-spec/#RFC3749)。由于HTTP/2提供了压缩功能，该压缩功能对上下文更了解，且由于性能，安全性或其它原因，它可能更合适，因而不需要通用的压缩。

基于TLS 1.2的HTTP/2部署 **必须(MUST)** 禁用重协商。终端 **必须(MUST)** 将TLS重协商作为类型是[PROTOCOL_ERROR](https://http2.github.io/http2-spec/#PROTOCOL_ERROR)的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。注意，禁用重协商可能导致长生存期的连接变得不可用，因为底层的加密套件可加密的消息个数的限制，

终端 **可以(MAY)** 使用重协商来为客户端在握手消息中提供的机密信息提供机密性保护，但是任何重协商 **必须(MUST)** 发生在发送连接前言之前。如果服务器在建立连接之后立即看到了一个重协商请求，它 **应该(SHOULD)** 请求一个客户端证书。


这有效地阻止了在一个对特定的受保护资源的请求的响应中使用重协商。未来的规范可能提供一种方式来支持这种使用场景。或者服务器可以使用一个类型为 [HTTP_1_1_REQUIRED](https://http2.github.io/http2-spec/#HTTP_1_1_REQUIRED) 的连接错误 ([Section 5.4](https://http2.github.io/http2-spec/#ErrorHandler))来请求客户端使用一个支持重协商的协议。

实现 **必须(MUST)** 为使用 finite field Diffie-Hellman (DHE) [[TLS12]](https://http2.github.io/http2-spec/#TLS12)的加密套件支持至少2048位的ephemeral key 交换大小，而为使用ephemeral elliptic curve Diffie-Hellman (ECDHE) [[RFC4492]](https://http2.github.io/http2-spec/#RFC4492)的加密套件支持至少224位的ephemeral key 交换大小。客户端 **必须(MUST)** 接受至多4096位的DHE大小。终端 **可以(MAY)** 将小于限制的key大小的协商作为一个类型是 [INADEQUATE_SECURITY](https://http2.github.io/http2-spec/#INADEQUATE_SECURITY) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler)) 。

### [9.2.2 TLS 1.2 密文族](https://http2.github.io/http2-spec/#rfc.section.9.2.2)

基于TLS 1.2的HTTP/2部署 **不应该(SHOULD NOT)** 使用在下面的加密套件黑名单 ([Appendix A](https://http2.github.io/http2-spec/#BadCipherSuites)) 中所列的加密套件。

如果协商了黑名单中的某个加密套件，终端 **可以(MAY)** 选择产生一个类型为 [INADEQUATE_SECURITY](https://http2.github.io/http2-spec/#INADEQUATE_SECURITY) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。选择使用黑名单中所列的加密套件的部署有触发一个连接错误的风险，除非已知潜在的对端集合接受那个加密套件。

实现 **一定不能(MUST NOT)** 为不在黑名单中的加密套件的协商响应这个错误。因此，当客户端提供了一个不在黑名单上的加密套件，它们不得不准备以HTTP/2使用那个加密套件。

黑名单包含了TLS 1.2强制性的加密套件，这意味着TLS 1.2部署可能与允许的加密套件没有交集。要避免这个问题导致TLS握手失败，使用TLS 1.2的HTTP/2部署 **必须(MUST)** 支持具有P-256椭圆曲线[[FIPS186]](https://http2.github.io/http2-spec/#FIPS186)的TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 [[TLS-ECDHE]](https://http2.github.io/http2-spec/#TLS-ECDHE)。

注意，客户端可能告知它支持黑名单上的加密套件，以允许连接到不支持HTTP/2的服务器。这允许服务器以HTTP/2黑名单上的加密套件选择HTTP/1.1。然而，如果应用层协议和加密套件是独立选择的话，这可能导致HTTP/2被以一个黑名单上的加密套件协商。

# [10. 安全注意事项](https://http2.github.io/http2-spec/#security)

## [10.1 服务器认证](https://http2.github.io/http2-spec/#authority)

HTTP/2依赖于HTTP/1.1定义的认证方式来确定提供给定响应的服务器(参见 [[RFC7230]](https://http2.github.io/http2-spec/#RFC7230)，[Section 9.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#establishing.authority)) 是否是可信的。这依赖于为"http" URI scheme做的本地域名解析，及"https" scheme的身份认证服务器 [[RFC2818]](https://http2.github.io/http2-spec/#RFC2818)，[Section 3](https://tools.ietf.org/html/rfc2818#section-3))。

## [10.2 跨协议攻击](https://http2.github.io/http2-spec/#rfc.section.10.2)

在跨协议攻击中，攻击者导致客户端以某个协议对它的服务器初始化一个事务，而服务器则理解另一个不同的协议。攻击者也许能够使得事务以在第二个协议中有效的形式出现。结合web上下文的容量，这可以被用于与私有网络中保护不好的服务器交互。

通过ALPN标识符为HTTP/2完成一个TLS握手，可以被认为对于保护跨协议攻击是足够的。ALPN提供了服务器想要采用HTTP/2的明确的指示，这阻止了其它基于TLS协议的攻击。

TLS中的加密使得攻击者难于控制在明文协议中可能被用于跨协议攻击的数据。

HTTP/2的明文版本对跨协议攻击有着最小的保护。连接前言 ([Section 3.5](https://http2.github.io/http2-spec/#ConnectionHeader)) 包含了一个字符串，被设计来迷惑HTTP/1.1服务器，但没有提供针对其它协议的特别保护。想要忽略 除了客户端连接前言外还包含Upgrade首部字段的HTTP/1.1请求 的一部分的服务器可能遭到跨协议攻击。

## [10.3 中间封装攻击](https://http2.github.io/http2-spec/#rfc.section.10.3)

HTTP/2首部字段编码允许表达在 HTTP/1.1使用的Internet Message Syntax 中无效的字段名字。包含了无效的首部字段名的请求或响应 **必须(MUST)** 被当作是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。因此中继不能将包含了无效字段名的HTTP/2请求或响应转换为HTTP/1.1消息。

类似地，HTTP/2允许首部字段值不是有效的。尽管能够编码的大部分值不会改变首部字段值的解析，如果carriage return (CR, ASCII 0xd), line feed (LF, ASCII 0xa), and the zero character (NUL, ASCII 0x0)被逐字翻译的话，则可能被攻击者利用。包含了不允许在首部字段值中出现的字符的请求或响应 **必须(MUST)**  被当作是损坏的([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed))。有效的字符在 [[RFC7230]](https://http2.github.io/http2-spec/#RFC7230) 的 [Section 3.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#header.fields) 字段内容 ABNF规则中定义。

## [10.4 推送的响应的可缓存性](https://http2.github.io/http2-spec/#rfc.section.10.4)

推送的响应没有一个来自客户端的显式的请求；请求由服务器在 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧中提供。

基于原始的服务器在Cache-Control首部字段中给出的引导，缓存推送的响应是可能的。然而，如果一个单独的服务器运行了多个服务，这可能会导致一些问题。比如，服务器可能为多个用户中的每一个提供了它的URI空间的一小部分。

在多个服务共享相同服务器空间的情况下，那个服务器 **必须(MUST)** 确保服务不能推送它们没有被授权的资源。未能执行这一点将使一个服务能够提供可缓存的表示，覆盖授权的服务提供的实际的表示。

未认证的原始服务器(参见 [Section 10.1](https://http2.github.io/http2-spec/#authority))推送的响应 **一定不能(MUST NOT)** 被使用或缓存。

## [10.5 拒绝服务注意事项](https://http2.github.io/http2-spec/#dos)

相对于HTTP/1.1连接，HTTP/2连接可以请求对资源操作更大的保证。首部压缩及流控的使用依赖于为存储更大量的状态的承诺的资源。这些功能的设置项确保为这些功能而保留的内存被严格限制。

以相同的方式，[PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧数量没有限制。接受服务器推送的客户端 **应该(SHOULD)** 限制允许进入"reserved (remote)"状态的流的数量。过多的服务器推送流可以被作为类型 [ENHANCE_YOUR_CALM](https://http2.github.io/http2-spec/#ENHANCE_YOUR_CALM) 的流错误 ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler)) 。

处理能力无法像保护状态容量那样有效。

[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS) 帧可能被滥用，从而导致一个端点耗费额外的处理时间。这可能通过不相干地改变 SETTINGS 参数，设置多个未定义的参数，或在相同的帧中多次改变相同的设置项来完成。[WINDOW_UPDATE](https://http2.github.io/http2-spec/#WINDOW_UPDATE) 或 [PRIORITY](https://http2.github.io/http2-spec/#PRIORITY) 帧可能被滥用而导致不必要的资源浪费。

大量小的或空的帧可能被滥用，而导致对端耗费时间来处理帧首部。注意，然后，一些使用是完全正当的，比如在一个流的最后发送空的[DATA](https://http2.github.io/http2-spec/#DATA) 或 [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) 帧。

首部压缩也提供了一些浪费资源的机会；参见 [[COMPRESSION]](https://http2.github.io/http2-spec/#COMPRESSION) 的 [Section 7](https://tools.ietf.org/html/rfc7541#section-7) 来了解更多关于潜在的滥用的细节。

[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS) 参数中的限制无法被立即地降低，这可能使一个端点的行为超出了新的限制。特别地，建立连接之后，由服务器立即设置的限制，客户端还不知道，则它可能在没有明显的违反协议的情况下超出限制。

所有这些功能——比如，[SETTINGS](https://http2.github.io/http2-spec/#SETTINGS)的改变，小帧，首部压缩——都有着正当的使用场景。这些功能只有在不必要的使用或过度的使用时才变成了负担。

不监视这种行为的终端，将有可能遭到拒绝服务攻击。实现 **应该(SHOULD)** 跟踪对这些功能的使用，并为它们的使用设定限制。终端 **可以(MAY)** 将可疑的行为作为类型 [ENHANCE_YOUR_CALM](https://http2.github.io/http2-spec/#ENHANCE_YOUR_CALM) 的连接错误 ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))。

### [10.5.1 首部块大小的限制](https://http2.github.io/http2-spec/#MaxHeaderBlock)

一个大首部块([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))可能导致实现提交大量的状态。那些对于路由非常重要的首部字段可能出现在首部块靠近结尾的地方，这阻止了首部字段的流到达它们最终的目的地。这种顺序及其它原因，比如确保缓存的正确性，意味着一个终端可能需要缓存整个首部块。由于对首部块大小没有硬性限制，某些终端可能被强制为首部字段提交大量的变量可用内存。

终端可以使用 [SETTINGS_MAX_HEADER_LIST_SIZE](https://http2.github.io/http2-spec/#SETTINGS_MAX_HEADER_LIST_SIZE) 来为它的对端提供关于可能应用的首部块大小限制的建议。这个设置项只是建议性的，因而终端 **可以(MAY)** 选择发送超出这个限制的首部块，而冒着请求或响应被当作已损坏的风险。这个设置项专门用于连接，因而任何请求或相应可能遇到更小的单跳，未知的限制。中继可以通过传递不同的对端值，来试着避免这个问题，但它们没有义务这样做。

接收到了大于想要处理的大小的首部块的服务器可以发送一个HTTP 431 (Request Header Fields Too Large) 状态码 [[RFC6585]](https://http2.github.io/http2-spec/#RFC6585)。客户端可以丢弃它不能处理的响应。除非连接被关闭了，否则首部块 **必须(MUST)** 得到处理，以保持一致的连接状态。

### [10.5.2 CONNECT问题](https://http2.github.io/http2-spec/#connectDos)

CONNECT方法可以被用于创建代理上不成比例的负载，因为相对于TCP连接的创建和维护，创建流不那么昂贵。除携带了CONNECT请求的流的关闭之外，代理也许还要为TCP连接维护一些资源，因为外发的TCP连接保持为TIME_WAIT状态。然而，代理不能只依赖 [SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS) 来决定CONNECT请求消耗的资源。

## [10.6 压缩的使用](https://http2.github.io/http2-spec/#rfc.section.10.6)

当以与攻击者控制的数据相同的上下文压缩时，压缩可以使得攻击恢复机密数据。HTTP/2启用了首部字段压缩([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))；下面的担忧同样适用于HTTP压缩 内容编码 ([[RFC7231]](https://http2.github.io/http2-spec/#RFC7231)，[Section 3.1.2.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#content.codings)) 的使用。

有一些关于压缩的攻击利用了web的特性 (例如，[[BREACH]
](https://http2.github.io/http2-spec/#BREACH))。攻击者引起了多个包含不同纯文本的请求，然后观察每个最终的加密文本的长度，当关于密文的一个猜测正确时这揭示了一个更短的长度。

基于一个安全通道通信的实现 **一定不能(MUST NOT)** 压缩同时包含机密的和攻击者控制的数据的内容，除非为每个数据源使用不同的压缩字典。如果数据源无法被可靠的确定，则 **一定不能(MUST NOT)** 使用压缩。通用的流压缩，比如TLS提供的， **一定不能(MUST NOT)** 被用在HTTP/2中 (参见 [Section 9.2](https://http2.github.io/http2-spec/#TLSUsage)) 。

关于首部压缩更多的注意事项在 [[COMPRESSION]
](https://http2.github.io/http2-spec/#COMPRESSION) 中描述。

## [10.7 填充的使用](https://http2.github.io/http2-spec/#padding)

HTTP/2中的填充并不打算替代通用的填充，比如可能由 [TLS](https://http2.github.io/http2-spec/#TLS12) [TLS12]提供的填充。
冗余填充甚至可能会适得其反。正确的应用程序可能依赖被填充的数据的特定知识。

要减少依赖压缩的攻击，相对于填充而言，禁用或限制压缩可能是一个更适当的对策。

填充可被用于模糊帧内容的准确大小，并被提供用以减少HTTP内特定的攻击，比如，压缩的内容包含了攻击者控制的纯文本和机密数据的攻击(e.g.，[[BREACH]](https://http2.github.io/http2-spec/#BREACH))。

相对于似乎很明显能看到的，对填充的使用可能导致更少的保护。最好，填充只是通过增加攻击者不得不观察的帧数量，而使得攻击者更难推断长度信息。不正确地实现填充方案很容易被探测出来。典型的，可预测分布的随机填充提供了非常少的保护；类似地，固定大小的填充载荷在跨固定大小边界时暴露了载荷的大小，如果攻击者可以控制文本的话这是可能的。

中继 **应该(SHOULD)** 为 [DATA](https://http2.github.io/http2-spec/#DATA) 帧保留填充，但 **可以(MAY)** 丢弃[HEADERS](https://http2.github.io/http2-spec/#HEADERS) 和 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧的填充。中继改变帧的填充大小的一个有效的原因是，加强填充提供的保护。

## [10.8 隐私注意事项](https://http2.github.io/http2-spec/#rfc.section.10.8)

HTTP/2的一些特性为观察者提供了一个机会，随着时间的推移关联一个客户端或服务器的行为。这包括设置项的值，管理流控窗口的方式，为流分配优先级的方式，对激励的响应的时序，以及由设置项控制的功能的处理。

这些在行为上建立了可观察的差别，它们可能被用作识别一个特定的客户端的基础，如同在 [[HTML5]](https://http2.github.io/http2-spec/#HTML5) 的 [Section 1.8](http://www.w3.org/TR/2014/REC-html5-20141028/introduction.html#fingerprint) 中定义的那样。

HTTP/2对于使用单个TCP连接的偏好，允许获得一个站点上用户活动的相关性。为不同的源复用连接，允许跨那些源追踪。

由于PING和SETTINGS帧请求立即响应，它们可能被终端用于测量延时。这在某些场景下可能对隐私有影响。

# [11. IANA 注意事项](https://http2.github.io/http2-spec/#iana)

一个用于识别HTTP/2的字符串进入了由 [[TLS-ALPN]
](https://http2.github.io/http2-spec/#TLS-ALPN) 建立的 "应用层协议协商 (ALPN) 协议IDs" 注册表

本文档为帧类型，设置项，和错误码建立了一个注册表。这些新的注册表项出现在新的"Hypertext Transfer Protocol version 2 (HTTP/2) Parameters"一节。

为了在HTTP中使用，本文档注册了HTTP2-Settings首部字段；它还注册了421 (Misdirected Request) 状态码。

为了在HTTP中使用，本文档还注册了PRI方法，以避免与连接前言 (connection preface)  ([Section 3.5](https://http2.github.io/http2-spec/#ConnectionHeader))冲突


## [11.1 HTTP/2 识别字符串的注册 ](https://http2.github.io/http2-spec/#iana-alpn)

本文档为HTTP/2的识别 (参见 [Section 3.3](https://http2.github.io/http2-spec/#discover-https)) 而向由 [[TLS-ALPN]
](https://http2.github.io/http2-spec/#TLS-ALPN) 建立的 "应用层协议协商 (ALPN) 协议IDs" 注册表中建立了两个注册项。

当在TLS之上使用HTTP/2时，用"h2"来标识HTTP/2：

协议：HTTP/2 over TLS

标识串：0x68 0x32 ("h2")

规范说明：本文档

"h2c"字符串标识裸TCP之上的HTTP/2。

协议：HTTP/2 over TCP

标识串：0x68 0x32 0x63 ("h2c")

规范说明：本文档

## [11.2 帧类型注册表](https://http2.github.io/http2-spec/#iana-frames)

本文档为HTTP/2帧类型码建立了一个注册表。"HTTP/2帧类型"注册表管理一个8位的空间。"HTTP/2帧类型"注册表在 ["IETF Review"或"IESG Approval" 策略](https://http2.github.io/http2-spec/#RFC5226) [RFC5226] 之下操作，其值在0x00和0xef之间，而0xf0 和 0xff之间的值被保留以备实验之用。

注册表中的新项需要下面的信息：

帧类型：帧类型的一个名称或者标签。

代码：与分配给帧类型的8位代码。

规范说明：一个规范说明的引用，其中包含了帧布局的描述，它的语义，帧类型使用的标记，包含基于标记的值而有条件的出现的帧的任何部分。

本文档注册了下表所列的项。


|Frame Type    |Code |Section                 |
|--------------|-----|------------------------|
|DATA          |0x0  |[Section 6.1](https://http2.github.io/http2-spec/#DATA)          |
|HEADERS       |0x1  |[Section 6.2](https://http2.github.io/http2-spec/#HEADERS)       |
|PRIORITY      |0x2  |[Section 6.3](https://http2.github.io/http2-spec/#PRIORITY)      |
|RST_STREAM    |0x3  |[Section 6.4](https://http2.github.io/http2-spec/#RST_STREAM)     |
|SETTINGS      |0x4  |[Section 6.5](https://http2.github.io/http2-spec/#SETTINGS)      |
|PUSH_PROMISE  |0x5  |[Section 6.6](https://http2.github.io/http2-spec/#PUSH_PROMISE)  |
|PING          |0x6  |[Section 6.7](https://http2.github.io/http2-spec/#PING)          |
|GOAWAY        |0x7  |[Section 6.8](https://http2.github.io/http2-spec/#GOAWAY)        |
|WINDOW_UPDATE |0x8  |[Section 6.9](https://http2.github.io/http2-spec/#WINDOW_UPDATE) |
|CONTINUATION  |0x9  |[Section 6.10](https://http2.github.io/http2-spec/#CONTINUATION) |

## [11.3 设置项注册表](https://http2.github.io/http2-spec/#iana-settings)

本文档为 HTTP/2设置项 建立了一个注册表。“HTTP/2设置项” 注册表管理一个16位的空间。“HTTP/2设置项” 注册表在 ["Expert Review" 策略](https://http2.github.io/http2-spec/#RFC5226) [RFC5226]之下操作，其取值范围在0x0000到0xefff之间，而0xf000到0xffff之间的值被保留以备实验之用。

建议为新的注册项提供如下的信息：


名称：设置项的一个符号形式的名称。指定一个设置项名称是可选的。

代码：分配给设置项的16位代码。

初始值：设置项的初始值。

规范说明：到一个规范说明的可选的引用，其中描述了设置项的使用。

下表是本文档注册的项。

|Name                   |Code  |Initial Value |Specification    |
|-----------------------|------|--------------|-----------------|
|HEADER_TABLE_SIZE      |0x1   |4096          | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|ENABLE_PUSH            |0x2   |1             | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_CONCURRENT_STREAMS |0x3   |(infinite)    | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|INITIAL_WINDOW_SIZE    |0x4   |65535         | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_FRAME_SIZE         |0x5   |16384         | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_HEADER_LIST_SIZE   |0x6   |(infinite)    | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|

## [11.4 错误码注册表](https://http2.github.io/http2-spec/#iana-errors)

本文档为HTTP/2错误码建立了一个注册表。“HTTP/2错误码”注册表管理一个32位的空间。“HTTP/2错误码”注册表在["Expert Review"策略](https://http2.github.io/http2-spec/#RFC5226) [RFC5226]之下操作。

注册错误码需要包含一个错误码的描述。建议有一个专家审查者来检验新的注册可能与已有错误码的重复。鼓励使用已有的注册，但不强制。

建议新的注册提供如下的信息：

名称：错误码的名称。指定一个错误码名称是可选的。

代码：32位的错误码值。

描述：一个错误码语义的清晰描述，如果没有提供详细的规范说明的话要更长一点。

规范说明：到一个规范说明的可选的引用，其中定义了错误码。

下表是本文档注册的项。

|Name                |Code |Description                              |Specification  |
|--------------------|-----|-----------------------------------------|-------------|
|NO_ERROR            |0x0  |Graceful shutdown                        |[Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|PROTOCOL_ERROR      |0x1  |Protocol error detected                  | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|INTERNAL_ERROR      |0x2  |Implementation fault                     | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|FLOW_CONTROL_ERROR  |0x3  |Flow-control limits exceeded             | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|SETTINGS_TIMEOUT    |0x4  |Settings not acknowledged                | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|STREAM_CLOSED       |0x5  |Frame received for closed stream         | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|FRAME_SIZE_ERROR    |0x6  |Frame size incorrect                     | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|REFUSED_STREAM      |0x7  |Stream not processed                     | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|CANCEL              |0x8  |Stream cancelled                         | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|COMPRESSION_ERROR   |0x9  |Compression state not updated            | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|CONNECT_ERROR       |0xa  |TCP connection error for CONNECT method  | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|ENHANCE_YOUR_CALM   |0xb  |Processing capacity exceeded             | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|INADEQUATE_SECURITY |0xc  |Negotiated TLS parameters not acceptable | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|
|HTTP_1_1_REQUIRED   |0xd  |Use HTTP/1.1 for the request             | [Section 7](https://http2.github.io/http2-spec/#ErrorCodes)|

## [11.5 HTTP2-Settings 首部字段注册](https://http2.github.io/http2-spec/#rfc.section.11.5)

这一节向 "Permanent Message Header Field Names" 注册表 [[BCP90]](https://http2.github.io/http2-spec/#BCP90) 注册了HTTP2-Settings首部字段。

首部字段名称：HTTP2-Settings

适用的协议：http

状态：标准

作者/变动控制者：IETF

规范文档：本文档的 [Section 3.2.1](https://http2.github.io/http2-spec/#Http2SettingsHeader)

相关信息：这个首部字段只被 HTTP/2 客户端用于基于升级 (Upgrade-based) 的协商。

## [11.6 PRI 方法注册](https://http2.github.io/http2-spec/#rfc.section.11.6)

这一节向"HTTP方法注册表" ([[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 8.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#method.registry))中注册了PRI方法。

方法名：PRI

安全的：是

幂等的：是

规范说明：本文档的 [Section 3.5](https://http2.github.io/http2-spec/#ConnectionHeader)

相关信息：这个方法从不会由实际的客户端使用。这个方法主要在HTTP/1.1服务器或中继在解析HTTP/2连接前言（connection preface）时使用。

## [11.7 421 (Misdirected Request) HTTP状态码](https://http2.github.io/http2-spec/#iana-MisdirectedRequest)

本文档向 "HTTP状态码" 注册表 ([[RFC7231]
](https://http2.github.io/http2-spec/#RFC7231)，[Section 8.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#status.code.registry))中注册了 421 (Misdirected Request) HTTP状态码。

状态码：421

简短描述：误重定向请求

规范说明：本文档的 [Section 9.1.2](https://http2.github.io/http2-spec/#MisdirectedRequest)

## [11.8  h2c 升级 Token](https://http2.github.io/http2-spec/#iana-h2c)

本文档向"HTTP升级Tokens"注册表中 ([[RFC7230]
](https://http2.github.io/http2-spec/#RFC7230)，[Section 8.6](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#upgrade.token.registry))中注册了 "h2c"升级token。

值：h2c

描述：超文本传输协议版本 2 (HTTP/2)

期待的版本Tokens：None

参考：本文档的 [Section 3.2](https://http2.github.io/http2-spec/#discover-http)。

# [12. 参考文献](https://http2.github.io/http2-spec/#rfc.section.12)

## [12.1 引用的标准](https://http2.github.io/http2-spec/#rfc.section.12.1)

**[COMPRESSION]** 
Peon, R. and H. Ruellan, “[HPACK: Header Compression for HTTP/2](https://tools.ietf.org/html/rfc7541)”, RFC 7541, [DOI 10.17487/RFC7541](http://dx.doi.org/10.17487/RFC7541), May 2015, <[http://www.rfc-editor.org/info/rfc7541](http://www.rfc-editor.org/info/rfc7541)>.

**[COOKIE]**
Barth, A., “[HTTP State Management Mechanism](https://tools.ietf.org/html/rfc6265)”, RFC 6265,[DOI 10.17487/RFC6265](http://dx.doi.org/10.17487/RFC6265), April 2011, <[http://www.rfc-editor.org/info/rfc6265](http://www.rfc-editor.org/info/rfc6265)>.

**[FIPS186]**
NIST, “[Digital Signature Standard (DSS)](http://dx.doi.org/10.6028/NIST.FIPS.186-4)”, FIPS PUB 186-4, July 2013, <[http://dx.doi.org/10.6028/NIST.FIPS.186-4](http://dx.doi.org/10.6028/NIST.FIPS.186-4)>.

**[RFC2119]**
Bradner, S., “[Key words for use in RFCs to Indicate Requirement Levels](https://tools.ietf.org/html/rfc2119)”, BCP 14, RFC 2119, [DOI 10.17487/RFC2119](http://dx.doi.org/10.17487/RFC2119), March 1997, <[http://www.rfc-editor.org/info/rfc2119](http://www.rfc-editor.org/info/rfc2119)>.

**[RFC2818]**
Rescorla, E., “[HTTP Over TLS](https://tools.ietf.org/html/rfc2818)”, RFC 2818, [DOI 10.17487/RFC2818](http://dx.doi.org/10.17487/RFC2818), May 2000, <[http://www.rfc-editor.org/info/rfc2818](http://www.rfc-editor.org/info/rfc2818)>.

**[RFC3986]**
Berners-Lee, T., Fielding, R., and L. Masinter, “[Uniform Resource Identifier (URI): Generic Syntax](https://tools.ietf.org/html/rfc3986)”, STD 66, RFC 3986, [DOI 10.17487/RFC3986](http://dx.doi.org/10.17487/RFC3986), January 2005, <[http://www.rfc-editor.org/info/rfc3986](http://www.rfc-editor.org/info/rfc3986)>.

**[RFC4648]**
Josefsson, S., “[The Base16, Base32, and Base64 Data Encodings](https://tools.ietf.org/html/rfc4648)”, RFC 4648,[DOI 10.17487/RFC4648](http://dx.doi.org/10.17487/RFC4648), October 2006, <[http://www.rfc-editor.org/info/rfc4648](http://www.rfc-editor.org/info/rfc4648)>.

**[RFC5226]**
Narten, T. and H. Alvestrand, “[Guidelines for Writing an IANA Considerations Section in RFCs](https://tools.ietf.org/html/rfc5226)”, BCP 26, RFC 5226, [DOI 10.17487/RFC5226](http://dx.doi.org/10.17487/RFC5226), May 2008, <[http://www.rfc-editor.org/info/rfc5226](http://www.rfc-editor.org/info/rfc5226)>.

**[RFC5234]**
Crocker, D., Ed. and P. Overell, “[Augmented BNF for Syntax Specifications: ABNF](https://tools.ietf.org/html/rfc5234)”, STD 68, RFC 5234, [DOI 10.17487/RFC5234](http://dx.doi.org/10.17487/RFC5234), January 2008, <[http://www.rfc-editor.org/info/rfc5234](http://www.rfc-editor.org/info/rfc5234)>.

**[RFC7230]**
Fielding, R., Ed. and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing](https://tools.ietf.org/html/rfc7230)”, RFC 7230, [DOI 10.17487/RFC7230](http://dx.doi.org/10.17487/RFC7230), June 2014, <[http://www.rfc-editor.org/info/rfc7230](http://www.rfc-editor.org/info/rfc7230)>.

**[RFC7231]**
Fielding, R., Ed. and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](https://tools.ietf.org/html/rfc7231)”, RFC 7231, [DOI 10.17487/RFC7231](http://dx.doi.org/10.17487/RFC7231), June 2014, <[http://www.rfc-editor.org/info/rfc7231](http://www.rfc-editor.org/info/rfc7231)>.

**[RFC7232]**
Fielding, R., Ed. and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests](https://tools.ietf.org/html/rfc7232)”, RFC 7232, [DOI 10.17487/RFC7232](http://dx.doi.org/10.17487/RFC7232), June 2014, <[http://www.rfc-editor.org/info/rfc7232](http://www.rfc-editor.org/info/rfc7232)>.

**[RFC7233]**
Fielding, R., Ed., Lafon, Y., Ed., and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Range Requests](https://tools.ietf.org/html/rfc7233)”, RFC 7233, [DOI 10.17487/RFC7233](http://dx.doi.org/10.17487/RFC7233), June 2014, <[http://www.rfc-editor.org/info/rfc7233](http://www.rfc-editor.org/info/rfc7233)>.

**[RFC7234]**
Fielding, R., Ed., Nottingham, M., Ed., and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Caching](https://tools.ietf.org/html/rfc7234)”, RFC 7234, [DOI 10.17487/RFC7234](http://dx.doi.org/10.17487/RFC7234), June 2014, <[http://www.rfc-editor.org/info/rfc7234](http://www.rfc-editor.org/info/rfc7234)>.

**[RFC7235]**
Fielding, R., Ed. and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): Authentication](https://tools.ietf.org/html/rfc7235)”, RFC 7235, [DOI 10.17487/RFC7235](http://dx.doi.org/10.17487/RFC7235), June 2014, <[http://www.rfc-editor.org/info/rfc7235](http://www.rfc-editor.org/info/rfc7235)>.

**[TCP]**
Postel, J., “[Transmission Control Protocol](https://tools.ietf.org/html/rfc793)”, STD 7, RFC 793,[DOI 10.17487/RFC0793](http://dx.doi.org/10.17487/RFC0793), September 1981, <[http://www.rfc-editor.org/info/rfc793](http://www.rfc-editor.org/info/rfc793)>.

**[TLS-ALPN]**
Friedl, S., Popov, A., Langley, A., and E. Stephan, “[Transport Layer Security (TLS) Application-Layer Protocol Negotiation Extension](https://tools.ietf.org/html/rfc7301)”, RFC 7301,[DOI 10.17487/RFC7301](http://dx.doi.org/10.17487/RFC7301), July 2014, <[http://www.rfc-editor.org/info/rfc7301](http://www.rfc-editor.org/info/rfc7301)>.

**[TLS-ECDHE]**
Rescorla, E., “[TLS Elliptic Curve Cipher Suites with SHA-256/384 and AES Galois Counter Mode (GCM)](https://tools.ietf.org/html/rfc5289)”, RFC 5289, [DOI 10.17487/RFC5289](http://dx.doi.org/10.17487/RFC5289), August 2008, <[http://www.rfc-editor.org/info/rfc5289](http://www.rfc-editor.org/info/rfc5289)>.

**[TLS-EXT]**
Eastlake 3rd, D., “[Transport Layer Security (TLS) Extensions: Extension Definitions](https://tools.ietf.org/html/rfc6066)”, RFC 6066, [DOI 10.17487/RFC6066](http://dx.doi.org/10.17487/RFC6066), January 2011, <[http://www.rfc-editor.org/info/rfc6066](http://www.rfc-editor.org/info/rfc6066)>.

**[TLS12]**
Dierks, T. and E. Rescorla, “[The Transport Layer Security (TLS) Protocol Version 1.2](https://tools.ietf.org/html/rfc5246)”, RFC 5246, [DOI 10.17487/RFC5246](http://dx.doi.org/10.17487/RFC5246), August 2008, <[http://www.rfc-editor.org/info/rfc5246](http://www.rfc-editor.org/info/rfc5246)>.

## [12.2 资料性引用](https://http2.github.io/http2-spec/#rfc.section.12.2)

**[ALT-SVC]**
Nottingham, M., McManus, P., and J. Reschke, “[HTTP Alternative Services](https://tools.ietf.org/html/draft-ietf-httpbis-alt-svc-06)”, Internet-Draft draft-ietf-httpbis-alt-svc-06 (work in progress), February 2015.

**[BCP90]**
Klyne, G., Nottingham, M., and J. Mogul, “[Registration Procedures for Message Header Fields](https://tools.ietf.org/html/rfc3864)”, BCP 90, RFC 3864, September 2004, <[http://www.rfc-editor.org/info/bcp90](http://www.rfc-editor.org/info/bcp90)>.

**[BREACH]**
Gluck, Y., Harris, N., and A. Prado, “[BREACH: Reviving the CRIME Attack](http://breachattack.com/resources/BREACH%20-%20SSL,%20gone%20in%2030%20seconds.pdf)”, July 2013, <[http://breachattack.com/resources/BREACH%20-%20SSL,%20gone%20in%2030%20seconds.pdf](http://breachattack.com/resources/BREACH%20-%20SSL,%20gone%20in%2030%20seconds.pdf)>.

**[HTML5]**
Hickson, I., Berjon, R., Faulkner, S., Leithead, T., Doyle Navara, E., O'Connor, E., and S. Pfeiffer, “[HTML5](http://www.w3.org/TR/2014/REC-html5-20141028/)”, W3C Recommendation REC-html5-20141028, October 2014, <[http://www.w3.org/TR/2014/REC-html5-20141028/](http://www.w3.org/TR/2014/REC-html5-20141028/)>.

**[RFC3749]**
Hollenbeck, S., “[Transport Layer Security Protocol Compression Methods](https://tools.ietf.org/html/rfc3749)”, RFC 3749,[DOI 10.17487/RFC3749](http://dx.doi.org/10.17487/RFC3749), May 2004, <[http://www.rfc-editor.org/info/rfc3749](http://www.rfc-editor.org/info/rfc3749)>.

**[RFC4492]**
Blake-Wilson, S., Bolyard, N., Gupta, V., Hawk, C., and B. Moeller, “[Elliptic Curve Cryptography (ECC) Cipher Suites for Transport Layer Security (TLS)](https://tools.ietf.org/html/rfc4492)”, RFC 4492,[DOI 10.17487/RFC4492](http://dx.doi.org/10.17487/RFC4492), May 2006, <[http://www.rfc-editor.org/info/rfc4492](http://www.rfc-editor.org/info/rfc4492)>.

**[RFC6585]**
Nottingham, M. and R. Fielding, “[Additional HTTP Status Codes](https://tools.ietf.org/html/rfc6585)”, RFC 6585,[DOI 10.17487/RFC6585](http://dx.doi.org/10.17487/RFC6585), April 2012, <[http://www.rfc-editor.org/info/rfc6585](http://www.rfc-editor.org/info/rfc6585)>.

**[RFC7323]**
Borman, D., Braden, B., Jacobson, V., and R. Scheffenegger, Ed., “[TCP Extensions for High Performance](https://tools.ietf.org/html/rfc7323)”, RFC 7323, [DOI 10.17487/RFC7323](http://dx.doi.org/10.17487/RFC7323), September 2014, <[http://www.rfc-editor.org/info/rfc7323](http://www.rfc-editor.org/info/rfc7323)>.

**[TALKING]**
Huang, L., Chen, E., Barth, A., Rescorla, E., and C. Jackson, “[Talking to Yourself for Fun and Profit](http://w2spconf.com/2011/papers/websocket.pdf)”, 2011, <[http://w2spconf.com/2011/papers/websocket.pdf](http://w2spconf.com/2011/papers/websocket.pdf)>.

**[TLSBCP]**
Sheffer, Y., Holz, R., and P. Saint-Andre, “[Recommendations for Secure Use of Transport Layer Security (TLS) and Datagram Transport Layer Security (DTLS)](https://tools.ietf.org/html/rfc7525)”, BCP 195, RFC 7525, [DOI 10.17487/RFC7525](http://dx.doi.org/10.17487/RFC7525), May 2015, <[http://www.rfc-editor.org/info/rfc7525](http://www.rfc-editor.org/info/rfc7525)>.

# [A. TLS 1.2 加密套件黑名单](https://http2.github.io/http2-spec/#BadCipherSuites)

HTTP/2实现 **可以(MAY)** 将以下面的TLS 1.2 加密套件中的任何一个的协商作为类型 [INADEQUATE_SECURITY](https://http2.github.io/http2-spec/#INADEQUATE_SECURITY) 的连接错误([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler))处理：

* TLS_NULL_WITH_NULL_NULL
* TLS_RSA_WITH_NULL_MD5
* TLS_RSA_WITH_NULL_SHA
* TLS_RSA_EXPORT_WITH_RC4_40_MD5
* TLS_RSA_WITH_RC4_128_MD5
* TLS_RSA_WITH_RC4_128_SHA
* TLS_RSA_EXPORT_WITH_RC2_CBC_40_MD5
* TLS_RSA_WITH_IDEA_CBC_SHA
* TLS_RSA_EXPORT_WITH_DES40_CBC_SHA
* TLS_RSA_WITH_DES_CBC_SHA
* TLS_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_DH_DSS_EXPORT_WITH_DES40_CBC_SHA
* TLS_DH_DSS_WITH_DES_CBC_SHA
* TLS_DH_DSS_WITH_3DES_EDE_CBC_SHA
* TLS_DH_RSA_EXPORT_WITH_DES40_CBC_SHA
* TLS_DH_RSA_WITH_DES_CBC_SHA
* TLS_DH_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_DHE_DSS_EXPORT_WITH_DES40_CBC_SHA
* TLS_DHE_DSS_WITH_DES_CBC_SHA
* TLS_DHE_DSS_WITH_3DES_EDE_CBC_SHA
* TLS_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA
* TLS_DHE_RSA_WITH_DES_CBC_SHA
* TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_DH_anon_EXPORT_WITH_RC4_40_MD5
* TLS_DH_anon_WITH_RC4_128_MD5
* TLS_DH_anon_EXPORT_WITH_DES40_CBC_SHA
* TLS_DH_anon_WITH_DES_CBC_SHA
* TLS_DH_anon_WITH_3DES_EDE_CBC_SHA
* TLS_KRB5_WITH_DES_CBC_SHA
* TLS_KRB5_WITH_3DES_EDE_CBC_SHA
* TLS_KRB5_WITH_RC4_128_SHA
* TLS_KRB5_WITH_IDEA_CBC_SHA
* TLS_KRB5_WITH_DES_CBC_MD5
* TLS_KRB5_WITH_3DES_EDE_CBC_MD5
* TLS_KRB5_WITH_RC4_128_MD5
* TLS_KRB5_WITH_IDEA_CBC_MD5
* TLS_KRB5_EXPORT_WITH_DES_CBC_40_SHA
* TLS_KRB5_EXPORT_WITH_RC2_CBC_40_SHA
* TLS_KRB5_EXPORT_WITH_RC4_40_SHA
* TLS_KRB5_EXPORT_WITH_DES_CBC_40_MD5
* TLS_KRB5_EXPORT_WITH_RC2_CBC_40_MD5
* TLS_KRB5_EXPORT_WITH_RC4_40_MD5
* TLS_PSK_WITH_NULL_SHA
* TLS_DHE_PSK_WITH_NULL_SHA
* TLS_RSA_PSK_WITH_NULL_SHA
* TLS_RSA_WITH_AES_128_CBC_SHA
* TLS_DH_DSS_WITH_AES_128_CBC_SHA
* TLS_DH_RSA_WITH_AES_128_CBC_SHA
* TLS_DHE_DSS_WITH_AES_128_CBC_SHA
* TLS_DHE_RSA_WITH_AES_128_CBC_SHA
* TLS_DH_anon_WITH_AES_128_CBC_SHA
* TLS_RSA_WITH_AES_256_CBC_SHA
* TLS_DH_DSS_WITH_AES_256_CBC_SHA
* TLS_DH_RSA_WITH_AES_256_CBC_SHA
* TLS_DHE_DSS_WITH_AES_256_CBC_SHA
* TLS_DHE_RSA_WITH_AES_256_CBC_SHA
* TLS_DH_anon_WITH_AES_256_CBC_SHA
* TLS_RSA_WITH_NULL_SHA256
* TLS_RSA_WITH_AES_128_CBC_SHA256
* TLS_RSA_WITH_AES_256_CBC_SHA256
* TLS_DH_DSS_WITH_AES_128_CBC_SHA256
* TLS_DH_RSA_WITH_AES_128_CBC_SHA256
* TLS_DHE_DSS_WITH_AES_128_CBC_SHA256
* TLS_RSA_WITH_CAMELLIA_128_CBC_SHA
* TLS_DH_DSS_WITH_CAMELLIA_128_CBC_SHA
* TLS_DH_RSA_WITH_CAMELLIA_128_CBC_SHA
* TLS_DHE_DSS_WITH_CAMELLIA_128_CBC_SHA
* TLS_DHE_RSA_WITH_CAMELLIA_128_CBC_SHA
* TLS_DH_anon_WITH_CAMELLIA_128_CBC_SHA
* TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
* TLS_DH_DSS_WITH_AES_256_CBC_SHA256
* TLS_DH_RSA_WITH_AES_256_CBC_SHA256
* TLS_DHE_DSS_WITH_AES_256_CBC_SHA256
* TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
* TLS_DH_anon_WITH_AES_128_CBC_SHA256
* TLS_DH_anon_WITH_AES_256_CBC_SHA256
* TLS_RSA_WITH_CAMELLIA_256_CBC_SHA
* TLS_DH_DSS_WITH_CAMELLIA_256_CBC_SHA
* TLS_DH_RSA_WITH_CAMELLIA_256_CBC_SHA
* TLS_DHE_DSS_WITH_CAMELLIA_256_CBC_SHA
* TLS_DHE_RSA_WITH_CAMELLIA_256_CBC_SHA
* TLS_DH_anon_WITH_CAMELLIA_256_CBC_SHA
* TLS_PSK_WITH_RC4_128_SHA
* TLS_PSK_WITH_3DES_EDE_CBC_SHA
* TLS_PSK_WITH_AES_128_CBC_SHA
* TLS_PSK_WITH_AES_256_CBC_SHA
* TLS_DHE_PSK_WITH_RC4_128_SHA
* TLS_DHE_PSK_WITH_3DES_EDE_CBC_SHA
* TLS_DHE_PSK_WITH_AES_128_CBC_SHA
* TLS_DHE_PSK_WITH_AES_256_CBC_SHA
* TLS_RSA_PSK_WITH_RC4_128_SHA
* TLS_RSA_PSK_WITH_3DES_EDE_CBC_SHA
* TLS_RSA_PSK_WITH_AES_128_CBC_SHA
* TLS_RSA_PSK_WITH_AES_256_CBC_SHA
* TLS_RSA_WITH_SEED_CBC_SHA
* TLS_DH_DSS_WITH_SEED_CBC_SHA
* TLS_DH_RSA_WITH_SEED_CBC_SHA
* TLS_DHE_DSS_WITH_SEED_CBC_SHA
* TLS_DHE_RSA_WITH_SEED_CBC_SHA
* TLS_DH_anon_WITH_SEED_CBC_SHA
* TLS_RSA_WITH_AES_128_GCM_SHA256
* TLS_RSA_WITH_AES_256_GCM_SHA384
* TLS_DH_RSA_WITH_AES_128_GCM_SHA256
* TLS_DH_RSA_WITH_AES_256_GCM_SHA384
* TLS_DH_DSS_WITH_AES_128_GCM_SHA256
* TLS_DH_DSS_WITH_AES_256_GCM_SHA384
* TLS_DH_anon_WITH_AES_128_GCM_SHA256
* TLS_DH_anon_WITH_AES_256_GCM_SHA384
* TLS_PSK_WITH_AES_128_GCM_SHA256
* TLS_PSK_WITH_AES_256_GCM_SHA384
* TLS_RSA_PSK_WITH_AES_128_GCM_SHA256
* TLS_RSA_PSK_WITH_AES_256_GCM_SHA384
* TLS_PSK_WITH_AES_128_CBC_SHA256
* TLS_PSK_WITH_AES_256_CBC_SHA384
* TLS_PSK_WITH_NULL_SHA256
* TLS_PSK_WITH_NULL_SHA384
* TLS_DHE_PSK_WITH_AES_128_CBC_SHA256
* TLS_DHE_PSK_WITH_AES_256_CBC_SHA384
* TLS_DHE_PSK_WITH_NULL_SHA256
* TLS_DHE_PSK_WITH_NULL_SHA384
* TLS_RSA_PSK_WITH_AES_128_CBC_SHA256
* TLS_RSA_PSK_WITH_AES_256_CBC_SHA384
* TLS_RSA_PSK_WITH_NULL_SHA256
* TLS_RSA_PSK_WITH_NULL_SHA384
* TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DH_DSS_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DH_RSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DHE_DSS_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DHE_RSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DH_anon_WITH_CAMELLIA_128_CBC_SHA256
* TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256
* TLS_DH_DSS_WITH_CAMELLIA_256_CBC_SHA256
* TLS_DH_RSA_WITH_CAMELLIA_256_CBC_SHA256
* TLS_DHE_DSS_WITH_CAMELLIA_256_CBC_SHA256
* TLS_DHE_RSA_WITH_CAMELLIA_256_CBC_SHA256
* TLS_DH_anon_WITH_CAMELLIA_256_CBC_SHA256
* TLS_EMPTY_RENEGOTIATION_INFO_SCSV
* TLS_ECDH_ECDSA_WITH_NULL_SHA
* TLS_ECDH_ECDSA_WITH_RC4_128_SHA
* TLS_ECDH_ECDSA_WITH_3DES_EDE_CBC_SHA
* TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDH_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_NULL_SHA
* TLS_ECDHE_ECDSA_WITH_RC4_128_SHA
* TLS_ECDHE_ECDSA_WITH_3DES_EDE_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDH_RSA_WITH_NULL_SHA
* TLS_ECDH_RSA_WITH_RC4_128_SHA
* TLS_ECDH_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_ECDH_RSA_WITH_AES_128_CBC_SHA
* TLS_ECDH_RSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_RSA_WITH_NULL_SHA
* TLS_ECDHE_RSA_WITH_RC4_128_SHA
* TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
* TLS_ECDH_anon_WITH_NULL_SHA
* TLS_ECDH_anon_WITH_RC4_128_SHA
* TLS_ECDH_anon_WITH_3DES_EDE_CBC_SHA
* TLS_ECDH_anon_WITH_AES_128_CBC_SHA
* TLS_ECDH_anon_WITH_AES_256_CBC_SHA
* TLS_SRP_SHA_WITH_3DES_EDE_CBC_SHA
* TLS_SRP_SHA_RSA_WITH_3DES_EDE_CBC_SHA
* TLS_SRP_SHA_DSS_WITH_3DES_EDE_CBC_SHA
* TLS_SRP_SHA_WITH_AES_128_CBC_SHA
* TLS_SRP_SHA_RSA_WITH_AES_128_CBC_SHA
* TLS_SRP_SHA_DSS_WITH_AES_128_CBC_SHA
* TLS_SRP_SHA_WITH_AES_256_CBC_SHA
* TLS_SRP_SHA_RSA_WITH_AES_256_CBC_SHA
* TLS_SRP_SHA_DSS_WITH_AES_256_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
* TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDH_ECDSA_WITH_AES_256_CBC_SHA384
* TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
* TLS_ECDH_RSA_WITH_AES_128_CBC_SHA256
* TLS_ECDH_RSA_WITH_AES_256_CBC_SHA384
* TLS_ECDH_ECDSA_WITH_AES_128_GCM_SHA256
* TLS_ECDH_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDH_RSA_WITH_AES_128_GCM_SHA256
* TLS_ECDH_RSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_PSK_WITH_RC4_128_SHA
* TLS_ECDHE_PSK_WITH_3DES_EDE_CBC_SHA
* TLS_ECDHE_PSK_WITH_AES_128_CBC_SHA
* TLS_ECDHE_PSK_WITH_AES_256_CBC_SHA
* TLS_ECDHE_PSK_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_PSK_WITH_AES_256_CBC_SHA384
* TLS_ECDHE_PSK_WITH_NULL_SHA
* TLS_ECDHE_PSK_WITH_NULL_SHA256
* TLS_ECDHE_PSK_WITH_NULL_SHA384
* TLS_RSA_WITH_ARIA_128_CBC_SHA256
* TLS_RSA_WITH_ARIA_256_CBC_SHA384
* TLS_DH_DSS_WITH_ARIA_128_CBC_SHA256
* TLS_DH_DSS_WITH_ARIA_256_CBC_SHA384
* TLS_DH_RSA_WITH_ARIA_128_CBC_SHA256
* TLS_DH_RSA_WITH_ARIA_256_CBC_SHA384
* TLS_DHE_DSS_WITH_ARIA_128_CBC_SHA256
* TLS_DHE_DSS_WITH_ARIA_256_CBC_SHA384
* TLS_DHE_RSA_WITH_ARIA_128_CBC_SHA256
* TLS_DHE_RSA_WITH_ARIA_256_CBC_SHA384
* TLS_DH_anon_WITH_ARIA_128_CBC_SHA256
* TLS_DH_anon_WITH_ARIA_256_CBC_SHA384
* TLS_ECDHE_ECDSA_WITH_ARIA_128_CBC_SHA256
* TLS_ECDHE_ECDSA_WITH_ARIA_256_CBC_SHA384
* TLS_ECDH_ECDSA_WITH_ARIA_128_CBC_SHA256
* TLS_ECDH_ECDSA_WITH_ARIA_256_CBC_SHA384
* TLS_ECDHE_RSA_WITH_ARIA_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_ARIA_256_CBC_SHA384
* TLS_ECDH_RSA_WITH_ARIA_128_CBC_SHA256
* TLS_ECDH_RSA_WITH_ARIA_256_CBC_SHA384
* TLS_RSA_WITH_ARIA_128_GCM_SHA256
* TLS_RSA_WITH_ARIA_256_GCM_SHA384
* TLS_DH_RSA_WITH_ARIA_128_GCM_SHA256
* TLS_DH_RSA_WITH_ARIA_256_GCM_SHA384
* TLS_DH_DSS_WITH_ARIA_128_GCM_SHA256
* TLS_DH_DSS_WITH_ARIA_256_GCM_SHA384
* TLS_DH_anon_WITH_ARIA_128_GCM_SHA256
* TLS_DH_anon_WITH_ARIA_256_GCM_SHA384
* TLS_ECDH_ECDSA_WITH_ARIA_128_GCM_SHA256
* TLS_ECDH_ECDSA_WITH_ARIA_256_GCM_SHA384
* TLS_ECDH_RSA_WITH_ARIA_128_GCM_SHA256
* TLS_ECDH_RSA_WITH_ARIA_256_GCM_SHA384
* TLS_PSK_WITH_ARIA_128_CBC_SHA256
* TLS_PSK_WITH_ARIA_256_CBC_SHA384
* TLS_DHE_PSK_WITH_ARIA_128_CBC_SHA256
* TLS_DHE_PSK_WITH_ARIA_256_CBC_SHA384
* TLS_RSA_PSK_WITH_ARIA_128_CBC_SHA256
* TLS_RSA_PSK_WITH_ARIA_256_CBC_SHA384
* TLS_PSK_WITH_ARIA_128_GCM_SHA256
* TLS_PSK_WITH_ARIA_256_GCM_SHA384
* TLS_RSA_PSK_WITH_ARIA_128_GCM_SHA256
* TLS_RSA_PSK_WITH_ARIA_256_GCM_SHA384
* TLS_ECDHE_PSK_WITH_ARIA_128_CBC_SHA256
* TLS_ECDHE_PSK_WITH_ARIA_256_CBC_SHA384
* TLS_ECDHE_ECDSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_ECDHE_ECDSA_WITH_CAMELLIA_256_CBC_SHA384
* TLS_ECDH_ECDSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_ECDH_ECDSA_WITH_CAMELLIA_256_CBC_SHA384
* TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384
* TLS_ECDH_RSA_WITH_CAMELLIA_128_CBC_SHA256
* TLS_ECDH_RSA_WITH_CAMELLIA_256_CBC_SHA384
* TLS_RSA_WITH_CAMELLIA_128_GCM_SHA256
* TLS_RSA_WITH_CAMELLIA_256_GCM_SHA384
* TLS_DH_RSA_WITH_CAMELLIA_128_GCM_SHA256
* TLS_DH_RSA_WITH_CAMELLIA_256_GCM_SHA384
* TLS_DH_DSS_WITH_CAMELLIA_128_GCM_SHA256
* TLS_DH_DSS_WITH_CAMELLIA_256_GCM_SHA384
* TLS_DH_anon_WITH_CAMELLIA_128_GCM_SHA256
* TLS_DH_anon_WITH_CAMELLIA_256_GCM_SHA384
* TLS_ECDH_ECDSA_WITH_CAMELLIA_128_GCM_SHA256
* TLS_ECDH_ECDSA_WITH_CAMELLIA_256_GCM_SHA384
* TLS_ECDH_RSA_WITH_CAMELLIA_128_GCM_SHA256
* TLS_ECDH_RSA_WITH_CAMELLIA_256_GCM_SHA384
* TLS_PSK_WITH_CAMELLIA_128_GCM_SHA256
* TLS_PSK_WITH_CAMELLIA_256_GCM_SHA384
* TLS_RSA_PSK_WITH_CAMELLIA_128_GCM_SHA256
* TLS_RSA_PSK_WITH_CAMELLIA_256_GCM_SHA384
* TLS_PSK_WITH_CAMELLIA_128_CBC_SHA256
* TLS_PSK_WITH_CAMELLIA_256_CBC_SHA384
* TLS_DHE_PSK_WITH_CAMELLIA_128_CBC_SHA256
* TLS_DHE_PSK_WITH_CAMELLIA_256_CBC_SHA384
* TLS_RSA_PSK_WITH_CAMELLIA_128_CBC_SHA256
* TLS_RSA_PSK_WITH_CAMELLIA_256_CBC_SHA384
* TLS_ECDHE_PSK_WITH_CAMELLIA_128_CBC_SHA256
* TLS_ECDHE_PSK_WITH_CAMELLIA_256_CBC_SHA384
* TLS_RSA_WITH_AES_128_CCM
* TLS_RSA_WITH_AES_256_CCM
* TLS_RSA_WITH_AES_128_CCM_8
* TLS_RSA_WITH_AES_256_CCM_8
* TLS_PSK_WITH_AES_128_CCM
* TLS_PSK_WITH_AES_256_CCM
* TLS_PSK_WITH_AES_128_CCM_8
* TLS_PSK_WITH_AES_256_CCM_8

**Note:** This list was assembled from the set of registered TLS cipher suites at the time of writing. This list includes those cipher suites that do not offer an ephemeral key exchange and those that are based on the TLS null, stream, or block cipher type (as defined in [Section 6.2.3](https://tools.ietf.org/html/rfc5246#section-6.2.3) of [[TLS12]
](https://http2.github.io/http2-spec/#TLS12)). Additional cipher suites with these properties could be defined; these would not be explicitly prohibited.


# Acknowledgements
This document includes substantial input from the following individuals:
Adam Langley, Wan-Teh Chang, Jim Morrison, Mark Nottingham, Alyssa Wilk, Costin Manolache, William Chan, Vitaliy Lvin, Joe Chan, Adam Barth, Ryan Hamilton, Gavin Peters, Kent Alstad, Kevin Lindsay, Paul Amer, Fan Yang, and Jonathan Leighton (SPDY contributors).
Gabriel Montenegro and Willy Tarreau (Upgrade mechanism).
William Chan, Salvatore Loreto, Osama Mazahir, Gabriel Montenegro, Jitu Padhye, Roberto Peon, and Rob Trace (Flow control).
Mike Bishop (Extensibility).
Mark Nottingham, Julian Reschke, James Snell, Jeff Pinner, Mike Bishop, and Herve Ruellan (Substantial editorial contributions).
Kari Hurtta, Tatsuhiro Tsujikawa, Greg Wilkins, Poul-Henning Kamp, and Jonathan Thackray.
Alexey Melnikov, who was an editor of this document in 2013.

A substantial proportion of Martin's contribution was supported by Microsoft during his employment there.

The Japanese HTTP/2 community provided invaluable contributions, including a number of implementations as well as numerous technical and editorial contributions.

# [Authors' Addresses](https://http2.github.io/http2-spec/#rfc.authors)
**Mike Belshe**BitGoEmail: mike@belshe.com
**Roberto Peon**Google, IncEmail: fenix@google.com
**Martin Thomson** (editor) Mozilla331 E Evelyn StreetMountain View, CA 94041United StatesEmail: martin.thomson@gmail.com
