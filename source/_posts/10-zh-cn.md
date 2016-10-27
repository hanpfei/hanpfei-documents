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
