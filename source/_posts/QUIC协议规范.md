---
title: QUIC协议规范
date: 2017-01-13 18:35:49
tags: 
- QUIC
- 网络
---

# 介绍
QUIC (Quick UDP Internet Connection，快速UDP互联网连接) 是一个新的基于UDP的多路复用和安全的传输协议，它是为HTTP/2语义而设计并专门优化的。以HTTP/2作为主要的应用协议构建，QUIC基于传输和安全领域数十年的经验而构建，并实现了使它成为有吸引力的现代通用传输协议的机制。QUIC提供了相当于HTTP/2的多路复用和流控，相当于TLS的安全性，及相当于TCP的连接语义、可靠性和拥塞控制。

<!--more-->

QUIC完全运行于用户空间，它当前被作为Chromium浏览器的一部分发布给用户，以便于快速的部署和实验。作为基于UDP的用户空间传输协议，QUIC可以做一些由于传统的客户端和中间设备的阻碍，或旷日持久的操作系统开发和部署周期，而被证明很难部署在现有的协议中的创新。

QUIC的一个重要目标是通过快速的实验来获得更好的传输设计。因此，我们希望将其中的一些精华的改动迁移进TCP和TLS，后者通常都有着长得多的迭代周期。

这份文档描述QUIC协议标准化前的概念设计和协议规范。补充资料描述加密和传输握手[QUIC-CRYPTO]，及丢失恢复和拥塞控制[draft-iyengar-quic-loss-recovery]。其它资源，包括一份更详细的相关文档，可以在[Chromium 的QUIC主页](https://www.chromium.org/quic)找到。

基于早期的部署的QUIC 标准化建议为[draft-hamilton-quic-transport-protocol]，[draft-shade-quic-http2-mapping]，[draft-iyengar-quic-loss-recovery], and [draft-thomson-quic-tls]。

# 术语和定义
QUIC中使用的所有整型值，包括长度、版本号和类型，都是小尾端字节序的，而不是网络字节序。QUIC不强制动态大小的帧中的类型对齐。

本文档中使用的一些术语定义如下。
* “客户端”：初始化QUIC连接的一端。
* “服务器”：接受进入的QUIC连接的一端。
* “端点”：连接的客户端或服务器端。
* “流”：QUIC链接中穿过一个逻辑通道的双向字节流。
* “连接”：两个QUIC端点之间具有一个加密上下文包含多路复用流的会话。
* “连接ID”：一个QUIC连接的标识符。
* “QUIC包”：一个经过良好格式化的UDP载荷，它可由QUIC的接收者解析。本文档中的QUIC包大小指UDP载荷大小。

# QUIC概述

我们现在清晰地描述QUIC的重要机制和优势。QUIC在功能上等价于TCP + TLS + HTTP/2，但基于UDP实现。QUIC相对于TCP + TLS + HTTP/2的主要优势包括
* 连接建立延迟
* 灵活的拥塞控制
* 多路复用而不存在队首阻塞
* 认证和加密的首部和载荷
* 流和连接的流量控制
* 连接迁移

## 连接建立延迟
QUIC将加密和传输的握手结合在一起，减少了建立一个安全的连接所需的往返。QUIC连接通常是0-RTT的，意味着相比于TCP + TLS中发送应用数据之前需要1-3个往返的情况不同，在大多数QUIC连接中，数据可以被立即发送而无需等待服务器的相应。

QUIC提供了一个专门的流（流ID为1）用于执行握手，但握手协议的详细内容不在本文当的范围内，要查看当前握手协议的完整描述，请参考 [QUIC Crypto Handshake](http://goo.gl/jOvOQ5) 文档。QUIC当前的握手将被未来的TLS 1.3替代。

## 灵活的拥塞控制

相对于TCP，QUIC具有可插入的拥塞控制和更丰富的信令，这使得QUIC相对于TCP可以提供更丰富的信息用于拥塞控制算法。当前，默认的拥塞控制是TCP Cubic算法的重实现；我们目前在实验替代的方法。

更丰富的信息的一个例子是，每个包，包括原始的和重传的，携带一个新的包序列号。这使得QUIC的发送端可以区分重传包的ACKs和原始包的ACKs，这样可以避免TCP的重传的歧义性问题。QUIC ACKs也显式地携带数据包的接收与它的确认被发送之间的延迟，与单调递增的包号一起，这使得QUIC可以精确的计算往返时间（RTT）。

最后，QUIC的ACK帧最多支持256个ack块，因此QUIC相对于TCP（使用SACK）在重排序时更有弹性，这也使得再重排序或丢失出现时，QUIC可以追踪更多在途字节。客户端和服务器端都可以更精确的了解到哪些包对端已经接收。

## 流和连接的流量控制

QUIC实现了流级和连接级的流量控制，接近于HTTP/2的流量控制。QUIC的流级流量控制像下面这样工作。QUIC接收者将每个流中接收者最多想要接收的数据的完整字节偏移发送给发送者。在发送，接收及在一个特定的流上传送数据时，接收者发送WINDOW_UPDATE帧来增大广告的流的偏移量限制，以允许对端在那个流上发送更多的数据。

除了每个流的流量控制，QUIC实现了连接级的流量控制来限制QUIC接收者想要为一个连接分配的总的缓冲区大小。连接的流量控制的工作方式与流的流量控制一样，但传送的字节和最大的接收偏移是所有流的总和。

与TCP的接收窗口自动调整类似，QUIC实现了流和连接流量控制器的流量控制积分的自动调整。如果WINDOW_UPDATE帧将限制发送者的速率，QUIC的自动调整增加发送的每个WINDOW_UPDATE帧的积分的大小，并当接收应用比较慢时限制发送者。

## 多路复用

基于TCP的HTTP/2深受TCP的队首阻塞问题的困扰。由于HTTP/2在TCP的单个字节流抽象之上多路复用许多流，一个TCP片段的丢失将导致所有后续片段的阻塞直到重传的数据到达，而封装在后续的片段中的HTTP/2流可能和丢失的片段毫无关系。

由于QUIC为多路复用操作而设计，携带了一个单独的流的包丢失时，通常只影响特定的流。每个流的帧在到达时可以被立即分发给那个流，因此没有丢失数据的流可以继续被重新汇集，并在应用程序中继续进行。

警告：QUIC当前通过HTTP/2 HPACK首部压缩来压缩一个专门的首部流的HTTP首部(3)，则在首部帧的传输中还是会有队首阻塞问题。

## 认证和加密的首部和载荷

TCP首部在网络中以明文出现，它没有认证，导致了大量的TCP注入和首部管理问题，比如接收窗口管理和序列号覆写。尽管这些问题中的一些是主动攻击，其它是一些由网络中的中间节点用来尝试透明的提升TCP性能的机制。然而，即使 “性能增强”中间设备依然有效地限制着传输协议的发展，这已经在MPTCP的设计及其后续的部署问题中观察到。

QUIC数据包总是认证的，而且典型地载荷是完全加密的。数据包头部不加密的部分依然会被接收者认证，以阻止任何数据包注入或第三方操作。QUIC阻止了连接被监听或中间设备的端到端通信的管理。

警告：PUBLIC _RESET 包复位一个当前未认证的连接。

## 连接迁移

TCP连接由源地址，源端口，目标地址和目标端口的4元组标识。TCP一个广为人知的问题是，IP地址改变（比如，由WiFi网络切换到了移动网络）或端口号改变（当客户端的NAT绑定超时导致服务器看到的端口号改变）。尽管MPTCP解决了TCP的连接迁移问题，它依然被缺少中间设备和OS部署支持而困扰。

QUIC连接由一个64-bit连接ID标识，它由客户端随机地产生。在IP地址改变和NAT重绑定时，QUIC连接可以不断开，因为连接ID在迁移过程中保持不变。QUIC还提供了迁移的客户端的自动的加密验证，由于一个迁移的客户端继续使用相同的session key来加密和解密数据包。

当连接用明确的4元组标识时，比如当服务器使用一个短暂的端口给客户端发送数据包时，有一个选项来不发送连接ID以保存线上数据。

# 包类型和格式
QUIC具有特殊包和普通包。有两种类型特殊包：版本协商包(Version Negotiation Packets) 和 公共复位包 (Public Reset Packets)，普通包包含帧。

所有QUIC包的大小应该适配路径的MTU以避免IP分片。路径MTU发现是正在进行中的工作，当前的QUIC实现为IPv6使用1350字节的最大QUIC包大小，为IPv4使用1370字节。两个大小都没有IP和UDP过载。

## QUIC公共包头
传输的所有QUIC包以一个大小介于1至51个字节之间的公共包头开始。公共包头的格式如下：

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
载荷可以包含多个如下所述依赖类型的头部字节。
公共头部中的字段如下：
* **Public Flags**：
  * 0x01 = PUBLIC_FLAG_VERSION。这个标记的解释依赖于包是由服务器还是客户端发送的。当由客户端发送时，设置它来指示头部包含一个QUIC版本(参考下面的说明)。客户端必须在所有的包中设置这个位，直到客户端收到服务器确认同意由客户端提议的版本号为止。服务器通过发送不设置该位的包来表示同意一个版本。当这个位由服务器设置时，包是一个版本协商包。版本协商将在后面做更详细地描述。
  * 0x02 = PUBLIC_FLAG_RESET。设置来指示包是一个公共复位包。
  * 0x04 = 指明头部中包含一个32字节的多元化现时标志。
  * 0x08 = 指明包中包含完整的8字节连接ID。必须为所有的包设置这个位，直到为给定的方向协商一个不同的值 (比如，客户端可能请求包含更少字节的连接ID)。
  * 0x30 处的两个位指明每个包中包含包号的低阶字节的个数。这些位只用于帧包。公共复位 和 版本协商包 (由服务器发送) 没有包号，则不使用这些位，且它们必须被设置为0。掩码这2位：
    * 0x30 指明包号占用6个字节。
    * 0x20 指明包号占用4个字节。
    * 0x10 指明包号占用2个字节。
    * 0x00 指明包号占用1个字节。
  * 0x40 保留给多路径使用。
  * 0x80 当前未使用，且必须被设置为0。
* **连接ID**：这是一个由客户端选择的无符号64位静态随机数，它用作连接的标识符。由于QUIC的连接被设计为即使客户端漫游了依然保持连接状态，IP 4元组（源IP，源端口，目标IP，目标端口）可能不足以标识连接。对每个传输方向，当4元组足够用以标识连接时，连接ID可以被省略。
* **QUIC版本**：一个32位的不透明的标记，表示QUIC协议的版本。只有在公共标记包含 FLAG_VERSION 时才会出现（比如 public_flags & FLAG_VERSION !=0）。客户端可以设置这个标记，并 **正好** 包含一个提议的版本，同时包含任意的数据（与那个版本一致）。当客户端提议的版本不支持时服务器可以设置这个标记，然后可以提供一个可接受的版本的列表（0或多个），但 **一定不能(MUST not)** 在版本信息之后包含任何数据。在最近的实验版本中版本值的示例包括 "Q025"，它对应于byte 9包含'Q"，byte 10包含 '0"，等等。[参考本文当末尾的不同版本变化列表。]
* **包号**：包号的低8，16，32，或48位，依赖于公共标记中的 FLAG_?BYTE_SEQUENCE_NUMBER 标记被设置为什么。每个普通包（与特别的公共复位包和版本协商包相反）由发送者分配包号。由某一端发送的第一个包的包号应该为1，后续每个包的包号应该比前一个包的包号大1。

包号的低64位被用作加密随机数的一部分；然而，一个QUIC端点一定不能发送一个包，其包号不能用64位表示。如果QUIC端点传输了一个包号为 (2^64-1) 的包，则那个包必须包含一个 包含了QUIC_SEQUENCE_NUMBER_LIMIT_REACHED错误码的CONNECTION_CLOSE 帧，对端一定不能再传输任何其它的包了。

至多传输包号的低48位。要使接收者可以明确的重建包号，QUIC端点一定不能传输一个包号比接收者已传输的已知已确认最大的包号大(2^(bitlength-2)) 的包号。然热，在途包的数目不能超过 (2^46)。

任何截断的包号应该被推断为具有一个与对端的最大已知包号且大于它的最接近的值，其传输的包最初包含了截断的包号。包号已传送的部分匹配推断的值的最低位。

一个公共标记处理流程图如下：
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
           Regular Packet         Regular Packet with                               QUIC Version present in header
---
```
## 特殊包
### *版本协商包*
只有服务器会发送版本协商包。版本协商包以一个8位的公共标记和64位的连接ID开始。公共标记必须设置PUBLIC_FLAG_VERSION，并 指示64位的连接ID。版本协商包的其余部分是一个服务器支持的4字节版本的列表：
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
公共复位包以一个8位的公共标记和64位的连接ID开始。公共标记必须设置 PUBLIC_FLAG_RESET，并指示64位的连接ID。公共复位包的其余部分犹如标记PRST的一个加密握手消息那样编码（参考[QUIC-CRYPTO]）：
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
* CADR (client address) - 发现的客户端IP地址和端口号。它当前只被用于调试目的，因而是可选的。

(TODO：公共复位包应该包含认证的（目标）服务器IP/端口号。)

## 普通包
普通包已认证且加密。公共头部已认证但未加密，从第一个帧开始的包的其余部分已加密。紧跟再公共头部之后，普通包包含AEAD（authenticated encryption and associated data）数据。这些数据在被作为内容解释之前一定要先解密。解密之后，明文由一系列的帧组成。

(TODO: Document the inputs to encryption and decryption and describe trial decryption.)

### *帧包*
帧包具有一个载荷，它是一系列的类型前缀帧。帧类型的格式将在本文档的后面定义，但帧包的通用格式如下：
```
--- src
+--------+---...---+--------+---...---+
| Type   | Payload | Type   | Payload |
+--------+---...---+--------+---...---+
---
```

# QUIC连接的生命
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
