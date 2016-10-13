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

Protocol: HTTP/2 over TLS

Identification Sequence: 0x68 0x32 ("h2")

Specification: This document

The "h2c" string identifies HTTP/2 when used over cleartext TCP:

Protocol: HTTP/2 over TCP

Identification Sequence: 0x68 0x32 0x63 ("h2c")

Specification: This document

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

This document establishes a registry for HTTP/2 settings. The "HTTP/2 Settings" registry manages a 16-bit space. The "HTTP/2 Settings" registry operates under the ["Expert Review" policy](https://http2.github.io/http2-spec/#RFC5226) [RFC5226]
for values in the range from 0x0000 to 0xefff, with values between and 0xf000 and 0xffff being reserved for Experimental Use.

New registrations are advised to provide the following information:
Name:
A symbolic name for the setting. Specifying a setting name is optional.
Code:
The 16-bit code assigned to the setting.
Initial Value:
An initial value for the setting.
Specification:
An optional reference to a specification that describes the use of the setting.

The entries in the following table are registered by this document.

|Name                   |Code  |Initial Value |Specification    |
|-----------------------|------|--------------|-----------------|
|HEADER_TABLE_SIZE      |0x1   |4096          | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|ENABLE_PUSH            |0x2   |1             | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_CONCURRENT_STREAMS |0x3   |(infinite)    | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|INITIAL_WINDOW_SIZE    |0x4   |65535         | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_FRAME_SIZE         |0x5   |16384         | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|
|MAX_HEADER_LIST_SIZE   |0x6   |(infinite)    | [Section 6.5.2](https://http2.github.io/http2-spec/#SettingValues)|

## [11.4](https://http2.github.io/http2-spec/#rfc.section.11.4) [Error Code Registry](https://http2.github.io/http2-spec/#iana-errors)

This document establishes a registry for HTTP/2 error codes. The "HTTP/2 Error Code" registry manages a 32-bit space. The "HTTP/2 Error Code" registry operates under the ["Expert Review" policy](https://http2.github.io/http2-spec/#RFC5226) [RFC5226].

Registrations for error codes are required to include a description of the error code. An expert reviewer is advised to examine new registrations for possible duplication with existing error codes. Use of existing registrations is to be encouraged, but not mandated.

New registrations are advised to provide the following information:
Name:
A name for the error code. Specifying an error code name is optional.
Code:
The 32-bit error code value.
Description:
A brief description of the error code semantics, longer if no detailed specification is provided.
Specification:
An optional reference for a specification that defines the error code.

The entries in the following table are registered by this document.

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

## [11.5 HTTP2-Settings Header Field Registration](https://http2.github.io/http2-spec/#rfc.section.11.5)

This section registers the HTTP2-Settings header field in the "Permanent Message Header Field Names" registry [[BCP90]](https://http2.github.io/http2-spec/#BCP90).

Header field name: HTTP2-Settings

Applicable protocol: http

Status: standard

Author/Change controller: IETF

Specification document(s): [Section 3.2.1](https://http2.github.io/http2-spec/#Http2SettingsHeader) of this document

Related information: This header field is only used by an HTTP/2 client for Upgrade-based negotiation.

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
