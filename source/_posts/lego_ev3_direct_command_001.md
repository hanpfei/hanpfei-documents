---
title: EV3 直接命令 - 第一课 无为的艺术
date: 2018-10-29 20:05:49
categories: ROS
tags:
- ROS
- 翻译
---

LEGO 的 EV3 是一个极好的游戏工具。它的标准编程方式是 LEGO 的图形化编程工具。你可以编写程序，把它们传到你的 EV3 brick 上，然后启动它们。但还有另外一种与你的 EV3 交互的方式。把它看作一个服务器并给它发送命令，命令将以数据和/或行为来应答。在这种情形下，你的程序所运行的机器是客户端。这打开了迷人的新视角。如果程序运行在你的智能手机上，你将获得很好的交互性和便利性。如果你的客户端是一个 PC 或笔记本电脑，你将获得舒服的键盘和显示器。另一种新选项是在一个机器人中结合多个 EV3。一个客户端与多个服务器通信，这允许机器人具有无限多个马达和传感器。或者把你的 EV3 当做一台机器，生产数据的那种。客户端可以持续地从 EV3 的传感器接收数据，这也将打开新的机会之门。如果你想进入这个新世界，你不得不使用 EV3 的直接命令，这需要你的一些工作。如果你准备投资它，则继续阅读，否则，开心地玩你的 EV3 并等待其它人做出很酷的新应用吧。
<!--more-->
在我们开始讨论正式的内容之前，我们先瞥一眼 EV3 的通信协议和 LEGO Direct Commands 的特性。你的 EV3 提供了三种通信类型，蓝牙，WiFI 和 USB。蓝牙和 USB 无需其它额外的设备即可使用，通过 WiFi 通信需要一个适配器。所有三种都允许你开发应用程序，运行在任何计算机上与你的 EV3 brick 通信。也许你了解 EV3 的前身，NXT 及其通信协议。它提供了大约 20 个系统命令，类似于函数调用。LEGO 改变了它的理念，EV3 以机器码提供了命令语法。一方面，这意味着实现真正算法的自由，但另一方面，它变得更难入门。

官方文档，我发现，是纯粹的技术文档。他们缺乏动力和教育方面。我希望这份文档能成为你深入你的 EV3 的适当入口。如果你已经是一个程序员，或试图称为一个程序员，你将沉浸在位和字节的世界。如果不是，它将变得困难，但唯一的选择是，保持双手分开。试一试，在与乐高 EV3 的有效沟通之下，你将学到很多关于机器代码，所有计算机的基础的知识。


# 要知道的文档

LEGO 已经发布了很好的详细的文档，你可以在这里下载：[http://www.lego.com/en-gb/mindstorms/downloads](http://www.lego.com/en-gb/mindstorms/downloads)。出于我们的目的，你绝对需要查看文档，你可以在标题为 *Advanced Users - Developer Kits（PC / MAC）* 的标题下找到。*[EV3 Firmware Developer Kit](https://www.lego.com/r/www/r/mindstorms/-/media/franchises/mindstorms%202014/downloads/firmware%20and%20software/advanced/lego%20mindstorms%20ev3%20firmware%20developer%20kit.pdf?l.r2=830923294)* 是 LEGO EV3 直接命令的参考书。我希望你能深入阅读它。

有一个 C# 的通信库，它使用直接命令与 LEGO EV3 通信。如果你喜欢使用开箱即用的软件，如果你喜欢 C#，这可能是你的选择：http://www.monobrick.dk。

我最初的想法是不发布任何源码。当功能逐步增长时，编程更有趣。Bugs 是游戏的一部分，搜索 bugs 很难，但它是每个编程人员的日常生活。简短的结论，代码增长到一定的规模和复杂性，最初的想法变得不现实。我已经在 Github 上发布了代码，你可以在这里下载： [ev3-python3](https://github.com/ChristophGaukel/ev3-python3)。

# 第一课 无为的艺术

你做到了，你真的想介入，那没很好！这一课是关于非常基本的通信的。我们将实现第一个调用和应答循环。通过 WiFi，蓝牙或 USB 给你的 EV3 发送信息，你将获得一个定义良好的应答。不要屏住呼吸，我们不会从一个令人惊奇的应用程序开始。相反，它什么都不做。这听起来不像是，如果你设法做到这一点，向后倾斜并感到快乐，你就在路上。

## 比特和字节命名的小游览

也许你已经知道了如何编写二进制和十六进制数字，大小尾端的含义等等。如果你真的可以把值 156 写为一个小尾端的 4 字节整数表示格式，则你可以跳过这一节。如果不能，你需要阅读它，因为你真的需要直到它。

让我们从基础开始！几乎所有现代计算机都将 8 位组合为 1 个字节，并且在内存中按字节寻址（你的 EV3 是一台现代计算机，因而也是这样）。在下文中我们使用如下表示法表示二进制数字：`0b 1001 1100`。

前导的 `0b` 告诉我们，后面是数字的二进制表示法，每个字节的 8 个数字被分为 4 和 4 的两个半个字节。它是数字 156 的二进制表示法，你可以把它读作：156 = 1*128 + 0*64 + 0*32 + 1*16 + 1*8 + 1*4 + 0*2 + 0*1。可以对相同字节进行替代解释。它可以被读做 8 个标记的序列，或它可以是符号 £ 的
ASCII 码。解释依赖于上下文。目前我们专注于数字。

二进制表示法非常长，因此常见的习惯是把半个字节写为 16 进制数，其中字母 A 到 F 表示数字 10 到 15。十六进制表示法是紧凑的，它与二进制表示法的转换很容易。这是因为一个十六进制数字表示半个字节。十六进制数字（这里是值 156）表示是： 0x 9C。你可以把它读作：156 = 9*16 + 12*1。前导的 0x 告诉我们，后面是十六进制表示。因为它的紧凑型，我们可以编写和读更大的数字。作为一个 4 字节整数，值 156 被写为：0x 00 00 00 9C。

我们将用冒号 ":" 或竖条 "|" 把字节分开。我们使用竖线表示高阶分离，使用冒号表示低阶。我们将把值为 156 的 2 字节整数写作：0x|00:9C|。现在我们可以在一行中保存值列表。256（无符号 1 字节整数），156（2 字节整数）和 65536（4 字节整数）的序列可以写为：0x|FF|00:9C|00:01:00:00|。

那负数呢？大多数计算机语言区分有符号和无符号整数。如果整数是有符号的，则它们的第一位是负号标志，整数是另一个范围的。有符号 1 字节整数的范围是 -128 到 127，有符号 2 字节整数的范围是 -32,768 到 32,767 等等。负数的值的计算方法为最小的值（-128，-32,768 等）加上其余的值。有符号 1 字节整数的最小值，-128 写为 0b 1000 0000 或 0x|80|，有符号 2
 字节整数值 -1 (-32,768 + 32,767) 为：0b 1111 1111 1111 1111 或 0x|FF:FF|

那什么是 [小尾端](https://en.wikipedia.org/wiki/Endianness) 呢？OK，我不再保守这个秘密了。小尾端格式反转字节的位置（你常常使用的，被称作 *大尾端*）。2 字节整数值 156 以小尾端格式则被写作：0x|9C:00|。

也许这听起来像个糟糕的玩笑，但非常抱歉，EV3 直接命令读和写所有数字都是以小尾端进行的，那不是我的错。但我可以给你一些安慰。首先，本课程使用数字。其次，存在管理小端数的好工具。在 Python 中，你可以使用`struct` 模块，在 Java 中，`ByteBuffer` 可能是你选择的对象。

## 什么都不做的直接命令

第一个例子展示所有可能的直接命令中最简单的那个。你将向你的 EV3 发送一条消息，并期待它将有所应答。让我们看一下要发送的消息，它由如下 8 个字节组成：
```

-------------------------      
 \ len \ cnt \ty\ hd  \op\     
  -------------------------    
0x|06:00|2A:00|00|00:00|01|    
  -------------------------    
   \ 6   \ 42  \Re\ 0,0 \N \   
    \     \     \  \     \o \  
     \     \     \  \     \p \ 
      -------------------------
```

消息本身是以 0x 开头的那一行。在消息的顶部，你会看到有关消息部分类型的一些注释。底部显示有关其值的注释。消息的 8 个字节由如下的部分组成：

 * **消息的长度**(字节 0，1)：开头的两个字节不是直接命令本身的组成部分。它们是通信协议的一部分，在 EV3 的情况下可以是 Wifi，蓝牙或 USB。长度被编码为小尾端格式的 2 字节无符号整数，因此 `0x|06:00|` 表示值 6。

 * **消息计数器**(字节 2，3)：接下来的两个字节是这个直接命令的指纹。消息计数器将包含在应答中，并可以用来匹配直接命令和它的应答。这也是一个小尾端格式的 2 字节无符号整数。在我们的例子中把消息计数器设置为 `0x|2A:00|`，其值为 42。

 * **消息类型**(字节 4)：它可以是如下的两个值之一：
    - DIRECT_COMMAND_REPLY = 0x|00|
    - DIRECT_COMMAND_NO_REPLY = 0x|80|
在我们的例子中我们希望 EV3 应答消息。

 * **头部**(字节 5，6)：接下来的两个字节，第一个操作之前最后的部分是头部。它包含了两个数字，它们定义了直接命令的内存大小（是的，它是复数，我们有两个内存，一个局部的和一个全局的）。我们将很快回到这个内存大小的具体细节。此时我们很幸运，我们的命令不需要任何内存，因而我们把头部设置为 `0x|00:00|`。

 * **操作**(从字节 7 开始)：在我们的例子中是单个字节，它表示：`opNOP` = `0x|01|`，什么也不做，EV3 的 idle 操作。

## 给 EV3 发送消息

我们的任务是，发送上面描述的消息给 EV3。如何做到呢？你可以在三种通信协议中选择，蓝牙，Wifi 和 USB，且你可以选择支持至少一种通信协议的任何编程语言。下面我展示了 Python 和 Java 的例子。如果没有你喜爱的语言，将程序翻译成你最喜爱的计算机语言并将其发送给我会很棒。它们将被发布在这里。

### 蓝牙

你需要访问开启了蓝牙的计算机，且你需要开启你的 EV3 上的蓝牙。接下来你需要为两个设备做配对。这可以从 EV3 或者你的计算机发起。EV3 的用户指南中对此过程有所描述。如果需要帮助，你可以在网上找到教程，这里有 LEGO 页面的链接：[http://www.lego.com/en-gb/mindstorms/support/](http://www.lego.com/en-gb/mindstorms/support/)。配对过程将向你展示你的 EV3 的 MAC 地址。你需要注意它。此外，你也可以在你的 EV3 的显示器中，在 *Brick Info* / *ID* 下面读取 MAC 地址。

#### python

你需要完成如下的步骤：

 * 把代码复制到名为 *EV3_do_nothing_bluetooth.py* 的文件中。
 * 把 MAC 地址由 `00:16:53:42:2B:99` 变为你的 EV3 的值。
 * 打开一个终端并切换到你的程序的目录。
 * 通过键入 `python3 EV3_do_nothing_bluetooth.py` 运行它。

```
#!/usr/bin/env python3

import socket
import struct

class EV3():
    def __init__(self, host: str):
        self._socket = socket.socket(
            socket.AF_BLUETOOTH,
            socket.SOCK_STREAM,
            socket.BTPROTO_RFCOMM
        )
        self._socket.connect((host, 1))

    def __del__(self):
        if isinstance(self._socket, socket.socket):
            self._socket.close()

    def send_direct_cmd(self, ops: bytes, local_mem: int=0, global_mem: int=0) -> bytes:
        cmd = b''.join([
            struct.pack('<h', len(ops) + 5),
            struct.pack('<h', 42),
            DIRECT_COMMAND_REPLY,
            struct.pack('<h', local_mem*1024 + global_mem),
            ops
        ])
        self._socket.send(cmd)
        print_hex('Sent', cmd)
        reply = self._socket.recv(5 + global_mem)
        print_hex('Recv', reply)
        return reply

def print_hex(desc: str, data: bytes) -> None:
    print(desc + ' 0x|' + ':'.join('{:02X}'.format(byte) for byte in data) + '|')

DIRECT_COMMAND_REPLY = b'\x00'
opNop = b'\x01'
my_ev3 = EV3('00:16:53:42:2B:99')
ops_nothing = opNop
my_ev3.send_direct_cmd(ops_nothing)
```

[socket](https://docs.python.org/3/library/socket.html) 的实现依赖于你的计算机的操作系统。如果 `AF_BLUETOOTH` 不被支持（你将看到一个错误消息如 *AttributeError: module 'socket' has no attribute 'AF_BLUETOOTH'*）。你可以使用 `pybluez`，那意味着你需要导入 `bluetooth` 而不是 `socket`。在我的情况下那是说：

 * 用 pip3 安装 [pybluez](https://github.com/karulis/pybluez)：`sudo pip3 install pybluez`
安装 pybluez 有两个前提条件：一是系统中配置的当前默认 Python 是 Python 3，这可以通过 `$ sudo update-alternatives --config python` 完成；二是已经安装了蓝牙开发包 libbluetooth-dev，这可以通过执行命令 `$ sudo apt-get install libbluetooth-dev` 完成，否则在安装 pybluez 时将报出如下错误：
```
    creating build/lib.linux-x86_64-3.5
    creating build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/osx.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/__init__.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/btcommon.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/msbt.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/widcomm.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/ble.py -> build/lib.linux-x86_64-3.5/bluetooth
    copying bluetooth/bluez.py -> build/lib.linux-x86_64-3.5/bluetooth
    running build_ext
    building 'bluetooth._bluetooth' extension
    creating build/temp.linux-x86_64-3.5
    creating build/temp.linux-x86_64-3.5/bluez
    x86_64-linux-gnu-gcc -pthread -DNDEBUG -g -fwrapv -O2 -Wall -Wstrict-prototypes -g -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -fPIC -I./port3 -I/usr/include/python3.5m -c bluez/btmodule.c -o build/temp.linux-x86_64-3.5/bluez/btmodule.o
    In file included from bluez/btmodule.c:20:0:
    bluez/btmodule.h:5:33: fatal error: bluetooth/bluetooth.h: 没有那个文件或目录
    compilation terminated.
    error: command 'x86_64-linux-gnu-gcc' failed with exit status 1
    
    ----------------------------------------
Command "/usr/bin/python -u -c "import setuptools, tokenize;__file__='/tmp/pip-install-mk3f5p_f/pybluez/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-record-gpk9_xhc/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /tmp/pip-install-mk3f5p_f/pybluez/
```

 * 修改程序：
```
#!/usr/bin/env python3

import bluetooth
import struct

class EV3():
    def __init__(self, host: str):
        self._socket = bluetooth.BluetoothSocket(bluetooth.RFCOMM)
        self._socket.connect((host, 1))

    def __del__(self):
        if isinstance(self._socket, bluetooth.BluetoothSocket):
            self._socket.close()
...
```

 * 运行程序

#### Java

与蓝牙设备通信，我的选择是 *bluecove*。下载了 Java 包 *bluecove-2.1.0.jar*（在 Unix 上也可以是 *bluecove-gpl-2.1.0.jar*） 之后，你可以把它们添加到你的 classpath 上。在我的 Unix 机器上，这通过这个命令来完成：
```
export CLASSPATH=$CLASSPATH:./bluecove-2.1.0.jar:./bluecove-gpl-2.1.0.jar
```
*bluecove-2.1.0.jar* 的下载地址为 https://mvnrepository.com/artifact/net.sf.bluecove/bluecove/2.1.0，*bluecove-gpl-2.1.0.jar* 的下载地址为 https://mvnrepository.com/artifact/net.sf.bluecove/bluecove-gpl/2.1.0。
然后，执行下面的步骤：

 * 把下面的代码拷贝到名为 *EV3_do_nothing_bluetooth.java* 的文件中。
 * 把 MAC 地址由 `001653422B99` 修改为你的 EV3 的值。
 * 打开一个终端并切换到你的程序的目录。
 * 键入 `javac EV3_do_nothing_bluetooth.java` 编译它。
 * 键入 `java EV3_do_nothing_bluetooth` 运行它。

```
import java.io.*;
import javax.microedition.io.*;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;

import java.io.*;

public class EV3_do_nothing_bluetooth {
    static final String mac_addr = "001653422B99";

    static final byte  opNop                        = (byte)  0x01;
    static final byte  DIRECT_COMMAND_REPLY         = (byte)  0x00;

    static InputStream in;
    static OutputStream out;

    public static void connectBluetooth ()
 throws IOException {
     String s = "btspp://" + mac_addr + ":1";
     StreamConnection c = (StreamConnection) Connector.open(s);
     in = c.openInputStream();
     out = c.openOutputStream();
    }

    public static ByteBuffer sendDirectCmd (ByteBuffer operations,
         int local_mem, int global_mem)
 throws IOException {
 ByteBuffer buffer = ByteBuffer.allocateDirect(operations.position() + 7);
 buffer.order(ByteOrder.LITTLE_ENDIAN);
 buffer.putShort((short) (operations.position() + 5));   // length
 buffer.putShort((short) 42);                            // counter
 buffer.put(DIRECT_COMMAND_REPLY);                       // type
 buffer.putShort((short) (local_mem*1024 + global_mem)); // header
 for (int i=0; i < operations.position(); i++) {         // operations
     buffer.put(operations.get(i));
 }

 byte[] cmd = new byte [buffer.position()];
 for (int i=0; i<buffer.position(); i++) cmd[i] = buffer.get(i);
 out.write(cmd);
 printHex("Sent", buffer);

 byte[] reply = new byte[global_mem + 5];
 in.read(reply);
 buffer = ByteBuffer.wrap(reply);
 buffer.position(reply.length);
 printHex("Recv", buffer);

 return buffer;
    }

    public static void printHex(String desc, ByteBuffer buffer) {
 System.out.print(desc + " 0x|");
 for (int i= 0; i < buffer.position() - 1; i++) {
     System.out.printf("%02X:", buffer.get(i));
 }
 System.out.printf("%02X|", buffer.get(buffer.position() - 1));
 System.out.println();
    }

    public static void main (String args[] ) {
 try {
     connectBluetooth();

     ByteBuffer operations = ByteBuffer.allocateDirect(1);
     operations.put(opNop);

     ByteBuffer reply = sendDirectCmd(operations, 0, 0);
 }
 catch (Exception e) {
     e.printStackTrace(System.err);
 }
    }
}
```

### Wifi

你需要一个 Wifi 适配器将你的 EV3 连接到你的本地网络。下面文档的第一部分描述了这个过程：[http://www.monobrick.dk/guides/how-to-establish-a-wifi-connection-with-the-ev3-brick/](http://www.monobrick.dk/guides/how-to-establish-a-wifi-connection-with-the-ev3-brick/)。现在你的 EV3 是本地网络的一部分，且具有一个网络地址了。从网络中的所有机器你都可以与之通信。如上面提到的文档所描述的那样，它需要如下的步骤来与 EV3 建立 TCP/IP 连接：

 * 在端口 3015 监听来自于 EV3 的 UDP 广播。
 * 向 EV3 发回一个 UDP 消息以使它接受一个 TCP/IP 连接。
 * 在端口 5555 上建立一个 TCP/IP 连接。
 * 通过 TCP/IP 给 EV3 发送一条解锁消息。

#### Python

你需要完成如下的步骤：

 * 把代码复制到名为 *EV3_do_nothing_wifi.py* 的文件中。
 * 打开一个终端，并切换到你的程序的目录。
 * 键入 `python3 EV3_do_nothing_wifi.py` 运行它。

```
#!/usr/bin/env python3

import socket
import struct
import re

class EV3():
    def __init__(self, host: str):
        # listen on port 3015 for a UDP broadcast from the EV3
        UDPSock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        UDPSock.bind(('', 3015))
        data, addr = UDPSock.recvfrom(67)

        # pick serial number, port, name and protocol
        # from the broadcast message
        matcher = re.search('Serial-Number: (\w*)\s\n' +
                            'Port: (\d{4,4})\s\n' +
                            'Name: (\w+)\s\n' +
                            'Protocol: (\w+)\s\n',
                            data.decode('utf-8'))
        serial_number = matcher.group(1)
        port = matcher.group(2)
        name = matcher.group(3)
        protocol = matcher.group(4)
        if serial_number.upper() != host.replace(':', '').upper():
            self._socket = None
            raise ValueError('found ev3 but not ' + host)

        # Send an UDP message back to the EV3
        # to make it accept a TCP/IP connection
        UDPSock.sendto(' '.encode('utf-8'), (addr[0], int(port)))
        UDPSock.close()

        # Establish a TCP/IP connection with EV3s address and port
        self._socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self._socket.connect((addr[0], int(port)))

        # Send an unlock message to the EV3 over TCP/IP
        msg = ''.join([
            'GET /target?sn=' + serial_number + 'VMTP1.0\n',
            'Protocol: ' + protocol
        ])
        self._socket.send(msg.encode('utf-8'))
        reply = self._socket.recv(16).decode('utf-8')
        if not reply.startswith('Accept:EV340'):
            raise IOError('No wifi connection to ' + name + ' established')

    def __del__(self):
        if isinstance(self._socket, socket.socket):
            self._socket.close()

    def send_direct_cmd(self, ops: bytes, local_mem: int=0, global_mem: int=0) -> bytes:
        cmd = b''.join([
            struct.pack('<h', len(ops) + 5),
            struct.pack('<h', 42),
            DIRECT_COMMAND_REPLY,
            struct.pack('<h', local_mem*1024 + global_mem),
            ops
        ])
        self._socket.send(cmd)
        print_hex('Sent', cmd)
        reply = self._socket.recv(5 + global_mem)
        print_hex('Recv', reply)
        return reply

def print_hex(desc: str, data: bytes) -> None:
    print(desc + ' 0x|' + ':'.join('{:02X}'.format(byte) for byte in data) + '|')

DIRECT_COMMAND_REPLY = b'\x00'
opNop = b'\x01'
my_ev3 = EV3('00:16:53:42:2B:99')
ops_nothing = opNop
my_ev3.send_direct_cmd(ops_nothing)
```

#### Java

你需要完成如下的步骤：

 * 把代码复制到名为 *EV3_do_nothing_wifi.java* 的文件中。
 * 打开一个终端，并切换到你的程序的目录。
 * 键入 `javac EV3_do_nothing_wifi.java` 编译它。
 * 键入 `java EV3_do_nothing_wifi` 运行它。

```
import java.net.Socket;
import java.net.SocketException;
import java.net.ServerSocket;
import java.net.DatagramSocket;
import java.net.DatagramPacket;
import java.net.InetAddress;

import java.nio.ByteBuffer;
import java.nio.IntBuffer;
import java.nio.ByteOrder;

import java.io.*;
import java.util.regex.*;

public class EV3_do_nothing_wifi {
    static final byte  opNop                        = (byte)  0x01;
    static final byte  DIRECT_COMMAND_REPLY         = (byte)  0x00;

    static InputStream in;
    static OutputStream out;

    public static void connectWifi ()
 throws IOException, SocketException {
     // listen for a UDP broadcast from the EV3 on port 3015
     DatagramSocket listener = new DatagramSocket(3015);
     DatagramPacket packet_r = new DatagramPacket(new byte[67], 67);
     listener.receive(packet_r); // receive the broadcast message
     String broadcast_message = new String(packet_r.getData());
     /* pick serial number, port, name and protocol 
        from the broadcast message */
     Pattern broadcast_pattern = 
     Pattern.compile("Serial-Number: (\\w*)\\s\\n" +
                            "Port:\\s(\\d{4,4})\\s\\n" +
                            "Name:\\s(\\w+)\\s\\n" +
                            "Protocol:\\s(\\w+)\\s\\n");
     Matcher matcher = broadcast_pattern.matcher(broadcast_message);
     String serial_number, name, protocol;
     int port;
     if(matcher.matches()) {
  serial_number = matcher.group(1);
  port = Integer.valueOf(matcher.group(2));
  name = matcher.group(3);
  protocol = matcher.group(4);
     }
     else {
  throw new IOException("Unexpected Broadcast message: " + broadcast_message);
     }
     InetAddress adr = packet_r.getAddress();
     // connect the EV3 with its address and port
     listener.connect(adr, port);
     /* Send an UDP message back to the EV3 
        to make it accept a TCP/IP connection */
     listener.send(new DatagramPacket(new byte[1], 1));
     // close the UDP connection
     listener.close();

     // Establish a TCP/IP connection with EV3s address and port
     Socket socket = new Socket(adr, port);
     in     = socket.getInputStream();
     out    = socket.getOutputStream();

     // Send an unlock message to the EV3 over TCP/IP
     String unlock_message = "GET /target?sn=" + serial_number + 
                                    "VMTP1.0\n" + 
                                    "Protocol: " + protocol;
     out.write(unlock_message.getBytes());

     byte[] reply = new byte[16];           // read reply
     in.read(reply);
     if (! (new String(reply)).startsWith("Accept:EV340")) {
  throw new IOException("No wifi connection established " + name);
     } 
    }

    public static ByteBuffer sendDirectCmd (ByteBuffer operations,
         int local_mem, int global_mem)
 throws IOException {
 ByteBuffer buffer = ByteBuffer.allocateDirect(operations.position() + 7);
 buffer.order(ByteOrder.LITTLE_ENDIAN);
 buffer.putShort((short) (operations.position() + 5));   // length
 buffer.putShort((short) 42);                            // counter
 buffer.put(DIRECT_COMMAND_REPLY);                       // type
 buffer.putShort((short) (local_mem*1024 + global_mem)); // header
 for (int i=0; i<operations.position(); i++) {           // operations
     buffer.put(operations.get(i));
 }

 byte[] cmd = new byte [buffer.position()];
 for (int i=0; i<buffer.position(); i++) cmd[i] = buffer.get(i);
 out.write(cmd);
 printHex("Sent", buffer);

 byte[] reply = new byte[global_mem + 5];
 in.read(reply);
 buffer = ByteBuffer.wrap(reply);
 buffer.position(reply.length);
 printHex("Recv", buffer);

 return buffer;
    }

    public static void printHex(String desc, ByteBuffer buffer) {
 System.out.print(desc + " 0x|");
 for (int i= 0; i<buffer.position() - 1; i++) {
     System.out.printf("%02X:", buffer.get(i));
 }
 System.out.printf("%02X|", buffer.get(buffer.position() - 1));
 System.out.println();
    }

    public static void main (String args[] ) {
 try {
     connectWifi();

     ByteBuffer operations = ByteBuffer.allocateDirect(1);
     operations.put(opNop);

     ByteBuffer reply = sendDirectCmd(operations, 0, 0);
 }
 catch (Exception e) {
     e.printStackTrace(System.err);
 }
    }
}
```

### USB

通用串行总线是一个连接电子设备的工业标准。你的 EV3 具有一个 2.0 Mini -B 插口（标有 PC 字样）。这是性能最好的通信协议，但它需要一条线。在你运行你的程序的电脑上，你需要与 LEGO EV3 通信的权限。在我的情况中，我需要添加如下的 udev 规则（我的操作系统是 Unix）：
```
ATTRS{idVendor}=="0694",ATTRS{idProduct}=="0005",MODE="0666",GROUP=<group>
```

这通过它们的 vendor-id 0x0694 和 product-id 0x0005 标识 EV3 设备。模式 0666 允许属于 `<group>` 的所有用户具有读和写权限（把 `<group>` 修改为你所属的那个） 。EV3-USB 设备描述符显示了一个具有一个接口和两个端点的配置，0x01 用于给 EV3 发送数据，0x81 用于从 EV3 接收数据。数据以 1024 字节的包发送和接收。

#### Python
也许你需要安装 `pyusb`。对于我的系统来说这通过如下命令完成：
```
sudo pip3 install --pre pyusb
```
如果已经安装了 `pyusb`，你需要完成如下的步骤：

 * 把代码复制到名为 *EV3_do_nothing_usb.py* 的文件中。
 * 打开一个终端，并切换到你的程序的目录。
 * 键入 `python3 EV3_do_nothing_usb.py` 运行它。

```
#!/usr/bin/env python3

import usb.core
import struct

class EV3():
    def __init__(self, host: str):
        self._device = usb.core.find(idVendor=ID_VENDOR_LEGO, idProduct=ID_PRODUCT_EV3)
        if self._device is None:
            raise RuntimeError("No Lego EV3 found")
        serial_number = usb.util.get_string(self._device, self._device.iSerialNumber)
        if serial_number.upper() != host.replace(':', '').upper():
            raise ValueError('found ev3 but not ' + host)
        if self._device.is_kernel_driver_active(0) is True:
            self._device.detach_kernel_driver(0)
        self._device.set_configuration()
        self._device.read(EP_IN, 1024, 100)

    def __del__(self): pass

    def send_direct_cmd(self, ops: bytes, local_mem: int=0, global_mem: int=0) -> bytes:
        cmd = b''.join([
            struct.pack('<h', len(ops) + 5),
            struct.pack('<h', 42),
            DIRECT_COMMAND_REPLY,
            struct.pack('<h', local_mem*1024 + global_mem),
            ops
        ])
        self._device.write(EP_OUT, cmd, 100)
        print_hex('Sent', cmd)
        reply = self._device.read(EP_IN, 1024, 100)[0:5+global_mem]
        print_hex('Recv', reply)
        return reply

def print_hex(desc: str, data: bytes) -> None:
    print(desc + ' 0x|' + ':'.join('{:02X}'.format(byte) for byte in data) + '|')

ID_VENDOR_LEGO = 0x0694
ID_PRODUCT_EV3 = 0x0005
EP_IN  = 0x81
EP_OUT = 0x01
DIRECT_COMMAND_REPLY = b'\x00'
opNop = b'\x01'
my_ev3 = EV3('00:16:53:42:2B:99')
ops_nothing = opNop
my_ev3.send_direct_cmd(ops_nothing)
```

#### Java

我选择的与 USB 设备通信的是 usb4java。下载了 Java 包之后，你可以把它们添加到你的 classpath 中。在我的 Unix 机器是，通过如下命令完成：
```
export CLASSPATH=$CLASSPATH:./usb4java-1.2.0.jar:./libusb4java-1.2.0-linux-x86_64.jar
```

然后，执行下面的步骤：

 * 把下面的代码拷贝到名为 *EV3_do_nothing_usb.java* 的文件中。
 * 打开一个终端并切换到你的程序的目录。
 * 键入 `javac EV3_do_nothing_usb.java` 编译它。
 * 键入 `java EV3_do_nothing_usb` 运行它。

```
import org.usb4java.Device;
import org.usb4java.DeviceDescriptor;
import org.usb4java.DeviceHandle;
import org.usb4java.DeviceList;
import org.usb4java.LibUsb;
import org.usb4java.LibUsbException;

import java.nio.ByteBuffer;
import java.nio.IntBuffer;
import java.nio.ByteOrder;

public class EV3_do_nothing_usb {
    static final short ID_VENDOR_LEGO = (short) 0x0694;
    static final short ID_PRODUCT_EV3 = (short) 0x0005;
    static final byte  EP_IN          = (byte)  0x81;
    static final byte  EP_OUT         = (byte)  0x01;

    static final byte  opNop                        = (byte)  0x01;
    static final byte  DIRECT_COMMAND_REPLY         = (byte)  0x00;

    static DeviceHandle handle;

    public static void connectUsb () {
 int result = LibUsb.init(null);
 Device device = null;
 DeviceList list = new DeviceList();
 result = LibUsb.getDeviceList(null, list);
 if (result < 0){
     throw new RuntimeException("Unable to get device list. Result=" + result);
 }
 boolean found = false;
 for (Device dev: list) {
     DeviceDescriptor descriptor = new DeviceDescriptor();
     result = LibUsb.getDeviceDescriptor(dev, descriptor);
     if (result != LibUsb.SUCCESS) {
  throw new LibUsbException("Unable to read device descriptor", result);
     }
     if (  descriptor.idVendor()  == ID_VENDOR_LEGO
    || descriptor.idProduct() == ID_PRODUCT_EV3) {
  device = dev;
  found = true;
  break;
     }
 }
 LibUsb.freeDeviceList(list, true);
 if (! found) throw new RuntimeException("Lego EV3 device not found.");

 handle = new DeviceHandle();
 result = LibUsb.open(device, handle);
 if (result != LibUsb.SUCCESS) {
     throw new LibUsbException("Unable to open USB device", result);
 }
 boolean detach = LibUsb.kernelDriverActive(handle, 0) != 0;

 if (detach) result = LibUsb.detachKernelDriver(handle, 0);
 if (result != LibUsb.SUCCESS) {
     throw new LibUsbException("Unable to detach kernel driver", result);
 }

 result = LibUsb.claimInterface(handle, 0);
 if (result != LibUsb.SUCCESS) {
     throw new LibUsbException("Unable to claim interface", result);
 }
    }

    public static ByteBuffer sendDirectCmd (ByteBuffer operations,
         int local_mem, int global_mem) {
 ByteBuffer buffer = ByteBuffer.allocateDirect(operations.position() + 7);
 buffer.order(ByteOrder.LITTLE_ENDIAN);
 buffer.putShort((short) (operations.position() + 5));   // length
 buffer.putShort((short) 42);                            // counter
 buffer.put(DIRECT_COMMAND_REPLY);                       // type
 buffer.putShort((short) (local_mem*1024 + global_mem)); // header
 for (int i=0; i < operations.position(); i++) {         // operations
     buffer.put(operations.get(i));
 }

 IntBuffer transferred = IntBuffer.allocate(1);
 int result = LibUsb.bulkTransfer(handle, EP_OUT, buffer, transferred, 100); 
 if (result != LibUsb.SUCCESS) {
     throw new LibUsbException("Unable to write data", transferred.get(0));
 }
 printHex("Sent", buffer);

 buffer = ByteBuffer.allocateDirect(1024);
 transferred = IntBuffer.allocate(1);
 result = LibUsb.bulkTransfer(handle, EP_IN, buffer, transferred, 100);
 if (result != LibUsb.SUCCESS) {
     throw new LibUsbException("Unable to read data", result);
 }
 buffer.position(global_mem + 5);
 printHex("Recv", buffer);

 return buffer;
    }

    public static void printHex(String desc, ByteBuffer buffer) {
 System.out.print(desc + " 0x|");
 for (int i= 0; i < buffer.position() - 1; i++) {
     System.out.printf("%02X:", buffer.get(i));
 }
 System.out.printf("%02X|", buffer.get(buffer.position() - 1));
 System.out.println();
    }

    public static void main (String args[] ) {
 try {
     connectUsb();

     ByteBuffer operations = ByteBuffer.allocateDirect(1);
     operations.put(opNop);

     ByteBuffer reply = sendDirectCmd(operations, 0, 0);

     LibUsb.releaseInterface(handle, 0);
     LibUsb.close(handle);
 }
 catch (Exception e) {
     e.printStackTrace(System.err);
 }
    }
}
```

## 应答消息

如果你使用上面方案中的其中一个方案成功了，则会获得以下输出，即直接命令的回复消息：

```
----------------    
 \ len \ cnt \rs\   
  –---------------  
0x|03:00|2A:00|02|  
  –---------------  
   \ 3   \ 42  \ok\ 
    ----------------
```

前两个字节是众所周知的，它是回复的小尾端消息长度。在我们情况中，恢复消息是 3 字节长的。接下来的两个字节是消息计数器，也是广为人知的，你发送的消息的指纹，是 42。

最后一个字节是返回状态，其有 2 个可能值：

 * `DIRECT_REPLY` = `0x|02|`：直接命令操作成功
 * `DIRECT_REPLY_ERROR` = `0x|04|`：直接命令以失败结束

如果你真的收到了这条回复信息，那么你就入门了。恭喜！

## 头部的细节

上面我们跳过了头部细节的描述。提到了它，头部包含两个数，它们定义了内存的大小。

第一个数是局部内存的大小，其是你可以在其中保存中间信息的地址空间。第二个数描述了全局内存的大小，其是输出的地址空间。在 `DIRECT_COMMAND_REPLY` 的情形中，全局内存将作为回应的一部分发回。

局部内存具有最大 63 个字节，全局内存具有最大 1019 字节。那意味着，局部内存的大小需要 6 位，全局内存的大小需要 10 位。所有的组合在一起可以包含在两个字节中，如果一个字节共用的话。确实这样做了。如果以相反的顺序写入头字节，这是熟悉的大尾端，并且以二进制表示法写半字节组，则得到：`0b LLLL LLGG GGGG GGGG`。开头的 6 位是局部内存大小，其范围为 0-63。尾部的 10 位是全局内存的大小，其范围为 0-1020。小尾端下是：`0b GGGG GGGG LLLL LLGG`。比如如果你的全局内存具有 6 字节大小，你的局部内存需要 16 字节大小，则你的头部是 `0b 0000 0110 0100 0000` 或以十六进制表示是 `0x 06 40`。

这是描述性版本，现在是声明式方式的第二种方法。如果 `local_mem` 是本地内存的大小而 `global_mem` 是全局内存的大小，则计算：`header` = `local_mem` * 1024 + `global_mem`。把头部以小尾端 2 字节整数写入你将得到两个头部字节。如果你还有疑问，请等待接下来的课程，你将看到大量的头部并从例子中学习，这将有望解答你的疑问。

## 什么都不做的变体

在离开我们的第一个例子并关闭第一章之前，我们将测试两个头部的变体。第一个尝试是直接命令具有 6 字节的全局内存空间：

```
 \ len \ cnt \ty\ hd  \op\     
  -------------------------    
0x|06:00|2A:00|00|06:00|01|    
  -------------------------    
   \ 6   \ 42  \Re\ 0,6 \N \   
    \     \     \  \     \o \  
     \     \     \  \     \p \ 
      -------------------------
```

我们期望获得 6 字节低值输出的回复。 因此，你必须将答案的长度从 5 增加到 11。如果这样做，你将得到：
```
----------------------------------    
 \ len \ cnt \rs\ Output          \   
  –---------------------------------  
0x|09:00|2A:00|02|00:00:00:00:00:00|  
  –---------------------------------  
   \ 9   \ 42  \ok\                 \ 
    ----------------------------------
```

我们添加 16 字节的局部内存空间，并将直接命令更改为以下内容：
```
-------------------------      
 \ len \ cnt \ty\ hd  \op\     
  -------------------------    
0x|06:00|2A:00|00|06:40|01|    
  -------------------------    
   \ 6   \ 42  \Re\16,6 \N \   
    \     \     \  \     \o \  
     \     \     \  \     \p \ 
      -------------------------
```

我们希望得到与上述相同的回复，实际上：
```
----------------------------------    
 \ len \ cnt \rs\ Output          \   
  –---------------------------------  
0x|09:00|2A:00|02|00:00:00:00:00:00|  
  –---------------------------------  
   \ 9   \ 42  \ok\                 \ 
    ----------------------------------
```

## 你的家庭作业

在进行第 2 课之前，你应该完成如下的家庭作业：

 * 把一个小程序转换为你喜欢的编程语言并把它集成进你喜欢的开发环境。
 * 准备一些工具，因为一遍又一遍的从头开始可不是一件让人愉快的事情。我想到了以下设计：
    - EV3 是一个类。
    - BLUETOOTH，USB，WIFI，STD，ASYNC，SYNC 和 opNop 是公共常量。
    - 连接 EV3 是 EV3 对象初始化的一部分，即协议的选择通过以特定的参数调用 EV3 对象的构造函数完成。EV3 对象需要记住它的协议的类型。`socket` 和 `device` 是 EV3 对象的 private 或 protected 变量。
    - 发送数据给 EV3 通过 EV3 的方法 `send_direct_cmd` 完成。你可以把示例的函数当作蓝图，但在内部你一定要区分协议。
    - 为了从 EV3 接收数据，我们使用方法 `wait_for_reply`。你必须把函数 `send_direct_cmd` 的代码拆分为两个新的方法 `send_direct_cmd` 和 `wait_for_reply`。
    - 添加一个属性 `verbosity`，它规定了已发送的直接命令和收到的回复的打印。
    - 添加一个属性 `sync_mode`，它通过如下值规定了通信的行为：
        * `SYNC`：总是使用类型 `DIRECT_COMMAND_REPLY` 并等待响应。
        * `ASYNC`：从不等待响应，设置 `DIRECT_COMMAND_NO_REPLY`，当不使用全局内存时，其它情况设置 `DIRECT_COMMAND_REPLY`。
        * `STD`：像 `ASYNC` 那样设置 `DIRECT_COMMAND_NO_REPLY` 或 `DIRECT_COMMAND_REPLY`，但在 `DIRECT_COMMAND_REPLY` 的情况下等待响应。
    - `msg_cnt` 是 EV3 对象的私有变量，每次调用 `send_direct_cmd` 这个值都会增加。使用它来设置消息计数器。
    - 如例子中那样，消息长度，消息计数器，消息类型和头部都是在 `send_direct_cmd` 内部自动添加的。因此 `send_direct_cmd` 方法的参数 `ops` 真正地持有操作而没有其它东西。
 * 做一些性能测试，并比较三种通信协议（你将看到，USB 最快，蓝牙最慢，Wifi 居于中间，但你可能会赌三个协议之间的绝对值和因素）。
    - 以 `DIRECT_COMMAND_REPLY` 重复发送 `opNop` 并计算一个发送接收循环的平均时间。
    - 把连接的时间从发送和接收的时间中分离出来。你将只连接一次，但发送和接收循环的性能将限制你的应用程序。

## 结论

你开始编写一个类 EV3，用于使用直接命令与 LEGO EV3 通信。这个类允许自由地选择通信协议，并提供蓝牙，USB 和 Wifi。我选择的编程语言是 Python3。我使用 pydoc3 来展示我们的工程的实际状态。我希望，你可以简单地把它转为你喜欢的语言。此刻，我们的类 EV3 具有如下的 API：

```
Help on module ev3:

NAME
    ev3 - LEGO EV3 direct commands

CLASSES
    builtins.object
        EV3
    
    class EV3(builtins.object)
     |  object to communicate with a LEGO EV3 using direct commands
     |  
     |  Methods defined here:
     |  
     |  __del__(self)
     |      closes the connection to the LEGO EV3
     |  
     |  __init__(self, protocol:str, host:str)
     |      Establish a connection to a LEGO EV3 device
     |      
     |      Arguments:
     |      protocol: 'Bluetooth', 'Usb' or 'Wifi'
     |      host: mac-address of the LEGO EV3 (f.i. '00:16:53:42:2B:99')
     |  
     |  send_direct_cmd(self, ops:bytes, local_mem:int=0, global_mem:int=0) -> bytes
     |      Send a direct command to the LEGO EV3
     |      
     |      Arguments:
     |      ops: holds netto data only (operations), the following fields are added:
     |        length: 2 bytes, little endian
     |        counter: 2 bytes, little endian
     |        type: 1 byte, DIRECT_COMMAND_REPLY or DIRECT_COMMAND_NO_REPLY
     |        header: 2 bytes, holds sizes of local and global memory
     |      
     |      Keyword Arguments:
     |      local_mem: size of the local memory
     |      global_mem: size of the global memory
     |      
     |      Returns: 
     |        sync_mode is STD: reply (if global_mem > 0) or message counter
     |        sync_mode is ASYNC: message counter
     |        sync_mode is SYNC: reply of the LEGO EV3
     |  
     |  wait_for_reply(self, counter:bytes) -> bytes
     |      Ask the LEGO EV3 for a reply and wait until it is received
     |      
     |      Arguments:
     |      counter: is the message counter of the corresponding send_direct_cmd
     |      
     |      Returns:
     |      reply to the direct command
     |  
     |  ----------------------------------------------------------------------
     |  Data descriptors defined here:
     |  
     |  __dict__
     |      dictionary for instance variables (if defined)
     |  
     |  __weakref__
     |      list of weak references to the object (if defined)
     |  
     |  sync_mode
     |      sync mode (standard, asynchronous, synchronous)
     |      
     |      STD:   Use DIRECT_COMMAND_REPLY if global_mem > 0,
     |             wait for reply if there is one.
     |      ASYNC: Use DIRECT_COMMAND_REPLY if global_mem > 0,
     |             never wait for reply (it's the task of the calling program).
     |      SYNC:  Always use DIRECT_COMMAND_REPLY and wait for reply.
     |      
     |      The general idea is:
     |      ASYNC: Interruption or EV3 device queues direct commands,
     |             control directly comes back.
     |      SYNC:  EV3 device is blocked until direct command is finished,
     |             control comes back, when direct command is finished.               
     |      STD:   NO_REPLY like ASYNC with interruption or EV3 queuing,
     |             REPLY like SYNC, synchronicity of program and EV3 device.
     |  
     |  verbosity
     |      level of verbosity (prints on stdout).

DATA
    BLUETOOTH = 'Bluetooth'
    USB = 'Usb'
    WIFI = 'Wifi'
    STD = 'STD'
    ASYNC = 'ASYSNC'
    SYNC = 'SYNC'
    opNop = b'\x01'
```

我的类 EV3 是模块 `ev3` 的一部分，文件名是 `ev3.py`。我使用如下的程序来测试我的类 EV3：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = ev3.opNop

print("*** SYNC ***")
my_ev3.sync_mode = ev3.SYNC
my_ev3.send_direct_cmd(ops)

print("*** ASYNC (no reply) ***")
my_ev3.sync_mode = ev3.ASYNC
my_ev3.send_direct_cmd(ops)

print("*** ASYNC (reply) ***")
counter_1 = my_ev3.send_direct_cmd(ops, global_mem=1)
counter_2 = my_ev3.send_direct_cmd(ops, global_mem=1)
my_ev3.wait_for_reply(counter_1)
my_ev3.wait_for_reply(counter_2)

print("*** STD (no reply) ***")
my_ev3.sync_mode = ev3.STD
my_ev3.send_direct_cmd(ops)

print("*** STD (reply) ***")
my_ev3.send_direct_cmd(ops, global_mem=5)
my_ev3.send_direct_cmd(ops, global_mem=5)
```

得到输出：
```
*** SYNC ***
15:15:05.084275 Sent 0x|06:00|2A:00|00|00:00|01|
15:15:05.168023 Recv 0x|03:00|2A:00|02|
*** ASYNC (no reply) ***
15:15:05.168548 Sent 0x|06:00|2B:00|80|00:00|01|
*** ASYNC (reply) ***
15:15:05.168976 Sent 0x|06:00|2C:00|00|01:00|01|
15:15:05.169315 Sent 0x|06:00|2D:00|00|01:00|01|
15:15:05.212077 Recv 0x|04:00|2C:00|02|00|
15:15:05.212708 Recv 0x|04:00|2D:00|02|00|
*** STD (no reply) ***
15:15:05.213034 Sent 0x|06:00|2E:00|80|00:00|01|
*** STD (reply) ***
15:15:05.213411 Sent 0x|06:00|2F:00|00|05:00|01|
15:15:05.254032 Recv 0x|08:00|2F:00|02|00:00:00:00:00|
15:15:05.254633 Sent 0x|06:00|30:00|00|05:00|01|
15:15:05.313027 Recv 0x|08:00|30:00|02|00:00:00:00:00|
```

一些备注：
 * `sync_mode` = `SYNC` 设置 `type` = `DIRECT_COMMAND_REPLY` 并自动地等待响应，ok。
 * `sync_mode` = `ASYNC` 设置 `type` = `DIRECT_COMMAND_NO_REPLY`，不等待，ok。
 * 全局内存设置 `type` = `DIRECT_COMMAND_REPLY` 时 `sync_mode` = `ASYNC`，不等待。显式地调用 `wait_for_reply` 方法获得响应。
 * 请特别尊重此变体。我们发送两个直接命令，它们都被执行，且 EV3 设备保存响应。
 * 当我们稍后询问响应时，我们首先读取第一条命令的响应。它似乎就像 EV3 是一个 FIFO（先进先出）。 
 * 但它也可以并行执行，但它也可以按照完成执行的顺序执行并行和重复。我们将回到这一点。
 * 模式 `ASYNC` 需要一些规律。如果你忘记了读取响应，当你等待另一件事时，它会来。 我们使用消息计数器来揭示这种情况！
 * 请小心 USB 协议！协议USB请小心！ 如果像我一样直接发送异步命令，这可能会太快。
 * `sync_mode` = `STD`，没有全局内存设置 `type` = `DIRECT_COMMAND_NO_REPLY`，并且不等待回复，ok。
 * `sync_mode` = `STD`，全局内存设置 `type` = `DIRECT_COMMAND_REPLY`，每个直接命令等待响应，ok。
 * 消息计数器随直接命令递增，ok。
 * 头部正确保存全局内存的大小，ok。

如果你完成了作业，那么你已经为新的冒险做好了充分的准备。 我希望在下一课中再次见到你。

[原文](http://ev3directcommands.blogspot.com/2016/01/no-title-specified-page-table-border_94.html)
