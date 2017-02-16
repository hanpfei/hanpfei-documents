---
title: 在Java中使用Protocol Buffers
date: 2016-12-2 16:43:49
categories: Java开发
tags:
- Java开发
---

这份教程为Java开发者提供了使用 **Protocol Buffer** 的基本介绍。通过创建一个简单的示例应用，它展示了
 * 在 **.proto** 文件中定义消息格式。
 * 使用 **Protocol Buffer** 编译器。
 * 使用Java  **Protocol Buffer** API读写消息。

<!--more-->

这不是一个在Java中使用 **Protocol Buffer** 的全面指南。更多详细的信息，请参考[Protocol Buffer语言指南](https://developers.google.com/protocol-buffers/docs/proto)， [Java API参考](https://developers.google.com/protocol-buffers/docs/reference/java/index.html)，[Java Generated Code Guide](https://developers.google.com/protocol-buffers/docs/reference/java-generated)，和 [编码参考](https://developers.google.com/protocol-buffers/docs/encoding)。

# 为什么使用Protocol Buffers？

我们将使用的例子是一个非常简单的 "address book" 应用，它可以从文件读取和向文件写入人们的联系人详情。地址簿中的每个人具有一个名字 (name)，ID，电子邮件地址 (email address)，和联系人电话号码 (contact phone)。

你要如何序列化和提取这样的结构化数据呢？有一些方法可以解决这个问题：
 * 使用Java序列化接口。这是默认的方法，因为它是编程语言内建的，但它有一个广为人知的问题 (参见Josh Bloch的Effective Java，pp. 213)，而且如果你需要与用C++或Python编写的应用共享数据时不能很好的工作。
 * 你可以发明一种特别的方式来将数据项编码为一个字符串 —— 比如将4个int值编码为"12:3:-23:67"。这是一个简单而灵活的方法，尽管它需要编写一次性的编码和解析代码，而且解析消耗一小段运行时代价。这对于编码非常简单的数据是最好的方式。
 * 将数据序列化为XML。这种方法可能非常具有吸引力，因为XML是 (有点) 人类可读的，而且它有大量编程语言的bindings库。如果你想要与其它的应用/项目共享数据的话，这可能是一个很好的选择。然而，XML是臭名昭著的空间密集，而且编码/解码它需要消耗应用大量的性能开销。而且，浏览一个XML DOM树也被认为比通常浏览类中的简单字段更复杂。

**Protocol buffers** 是解决这个问题灵活，高效，自动化的方案。通过 **Protocol buffers** ，你可以编写一个 .proto 描述你想要存储的数据结构。通过它， **Protocol buffers** 编译器创建一个类，以一种高效的二进制格式实现自动地编码和解析 **Protocol buffers** 数据。生成的类为构成一个 **Protocol buffers** 的字段提供了getters和setters方法，并处理读取和写入 **Protocol buffers** 的细节。重要地是， **Protocol buffers** 格式通过使代码依然能够读取用老的格式编码的数据来支持随着时间对格式的扩展。

# 在哪里可以找到示例代码

源码包中包含的示例代码，在"examples" 目录下。[在这里下载。](https://developers.google.com/protocol-buffers/docs/downloads.html)

# 定义你的协议格式
为了创建你的地址簿应用，你需要先创建一个 .proto 文件。 .proto 文件中的定义很简单：为每个你想要序列化的数据结构添加一个 *消息(message)* ，然后为消息中的每个字段指定一个名字和类型。这里是定义你的消息的 .proto 文件，addressbook.proto。

```
package tutorial;

option java_package = "com.example.tutorial";
option java_outer_classname = "AddressBookProtos";

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

.proto 文件以一个包声明开始，这用于防止不同项目间的命名冲突。在Java中，包名被用作Java包，除非你已经显式地指定了 **java_package**，如我们这里看到的。即使你不提供 **java_package**，你依然应该定义一个普通的 **package** 以避免Protocol Buffers命名空间中的冲突，以及在非Java语言中。

声明了包之后，你可以看到两个Java特有的选项： **java_package** 和 **java_outer_classname**。**java_package** 指定生成的类应该放在什么Java包名下。如果你没有显式地指定这个值，则它简单地匹配由**package** 声明给出的Java包名，但这些名字通常都不是合适的Java包名 (由于它们通常不以一个域名打头)。 **java_outer_classname** 选项定义应该包含这个文件中所有类的类名。如果你没有显式地给定**java_outer_classname** ，则将通过把文件名转换为首字母大写来生成。比如"my_proto.proto"，默认情况下，将使用 "MyProto" 做为它的外层类的类名。

接下来，定义你的消息。消息只是包含了具有类型的字段的聚合。许多标准的简单数据类型可用作字段类型，包括bool，int32，float，double，和string。你也可以通过使用消息类型作为字段类型来给你的消息添加更多结构 —— 在上面的例子中，Person消息包含了多个PhoneNumber消息，同时AddressBook消息包含Person消息。你甚至可以在其它消息中嵌套的定义消息类型 —— 如你所见，PhoneNumber类型是在Person中定义的。如果你想要你的字段值为某个预定义的值列表中的某个值的话，你也可以定义enum类型 —— 这里你想要指定电话号码是MOBILE，HOME，或WORK中的一个。

每个元素上的 " = 1"，" = 2"标记标识在二进制编码中使用的该字段唯一的 "tag" 。Tag数字 1-15 比更大的数字在编码上少一个字节，因而作为一种优化，你可以决定将那些数字用作常用的或重复的元素的tag，而将16及更大的数字tag留给更加不常用的可选元素。重复字段中的每个元素需要重编码tag数字，因而这种优化特别适用于重复字段。

每个字段必须用下面的修饰符中的一个来注解：

 * required：字段必须提供，否则消息将被认为是 "未初始化的 (uninitialized)"。尝试构建一个未初始化的消息将抛出一个 **RuntimeException**。解析一个未初始化的消息将抛出一个 **IOException**。此外，required字段的行为与optional字段完全相同。

 * optional：字段可以设置也可以不设置。如果可选的字段值没有设置，则将使用默认值。对于简单的类型，你可以指定你自己的默认值，如我们在例子中为电话号码 **类型** 做的那样。否则，将使用系统默认值：数字类型为0，字符串类型为空字符串，bools值为false。对于内嵌的消息，默认值总是消息的 "默认实例 (default instance)" 或 "原型(prototype)"，它们没有自己的字段集。调用accessor获取还没有显式地设置的 optional (或required) 字段的值总是返回字段的默认值。

 * repeated：字段可以重复任意多次 (包括0)。在 **protocol buffer** 中，重复值的顺序将被保留。将重复字段想象为动态大小的数组。

你将找到一个编写 .proto 文件的完整指南 —— 包括所有可能的字段类型 —— 在[Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto) 一文中。不要寻找与类继承类似的设施 ——  **protocol buffer** 不那样做。

# 编译你的Protocol Buffers
现在你有了一个.proto，接下来你需要做的事情是生成读写 AddressBook (及Person 和 PhoneNumber) 消息所需的类。要做到这一点，你需要在你的 .proto 上运行 **Protocol Buffers** 编译器protoc：

1. 如果你还没有安装编译器，则[下载包](https://developers.google.com/protocol-buffers/docs/downloads.html)，并按照README的指示进行。

2. 现在运行编译器，指定源目录 (放置你的应用程序源代码的地方 —— 如果你没有提供则使用当前目录)，目的目录 (你希望放置生成的代码的位置；通常与$SRC_DIR相同)，你的.proto的路径。在这个例子中，你... ：

```
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```
由于你想要Java类，所以使用 --java_out 选项 —— 也为其它支持的语言提供了类似的选项。

这将在你指定的目的目录下生成**com/example/tutorial/AddressBookProtos.java**。

# Protocol Buffer API
让我们看一下生成的代码，并看一下编译器都为你创建了什么类和函数。如果查看 **AddressBookProtos.java**，你可以看到它定义了一个称为 **AddressBookProtos** 的类，其中嵌套了为你在 **addressbook.proto** 中描述的每个消息的类。每个类都有它自己的 **Builder** 类，你可以用来创建那个类的实例。你可以在下面的 [Builders vs. Messages](https://developers.google.com/protocol-buffers/docs/javatutorial#builders) 小节中找到更多关于builders的信息。

消息和builders具有为消息的每个字段自动生成的accessor方法；消息只有getters，而builders则同时具有getters和setters。这里是 **Person** 类的一些accessors (省略实现以便于简洁)：
 ```
// required string name = 1;
public boolean hasName();
public String getName();

// required int32 id = 2;
public boolean hasId();
public int getId();

// optional string email = 3;
public boolean hasEmail();
public String getEmail();

// repeated .tutorial.Person.PhoneNumber phone = 4;
public List<PhoneNumber> getPhoneList();
public int getPhoneCount();
public PhoneNumber getPhone(int index);
```
同时 **Person.Person** 类具有相同的getters外加setters：
```
// required string name = 1;
public boolean hasName();
public java.lang.String getName();
public Builder setName(String value);
public Builder clearName();

// required int32 id = 2;
public boolean hasId();
public int getId();
public Builder setId(int value);
public Builder clearId();

// optional string email = 3;
public boolean hasEmail();
public String getEmail();
public Builder setEmail(String value);
public Builder clearEmail();

// repeated .tutorial.Person.PhoneNumber phone = 4;
public List<PhoneNumber> getPhoneList();
public int getPhoneCount();
public PhoneNumber getPhone(int index);
public Builder setPhone(int index, PhoneNumber value);
public Builder addPhone(PhoneNumber value);
public Builder addAllPhone(Iterable<PhoneNumber> value);
public Builder clearPhone();
```

如你所见，每个自动都有简单的JavaBeans风格的getters和setters。每个单数的 (required 或 optional) 字段还有 **has** 方法，如果那个字段已经被设置了则它们放回true。最后，每个字段具有一个 **clear** 方法，用于将字段设置回它的空状态。

重复的字段还有一些额外的方法 —— 一个 **Count** 方法(是列表大小的速记)，通过索引获取和设置列表的特定元素的getters和setters，一个 **add** 方法，将新元素添加到列表的末尾，及一个 **addAll** 方法，它将一个装满元素的整个容器添加到列表中。

注意这些accessor方法是如何以驼峰形式命名的，即使 **.proto** 文件使用了小写字母加下划线。这种转换是由protocol buffer编译器自动地完成的，以产生与标准Java风格规范匹配的类。你应该总是在你的 **.proto** 文件中为字段使用小写字母加下划线；这确保了在所有生成的语言中良好的命名实践。参考 [风格指南](https://developers.google.com/protocol-buffers/docs/style) 来了解更多好的 **.proto** 风格。

关于protocol编译器为任何特定的字段定义产生什么成员的更多信息，请参考 [Java 生成代码参考](https://developers.google.com/protocol-buffers/docs/reference/java-generated)。

## 枚举和嵌套类
生成的代码包含一个PhoneType Java 5枚举，嵌套在 **Person** 中：
```
public static enum PhoneType {
  MOBILE(0, 0),
  HOME(1, 1),
  WORK(2, 2),
  ;
  ...
}
```
生成的嵌套类型 **Person.PhoneNumber**，如你期待的那样，是 **Person** 的嵌套类。

## Builders和Messages

由protocol buffer编译器生成的所有消息类都是*不可变的*。一旦某个消息对象构造完成 ，则它不能被修改，如同Java的 **String** 一样。要构造一个消息，你必须首先构造一个builder，设置你想要设置的字段为你选择的值，然后调用builder的 **build()** 方法。

你可能已经注意到了builder的每个方法都修改消息并返回另一个builder。返回的对象实际上与调用方法的那个builder是同一个。它被返回以使你可以将多个setters串在一起放在单独的一行代码上。

这里是如何创建你想要的 "Person" 实例一个例子：
```
Person john =
  Person.newBuilder()
    .setId(1234)
    .setName("John Doe")
    .setEmail("jdoe@example.com")
    .addPhone(
      Person.PhoneNumber.newBuilder()
        .setNumber("555-4321")
        .setType(Person.PhoneType.HOME))
    .build();
```

## 标准的消息方法
每个消息和builder类还包含大量的其它方法，来让你检查或管理整个消息，包括：
* isInitialized() : 检查是否所有的required字段都已经被设置了。
* toString() : 返回一个人类可读的消息表示，对调试特别有用。
* mergeFrom(Message other): (只有builder可用) 将 **other** 的内容合并到这个消息中，覆写单数的字段，附接重复的。
* clear(): (只有builder可用) 清空所有的元素为空状态。

这些方法实现由所有的Java消息和builders所共享的 **Message** 和 **Message.Builder** 接口。更多信息，请参考 [Message的完整API文档](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message)。

## 解析和序列化
最后，每个protocol buffer类都有使用protocol buffer  [二进制格式](https://developers.google.com/protocol-buffers/docs/encoding)写和读你所选择类型的消息的方法。这些方法包括：
* byte[] toByteArray();: 序列化消息并返回一个包含它的原始字节的字节数组。
* static Person parseFrom(byte[] data);: 从给定的字节数组解析一个消息。
* void writeTo(OutputStream output);: 序列化消息并将消息写入 **OutputStream**。
* static Person parseFrom(InputStream input);: 从一个 **InputStream** 读取并解析消息。

这些只是解析和序列化提供的一些选项。再次，请参考 [Message API 参考](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message.html#Message) 来获得完整的列表。

# 写消息
现在让我们试着使用protocol buffer类。你想要你的地址簿应用能够做的第一件事情是将个人详情写入地址簿文件。要做到这一点，你需要创建并放置你的protocol buffer类的实例，然后将它们写入一个输出流。

这里是一个程序，它从一个文件读取一个AddressBook，基于用户输入给它添加一个新Person，并再次将新的AddressBook写回文件。直接调用或引用由protocol编译器生成的代码的部分都被高亮了。
```
import com.example.tutorial.AddressBookProtos.AddressBook;
import com.example.tutorial.AddressBookProtos.Person;
import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.InputStreamReader;
import java.io.IOException;
import java.io.PrintStream;

class AddPerson {
  // This function fills in a Person message based on user input.
  static Person PromptForAddress(BufferedReader stdin,
                                 PrintStream stdout) throws IOException {
    Person.Builder person = Person.newBuilder();

    stdout.print("Enter person ID: ");
    person.setId(Integer.valueOf(stdin.readLine()));

    stdout.print("Enter name: ");
    person.setName(stdin.readLine());

    stdout.print("Enter email address (blank for none): ");
    String email = stdin.readLine();
    if (email.length() > 0) {
      person.setEmail(email);
    }

    while (true) {
      stdout.print("Enter a phone number (or leave blank to finish): ");
      String number = stdin.readLine();
      if (number.length() == 0) {
        break;
      }

      Person.PhoneNumber.Builder phoneNumber =
        Person.PhoneNumber.newBuilder().setNumber(number);

      stdout.print("Is this a mobile, home, or work phone? ");
      String type = stdin.readLine();
      if (type.equals("mobile")) {
        phoneNumber.setType(Person.PhoneType.MOBILE);
      } else if (type.equals("home")) {
        phoneNumber.setType(Person.PhoneType.HOME);
      } else if (type.equals("work")) {
        phoneNumber.setType(Person.PhoneType.WORK);
      } else {
        stdout.println("Unknown phone type.  Using default.");
      }

      person.addPhone(phoneNumber);
    }

    return person.build();
  }

  // Main function:  Reads the entire address book from a file,
  //   adds one person based on user input, then writes it back out to the same
  //   file.
  public static void main(String[] args) throws Exception {
    if (args.length != 1) {
      System.err.println("Usage:  AddPerson ADDRESS_BOOK_FILE");
      System.exit(-1);
    }

    AddressBook.Builder addressBook = AddressBook.newBuilder();

    // Read the existing address book.
    try {
      addressBook.mergeFrom(new FileInputStream(args[0]));
    } catch (FileNotFoundException e) {
      System.out.println(args[0] + ": File not found.  Creating a new file.");
    }

    // Add an address.
    addressBook.addPerson(
      PromptForAddress(new BufferedReader(new InputStreamReader(System.in)),
                       System.out));

    // Write the new address book back to disk.
    FileOutputStream output = new FileOutputStream(args[0]);
    addressBook.build().writeTo(output);
    output.close();
  }
}
```

# 读消息
当然，如果你不能从地址簿中获取信息的话，那它就没什么用了。这个例子读取上面例子创建的文件并打印它的所有信息。
```
import com.example.tutorial.AddressBookProtos.AddressBook;
import com.example.tutorial.AddressBookProtos.Person;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.PrintStream;

class ListPeople {
  // Iterates though all people in the AddressBook and prints info about them.
  static void Print(AddressBook addressBook) {
    for (Person person: addressBook.getPersonList()) {
      System.out.println("Person ID: " + person.getId());
      System.out.println("  Name: " + person.getName());
      if (person.hasEmail()) {
        System.out.println("  E-mail address: " + person.getEmail());
      }

      for (Person.PhoneNumber phoneNumber : person.getPhoneList()) {
        switch (phoneNumber.getType()) {
          case MOBILE:
            System.out.print("  Mobile phone #: ");
            break;
          case HOME:
            System.out.print("  Home phone #: ");
            break;
          case WORK:
            System.out.print("  Work phone #: ");
            break;
        }
        System.out.println(phoneNumber.getNumber());
      }
    }
  }

  // Main function:  Reads the entire address book from a file and prints all
  //   the information inside.
  public static void main(String[] args) throws Exception {
    if (args.length != 1) {
      System.err.println("Usage:  ListPeople ADDRESS_BOOK_FILE");
      System.exit(-1);
    }

    // Read the existing address book.
    AddressBook addressBook =
      AddressBook.parseFrom(new FileInputStream(args[0]));

    Print(addressBook);
  }
}
```

# 扩展一个Protocol Buffer
在你发布使用你的protocol buffer的代码之后或早或完，你都将毫无疑问的想要 "提升" protocol buffer的定义。如果你想要你的新buffers向后兼容，你的老buffers向前兼容 —— 你当然几乎总是想要这样 —— 然后你有一些规则要遵守。在新版本的protocol buffer中：
* 你 *一定不能* 修改任何已有字段的tag数字。
* 你 *一定不能* 添加或删除required字段。
* 你 *可以* 删除可选的或重复的字段。
* 你 *可以* 添加可选或重复的字段，但你必须使用新的tag数字 (比如，从未在这个protocol buffer中使用过的tag数字，甚至是在删除的字段中也是)。

(这些规则有 [一些例外](https://developers.google.com/protocol-buffers/docs/proto.html#updating) ，但它们几乎从未用到)

如果你按照这些规则，老代码将开心地读取新消息并简单地忽略新字段。对于老代码来说，删除的可选字段将简单的具有它们的默认值，删除的重复字段将是空的。新代码将透明地读取老消息。然而，请记住新的可选字段将不会出现在老的消息中，因此你将需要通过has_显式地检查它们是否设置了，或通过 [default = value] 在你的 .proto 文件中的tag数字后面提供一个合理的默认值。如果没有为可选元素指定默认值，则会使用特定于类型的默认值代替：对于字符串，默认值是空字符串。对于booleans，默认值是false。对于数字类型，默认值是0。还要注意如果你添加了一个新的重复字段，你的新代码将不能区别他是空的 (通过新代码) 还是从来没有设置 (通过老代码) ，因为它没有 has_ 标记。

# 高级用法
Protocol buffers的使用场景不仅仅是简单的存取器和序列化。务必浏览  [Java API 参考](https://developers.google.com/protocol-buffers/docs/reference/java/index.html) 来了解你还可以用它做什么。

由protocol消息类提供的一个重要功能是 *反射* 。你可以迭代一个消息的字段，并在不针对特定的消息类型编写你的代码的情况下，管理它们的值。使用反射的一个非常有用的方式是将protocol消息转换为其它编码方式，或从其它编码方式转换，比如XML或JSON。反射的一个更高级的使用可能是查找相同类型的两个消息之间的差异，或者开发某种"protocol消息正则表达式"，你可以编写表达式用它匹配某一消息内容。如果使用你想象力，则将Protocol Buffers用到比你最初期望的更加广泛的问题的解决中是有可能的！

反射是作为[Message
](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/Message) 和 [Message.Builder
](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/Message.Builder)接口的一部分提供的。

[原文](https://developers.google.com/protocol-buffers/docs/javatutorial)
