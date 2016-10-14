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

