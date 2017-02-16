---
title: HPACK：HTTP/2的首部压缩 (RFC7541)
date: 2016-10-29 13:46:49
categories: 网络协议
tags:
- 网络协议
- HTTP2
---

# 摘要

这份规范定义了HPACK，一个用于HTTP/2，为有效地表示HTTP首部字段而设计的压缩格式。

# 此备忘录的状态

这是一份互联网标准跟踪文档。

<!--more-->

本文档是互联网工程任务组（IETF）的产品。它代表了IETF社区的共识。它已经接受公众审查，并已获得互联网工程指导小组（IESG）的发布允准。有关互联网标准的更多信息，请参见[RFC 5741的第2节](https://tools.ietf.org/html/rfc5741#section-2)。

有关本文档的当前状态，任何勘误以及如何提供反馈的信息，请访问 http://www.rfc-editor.org/info/rfc7541。

# 版权声明

版权所有©2015 IETF信托和被确定为文档作者的人。版权所有。

本文件受BCP 78和IETF信托有关IETF文件的法律规定（[http://trustee.ietf.org/license-info](http://trustee.ietf.org/license-info)）对本文件出版日期的影响。请仔细阅读这些文档，它们描述了您对本文档的权利和限制。从本文档中提取的代码组件必须包括如信托法律规定第4.e节所述的简化BSD许可文本，并且不提供简化BSD许可中描述的保证。

# 目录

## [1. 引言](https://tools.ietf.org/html/rfc7541#section-1)
### [1.1 概述](https://tools.ietf.org/html/rfc7541#section-1.1)
### [1.2 约定](https://tools.ietf.org/html/rfc7541#section-1.2)
### [1.3 术语](https://tools.ietf.org/html/rfc7541#section-1.3)

## [2. 压缩过程概述](https://tools.ietf.org/html/rfc7541#section-2)
### [2.1 头部列表排序](https://tools.ietf.org/html/rfc7541#section-2.1)
### [2.2 编码和解码上下文](https://tools.ietf.org/html/rfc7541#section-2.2)
### [2.3 索引表](https://tools.ietf.org/html/rfc7541#section-2.3)
#### [2.3.1 静态表](https://tools.ietf.org/html/rfc7541#section-2.3.1)
#### [2.3.2 动态表](https://tools.ietf.org/html/rfc7541#section-2.3.2)
#### [2.3.3 索引地址空间](https://tools.ietf.org/html/rfc7541#section-2.3.3)
### [2.4 头部字段表示](https://tools.ietf.org/html/rfc7541#section-2.4)

## [3. 头部块解码](https://tools.ietf.org/html/rfc7541#section-3)
### [3.1 头部块处理](https://tools.ietf.org/html/rfc7541#section-3.1)
### [3.2 头部字段表示处理](https://tools.ietf.org/html/rfc7541#section-3.2)

## [4. 动态表管理](https://tools.ietf.org/html/rfc7541#section-4)
### [4.1 计算表大小](https://tools.ietf.org/html/rfc7541#section-4.1)
### [4.2 最大表大小](https://tools.ietf.org/html/rfc7541#section-4.2)
### [4.3 动态表大小改变时的条目逐出](https://tools.ietf.org/html/rfc7541#section-4.3)
### [4.4 添加新条目时的条目逐出](https://tools.ietf.org/html/rfc7541#section-4.4)

## [5. 原始数据类型表示](https://tools.ietf.org/html/rfc7541#section-5)
### [5.1 整数的表示](https://tools.ietf.org/html/rfc7541#section-5.1)
### [5.2 字符串字面量的表示](https://tools.ietf.org/html/rfc7541#section-5.2)

## [6. 二进制格式](https://tools.ietf.org/html/rfc7541#section-6)
### [6.1 索引的头部字段表示](https://tools.ietf.org/html/rfc7541#section-6.1)
### [6.2 字面量头部字段表示](https://tools.ietf.org/html/rfc7541#section-6.2)
#### [6.2.1 增量索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.1)
#### [6.2.2 无索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.2)
#### [6.2.3 从不索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.3)
### [6.3 动态表 大小更新](https://tools.ietf.org/html/rfc7541#section-6.3)

## [7. 安全注意事项](https://tools.ietf.org/html/rfc7541#section-7)
### [7.1 探测动态表状态](https://tools.ietf.org/html/rfc7541#section-7.1)
#### [7.1.1 HPACK 和 HTTP的适用性](https://tools.ietf.org/html/rfc7541#section-7.1.1)
#### [7.1.2 缓解](https://tools.ietf.org/html/rfc7541#section-7.1.2)
#### [7.1.3 从不索引的字面量](https://tools.ietf.org/html/rfc7541#section-7.1.3)
### [7.2 静态Huffman编码](https://tools.ietf.org/html/rfc7541#section-7.2)
### [7.3 内存消耗](https://tools.ietf.org/html/rfc7541#section-7.3)
### [7.4 实现的限制](https://tools.ietf.org/html/rfc7541#section-7.4)

## [8. 参考文献](https://tools.ietf.org/html/rfc7541#section-8)
### [8.1 规范性参考文献](https://tools.ietf.org/html/rfc7541#section-8.1)
### [8.2 参考资料](https://tools.ietf.org/html/rfc7541#section-8.2)

## [附录 A. 静态表定义](https://tools.ietf.org/html/rfc7541#appendix-A)

## [附录 B. Huffman 码](https://tools.ietf.org/html/rfc7541#appendix-B)

## [附录 C. 示例](https://tools.ietf.org/html/rfc7541#appendix-C)
### [C.1 整数表示示例](https://tools.ietf.org/html/rfc7541#appendix-C.1)
#### [C.1.1 示例 1：使用5位前缀编码10](https://tools.ietf.org/html/rfc7541#appendix-C.1.1)
#### [C.1.2 示例 2：使用5位前缀编码1337](https://tools.ietf.org/html/rfc7541#appendix-C.1.2)
#### [C.1.3 示例 3: 以字节边界为起始位置编码42](https://tools.ietf.org/html/rfc7541#appendix-C.1.3)
### [C.2 头部字段表示示例](https://tools.ietf.org/html/rfc7541#appendix-C.2)
#### [C.2.1 有索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#appendix-C.2.1)
#### [C.2.2 无索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#appendix-C.2.2)
#### [C.2.3 从不索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#appendix-C.2.3)
#### [C.2.4 索引的头部字段](https://tools.ietf.org/html/rfc7541#appendix-C.2.4)
### [C.3 不带Huffman编码的请求示例](https://tools.ietf.org/html/rfc7541#appendix-C.3)
#### [C.3.1 第一个请求](https://tools.ietf.org/html/rfc7541#appendix-C.3.1)
#### [C.3.2 第二个请求](https://tools.ietf.org/html/rfc7541#appendix-C.3.2)
#### [C.3.3 第三个请求](https://tools.ietf.org/html/rfc7541#appendix-C.3.3)
### [C.4 带Huffman编码的请求示例](https://tools.ietf.org/html/rfc7541#appendix-C.4)
#### [C.4.1 第一个请求](https://tools.ietf.org/html/rfc7541#appendix-C.4.1)
#### [C.4.2 第二个请求](https://tools.ietf.org/html/rfc7541#appendix-C.4.2)
#### [C.4.3 第三个请求](https://tools.ietf.org/html/rfc7541#appendix-C.4.3)
### [C.5 不带Huffman编码的响应示例](https://tools.ietf.org/html/rfc7541#appendix-C.5)
#### [C.5.1 第一个响应](https://tools.ietf.org/html/rfc7541#appendix-C.5.1)
#### [C.5.2 第二个响应](https://tools.ietf.org/html/rfc7541#appendix-C.5.2)
#### [C.5.3 第三个响应](https://tools.ietf.org/html/rfc7541#appendix-C.5.3)
### [C.6 带Huffman编码的响应示例](https://tools.ietf.org/html/rfc7541#appendix-C.6)
#### [C.6.1 第一个响应](https://tools.ietf.org/html/rfc7541#appendix-C.6.1)
#### [C.6.2 第二个响应](https://tools.ietf.org/html/rfc7541#appendix-C.6.2)
#### [C.6.3 第三个响应](https://tools.ietf.org/html/rfc7541#appendix-C.6.3)

## 致谢

## 作者地址

# [1. 简介](https://tools.ietf.org/html/rfc7541#section-1)

在 HTTTP/1.1中 (见 [[RFC7230](https://tools.ietf.org/html/rfc7230)])， 首部字段是不压缩的。随着web页面逐步成长到需要发出几十甚至几百个请求，这些请求中冗余的首部字段不必要地消耗了带宽，并导致了可测量的延迟增加。

[SPDY](https://tools.ietf.org/html/rfc7541#ref-SPDY) [SPDY] 最初通过使用 [DEFLATE](https://tools.ietf.org/html/rfc7541#ref-DEFLATE) [DEFLATE] 格式压缩头部字段来解决这种冗余，这在有效地表示冗余的首部字段方面非常有效。然而，该方法暴露了CRIME (Compression Ratio Info-leak Made Easy) 攻击 (参见 [[CRIME]](https://tools.ietf.org/html/rfc7541#ref-CRIME)) 所示的安全风险。

本规范定义了HPACK，它是一种新的压缩器，它消除了冗余的首部字段，限制了对已知安全攻击的脆弱性，并却具有在受限环境中使用的有限内存要求。[第 7 节](https://tools.ietf.org/html/rfc7541#section-7)介绍了HPACK的潜在安全问题。

HPACK格式有意地被设计的简单而不灵活。这两个特性降低了由于实现错误而导致互操作性和安全性问题的风险。没有定义可扩展性机制；只能通过定义完全的替代品来更改格式。

## [1.1 概述](https://tools.ietf.org/html/rfc7541#section-1.1)

本规范中定义的格式将首部字段列表视为可包含重复对的名-值对的有序集合。名字和值被认为是不透明的字节序列，并且头部字段的顺序在压缩和解压后被保留。

编码通过将头部字段映射到索引值的头部字段表来获知。这些头部字段表可以随着新头部字段的编码和解码而增量地更新。

在编码形式中，头部字段被字面地表示，或者作为对头部字段表之一中的头部字段。因此，可以使用引用和字面量值的混合来编码头部字段列表。

字面量值直接编码或使用一个静态Huffman码。

编码器负责决定将哪些头部字段作为新条目插入头部字段表。解码器执行由编码器规定的对头部字段表的修改，及重建头部字段列表的过程。这使解码器保持简单，并能与各种编码器的互操作。

[附录 C](https://tools.ietf.org/html/rfc7541#appendix-C) 中提供了说明使用这些不同机制表示头字段的示例。

## [1.2 约定](https://tools.ietf.org/html/rfc7541#section-1.2)

本文档中的关键字 "MUST"，"MUST NOT"，"REQUIRED"，"SHALL"，"SHALL NOT"，"SHOULD"，"SHOULD NOT"，"RECOMMENDED"，"MAY"，和 "OPTIONAL"以如[[RFC2119](https://tools.ietf.org/html/rfc2119)]中所描述的来解释。

所有的数值都是网络字节序。除非另有说明，否则值为无符号的。字面值以十进制或十六进制的形式提供。

## [1.3 术语](https://tools.ietf.org/html/rfc7541#section-1.3)

本规范使用了如下的术语：

**头部字段**：一个名-值对。名字和值都被视为不透明的字节序列。

**动态表**：动态表 (参见 [2.3.2节](https://tools.ietf.org/html/rfc7541#section-2.3.2)) 是将存储的**头部字段**和**索引值**关联起来的表。这个表是动态的，并且特定于编码或解码上下文。

**静态表**：静态表是 (参见 [2.3.1节](https://tools.ietf.org/html/rfc7541#section-2.3.1)) 是静态地将**频繁出现的头部字段**与**索引值**关联起来的表。这个表是有序的，只读的，总是可访问的，且可以在所有的编码或解码上下文间共享的。

**头部列表**：头部列表是一个联合编码的头部字段可出现重复的有序头部字段集合。一个HTTP/2 **头部块** 中包含的完整的 **头部字段列表** 是一个 **头部列表**。

**头部字段表示**：头部字段可以以字面量或索引的编码形式表示( 见 [2.4节](https://tools.ietf.org/html/rfc7541#section-2.4))。

**头部块**：头部字段表示的有序列表，当解码时，产生完整的头部列表。

# [2. 压缩过程概述](https://tools.ietf.org/html/rfc7541#section-2)

本规范没有描述用于编码器的特定算法。相反，它精确地定义了解码器如何操作，以允许编码器产生本定义允许的任何编码。

## [2.1 头部列表排序](https://tools.ietf.org/html/rfc7541#section-2.1)

HPACK保留头部列表中头部字段的顺序。编码器 **必须(MUST)** 根据 **头部字段** 在原始头部列表中的顺序在头部块中对 **头部字段表示** 进行排序。解码器 **必须(MUST)** 根据 **头部字段表示** 在头部块中的顺序在解码的头部列表中对 **头部字段** 进行排序。

## [2.2 编码和解码上下文](https://tools.ietf.org/html/rfc7541#section-2.2)

要解压缩 **头部块**，解码器只需维护 **动态表** (参见 [2.3.2节](https://tools.ietf.org/html/rfc7541#section-2.3.2)) 作为解码上下文。不需要其它的动态状态。

当被用于双向通信时，比如在HTTP中，编码和解码动态表完全独立地由一端维护，比如，请求和响应的动态表是分开的。

## [2.3 索引表](https://tools.ietf.org/html/rfc7541#section-2.3)

HPACK使用两个表将 **头部字段** 与 **索引** 关联起来。 **静态表** (参见 [2.3.1节](https://tools.ietf.org/html/rfc7541#section-2.3.1)) 是预定义的，它包含了常见的头部字段（其中的大多数值为空）。 **动态表** (参见 [2.3.2节](https://tools.ietf.org/html/rfc7541#section-2.3.2)) 是动态的，它可被编码器用于在编码的头部列表中重复地索引头部字段。

这两个表被组合成用于定义索引值的单个地址空间(参见 [2.3.3节](https://tools.ietf.org/html/rfc7541#section-2.3.3))。

### [2.3.1 静态表](https://tools.ietf.org/html/rfc7541#section-2.3.1)

静态表由一个预定义的头部字段的静态列表组成。它的条目在 [附录 A](https://tools.ietf.org/html/rfc7541#appendix-A) 中定义。

### [2.3.2 动态表](https://tools.ietf.org/html/rfc7541#section-2.3.2)

动态表由以先进先出顺序维护的 **头部字段列表** 组成。动态表中第一个且最新的条目索引值最低，动态表最旧的条目索引值最高。

动态表最初是空的。条目随着每个头部块的解压而添加。

动态表可以包含重复的条目 (比如，具有相同的名字和相同的值的条目)。因而，重复条目 **一定不能(MUST NOT)** 被解码器视为错误。

编码器决定如何更新动态表，并因此可以控制动态表使用多少内存。为了限制解码器的内存需求，动态表的大小被严格地进行限制(参见 [4.2节](https://tools.ietf.org/html/rfc7541#section-4.2))。

解码器在处理 **头部字段表示** (参见[3.2节](https://tools.ietf.org/html/rfc7541#section-3.2)) 列表的过程中更新动态表

### [2.3.3 索引地址空间](https://tools.ietf.org/html/rfc7541#section-2.3.3)

静态表和动态表被组合为单独的索引地址空间。

在1和静态表的长度(包含)之间的索引值指向静态表 (见第 [2.3.1节](https://tools.ietf.org/html/rfc7541#section-2.3.1)) 中的元素。

严格地大于静态表长度的索引值指向动态表 (见第 [2.3.2节](https://tools.ietf.org/html/rfc7541#section-2.3.2)) 中的元素。通过减去静态表的长度来查找指向动态表的索引。

严格地大于两表长度之和的索引值 **必须(MUST)** 被视为一个解码错误。

对于静态表大小为s，动态表大小为k的情况，下图展示了完整的有效索引地址空间。

```
        <----------  Index Address Space ---------->
        <-- Static  Table -->  <-- Dynamic Table -->
        +---+-----------+---+  +---+-----------+---+
        | 1 |    ...    | s |  |s+1|    ...    |s+k|
        +---+-----------+---+  +---+-----------+---+
                               ^                   |
                               |                   V
                        Insertion Point      Dropping Point
```
图 1：索引地址空间

## [2.4 头部字段表示](https://tools.ietf.org/html/rfc7541#section-2.4)

一个编码的头部字段可由一个索引或一个字面量表示。

索引的表示法将头部字段定义为对静态表或动态表中条目的引用(见第 [6.1节](https://tools.ietf.org/html/rfc7541#section-6.1))。

字面量的表示通过描述头部字段的名称和值来定义头部字段。头部字段的名称可被字面地表示，或表示为对静态表或动态表中条目的引用。头部字段的值由字面量表示。

定义了三种不同的字面的表示：

* 将头部字段作为新条目添加到动态表的起始位置的字面地表示(见第 [6.2.1节](https://tools.ietf.org/html/rfc7541#section-6.2.1))。
* 不向动态表添加 **头部字段** 的字面量表示(见第 [6.2.2节](https://tools.ietf.org/html/rfc7541#section-6.2.2))。
* 不向动态表添加头部字段的字面量表示，此外还规定这个头部字段总是使用字面的表示，特别是由一个中介重编码时 (见第 [6.2.3节](https://tools.ietf.org/html/rfc7541#section-6.2.3))。这种表示的目的是保护那些在压缩时有一定风险的头部字段值 (见第 [7.1.3节](https://tools.ietf.org/html/rfc7541#section-7.1.3) 来获取更多详细信息)。

可按照 **安全注意事项** 的指导来选择这三种字面的表示中的一个，以保护敏感的头部字段值 (见第 [7.1节](https://tools.ietf.org/html/rfc7541#section-7.1))。

头部字段的名字或值的字面表示的字节序列可以是直接地编码，或使用静态Huffman码 (见第 [5.2节](https://tools.ietf.org/html/rfc7541#section-5.2))。

# [3. 头部块解码](https://tools.ietf.org/html/rfc7541#section-3)

## [3.1 头部块处理](https://tools.ietf.org/html/rfc7541#section-3.1)

解码器顺序地处理头部块来重建原始的头部列表。

头部块是 **头部字段表示** 的连接。不同的可能的头部字段表示在第 [6节](https://http2.github.io/http2-spec/compression.html#detailed.format) 描述。

一旦头部字段被解码并加进了重建的头部列表，它就不能被移除了。被加进头部列表的头部字段可以被安全地传给应用程序。

通过将得到的头部字段传给应用程序，除了动态表所需的内存外，可以利用最小的临时存储器承诺来实现解码器。

## [3.2 头部字段表示处理](https://tools.ietf.org/html/rfc7541#section-3.2)

本节定义对头部块进行处理以获得头部列表。为了确保解码将成功地产生 **头部列表**，解码器 **必须(MUST)** 遵守以下规则。

包含在一个头部块中的所有头部字段表示以它们出现的顺序处理，如下所述。关于不同的头部字段表示的格式的细节和一些额外的处理指令参见 [第6节](https://tools.ietf.org/html/rfc7541#section-6)。

***索引的表示*** 需要执行以下操作：

* 静态表或动态表中被引用的条目对应的头部字段被加到解码的头部列表。

***不加进*** 动态表的 ***字面的表示*** 需要执行以下操作：

* 头部字段被加到解码的头部列表。

***加进*** 动态表的 ***字面的表示*** 需要执行以下操作：

* 头部字段被加到解码的头部列表。
* 头部字段被插入动态表的起始位置。这种插入可能导致动态表中之前的条目被逐出 (参见 [第 4.4节](https://tools.ietf.org/html/rfc7541#section-4.4))。

# [4. 动态表管理](https://tools.ietf.org/html/rfc7541#section-4)

为了限制解码器端的内存需求，动态表大小受到限制。

## [4.1 计算表大小](https://tools.ietf.org/html/rfc7541#section-4.1)

动态表大小是它的条目大小的总和。

条目大小是它名字的字节长度 ( 如 [第 5.2 节](https://tools.ietf.org/html/rfc7541#section-5.2) 所述) 及它值的字节长度的总和，外加32。

任何条目的大小使用未应用Huffman编码条件下，它的名字和值的长度来计算。

**注意**：额外的32字节是考虑到与一个条目关联的预估开销。比如，条目结构使用两个64位的指针引用条目的名字和值，及两个64位整数来计数引用了名字和值的引用数目，这将有32字节的开销。

## [4.2 最大表大小](https://tools.ietf.org/html/rfc7541#section-4.2)

使用HPACK的协议决定允许编码器使用的动态表最大大小。在HTTP/2中，这个值由SETTINGS_HEADER_TABLE_SIZE 设置项决定 (参见 [[HTTP2]](https://tools.ietf.org/html/rfc7541#ref-HTTP2) 的第 6.5.2节)。

编码器可以选择使用比此最大大小(参见 [第6.3节](https://tools.ietf.org/html/rfc7541#section-6.3))更小的容量，但选择的大小 **必须(MUST)** 小于等于协议设置的最大大小。

动态表最大大小的修改通过动态表大小更新(见 [第6.3节](https://tools.ietf.org/html/rfc7541#section-6.3))来通知。这种动态表大小更新 **必须(MUST)** 发生在遵循该动态表大小改变的第一个头部块的起始处。在HTTP/2中，这发生在设置项确认(参见 [[HTTP2]](https://tools.ietf.org/html/rfc7541#ref-HTTP2) 的第6.5.3节)之后。

两个头部块的传输之间可能发生多次对最大表大小的更新。对于在这个间隔内该大小改变超过一次的情况，那个间隔内发生的最小最大表大小 **必须(MUST)** 在一个动态表大小更新中通知。最终的最大大小总是用信号通知的，这导致至多两个动态表大小更新。这确保解码器能够基于减小的动态表大小来执行逐出(见 [第 4.3节](https://tools.ietf.org/html/rfc7541#section-4.3))。

通过将最大大小设置为0，此机制可被用于完全地清除动态表中的条目，随后可以恢复这些条目。

## [4.3 动态表大小改变时的条目逐出](https://tools.ietf.org/html/rfc7541#section-4.3)

无论何时动态表最大大小减小，将条目从动态表的结尾处逐出，直到动态表的大小小于或等于最大大小。

## [4.4 添加新条目时的条目逐出](https://tools.ietf.org/html/rfc7541#section-4.4)

在把新条目加到动态表之前，需要先从动态表的结尾处把条目逐出，直到动态表大小小于或等于(最大大小 - 新条目大小)，或直到表为空。

如果新条目小于等于最大大小，则将那个条目加进表里。尝试将大于最大大小的条目加进表里不是错误；尝试添加大于最大大小的条目导致所有已有条目被清空，并产生一个空表。

当把新条目加进动态表时，该新条目可以引用动态表中将被逐出的条目的名字。实现要注意，如果引用的条目在新条目插入之前被从动态表逐出了，则要避免删除引用的名字。

# [5. 原始数据类型的表示](https://tools.ietf.org/html/rfc7541#section-5)

HPACK编码使用两种原始数据类型：无符号的变长整数和字节串。

## [5.1 整数的表示](https://tools.ietf.org/html/rfc7541#section-5.1)

整数用于表示名字索引，头部字段索引或字符串长度。整数的表示可在字节内的任何位置开始。为了处理上的优化，整数的表示总是在字节的结尾处结束。

整数由两部分表示：填满当前字节的前缀，及前缀不足以表示整数时的一个可选字节列表。前缀的位数 (称为 N) 是整数表示过程的一个参数。

如果整数值足够小，比如，严格地小于2^N-1，则把它编码进N位前缀。
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | ? |       Value       |
+---+---+---+-------------------+
```
图 2：前缀中编码的整数值 (显示 N = 5)

否则，前缀的所有位被设置为1，值减去2^N-1，然后使用一个或多个字节的列表编码。每个字节的最高有效位被用作连续标记：除列表的最后一个字节外，该位的值都被设为1。字节中剩余的位被用于编码减小后的值。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | ? | 1   1   1   1   1 |
+---+---+---+-------------------+
| 1 |    Value-(2^N-1) LSB      |
+---+---------------------------+
               ...
+---+---------------------------+
| 0 |    Value-(2^N-1) MSB      |
+---+---------------------------+
```
图 3：前缀之后的整数值的编码(显示 N =  5)

要由字节列表解码整数值，首先需要将列表中的字节顺序反过来。然后，移除每个字节的最高有效位。连接字节的剩余位，再将结果加2^N-1获得整数值。

前缀大小，N，总是在1到8位之间。从字节边界开始编码的整数值其前缀为8位。

表示整数I的伪代码如下：
```
if I < 2^N - 1, encode I on N bits
else
    encode (2^N - 1) on N bits
    I = I - (2^N - 1)
    while I >= 128
         encode (I % 128 + 128) on 8 bits
         I = I / 128
    encode I on 8 bits
```
<font color=red>注意，在表示大整数时，字节序列中每个字节的最高有效位仅仅被用作了连续标记，它没有被用于表示整数值。(I % 128 + 128) 这个式子只是为了说明，字节的最高有效位要被设置为1。</font>

解码整数I的伪代码如下：
```
decode I from the next N bits
if I < 2^N - 1, return I
else
    M = 0
    repeat
        B = next octet
        I = I + (B & 127) * 2^M
        M = M + 7
    while B & 128 == 128
    return I
```

<font color=red>要如何判断I小于 (2^N - 1) 呢？ 可以根据下一个字节的最高有效位判断么？好像不行吧。</font>

说明了 **整数的编码** 的示例可参见[附录C.1](https://tools.ietf.org/html/rfc7541#appendix-C.1)。

这种整数表示法允许无限大的值。编码器还可能发送大量0值，这可以浪费字节，并且可能用于溢出整数值。超出实现限制的整数的编码——值或字节长度—— **必须(MUST)** 被视为解码错误。可以基于实现的限制，为整数的每个不同使用设置不同的限制。

<font color=red>浪费字节 是什么鬼？溢出整数值 又是要干什么？**整数的每个不同使用** 是神马意思？</font>

## [5.2 字符串的字面表示](https://tools.ietf.org/html/rfc7541#section-5.2)

头部字段名和头部字段值可使用字符串字面量表示。字符串字面量可通过直接编码其字节或使用Huffman码 (见 [[HUFFMAN]](https://tools.ietf.org/html/rfc7541#ref-HUFFMAN))编码为字节序列。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| H |    String Length (7+)     |
+---+---------------------------+
|  String Data (Length octets)  |
+-------------------------------+
```
图 4：字符串字面量表示

字符串字面量表示包含如下的字段：

**H：** 一位的标记，H，指示字符串的字节是否为Huffman编码。

**字符串长度：** 编码字符串字面量的字节数，用一个7位前缀的整数编码 (参见 [第5.1节](https://tools.ietf.org/html/rfc7541#section-5.1))。

**字符串数据：** 字符串字面量的编码数据。如果H是'0'，则编码数据是字符串字面量的原始字节。如果H是'1'，则编码数据是字符串字面量的Huffman编码。

以Huffman码编码的字符串字面量使用 [附录 B](https://tools.ietf.org/html/rfc7541#appendix-B) 中定义的Huffman码 ( 参见 [附录 C.4](https://tools.ietf.org/html/rfc7541#appendix-C.4) 中请求的例子和 [附录  C.6](https://tools.ietf.org/html/rfc7541#appendix-C.6) 中响应的例子)。编码的数据是字符串字面量每个字节对应的代码的位的连接。

由于Huffman编码数据不总是在字节边界处结束，因而要在它的后面插入一些填充，至下个字节的边界。为了防止这种填充被错误地解释为字符串字面量的一部分，而将代码的最高有效位用作 EOS (end-of-string) 符号。

一旦解码完成，编码数据结尾处不完整的代码被视为填充并丢弃。严格地长于7位的填充 **必须(MUST)** 被视为解码错误。包含EOS符号的Huffman编码字符串字面量 **必须(MUST)** 被视为解码错误。

<font color=red>所以EOS符号究竟是如何用的？Hufffman编码字符串字面的填充又是如何识别的？</font>

# [6. 二进制格式](https://tools.ietf.org/html/rfc7541#section-6)

本节描述每个不同头部字段表示和动态表大小更新指令的详细格式。

## [6.1 索引的头部字段表示](https://tools.ietf.org/html/rfc7541#section-6.1)

<font color=red>指将头部字段表示为索引值。</font>

索引的头部字段表示标识静态表或动态表 (参见 [第2.3节](https://tools.ietf.org/html/rfc7541#section-2.3)) 中的条目。

索引的头部字段表示使头部字段被加进解码的头部列表，如 [第 3.2节](https://tools.ietf.org/html/rfc7541#section-3.2) 所述。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 1 |        Index (7+)         |
+---+---------------------------+
```
图 5：索引的头部字段

索引的头部字段以'1'的 1位模式开始，后面是匹配的头部字段索引，由一个7位前缀的整数 (参见 [第5.1节](https://tools.ietf.org/html/rfc7541#section-5.1)) 表示。

不使用索引值0。如果在索引的头部字段表示中发现则 **必须(MUST)** 将其视为解码错误。

## [6.2 字面的头部字段表示](https://tools.ietf.org/html/rfc7541#section-6.2)

字面的头部字段表示包含一个 **字面的头部字段值**。头部字段名由一个字面量或对静态表或动态表 (参见 [第2.3节](https://tools.ietf.org/html/rfc7541#section-2.3))中一个已有表条目的引用表示。

<font color=red>字面的头部字段表示法中，头部字段的名可以用索引或字面量表示，而值则必须以字面量表示。</font>

本规范定义三种 **字面的头部字段** 表示格式：有索引的，无索引的，和从不索引的。

### [6.2.1 增量索引表示的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.1)

**增量索引表示的字面量头部字段** 导致将头部字段加进解码的头部列表，并将其作为新条目插入到动态表中。

<font color=red>通信的双方不会专门发送动态表。编码时动态表被编码进头部列表。解码时解码器从头部列表动态地构建出动态表。</font>

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |      Index (6+)       |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
图 6：增量索引表示的字面量头部字段 - 索引的名字
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |           0           |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
图 7：增量索引表示的字面量头部字段 - 新的名字

增量索引表示的字面量头部字段表示以'01' 2位模式开始。

如果头部字段名与静态表或动态表中存储的条目的头部字段名匹配，则头部字段名称可用那个条目的索引表示。在这种情况下，条目的索引以一个具有6位前缀的整数(参见 [第5.1节](https://tools.ietf.org/html/rfc7541#section-5.1))表示。这个值总是非0。

否则，头部字段名由一个字符串字面量 (参见 [第5.2节](https://tools.ietf.org/html/rfc7541#section-5.2)) 表示，使用0值代替6位索引，其后是头字段名。

两种形式的 **头部字段名表示** 之后是字符串字面量表示的头部字段值 (参见 [第5.2节](https://tools.ietf.org/html/rfc7541#section-5.2))。

### [6.2.2 无索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.2)

无索引的字面量头部字段表示导致将头部字段加进解码的头部列表而 **不改变动态表**。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
图 8：无索引的字面量头部字段 - 索引的名字
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
图 9：无索引的字面量头部字段 - 新的名字

无索引的字面量头部字段表示以'0000' 4位模式开始。

如果头部字段名与静态表或动态表中存储的条目的头部字段名匹配，则头部字段名可使用那个条目的索引表示。在这种情况下，条目的索引以一个有着4位前缀的整数 (参见 [第5.1节](https://tools.ietf.org/html/rfc7541#section-5.1))表示。这个值总是非0。

否则，头部字段名由一个字符串字面量 (参见 [第5.2节](https://http2.github.io/http2-spec/compression.html#string.literal.representation)) 表示，使用值0代替4位索引，其后是头部字段名。

两种形式的 **头部字段名表示** 之后是字符串字面量形式表示的头部字段值 (见 [第5.2节](https://tools.ietf.org/html/rfc7541#section-5.2))。

### [6.2.3 从不索引的字面量头部字段](https://tools.ietf.org/html/rfc7541#section-6.2.3)

从不索引的字面量头部字段表示导致头部字段被加进解码的头部列表而不改变动态表。中介在编码这个头部字段时 **必须(MUST)** 使用相同的表示。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+

```
图 10：从不索引的字面量头部字段 - 索引的名字
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
图 11：从不索引的字面量头部字段 - 新的名字

从不索引的字面量头部字段表示以'0001' 4位模式开始。

当头部字段被表示为从 **不索引的字面量头部字段** 时，它 **必须(MUST)** 总是以该特定的 **字面量表示** 编码。尤其是当一个端点发送它接收的以 **从不索引的字面量头部字段** 表示的头部字段，它 **必须(MUST)** 以相同的表示转发该头部字段。

这种表示旨在保护不会由于压缩而带来风险的头部字段值 (参见 [第 7.1 节](https://tools.ietf.org/html/rfc7541#section-7.1) 获得更多细节)。

<font color=red>这是啥意思呢？</font>

此表示的编码与无索引的字面量头部字段(参见 [第 6.2.2节](https://tools.ietf.org/html/rfc7541#section-6.2.2))一致。

## [6.3 动态表 大小更新](https://tools.ietf.org/html/rfc7541#section-6.3)

**动态表大小更新** 通知一个对动态表大小的改变。
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 |   Max size (5+)   |
+---+---------------------------+
```
图 12：最大动态表大小更改

动态表大小更新以 '001' 的 3位模式开始，后面是新的最大大小，以一个具有5位前缀的整数 (参见 [第5.1节](https://tools.ietf.org/html/rfc7541#section-5.1)) 表示。

新的最大大小 **必须(MUST)** 小于等于使用HPACK的协议决定的限制。超出该限制的值 **必须(MUST)** 被视为解码错误。在HTTP/2中，这个限制是从解码器接收而由编码器确认 (参见 [[HTTP2](https://tools.ietf.org/html/rfc7541#ref-HTTP2)] 的 第6.5.3节) 的最后的SETTINGS_HEADER_TABLE_SIZE 参数值 (参见 [[HTTP2](https://tools.ietf.org/html/rfc7541#ref-HTTP2)] 的第 6.5.2节)，

减小动态表的最大大小可能导致条目被逐出 (参见 [第4.3节](https://tools.ietf.org/html/rfc7541#section-4.3)) 。

# [7. 安全注意事项](https://tools.ietf.org/html/rfc7541#section-7)

本节介绍与HPACK有关可能发生安全问题的潜在区域：

* 使用压缩作为基于长度的预示来验证关于压缩到共享的压缩上下文的机密的猜测。
* 由于解码器耗尽处理或存储能力而导致的拒绝服务。

## [7.1 探测动态表状态](https://tools.ietf.org/html/rfc7541#section-7.1)

HPACK通过利用诸如HTTP之类的协议中固有的冗余来减少头部字段编码的长度。最终的目标是减少发送HTTP请求或响应所需的数据量。

可以同时定义编码并传输的头部字段，且在它们被编码之后可以立即观察那些字段的长度的攻击者，可以探测到用于编码头部字段的上下文。当一个攻击者可以做到这两点，他们可以自适应地修改请求以确认关于动态表状态的猜测。如果猜测被压缩到更短的长度，攻击者可以观察编码长度并推断猜测是正确的。

即使是基于 传输层安全 (TLS) 协议 (参见 [[TLS12](https://tools.ietf.org/html/rfc7541#ref-TLS12)]这也是可能的，因为尽管TLS为内容提供了机密的保护，但只为内容的长度提供了有限数量的保护。

**注意：**填充方案只提供了针对具有这些能力的攻击者的有限的保护，潜在地仅仅迫使增加猜测的数量来了解与给定猜测相关联的长度。填充方案也通过增加发送的比特数而直接对抗压缩。

类似 CRIME [[CRIME](https://tools.ietf.org/html/rfc7541#ref-CRIME)] 的攻击证明了这些一般攻击者能力的存在。特定攻击利用了 DEFLATE [[DEFLATE](https://tools.ietf.org/html/rfc7541#ref-DEFLATE)]  基于前缀匹配删除冗余的事实。这允许攻击者一次确认一个字符的猜测，将指数时间的攻击减少为线性时间的攻击。

### [7.1.1 HPACK 和 HTTP的适用性](https://tools.ietf.org/html/rfc7541#section-7.1.1)

HPACK通过强制一个猜测匹配整个头部字段值而不是单独的字符，而缓解但没有完全消除基于模型 CRIME [[CRIME](https://http2.github.io/http2-spec/compression.html#CRIME)]的攻击。攻击者只能知道一个猜测是对的还是错的，因此他们被降低为粗略的猜测头部字段值。

然而恢复特定的头部字段值依赖于值的熵。结果是，具有高熵的值不可能成功地恢复，低熵的值仍然脆弱。

这种属性的攻击在两个互不信任的实体控制放在单独的HTTP/2连接上的请求或响应的任何时候都是可能的。如果共享的HPACK压缩器允许一个实体向动态表中添加条目另一个访问那些条目，则表的状态可以被获取。

具有来自相互不信任实体的请求或响应的情况发生在中间方：在单个连接上向原始服务器发送来自于多个客户端的请求，或者从多个源服务器获得响应并将它们放置在朝向客户端的共享连接上。

Web浏览器还需要假设由不同的web源 [[ORIGIN](https://tools.ietf.org/html/rfc7541#ref-ORIGIN)] 在同一连接上发出的请求由相互不可信的实体创建。

### [7.1.2 减轻](https://tools.ietf.org/html/rfc7541#section-7.1.2)

需要对头部字段保密的HTTP用户可以使用具有足以使猜测不可行的熵的值。然而，作为一般解决方案这是不切实际的，因为它迫使所有的HTTP用户采取措施以减轻攻击。它将在如何使用HTTP方面引入新的限制。它将对如何使用HTTP施加新的约束。

不是对HTTP的用户施加约束，HPACK的实现可以通过限制应用压缩的顺序来限制潜在的对动态表的探测。

不是对HTTP的用户施加约束，HPACK的实现可以替代地限制如何应用压缩以便限制动态表探测的可能性。

理想的方案是基于构建头部字段的实体隔离对动态表的访问。添加到表里的头部字段值归属于实体，只有创建了特定值的实体可以提取那个值。

要提升这个选项的压缩性能，某些条目可以被标记为public。比如，浏览器可以使所有的请求都能访问Accept-Encoding头部字段的值。

不知道头部字段的来源的编码器可以替代地对具有许多不同值的头部字段引入惩罚机制，以此大量地尝试猜测头部字段值，使得在未来的消息中，头部字段不再与动态表的条目比较，从而有效地避免进一步的猜测。

**注意：**简单地移除对应于来自动态表的头部字段的条目可能是无效的，如果攻击者具有可靠的使值被重新安装的方法的话。例如，在网络浏览器中加载图像的请求通常包括Cookie头部字段 (对于这种攻击而言潜在地高价值目标)，并且网站可以简单地强制加载图像，从而刷新动态表。

该响应可以与头部字段值的长度成反比。将头部字段标记为不再使用动态表可以对于更短的值更快地或以比更长的值更高的概率发生。

### [7.1.3 从不索引的字面量](https://tools.ietf.org/html/rfc7541#section-7.1.3)

实现也可以通过选择不压缩敏感头部字段而以字面值编码来保护它们。

拒绝为头部字段产生索引的表示只有在所有跳段上都避免压缩时才有效。从不索引字面量(参见[第6.2.3节](https://tools.ietf.org/html/rfc7541#section-6.2.3))可被用于通知中介，一个特定的值有意以字面的形式发送。

中介 **一定不能(MUST NOT)** 以将索引面量表示的另一种表示重编码使用从不索引字面量表示的值。如果HPACK被用于重编码，则 **必须(MUST)** 使用从不索引字面量表示。

选择为头部字段使用从不索引字面量表示依赖于几个因素。由于HPACK不保护对完整的头部字段值的猜测，短的或低熵的值更容易被对手恢复。因而编码器可以选择不索引低熵的值

编码器也可以选择不索引被认为具有很高价值或敏感的头部字段值，比如Cookie或Authorization头部字段。

相反地，编码器可以优先索引那些如果暴露，价值比较小或无价值的头部字段。比如，在发送给任何服务器的请求间通常不变的User-Agent头部字段。在这种情况下，确认使用了一个特定的User-Agent值提供很低的价值。

注意，决定使用从不索引字面量表示的这些标准将随着时间流逝，新的攻击被发现而发展。

## [7.2 静态Huffman编码](https://http2.github.io/http2-spec/compression.html#rfc.section.7.2)

目前还没有已知的针对静态Huffman编码的攻击。一项研究显示使用静态Huffman编码表造成了信息泄漏；然而，相同的研究得出结论攻击者无法利用这些信息泄漏来恢复任何有意义数量的信息 (见 [[PETAL]](https://http2.github.io/http2-spec/compression.html#PETAL))。

## [7.3 内存消耗](https://http2.github.io/http2-spec/compression.html#rfc.section.7.3)

攻击者可能尝试导致一个终端耗尽它的内存。HPACK旨在限制端点分配的内存的峰值和状态量

编码器使用的内存量由使用HPACK的协议通过定义动态表的最大大小来限制。在HTTP/2中，这个值由解码器通过设置参数 SETTINGS_HEADER_TABLE_SIZE (见 [[HTTP2]](https://http2.github.io/http2-spec/compression.html#HTTP2) 的 [Section 6.5.2](https://tools.ietf.org/html/rfc7540#section-6.5.2)) 控制。这个限制考虑了存储在动态表中的数据的大小，加一个小的开销的宽限两者。

解码器可以通过为动态表的最大大小设置一个适当的值来限制内存使用的状态数量。在HTTP/2中，这是通过为SETTINGS_HEADER_TABLE_SIZE参数设置一个适当的值来实现的。编码器可以通过发信号通知一个比解码器允许的更小的动态表大小来限制内存使用的状态数量 (见 [Section 6.3](https://http2.github.io/http2-spec/compression.html#encoding.context.update))。

编码器或解码器临时内存的消耗数量可以通过顺序地处理头部字段来限制。实现不需要保留完整的头部字段的列表。注意，然而，应用程序可能由于其它原因而需要保留完整的头部列表；即使HPACK不强制这种情况发生，应用程序约束可能使这是必要的。

## [7.4 实现的限制](https://http2.github.io/http2-spec/compression.html#rfc.section.7.4)

HPACK的实现需要确保整数的大值，整数的长编码，或长的字符串字面量不会创造安全弱点。

实现不得不为它接收的整数值，及编码的长度设置一个限制(见 [Section 5.1](https://http2.github.io/http2-spec/compression.html#integer.representation))。以同样的方式，它不得不为它接收的字符串字面量(见 [Section 5.2](https://http2.github.io/http2-spec/compression.html#string.literal.representation) )的长度设置一个限制。

# [8. 参考文献](https://http2.github.io/http2-spec/compression.html#rfc.section.8) 

## [8.1 规范性参考文献](https://http2.github.io/http2-spec/compression.html#rfc.section.8.1) 

```
**[HTTP2]**     Belshe, M., Peon, R., and M. Thomson, Ed., “[Hypertext Transfer Protocol Version 2 
                (HTTP/2)](https://tools.ietf.org/html/rfc7540)”, RFC 7540, [DOI 10.17487/RFC7540](http://dx.doi.org/10.17487/RFC7540), May 2015, <[http://www.rfc-
                editor.org/info/rfc7540](http://www.rfc-editor.org/info/rfc7540)>.

**[RFC2119]**   Bradner, S., “[Key words for use in RFCs to Indicate Requirement Levels](https://tools.ietf.org/html/rfc2119)”, BCP 14, 
                RFC 2119, [DOI 10.17487/RFC2119](http://dx.doi.org/10.17487/RFC2119), March 1997, <[http://www.rfc-
                editor.org/info/rfc2119](http://www.rfc-editor.org/info/rfc2119)>.

**[RFC7230]**   Fielding, R., Ed. and J. Reschke, Ed., “[Hypertext Transfer Protocol (HTTP/1.1): 
                Message Syntax and Routing](https://tools.ietf.org/html/rfc7230)”, RFC 7230, [DOI 10.17487/RFC7230](http://dx.doi.org/10.17487/RFC7230), June 2014, 
                <[http://www.rfc-editor.org/info/rfc7230](http://www.rfc-editor.org/info/rfc7230)>.
```

## [8.2 参考资料](https://http2.github.io/http2-spec/compression.html#rfc.section.8.2)
```
**[CANONICAL]**  Schwartz, E. and B. Kallick, “[Generating a canonical prefix encoding](https://dl.acm.org/citation.cfm?id=363991)”, 
                 Communications of the ACM, Volume 7 Issue 3, pp. 166-169, March 1964, 
                 <[https://dl.acm.org/citation.cfm?id=363991](https://dl.acm.org/citation.cfm?id=363991)>.

**[CRIME]**      Wikipedia, “[CRIME](http://en.wikipedia.org/w/index.php?
                 title=CRIME&oldid=660948120)”, May 2015, <[http://en.wikipedia.org/w/index.php?title=CRIME&oldid=660948120](http://en.wikipedia.org/w/index.php?title=CRIME&oldid=660948120)>.

**[DEFLATE]**    Deutsch, P., “[DEFLATE Compressed Data Format Specification version 1.3](https://tools.ietf.org/html/rfc1951)”, 
                 RFC 1951, [DOI 10.17487/RFC1951](http://dx.doi.org/10.17487/RFC1951), May 1996, <[http://www.rfc-
                 editor.org/info/rfc1951](http://www.rfc-editor.org/info/rfc1951)>.

**[HUFFMAN]**    Huffman, D., “[A Method for the Construction of Minimum-Redundancy Codes](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=4051119)”, 
                 Proceedings of the Institute of Radio Engineers, Volume 40, Number 9, pp. 1098-
                 1101, September 1952, <[http://ieeexplore.ieee.org/xpl/articleDetails.jsp?
                 arnumber=4051119](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=4051119)>.

**[ORIGIN]**     Barth, A., “[The Web Origin Concept](https://tools.ietf.org/html/rfc6454)”, RFC 6454, [DOI 10.17487/RFC6454](http://dx.doi.org/10.17487/RFC6454), 
                 December 2011, <[http://www.rfc-editor.org/info/rfc6454](http://www.rfc-editor.org/info/rfc6454)>.

**[PETAL]**      Tan, J. and J. Nahata, “[PETAL: Preset Encoding Table Information Leakage](http://www.pdl.cmu.edu/PDL-FTP/associated/CMU-PDL-13-106.pdf)”, 
                 April 2013, <[http://www.pdl.cmu.edu/PDL-FTP/associated/CMU-PDL-13-106.pdf](http://www.pdl.cmu.edu/PDL-FTP/associated/CMU-PDL-13-106.pdf)>.

**[SPDY]**       Belshe, M. and R. Peon, “[SPDY Protocol](https://tools.ietf.org/html/draft-mbelshe-httpbis-spdy-00)”, Internet-Draft draft-mbelshe-httpbis-
                 spdy-00 (work in progress), February 2012.

**[TLS12]**      Dierks, T. and E. Rescorla, “[The Transport Layer Security (TLS) Protocol Version 
                 1.2](https://tools.ietf.org/html/rfc5246)”, RFC 5246, [DOI 10.17487/RFC5246](http://dx.doi.org/10.17487/RFC5246), August 2008, <[http://www.rfc-
                 editor.org/info/rfc5246](http://www.rfc-editor.org/info/rfc5246)>.
```

# [A. 静态表定义](https://http2.github.io/http2-spec/compression.html#static.table.definition)

静态表 (参见 [Section 2.3.1](https://http2.github.io/http2-spec/compression.html#static.table)) 由一个预定义的且不可改变的头部字段列表组成。

静态表从流行的web站点最常使用的头部字段创建，还包含了HTTP/2特有的伪头部字段 (参见 [[HTTP2]
](https://http2.github.io/http2-spec/compression.html#HTTP2) 的 [Section 8.1.2.1](https://tools.ietf.org/html/rfc7540#section-8.1.2.1))。对于有多个常用值的头部字段，将为这些常用值的每个都添加一个条目。对于其它的头部字段，将为其添加一个值为空的条目。

[Table 1](https://http2.github.io/http2-spec/compression.html#static.table.entries) 列出了组成静态表的预定义的头部字段，并给出了每个条目的索引。

|Index |Header Name	                |Header Value  |
|------|----------------------------|--------------|
|1	   |`:authority`	            |              |
|2	   |`:method`	                |GET           |
|3	   |`:method`	                |POST          |
|4	   |`:path`	                	|/             |
|5	   |`:path`	                	|/index.html   |
|6	   |`:scheme`	                |http          |
|7	   |`:scheme`	            	|https         |
|8	   |`:status`	            	|200           |
|9	   |`:status`               	|204           |
|10	   |`:status`	            	|206           |
|11	   |`:status`	            	|304           |
|12	   |`:status`	            	|400           |
|13	   |`:status`	            	|404           |
|14    |`:status`	            	|500           |
|15	   |accept-charset	            |              |
|16	   |accept-encoding	            |gzip, deflate |
|17	   |accept-language	            |              |
|18	   |accept-ranges	            |              |
|19	   |accept	                    |              |
|20	   |access-control-allow-origin	|              |
|21	   |age	                        |              |
|22	   |allow	                    |              |
|23	   |authorization	            |              |
|24	   |cache-control	            |              |
|25	   |content-disposition	        |              |
|26	   |content-encoding	        |              |
|27	   |content-language	        |              |
|28	   |content-length	            |              |
|29	   |content-location	        |              |
|30	   |content-range	            |              |
|31	   |content-type	            |              |
|32	   |cookie	                    |              |
|33	   |date	                    |              |
|34	   |etag	                    |              |
|35	   |expect	                    |              |
|36	   |expires	                    |              |
|37	   |from	                    |              |
|38	   |host	                    |              |
|39	   |if-match	                |              |
|40	   |if-modified-since	        |              |
|41	   |if-none-match	            |              |
|42	   |if-range	                |              |
|43	   |if-unmodified-since	        |              |
|44	   |last-modified	            |              |
|45	   |link	                    |              |
|46	   |location	                |              |
|47	   |max-forwards	            |              |
|48	   |proxy-authenticate	        |              |
|49	   |proxy-authorization	        |              |
|50	   |range	                    |              |
|51	   |referer	                    |              |
|52	   |refresh	                    |              |
|53	   |retry-after	                |              |
|54	   |server	                    |              |
|55	   |set-cookie	                |              |
|56	   |strict-transport-security	|              |
|57	   |transfer-encoding	        |              |
|58	   |user-agent	                |              |
|59	   |vary	                    |              |
|60	   |via	                        |              |
|61	   |www-authenticate	        |              |

Table 1: 静态表条目

# [B. Huffman码](https://http2.github.io/http2-spec/compression.html#huffman.code)

当使用Huffman编码对字符串字面量进行编码时，使用了以下Huffman码（[见第5.2节](https://http2.github.io/http2-spec/compression.html#string.literal.representation)）。

这里的Huffman码是从基于大量的HTTP头部采样获取的统计而产生的。它是一个包含了一些调整以确保没有符号具有唯一的代码长度的规范Huffman码 (见 [[CANONICAL]
](https://http2.github.io/http2-spec/compression.html#CANONICAL) )。

表中的每一行定义了用于表示一个符号的代码：

**sym:** 要表示的符号。它是一个字节的十进制值，可能在其ASCII表示前面。一个特别的符号， "EOS"，被用于指示一个字符传字面量的结束。

**code as bits:** 二进制整数表示的符号的Huffman码，在最高有效位 (MSB) 对齐。

**code as hex:** 符号的Huffman码，以16进制整数表示，在最低有效位对齐。

**len:** 表示符号的码的位数。

比如，符号 47 (对应于ASCII字符"/") 的代码由 6 位"0"，"1"，"1"，"0"，"0"，"0"组成。这对应于用6位编码的值 0x18 (以十六进制)。

```
                                                     code
                       code as bits                 as hex   len
     sym              aligned to MSB                aligned   in
                                                    to LSB   bits
    (  0)  |11111111|11000                             1ff8  [13]
    (  1)  |11111111|11111111|1011000                7fffd8  [23]
    (  2)  |11111111|11111111|11111110|0010         fffffe2  [28]
    (  3)  |11111111|11111111|11111110|0011         fffffe3  [28]
    (  4)  |11111111|11111111|11111110|0100         fffffe4  [28]
    (  5)  |11111111|11111111|11111110|0101         fffffe5  [28]
    (  6)  |11111111|11111111|11111110|0110         fffffe6  [28]
    (  7)  |11111111|11111111|11111110|0111         fffffe7  [28]
    (  8)  |11111111|11111111|11111110|1000         fffffe8  [28]
    (  9)  |11111111|11111111|11101010               ffffea  [24]
    ( 10)  |11111111|11111111|11111111|111100      3ffffffc  [30]
    ( 11)  |11111111|11111111|11111110|1001         fffffe9  [28]
    ( 12)  |11111111|11111111|11111110|1010         fffffea  [28]
    ( 13)  |11111111|11111111|11111111|111101      3ffffffd  [30]
    ( 14)  |11111111|11111111|11111110|1011         fffffeb  [28]
    ( 15)  |11111111|11111111|11111110|1100         fffffec  [28]
    ( 16)  |11111111|11111111|11111110|1101         fffffed  [28]
    ( 17)  |11111111|11111111|11111110|1110         fffffee  [28]
    ( 18)  |11111111|11111111|11111110|1111         fffffef  [28]
    ( 19)  |11111111|11111111|11111111|0000         ffffff0  [28]
    ( 20)  |11111111|11111111|11111111|0001         ffffff1  [28]
    ( 21)  |11111111|11111111|11111111|0010         ffffff2  [28]
    ( 22)  |11111111|11111111|11111111|111110      3ffffffe  [30]
    ( 23)  |11111111|11111111|11111111|0011         ffffff3  [28]
    ( 24)  |11111111|11111111|11111111|0100         ffffff4  [28]
    ( 25)  |11111111|11111111|11111111|0101         ffffff5  [28]
    ( 26)  |11111111|11111111|11111111|0110         ffffff6  [28]
    ( 27)  |11111111|11111111|11111111|0111         ffffff7  [28]
    ( 28)  |11111111|11111111|11111111|1000         ffffff8  [28]
    ( 29)  |11111111|11111111|11111111|1001         ffffff9  [28]
    ( 30)  |11111111|11111111|11111111|1010         ffffffa  [28]
    ( 31)  |11111111|11111111|11111111|1011         ffffffb  [28]
' ' ( 32)  |010100                                       14  [ 6]
'!' ( 33)  |11111110|00                                 3f8  [10]
'"' ( 34)  |11111110|01                                 3f9  [10]
'#' ( 35)  |11111111|1010                               ffa  [12]
'$' ( 36)  |11111111|11001                             1ff9  [13]
'%' ( 37)  |010101                                       15  [ 6]
'&' ( 38)  |11111000                                     f8  [ 8]
''' ( 39)  |11111111|010                                7fa  [11]
'(' ( 40)  |11111110|10                                 3fa  [10]
')' ( 41)  |11111110|11                                 3fb  [10]
'*' ( 42)  |11111001                                     f9  [ 8]
'+' ( 43)  |11111111|011                                7fb  [11]
',' ( 44)  |11111010                                     fa  [ 8]
'-' ( 45)  |010110                                       16  [ 6]
'.' ( 46)  |010111                                       17  [ 6]
'/' ( 47)  |011000                                       18  [ 6]
'0' ( 48)  |00000                                         0  [ 5]
'1' ( 49)  |00001                                         1  [ 5]
'2' ( 50)  |00010                                         2  [ 5]
'3' ( 51)  |011001                                       19  [ 6]
'4' ( 52)  |011010                                       1a  [ 6]
'5' ( 53)  |011011                                       1b  [ 6]
'6' ( 54)  |011100                                       1c  [ 6]
'7' ( 55)  |011101                                       1d  [ 6]
'8' ( 56)  |011110                                       1e  [ 6]
'9' ( 57)  |011111                                       1f  [ 6]
':' ( 58)  |1011100                                      5c  [ 7]
';' ( 59)  |11111011                                     fb  [ 8]
'<' ( 60)  |11111111|1111100                           7ffc  [15]
'=' ( 61)  |100000                                       20  [ 6]
'>' ( 62)  |11111111|1011                               ffb  [12]
'?' ( 63)  |11111111|00                                 3fc  [10]
'@' ( 64)  |11111111|11010                             1ffa  [13]
'A' ( 65)  |100001                                       21  [ 6]
'B' ( 66)  |1011101                                      5d  [ 7]
'C' ( 67)  |1011110                                      5e  [ 7]
'D' ( 68)  |1011111                                      5f  [ 7]
'E' ( 69)  |1100000                                      60  [ 7]
'F' ( 70)  |1100001                                      61  [ 7]
'G' ( 71)  |1100010                                      62  [ 7]
'H' ( 72)  |1100011                                      63  [ 7]
'I' ( 73)  |1100100                                      64  [ 7]
'J' ( 74)  |1100101                                      65  [ 7]
'K' ( 75)  |1100110                                      66  [ 7]
'L' ( 76)  |1100111                                      67  [ 7]
'M' ( 77)  |1101000                                      68  [ 7]
'N' ( 78)  |1101001                                      69  [ 7]
'O' ( 79)  |1101010                                      6a  [ 7]
'P' ( 80)  |1101011                                      6b  [ 7]
'Q' ( 81)  |1101100                                      6c  [ 7]
'R' ( 82)  |1101101                                      6d  [ 7]
'S' ( 83)  |1101110                                      6e  [ 7]
'T' ( 84)  |1101111                                      6f  [ 7]
'U' ( 85)  |1110000                                      70  [ 7]
'V' ( 86)  |1110001                                      71  [ 7]
'W' ( 87)  |1110010                                      72  [ 7]
'X' ( 88)  |11111100                                     fc  [ 8]
'Y' ( 89)  |1110011                                      73  [ 7]
'Z' ( 90)  |11111101                                     fd  [ 8]
'[' ( 91)  |11111111|11011                             1ffb  [13]
'\' ( 92)  |11111111|11111110|000                     7fff0  [19]
']' ( 93)  |11111111|11100                             1ffc  [13]
'^' ( 94)  |11111111|111100                            3ffc  [14]
'_' ( 95)  |100010                                       22  [ 6]
'`' ( 96)  |11111111|1111101                           7ffd  [15]
'a' ( 97)  |00011                                         3  [ 5]
'b' ( 98)  |100011                                       23  [ 6]
'c' ( 99)  |00100                                         4  [ 5]
'd' (100)  |100100                                       24  [ 6]
'e' (101)  |00101                                         5  [ 5]
'f' (102)  |100101                                       25  [ 6]
'g' (103)  |100110                                       26  [ 6]
'h' (104)  |100111                                       27  [ 6]
'i' (105)  |00110                                         6  [ 5]
'j' (106)  |1110100                                      74  [ 7]
'k' (107)  |1110101                                      75  [ 7]
'l' (108)  |101000                                       28  [ 6]
'm' (109)  |101001                                       29  [ 6]
'n' (110)  |101010                                       2a  [ 6]
'o' (111)  |00111                                         7  [ 5]
'p' (112)  |101011                                       2b  [ 6]
'q' (113)  |1110110                                      76  [ 7]
'r' (114)  |101100                                       2c  [ 6]
's' (115)  |01000                                         8  [ 5]
't' (116)  |01001                                         9  [ 5]
'u' (117)  |101101                                       2d  [ 6]
'v' (118)  |1110111                                      77  [ 7]
'w' (119)  |1111000                                      78  [ 7]
'x' (120)  |1111001                                      79  [ 7]
'y' (121)  |1111010                                      7a  [ 7]
'z' (122)  |1111011                                      7b  [ 7]
'{' (123)  |11111111|1111110                           7ffe  [15]
'|' (124)  |11111111|100                                7fc  [11]
'}' (125)  |11111111|111101                            3ffd  [14]
'~' (126)  |11111111|11101                             1ffd  [13]
    (127)  |11111111|11111111|11111111|1100         ffffffc  [28]
    (128)  |11111111|11111110|0110                    fffe6  [20]
    (129)  |11111111|11111111|010010                 3fffd2  [22]
    (130)  |11111111|11111110|0111                    fffe7  [20]
    (131)  |11111111|11111110|1000                    fffe8  [20]
    (132)  |11111111|11111111|010011                 3fffd3  [22]
    (133)  |11111111|11111111|010100                 3fffd4  [22]
    (134)  |11111111|11111111|010101                 3fffd5  [22]
    (135)  |11111111|11111111|1011001                7fffd9  [23]
    (136)  |11111111|11111111|010110                 3fffd6  [22]
    (137)  |11111111|11111111|1011010                7fffda  [23]
    (138)  |11111111|11111111|1011011                7fffdb  [23]
    (139)  |11111111|11111111|1011100                7fffdc  [23]
    (140)  |11111111|11111111|1011101                7fffdd  [23]
    (141)  |11111111|11111111|1011110                7fffde  [23]
    (142)  |11111111|11111111|11101011               ffffeb  [24]
    (143)  |11111111|11111111|1011111                7fffdf  [23]
    (144)  |11111111|11111111|11101100               ffffec  [24]
    (145)  |11111111|11111111|11101101               ffffed  [24]
    (146)  |11111111|11111111|010111                 3fffd7  [22]
    (147)  |11111111|11111111|1100000                7fffe0  [23]
    (148)  |11111111|11111111|11101110               ffffee  [24]
    (149)  |11111111|11111111|1100001                7fffe1  [23]
    (150)  |11111111|11111111|1100010                7fffe2  [23]
    (151)  |11111111|11111111|1100011                7fffe3  [23]
    (152)  |11111111|11111111|1100100                7fffe4  [23]
    (153)  |11111111|11111110|11100                  1fffdc  [21]
    (154)  |11111111|11111111|011000                 3fffd8  [22]
    (155)  |11111111|11111111|1100101                7fffe5  [23]
    (156)  |11111111|11111111|011001                 3fffd9  [22]
    (157)  |11111111|11111111|1100110                7fffe6  [23]
    (158)  |11111111|11111111|1100111                7fffe7  [23]
    (159)  |11111111|11111111|11101111               ffffef  [24]
    (160)  |11111111|11111111|011010                 3fffda  [22]
    (161)  |11111111|11111110|11101                  1fffdd  [21]
    (162)  |11111111|11111110|1001                    fffe9  [20]
    (163)  |11111111|11111111|011011                 3fffdb  [22]
    (164)  |11111111|11111111|011100                 3fffdc  [22]
    (165)  |11111111|11111111|1101000                7fffe8  [23]
    (166)  |11111111|11111111|1101001                7fffe9  [23]
    (167)  |11111111|11111110|11110                  1fffde  [21]
    (168)  |11111111|11111111|1101010                7fffea  [23]
    (169)  |11111111|11111111|011101                 3fffdd  [22]
    (170)  |11111111|11111111|011110                 3fffde  [22]
    (171)  |11111111|11111111|11110000               fffff0  [24]
    (172)  |11111111|11111110|11111                  1fffdf  [21]
    (173)  |11111111|11111111|011111                 3fffdf  [22]
    (174)  |11111111|11111111|1101011                7fffeb  [23]
    (175)  |11111111|11111111|1101100                7fffec  [23]
    (176)  |11111111|11111111|00000                  1fffe0  [21]
    (177)  |11111111|11111111|00001                  1fffe1  [21]
    (178)  |11111111|11111111|100000                 3fffe0  [22]
    (179)  |11111111|11111111|00010                  1fffe2  [21]
    (180)  |11111111|11111111|1101101                7fffed  [23]
    (181)  |11111111|11111111|100001                 3fffe1  [22]
    (182)  |11111111|11111111|1101110                7fffee  [23]
    (183)  |11111111|11111111|1101111                7fffef  [23]
    (184)  |11111111|11111110|1010                    fffea  [20]
    (185)  |11111111|11111111|100010                 3fffe2  [22]
    (186)  |11111111|11111111|100011                 3fffe3  [22]
    (187)  |11111111|11111111|100100                 3fffe4  [22]
    (188)  |11111111|11111111|1110000                7ffff0  [23]
    (189)  |11111111|11111111|100101                 3fffe5  [22]
    (190)  |11111111|11111111|100110                 3fffe6  [22]
    (191)  |11111111|11111111|1110001                7ffff1  [23]
    (192)  |11111111|11111111|11111000|00           3ffffe0  [26]
    (193)  |11111111|11111111|11111000|01           3ffffe1  [26]
    (194)  |11111111|11111110|1011                    fffeb  [20]
    (195)  |11111111|11111110|001                     7fff1  [19]
    (196)  |11111111|11111111|100111                 3fffe7  [22]
    (197)  |11111111|11111111|1110010                7ffff2  [23]
    (198)  |11111111|11111111|101000                 3fffe8  [22]
    (199)  |11111111|11111111|11110110|0            1ffffec  [25]
    (200)  |11111111|11111111|11111000|10           3ffffe2  [26]
    (201)  |11111111|11111111|11111000|11           3ffffe3  [26]
    (202)  |11111111|11111111|11111001|00           3ffffe4  [26]
    (203)  |11111111|11111111|11111011|110          7ffffde  [27]
    (204)  |11111111|11111111|11111011|111          7ffffdf  [27]
    (205)  |11111111|11111111|11111001|01           3ffffe5  [26]
    (206)  |11111111|11111111|11110001               fffff1  [24]
    (207)  |11111111|11111111|11110110|1            1ffffed  [25]
    (208)  |11111111|11111110|010                     7fff2  [19]
    (209)  |11111111|11111111|00011                  1fffe3  [21]
    (210)  |11111111|11111111|11111001|10           3ffffe6  [26]
    (211)  |11111111|11111111|11111100|000          7ffffe0  [27]
    (212)  |11111111|11111111|11111100|001          7ffffe1  [27]
    (213)  |11111111|11111111|11111001|11           3ffffe7  [26]
    (214)  |11111111|11111111|11111100|010          7ffffe2  [27]
    (215)  |11111111|11111111|11110010               fffff2  [24]
    (216)  |11111111|11111111|00100                  1fffe4  [21]
    (217)  |11111111|11111111|00101                  1fffe5  [21]
    (218)  |11111111|11111111|11111010|00           3ffffe8  [26]
    (219)  |11111111|11111111|11111010|01           3ffffe9  [26]
    (220)  |11111111|11111111|11111111|1101         ffffffd  [28]
    (221)  |11111111|11111111|11111100|011          7ffffe3  [27]
    (222)  |11111111|11111111|11111100|100          7ffffe4  [27]
    (223)  |11111111|11111111|11111100|101          7ffffe5  [27]
    (224)  |11111111|11111110|1100                    fffec  [20]
    (225)  |11111111|11111111|11110011               fffff3  [24]
    (226)  |11111111|11111110|1101                    fffed  [20]
    (227)  |11111111|11111111|00110                  1fffe6  [21]
    (228)  |11111111|11111111|101001                 3fffe9  [22]
    (229)  |11111111|11111111|00111                  1fffe7  [21]
    (230)  |11111111|11111111|01000                  1fffe8  [21]
    (231)  |11111111|11111111|1110011                7ffff3  [23]
    (232)  |11111111|11111111|101010                 3fffea  [22]
    (233)  |11111111|11111111|101011                 3fffeb  [22]
    (234)  |11111111|11111111|11110111|0            1ffffee  [25]
    (235)  |11111111|11111111|11110111|1            1ffffef  [25]
    (236)  |11111111|11111111|11110100               fffff4  [24]
    (237)  |11111111|11111111|11110101               fffff5  [24]
    (238)  |11111111|11111111|11111010|10           3ffffea  [26]
    (239)  |11111111|11111111|1110100                7ffff4  [23]
    (240)  |11111111|11111111|11111010|11           3ffffeb  [26]
    (241)  |11111111|11111111|11111100|110          7ffffe6  [27]
    (242)  |11111111|11111111|11111011|00           3ffffec  [26]
    (243)  |11111111|11111111|11111011|01           3ffffed  [26]
    (244)  |11111111|11111111|11111100|111          7ffffe7  [27]
    (245)  |11111111|11111111|11111101|000          7ffffe8  [27]
    (246)  |11111111|11111111|11111101|001          7ffffe9  [27]
    (247)  |11111111|11111111|11111101|010          7ffffea  [27]
    (248)  |11111111|11111111|11111101|011          7ffffeb  [27]
    (249)  |11111111|11111111|11111111|1110         ffffffe  [28]
    (250)  |11111111|11111111|11111101|100          7ffffec  [27]
    (251)  |11111111|11111111|11111101|101          7ffffed  [27]
    (252)  |11111111|11111111|11111101|110          7ffffee  [27]
    (253)  |11111111|11111111|11111101|111          7ffffef  [27]
    (254)  |11111111|11111111|11111110|000          7fffff0  [27]
    (255)  |11111111|11111111|11111011|10           3ffffee  [26]
EOS (256)  |11111111|11111111|11111111|111111      3fffffff  [30]
```

# [C. 示例](https://http2.github.io/http2-spec/compression.html#examples)

本附录包含了一些例子，覆盖了整数编码，头部字段表示，及请求和响应的整个头部字段列表的编码，使用或未使用Huffman编码的情况。

## [C.1 整数表示示例](https://http2.github.io/http2-spec/compression.html#integer.representation.examples)

这一节展示整数值表示的细节 (见 [Section 5.1](https://http2.github.io/http2-spec/compression.html#integer.representation))。

### [C.1.1 示例 1：使用5位前缀编码10](https://http2.github.io/http2-spec/compression.html#integer.representation.example1)

使用5位前缀编码10。

* 10小于31 (2^5 - 1)，使用5位前缀表示。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| X | X | X | 0 | 1 | 0 | 1 | 0 |   10 stored on 5 bits
+---+---+---+---+---+---+---+---+
```

### [C.1.2 示例 2：使用5位前缀编码1337](https://http2.github.io/http2-spec/compression.html#integer.representation.example2)

使用5位前缀编码值I=1337。

```
1337大于31 (2^5 - 1)。
       5位前缀用它的最大值 (31) 填充。
I = 1337 - (25 - 1) = 1306.
       I (1306) 大于等于128，因而while循环体执行：
              I % 128 == 26
              26 + 128 == 154
              154 用8位编码为：10011010
              I 被设置为 10 (1306 / 128 == 10)
              I 不再大于等于128,因而while循环终止。
       I, 现在是 10, 被编码为8位值：00001010.
过程结束

```

### [C.1.3 示例 3: 以字节边界为起始编码42](https://http2.github.io/http2-spec/compression.html#integer.representation.example3)

以字节边界为起始编码值42。这暗示将使用一个8位前缀。

* 42 小于 255 (2^8 - 1)，使用8位前缀表示.

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 | 0 | 1 | 0 | 1 | 0 |   42 stored on 8 bits
+---+---+---+---+---+---+---+---+
```

##  [C.2 头部字段表示示例](https://http2.github.io/http2-spec/compression.html#header.field.representation.examples)

本节显示了几个独立的表示的例子。

### [C.2.1 有索引的字面量头部字段](https://http2.github.io/http2-spec/compression.html#rfc.section.C.2.1)

头部字段表示使用一个字面量名字和一个字面量值。头部字段被添加进动态表。

要编码的头部列表：

```
custom-key: custom-header
```

编码数据的十六进制转储：

```
400a 6375 7374 6f6d 2d6b 6579 0d63 7573 | @.custom-key.cus
746f 6d2d 6865 6164 6572                | tom-header
```

解码过程：

```
40                                      | == Literal indexed ==
0a                                      |   Literal name (len = 10)
6375 7374 6f6d 2d6b 6579                | custom-key
0d                                      |   Literal value (len = 13)
6375 7374 6f6d 2d68 6561 6465 72        | custom-header
                                        | -> custom-key:
                                        |   custom-header
```

动态表（解码后）：

```
[  1] (s =  55) custom-key: custom-header
      Table size:  55
```

解码后的头部列表：

```
custom-key: custom-header
```

### [C.2.2 无索引的字面量头部字段](https://http2.github.io/http2-spec/compression.html#rfc.section.C.2.2)

头部字段表示使用一个索引的名字和一个字面量值。头部字段不加进动态表。

要编码的头部列表：

```
:path: /sample/path
```

编码数据的十六进制转储：

```
040c 2f73 616d 706c 652f 7061 7468      | ../sample/path
```

解码过程：

```
04                                      | == Literal not indexed ==
                                        |   Indexed name (idx = 4)
                                        |     :path
0c                                      |   Literal value (len = 12)
2f73 616d 706c 652f 7061 7468           | /sample/path
                                        | -> :path: /sample/path
```

动态表（解码后）：空的

解码后的头部列表：

```
:path: /sample/path
```

### [C.2.3 从不索引的字面量头部字段](https://http2.github.io/http2-spec/compression.html#rfc.section.C.2.3)

头部字段表示使用一个字面量名字和一个字面量值。头部字段不加进动态表，而且如果由一个中介重编码的话必须使用相同的表示。

要编码的头部列表：

```
password: secre
```

编码数据的十六进制转储：

```
1008 7061 7373 776f 7264 0673 6563 7265 | ..password.secre
74                                      | t
```

解码过程：

```
10                                      | == Literal never indexed ==
08                                      |   Literal name (len = 8)
7061 7373 776f 7264                     | password
06                                      |   Literal value (len = 6)
7365 6372 6574                          | secret
                                        | -> password: secret
```

动态表（解码后）：空的

解码后的头部列表：

```
password: secret
```

### [C.2.4 索引的头部字段](https://http2.github.io/http2-spec/compression.html#rfc.section.C.2.4)

头部字段表示使用静态表中一个索引的头部字段。

要编码的头部列表：

```
:method: GET
```

编码数据的十六进制转储：

```
82                                      | .
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
```

动态表（解码后）：空的

解码后的头部列表：

```
:method: GET
```

## [C.3 没有Huffman编码的请求的例子](https://http2.github.io/http2-spec/compression.html#request.examples.without.huffman.coding)

这一节展示了相同连接上对应于HTTP请求的多个连续的头部列表。

### [C.3.1 第一个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.3.1)

要编码的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
```

编码数据的十六进制转储：

```
8286 8441 0f77 7777 2e65 7861 6d70 6c65 | ...A.www.example
2e63 6f6d                               | .com
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
86                                      | == Indexed - Add ==
                                        |   idx = 6
                                        | -> :scheme: http
84                                      | == Indexed - Add ==
                                        |   idx = 4
                                        | -> :path: /
41                                      | == Literal indexed ==
                                        |   Indexed name (idx = 1)
                                        |     :authority
0f                                      |   Literal value (len = 15)
7777 772e 6578 616d 706c 652e 636f 6d   | www.example.com
                                        | -> :authority: 
                                        |   www.example.com
```

动态表（解码后）：

```
[  1] (s =  57) :authority: www.example.com
      Table size:  57
```

解码后的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
```

### [C.3.2 第二个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.3.2)

要编码的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
cache-control: no-cache
```

编码数据的十六进制转储：

```
8286 84be 5808 6e6f 2d63 6163 6865      | ....X.no-cache
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
86                                      | == Indexed - Add ==
                                        |   idx = 6
                                        | -> :scheme: http
84                                      | == Indexed - Add ==
                                        |   idx = 4
                                        | -> :path: /
be                                      | == Indexed - Add ==
                                        |   idx = 62
                                        | -> :authority:
                                        |   www.example.com
58                                      | == Literal indexed ==
                                        |   Indexed name (idx = 24)
                                        |     cache-control
08                                      |   Literal value (len = 8)
6e6f 2d63 6163 6865                     | no-cache
                                        | -> cache-control: no-cache
```

动态表（解码后）：

```
[  1] (s =  53) cache-control: no-cache
[  2] (s =  57) :authority: www.example.com
      Table size: 110
```

解码后的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
cache-control: no-cache
```

### [C.3.3 第三个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.3.3)

要编码的头部列表：

```
:method: GET
:scheme: https
:path: /index.html
:authority: www.example.com
custom-key: custom-value
```

编码数据的十六进制转储：

```
8287 85bf 400a 6375 7374 6f6d 2d6b 6579 | ....@.custom-key
0c63 7573 746f 6d2d 7661 6c75 65        | .custom-value
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
87                                      | == Indexed - Add ==
                                        |   idx = 7
                                        | -> :scheme: https
85                                      | == Indexed - Add ==
                                        |   idx = 5
                                        | -> :path: /index.html
bf                                      | == Indexed - Add ==
                                        |   idx = 63
                                        | -> :authority:
                                        |   www.example.com
40                                      | == Literal indexed ==
0a                                      |   Literal name (len = 10)
6375 7374 6f6d 2d6b 6579                | custom-key
0c                                      |   Literal value (len = 12)
6375 7374 6f6d 2d76 616c 7565           | custom-value
                                        | -> custom-key:
                                        |   custom-value
```

动态表（解码后）：

```
[  1] (s =  54) custom-key: custom-value
[  2] (s =  53) cache-control: no-cache
[  3] (s =  57) :authority: www.example.com
      Table size: 164
```

解码后的头部列表：

```
:method: GET
:scheme: https
:path: /index.html
:authority: www.example.com
custom-key: custom-value
```

## [C.4 Huffman编码的请求示例](https://http2.github.io/http2-spec/compression.html#request.examples.with.huffman.coding)

这一节展示了与前一节相同的例子，但为字面值使用了Huffman编码。

### [C.4.1 第一个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.4.1)

要编码的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
```

编码数据的十六进制转储：

```
8286 8441 8cf1 e3c2 e5f2 3a6b a0ab 90f4 | ...A......:k....
ff                                      | .
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
86                                      | == Indexed - Add ==
                                        |   idx = 6
                                        | -> :scheme: http
84                                      | == Indexed - Add ==
                                        |   idx = 4
                                        | -> :path: /
41                                      | == Literal indexed ==
                                        |   Indexed name (idx = 1)
                                        |     :authority
8c                                      |   Literal value (len = 12)
                                        |     Huffman encoded:
f1e3 c2e5 f23a 6ba0 ab90 f4ff           | .....:k.....
                                        |     Decoded:
                                        | www.example.com
                                        | -> :authority:
                                        |   www.example.com
```

动态表（解码后）：

```
[  1] (s =  57) :authority: www.example.com
      Table size:  57
```

解码后的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
```

### [C.4.2 第二个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.4.2)

要编码的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
cache-control: no-cache
```

编码数据的十六进制转储：

```
8286 84be 5886 a8eb 1064 9cbf           | ....X....d..
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
86                                      | == Indexed - Add ==
                                        |   idx = 6
                                        | -> :scheme: http
84                                      | == Indexed - Add ==
                                        |   idx = 4
                                        | -> :path: /
be                                      | == Indexed - Add ==
                                        |   idx = 62
                                        | -> :authority:
                                        |   www.example.com
58                                      | == Literal indexed ==
                                        |   Indexed name (idx = 24)
                                        |     cache-control
86                                      |   Literal value (len = 6)
                                        |     Huffman encoded:
a8eb 1064 9cbf                          | ...d..
                                        |     Decoded:
                                        | no-cache
                                        | -> cache-control: no-cache
```

动态表（解码后）：

```
[  1] (s =  53) cache-control: no-cache
[  2] (s =  57) :authority: www.example.com
      Table size: 110
```

解码后的头部列表：

```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
cache-control: no-cache
```

### [C.4.3 第三个请求](https://http2.github.io/http2-spec/compression.html#rfc.section.C.4.3)

要编码的头部列表：

```
:method: GET
:scheme: https
:path: /index.html
:authority: www.example.com
custom-key: custom-value
```

编码数据的十六进制转储：

```
8287 85bf 4088 25a8 49e9 5ba9 7d7f 8925 | ....@.%.I.[.}..%
a849 e95b b8e8 b4bf                     | .I.[....
```

解码过程：

```
82                                      | == Indexed - Add ==
                                        |   idx = 2
                                        | -> :method: GET
87                                      | == Indexed - Add ==
                                        |   idx = 7
                                        | -> :scheme: https
85                                      | == Indexed - Add ==
                                        |   idx = 5
                                        | -> :path: /index.html
bf                                      | == Indexed - Add ==
                                        |   idx = 63
                                        | -> :authority:
                                        |   www.example.com
40                                      | == Literal indexed ==
88                                      |   Literal name (len = 8)
                                        |     Huffman encoded:
25a8 49e9 5ba9 7d7f                     | %.I.[.}.
                                        |     Decoded:
                                        | custom-key
89                                      |   Literal value (len = 9)
                                        |     Huffman encoded:
25a8 49e9 5bb8 e8b4 bf                  | %.I.[....
                                        |     Decoded:
                                        | custom-value
                                        | -> custom-key:
                                        |   custom-value
```

动态表（解码后）：

```
[  1] (s =  54) custom-key: custom-value
[  2] (s =  53) cache-control: no-cache
[  3] (s =  57) :authority: www.example.com
      Table size: 164
```

解码后的头部列表：

```
:method: GET
:scheme: https
:path: /index.html
:authority: www.example.com
custom-key: custom-value
```

## [C.5 没有Huffman编码的响应示例](https://http2.github.io/http2-spec/compression.html#response.examples.without.huffman.coding)

这一节展示了一些相同连接上连续的对应于HTTP响应的头部列表。HTTP/2 设置参数 SETTINGS_HEADER_TABLE_SIZE 被设置为了256字节的值，这使得某些 **驱逐(eviction)**  发生了。


This section shows several consecutive header lists, corresponding to HTTP responses, on the same connection. The HTTP/2 setting parameter SETTINGS_HEADER_TABLE_SIZE is set to the value of 256 octets, causing some evictions to occur.

### [C.5.1 第一个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.5.1)

要编码的头部列表：

```
:status: 302
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

编码数据的十六进制转储：

```
4803 3330 3258 0770 7269 7661 7465 611d | H.302X.privatea.
4d6f 6e2c 2032 3120 4f63 7420 3230 3133 | Mon, 21 Oct 2013
2032 303a 3133 3a32 3120 474d 546e 1768 |  20:13:21 GMTn.h
7474 7073 3a2f 2f77 7777 2e65 7861 6d70 | ttps://www.examp
6c65 2e63 6f6d                          | le.com
```

解码过程：

```
48                                      | == Literal indexed ==
                                        |   Indexed name (idx = 8)
                                        |     :status
03                                      |   Literal value (len = 3)
3330 32                                 | 302
                                        | -> :status: 302
58                                      | == Literal indexed ==
                                        |   Indexed name (idx = 24)
                                        |     cache-control
07                                      |   Literal value (len = 7)
7072 6976 6174 65                       | private
                                        | -> cache-control: private
61                                      | == Literal indexed ==
                                        |   Indexed name (idx = 33)
                                        |     date
1d                                      |   Literal value (len = 29)
4d6f 6e2c 2032 3120 4f63 7420 3230 3133 | Mon, 21 Oct 2013
2032 303a 3133 3a32 3120 474d 54        |  20:13:21 GMT
                                        | -> date: Mon, 21 Oct 2013
                                        |   20:13:21 GMT
6e                                      | == Literal indexed ==
                                        |   Indexed name (idx = 46)
                                        |     location
17                                      |   Literal value (len = 23)
6874 7470 733a 2f2f 7777 772e 6578 616d | https://www.exam
706c 652e 636f 6d                       | ple.com
                                        | -> location:
                                        |   https://www.example.com
```

动态表（解码后）：

```
[  1] (s =  63) location: https://www.example.com
[  2] (s =  65) date: Mon, 21 Oct 2013 20:13:21 GMT
[  3] (s =  52) cache-control: private
[  4] (s =  42) :status: 302
      Table size: 222
```

解码后的头部列表：

```
:status: 302
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

### [C.5.2 第二个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.5.2)

`(":status", "302")` 头部字段被从动态表中驱逐以腾出空间而允许添加 `(":status", "307")` 头部字段。

要编码的头部列表：

```
:status: 307
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

编码数据的十六进制转储：

```
4803 3330 37c1 c0bf                     | H.307...
```

解码过程：

```
48                                      | == Literal indexed ==
                                        |   Indexed name (idx = 8)
                                        |     :status
03                                      |   Literal value (len = 3)
3330 37                                 | 307
                                        | - evict: :status: 302
                                        | -> :status: 307
c1                                      | == Indexed - Add ==
                                        |   idx = 65
                                        | -> cache-control: private
c0                                      | == Indexed - Add ==
                                        |   idx = 64
                                        | -> date: Mon, 21 Oct 2013
                                        |   20:13:21 GMT
bf                                      | == Indexed - Add ==
                                        |   idx = 63
                                        | -> location:
                                        |   https://www.example.com
```

动态表（解码后）：

```
[  1] (s =  42) :status: 307
[  2] (s =  63) location: https://www.example.com
[  3] (s =  65) date: Mon, 21 Oct 2013 20:13:21 GMT
[  4] (s =  52) cache-control: private
      Table size: 222
```

解码后的头部列表：

```
:status: 307
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

### [C.5.3 第三个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.5.3)

一些头部字段在处理该头部列表期间被从动态表 **驱逐(eviction)** 了。

要编码的头部列表：

```
:status: 200
cache-control: private
date: Mon, 21 Oct 2013 20:13:22 GMT
location: https://www.example.com
content-encoding: gzip
set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU; max-age=3600; version=1
```

编码数据的十六进制转储：

```
88c1 611d 4d6f 6e2c 2032 3120 4f63 7420 | ..a.Mon, 21 Oct
3230 3133 2032 303a 3133 3a32 3220 474d | 2013 20:13:22 GM
54c0 5a04 677a 6970 7738 666f 6f3d 4153 | T.Z.gzipw8foo=AS
444a 4b48 514b 425a 584f 5157 454f 5049 | DJKHQKBZXOQWEOPI
5541 5851 5745 4f49 553b 206d 6178 2d61 | UAXQWEOIU; max-a
6765 3d33 3630 303b 2076 6572 7369 6f6e | ge=3600; version
3d31                                    | =1
```

解码过程：

```
88                                      | == Indexed - Add ==
                                        |   idx = 8
                                        | -> :status: 200
c1                                      | == Indexed - Add ==
                                        |   idx = 65
                                        | -> cache-control: private
61                                      | == Literal indexed ==
                                        |   Indexed name (idx = 33)
                                        |     date
1d                                      |   Literal value (len = 29)
4d6f 6e2c 2032 3120 4f63 7420 3230 3133 | Mon, 21 Oct 2013
2032 303a 3133 3a32 3220 474d 54        |  20:13:22 GMT
                                        | - evict: cache-control:
                                        |   private
                                        | -> date: Mon, 21 Oct 2013
                                        |   20:13:22 GMT
c0                                      | == Indexed - Add ==
                                        |   idx = 64
                                        | -> location: 
                                        |   https://www.example.com
5a                                      | == Literal indexed ==
                                        |   Indexed name (idx = 26)
                                        |     content-encoding
04                                      |   Literal value (len = 4)
677a 6970                               | gzip
                                        | - evict: date: Mon, 21 Oct 
                                        |    2013 20:13:21 GMT
                                        | -> content-encoding: gzip
77                                      | == Literal indexed ==
                                        |   Indexed name (idx = 55)
                                        |     set-cookie
38                                      |   Literal value (len = 56)
666f 6f3d 4153 444a 4b48 514b 425a 584f | foo=ASDJKHQKBZXO
5157 454f 5049 5541 5851 5745 4f49 553b | QWEOPIUAXQWEOIU;
206d 6178 2d61 6765 3d33 3630 303b 2076 |  max-age=3600; v
6572 7369 6f6e 3d31                     | ersion=1
                                        | - evict: location:
                                        |   https://www.example.com
                                        | - evict: :status: 307
                                        | -> set-cookie: foo=ASDJKHQ
                                        |   KBZXOQWEOPIUAXQWEOIU; ma
                                        |   x-age=3600; version=1
```

动态表（解码后）：

```
[  1] (s =  98) set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU;
                 max-age=3600; version=1
[  2] (s =  52) content-encoding: gzip
[  3] (s =  65) date: Mon, 21 Oct 2013 20:13:22 GMT
      Table size: 215
```

解码后的头部列表：

```
:status: 200
cache-control: private
date: Mon, 21 Oct 2013 20:13:22 GMT
location: https://www.example.com
content-encoding: gzip
set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU; max-age=3600; version=1
```

## [C.6 Huffman编码的响应示例](https://http2.github.io/http2-spec/compression.html#response.examples.with.huffman.coding)

这一节展示了如前一节相同的示例，但它对字面值使用了Huffman编码。HTTP/2 设置参数 SETTINGS_HEADER_TABLE_SIZE 被设置为了256字节的值，这使得某些 **驱逐(eviction)**  发生了。 **驱逐(eviction)** 机制使用了解码的字面值的长度，因此发生的 **驱逐(eviction)** 与前一节的相同。

### [C.6.1 第一个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.6.1)

要编码的头部列表：

```
:status: 302
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

编码数据的十六进制转储：

```
4882 6402 5885 aec3 771a 4b61 96d0 7abe | H.d.X...w.Ka..z.
9410 54d4 44a8 2005 9504 0b81 66e0 82a6 | ..T.D. .....f...
2d1b ff6e 919d 29ad 1718 63c7 8f0b 97c8 | -..n..)...c.....
e9ae 82ae 43d3                          | ....C.
```

解码过程：

```
48                                      | == Literal indexed ==
                                        |   Indexed name (idx = 8)
                                        |     :status
82                                      |   Literal value (len = 2)
                                        |     Huffman encoded:
6402                                    | d.
                                        |     Decoded:
                                        | 302
                                        | -> :status: 302
58                                      | == Literal indexed ==
                                        |   Indexed name (idx = 24)
                                        |     cache-control
85                                      |   Literal value (len = 5)
                                        |     Huffman encoded:
aec3 771a 4b                            | ..w.K
                                        |     Decoded:
                                        | private
                                        | -> cache-control: private
61                                      | == Literal indexed ==
                                        |   Indexed name (idx = 33)
                                        |     date
96                                      |   Literal value (len = 22)
                                        |     Huffman encoded:
d07a be94 1054 d444 a820 0595 040b 8166 | .z...T.D. .....f
e082 a62d 1bff                          | ...-..
                                        |     Decoded:
                                        | Mon, 21 Oct 2013 20:13:21
                                        | GMT
                                        | -> date: Mon, 21 Oct 2013
                                        |   20:13:21 GMT
6e                                      | == Literal indexed ==
                                        |   Indexed name (idx = 46)
                                        |     location
91                                      |   Literal value (len = 17)
                                        |     Huffman encoded:
9d29 ad17 1863 c78f 0b97 c8e9 ae82 ae43 | .)...c.........C
d3                                      | .
                                        |     Decoded:
                                        | https://www.example.com
                                        | -> location:
                                        |   https://www.example.com
```

动态表（解码后）：

```
[  1] (s =  63) location: https://www.example.com
[  2] (s =  65) date: Mon, 21 Oct 2013 20:13:21 GMT
[  3] (s =  52) cache-control: private
[  4] (s =  42) :status: 302
      Table size: 222
```

解码后的头部列表：

```
:status: 302
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

### [C.6.2 第二个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.6.2)

`(":status", "302")` 头部字段被从动态表驱逐出来腾出空间以允许添加 `(":status", "307")` 头部字段。

要编码的头部列表：

```
:status: 307
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

编码数据的十六进制转储：

```
4883 640e ffc1 c0bf                     | H.d.....
```

解码过程：

```
48                                      | == Literal indexed ==
                                        |   Indexed name (idx = 8)
                                        |     :status
83                                      |   Literal value (len = 3)
                                        |     Huffman encoded:
640e ff                                 | d..
                                        |     Decoded:
                                        | 307
                                        | - evict: :status: 302
                                        | -> :status: 307
c1                                      | == Indexed - Add ==
                                        |   idx = 65
                                        | -> cache-control: private
c0                                      | == Indexed - Add ==
                                        |   idx = 64
                                        | -> date: Mon, 21 Oct 2013
                                        |   20:13:21 GMT
bf                                      | == Indexed - Add ==
                                        |   idx = 63
                                        | -> location:
                                        |   https://www.example.com
```

动态表（解码后）：

```
[  1] (s =  42) :status: 307
[  2] (s =  63) location: https://www.example.com
[  3] (s =  65) date: Mon, 21 Oct 2013 20:13:21 GMT
[  4] (s =  52) cache-control: private
      Table size: 222
```

解码后的头部列表：

```
:status: 307
cache-control: private
date: Mon, 21 Oct 2013 20:13:21 GMT
location: https://www.example.com
```

### [C.6.3 第三个响应](https://http2.github.io/http2-spec/compression.html#rfc.section.C.6.3)

一些头部字段是在头部列表处理器间从动态表中逐出的。

要编码的头部列表：

```
:status: 200
cache-control: private
date: Mon, 21 Oct 2013 20:13:22 GMT
location: https://www.example.com
content-encoding: gzip
set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU; max-age=3600; version=1
```

编码数据的十六进制转储：

```
88c1 6196 d07a be94 1054 d444 a820 0595 | ..a..z...T.D. ..
040b 8166 e084 a62d 1bff c05a 839b d9ab | ...f...-...Z....
77ad 94e7 821d d7f2 e6c7 b335 dfdf cd5b | w..........5...[
3960 d5af 2708 7f36 72c1 ab27 0fb5 291f | 9`..'..6r..'..).
9587 3160 65c0 03ed 4ee5 b106 3d50 07   | ..1`e...N...=P.
```

解码过程：

```
88                                      | == Indexed - Add ==
                                        |   idx = 8
                                        | -> :status: 200
c1                                      | == Indexed - Add ==
                                        |   idx = 65
                                        | -> cache-control: private
61                                      | == Literal indexed ==
                                        |   Indexed name (idx = 33)
                                        |     date
96                                      |   Literal value (len = 22)
                                        |     Huffman encoded:
d07a be94 1054 d444 a820 0595 040b 8166 | .z...T.D. .....f
e084 a62d 1bff                          | ...-..
                                        |     Decoded:
                                        | Mon, 21 Oct 2013 20:13:22
                                        | GMT
                                        | - evict: cache-control:
                                        |   private
                                        | -> date: Mon, 21 Oct 2013 
                                        |   20:13:22 GMT
c0                                      | == Indexed - Add ==
                                        |   idx = 64
                                        | -> location:
                                        |   https://www.example.com
5a                                      | == Literal indexed ==
                                        |   Indexed name (idx = 26)
                                        |     content-encoding
83                                      |   Literal value (len = 3)
                                        |     Huffman encoded:
9bd9 ab                                 | ...
                                        |     Decoded:
                                        | gzip
                                        | - evict: date: Mon, 21 Oct
                                        |    2013 20:13:21 GMT
                                        | -> content-encoding: gzip
77                                      | == Literal indexed ==
                                        |   Indexed name (idx = 55)
                                        |     set-cookie
ad                                      |   Literal value (len = 45)
                                        |     Huffman encoded:
94e7 821d d7f2 e6c7 b335 dfdf cd5b 3960 | .........5...[9`
d5af 2708 7f36 72c1 ab27 0fb5 291f 9587 | ..'..6r..'..)...
3160 65c0 03ed 4ee5 b106 3d50 07        | 1`e...N...=P.
                                        |     Decoded:
                                        | foo=ASDJKHQKBZXOQWEOPIUAXQ
                                        | WEOIU; max-age=3600; versi
                                        | on=1
                                        | - evict: location:
                                        |   https://www.example.com
                                        | - evict: :status: 307
                                        | -> set-cookie: foo=ASDJKHQ
                                        |   KBZXOQWEOPIUAXQWEOIU; ma
                                        |   x-age=3600; version=1
```

动态表（解码后）：

```
[  1] (s =  98) set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU;
                 max-age=3600; version=1
[  2] (s =  52) content-encoding: gzip
[  3] (s =  65) date: Mon, 21 Oct 2013 20:13:22 GMT
      Table size: 215
```

解码后的头部列表：

```
:status: 200
cache-control: private
date: Mon, 21 Oct 2013 20:13:22 GMT
location: https://www.example.com
content-encoding: gzip
set-cookie: foo=ASDJKHQKBZXOQWEOPIUAXQWEOIU; max-age=3600; version=1
```

# 鸣谢

本规范包括以下个人的实质性意见：

* Mike Bishop，Jeff Pinner，Julian Reschke，和 Martin Thomson (实质性的编辑贡献).

* Johnny Graettinger (Huffman code statistics).

# [作者地址](https://http2.github.io/http2-spec/compression.html#rfc.authors)

```
**Roberto Peon**
Google, Inc
Email: fenix@google.com

**Hervé Ruellan**
Canon CRF
Email: herve.ruellan@crf.canon.fr
```
