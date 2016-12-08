---
title: 在Android中使用Protocol Buffers
date: 2016-12-2 20:36:49
tags:
- 网络
- Android
---

网络性能优化的终极手法就是不通过网络传输，但这常常是不可能的。但我们还是可以通过对网络传输的数据本身做优化，来获得更好的性能，性能就应该从每一个可能的地方榨取。这里来看一下 **Protocol Buffers** 。

**Protocol Buffers** 是一个序列化结构数据的灵活、高效且自动化的机制——类似于XML，但***更小***，***更快***，***更简单***。定义一次结构化数据的方式，然后就可以使用专门生成的代码简单地写入，或用不同的语言从大量的数据流读出结构化数据。甚至可以更新数据结构而不破坏已部署的基于 **老** 格式编译的程序。我们看一下要如何将 **Protocol Buffers** 用到我们的Android项目中。

<!--more-->

#总览
先来看一下 **Protocol Buffers** 项目已经为我们提供了什么，我们在使用 **Protocol Buffers** 时需要做什么的整体流程。如下图：

![Protocol Buffers Architecture](http://upload-images.jianshu.io/upload_images/1315506-b9bc1c0cc87c80e4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在使用 **Protocol Buffers** 时，我们需要以特殊的方式定义我们的结构化数据，保存为 .proto 消息定义文件。 **Protocol Buffers** 项目为我们提供了编译器，可以将 .proto 文件编译为Java文件以用于我们的Java 或 Android应用项目。这个编译器在我们的PC机上编译并运行。产生的Java文件是依赖于 **Protocol Buffers** 的Java库的，比如这些文件实现了库的借口等。我们将生成的这些Java文件和 **Protocol Buffers** 的Java库引入我们的Android应用项目，就可以方便地以 **Protocol Buffers** 的二进制格式操作结构化数据了。

每次手动执行 **Protocol Buffers** 编译器将 .proto 文件转换为Java文件显然有点太麻烦了。 **Protocol Buffers** 项目的开发者显然也想到了这一点，因而他们还为我们提供了一个Android Studio gradle插件 **protobuf-gradle-plugin** ，以便于在我们项目的编译期间自动地执行 **Protocol Buffers** 编译器。

我们可以为 **protobuf-gradle-plugin** 指定本地 **Protocol Buffers** 编译器的路径让它使用本地的编译执行编译，也可以使用 **Protocol Buffers** 项目提供的另外一个工具，在编译时动态地下载并执行编译过程。

后面我们详细地来看这个过程。

# 下载编译Protocol编译器

我们可以在如下位置：
```
https://github.com/google/protobuf/releases
```
下载打包好的protobuf，也可以直接clone protobuf的代码，自己手动编译编译器。这里我们从GitHub上clone代码并手动编译编译器：
```
$ git clone https://github.com/google/protobuf.git
正克隆到 'protobuf'...
remote: Counting objects: 38993, done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 38993 (delta 4), reused 0 (delta 0), pack-reused 38974
接收对象中: 100% (38993/38993), 36.14 MiB | 239.00 KiB/s, 完成.
处理 delta 中: 100% (26220/26220), 完成.
检查连接... 完成。
```
下载代码之后，进入protobuf目录并执行 **autogen.sh** ：
```
$ cd protobuf
$ ./autogen.sh 
Google Mock not present.  Fetching gmock-1.7.0 from the web...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   129    0   129    0     0    111      0 --:--:--  0:00:01 --:--:--   112
100  362k  100  362k    0     0  67764      0  0:00:05  0:00:05 --:--:-- 92816
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   129    0   129    0     0    112      0 --:--:--  0:00:01 --:--:--   112
100  618k  100  618k    0     0  56321      0  0:00:11  0:00:11 --:--:--  115k
+ autoreconf -f -i -Wall,no-obsolete
libtoolize: putting auxiliary files in AC_CONFIG_AUX_DIR, 'build-aux'.
......
```
这个脚本主要用于下载测试用的gmock-1.7.0，并生成用于编译配置的 **configure** 等文件。可以通过如下命令了解我们可以对protobuf的编译做哪些配置，以及默认配置的信息：
```
$ ./configure --help
`configure' configures Protocol Buffers 3.1.0 to adapt to many kinds of systems.

Usage: ./configure [OPTION]... [VAR=VALUE]...

To assign environment variables (e.g., CC, CFLAGS...), specify them as
VAR=VALUE.  See below for descriptions of some of the useful variables.

Defaults for the options are specified in brackets.

Configuration:
  -h, --help              display this help and exit
      --help=short        display options specific to this package
      --help=recursive    display the short help of all the included packages
  -V, --version           display version information and exit
  -q, --quiet, --silent   do not print `checking ...' messages
      --cache-file=FILE   cache test results in FILE [disabled]
  -C, --config-cache      alias for `--cache-file=config.cache'
  -n, --no-create         do not create output files
      --srcdir=DIR        find the sources in DIR [configure dir or `..']

Installation directories:
  --prefix=PREFIX         install architecture-independent files in PREFIX
                          [/usr/local]
  --exec-prefix=EPREFIX   install architecture-dependent files in EPREFIX
                          [PREFIX]

By default, `make install' will install all the files in
`/usr/local/bin', `/usr/local/lib' etc.  You can specify
an installation prefix other than `/usr/local' using `--prefix',
for instance `--prefix=$HOME'.

For better control, use the options below.

Fine tuning of the installation directories:
  --bindir=DIR            user executables [EPREFIX/bin]
......
```
执行configure对编译进行配置：
```
$ ./configure 
checking whether to enable maintainer-specific portions of Makefiles... yes
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking target system type... x86_64-pc-linux-gnu
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /bin/mkdir -p
......
```
这样就生成了makefile文件，编译并安装：
```
$ make
$ sudo make install
```
这个过程在编译并安装 **Protocol Buffers** 编译器之外，还会为host编译用于支持在C++中使用 **Protocol Buffers** 的库。（编译生成的二进制文加在 **protobuf/src/.libs** 下。）

安装之后执行如下命令以确认已经装好：
```
$ protoc --version
libprotoc 3.1.0
```

在执行protoc时通过给它加上 **--help** 参数可以了解到这个工具更多的用法。
```
$ protoc --help
Usage: protoc [OPTION] PROTO_FILES
Parse PROTO_FILES and generate output based on the options given:
  -IPATH, --proto_path=PATH   Specify the directory in which to search for
                              imports.  May be specified multiple times;
                              directories will be searched in order.  If not
                              given, the current working directory is used.
  --version                   Show version info and exit.
  -h, --help                  Show this text and exit.
  --encode=MESSAGE_TYPE       Read a text-format message of the given type
                              from standard input and write it in binary
                              to standard output.  The message type must
                              be defined in PROTO_FILES or their imports.
  --decode=MESSAGE_TYPE       Read a binary message of the given type from
                              standard input and write it in text format
                              to standard output.  The message type must
                              be defined in PROTO_FILES or their imports.
  --decode_raw                Read an arbitrary protocol message from
                              standard input and write the raw tag/value
                              pairs in text format to standard output.  No
                              PROTO_FILES should be given when using this
                              flag.
  -oFILE,                     Writes a FileDescriptorSet (a protocol buffer,
    --descriptor_set_out=FILE defined in descriptor.proto) containing all of
                              the input files to FILE.
  --include_imports           When using --descriptor_set_out, also include
                              all dependencies of the input files in the
                              set, so that the set is self-contained.
  --include_source_info       When using --descriptor_set_out, do not strip
                              SourceCodeInfo from the FileDescriptorProto.
                              This results in vastly larger descriptors that
                              include information about the original
                              location of each decl in the source file as
                              well as surrounding comments.
  --dependency_out=FILE       Write a dependency output file in the format
                              expected by make. This writes the transitive
                              set of input file paths to FILE
  --error_format=FORMAT       Set the format in which to print errors.
                              FORMAT may be 'gcc' (the default) or 'msvs'
                              (Microsoft Visual Studio format).
  --print_free_field_numbers  Print the free field numbers of the messages
                              defined in the given proto files. Groups share
                              the same field number space with the parent 
                              message. Extension ranges are counted as 
                              occupied fields numbers.

  --plugin=EXECUTABLE         Specifies a plugin executable to use.
                              Normally, protoc searches the PATH for
                              plugins, but you may specify additional
                              executables not in the path using this flag.
                              Additionally, EXECUTABLE may be of the form
                              NAME=PATH, in which case the given plugin name
                              is mapped to the given executable even if
                              the executable's own name differs.
  --cpp_out=OUT_DIR           Generate C++ header and source.
  --csharp_out=OUT_DIR        Generate C# source file.
  --java_out=OUT_DIR          Generate Java source file.
  --javanano_out=OUT_DIR      Generate Java Nano source file.
  --js_out=OUT_DIR            Generate JavaScript source.
  --objc_out=OUT_DIR          Generate Objective C header and source.
  --php_out=OUT_DIR           Generate PHP source file.
  --python_out=OUT_DIR        Generate Python source file.
  --ruby_out=OUT_DIR          Generate Ruby source file.
```

# 创建 .proto 文件

 .proto 文件中的定义很简单：为每个想要序列化的数据结构添加一个 *消息(message)* ，然后为消息中的每个字段指定一个名字和类型以及一个tag数字。如官方提供的一个例子addressbook.proto：
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
可以参考 [在Java中使用Protocol Buffers](http://www.jianshu.com/p/1bf426a9f8f4) 一文了解更多关于创建 .proto 文件的基础知识。

# 编译 .proto 文件
可以通过如下命令编译 .proto 文件：
```
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```
-I，--java_out 分别用于指定源目录 (放置应用程序源代码的地方 —— 如果没有提供则使用当前目录)，目的目录 (希望放置生成的代码的位置；通常与$SRC_DIR相同)，最后的参数为 .proto 文件的路径。protoc会按照标准Java风格，生成Java类及目录结构。如对于上面的例子，会生成 **com/example/tutorial/** 目录结构，及 **AddressBookProtos.java** 文件。

# 在Android项目中使用 Protocol Buffers

我们将 由 .proto 文件生成的Java文件复制到我们的Android项目中：


![](http://upload-images.jianshu.io/upload_images/1315506-a61387d76b3fb420.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在我们app的build.gradle中添加对 **protobuf-java** 的依赖，就像依赖其它那些Java库一样：
```
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'

    compile 'com.android.support:appcompat-v7:23.4.0'
    compile 'com.google.protobuf:protobuf-java:3.0.0'
}
```

添加访问Protocol Buffers的类的类。这里我们添加两个类，AddPerson用于构造Person对象：
```
package com.netease.volleydemo;

import com.example.tutorial.AddressBookProtos.Person;

public class AddPerson {
    static Person createPerson(String personName) {
        Person.Builder person = Person.newBuilder();

        int id = 13958235;
        person.setId(id);

        String name = personName;
        person.setName(name);

        String email = "zhangsan@gmail.com";
        person.setEmail(email);

        Person.PhoneNumber.Builder phoneNumber = Person.PhoneNumber.newBuilder();
        phoneNumber.setType(Person.PhoneType.HOME);
        phoneNumber.setNumber("0157-23443276");

        person.addPhone(phoneNumber.build());

        phoneNumber = Person.PhoneNumber.newBuilder();
        phoneNumber.setType(Person.PhoneType.MOBILE);
        phoneNumber.setNumber("136183667387");

        person.addPhone(phoneNumber.build());

        return person.build();
    }
}
```

AddressBookProtobuf类则用于编码/解码AddressBook对象：
```
package com.netease.volleydemo;

import com.example.tutorial.AddressBookProtos;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;

public class AddressBookProtobuf {
    public static byte[] encodeTest(String[] names) {

        AddressBookProtos.AddressBook.Builder addressBook = AddressBookProtos.AddressBook.newBuilder();

        for(int i = 0; i < names.length; ++ i) {
            addressBook.addPerson(AddPerson.createPerson(names[i]));
        }
        AddressBookProtos.AddressBook book = addressBook.build();
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try {
            book.writeTo(baos);
        } catch (IOException e) {
        }

        return baos.toByteArray();
    }


    public static byte[] encodeTest(String[] names, int times) {
        for (int i = 0; i < times - 1; ++ i) {
            encodeTest(names);
        }
        return encodeTest(names);
    }

    public static AddressBookProtos.AddressBook decodeTest(InputStream is) {
        AddressBookProtos.AddressBook addressBook = null;
        try {
            addressBook = AddressBookProtos.AddressBook.parseFrom(is);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return addressBook;
    }

    public static AddressBookProtos.AddressBook decodeTest(InputStream is, int times) {
        AddressBookProtos.AddressBook addressBook = null;
        for (int i = 0; i < times; ++ i) {
            addressBook = decodeTest(is);
        }
        return addressBook;
    }
}
```

# 使用protobuf-gradle-plugin
每次单独执行protoc编译 .proto 文件总是太麻烦，通过protobuf-gradle-plugin可以在编译我们的app时自动地编译 .proto 文件，这样就大大降低了我们在Android项目中使用 **Protocol Buffers** 的难度。

首先我们需要将 .proto 文件添加进我们的项目中，如：
![](http://upload-images.jianshu.io/upload_images/1315506-ec51f69a6134805e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后修改 **app/build.gradle** 对protobuf gradle插件做配置：
1. 为buildscript添加对`protobuf-gradle-plugin`的依赖：
```
buildscript {
    repositories {
        jcenter()
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.0'
    }
}
```
2. 在`apply plugin: 'com.android.application'`后面应用protobuf的plugin：
```
apply plugin: 'com.android.application'
apply plugin: 'com.google.protobuf'
```
3. 添加protobuf块，对protobuf-gradle-plugin的执行做配置：
```
protobuf {
    protoc {
        path = '/usr/local/bin/protoc'
    }

    generateProtoTasks {
        all().each { task ->
            task.builtins {
                remove java
            }
            task.builtins {
                java { }
                // Add cpp output without any option.
                // DO NOT omit the braces if you want this builtin to be added.
                cpp { }
            }
        }
    }
}
```
`protoc`块用于配置Protocol Buffers编译器，这里我们指定用我们之前手动编译的编译器。
`task.builtins`的块必不可少，这个块用于指定我们要为那些编程语言生成代码，这里我们为C++和Java生成代码。缺少这个块的话，在编译时会报出如下的错误：
```
Information:Gradle tasks [:app:generateDebugSources, :app:mockableAndroidJar, :app:prepareDebugUnitTestDependencies, :app:generateDebugAndroidTestSources, :netlib:generateDebugSources, :netlib:mockableAndroidJar, :netlib:prepareDebugUnitTestDependencies, :netlib:generateDebugAndroidTestSources]
Error:Execution failed for task ':app:generateDebugProto'.
> protoc: stdout: . stderr: /media/data/CorpProjects/netlibdemo/app/build/extracted-protos/main: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/src/debug/proto: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/build/extracted-protos/debug: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/build/extracted-include-protos/debug: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/src/debug/proto: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/build/extracted-protos/debug: warning: directory does not exist.
  /media/data/CorpProjects/netlibdemo/app/build/extracted-include-protos/debug: warning: directory does not exist.
  Missing output directives.
```
提示说没有指定输出目录的路径。
这是由于 protobuf-gradle-plugin 执行的protobuf编译器命令的参数是在`protobuf-gradle-plugin/src/main/groovy/com/google/protobuf/gradle/GenerateProtoTask.groovy`中构造的：
```
    def cmd = [ tools.protoc.path ]
    cmd.addAll(dirs)

    // Handle code generation built-ins
    builtins.each { builtin ->
      String outPrefix = makeOptionsPrefix(builtin.options)
      cmd += "--${builtin.name}_out=${outPrefix}${getOutputDir(builtin)}"
    }

    // Handle code generation plugins
    plugins.each { plugin ->
      String name = plugin.name
      ExecutableLocator locator = tools.plugins.findByName(name)
      if (locator == null) {
        throw new GradleException("Codegen plugin ${name} not defined")
      }
      String pluginOutPrefix = makeOptionsPrefix(plugin.options)
      cmd += "--${name}_out=${pluginOutPrefix}${getOutputDir(plugin)}"
      cmd += "--plugin=protoc-gen-${name}=${locator.path}"
    }

    if (generateDescriptorSet) {
      def path = getDescriptorPath()
      // Ensure that the folder for the descriptor exists;
      // the user may have set it to point outside an existing tree
      def folder = new File(path).parentFile
      if (!folder.exists()) {
        folder.mkdirs()
      }
      cmd += "--descriptor_set_out=${path}"
      if (descriptorSetOptions.includeImports) {
        cmd += "--include_imports"
      }
      if (descriptorSetOptions.includeSourceInfo) {
        cmd += "--include_source_info"
      }
    }

    cmd.addAll protoFiles
    logger.log(LogLevel.INFO, cmd.toString())
    def stdout = new StringBuffer()
    def stderr = new StringBuffer()
    Process result = cmd.execute()
    result.waitForProcessOutput(stdout, stderr)
    def output = "protoc: stdout: ${stdout}. stderr: ${stderr}"
    logger.log(LogLevel.INFO, cmd)
    if (result.exitValue() == 0) {
      logger.log(LogLevel.INFO, output)
    } else {
      throw new GradleException(output)
    }
```
可以看到，输出目录是由builtins构造的。
4. 指定 .proto 文件的路径
```
    sourceSets {
        main {
            java {
                srcDir 'src/main/java'
            }
            proto {
                srcDir 'src/main/proto'
            }
        }
    }
```
这样我们就不用那么麻烦每次手动执行protoc了。

对前面的protobuf块做一点点修改，我们甚至来编译protobuf编译器都不需要了。修改如下：
```
protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.0.0'
    }

    generateProtoTasks {
        all().each { task ->
            task.builtins {
                remove java
            }
            task.builtins {
                java { }
                cpp { }
            }
        }
    }
}
```
关于 .proto 文件的编写方法，Protocol Buffers API等更多内容，可以参考 [Protobuf开发者指南](http://www.jianshu.com/p/3ab14a2cb477)、[在Java中使用Protocol Buffers](http://www.jianshu.com/p/1bf426a9f8f4)及其它相关官方文档。

# Protobuf 与 JSON 对比测试
说了半天，Protobuf的表现究竟如何呢？这里我们就对比一下我们最常用到的JSON格式与Protobuf的表现。测试基于在开发者中一向有着良好口碑的fastjson进行。

测试用的数据结构如我们前面看到的AddressBook。我们通过构造包含不同个数Person的AddressBook数据，并对这些数据执行多次编码解码操作，来测试Protobuf 与 JSON的表现。Protobuf的编码/解码测试代码如前面看到的AddressBookProtobuf。JSON的测试代码则如下面这样：
```
package com.netease.volleydemo;

import com.alibaba.fastjson.JSON;

import java.util.ArrayList;
import java.util.List;

public class AddressBookJson {
    private enum PhoneType {
        MOBILE,
        HOME,
        WORK
    }

    private static final class Phone {
        private String number;
        private PhoneType type;

        public Phone() {

        }

        public void setNumber(String number) {
            this.number = number;
        }

        public String getNumber() {
            return number;
        }

        public void setType(PhoneType phoneType) {
            this.type = phoneType;
        }

        public PhoneType getType() {
            return type;
        }
    }
    private static final class Person {
        private String name;
        private int id;
        private String email;

        private List<Phone> phones;

        public Person() {
            phones = new ArrayList<>();
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getName() {
            return name;
        }

        public void setId(int id) {
            this.id = id;
        }

        public int getId() {
            return id;
        }

        public void setEmail(String email) {
            this.email = email;
        }

        public String getEmail() {
            return email;
        }

        public void addPhone(Phone phone) {
            phones.add(phone);
        }

        public List<Phone> getPhones() {
            return phones;
        }
    }

    private static final class AddressBook {
        private List<Person> persons;

        public AddressBook() {
            persons = new ArrayList<>();
        }

        public void addPerson(Person person) {
            persons.add(person);
        }

        public List<Person> getPersons() {
            return persons;
        }
    }

    public static String encodeTest(String[] names) {
        AddressBook addressBook = new AddressBook();
        for (int i = 0; i < names.length; ++ i) {
            Person person = new Person();
            person.setName(names[i]);
            person.setEmail("zhangsan@gmail.com");
            person.setId(13958235);

            Phone phone = new Phone();
            phone.setNumber("0157-23443276");
            phone.setType(PhoneType.HOME);
            person.addPhone(phone);

            phone = new Phone();
            phone.setNumber("136183667387");
            phone.setType(PhoneType.MOBILE);
            person.addPhone(phone);

            addressBook.addPerson(person);
        }
        String jsonString = JSON.toJSONString(addressBook);
        return jsonString;
    }

    public static String encodeTest(String[] names, int times) {
        for (int i = 0; i < times - 1; ++ i) {
            encodeTest(names);
        }
        return encodeTest(names);
    }

    public static AddressBook decodeTest(String jsonStr, int times) {
        AddressBook addressBook = null;
        for (int i = 0; i < times; ++ i) {
            addressBook = JSON.parseObject(jsonStr, AddressBook.class);
        }
        return addressBook;
    }
}
```

通过如下的这段代码来执行测试：
```
    private class ProtobufTestTask extends AsyncTask<Void, Void, Void> {
        private static final int BUFFER_LEN = 8192;

        private void doEncodeTest(String[] names, int times) {
            long startTime = System.nanoTime();
            AddressBookProtobuf.encodeTest(names, times);
            long protobufTime = System.nanoTime();
            protobufTime = protobufTime - startTime;

            startTime = System.nanoTime();
            AddressBookJson.encodeTest(names, times);
            long jsonTime = System.nanoTime();
            jsonTime = jsonTime - startTime;
            Log.i(TAG, String.format("%-20s%-20s%-20s%-20s", "ProtobufTime", String.valueOf(protobufTime),
                    "JsonTime", String.valueOf(jsonTime)));
        }

        private void doEncodeTest10(int times) {
            doEncodeTest(TestUtils.sTestNames10, times);
        }

        private void doEncodeTest50(int times) {
            doEncodeTest(TestUtils.sTestNames50, times);
        }

        private void doEncodeTest100(int times) {
            doEncodeTest(TestUtils.sTestNames100, times);
        }

        private void doEncodeTest(int times) {
            doEncodeTest10(times);
            doEncodeTest50(times);
            doEncodeTest100(times);
        }

        private void compress(InputStream is, OutputStream os)
                throws Exception {

            GZIPOutputStream gos = new GZIPOutputStream(os);

            int count;
            byte data[] = new byte[BUFFER_LEN];
            while ((count = is.read(data, 0, BUFFER_LEN)) != -1) {
                gos.write(data, 0, count);
            }

            gos.finish();
            gos.close();
        }

        private void doDecodeTest(String[] names, int times) {
            byte[] protobufBytes = AddressBookProtobuf.encodeTest(names);
            ByteArrayInputStream bais = new ByteArrayInputStream(protobufBytes);

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try {
                compress(bais, baos);
            } catch (Exception e) {
                e.printStackTrace();
            }
            Log.i(TAG, String.format("%-20s%-20s%-20s%-20s", "Protobuf Length", String.valueOf(protobufBytes.length),
                    "Protobuf(GZIP) Length", String.valueOf(baos.toByteArray().length)));

            bais = new ByteArrayInputStream(protobufBytes);
            long startTime = System.nanoTime();
            AddressBookProtobuf.decodeTest(bais, times);
            long protobufTime = System.nanoTime();
            protobufTime = protobufTime - startTime;

            String jsonStr = AddressBookJson.encodeTest(names);
            ByteArrayInputStream jsonBais = new ByteArrayInputStream(jsonStr.getBytes());
            ByteArrayOutputStream jsonBaos = new ByteArrayOutputStream();
            try {
                compress(jsonBais, jsonBaos);
            } catch (Exception e) {
                e.printStackTrace();
            }
            Log.i(TAG, String.format("%-20s%-20s%-20s%-20s", "Json Length", String.valueOf(jsonStr.getBytes().length),
                    "Json(GZIP) Length", String.valueOf(jsonBaos.toByteArray().length)));

            startTime = System.nanoTime();
            AddressBookJson.decodeTest(jsonStr, times);
            long jsonTime = System.nanoTime();
            jsonTime = jsonTime - startTime;

            Log.i(TAG, String.format("%-20s%-20s%-20s%-20s", "ProtobufTime", String.valueOf(protobufTime),
                    "JsonTime", String.valueOf(jsonTime)));
        }

        private void doDecodeTest10(int times) {
            doDecodeTest(TestUtils.sTestNames10, times);
        }

        private void doDecodeTest50(int times) {
            doDecodeTest(TestUtils.sTestNames50, times);
        }

        private void doDecodeTest100(int times) {
            doDecodeTest(TestUtils.sTestNames100, times);
        }

        private void doDecodeTest(int times) {
            doDecodeTest10(times);
            doDecodeTest50(times);
            doDecodeTest100(times);
        }

        @Override
        protected Void doInBackground(Void... params) {
            TestUtils.initTest();
            doEncodeTest(5000);

            doDecodeTest(5000);
            return null;
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            super.onPostExecute(aVoid);
        }
    }
```
这里我们执行3组编码测试及3组解码测试。对于编码测试，第一组的单个数据中包含10个Person，第二组的包含50个，第三组的包含100个，然后对每个数据分别执行5000次的编码操作。

对于解码测试，三组中单个数据同样包含10个Person、50个及100个，然后对每个数据分别执行5000次的解码码操作。

在Galaxy Nexus的Android 4.4.4 CM平台上执行上述测试，最终得到如下结果：

#### 编码后数据长度对比 (Bytes)

| Person个数  | Protobuf      | Protobuf（GZIP）  | JSON  | JSON（GZIP） |
| -----------------|:----------------:|--------------------------:|---------:|----------------------:|
|10                 |860               |291                           |1703    |344                     |
|50                 |4300             |984                           |8463    |1047                   |
|100               |8600             |1840                         |16913  |1913                   |

相同的数据，经过Protobuf编码的数据长度，大概只有JSON编码的数据长度的一半。但对编码后的数据再进行压缩，两者则差别比较小。

#### 编码性能对比 (S)
|Person个数|Protobuf   |JSON   |
|----      |-----------|-------|
|10        |4.687      |6.558  |
|50        |23.728     |41.315 |
|100       |45.604     |81.667 |
编码性能最少提高了 28.5%，最多则提高了44.2%。Protobuf在编码性能上，相对于JSON还是有较大幅度的提升的。

#### 解码性能对比 (S)
|Person个数|Protobuf   |JSON   |
|----      |-----------|-------|
|10        |0.226      |8.839  |
|50        |0.291      |43.869 |
|100       |0.220      |85.444 |

解码性能方面，Protobuf相对于JSON，则更是有惊人的提升。Protobuf的解码时间几乎不随着数据长度的增长而有太大的增长，而JSON则随着数据长度的增加，解码所需要的时间也越来越长。

Done。
