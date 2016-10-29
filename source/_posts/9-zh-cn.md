---
title: HTTP/2规范：9. 额外的HTTP要求/注意事项
date: 2016-10-29 12:34:49
categories: HTTP2相关规范
---

这一节概括了提升互操作性，降低暴露给已知的安全脆弱性，或降低潜在的实现变异的HTTP协议属性。

<!--more-->

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
