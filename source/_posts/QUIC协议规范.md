---
title: QUIC协议规范
date: 2017-01-13 18:35:49
categories: 网络协议
tags:
- 网络协议
- QUIC
- 翻译
---

# 介绍
QUIC (Quick UDP Internet Connection，快速UDP互联网连接) 是一个新的基于UDP的多路复用且安全的传输协议，它从头开始设计，且为 HTTP/2 语义做了优化。尽管以 HTTP/2 作为主要的应用协议而构建，然而 QUIC 的构建是基于传输和安全领域数十年的经验的，且实现了使它成为有吸引力的现代通用传输协议的机制。QUIC提供了等价于
 HTTP/2 的多路复用和流控，等价于 TLS 的安全机制，及等价于 TCP 的连接语义、可靠性和拥塞控制。

<!--more-->

QUIC完全运行于用户空间，它当前作为 Chromium 浏览器的一部分发布给用户，以便于快速的部署和实验。作为基于
 UDP 的用户空间传输协议，QUIC 可以做一些由于遗留的客户端和中间设备，或旷日持久的操作系统开发和部署周期的阻碍，而被证明很难在现有的协议中部署的创新。

QUIC 的一个重要目标是通过快速的实验获得更好的传输设计相关的知识。作为结果，我们希望将其中的一些精华的改动迁移进 TCP 和 TLS，后者通常有着长得多的迭代周期。

这份文档描述标准化前 QUIC 协议的概念设计和协议规范。补充资料描述了加密和传输握手 [QUIC-CRYPTO]，及丢失恢复和拥塞控制 [draft-iyengar-quic-loss-recovery]。其它资源，包括一份更详细的相关文档，可以在 [Chromium 的 QUIC 主页](https://www.chromium.org/quic) 找到。

基于早期的部署的 QUIC 标准化建议为 [draft-hamilton-quic-transport-protocol]，[draft-shade-quic-http2-mapping]，[draft-iyengar-quic-loss-recovery]，和 [draft-thomson-quic-tls]。

# 术语和定义
QUIC中使用的所有整型值，包括长度、版本号和类型，都是小尾端字节序，而不是网络字节序。QUIC不强制动态大小的帧中的类型对齐。

贯穿本文档使用的一些术语定义如下。
* “客户端”：初始化 QUIC 连接的端点。
* “服务器”：接受进入的 QUIC 连接的端点。
* “端点”：连接的客户端或服务器端。
* “流”：QUIC 连接中穿过一个逻辑通道的双向字节流。
* “连接”：两个 QUIC 端点之间的会话，它具有一个单独的加密上下文且包含多路复用流。
* “连接ID”：QUIC 连接的标识符。
* “QUIC包”：经过良好格式化的 UDP 载荷，可由 QUIC 接收者解析。本文档中的 QUIC 包大小指 UDP 载荷大小。

# QUIC概述

我们现在简要介绍 QUIC 的关键机制和优势。QUIC 在功能上等价于 TCP + TLS + HTTP/2，但基于 UDP 实现。QUIC 相对于 TCP + TLS + HTTP/2 的主要优势包括：
* 连接建立延迟
* 灵活的拥塞控制
* 多路复用而不存在队首阻塞
* 认证和加密的首部和载荷
* 流和连接的流量控制
* 连接迁移

## 连接建立延迟
QUIC将加密和传输握手结合在一起，减少了建立一条安全连接所需的往返。QUIC 连接通常是 0-RTT，意味着相比于 TCP + TLS 中发送应用数据前需要 1-3 个往返的情况，在大多数 QUIC 连接中，数据可以被立即发送而无需等待服务器的响应。

QUIC 提供了一个专门的流（流 ID 为1）用于执行握手，但握手协议的详细内容超出了本文档的范围。要查看当前握手协议的完整描述，请参考 [QUIC Crypto Handshake](http://goo.gl/jOvOQ5) 文档。QUIC 当前的握手协议将在未来被 TLS 1.3 替代。

## 灵活的拥塞控制

QUIC 具有可插入的拥塞控制，且有着比 TCP 更丰富的信令，这使得 QUIC 相对于 TCP 可以为拥塞控制算法提供更丰富的信息。当前，默认的拥塞控制是
 TCP Cubic 的重实现；我们目前在实验替代的方法。

更丰富的信息的一个例子是，每个包，包括原始的和重传的，都携带一个新的包序列号。这使得 QUIC 发送者可以将重传包的 ACKs 与原始传输包的 ACKs 区分开来，这样可以避免 TCP 的重传模糊问题。QUIC ACKs 也显式地携带数据包的接收与其确认被发送之间的延迟，与单调递增的包序列号一起，这样可以精确地计算往返时间（RTT）。

最后，QUIC 的 ACK 帧最多支持 256 个 ack 块，因此在重排序时，QUIC 相对于 TCP（使用SACK）更有弹性，这也使得在重排序或丢失出现时，QUIC 可以在线上保留更多在途字节。客户端和服务器都可以更精确地了解到哪些包对端已经接收。

## 流和连接的流量控制

QUIC 实现了流级和连接级的流量控制，紧跟 HTTP/2 的流量控制。QUIC 的流级流控工作如下。QUIC 接收者通告每个流中接收者最多想要接收的数据的绝对字节偏移。随着数据在特定流中的发送，接收和传送，接收者发送WINDOW_UPDATE 帧，帧增加该流的通告偏移量限制，允许对端在该流上发送更多的数据。

除了每个流的流控制外，QUIC 还实现连接级的流控制，以限制 QUIC 接收者愿意为连接分配的总缓冲区。连接的流控制工作方式与流的流控制一样，但传送的字节和最大的接收偏移是所有流的总和。

与 TCP 的接收窗口自动调整类似，QUIC 实现流和连接流控制器的流控制信用的自动调整。如果 QUIC 的自动调整似乎限制了发送方的速率，并且在接收应用程序缓慢的时候抑制发送方，则 QUIC 的自动调整会增加每个 WINDOW_UPDATE 帧发送的信用额。

## 多路复用

基于 TCP 的 HTTP/2 深受 TCP 的队首阻塞问题困扰。由于 HTTP/2 在 TCP 的单个字节流抽象之上多路复用许多流，一个 TCP 片段的丢失将导致所有后续片段的阻塞直到重传到达，而封装在后续片段中的 HTTP/2 流可能和丢失的片段毫无关系。

由于 QUIC 是为多路复用操作从头设计的，携带个别流的的数据的包丢失时，通常只影响该流。每个流的帧可以在到达时立即发送给该流，因此，没有丢失数据的流可以继续重新汇集，并在应用程序中继续进行。

警告：当前 QUIC 在一个专门的首部流 (3) 中，通过 HTTP/2 HPACK 首部压缩压缩 HTTP 首部，则只有首部帧会出现队首阻塞问题。

## 认证和加密的首部和载荷

TCP 首部在网络中以明文出现，它没有经过认证，这导致了大量的 TCP 注入和首部管理问题，比如接收窗口管理和序列号覆写。尽管这些问题中的一些是主动攻击，有时其它则是一些网络中的中间盒子用来尝试透明地提升 TCP 性能的机制。然而，甚至 “性能增强” 中间设备依然有效地限制着传输协议的发展，这已经在 MPTCP 的设计及其后续的部署问题中观察到。

QUIC 数据包总是经过认证的，而且典型情况下载荷是全加密的。数据包头部不加密的部分依然会被接收者认证，以阻止任何第三方的数据包注入或操纵。QUIC 保护连接的端到端通信免遭智能或不知情的中间设备操纵。

警告：复位连接的 PUBLIC _RESET 包当前未经认证。

## 连接迁移

TCP 连接由源地址，源端口，目标地址和目标端口的4元组标识。TCP 一个广为人知的问题是，IP 地址改变（比如，由 WiFi 网络切换到移动网络）或端口号改变（当客户端的NAT绑定超时导致服务器看到的端口号改变）时连接会断掉。尽管 MPTCP 解决了 TCP 的连接迁移问题，但它依然为缺少中间设备和OS部署支持所困扰。

QUIC连接由一个 64-bit 连接 ID 标识，它由客户端随机地产生。在IP地址改变和 NAT 重绑定时，QUIC 连接可以继续存活，因为连接 ID 在这些迁移过程中保持不变。由于迁移客户端继续使用相同的会话密钥来加密和解密数据包，QUIC还提供了迁移客户端的自动加密验证。

当连接明确地用4元组标识时，比如服务器使用短暂的端口给客户端发送数据包时，有一个选项可用来不发送连接 ID 以节省线上传输的字节。

# 包类型和格式

QUIC 具有特殊包和普通包。有两种类型特殊包：版本协商包 (Version Negotiation Packets) 和 公共复位包 (Public Reset Packets)，普通包包含帧。

所有 QUIC 包的大小应该适配在路径的 MTU 以避免IP分片。路径 MTU 发现是正在进行中的工作，而当前 QUIC 实现为 IPv6 使用 1350 字节的最大QUIC包大小，IPv4 使用1370字节。两个大小都没有 IP 和 UDP 过载。

## QUIC公共包头

传输的所有 QUIC 包以大小介于1至51字节的公共包头开始。公共包头的格式如下：

```
--- src
     0        1        2        3        4            8
+--------+--------+--------+--------+--------+---    ---+
| Public |    Connection ID (64)    ...                 | ->
|Flags(8)|      (optional)                              |
+--------+--------+--------+--------+--------+---    ---+
     9       10       11        12   
+--------+--------+--------+--------+
|      QUIC Version (32)            | ->
|         (optional)                |                           
+--------+--------+--------+--------+
    13       14       15        16      17       18       19       20
+--------+--------+--------+--------+--------+--------+--------+--------+
|                        Diversification Nonce                          | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+
    21       22       23        24      25       26       27       28
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+
    29       30       31        32      33       34       35       36
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+
    37       38       39        40      41       42       43       44
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+
    45      46       47        48       49       50
+--------+--------+--------+--------+--------+--------+
|           Packet Number (8, 16, 32, or 48)          |
|                  (variable length)                  |
+--------+--------+--------+--------+--------+--------+
---
```
载荷可以包含多个如下所述类型相关的头部字节。

公共头部中的字段如下：
* **公共标记（Public Flags）**：
  * 0x01 = PUBLIC_FLAG_VERSION。这个标记的含义与包是由服务器还是客户端发送的有关。当由客户端发送时，设置它表示头部包含 QUIC 版本 (参考下面的说明)。客户端必须在所有的包中设置这个位，直到客户端收到来自服务器的确认，同意所提议的版本。服务器通过发送不设置该位的包来表示同意版本。当这个位由服务器设置时，包是版本协商包。版本协商在后面更详细地描述。
  * 0x02 = PUBLIC_FLAG_RESET。设置来表示包是公共复位包。
  * 0x04 = 表明在头部中存在 32字节的多样化随机数。
  * 0x08 = 表明包中存在完整的8字节连接ID。必须为所有包设置该位，直到为给定方向协商出不同的值 (比如，客户端可以请求包含更少字节的连接ID)。
  * 0x30 处的两位表示每个包中存在的数据包编号的低位字节数。这些位只用于帧包。没有包号的公共复位和版本协商包 (由服务器发送) ，不使用这些位，且必须被设置为0。这2位的掩码：
    * 0x30 表示包号占用6个字节。
    * 0x20 表示包号占用4个字节。
    * 0x10 表示包号占用2个字节。
    * 0x00 表示包号占用1个字节。
  * 0x40 为多路径使用保留。
  * 0x80 当前未使用，且必须被设置为0。
* **连接ID**：这是客户端选择的无符号64位统计随机数，该数字是连接的标识符。由于 QUIC 的连接被设计为，即使客户端漫游，连接依然保持建立状态，因而 IP 4元组（源IP，源端口，目标IP，目标端口）可能不足以标识连接。对每个传输方向，当4元组足以标识连接时，连接ID可以省略。
* **QUIC版本**：表示 QUIC 协议版本的32位不透明标记。只有在公共标记包含 FLAG_VERSION（比如 public_flags & FLAG_VERSION !=0） 时才存在。客户端可以设置这个标记，并 **准确** 包含一个提议版本，同时包含任意的数据（与该版本一致）。当客户端提议的版本不支持时，服务器可以设置这个标记，并可以提供一个可接受版本的列表（0或多个），但 **一定不能(MUST not)** 在版本信息之后包含任何数据。最近的实验版本的版本值示例包括 "Q025"，它对应于 byte 9 包含 'Q"，byte 10 包含 '0"，等等。[参考本文末尾的不同版本变化列表。]
* **包号**：包号的低 8，16，32，或 48 位，基于公共标记的 FLAG_?BYTE_SEQUENCE_NUMBER 标记被设置为什么。每个普通包（与特别的公共复位和版本协商包相反）由发送者分配包号。由某一端发送的首包包号应该为1，后续每个包的包号应该比前一个的大1。
包号的低64位被用作加密随机数的一部分；然而，QUIC 端点一定不能发送其包号无法以 64 位表示的包。如果 QUIC 端点传输了包号为 (2^64-1) 的包，则该包必须包含错误码为 QUIC_SEQUENCE_NUMBER_LIMIT_REACHED 的 CONNECTION_CLOSE 帧，且对端一定不能再传输任何其它包。
最多传输包号的低48位。要使接收者可以明确的重建包号，QUIC端点一定不能传输一个确认包已知已经由接收者发送的最大包号大 (2^(bitlength-2)) 的包。然而，在途包的数目不能超过 (2^46)。
任何截断的包号应该被推断为具有  最接近但大于  传输最初包含了截断包号的包的对端  的最大已知包号的值。包号的发送部分匹配推断值的最低位。

公共标记处理流程图如下：
```
--- src
Check the public flags in public header
                 |
                 |
                 V
           +--------------+
           | Public Reset |    YES
           | flag set?    |---------------> Public Reset Packet
           +--------------+
                 |
                 | NO
                 V
           +------------+          +-------------+
           | Version    |   YES    | Packet sent |  YES
           | flag set?  |--------->| by server?  |--------> Version Negotiation
           +------------+          +-------------+               Packet
                 |                        |
                 | NO                     | NO
                 V                        V
           Regular Packet         Regular Packet with
                               QUIC Version present in header
---
```

## 特殊包

### *版本协商包*

只有服务器会发送版本协商包。版本协商包以8位的公共标记和64位的连接ID开始。公共标记必须设置PUBLIC_FLAG_VERSION，并指明64位的连接ID。版本协商包的其余部分是服务器支持的版本的4字节列表：
```
--- src
     0        1        2        3        4        5        6        7       8
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| Public |    Connection ID (64)                                                 | ->
|Flags(8)|                                                                       |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
     9       10       11        12       13      14       15       16       17
+--------+--------+--------+--------+--------+--------+--------+--------+---...--+
|      1st QUIC version supported   |     2nd QUIC version supported    |   ...
|      by server (32)               |     by server (32)                |             
+--------+--------+--------+--------+--------+--------+--------+--------+---...--+
---
```

### *公共复位包*
公共复位包以8位的公共标记和64位的连接ID开始。公共标记必须设置 PUBLIC_FLAG_RESET，并表明64位的连接ID。公共复位包的其余部分像标记 PRST 的加密握手消息那样编码（参考[QUIC-CRYPTO]）：
```
--- src
     0        1        2        3        4         8
+--------+--------+--------+--------+--------+--   --+
| Public |    Connection ID (64)                ...  | ->
|Flags(8)|                                           |
+--------+--------+--------+--------+--------+--   --+
     9       10       11        12       13      14       
+--------+--------+--------+--------+--------+--------+---
|      Quic Tag (32)                |  Tag value map      ... ->
|         (PRST)                    |  (variable length)                         
+--------+--------+--------+--------+--------+--------+---
---
```
**标记值映射**：标记值映射包含如下的标记值：
* RNON (public reset nonce proof) - 一个64位的无符号整数。必须。
* RSEQ (rejected packet number) - 一个64位的包号。必须。
* CADR (client address) - 观察到的客户端IP地址和端口号。它当前只被用于调试，因而是可选的。

(TODO：公共复位包应该包含认证的（目标）服务器 IP/端口。)

## 普通包
普通包已经过认证和加密。公共头部已认证但未加密，从第一帧开始的包的其余部分已加密。紧随公共头部之后，普通包包含 AEAD（authenticated encryption and associated data）数据。要解释内容，这些数据必须先解密。解密之后，明文由一系列帧组成。

(TODO: 文档化加密和解密的输入，并描述试用解密)

### *帧包*
帧包具有一个载荷，它是一系列的类型前缀帧。帧类型的格式将在本文档的后面定义，但帧包的通用格式如下：
```
--- src
+--------+---...---+--------+---...---+
| Type   | Payload | Type   | Payload |
+--------+---...---+--------+---...---+
---
```

# QUIC 连接的生命周期

## 连接建立

QUIC客户端初始化一个连接。QUIC的连接建立将版本协商与加密和传输握手交织在一起以减少连接建立延迟。我们将在下面首先描述版本协商。

最初由客户端发向服务器的每个包必须设置版本标记，而且必须指定使用的协议版本。客户端发送的每个包必须开启版本标记，直到它从服务器收到了版本标记关闭的包。在服务器从客户端收到了第一个版本标记关闭的包之后，它必须忽略任何版本标记打开的包（可能由于延迟）。

当服务器收到一个含有新连接的连接ID的包，它将对比客户端的版本和它支持的版本。如果服务器可以接受客户端的版本，服务器将为连接的整个生命周期使用这个协议版本。在这种情况下，服务器发送的所有包的版本标记都是关闭的。

如果客户端的版本不被服务器接受，则将导致1-RTT的延迟。服务器将发送一个版本协商包给客户端。这个包将设置版本标记，并将包含服务器支持的版本的集合。

当客户端从服务器收到一个版本协商包，它将选择一个可接受的协议版本并使用这个版本重发所有包。这些包必须持续设置版本标记，而且必须包含新协商的协议版本。最后，客户端从服务器收到第一个普通包（比如，一个非版本协商包）表明版本协商的结束，此后客户端发送的所有后续包版本标记关闭。

为了避免降级攻击，客户端在第一个包中指定的协议版本，以及服务器支持的版本集合必须被包含在加密的握手数据中。客户端需要验证握手中的服务器版本列表与版本协商包中的版本列表匹配。服务器需要验证握手中的客户端版本表示一个它实际上不支持的协议版本。

连接建立的其余部分在握手文档中描述 [QUIC-CRYPTO]。加密握手在专门的加密流（流 ID 1）中执行。

在连接握手期间，握手必须协商多种传输参数。当前已定义的传输参数在本文档的后面描述。

## 数据传输
QUIC实现了连接可靠性，拥塞控制，和流量控制。QUIC流量控制与HTTP/2的流量控制很接近。QUIC可靠性和拥塞控制在一份附带文档中描述。QUIC连接为跨连接的共享拥塞控制和丢失恢复，而使用一个单独的包序列号空间。

QUIC连接中传输的所有数据，包括加密握手，被作为流内的数据发送，但ACKs确认QUIC包。

这个部分概念性地描述一个QUIC连接内数据传输的流的使用。本节提到的各种各样的帧在 帧类型和格式 一节中描述。

### *QUIC流的生命*
流是独立的双向数据序列，且被切割为流帧。流可以由客户端创建，也可以由服务器创建，可以与其它流并行交错地发送数据，且可以取消。QUIC流的生命周期模型与HTTP/2 [RFC 7540] 的很接近。（QUIC流的HTTP/2使用在本文档的后面部分有更详细的描述。）

流创建显式地完成，通过为一个给定的流发送一个STREAM帧。为了避免流ID冲突，流 ID 必须是偶数，如果流是由服务器初始化的话，如果流由客户端初始化则必须为奇数。0不是一个有效的流 ID。流 1 被保留用来加密握手，它应该是第一个客户端初始化的流。当基于QUIC使用HTTP/2时，流 3 被保留来为其它流传输压缩的首部，以确保首部的处理和传送可靠且有序。

随着新流的创建，连接的每一边的 流 ID 必须单调地递增。比如 流 2 可能在 流 3 之后创建，但 流 7 一定不能在 流 9 之后创建。对端可以接收乱序的流。比如，如果服务器收到了包 10，其中包含 流 9 的帧，在它收到包含 流 7 的帧的 包 9 之前，它应该优雅地处理这种情况。

如果端点收到一个STREAM帧，但它不想接受流，它可以立即以一个RST_STREAM帧（稍后描述）响应。注意，然而，初始化流的端点可能也已经在那个流上发送了数据；这些数据必须被忽略。

一旦流创建好，它可被用于发送和接收数据。这意味着一系列的流帧可被QUIC端点在那个流上发送，直到流在那个方向上被终止。

QUIC连接的任何一端都可以正常地终止一个流。有三种方式可以终止流：
1. **正常终止：**由于流是双向的，流可以是 "half-closed（半关闭）"或"closed（关闭）"状态。当流的一边发送一个FIN位被设为ture的帧，流被认为在那个方向上是"half-closed（半关闭）"的。FIN指明这个流上打开了FIN的发送者将不会在这个流上发送更多数据了。当QUIC的两个端点都发送并接收到了FIN，则端点认为流是"closed（关闭）"状态的。尽管FIN应该随着流的最后的用户数据一起发送，但FIN位可以被流的最后的数据帧后面的空流帧发送。
2. **异常终止：**客户端或服务器可以在任何时候为一个流发送RST_STREAM帧。RST_STREAM帧包含一个错误码用以指示失败原因（本文当的后面部分会列出错误码）。当流的发起者发送了一个RST_STREAM帧，它表示完成流失败了，而且不会有更多的数据在那个流上发送了。当RST_STREAM帧是由流的接收者发送的时，发送者，一旦接收，应该停止在那个流上发送任何数据。流接收者应该意识到发送者已经传输的数据和RST_STREAM帧接收的时间之间存在着竞态。为了确保连接级的流量控制可以被正确的实现，即使收到了一个RST_STREAM帧，发送者依然需要确保两者之一：对端收到流的FIN和所有字节或者对端收到一个RST_STREAM帧。这还意味着RST_STREAM帧的发送者需要持续以适当的WINDOW_UPDATE响应进入的那个流的STREAM_FREAME以确保发送者不让流量控制被阻塞而试图传送FIN。
3. 当连接终止时流也会被终止，如在下一节描述的那样。

## 连接终止
连接应该保持打开状态，直到他们在预协商周期的时间后变为空闲。当服务器决定终止一个空闲的连接时，它不应该通知客户端来避免唤醒移动设备的无线电模块。QUIC连接，一旦建立，可由两种方式中的一种终止：
1. **显式关闭：**一个端点发送一个CONNECTION_CLOSE帧给对端来初始化一个连接终止。一个端点可以在一个CONNECTION_CLOSE之前发送一个GOAWAY帧给对端来表明连接将在不久后终止。当发送GOAWAY帧时，通知对端任何活跃的流将继续被处理，但GOAWAY的发送者将不再初始化任何额外的流，且不接受任何新进入的流。在任何活跃的流的终止中，可以发送CONNECTION_CLOSE。如果一个端点在未终止的流活跃时发送了一个CONNECTION_CLOSE帧（一个或多个流还没有FIN位或RST_STREAM帧被发送或接收），则对端必须假设流是不完整的且被异常地终止。
2. **隐式关闭：**QUIC连接默认的空闲超时时间是30秒，且是连接协商中的一个必须参数("ICSL")。最大值是10分钟。如果在空闲超时期间没有网络活动，连接将关闭。默认情况下将发送一个CONNECTION_CLSOE帧。当发送一个显式的关闭比较昂贵时可以启用安静关闭选项，比如移动网络必须唤醒无线电模块。

一个端点还可以在连接期间的任何时间发送一个PUBLIC_RESET包来突然地终止活跃的连接。QUIC中的PUBLIC_RESET等价于TCP的RST。

# 帧类型和格式
QUIC帧包由帧填充。它具有一个帧类型字节，它本身具有一个依赖类型的解释，后面是依赖类型的帧首部字段。所有的帧被包含在单独的QUIC包中，且没有帧可以跨越QUIC包边界。

## 帧类型
帧类型字节有两种解释，导致两种帧类型：特殊帧类型，和普通帧类型。特殊帧类型在帧类型字节中同时编码帧类型和对应的标记，而普通帧类型简单地使用帧类型字节。

当前定义的特殊帧类型如下：
```
--- src
   +------------------+-----------------------------+
   | Type-field value |     Control Frame-type      |
   +------------------+-----------------------------+
   |     1fdooossB    |  STREAM                     |
   |     01ntllmmB    |  ACK                        |
   |     001xxxxxB    |  CONGESTION_FEEDBACK        |
   +------------------+-----------------------------+
---
```
当前定义的普通帧类型如下：
```
--- src
   +------------------+-----------------------------+
   | Type-field value |     Control Frame-type      |
   +------------------+-----------------------------+
   | 00000000B (0x00) |  PADDING                    |
   | 00000001B (0x01) |  RST_STREAM                 |
   | 00000010B (0x02) |  CONNECTION_CLOSE           |
   | 00000011B (0x03) |  GOAWAY                     |
   | 00000100B (0x04) |  WINDOW_UPDATE              |
   | 00000101B (0x05) |  BLOCKED                    |
   | 00000110B (0x06) |  STOP_WAITING               |
   | 00000111B (0x07) |  PING                       |
   +------------------+-----------------------------+
---
```
## **STREAM帧**
STREAM帧同时被用于隐式地创建流和在流上发送数据，它的格式如下：
```
--- src
     0        1       …               SLEN
+--------+--------+--------+--------+--------+
|Type (8)| Stream ID (8, 16, 24, or 32 bits) |
|        |    (Variable length SLEN bytes)   |
+--------+--------+--------+--------+--------+
  SLEN+1  SLEN+2     …                                         SLEN+OLEN   
+--------+--------+--------+--------+--------+--------+--------+--------+
|   Offset (0, 16, 24, 32, 40, 48, 56, or 64 bits) (variable length)    |
|                    (Variable length: OLEN  bytes)                     |
+--------+--------+--------+--------+--------+--------+--------+--------+
  SLEN+OLEN+1   SLEN+OLEN+2
+-------------+-------------+
| Data length (0 or 16 bits)|
|  Optional(maybe 0 bytes)  |
+------------+--------------+
---
```
STREAM帧首部中的字段如下：
* **帧类型：**帧类型字节是一个包含多种标记 (1fdooossB) 的8位值：
  * 最左边的位必须被设为 1 以指明这是一个STREAM帧。
  * 'f' 位是FIN位。当被设置为 1 时，这个位表明发送者已经完成在流上的发送并希望 "half-close（半关闭）"（稍后将详细描述）。本文档的后面将更详细地描述。
  * 'd' 位表明STREAM头部中是否包含数据长度。当设为0时，这个字段表明STREAM帧扩展至包的结尾。
  * 接下来的三个'ooo'位编码Offset头部字段的长度为0，16，24，32，40，48，56，或64位长。
  * 接下来的两个 'ss' 位编码流 ID头部字段的长度为 8，16，24，或32位长。
* **流 ID：**一个大小可变的流唯一的无符号ID。
* **偏移：**一个大小可变的无符号数字指定流中这块数据的字节偏移。
* **数据长度：**一个可选的16位无符号数字指定这个流帧中数据的长度。只有当包是 "全大小(full-sized)" 包时，才应该省略长度，来避免填充破坏的风险。

一个流帧必须总是要么具有非零的数据长度，要么设置了FIN位。

## ACK帧
ACK帧被发送用以通知对端哪些包已经收到，还有哪些包依然被接收者认为丢失了（丢失包的内容可能需要被重发）。ACK帧包含1 到 256 个ack块。Ack块是确认的包的范围，与TCP的SACK块类似，但QUIC没有等价的TCP的累积ack点，由于包将以新的序列号重传。

要限制ACK块为还没有被对端接收的，对端周期性地发送STOP_WAITING帧，用来通知接收者停止确认小于特定序列号的包，再接收者产生"最小未确认"包数字。ACK帧的发送者这样只报告那些在接收到的最小未确认和报告的最大已发现包号之间的ACK块。建议发送者在ack中发送它已经接收到的最近最大确认包，作为stop waiting帧的最小未确认值。

不像TCP SACK，QUIC ACK块是不可变的，因此一旦一个包被确认了，即使它没有出现在未来的ack帧中，它也被假设已经确认。

作为QUIC的已废弃的熵的替代，发送者可以有意地跳过包号来为连接引入熵。如果一个未发送的包号被确认了，则发送者必须总是关闭连接，因此这种机制自动地防御了任何潜在的攻击。ack格式在表达丢失包的块上是比较高效的，因此对于接收者和发送者这还是成本比较低的，且可以根据需要有效地提供至多8 位的熵，而不是通过恒定的开销来实现8位的熵。8位是ack范围和ack格式之间可高效地表达的最大gap。


### **段偏移**
**0**：Ack帧的起始位置。
**T**：时间戳段起始位置的字节偏移量。
**A**：Ack块段起始位置的字节偏移量。
**N**：最大已确认的字节长度。
```
--- src
     0                            1  => N                     N+1 => A(aka N + 3)
+---------+-------------------------------------------------+--------+--------+
|   Type  |                   Largest Acked                 |  Largest Acked  |
|   (8)   |    (8, 16, 32, or 48 bits, determined by ll)    | Delta Time (16) |
|01nullmm |                                                 |                 |
+---------+-------------------------------------------------+--------+--------+
     A             A + 1  ==>  A + N
+--------+----------------------------------------+              
| Number |             First Ack                  |
|Blocks-1|           Block Length                 |
| (opt)  |(8, 16, 32 or 48 bits, determined by mm)|
+--------+----------------------------------------+
  A + N + 1                A + N + 2  ==>  T(aka A + 2N + 1)
+------------+-------------------------------------------------+
| Gap to next|              Ack Block Length                   |
| Block (8)  |   (8, 16, 32, or 48 bits, determined by mm)     |
| (Repeats)  |       (repeats Number Ranges times)             |
+------------+-------------------------------------------------+
     T        T+1             T+2                 (Repeated Num Timestamps)
+----------+--------+---------------------+ ...  --------+------------------+  
|   Num    | Delta  |     Time Since      |     | Delta  |       Time       |
|Timestamps|Largest |    Largest Acked    |     |Largest |  Since Previous  |
|   (8)    | Acked  |      (32 bits)      |     | Acked  |Timestamp(16 bits)|
+----------+--------+---------------------+     +--------+------------------+
---
```
ACK帧中的字段如下：
* **帧类型：**帧类型字节是一个8位的值，其中包含了多种标记（01nullmmB）。
   * 开始的两位必须被设置为 01，以表明这是一个ACK帧。
   * 'n' 位表明帧是否有多于 1 个ack帧。
   * 'u' 位未使用。
   * 两个'll'位编码最大已观察字段的长度为1，2，4，或者 6字节长。
   * 两个'mm'位编码丢失包序列号差值字段的长度为 1，2，4，或者 6字节长。
* **最大已确认(Largest Acked)：**一个大小可变的无符号值，表示对端已观察到的最大的包号。
* **最大已确认差值时间(Largest Acked Delta Time)：**一个16 位的无符号浮点数，其中11个显式的位为底数，5位的显式指数，描述了从最大已确认包收到到这个Ack帧发送之间经过的微秒数。位格式近似于以IEEE 754建模。比如，1 微秒用 0x1表示，它的指数为0，在高5位中表示，底数为1，在低 11 位中表示。当显式的指数大于0的时候，则假设底数包含一个隐式的高阶12位的1。比如，浮点值 0x800 有一个显式的指数1，同时有一个显式的底数0，但之后有一个有效的底数 4096（假设12位为1）。此外，实际的指数比显式的指数小1，值表示4096微秒。任何大于可表示范围的值限定为 0xFFFF。
* **Ack 块段 (Ack Block Section)：**
   * **块个数 (Num Blocks)**：一个可选的8位无符号值描述了ack块的个数减一。只有在 'n' 标记位为 1 时才有。
   * **Ack块长度 (Ack block length)**：一个大小可变包号差值。对于第一个丢失包范围，ack块以最大已确认包开始。对于首个ack块，ack块的长度为 1 + 该值。对于后续的ack块，它是ack块的长度。对于非首个块，0值表示多于256个包丢失了。
   * **到下一块的间隙 (Gap to next block)**：一个8位的无符号值，描述了ack块之间包的个数。
* **时间戳段 (Timestamp Section)：**
   * **时间戳个数 (Num Timestamp)**：一个8位无符号值描述了包含在这个ack帧中的时间戳的个数。在后面的timestamps中将由许多的<packet number, timestamp>对。
   * **已观察最大差值 (Delta Largest Observed)**：一个8位无符号值描述了首个时间戳和最大已观察包之间包号的差值。然而，包号为最大已观察包号 减去 已观察最大差值 (delta largest observed)。
   * **首个时间戳 (First Timestamp)**：一个32位无符号值描述自由最大已观察包号描述的包的到达的连接的开始，减去 已观察最大差值，所得到的时间差值的微秒数。
   * **已观察最大差值（重复）(Delta Largest Observed(Repeated))**：（同上。）
   * **自前一个时间戳的时间（重复） (Time Since Previous Timestamp (Repeated))**：一个16位的无符号值描述了与前一个时间戳的差值。它的编码格式与 Ack Delay Time相同。

## **STOP_WAITING 帧**
STOP_WAITING 帧用于通知对端，它不应该继续等待包号小于特定值的包。包号以1，2，4或6字节编码，using the same coding length as is specified for the packet number for the enclosing packet's header (specified in the QUIC Frame Packet's Public Flags field.) 这个帧如下:
```
--- src
     0        1        2        3         4       5       6  
+--------+--------+--------+--------+--------+-------+-------+
|Type (8)|   Least unacked delta (8, 16, 32, or 48 bits)     |
|        |                       (variable length)           |
+--------+--------+--------+--------+--------+--------+------+
---
```
STOP_WAITING帧中的字段如下：
* **帧类型：**帧类型是一个8位的值，它必须被设置为0x06以表明这是一个STOP_WAITING帧。 

* **最小未确认差值：**一个可变长度的包号差值，与包首部的包号长度相同。将它从头部的包号减去以确定最小的未确认包。结果的最小未确认包是发送者依然在等待确认的包号最小的包。如果接收者丢失了任何比这个值小的包，接收者应该将那些包认做无可挽回的丢失。

## **WINDOW_UPDATE 帧**

WINDOW_UPDATE 帧用于通知对端一个端点的流量控制接收窗口的增长。流ID可以是0，表示这个WINDOW_UPDATE应用于连接级的流量控制窗口，或者 > 0 表示指定的流应该增长它的流量控制窗口。帧如下：

指定一个完全的字节偏移量，WINDOW_UPDATE帧的接收者可以只在那个流上至多发送那个字节数。发送更多字节而违背流量控制将导致接收端关闭连接。

为特定流ID收到多个WINDOW_UPDATE帧时，只需要追踪最大的字节偏移即可。

流和会话窗口都以一个默认值16KB开始，但是这个值典型地在握手期间增长。为了做到这一点，端点应该在握手中协商 SFCW (Stream Flow Control Window) 和 CFCW (Connection/Session Flow Control Window) 参数。与每个标记关联的值应该分别是初始流窗口和初始连接窗口的字节数。

帧如下：
```
--- src
    0         1                 4        5                 12
+--------+--------+-- ... --+-------+--------+-- ... --+-------+
|Type(8) |    Stream ID (32 bits)   |  Byte offset (64 bits)   | 
+--------+--------+-- ... --+-------+--------+-- ... --+-------+
---
```
WINDOW_UPDATE帧中的字段如下：
* **帧类型：**帧类型是一个8位值，它必须被设置为0x04以表示这是一个WINDOW_UPDATE帧。

* **流 ID：**要更新流控制窗口的流的ID，或者为0来描述连接级的流控制窗口。

* **字节偏移：** 一个64位无符号整型值，表示在给定的流上可以发送的数据的完整字节偏移量。在连接级流量控制的情况下，是在当前所有打开的流上可以发送的字节的总和。

## **BLOCKED 帧**
BLOCKED帧用于向远端指明本端点已经准备好发送数据了（且有数据要发送），但是当前被流量控制阻塞了。这是一个纯粹的信息帧，它对于调试极其有用。BLOCKED帧的接收者应该简单的丢弃它（可能在打印了一条有帮助的log消息之后）。帧如下：
```
--- src
     0        1        2        3         4
+--------+--------+--------+--------+--------+
|Type(8) |          Stream ID (32 bits)      |  
+--------+--------+--------+--------+--------+
---
```
BLOCKED帧中的字段如下：
* **帧类型：**帧类型是一个8位值，它必须被设置为0x05以表示这是一个BLOCKED帧。

* **流 ID：**一个32位的无符号数，表示流量控制阻塞的流。非零 流 ID 字段描述了被流量控制阻塞的流。当这个值为0时，流 ID字段在连接级指明连接被流量控制阻塞了。

## **CONGESTION_FEEDBACK 帧**
The CONGESTION_FEEDBACK frame is an experimental frame currently not used. It is intended to provide extra congestion feedback information outside the scope of the standard ack frame. A CONGESTION_FEEDBACK frame must have the first three bits of the Frame Type set to 001. The last 5 bits of the Frame Type field are reserved for future use.

## **PADDING 帧**
The PADDING frame pads a packet with 0x00 bytes. When this frame is encountered, the rest of the packet is expected to be padding bytes. The frame contains 0x00 bytes and extends to the end of the QUIC packet. A PADDING frame only has a Frame Type field, and must have the 8-bit Frame Type field set to 0x00.

## **RST_STREAM 帧**
The RST_STREAM frame allows for abnormal termination of a stream. When sent by the creator of a stream, it indicates the creator wishes to cancel the stream. When sent by the receiver of a stream, it indicates an error or that the receiver did not want to accept the stream, so the stream should be closed. The frame is as follows:
```
--- src
     0        1            4      5              12     8             16
+-------+--------+-- ... ----+--------+-- ... ------+-------+-- ... ------+
|Type(8)| StreamID (32 bits) | Byte offset (64 bits)| Error code (32 bits)|
+-------+--------+-- ... ----+--------+-- ... ------+-------+-- ... ------+
---
```
The fields in a RST_STREAM frame are as follows:
* **Frame type:** The Frame Type is an 8-bit value that must be set to 0x01 specifying that this is a RST_STREAM frame.

* **Stream ID:** The 32-bit Stream ID of the stream being terminated.

* **Byte offset:** A 64-bit unsigned integer indicating the absolute byte offset of the end of data for this stream.

* **Error code:** A 32-bit QuicErrorCode which indicates why the stream is being closed. QuicErrorCodes are listed later in this document.

## **PING  帧**
The PING frame can be used by an endpoint to verify that a peer is still alive. The PING frame contains no payload. The receiver of a PING frame simply needs to ACK the packet containing this frame. The PING frame should be used to keep a connection alive when a stream is open. The default is to do this after 15 seconds of quiescence, which is much shorter than most NATs time out. A PING frame only has a Frame Type field, and must have the 8-bit Frame Type field set to 0x07.**

## **CONNECTION_CLOSE 帧**
The CONNECTION_CLOSE frame allows for notification that the connection is being closed. If there are streams in flight, those streams are all implicitly closed when the connection is closed. (Ideally, a GOAWAY frame would be sent with enough time that all streams are torn down.) The frame is as follows:**
```
--- src
     0        1             4        5        6       7       
+--------+--------+-- ... -----+--------+--------+--------+----- ...
|Type(8) | Error code (32 bits)| Reason phrase   |  Reason phrase  
|        |                     | length (16 bits)|(variable length)
+--------+--------+-- ... -----+--------+--------+--------+----- ...
---
```
The fields of a CONNECTION_CLOSE frame are as follows:
* **Frame Type:** An 8-bit value that must be set to 0x02 specifying that this is a CONNECTION_CLOSE frame.

* **Error Code:** A 32-bit field containing the QuicErrorCode which indicates the reason for closing this connection.

* **Reason Phrase Length:** A 16-bit unsigned number specifying the length of the reason phrase. This may be zero if the sender chooses to not give details beyond the QuicErrorCode.

* **Reason Phrase:** An optional human-readable explanation for why the connection was closed.**

## **GOAWAY 帧**
The GOAWAY frame allows for notification that the connection should stop being used, and will likely be aborted in the future. Any active streams will continue to be processed, but the sender of the GOAWAY will not initiate any additional streams, and will not accept any new streams. The frame is as follows:**
```
--- src
     0        1             4      5       6       7      8
+--------+--------+-- ... -----+-------+-------+-------+------+
|Type(8) | Error code (32 bits)| Last Good Stream ID (32 bits)| ->
+--------+--------+-- ... -----+-------+-------+-------+------+
      9        10       11  
+--------+--------+--------+----- ... 
| Reason phrase   |  Reason phrase
| length (16 bits)|(variable length)
+--------+--------+--------+----- ...
---
```
The fields of a GOAWAY frame are as follows:
* **Frame type:** An 8-bit value that must be set to 0x03 specifying that this is a GOAWAY frame.

* **Error Code:** A 32-bit field containing the QuicErrorCode which indicates the reason for closing this connection. 

* **Last Good Stream ID:** The last Stream ID which was accepted by the sender of the GOAWAY message. If no streams were replied to, this value must be set to 0.

* **Reason Phrase Length:** A 16-bit unsigned number specifying the length of the reason phrase. This may be zero if the sender chooses to not give details beyond the error code.

* **Reason Phrase:** An optional human-readable explanation for why the connection was closed.**

# QUIC传输参数
The handshake is responsible for negotiating a variety of transport parameters for a QUIC connection.

## **Required Parameters**
* SFCW - Stream Flow Control Window. The size in bytes of the stream level flow control window.

* CFCW - Connection Flow Control Window. The size in bytes of the connection level flow control window.**

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit)
