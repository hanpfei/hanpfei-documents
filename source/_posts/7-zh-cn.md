---
title: HTTP/2规范：7. 错误码
date: 2016-10-29 12:32:49
categories: 网络协议
tags:
- 网络协议
- HTTP2
---

错误码是用在[RST_STREAM](https://http2.github.io/http2-spec/#RST_STREAM)和[GOAWAY](https://http2.github.io/http2-spec/#GOAWAY)帧中的32位字段，来携带流或连接错误的原因的。

错误码共享一个共同的码空间。一些错误码只应用于流或整个连接，而在其它上下文中没有定义语义。

<!--more-->

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
