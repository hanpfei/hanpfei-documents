---
title: Protobuf开发者指南
date: 2016-11-22 20:35:49
tags: 
- 网络
---

欢迎访问 **Protocol Buffers** ——一个用于通信协议、数据存储及其它场景中，语言无关、平台无关、可扩展的结构化数据序列化方法——的开发者文档。

<!--more-->

本文档是为那些想要在自己的应用中使用 **Protocol Buffers** 的Java、C++或Python开发者而写的。这份概述介绍 **Protocol Buffers** ，并告诉你如何将它用起来——然后你可以通过 [教程](https://developers.google.com/protocol-buffers/docs/tutorials) 继续学习，或深入了解 [ **Protocol Buffers** 编码规则](https://developers.google.com/protocol-buffers/docs/encoding)。也为这三种语言提供了API [参考文档](https://developers.google.com/protocol-buffers/docs/reference/overview)，以及编写 **.proto** 文件的 [语言](https://developers.google.com/protocol-buffers/docs/proto) 和 [风格](https://developers.google.com/protocol-buffers/docs/style) 指导。

# 什么是Protocol Buffers?
**Protocol Buffers** 是一个序列化结构化数据的灵活、高效且自动化的机制——类似XML，但更小，更快，更简单。你定义一次结构化你的数据的方式，然后使用特别生成的代码简单地写入，或使用不同的语言从大量的数据流读出你的结构化数据。你甚至可以更新你的数据结构而不破坏已部署的基于 **老** 格式编译的程序。

# 它们如何工作的？

通过在 **.proto** 文件中定义 **Protocol Buffers** 消息类型来描述你想要结构化你在序列化的信息的方式。每个 **Protocol Buffers** 消息是一个信息的小逻辑记录，包含一系列名-值对。这里是一个非常基本的 **.proto** 文件的例子，它定义包含关于一个人的信息的消息：
```
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phone = 4;
}
```
如你所见，消息的格式很简单——每个消息类型具有一个或多个唯一编号的字段，每个字段具有一个名和一个值，其中值类型可以是数字（整数或浮点数），布尔值，字符串，原始的字节，或者甚至是（如上面的例子所示）其它的Protocol Buffer消息类型，这允许层次式地结构化你的数据。你可以指定可选的字段、[必需的字段](https://developers.google.com/protocol-buffers/docs/proto#required_warning)，和重复的字段。你可以在[Protocol Buffer 语言指南](https://developers.google.com/protocol-buffers/docs/proto) 找到更多关于编写 **.proto** 文件的信息。

一旦定义好消息，你就可以运行 **Protocol Buffer** 编译器为你的 **.proto** 文件产生应用程序的语言的数据访问类。这为每个字段提供了简单的访问器 (如**name()** 和 **set_name()**)，以及将整个结构序列化为原始字节，或从原始字节解析为结构的方法——因而，比如你选择C++，为上面的例子运行编译器将产生名为Person的类。然后你可以在你的应用程序中使用这个类，来放置、序列化和提取Person  **Protocol Buffer** 消息。然后你可以编写如下这样的代码：
```
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
```
随后你可以将消息读回：
```
fstream input("myfile", ios::in | ios::binary);
Person person;
person.ParseFromIstream(&input);
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```

你可以在不破坏向后兼容性的前提下为你的消息格式添加新字段；老的程序在解析时简单地忽略新字段。如果你有一个以 **Protocol Buffers** 为数据格式的通信协议，则可以轻松地扩展你的协议而不用担心破坏已有的代码。你可以在 [API 参考](https://developers.google.com/protocol-buffers/docs/reference/overview) 找到使用生成的 **Protocol Buffers** 代码的完整参考，你可以在 [Protocol Buffer编码](https://developers.google.com/protocol-buffers/docs/encoding) 中找到更多关于 **Protocol Buffers** 消息编码的内容。

# 为什么不使用XML呢？
在序列化数据方面，相对于XML， **Protocol Buffers** 有许多有点。 **Protocol Buffers** ：
* 更简单
* 小3至10倍
* 快20至100倍
* 更少歧义
* 产生数据访问类方便编程使用

比如，你想要为 **person** 建模，它有一个 **name** 字段和一个 **email** 字段。在XML中，你需要：
```
  <person>
    <name>John Doe</name>
    <email>jdoe@example.com</email>
  </person>
```
而对应的 **Protocol Buffers** 消息 (以 **Protocol Buffers** 的[文本格式](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.text_format)描述) 则是：
```
# Textual representation of a protocol buffer.
# This is *not* the binary format used on the wire.
person {
  name: "John Doe"
  email: "jdoe@example.com"
}
```
当消息被编码为 **Protocol Buffers** 的[二进制格式](https://developers.google.com/protocol-buffers/docs/encoding) (上边的文本格式只是为了方便调试和编辑的人类可读的表示)，它可能是28字节长，并需要大概100-200 纳秒来解析。如果移除空白符的话，XML版本至少需要69字节，并需要大概 5,000-10,000 纳秒来解析。

管理一个 **Protocol Buffers** 更简单：
```
  cout << "Name: " << person.name() << endl;
  cout << "E-mail: " << person.email() << endl;
```
使用XML的话你将不得不像下面这样：
```
  cout << "Name: "
       << person.getElementsByTagName("name")->item(0)->innerText()
       << endl;
  cout << "E-mail: "
       << person.getElementsByTagName("email")->item(0)->innerText()
       << endl;
```
然而， **Protocol Buffers** 也不总是比XML好——比如， **Protocol Buffers** 不是建模 含有标记的基于文本的文档 (如HTML) 的好方法，因为你不能简单地交叉含有文本的结构，此外，XML是人类可读且人类可编辑的； **Protocol Buffers** ，至少在它们的本地格式，不是。XML还——在一定程度上——是自描述的。 **Protocol Buffers** 只在你有消息定义 (.proto 文件) 时才有意义。

# 听起来正是我想要的方案！我要如何将它用起来呢？
[下载 **Protocol Buffers** 包](https://developers.google.com/protocol-buffers/docs/downloads.html) ——其中包含完整的Java、Python和C++  **Protocol Buffers** 编译器的代码，还包含你可以用于I/O测试的类。要构建并安装你的编译器，请依照README的指导进行。

一旦都设置好了，则可以试着按照 你选择的语言的 [教程](https://developers.google.com/protocol-buffers/docs/tutorials) 继续学习 ——这将带你创建一个使用 **Protocol Buffers** 的简单应用。

# proto3简介

我们最近的版本 3 [发布](https://github.com/google/protobuf/releases) 引入了一个新的语言版本 -  **Protocol Buffers** 语言版本 3 (亦称proto3)，并在我们已有的语言版本  (亦称proto2) 引入了一些新功能。Proto3简化了 **Protocol Buffers** 语言，使使用变得更简单，并可以在更广泛的语言中使用：我们当前的发行版让你可以为Java，C++，Python，Java Lite，Ruby，JavaScript，Objective-C，和C#产生 **Protocol Buffers** 代码。此外，你可以使用最新的Go protoc插件为Go产生proto3代码，可在 [golang/protobuf](https://github.com/golang/protobuf) Github 仓库找到。更多语言还在计划中。

当前我们建议只试用proto3：
* 如果你想要试用我们新支持的语言。
* 如果你想要使用我们的新开源RPC实现 [gRPC](http://github.com/grpc/grpc-common) – 我们建议为所有的新gRPC服务器和客户端使用proto3，以避免出现兼容性问题。

**注意**，两个语言版本的APIs不完全兼容。为了避免给现有用户造成不便，我们将在新的 **Protocol Buffers** 发行版中继续支持之前的语言版本。

你可以在 [发行说明](https://github.com/google/protobuf/releases) 中查看当前默认版本的主要差异，并在 [Proto3语言指南](https://developers.google.com/protocol-buffers/docs/proto3) 学习关于proto3语法的内容。完整的proto3文档很快就要到来了！

(如果说名字proto2和proto3似乎有点混乱，那是由于我们最初在开源 **Protocol Buffers** 时，它实际上是Google的第二个语言版本——也被称为proto2。这也是为什么我们的开源版本号是从v2.0.0开始的)。

# 历史
 **Protocol Buffers** 最初是在Google开发的，用来处理一个索引服务器请求/响应协议。在 **Protocol Buffers** 之前，有一个请求和响应的格式用于手动序列化/反序列化请求和响应，而且它支持协议的大量版本。这导致了一些非常丑陋的代码，比如：
```
 if (version == 3) {
   ...
 } else if (version > 4) {
   if (version == 5) {
     ...
   }
   ...
 }
```
显式地格式化协议也使新协议版本的上线很复杂，因为开发者不得不在他们切换到新协议之前，确保所有发起请求的服务器和实际处理请求的服务器理解新协议。

 **Protocol Buffers** 设计来解决许多这些问题：
* 可以简单地引入新字段，无需深入理解数据的中间服务器可以简单地解析并传递数据而无需知道所有的字段。
* 格式更加具有自描述性，且可由大量的语言 (C++，Java，等等) 处理。

然而，用户依然需要手写它们自己的解析代码。

随着系统的发展，它得到了大量的其它功能及使用：
* 自动生成序列化和反序列化的代码以避免手动解析。
* 此外被用于短暂的 RPC (Remote Procedure Call) 请求，人们开始使用 **Protocol Buffers** 作为持久存储数据的便利的自描述格式。
* 服务器RPC接口开始被声明为协议文件的一部分，并以协议编译器生成stub类，用户可以以服务器的接口的实际实现覆盖。

 **Protocol Buffers** 现在是Google的数据的通用语言——在写作本文的时候，有48,162个不同的消息类型定义在Google代码库的12,183 个 **.proto** 文件中。它们同时在RPC系统及不同的存储系统的数据存储中使用。

[原文](https://developers.google.com/protocol-buffers/docs/overview)
