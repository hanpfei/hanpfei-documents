# [10. 安全注意事项](https://http2.github.io/http2-spec/#security)

## [10.1 服务器认证](https://http2.github.io/http2-spec/#authority)

HTTP/2 relies on the HTTP/1.1 definition of authority for determining whether a server is authoritative in providing a given response (see [[RFC7230]](https://http2.github.io/http2-spec/#RFC7230), [Section 9.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#establishing.authority)). This relies on local name resolution for the "http" URI scheme and the authenticated server identity for the "https" scheme (see [[RFC2818]](https://http2.github.io/http2-spec/#RFC2818),[Section 3](https://tools.ietf.org/html/rfc2818#section-3)).

## [10.2 跨协议攻击](https://http2.github.io/http2-spec/#rfc.section.10.2)

在跨协议攻击中，攻击者导致客户端以某个协议对它的服务器初始化一个事务，而服务器则理解另一个不同的协议。攻击者也许能够使得事务以在第二个协议中是有效的样子出现。结合web上下文的容量，这可以被用于与私有网络中保护不好的服务器交互。

Completing a TLS handshake with an ALPN identifier for HTTP/2 can be considered sufficient protection against cross-protocol attacks. ALPN provides a positive indication that a server is willing to proceed with HTTP/2, which prevents attacks on other TLS-based protocols.

The encryption in TLS makes it difficult for attackers to control the data that could be used in a cross-protocol attack on a cleartext protocol.

The cleartext version of HTTP/2 has minimal protection against cross-protocol attacks. The connection preface ([Section 3.5](https://http2.github.io/http2-spec/#ConnectionHeader)) contains a string that is designed to confuse HTTP/1.1 servers, but no special protection is offered for other protocols. A server that is willing to ignore parts of an HTTP/1.1 request containing an Upgrade header field in addition to the client connection preface could be exposed to a cross-protocol attack.

## [10.3 中间封装攻击](https://http2.github.io/http2-spec/#rfc.section.10.3)

The HTTP/2 header field encoding allows the expression of names that are not valid field names in the Internet Message Syntax used by HTTP/1.1. Requests or responses containing invalid header field names MUST be treated as malformed ([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed)). An intermediary therefore cannot translate an HTTP/2 request or response containing an invalid field name into an HTTP/1.1 message.

Similarly, HTTP/2 allows header field values that are not valid. While most of the values that can be encoded will not alter header field parsing, carriage return (CR, ASCII 0xd), line feed (LF, ASCII 0xa), and the zero character (NUL, ASCII 0x0) might be exploited by an attacker if they are translated verbatim. Any request or response that contains a character not permitted in a header field value MUST be treated as malformed ([Section 8.1.2.6](https://http2.github.io/http2-spec/#malformed)). Valid characters are defined by the field-content  ABNF rule in [Section 3.2](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7230.html#header.fields) of [[RFC7230]](https://http2.github.io/http2-spec/#RFC7230).

## [10.4 推送的响应的可缓存性](https://http2.github.io/http2-spec/#rfc.section.10.4)

Pushed responses do not have an explicit request from the client; the request is provided by the server in the [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) frame.

Caching responses that are pushed is possible based on the guidance provided by the origin server in the Cache-Control header field. However, this can cause issues if a single server hosts more than one tenant. For example, a server might offer multiple users each a small portion of its URI space.

Where multiple tenants share space on the same server, that server MUST ensure that tenants are not able to push representations of resources that they do not have authority over. Failure to enforce this would allow a tenant to provide a representation that would be served out of cache, overriding the actual representation that the authoritative tenant provides.

Pushed responses for which an origin server is not authoritative (see [Section 10.1](https://http2.github.io/http2-spec/#authority)) MUST NOT be used or cached.

## [10.5 拒绝服务注意事项](https://http2.github.io/http2-spec/#dos)

An HTTP/2 connection can demand a greater commitment of resources to operate than an HTTP/1.1 connection. The use of header compression and flow control depend on a commitment of resources for storing a greater amount of state. Settings for these features ensure that memory commitments for these features are strictly bounded.

The number of [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) frames is not constrained in the same fashion. A client that accepts server push SHOULD limit the number of streams it allows to be in the "reserved (remote)" state. An excessive number of server push streams can be treated as a stream error ([Section 5.4.2](https://http2.github.io/http2-spec/#StreamErrorHandler)) of type[ENHANCE_YOUR_CALM](https://http2.github.io/http2-spec/#ENHANCE_YOUR_CALM).

Processing capacity cannot be guarded as effectively as state capacity.

The [SETTINGS](https://http2.github.io/http2-spec/#SETTINGS) frame can be abused to cause a peer to expend additional processing time. This might be done by pointlessly changing SETTINGS parameters, setting multiple undefined parameters, or changing the same setting multiple times in the same frame. [WINDOW_UPDATE](https://http2.github.io/http2-spec/#WINDOW_UPDATE) or[PRIORITY](https://http2.github.io/http2-spec/#PRIORITY) frames can be abused to cause an unnecessary waste of resources.

Large numbers of small or empty frames can be abused to cause a peer to expend time processing frame headers. Note, however, that some uses are entirely legitimate, such as the sending of an empty [DATA](https://http2.github.io/http2-spec/#DATA) or [CONTINUATION](https://http2.github.io/http2-spec/#CONTINUATION) frame at the end of a stream.

Header compression also offers some opportunities to waste processing resources; see [Section 7](https://tools.ietf.org/html/rfc7541#section-7) of[[COMPRESSION]](https://http2.github.io/http2-spec/#COMPRESSION) for more details on potential abuses.

Limits in [SETTINGS](https://http2.github.io/http2-spec/#SETTINGS) parameters cannot be reduced instantaneously, which leaves an endpoint exposed to behavior from a peer that could exceed the new limits. In particular, immediately after establishing a connection, limits set by a server are not known to clients and could be exceeded without being an obvious protocol violation.

All these features — i.e., [SETTINGS](https://http2.github.io/http2-spec/#SETTINGS) changes, small frames, header compression — have legitimate uses. These features become a burden only when they are used unnecessarily or to excess.

An endpoint that doesn't monitor this behavior exposes itself to a risk of denial-of-service attack. Implementations SHOULD track the use of these features and set limits on their use. An endpoint MAY treat activity that is suspicious as a connection error ([Section 5.4.1](https://http2.github.io/http2-spec/#ConnectionErrorHandler)) of type[ENHANCE_YOUR_CALM](https://http2.github.io/http2-spec/#ENHANCE_YOUR_CALM).

### [10.5.1 首部块大小的限制](https://http2.github.io/http2-spec/#MaxHeaderBlock)

A large header block ([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock)) can cause an implementation to commit a large amount of state. Header fields that are critical for routing can appear toward the end of a header block, which prevents streaming of header fields to their ultimate destination. This ordering and other reasons, such as ensuring cache correctness, mean that an endpoint might need to buffer the entire header block. Since there is no hard limit to the size of a header block, some endpoints could be forced to commit a large amount of available memory for header fields.

An endpoint can use the [SETTINGS_MAX_HEADER_LIST_SIZE](https://http2.github.io/http2-spec/#SETTINGS_MAX_HEADER_LIST_SIZE) to advise peers of limits that might apply on the size of header blocks. This setting is only advisory, so endpoints MAY choose to send header blocks that exceed this limit and risk having the request or response being treated as malformed. This setting is specific to a connection, so any request or response could encounter a hop with a lower, unknown limit. An intermediary can attempt to avoid this problem by passing on values presented by different peers, but they are not obligated to do so.

A server that receives a larger header block than it is willing to handle can send an HTTP 431 (Request Header Fields Too Large) status code [[RFC6585]](https://http2.github.io/http2-spec/#RFC6585). A client can discard responses that it cannot process. The header block MUST be processed to ensure a consistent connection state, unless the connection is closed.

### [10.5.2 CONNECT问题](https://http2.github.io/http2-spec/#connectDos)

CONNECT方法可以被用于创建代理上不成比例的负载，因为相对于TCP连接的创建和维护，创建流不那么昂贵。除携带了CONNECT请求的流的关闭之外，代理也许还要为TCP连接维护一些资源，因为外发的TCP连接保持为TIME_WAIT状态。然而，代理不能只依赖 [SETTINGS_MAX_CONCURRENT_STREAMS](https://http2.github.io/http2-spec/#SETTINGS_MAX_CONCURRENT_STREAMS) 来决定CONNECT请求消耗的资源。

## [10.6 压缩的使用](https://http2.github.io/http2-spec/#rfc.section.10.6)

当以与攻击者控制的数据相同的上下文压缩时，压缩可以使得攻击恢复机密数据。HTTP/2启用了首部字段压缩([Section 4.3](https://http2.github.io/http2-spec/#HeaderBlock))；下面的担忧同样适用于HTTP压缩 内容编码 ([[RFC7231]](https://http2.github.io/http2-spec/#RFC7231)，[Section 3.1.2.1](https://svn.tools.ietf.org/svn/wg/httpbis/specs/rfc7231.html#content.codings)) 的使用。



There are demonstrable attacks on compression that exploit the characteristics of the web (e.g.,[[BREACH]
](https://http2.github.io/http2-spec/#BREACH)). The attacker induces multiple requests containing varying plaintext, observing the length of the resulting ciphertext in each, which reveals a shorter length when a guess about the secret is correct.

Implementations communicating on a secure channel MUST NOT compress content that includes both confidential and attacker-controlled data unless separate compression dictionaries are used for each source of data. Compression MUST NOT be used if the source of data cannot be reliably determined. Generic stream compression, such as that provided by TLS, MUST NOT be used with HTTP/2 (see [Section 9.2](https://http2.github.io/http2-spec/#TLSUsage)).

Further considerations regarding the compression of header fields are described in[[COMPRESSION]
](https://http2.github.io/http2-spec/#COMPRESSION).

## [10.7 填充的使用](https://http2.github.io/http2-spec/#padding)

HTTP/2中的填充并不打算替代通用的填充，比如可能由 [TLS](https://http2.github.io/http2-spec/#TLS12) [TLS12]提供的填充。
冗余填充甚至可能会适得其反。正确的应用程序可能依赖被填充的数据的特定知识。

要减少依赖压缩的攻击，相对于填充而言，禁用或限制压缩可能是一个更适当的对策。

填充可被用于模糊帧内容的准确大小，并被提供用以减少HTTP内特定的攻击，比如，压缩的内容包含了攻击者控制的纯文本和机密数据的攻击(e.g.，[[BREACH]](https://http2.github.io/http2-spec/#BREACH))。

相对于似乎很明显能看到的，对填充的使用可能导致更少的保护。最好，填充只是通过增加攻击者不得不观察的帧数量，而使得攻击者更难推断长度信息。不正确地实现填充方案很容易被探测出来。特别地，可预测分布的随机填充提供了非常少的保护；类似地，固定大小的填充载荷

Use of padding can result in less protection than might seem immediately obvious. At best, padding only makes it more difficult for an attacker to infer length information by increasing the number of frames an attacker has to observe. Incorrectly implemented padding schemes can be easily defeated. In particular, randomized padding with a predictable distribution provides very little protection; similarly, padding payloads to a fixed size exposes information as payload sizes cross the fixed-sized boundary, which could be possible if an attacker can control plaintext.

中继 **应该(SHOULD)** 为 [DATA](https://http2.github.io/http2-spec/#DATA) 帧保留填充，但 **可以(MAY)** 丢弃[HEADERS](https://http2.github.io/http2-spec/#HEADERS) 和 [PUSH_PROMISE](https://http2.github.io/http2-spec/#PUSH_PROMISE) 帧的填充。中继改变帧的填充大小的一个有效的原因是，加强填充提供的保护。

## [10.8 隐私注意事项](https://http2.github.io/http2-spec/#rfc.section.10.8)

Several characteristics of HTTP/2 provide an observer an opportunity to correlate actions of a single client or server over time. These include the value of settings, the manner in which flow-control windows are managed, the way priorities are allocated to streams, the timing of reactions to stimulus, and the handling of any features that are controlled by settings.

As far as these create observable differences in behavior, they could be used as a basis for fingerprinting a specific client, as defined in [Section 1.8](http://www.w3.org/TR/2014/REC-html5-20141028/introduction.html#fingerprint) of [[HTML5]](https://http2.github.io/http2-spec/#HTML5).



HTTP/2's preference for using a single TCP connection allows correlation of a user's activity on a site. Reusing connections for different origins allows tracking across those origins.

由于PING和SETTINGS帧请求立即响应，它们可能被终端用于测量延时。这在某些场景下可能对隐私有影响。
