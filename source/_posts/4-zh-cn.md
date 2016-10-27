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
