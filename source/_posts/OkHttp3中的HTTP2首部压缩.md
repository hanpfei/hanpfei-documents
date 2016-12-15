---
title: HPACK：HTTP/2的首部压缩 (RFC7541)
date: 2016-10-29 13:46:49
categories: HTTP2相关规范
---

当前网络环境中，同一个页面发出几十个HTTP请求已经是司空见惯的事情了。在HTTP/1.1中，请求之间完全相互独立，使得请求中冗余的首部字段不必要地浪费了大量的网络带宽，并增加了网络延时。以对某站点的一次页面访问为例，直观地看一下这种状况：

![Header 1](http://upload-images.jianshu.io/upload_images/1315506-b172bdb63e325d50.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Header 2](http://upload-images.jianshu.io/upload_images/1315506-7b62c9574493209c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图，同一个页面中对两个资源的请求，请求中的头部字段绝大部分是完全相同的。"User-Agent" 等头部字段通常还会消耗大量的带宽。

首部压缩正是为了解决这样的问题而设计。

首部压缩是HTTP/2中一个非常重要的特性，它大大减少了网络中HTTP请求/响应头部传输所需的带宽。HTTP/2的首部压缩，主要从两个方面实现，一是首部表示，二是请求间首部字段内容的复用。

# 首部表示
在HTTP中，首部字段是一个名值队，所有的首部字段组成首部字段列表。在HTTP/1.x中，首部字段都被表示为字符串，一行一行的首部字段字符串组成首部字段列表。而在HTTP/2的首部压缩HPACK算法中，则有着不同的表示方法。

HPACK算法表示的对象，主要有原始数据类型的整型值和字符串，头部字段，以及头部字段列表。

## 整数的表示
在HPACK中，整数用于表示 ***头部字段的名字的索引***，***头部字段索引*** 或 ***字符串长度***。整数的表示可在字节内的任何位置开始。但为了处理上的优化，整数的表示总是在字节的结尾处结束。

整数由两部分表示：填满当前字节的前缀，以及在前缀不足以表示整数时的一个可选字节列表。如果整数值足够小，比如，小于2^N-1，那么把它编码进前缀即可，而不需要额外的空间。如：
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | ? |       Value       |
+---+---+---+-------------------+
```
在这个图中，前缀有5位，而要表示的数足够小，因此无需更多空间就可以表示整数了。

当前缀不足以表示整数时，前缀的所有位被置为1，再将值减去2^N-1之后用一个或多个字节编码。每个字节的最高有效位被用作连续标记：除列表的最后一个字节外，该位的值都被设为1。字节中剩余的位被用于编码减小后的值。
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
要由字节列表解码出整数值，首先需要将列表中的字节顺序反过来。然后，移除每个字节的最高有效位。连接字节的剩余位，再将结果加2^N-1获得整数值。

前缀大小N，总是在1到8之间。从字节边界处开始编码的整数值其前缀为8位。

这种整数表示法允许编码无限大的值。

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
***encode (I % 128 + 128) on 8 bits*** 一行中，加上128的意思是，最高有效位是1。如果要编码的整数值等于 (2^N - 1)，则用前缀和紧跟在前缀后面的值位0的一个字节来表示。

OkHttp中，这个算法的实现在 ***okhttp3.internal.http2.Hpack.Writer*** 中：
```
    // http://tools.ietf.org/html/draft-ietf-httpbis-header-compression-12#section-4.1.1
    void writeInt(int value, int prefixMask, int bits) {
      // Write the raw value for a single byte value.
      if (value < prefixMask) {
        out.writeByte(bits | value);
        return;
      }

      // Write the mask to start a multibyte value.
      out.writeByte(bits | prefixMask);
      value -= prefixMask;

      // Write 7 bits at a time 'til we're done.
      while (value >= 0x80) {
        int b = value & 0x7f;
        out.writeByte(b | 0x80);
        value >>>= 7;
      }
      out.writeByte(value);
    }
```
这里给最高有效位置 1 的方法就不是加上128，而是与0x80执行或操作。

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
***decode I from the next N bits*** 这一行等价于一个赋值语句 ****I = byteValue & (2^N - 1)***

OkHttp中，这个算法的实现在 ***okhttp3.internal.http2.Hpack.Reader*** ：
```
    int readInt(int firstByte, int prefixMask) throws IOException {
      int prefix = firstByte & prefixMask;
      if (prefix < prefixMask) {
        return prefix; // This was a single byte value.
      }

      // This is a multibyte value. Read 7 bits at a time.
      int result = prefixMask;
      int shift = 0;
      while (true) {
        int b = readByte();
        if ((b & 0x80) != 0) { // Equivalent to (b >= 128) since b is in [0..255].
          result += (b & 0x7f) << shift;
          shift += 7;
        } else {
          result += b << shift; // Last byte.
          break;
        }
      }
      return result;
    }
```
尽管HPACK的整数表示方法可以表示无限大的数，但实际的实现中并不会将整数当做无限大的整数来处理。

## 字符串字面量的编码
头部字段名和头部字段值可使用字符串字面量表示。字符串字面量有两种表示方式，一种是直接用UTF-8这样的字符串编码方式表示，另一种是将字符串编码用Huffman 码表示。 字符串表示的格式如下：
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| H |    String Length (7+)     |
+---+---------------------------+
|  String Data (Length octets)  |
+-------------------------------+
```
先是标记位 **H** + 字符串长度，然后是字符串的实际数据。各部分说明如下：

* **H：** 一位的标记，指示字符串的字节是否为Huffman编码。
* **字符串长度：** 编码字符串字面量的字节数，一个整数，编码方式可以参考前面 **整数的表示** 的部分，一个7位前缀的整数编码。
* **字符串数据：** 字符串的实际数据。如果H是'0'，则数据是字符串字面量的原始字节。如果H是'1'，则数据是字符串字面量的Huffman编码。

在OkHttp3中，总是会使用直接的字符串编码，而不是Huffman编码， ***okhttp3.internal.http2.Hpack.Writer*** 中编码字符串的过程如下：
```
    void writeByteString(ByteString data) throws IOException {
      writeInt(data.size(), PREFIX_7_BITS, 0);
      out.write(data);
    }
```

OkHttp中，解码字符串在 ***okhttp3.internal.http2.Hpack.Reader*** 中实现：
```
    /** Reads a potentially Huffman encoded byte string. */
    ByteString readByteString() throws IOException {
      int firstByte = readByte();
      boolean huffmanDecode = (firstByte & 0x80) == 0x80; // 1NNNNNNN
      int length = readInt(firstByte, PREFIX_7_BITS);

      if (huffmanDecode) {
        return ByteString.of(Huffman.get().decode(source.readByteArray(length)));
      } else {
        return source.readByteString(length);
      }
    }
```
字符串编码没有使用Huffman编码时，解码过程比较简单，而使用了Huffman编码时会借助于***Huffman***类来解码。

Huffman编码是一种变长字节编码，对于使用频率高的字节，使用更少的位数，对于使用频率低的字节则使用更多的位数。每个字节的Huffman码是根据统计经验值分配的。为每个字节分配Huffman码的方法可以参考 [哈夫曼（huffman）树和哈夫曼编码](http://www.cnblogs.com/kubixuesheng/p/4397798.html) 。

### 哈夫曼树的构造
***Huffman*** 类被设计为一个单例类。对象在创建时构造一个哈夫曼树以用于编码和解码操作。
```
  private static final Huffman INSTANCE = new Huffman();

  public static Huffman get() {
    return INSTANCE;
  }

  private final Node root = new Node();

  private Huffman() {
    buildTree();
  }
......

  private void buildTree() {
    for (int i = 0; i < CODE_LENGTHS.length; i++) {
      addCode(i, CODES[i], CODE_LENGTHS[i]);
    }
  }

  private void addCode(int sym, int code, byte len) {
    Node terminal = new Node(sym, len);

    Node current = root;
    while (len > 8) {
      len -= 8;
      int i = ((code >>> len) & 0xFF);
      if (current.children == null) {
        throw new IllegalStateException("invalid dictionary: prefix not unique");
      }
      if (current.children[i] == null) {
        current.children[i] = new Node();
      }
      current = current.children[i];
    }

    int shift = 8 - len;
    int start = (code << shift) & 0xFF;
    int end = 1 << shift;
    for (int i = start; i < start + end; i++) {
      current.children[i] = terminal;
    }
  }
......

  private static final class Node {

    // Null if terminal.
    private final Node[] children;

    // Terminal nodes have a symbol.
    private final int symbol;

    // Number of bits represented in the terminal node.
    private final int terminalBits;

    /** Construct an internal node. */
    Node() {
      this.children = new Node[256];
      this.symbol = 0; // Not read.
      this.terminalBits = 0; // Not read.
    }

    /**
     * Construct a terminal node.
     *
     * @param symbol symbol the node represents
     * @param bits length of Huffman code in bits
     */
    Node(int symbol, int bits) {
      this.children = null;
      this.symbol = symbol;
      int b = bits & 0x07;
      this.terminalBits = b == 0 ? 8 : b;
    }
  }
```
OkHttp3中的 哈夫曼树 并不是一个二叉树，它的每个节点最多都可以有256个字节点。OkHttp3用这种方式来优化Huffman编码解码的效率。用一个图来表示，将是下面这个样子的：

![Huffman Tree](http://upload-images.jianshu.io/upload_images/1315506-b03c870c6cdcd44c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Huffman 编码
```
  void encode(byte[] data, OutputStream out) throws IOException {
    long current = 0;
    int n = 0;

    for (int i = 0; i < data.length; i++) {
      int b = data[i] & 0xFF;
      int code = CODES[b];
      int nbits = CODE_LENGTHS[b];

      current <<= nbits;
      current |= code;
      n += nbits;

      while (n >= 8) {
        n -= 8;
        out.write(((int) (current >> n)));
      }
    }

    if (n > 0) {
      current <<= (8 - n);
      current |= (0xFF >>> n);
      out.write((int) current);
    }
  }
```
逐个字节地编码数据。编码的最后一个字节没有字节对齐时，会在低位填充1。

### Huffman 解码
```

  byte[] decode(byte[] buf) {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    Node node = root;
    int current = 0;
    int nbits = 0;
    for (int i = 0; i < buf.length; i++) {
      int b = buf[i] & 0xFF;
      current = (current << 8) | b;
      nbits += 8;
      while (nbits >= 8) {
        int c = (current >>> (nbits - 8)) & 0xFF;
        node = node.children[c];
        if (node.children == null) {
          // terminal node
          baos.write(node.symbol);
          nbits -= node.terminalBits;
          node = root;
        } else {
          // non-terminal node
          nbits -= 8;
        }
      }
    }

    while (nbits > 0) {
      int c = (current << (8 - nbits)) & 0xFF;
      node = node.children[c];
      if (node.children != null || node.terminalBits > nbits) {
        break;
      }
      baos.write(node.symbol);
      nbits -= node.terminalBits;
      node = root;
    }

    return baos.toByteArray();
  }
```
配合Huffman树的构造过程，分几种情况来看。Huffman码自己对齐时；Huffman码没有字节对齐，最后一个字节的最低有效位包含了数据流中下一个Huffman码的最高有效位；Huffman码没有字节对齐，最后一个字节的最低有效位包含了填充的1。

有兴趣的可以参考其它文档对Huffman编码算法做更多了解。

## 首部字段及首部块的表示
首部字段主要有两种表示方法，分别是索引表示和字面量表示。字面量表示又分为首部字段的名字用索引表示值用字面量表示和名字及值都用字面量表示等方法。

说到用索引表示首部字段，就不能不提一下HPACK的动态表和静态表。

HPACK使用两个表将 头部字段 与 索引 关联起来。 静态表 是预定义的，它包含了常见的头部字段（其中的大多数值为空）。 动态表 是动态的，它可被编码器用于编码重复的头部字段。

静态表由一个预定义的头部字段静态列表组成。它的条目在 HPACK规范的 [附录 A](https://tools.ietf.org/html/rfc7541#appendix-A) 中定义。

动态表由以先进先出顺序维护的 **头部字段列表** 组成。动态表中第一个且最新的条目索引值最低，动态表最旧的条目索引值最高。

动态表最初是空的。条目随着每个头部块的解压而添加。

静态表和动态表被组合为统一的索引地址空间。

在 (1 ~ 静态表的长度(包含)) 之间的索引值指向静态表中的元素。

大于静态表长度的索引值指向动态表中的元素。通过将头部字段的索引减去静态表的长度来查找指向动态表的索引。

对于静态表大小为 s，动态表大小为 k 的情况，下图展示了完整的有效索引地址空间。

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

### 用索引表示头部字段
当一个头部字段的名-值已经包含在了静态表或动态表中时，就可以用一个指向静态表或动态表的索引来表示它了。表示方法如下：
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 1 |        Index (7+)         |
+---+---------------------------+
```
头部字段表示的最高有效位置1，然后用前面看到的表示整数的方法表示索引，即索引是一个7位前缀编码的整数。

### 用字面量表示头部字段
在这种表示法中，头部字段的值是用字面量表示的，但头部字段的名字则不一定。根据名字的表示方法的差异，以及是否将头部字段加进动态表等，而分为多种情况。

#### 增量索引的字面量表示
以这种方法表示的头部字段需要被 加进动态表中。在这种表示方法下，头部字段的值用索引表示时，头部字段的表示如下：
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
头部字段的名字和值都用字面量表示时，表示如下：
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

增量索引的字面量头部字段表示以'01' 的2位模式开始。

如果头部字段名与静态表或动态表中存储的条目的头部字段名匹配，则头部字段名称可用那个条目的索引表示。在这种情况下，条目的索引以一个具有6位前缀的整数 表示。这个值总是非0。否则，头部字段名由一个字符串字面量 表示，使用0值代替6位索引，其后是头部字段名。

两种形式的 **头部字段名表示** 之后是字符串字面量表示的头部字段值。

#### 无索引的字面量头部字段

这种表示方法不改变动态表。头部字段名用索引表示时的头部字段表示如下：
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
头部字段名不用索引表示时的头部字段表示如下：
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
无索引的字面量头部字段表示以'0000' 的4位模式开始，其它方面与 增量索引的字面量表示 类似。

#### 从不索引的字面量头部字段
这种表示方法与 无索引的字面量头部字段 类似，但它主要影响网络中的中间节点。头部字段名用索引表示时的头部字段如：
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
头部字段名不用索引表示时的头部字段如：
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

### 首部列表的表示
各个首部字段表示合并起来形成首部列表。在 okhttp3.internal.framed.Hpack.Writer 的writeHeaders() 中完成编码首部块的动作：
```
    /** This does not use "never indexed" semantics for sensitive headers. */
    // http://tools.ietf.org/html/draft-ietf-httpbis-header-compression-12#section-6.2.3
    void writeHeaders(List<Header> headerBlock) throws IOException {
      if (emitDynamicTableSizeUpdate) {
        if (smallestHeaderTableSizeSetting < maxDynamicTableByteCount) {
          // Multiple dynamic table size updates!
          writeInt(smallestHeaderTableSizeSetting, PREFIX_5_BITS, 0x20);
        }
        emitDynamicTableSizeUpdate = false;
        smallestHeaderTableSizeSetting = Integer.MAX_VALUE;
        writeInt(maxDynamicTableByteCount, PREFIX_5_BITS, 0x20);
      }
      // TODO: implement index tracking
      for (int i = 0, size = headerBlock.size(); i < size; i++) {
        Header header = headerBlock.get(i);
        ByteString name = header.name.toAsciiLowercase();
        ByteString value = header.value;
        Integer staticIndex = NAME_TO_FIRST_INDEX.get(name);
        if (staticIndex != null) {
          // Literal Header Field without Indexing - Indexed Name.
          writeInt(staticIndex + 1, PREFIX_4_BITS, 0);
          writeByteString(value);
        } else {
          int dynamicIndex = Util.indexOf(dynamicTable, header);
          if (dynamicIndex != -1) {
            // Indexed Header.
            writeInt(dynamicIndex - nextHeaderIndex + STATIC_HEADER_TABLE.length, PREFIX_7_BITS,
                0x80);
          } else {
            // Literal Header Field with Incremental Indexing - New Name
            out.writeByte(0x40);
            writeByteString(name);
            writeByteString(value);
            insertIntoDynamicTable(header);
          }
        }
      }
    }
```
HPACK的规范描述了多种头部字段的表示方法，但并没有指明各个表示方法的适用场景。

在OkHttp3中，实现了3种表示头部字段的表示方法：
1. 头部字段名在静态表中，头部字段名用指向静态表的索引表示，值用字面量表示。头部字段无需加入动态表。
2. 头部字段的 名-值 对在动态表中，用指向动态表的索引表示头部字段。
3. 其它情况，用字面量表示头部字段名和值，头部字段需要加入动态表。

如果头部字段的 名-值 对在静态表中，OkHttp3也不会用索引表示。

# 请求间首部字段内容的复用
HPACK中，最重要的优化就是消除请求间冗余的首部字段。在实现上，主要有两个方面，一是前面看到的首部字段的索引表示，另一方面则是动态表的维护。

HTTP/2中数据发送方向和数据接收方向各有一个动态表。通信的双方，一端发送方向的动态表需要与另一端接收方向的动态表保持一致，反之亦然。

HTTP/2的连接复用及请求并发执行指的是逻辑上的并发。由于底层传输还是用的TCP协议，因而，发送方发送数据的顺序，与接收方接收数据的顺序是一致的。

数据发送方在发送一个请求的首部数据时会顺便维护自己的动态表，接收方在收到首部数据时，也需要立马维护自己接收方向的动态表，然后将解码之后的首部字段列表dispatch出去。

如果通信双方同时在进行2个HTTP请求，分别称为Req1和Req2，假设在发送方Req1的头部字段列表先发送，Req2的头部字段后发送。接收方必然先收到Req1的头部字段列表，然后是Req2的。如果接收方在收到Req1的头部字段列表后，没有立即解码，而是等Req2的首部字段列表接收并处理完成之后，再来处理Req1的，则两端的动态表必然是不一致的。

这里来看一下OkHttp3中的动态表维护。

发送方向的动态表，在  okhttp3.internal.framed.Hpack.Writer 中维护。在HTTP/2中，动态表的最大大小在连接建立的初期会进行协商，后面在数据收发过程中也会进行更新。

在编码头部字段列表的 writeHeaders(List<Header> headerBlock) 中，会在需要的时候，将头部字段插入动态表，具体来说，就是在头部字段的名字不在静态表中，同时 名-值对不在动态表中的情况。

将头部字段插入动态表的过程如下：
```
    private void clearDynamicTable() {
      Arrays.fill(dynamicTable, null);
      nextHeaderIndex = dynamicTable.length - 1;
      headerCount = 0;
      dynamicTableByteCount = 0;
    }

    /** Returns the count of entries evicted. */
    private int evictToRecoverBytes(int bytesToRecover) {
      int entriesToEvict = 0;
      if (bytesToRecover > 0) {
        // determine how many headers need to be evicted.
        for (int j = dynamicTable.length - 1; j >= nextHeaderIndex && bytesToRecover > 0; j--) {
          bytesToRecover -= dynamicTable[j].hpackSize;
          dynamicTableByteCount -= dynamicTable[j].hpackSize;
          headerCount--;
          entriesToEvict++;
        }
        System.arraycopy(dynamicTable, nextHeaderIndex + 1, dynamicTable,
            nextHeaderIndex + 1 + entriesToEvict, headerCount);
        Arrays.fill(dynamicTable, nextHeaderIndex + 1, nextHeaderIndex + 1 + entriesToEvict, null);
        nextHeaderIndex += entriesToEvict;
      }
      return entriesToEvict;
    }

    private void insertIntoDynamicTable(Header entry) {
      int delta = entry.hpackSize;

      // if the new or replacement header is too big, drop all entries.
      if (delta > maxDynamicTableByteCount) {
        clearDynamicTable();
        return;
      }

      // Evict headers to the required length.
      int bytesToRecover = (dynamicTableByteCount + delta) - maxDynamicTableByteCount;
      evictToRecoverBytes(bytesToRecover);

      if (headerCount + 1 > dynamicTable.length) { // Need to grow the dynamic table.
        Header[] doubled = new Header[dynamicTable.length * 2];
        System.arraycopy(dynamicTable, 0, doubled, dynamicTable.length, dynamicTable.length);
        nextHeaderIndex = dynamicTable.length - 1;
        dynamicTable = doubled;
      }
      int index = nextHeaderIndex--;
      dynamicTable[index] = entry;
      headerCount++;
      dynamicTableByteCount += delta;
    }
```

动态表占用的空间超出限制时，老的头部字段将被移除。在OkHttp3中，动态表是一个自后向前生长的表。

在数据的接收防线，okhttp3.internal.http2.Http2Reader 的 nextFrame(Handler handler) 会不停从网络读取一帧帧的数据：
```
  public boolean nextFrame(Handler handler) throws IOException {
    try {
      source.require(9); // Frame header size
    } catch (IOException e) {
      return false; // This might be a normal socket close.
    }

      /*  0                   1                   2                   3
       *  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       * +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       * |                 Length (24)                   |
       * +---------------+---------------+---------------+
       * |   Type (8)    |   Flags (8)   |
       * +-+-+-----------+---------------+-------------------------------+
       * |R|                 Stream Identifier (31)                      |
       * +=+=============================================================+
       * |                   Frame Payload (0...)                      ...
       * +---------------------------------------------------------------+
       */
    int length = readMedium(source);
    if (length < 0 || length > INITIAL_MAX_FRAME_SIZE) {
      throw ioException("FRAME_SIZE_ERROR: %s", length);
    }
    byte type = (byte) (source.readByte() & 0xff);
    byte flags = (byte) (source.readByte() & 0xff);
    int streamId = (source.readInt() & 0x7fffffff); // Ignore reserved bit.
    if (logger.isLoggable(FINE)) logger.fine(frameLog(true, streamId, length, type, flags));

    switch (type) {
      case TYPE_DATA:
        readData(handler, length, flags, streamId);
        break;

      case TYPE_HEADERS:
        readHeaders(handler, length, flags, streamId);
        break;
```

读到头部块时，会立即维护本地接收方向的动态表：
```
  private void readHeaders(Handler handler, int length, byte flags, int streamId)
      throws IOException {
    if (streamId == 0) throw ioException("PROTOCOL_ERROR: TYPE_HEADERS streamId == 0");

    boolean endStream = (flags & FLAG_END_STREAM) != 0;

    short padding = (flags & FLAG_PADDED) != 0 ? (short) (source.readByte() & 0xff) : 0;

    if ((flags & FLAG_PRIORITY) != 0) {
      readPriority(handler, streamId);
      length -= 5; // account for above read.
    }

    length = lengthWithoutPadding(length, flags, padding);

    List<Header> headerBlock = readHeaderBlock(length, padding, flags, streamId);

    handler.headers(endStream, streamId, -1, headerBlock);
  }

  private List<Header> readHeaderBlock(int length, short padding, byte flags, int streamId)
      throws IOException {
    continuation.length = continuation.left = length;
    continuation.padding = padding;
    continuation.flags = flags;
    continuation.streamId = streamId;

    // TODO: Concat multi-value headers with 0x0, except COOKIE, which uses 0x3B, 0x20.
    // http://tools.ietf.org/html/draft-ietf-httpbis-http2-17#section-8.1.2.5
    hpackReader.readHeaders();
    return hpackReader.getAndResetHeaderList();
  }
```
okhttp3.internal.http2.Hpack.Reader的readHeaders()如下：
```
  static final class Reader {

    private final List<Header> headerList = new ArrayList<>();
    private final BufferedSource source;

    private final int headerTableSizeSetting;
    private int maxDynamicTableByteCount;

    // Visible for testing.
    Header[] dynamicTable = new Header[8];
    // Array is populated back to front, so new entries always have lowest index.
    int nextHeaderIndex = dynamicTable.length - 1;
    int headerCount = 0;
    int dynamicTableByteCount = 0;

    Reader(int headerTableSizeSetting, Source source) {
      this(headerTableSizeSetting, headerTableSizeSetting, source);
    }

    Reader(int headerTableSizeSetting, int maxDynamicTableByteCount, Source source) {
      this.headerTableSizeSetting = headerTableSizeSetting;
      this.maxDynamicTableByteCount = maxDynamicTableByteCount;
      this.source = Okio.buffer(source);
    }

    int maxDynamicTableByteCount() {
      return maxDynamicTableByteCount;
    }

    private void adjustDynamicTableByteCount() {
      if (maxDynamicTableByteCount < dynamicTableByteCount) {
        if (maxDynamicTableByteCount == 0) {
          clearDynamicTable();
        } else {
          evictToRecoverBytes(dynamicTableByteCount - maxDynamicTableByteCount);
        }
      }
    }

    private void clearDynamicTable() {
      headerList.clear();
      Arrays.fill(dynamicTable, null);
      nextHeaderIndex = dynamicTable.length - 1;
      headerCount = 0;
      dynamicTableByteCount = 0;
    }

    /** Returns the count of entries evicted. */
    private int evictToRecoverBytes(int bytesToRecover) {
      int entriesToEvict = 0;
      if (bytesToRecover > 0) {
        // determine how many headers need to be evicted.
        for (int j = dynamicTable.length - 1; j >= nextHeaderIndex && bytesToRecover > 0; j--) {
          bytesToRecover -= dynamicTable[j].hpackSize;
          dynamicTableByteCount -= dynamicTable[j].hpackSize;
          headerCount--;
          entriesToEvict++;
        }
        System.arraycopy(dynamicTable, nextHeaderIndex + 1, dynamicTable,
            nextHeaderIndex + 1 + entriesToEvict, headerCount);
        nextHeaderIndex += entriesToEvict;
      }
      return entriesToEvict;
    }

    /**
     * Read {@code byteCount} bytes of headers from the source stream. This implementation does not
     * propagate the never indexed flag of a header.
     */
    void readHeaders() throws IOException {
      while (!source.exhausted()) {
        int b = source.readByte() & 0xff;
        if (b == 0x80) { // 10000000
          throw new IOException("index == 0");
        } else if ((b & 0x80) == 0x80) { // 1NNNNNNN
          int index = readInt(b, PREFIX_7_BITS);
          readIndexedHeader(index - 1);
        } else if (b == 0x40) { // 01000000
          readLiteralHeaderWithIncrementalIndexingNewName();
        } else if ((b & 0x40) == 0x40) {  // 01NNNNNN
          int index = readInt(b, PREFIX_6_BITS);
          readLiteralHeaderWithIncrementalIndexingIndexedName(index - 1);
        } else if ((b & 0x20) == 0x20) {  // 001NNNNN
          maxDynamicTableByteCount = readInt(b, PREFIX_5_BITS);
          if (maxDynamicTableByteCount < 0
              || maxDynamicTableByteCount > headerTableSizeSetting) {
            throw new IOException("Invalid dynamic table size update " + maxDynamicTableByteCount);
          }
          adjustDynamicTableByteCount();
        } else if (b == 0x10 || b == 0) { // 000?0000 - Ignore never indexed bit.
          readLiteralHeaderWithoutIndexingNewName();
        } else { // 000?NNNN - Ignore never indexed bit.
          int index = readInt(b, PREFIX_4_BITS);
          readLiteralHeaderWithoutIndexingIndexedName(index - 1);
        }
      }
    }

    public List<Header> getAndResetHeaderList() {
      List<Header> result = new ArrayList<>(headerList);
      headerList.clear();
      return result;
    }

    private void readIndexedHeader(int index) throws IOException {
      if (isStaticHeader(index)) {
        Header staticEntry = STATIC_HEADER_TABLE[index];
        headerList.add(staticEntry);
      } else {
        int dynamicTableIndex = dynamicTableIndex(index - STATIC_HEADER_TABLE.length);
        if (dynamicTableIndex < 0 || dynamicTableIndex > dynamicTable.length - 1) {
          throw new IOException("Header index too large " + (index + 1));
        }
        headerList.add(dynamicTable[dynamicTableIndex]);
      }
    }

    // referencedHeaders is relative to nextHeaderIndex + 1.
    private int dynamicTableIndex(int index) {
      return nextHeaderIndex + 1 + index;
    }

    private void readLiteralHeaderWithoutIndexingIndexedName(int index) throws IOException {
      ByteString name = getName(index);
      ByteString value = readByteString();
      headerList.add(new Header(name, value));
    }

    private void readLiteralHeaderWithoutIndexingNewName() throws IOException {
      ByteString name = checkLowercase(readByteString());
      ByteString value = readByteString();
      headerList.add(new Header(name, value));
    }

    private void readLiteralHeaderWithIncrementalIndexingIndexedName(int nameIndex)
        throws IOException {
      ByteString name = getName(nameIndex);
      ByteString value = readByteString();
      insertIntoDynamicTable(-1, new Header(name, value));
    }

    private void readLiteralHeaderWithIncrementalIndexingNewName() throws IOException {
      ByteString name = checkLowercase(readByteString());
      ByteString value = readByteString();
      insertIntoDynamicTable(-1, new Header(name, value));
    }

    private ByteString getName(int index) {
      if (isStaticHeader(index)) {
        return STATIC_HEADER_TABLE[index].name;
      } else {
        return dynamicTable[dynamicTableIndex(index - STATIC_HEADER_TABLE.length)].name;
      }
    }

    private boolean isStaticHeader(int index) {
      return index >= 0 && index <= STATIC_HEADER_TABLE.length - 1;
    }

    /** index == -1 when new. */
    private void insertIntoDynamicTable(int index, Header entry) {
      headerList.add(entry);

      int delta = entry.hpackSize;
      if (index != -1) { // Index -1 == new header.
        delta -= dynamicTable[dynamicTableIndex(index)].hpackSize;
      }

      // if the new or replacement header is too big, drop all entries.
      if (delta > maxDynamicTableByteCount) {
        clearDynamicTable();
        return;
      }

      // Evict headers to the required length.
      int bytesToRecover = (dynamicTableByteCount + delta) - maxDynamicTableByteCount;
      int entriesEvicted = evictToRecoverBytes(bytesToRecover);

      if (index == -1) { // Adding a value to the dynamic table.
        if (headerCount + 1 > dynamicTable.length) { // Need to grow the dynamic table.
          Header[] doubled = new Header[dynamicTable.length * 2];
          System.arraycopy(dynamicTable, 0, doubled, dynamicTable.length, dynamicTable.length);
          nextHeaderIndex = dynamicTable.length - 1;
          dynamicTable = doubled;
        }
        index = nextHeaderIndex--;
        dynamicTable[index] = entry;
        headerCount++;
      } else { // Replace value at same position.
        index += dynamicTableIndex(index) + entriesEvicted;
        dynamicTable[index] = entry;
      }
      dynamicTableByteCount += delta;
    }
```
HTTP/2中数据收发两端的动态表一致性主要是依赖TCP来实现的。

Done。