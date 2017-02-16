---
title: 在C++中使用Protocol Buffers
date: 2016-12-2 11:43:49
categories:
- C/C++开发
tags:
- C/C++开发
---

# 下载并编译Protocol Buffer

这份教程为C++开发者提供了使用 **Protocol Buffer** 的基本介绍。通过创建一个简单应用，它展示了
 * 在 **.proto** 文件中定义消息格式。
 * 使用 **Protocol Buffer** 编译器。
 * 使用C++  **Protocol Buffer** API读写消息。

<!--more-->

这不是一个在C++中使用 **Protocol Buffer** 的全面指南。更多详细的信息，请参考[Protocol Buffer语言指南](https://developers.google.com/protocol-buffers/docs/proto)， [C++ API参考](https://developers.google.com/protocol-buffers/docs/reference/cpp/index.html)，[C++ Generated Code Guide](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated)，和 [编码参考](https://developers.google.com/protocol-buffers/docs/encoding)。

# 为什么使用Protocol Buffers？

我们将使用的例子是一个非常简单的 "address book" 应用，它可以从文件读取和向文件写入人们的联系人详情。地址簿中的每个人具有一个名字 (name)，ID，电子邮件地址 (email address)，和联系人电话号码 (contact phone)。

你要如何序列化和提取这样的结构化数据呢？有一些方法可以解决这个问题：
 * 原始的内存数据结构可以以二进制的形式发送/保存。随着时间的流逝，这是一种脆弱的方法，因为接收/读取的代码必须以完全相同的内存布局、尾端等等编译。此外，随着文件以原始的格式累积数据及处理那种格式的软件的复制，那种格式被不断传播，则它是非常难以扩展的格式。

 * 你可以发明一种特别的方式来将数据项编码编码为一个字符串 —— 比如将4个int值编码为"12:3:-23:67"。这是一个简单而灵活的方法，尽管它需要编写一次性的编码和解析代码，而且解析消耗一小段运行时代价。这对于编码非常简单的数据是最好的方式。
将数据序列化为XML。这种方法可能非常具有吸引力，因为XML是 (有点) 人类可读的，而且它有大量编程语言的bindings库。如果你想要与其它的应用/项目共享数据的话，这可能是一个很好的选择。然而，XML是臭名昭著的空间密集，而且编码/解码它需要消耗应用大量的性能开销。而且，浏览一个XML DOM树也被认为比通常浏览类中的简单字段更复杂。

**Protocol buffers** 是解决这个问题灵活，高效，自动化的方案。通过 **Protocol buffers** ，你可以编写一个 .proto 描述你想要存储的数据结构。通过它， **Protocol buffers** 编译器创建一个类，以一种高效的二进制格式实现自动的编码和解析 **Protocol buffers** 数据。生成的类为构成一个 **Protocol buffers** 的字段提供了getters和setters方法，并处理读取和写入 **Protocol buffers** 的细节。重要地是， **Protocol buffers** 格式通过使代码依然能够读取用老的格式编码的数据来支持随着时间对格式的扩展。

# 在哪里可以找到示例代码

源码包中包含的示例代码，在"examples" 目录下。[在这里下载。](https://developers.google.com/protocol-buffers/docs/downloads.html)

# 定义你的协议格式
为了创建你的地址簿应用，你需要先创建一个 .proto 文件。 .proto 文件中的定义很简单：为每个你想要序列化的数据结构添加一个 *消息(message)* ，然后为消息中的每个字段指定一个名字和类型。这里是定义你的消息的 .proto 文件，addressbook.proto。

```
package tutorial;

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

message AddressBook {
  repeated Person person = 1;
}
```

如你所见，语法与C++或Java类似。让我们看一下这个文件的每个部分，并看一下它做了什么。

.proto 文件以一个包声明开始，这用于防止不同项目间的命名冲突。在C++中，生成的类将被放置在与包名匹配的命名空间中。

接下来，定义你的消息。消息只是包含了具有类型的字段的聚合。许多标准的简单数据类型可用作字段类型，包括bool，int32，float，double，和string。你也可以通过使用消息类型作为字段类型来给你的消息添加更多结构 —— 在上面的例子中，Person消息包含了多个PhoneNumber消息，同时AddressBook消息包含Person消息。你甚至可以在其它消息中嵌套的定义消息类型 —— 如你所见，PhoneNumber类型是在Person中定义的。如果你想要你的字段值为某个预定义的值列表中的某个值的话，你也可以定义enum类型 —— 这里你想要指定电话号码是MOBILE，HOME，或WORK中的一个。

每个元素上的 " = 1"，" = 2"标记标识在二进制编码中使用的该字段唯一的 "tag" 。Tag数字 1-15 比更大的数字在编码上少一个字节，因而作为一种优化，你可以决定将那些数字用作常用的或重复的元素的tag，而将16及更大的数字tag留给更加不常用的可选元素。重复字段中的每个元素需要重编码tag数字，因而这种优化特别适用于重复字段。

每个字段必须用下面的修饰符中的一个来注解：

 * required：字段必须提供，否则消息将被认为是 "未初始化的 (uninitialized)"。如果libprotobuf以debug模式编译，则序列化未初始化的消息将导致断言失败。在优化的构建中，检查将被跳过，消息仍将被写入。然而，解析未初始化的消息将总是失败 (通过喜爱parse方法中返回false)。否则，required字段的行为将与optional字段完全相同。

 * optional：字段可以设置也可以不设置。如果可选的字段值没有设置，则将使用默认值。对于简单的类型，你可以指定你自己的默认值，如我们在例子中为电话号码类型做的那样。否则，将使用系统默认值：数字类型为0，字符串类型为空字符串，bools值为false。对于内嵌的消息，默认值总是消息的 "默认实例 (default instance)" 或 "原型(prototype)"，它们没有自己的字段集。调用accessor获取还没有显式地设置的 optional (或required) 字段的值总是返回字段的默认值。

 * repeated：字段可以重复任意多次 (包括0)。在 **protocol buffer** 中，重复值的顺序将被保留。将重复字段想象为动态大小的数组。

你将找到一个编写 .proto 文件的完整指南 —— 包括所有可能的字段类型 —— 在[Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto) 一文中。不要寻找与类继承类似的设施 ——  **protocol buffer** 不那样做。

# 编译你的Protocol Buffers
现在你有了一个.proto，接下来你需要做的事情是生成读写 AddressBook (及Person 和 PhoneNumber) 消息所需的类。要做到这一点，你需要在你的 .proto 上运行 **Protocol Buffers** 编译器protoc：

1. 如果你还没有安装编译器，则[下载包](https://developers.google.com/protocol-buffers/docs/downloads.html)，并按照README的指示进行。

2. 现在运行编译器，指定源目录 (放置你的应用程序源代码的地方 —— 如果你没有提供则使用当前目录)，目的目录 (你希望放置生成的代码的位置；通常与$SRC_DIR相同)，你的.proto的路径。在这个例子中，你... ：

```
protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto
```
由于你想要C++类，所以使用 --cpp_out 选项 —— 也为其它支持的语言提供了类似的选项。

这将在你指定的目的目录下生成下面的文件：
* addressbook.pb.h，声明你的生成类的头文件。
* addressbook.pb.cc，包含了你的类的实现。

# Protocol Buffer API
让我们看一下生成的代码，并看一下编译器都为你创建了什么类和函数。如果查看tutorial.pb.h，你可以看到你在tutorial.proto中描述的每个消息都有一个类。进一步看Person类的话，你可以看到编译器已经为每个字段生成了accessors。比如，name，id，email，和phone字段，你具有这些方法：
 ```
  // name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phone
  inline int phone_size() const;
  inline void clear_phone();
  inline const ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >& phone() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phone();
  inline const ::tutorial::Person_PhoneNumber& phone(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phone(int index);
  inline ::tutorial::Person_PhoneNumber* add_phone();
```
如你所见，getters的名字与字段名的小写形式完全一样，而setter方法则以set_开头。每个单数的 (required 或 optional) 字段还有has_ 方法，如果那个字段已经被设置了则它们放回true。最后，每个字段具有一个 clear_ 方法，用于将字段设置回它的空状态。

数字的id字段只有基本的如上所述的accessor set，而name和email字段则有一对额外的方法，因为它们是字符串 —— 一个mutable_ getter，让你获取指向字符串的直接的指针，及一个额外的setter。注意你可以调用mutable_email()，即使email还没有设置；它将被自动地初始化为一个空字符窜。如果在这个例子中你有一个单数的消息字段，它将还有一个mutable_方法，而没有set_方法。

重复的字段还有一些特别的方法 —— 如果你查看重复的phone字段的方法的话，你将看到你可以
* 检查重复字段的 _size (换句话说，与这个Person关联的电话号码有多少个)。
* 使用索引得到一个特定的电话号码。
* 更新特定位置处的已有电话号码。
* 给消息添加另一个后面你可以编辑的电话号码 (重复的标量类型具有一个add_ 以使你可以传入新值)。

关于protocol编译器为任何特定的字段定义产生什么成员的更多信息，请参考 [C++ 生成代码参考](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated)。

## 枚举和嵌套类
生成的代码包含一个PhoneType枚举，它对应于你的.proto枚举。你可以以Person::PhoneType引用这个类型，它的值包括 Person::MOBILE，Person::HOME，和Person::WORK (实现细节要复杂一点，但你使用枚举时无需理解它们)。

编译器还为你生成了称为Person::PhoneNumber的嵌套类。如果你看代码，会发现 "真实的" 类实际称为 Person_PhoneNumber，但定义在Person内的typedef使你可以像一个嵌套类一样使用它。会影响到的仅有的情况是，如果你想要在另一个文件中前向声明类 —— 你不能在C++中前向声明嵌套类型，但你可以前向声明Person_PhoneNumber。

## 标准的消息方法
每个消息类还包含大量的其它方法，来让你检查或管理整个消息，包括：
* bool IsInitialized() const;: 检查是否所有的required字段都已经被设置了。
* string DebugString() const;: 返回一个人类可读的消息表示，对调试特别有用。
* void CopyFrom(const Person& from);: 用给定消息的值覆写消息。
* void Clear();: 清空所有的元素为空状态。

这些方法以及在后面的小节中描述的I/O方法实现了所有C++ protocol buffer类共享的Message接口。更多信息，请参考 [Message的完整API文档](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message)。

## Parsing and Serialization解析和序列化
最后，每个protocol buffer类都有使用protocol buffer  [二进制格式](https://developers.google.com/protocol-buffers/docs/encoding)写和读你所选择类型的消息的方法。这些方法包括：
* bool SerializeToString(string* output) const;: 序列化消息并将字节存储进给定的字符串中。注意，字节是二进制格式的，而不是文本；我们只将string类用作适当的容器。
* bool ParseFromString(const string& data);: 从给定的字符串解析一个消息。
* bool SerializeToOstream(ostream* output) const;: 将消息写入给定的C++ ostream。
* bool ParseFromIstream(istream* input);: 从给定的C++ istream解析消息。

这些只是解析和序列化提供的一些选项。再次，请参考 [Message API 参考](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message) 来获得完整的列表。

# 写消息
现在让我们试着使用protocol buffer类。你想要你的地址簿应用能够做的第一件事情是将个人详情写入地址簿文件。要做到这一点，你需要创建并防止你的protocol buffer类的实例，然后将它们写入一个输出流。这里是一个程序，它从一个文件读取一个AddressBook，基于用户输入给它添加一个新Person，并再次将新的AddressBook写回文件。直接调用或引用由protocol编译器生成的代码的部分都被高亮了。
```
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// This function fills in a Person message based on user input.
void PromptForAddress(tutorial::Person* person) {
  cout << "Enter person ID number: ";
  int id;
  cin >> id;
  person->set_id(id);
  cin.ignore(256, '\n');

  cout << "Enter name: ";
  getline(cin, *person->mutable_name());

  cout << "Enter email address (blank for none): ";
  string email;
  getline(cin, email);
  if (!email.empty()) {
    person->set_email(email);
  }

  while (true) {
    cout << "Enter a phone number (or leave blank to finish): ";
    string number;
    getline(cin, number);
    if (number.empty()) {
      break;
    }

    tutorial::Person::PhoneNumber* phone_number = person->add_phone();
    phone_number->set_number(number);

    cout << "Is this a mobile, home, or work phone? ";
    string type;
    getline(cin, type);
    if (type == "mobile") {
      phone_number->set_type(tutorial::Person::MOBILE);
    } else if (type == "home") {
      phone_number->set_type(tutorial::Person::HOME);
    } else if (type == "work") {
      phone_number->set_type(tutorial::Person::WORK);
    } else {
      cout << "Unknown phone type.  Using default." << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file,
//   adds one person based on user input, then writes it back out to the same
//   file.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!input) {
      cout << argv[1] << ": File not found.  Creating a new file." << endl;
    } else if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  // Add an address.
  PromptForAddress(address_book.add_person());

  {
    // Write the new address book back to disk.
    fstream output(argv[1], ios::out | ios::trunc | ios::binary);
    if (!address_book.SerializeToOstream(&output)) {
      cerr << "Failed to write address book." << endl;
      return -1;
    }
  }

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```
注意GOOGLE_PROTOBUF_VERIFY_VERSION宏。它是良好的实践 —— 尽管不是严格必须的 —— 在使用C++ Protocol Buffer库之前执行这个宏。它验证你没有偶然地链接一个与你编译的头文件版本不兼容的库版本。如果探测到版本不匹配，程序将终止。注意每个.pb.cc文件自动地在启动时调用这个宏。

还要主要在程序的最后调用ShutdownProtobufLibrary()。这个步骤做的所有事情是删除Protocol Buffer库分配的全局对象。这对大多数程序都是不必要的，因为进程退出后，OS将回收它的所有内从。然而，如果你使用了内存泄漏检查工具，它需要每个对象都被释放，或者如果你在编写一个库，它可能被单独的进程加载和卸载多次，则你可能想要强制Protocol Buffers清理所有的东西。

# 读消息
当然，如果你不能从地址簿中获取信息的话，那它就每什么用了。这个例子读取上面例子创建的文件并打印它的所有信息。
```
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// Iterates though all people in the AddressBook and prints info about them.
void ListPeople(const tutorial::AddressBook& address_book) {
  for (int i = 0; i < address_book.person_size(); i++) {
    const tutorial::Person& person = address_book.person(i);

    cout << "Person ID: " << person.id() << endl;
    cout << "  Name: " << person.name() << endl;
    if (person.has_email()) {
      cout << "  E-mail address: " << person.email() << endl;
    }

    for (int j = 0; j < person.phone_size(); j++) {
      const tutorial::Person::PhoneNumber& phone_number = person.phone(j);

      switch (phone_number.type()) {
        case tutorial::Person::MOBILE:
          cout << "  Mobile phone #: ";
          break;
        case tutorial::Person::HOME:
          cout << "  Home phone #: ";
          break;
        case tutorial::Person::WORK:
          cout << "  Work phone #: ";
          break;
      }
      cout << phone_number.number() << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file and prints all
//   the information inside.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  ListPeople(address_book);

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```

# 扩展一个Protocol Buffer
在你发布使用你的protocol buffer的代码之后或早或完，你都将毫无疑问的想要 "提升" protocol buffer的定义。如果你想要你的新buffers向后兼容，你的老buffers向前兼容 —— 你当然几乎总是想要这样 —— 然后你有一些规则要遵守。在新版本的protocol buffer中：
* 你 *一定不能* 修改任何已有字段的tag数字。
* 你 *一定不能* 添加或删除required字段。
* 你 *可以* 删除可选的或重复的字段。
* 你 *可以* 添加可选或重复的字段，但你必须使用新的tag数字 (比如，从未在这个protocol buffer中使用过的tag数字，甚至是在删除的字段中也是)。

(这些规则有 [一些例外](https://developers.google.com/protocol-buffers/docs/proto.html#updating) ，但它们几乎从未用到)

如果你按照这些规则，老代码将开心地读取新消息并简单地忽略新字段。对于老代码来说，删除的可选字段将简单的具有它们的默认值，删除的重复字段将是空的。新代码将透明地读取老消息。然而，请记住新的可选字段将不会出现在老的消息中，因此你将需要显示地检查它们是否通过has_设置了，或通过 [default = value] 在你的 .proto 文件中的tag数字后面提供一个合理的默认值。如果没有为可选元素指定默认值，则会使用特定于类型的默认值代替：对于字符串，默认值是空字符串。对于booleans，默认值是false。对于数字类型，默认值是0。还要注意如果你添加了一个新的重复字段，你的新代码将不能区别他是空的 (通过新代码) 还是从来没有设置 (通过老代码) ，因为它没有 has_ 标记。

# 优化建议
C++ Protocol Buffers是经过高度优化了的。然而，适当的使用可以提升更多性能。这里是一些提示，用于从库中挤出每一滴性能：

 * 只要可能就重用消息。消息消息会尝试保留它们分配的内存以复用，甚至当它们被清理的时候。这样，如果你在连续处理相同类型及类似结构的许多消息，则每次复用相同消息对象就是一个降低内存分配器负载的好主意。然后，对象可能随着变得膨胀，特别是如果你的消息在 "形状(shape)" 上经常改变，或如果你偶然构造了一个比通常情况大很多的消息。你应该通过调用 [SpaceUsed](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message.SpaceUsed.details) 方法监视你的消息对象的大小，并在它们变的太大时删除它们。

 * 你的系统的内存分配器可能没有针对在多线程中分配大量小对象做个很好的优化。则尝试使用 [Google的tcmalloc](http://code.google.com/p/google-perftools/) 来代替。

# 高级用法
Protocol buffers的使用场景不仅仅是简单的存取器和序列化。确保浏览  [C++ API 参考](https://developers.google.com/protocol-buffers/docs/reference/cpp/index.html) 来了解你还可以用它做什么。

由protocol消息类提供的一个重要功能是 *反射* 。你可以迭代一个消息的字段，并在不针对特定的消息类型编写你的代码的情况下，管理它们的值。使用反射的一个非常有用的方式是将protocol消息转换为其它编码方式，或从其它编码方式转换，比如XML或JSON。反射的一个更高级的使用可能是查找相同类型的两个消息之间的差异，或者开发某种"protocol消息正则表达式"，你可以编写表达式用它匹配某一消息内容。如果使用你想象力，则将Protocol Buffers用到比你最初期望的更加广泛的问题的解决中是有可能的！

[Message::Reflection 接口](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message.Reflection)提供了反射.

[原文](https://developers.google.com/protocol-buffers/docs/cpptutorial)
