---
title: HTTP/2规范：3. 启动 HTTP/2
date: 2016-10-29 12:28:49
categories: HTTP2相关规范
---

一个HTTP/2连接是一个运行于一个TCP连接([[TCP]
](http://httpwg.org/specs/rfc7540.html#TCP))之上的应用层协议。客户端是TCP连接的发起者。

HTTP/2使用了与HTTP/1.1所使用的相同的"http"和"https" URI schemes。HTTP/2共享了相同的默认端口号："http" URIs的是80，"https" URIs的是443。作为结果，HTTP/2的实现在为处理诸如 http://example.org/foo 或 https://example.com/bar 这样的URIs的目标资源的请求时，需要首先发现upstream server(客户端希望建立连接的中间对端)是支持HTTP/2的。

<!--more-->

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
