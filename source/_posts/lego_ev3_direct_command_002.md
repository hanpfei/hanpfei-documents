---
title: EV3 直接命令 - 第 2 课 让你的 EV3 做点什么
date: 2018-10-30 20:05:49
categories: ROS
tags:
- ROS
- 翻译
---

# 介绍

上一课我们编写了类 EV3，它可以用于与 LEGO EV3 设备通信。我们通过什么也不做的 `opNop` 操作测试它。这一课是关于带有参数的真实指令的。这将使你的 EV3 设备成为你程序的活动部分。目前，我们不从我们的 EV3 接收数据。这个主题需要等稍后的一些课程。我们选取了如下这些种类的操作：

 * 设置 EV3 的名称
 * 播放声音和音调
 * 控制它的LED
 * 显示图像
 * 定时器
 * 启动程序
 * 模拟按钮动作
<!--more-->
请拿出 LEGO EV3 操作集的官方文档 *EV3 Firmware Developer Kit*，并阅读它。LEGO 的官方文档 *EV3 Communication Developer Kit* 还包含一些直接命令的例子。

如果你没有编写 `EV3` 类，但想要运行本课的程序，你可以自由地从 [ev3-python3](https://github.com/ChristophGaukel/ev3-python3) 下载模块 `ev3`。程序中仅有的需要你修改的地方是 MAC 地址。把 `00:16:53:42:2B:99` 替换为你的 EV3 设备的值。

# 设置 EV3 的名字

编程艺术的一个重要部分是选择好的名字。我们思考事物的方式强烈依赖于我们为它使用的名字。因此我们从设置名字开始。为了把你的 EV3 的名字修改为 *myEV3*，你需要发送如下的直接命令：

```
-------------------------------------------------                
 \ len \ cnt \ty\ hd  \op\cd\ Name               \               
  -------------------------------------------------              
0x|0E:00|2A:00|00|00:00|D4|08|84:6D:79:45:56:33:00|              
  -------------------------------------------------              
   \ 14  \ 42  \Re\ 0,0 \C \S \ "myEV3"            \             
    \     \     \  \     \o \E \                    \            
     \     \     \  \     \m \T \                    \           
      \     \     \  \     \_ \_ \                    \          
       \     \     \  \     \S \B \                    \         
        \     \     \  \     \e \R \                    \        
         \     \     \  \     \t \I \                    \       
          \     \     \  \     \  \C \                    \      
           \     \     \  \     \  \K \                    \     
            \     \     \  \     \  \N \                    \    
             \     \     \  \     \  \A \                    \   
              \     \     \  \     \  \M \                    \  
               \     \     \  \     \  \E \                    \ 
                -------------------------------------------------
```

响应是：
```
----------------    
 \ len \ cnt \rs\   
  ----------------  
0x|03:00|2A:00|02|  
  ----------------  
   \ 3   \ 42  \ok\ 
    ----------------
```

这说明，直接命令被成功执行了。你可以通过查看 brick 显示器来检查操作的结果，在它的第一行应该显示新名称。此外，如果某些蓝牙设备搜索设备并找到了你的 EV3，它将以新的名字显示。

几点备注：

 * 我们使用了一个新操作，它执行一些设置：`opCom_Set` = `0x|D4|`
 * 由于操作 `opCom_Set` 用于执行不同的设置，因而它的后面总是跟一个指定操作内容的 CMD。CMD 告诉我们，具体是哪一个。你可以把它们想成是一个两字节操作。但在 EV3 的术语中，它是一个操作（即 指令）和它的 CMD。
 * `opCom_Set` 的 CMD `SET_BRICKNAME` = `0x|08|` 需要一个参数：`NAME`。在 LEGO 的操作描述中，你可以读到：*(DATA8) NAME – First character in character string*。但事实上，我们发送 `0x|84:6D:79:45:56:33:00|` 作为参数 `NAME` 的值。这需要一些解释：
    - `0x|6D:79:45:56:33|` 是字符串 *myEV3* 的 ASCII 码，其中 `0x|6D|` = `“m”`，`0x|79|` = `“y”` 等等。
    - `0x|00|` 终止字符串（这被称为零终止字符串）。
    - `0x|84|` 是 LCS 字符串的前导标识字节（以二进制表示是：`0b 1000 0100`）。

结论是，你发送给你的 EV3 作为操作的常量参数的每个字符串，必须包含前导 `0x|84|` 和后缀 `0x|00|`。在我的情况中，连接的结果是` LCS("myEV3")` = `0x|84:6D:79:45:56:33:00|`，作为参数 `NAME` 的值。

请给你的类 EV3 添加一个静态类方法（在 Python 中，是一个模块级的函数）：

 * `LCS(value: str)` 以 LCS 格式返回一个表示字符串值的字节数组。

然后添加两个常量 `opCom_Set` = `0x|D4|` 和 `SET_BRICKNAME` = `0x|08|`。

可以编写一个小程序来修改名称。我通过如下的代码来完成：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1
ops = b''.join([
    ev3.opCom_Set,
    ev3.SET_BRICKNAME,
    ev3.LCS("myEV3")
])
my_ev3.send_direct_cmd(ops)
```

它的输出是：
```
09:12:31.011558 Sent 0x|0E:00|2A:00|80|00:00|D4:08:84:6D:79:45:56:33:00|
```

请看一下你 EV3 的显示器，它的名字改变了。

# 常量整数参数

字符串是一种参数类型，但还有其它的。共同的是，参数的类型由前导字节，*标识字节* 标识。在这一课，我们集中于常量参数和局部变量。术语 *常量参数* 并不是很精确，但它意味着参数具有如下两个特性：

 * 它们是操作的参数。
 * 它们总是保存值而永远没有地址。

EV3 直接命令支持如下格式的常量参数：

 * 字符串（你已经学习了它们中的一个）
 * 整数值

字符串的长度可变，整数是有符号的，且可以包含 5 位，8 位，16 位和 32 位。也许你没有看到浮点数，但是没有操作需要以浮点数作为参数。事实上，只有 5 种常量参数。

你应该特别专注于第一个字节，即标识字节，它定义了变量的类型和长度。标识字节的位 0 ***（最高有效位）*** 代表长或短格式：

 * `0b 0... ....` 短格式（只有一个字节，标识字节包含值）
 * `0b 1... ....` 长格式（标识字节不包含任何值的位）

位 5 （在长格式的情况下）代表长度类型

 * `0b .... .0..` 意味着固定长度。
 * `0b .... .1..` 意味着以零结束的字符串。

位 6 和 7 （仅长格式）代表后续的整数的长度

 * `0b .... ..00` 意味着可变长度，
 * `0b .... ..01` 意味着后面有一个字节，
 * `0b .... ..10` 是说，后面有两个字节，
 * `0b .... ..11` 是说，后面有四个字节。

现在我们写 5 个常量作为二进制掩码，其中 S 代表符号（0 是正的，1 是负的），V 代表值的一位。

 * `LC0`：`0b 00SV VVVV`，5 位整数值，范围：-32 - 31，长度：1 字节，由 2 个前导位 00 标识。***整数值其实是 6 位的。***
 * `LC1`：`0b 1000 0001 SVVV VVVV`，8 位整数值，范围：-127 - 127，长度：2 字节，由前导字节 `0x|81|` 标识。值 `0x|80|` 是 `NaN`。
 * `LC2`：`0b 1000 0010 VVVV VVVV SVVV VVVV`，16 位整数值，范围：-32,767 – 32,767，长度：3 字节，由前导字节 ` 0x|82|` 标识。值 `0x|80:00|` 是 `NaN`。
 * `LC4`：`0b 1000 0011 VVVV VVVV VVVV VVVV VVVV VVVV SVVV VVVV`，32 位整数值，范围：-2,147,483,647 – 2,147,483,647，长度：5 字节，由前导字节 `0x|83|` 标识。值 `0x|80:00:00:00|` 是 `NaN`。
 * `LCS`：`0b 1000 0100 VVVV VVVV ... 0000 0000`，以零结尾的字符串，长度：可变，由前导字节 `0x|84|` 标识。

LC2 和 LC4 的字节序列是小尾端的。这意味着，正如你从第 1 课学到的那样，标识字节是头部，后面的字节与你习惯的顺序相反。如果操作以整数常量作为参数，则可以在 LC0，LC1，LC2 或 LC4 之间进行选择。对于小值（范围在 -32 到 31 之间），用 LC0，对于非常大的值，用 LC4。直接命令从左到右读取。当解释一个参数的第一个字节时，哪个附加字节属于它以及在哪里找到该值是清楚的。总是使用最短的可能变体以消除通信流量，并因此加速直接命令的操作，但这种影响很小。更多关于参数的标识字节的细节可以在 LEGO 的 *EV3 Firmware Developer Kit* 的 3.4 节找到。

请给你的 `EV3` 类添加另外的静态类方法（或模块方法）：

 * LCX(value: int) 以格式 LC0，LC1，LC2 或 LC4 返回一个依赖于值的范围的字节数组。

# 播放声音文件

我们想要我们的 EV3 brick 播放声音文件 `/home/root/lms2012/sys/ui/DownloadSucces.rsf`，这通过如下操作完成：

 * `opSound` = `0x|94|` 的 `CMD PLAY` = 0x|02|，且参数为：
    - VOLUME：百分比 [0 - 100]
    - NAME：声音文件的绝对路径，或相对于 `/home/root/lms2012/sys/ ` (不包含扩展名 ".rsf")

程序：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opSound,
    ev3.PLAY,
    ev3.LCX(100),                  # VOLUME
    ev3.LCS('./ui/DownloadSucces') # NAME
])
my_ev3.send_direct_cmd(ops)
```

输出：
```
09:42:03.575103 Sent 0x|1E:00|2A:00|80|00:00|94:02:81:64:84:2E:2F:75:69:2F:44:6F:77:6E:6C:6F:61:64:53:75:63:63:65:73:00|
```

EV3 brick 的文件系统不是本节课的主题。更多信息请参考 [Folder Structure](http://ev3.fantastic.computer/doxygen/UIdesign.html)。

## 重复播放声音文件

操作 `opSound` 具有一个 CMD `REPEAT`，它以无限循环播放声音文件，这可以由操作 `opSound` 的 CMD `BREAK` 中断。有两个额外的操作：

 * `opSound` = `0x|94|` 的 CMD `REPEAT` = `0x|03|`，且具有参数：
    - VOLUME：百分比 [0 - 100]
    - NAME：声音文件的绝对路径，或相对于 `/home/root/lms2012/sys/ ` 的相对路径 (不包含扩展名 ".rsf")
 * `opSound` = `0x|94|` 的 CMD `BREAK` = `0x|00|`，没有参数。

我们用如下的程序来测试它：
```
#!/usr/bin/env python3

import ev3, time

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opSound,
    ev3.REPEAT,
    ev3.LCX(100),                  # VOLUME
    ev3.LCS('./ui/DownloadSucces') # NAME
])
my_ev3.send_direct_cmd(ops)
time.sleep(5)
ops = b''.join([
    ev3.opSound,
    ev3.BREAK
])
my_ev3.send_direct_cmd(ops)
```

它播放声音文件 5 秒，然后停止播放。输出为：
```
09:55:28.814320 Sent 0x|1E:00|2A:00|80|00:00|94:03:81:64:84:2E:2F:75:69:2F:44:6F:77:6E:6C:6F:61:64:53:75:63:63:65:73:00|
09:55:33.822352 Sent 0x|07:00|2B:00|80|00:00|94:00|
```

## 播放音调

我们想要我们的 EV3 brick 播放音调，这通过如下操作完成：

 * `opSound` = `0x|94|` 的 `CMD TONE` = 0x|01|，且参数为：
    - VOLUME：百分比 [0 - 100]
    - FREQUENCY：单位为 Hz，[250 - 10000]
    - DURATION：单位为毫秒（0 表示无限制）

播放一个 a' 一秒的直接命令：
```
-------------------------------------------------        
 \ len \ cnt \ty\ hd  \op\cd\vo\ fr     \ du     \       
  -------------------------------------------------      
0x|0E:00|2A:00|80|00:00|94|01|01|82:B8:01|82:E8:03|      
  -------------------------------------------------      
   \ 14  \ 42  \no\ 0,0 \S \T \1 \ 440    \ 1000   \      
    \     \     \  \     \o \O \  \        \        \     
     \     \     \  \     \u \N \  \        \        \    
      \     \     \  \     \n \E \  \        \        \   
       \     \     \  \     \d \  \  \        \        \  
        -------------------------------------------------
```

程序发送它：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
ops = b''.join([
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),    # VOLUME
    ev3.LCX(440),  # FREQUENCY
    ev3.LCX(1000), # DURATION
])
my_ev3.send_direct_cmd(ops)
```

尽管我们很自豪，但我们希望我们的 EV3 以 c' 播放三和弦：

 * c' (262 Hz)
 * e' (330 Hz)
 * g' (392 Hz)
 * c'' (523 Hz)

我们把我们的程序修改为：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
ops = b''.join([
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(262),
    ev3.LCX(500),
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(330),
    ev3.LCX(500),
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(392),
    ev3.LCX(500),
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(2),
    ev3.LCX(523),
    ev3.LCX(1000)
])
my_ev3.send_direct_cmd(ops)
```

但我们只听到了一个音调，即最后的那个 (c'')。为什么？

这是因为操作彼此中断。你必须将操作视为不耐烦且表现不佳的角色。中断是他们的标准。如果想要避免中断，则必须明确告诉它。在播放声音的场景中，可以通过如下操作完成：

 * `opSound_Ready` = `0x|96|`

它将一直等到声音结束。我们再次修改程序：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
ops = b''.join([
        opSound,
        cmdSound_PlayTone,
        LCX(1),
        LCX(262),
        LCX(500),
        opSound_Ready,

        opSound,
        cmdSound_PlayTone,
        LCX(1),
        LCX(330),
        LCX(500),
        opSound_Ready,

        opSound,
        cmdSound_PlayTone,
        LCX(1),
        LCX(392),
        LCX(500),
        opSound_Ready,

        opSound,
        cmdSound_PlayTone,
        LCX(2),
        LCX(523),
        LCX(1000),
])
my_ev3.send_direct_cmd(ops)
```

现在我们听到了我们期望的！

# 修改 LEDs 的颜色

我们的 EV3 永远不会达到真正的自动点唱机的质量，但为什么不添加一些灯光效果？这需要一个新操作：

 * `opUI_Write` = `0x|82|` 的 `CMD LED` = `0x|1B|`，且参数为：
    - `PATTERN`：`GREEN` = `0x|01|`，`RED` = `0x|02|`，等等。

LED Patterns 可以取的值如下：
 * 0x00 : Led off
 * 0x01 : Led green
 * 0x02 : Led red
 * 0x03 : Led orange
 * 0x04 : Led green flashing
 * 0x05 : Led red flashing
 * 0x06 : Led orange flashing
 * 0x07 : Led green pulse
 * 0x08 : Led red pulse
 * 0x09 : Led orange pulse

我们再次给我们的程序添加一些代码：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1
ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(262),
    ev3.LCX(500),
    ev3.opSound_Ready,
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_GREEN,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(330),
    ev3.LCX(500),
    ev3.opSound_Ready,
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(392),
    ev3.LCX(500),
    ev3.opSound_Ready,
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED_FLASH,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(2),
    ev3.LCX(523),
    ev3.LCX(2000),
    ev3.opSound_Ready,
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_GREEN
])
my_ev3.send_direct_cmd(ops)
```

我们发送的是一个 60 字节长的直接命令：
```
11:39:49.039902 Sent 0x|3C:00|2A:00|80|00:00|82:1B:02:94:01:01:82:06:01:82:F4:01:96:82:1B:01:94:01:01:82:4A:01:82:F4:01:96:82:1B:02:...
```

这小于它最大长度的 6%。

# 显示图像

EV3 的显示屏是单色的，分辨率为 180 x 128 像素。这听起来有点过时，但允许显示图标和表情符号或绘制图片。操作 `opUI_Draw` 具有大量不同的操作显示器的 CMD。这里我们使用其中四个：

 * `opUI_Draw` = `0x|84|` 的 CMD `UPDATE` = `0x|00|`，没有参数。
 * `opUI_Draw` = `0x|84|` 的 `CMD TOPLINE` = `0x|12|`，具有参数：
    - (Data8) `ENABLE`：启用或禁用顶部状态行， [0：禁用，1：启用]
 * `opUI_Draw` = `0x|84|` 的 CMD `FILLWINDOW` = 0x|13|，具有参数：
    - (Data8) COLOR：指定黑色或白色，[0：白色，1：黑色]
    - (Data16) Y0：指定 Y 起始点，[0 - 127]
    - (Data16) Y1：指定 Y 大小
 * `opUI_Draw` = `0x|84|` 的 CMD `BMPFILE` = `0x|1C|`，具有参数：
    - (Data8) COLOR：指定黑色或白色，[0：白色，1：黑色]
    - (Data16) X0：指定 X 起始点，[0 - 127]
    - (Data16) Y0：指定 Y 起始点，[0 - 127]
    - NAME：图像文件的绝对路径，或相对于 `/home/root/lms2012/sys/`（具有扩展名 ".rgf"）。该命令的名称具有误导性。该文件的扩展名必须是 `.rgf`（代表 *机器人图形格式(robot graphic format)*）而不是 bmp 图形的文件。

我们运行这个程序：
```
#!/usr/bin/env python3

import ev3, time

my_ev3 = ev3.EV3(
    protocol=ev3.USB,
    host='00:16:53:42:2B:99'
)
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opUI_Draw,
    ev3.TOPLINE,
    ev3.LCX(0),                                      # ENABLE
    ev3.opUI_Draw,
    ev3.BMPFILE,
    ev3.LCX(1),                                      # COLOR
    ev3.LCX(0),                                      # X0
    ev3.LCX(0),                                      # Y0
    ev3.LCS("../apps/Motor Control/MotorCtlAD.rgf"), # NAME
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
time.sleep(5)
ops = b''.join([
    ev3.opUI_Draw,
    ev3.TOPLINE,
    ev3.LCX(1),     # ENABLE
    ev3.opUI_Draw,
    ev3.FILLWINDOW,
    ev3.LCX(0),     # COLOR
    ev3.LCX(0),     # Y0
    ev3.LCX(0),     # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
```

输出为：
```
12:01:00.253855 Sent 0x|35:00|2A:00|80|00:00|84:12:00:84:1C:01:00:00:84:2E:2E:2F:61:70:70:...
12:01:05.265584 Sent 0x|0F:00|2B:00|80|00:00|84:12:01:84:13:00:00:00:84:00|
```

显示屏显示图像 `MotorCtlAD.rgf` 五秒钟，然后显示屏变为空，除了顶线。 一些注释：

 * 绘制需要画布。这是显示的实际图像。我们添加一些元素，然后调用 `UPDATE` 使画布可见。如果你更喜欢以空白画布开始，则必须明确清除画布的内容。
 * CMD `TOPLINE` 允许打开或关闭顶线。
 * CMD `FILLWINDOW` 允许填充或擦除窗口的一部分。如果参数 `Y0` 和 `Y1` 都为零，则表示整个显示屏。
 * 将 CMD `BMPFILE` 的参数 COLOR 设置为值 0 会反转图像的颜色。
 * 操作 `opUI_Draw` 允许存储和恢复图像（CMD `STORE` 和 `RESTORE`）。  但是当实际的直接命令执行结束时，存储的图像将丢失。

欢迎你测试更多操作 `opUI_Draw` 的 CMD。

# 局部内存

在第 1 课我们读到，本地内存是保存中间信息的地址空间。现在我们学习如何使用它，我们再次讨论标识字节，它定义变量的类型和长度。我们将编写另一个函数 LVX，它返回本地内存的地址。如你所知，标识字节的第 0 位代表短格式或长格式：

 * `0b 0... ....` 短格式（只有一个字节，标识字节包含值）
 * `0b 1... ....` 长格式（标识字节不包含任何值的位）

如果位 1 和 2 是 `0b .10. ....,`，它们代表局部变量，它们是局部内存的地址。

位 6 和 7 代表后续的值的长度

 * `0b .... ..00` 意味着可变长度，
 * `0b .... ..01` 意味着后面有一个字节，
 * `0b .... ..10` 是说，后面有两个字节，
 * `0b .... ..11` 是说，后面有四个字节。

这允许将 4 个局部变量写为二进制掩码，我们不需要符号，因为地址总是正数。 V 代表地址（值）的一位。

 * `LV0`： `0b 010V VVVV`，5 位地址，范围：0 - 31，长度：1 字节，由 3 个前导位 010 标识。
 * `LV1`：` 0b 1100 0001 VVVV VVVV`，8 位地址，范围：0 - 255，长度：2 字节，由前导字节 `0x|C1|` 标识。
 * `LV2`：`0b 1100 0010 VVVV VVVV VVVV VVVV`，16 位地址，范围：0 – 65, 535，长度：3 字节，由前导字节 ` 0x|C2|` 标识。
 * `LV4`：`0b 1100 0011 VVVV VVVV VVVV VVVV VVVV VVVV VVVV VVVV`，32 位地址，范围：0 – 4,294,967,296，长度：5 字节，由前导字节 `0x|C3|` 标识。

一些说明：

 * 在直接命令中，不需要 LV2 和 LV4！你记得局部内存最多有 63 个字节。
 * 必须正确放置局部内存的地址。 如果将 4 字节值写入局部内存，则其地址必须为0，4，8，......（4的倍数）。对于 2 字节值也是一样，它们的地址必须是 2 的倍数。
 * 你需要将局部内存拆分为所需长度的段，然后使用每个段的第一个字节的地址。
 * 头字节包含局部内存的总长度（有关详细内容，请参阅第 1 课）。 不要忘记正确发送头字节！

### 一个新的模块函数：LVX

请将函数 `LVX(value)` 添加到模块 `ev3` 中，它取决于实际值，返回 LV0，LV1，LV2 或 LV4 类型中最短的。 我已经完成了，现在我的 ev3 模块的文档如下：
```
FUNCTIONS
    LCS(value:str) -> bytes
        pack a string into a LCS
    
    LCX(value:int) -> bytes
        create a LC0, LC1, LC2, LC4, dependent from the value
    
    LVX(value:int) -> bytes
        create a LV0, LV1, LV2, LV4, dependent from the value
```

# 计时器

控制时间是实时程序的一个重要方面。我们已经看到如何等待音调结束，我们在本地程序中等待，直到我们停止重复播放的声音文件。EV3 的操作集包含计时器操作，它们允许在直接命令的执行中等待。我们使用以下两个操作：

 * `opTimer_Wait` = `0x|85|`，具有参数：
    - (Data32) `TIME`：等待的时间（单位为毫秒）
    - (Data32) TIMER：用于计时的变量
这个操作向局部或全局内存中写入 4 个字节的时间戳

 * `opTimer_Ready` = `0x|86|`，具有参数：
    - (Data32) TIMER：用于计时的变量
这个操作读取时间戳并等待直到实际时间到达这个时间戳的值。

我们用一个绘制三角形的程序测试计时器操作。这需要操作 `opUI_Draw` 的另一个 CMD：

 * `opUI_Draw` = `0x|84|` 的 CMD `LINE` = 0x|03|，具有参数：
    - (Data8) COLOR：指定黑色或白色，[0：白色，1：黑色]
    - (Data16) X0：指定 X 起始点，[0 - 177]
    - (Data16) Y0：指定 Y 起始点，[0 - 127]
    - (Data16) X1：指定 X 结束点
    - (Data16) Y1：指定 Y 结束点

程序：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
ops = b''.join([
    ev3.opUI_Draw,
    ev3.TOPLINE,
    ev3.LCX(0),     # ENABLE
    ev3.opUI_Draw,
    ev3.FILLWINDOW,
    ev3.LCX(0),     # COLOR
    ev3.LCX(0),     # Y0
    ev3.LCX(0),     # Y1
    ev3.opUI_Draw,
    ev3.UPDATE,
    ev3.opTimer_Wait,
    ev3.LCX(1000),
    ev3.LVX(0),
    ev3.opTimer_Ready,
    ev3.LVX(0),
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(2),     # X0
    ev3.LCX(125),   # Y0
    ev3.LCX(88),    # X1
    ev3.LCX(2),     # Y1
    ev3.opUI_Draw,
    ev3.UPDATE,
    ev3.opTimer_Wait,
    ev3.LCX(500),
    ev3.LVX(0),
    ev3.opTimer_Ready,
    ev3.LVX(0),
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(88),    # X0
    ev3.LCX(2),     # Y0
    ev3.LCX(175),   # X1
    ev3.LCX(125),   # Y1
    ev3.opUI_Draw,
    ev3.UPDATE,
    ev3.opTimer_Wait,
    ev3.LCX(500),
    ev3.LVX(0),
    ev3.opTimer_Ready,
    ev3.LVX(0),
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(175),   # X0
    ev3.LCX(125),   # Y0
    ev3.LCX(2),     # X1
    ev3.LCX(125),   # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops, local_mem=4)
```

这个程序清除显示屏，然后等待一秒，绘制一条线，等待半秒，绘制第二条线，等待并最终绘制第三条线。它需要 4 个字节的本地内存，可以多次写入和读出。

显然，计时可以在本地程序或直接命令中完成。 我们修改程序：
```
#!/usr/bin/env python3

import ev3, time

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
ops = b''.join([
    ev3.opUI_Draw,
    ev3.TOPLINE,
    ev3.LCX(0),     # ENABLE
    ev3.opUI_Draw,
    ev3.FILLWINDOW,
    ev3.LCX(0),     # COLOR
    ev3.LCX(0),     # Y0
    ev3.LCX(0),     # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
time.sleep(1)
ops = b''.join([
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(2),     # X0
    ev3.LCX(125),   # Y0
    ev3.LCX(88),    # X1
    ev3.LCX(2),     # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
time.sleep(0.5)
ops = b''.join([
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(88),    # X0
    ev3.LCX(2),     # Y0
    ev3.LCX(175),   # X1
    ev3.LCX(125),   # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
time.sleep(0.5)
ops = b''.join([
    ev3.opUI_Draw,
    ev3.LINE,
    ev3.LCX(1),     # COLOR
    ev3.LCX(175),   # X0
    ev3.LCX(125),   # Y0
    ev3.LCX(2),     # X1
    ev3.LCX(125),   # Y1
    ev3.opUI_Draw,
    ev3.UPDATE
])
my_ev3.send_direct_cmd(ops)
```

两种方案下，显示器具有相同的行为，但又有些不同。 第一个版本所需的通信量较少，但它会阻塞 EV3，直到直接命令执行结束。第二个版本需要四个直接命令，但允许在绘图休眠时发送其它直接命令。

# 启动程序

直接命令可以启动程序。通常你通过按下 EV3 设备的按钮完成。程序是一个扩展名为 ".rbf" 的文件，它存放在 EV3 的文件系统上。我们将启动程序 `/home/root/lms2012/apps/Motor Control/Motor Control.rbf`。这需要两个新操作：

 * `opFile` = `0x|C0|` 的 CMD `LOAD_IMAGE` = 0x|08|，具有参数：
    - (Data16) PRGID：程序运行的 Slot。值 `0x|01|` 用于执行用户工程，apps 和工具。
    - (Data8) NAME：可执行文件的完整路径，或相对于 `/home/root/lms2012/sys/`（具有扩展名 ".rbf"）的路径
  返回：
    - (Data32) SIZE：镜像的字节大小
    - (Data32) *IP：镜像的地址
这个操作是 [加载器](https://en.wikipedia.org/wiki/Loader_(computing))。它把程序加载进内存中，并准备执行。

 * `opProgram_Start` = `0x|C0|`，具有参数：
    - (Data16) PRGID：程序运行的 Slot。
    - (Data32) SIZE：镜像的字节大小
    - (Data32) *IP：镜像的地址
    - (Data8) DEBUG：调试模式，值 0 代表普通模式

程序：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.USB, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opFile,
    ev3.LOAD_IMAGE,
    ev3.LCX(1),                                         # SLOT
    ev3.LCS('../apps/Motor Control/Motor Control.rbf'), # NAME
    ev3.LVX(0),                                         # SIZE
    ev3.LVX(4),                                         # IP*
    ev3.opProgram_Start,
    ev3.LCX(1),                                         # SLOT
    ev3.LVX(0),                                         # SIZE
    ev3.LVX(4),                                         # IP*
    ev3.LCX(0)                                          # DEBUG
])
my_ev3.send_direct_cmd(ops, local_mem=8)
```

第一个操作的返回值是 `SIZE` 和 `IP*`。我们把它们写入局部内存的地址 0 和 4。第二个操作从局部内存读取它的参数 `SIZE` 和 `IP*`。它的参数 `SLOT` 和 `DEBUG` 是给定的常量值。程序的输出是：
```
12:50:45.332826 Sent 0x|38:00|2A:00|80|00:20|C0:08:01:84:2E:2E:2F:61:70:70:73:2F:4D:6F:74:6F:...
```

它真的启动了程序 `/home/root/lms2012/apps/Motor Control/Motor Control.rbf`。

# 模拟按钮按下

在这个例子中，我们通过模拟如下的按钮按下事件关闭 EV3 brick：

 * `BACK_BUTTON` = `0x|06|`
 * `RIGHT_BUTTON` = `0x|04|`
 * `ENTER_BUTTON` = `0x|02|`

我们需要等待直到初始化操作完成。这可以通过操作 `opUI_Button` 的 CMD `WAIT_FOR_PRESS` 完成，这再次预防了中断。使用下面的新操作：

 * `opUI_Button` = `0x|83|` 的 CMD `PRESS` = 0x|05|，具有参数：
    - BUTTON：Up Button = 0x|01|，Enter Button = 0x|02|，等等。
 * `opUI_Button` = `0x|83|` 的 CMD `WAIT_FOR_PRESS` = 0x|03|

直接命令具有如下的结构：
```
-------------------------------------------------------------                 
 \ len \ cnt \ty\ hd  \op\cd\bu\op\cd\op\cd\bu\op\cd\op\cd\bu\                
  -------------------------------------------------------------               
0x|12:00|2A:00|80|00:00|83|05|06|83|03|83|05|04|83|03|83|05|02|               
  -------------------------------------------------------------               
   \ 18  \ 42  \no\ 0,0 \U \P \B \U \W \U \P \R \U \W \U \P \E \              
    \     \     \  \     \I \R \A \I \A \I \R \I \I \A \I \R \N \             
     \     \     \  \     \_ \E \C \_ \I \_ \E \G \_ \I \_ \E \T \            
      \     \     \  \     \B \S \K \B \T \B \S \H \B \T \B \S \E \           
       \     \     \  \     \U \S \_ \U \_ \U \S \T \U \_ \U \S \R \          
        \     \     \  \     \T \  \B \T \F \T \  \_ \T \F \T \  \_ \         
         \     \     \  \     \T \  \U \T \O \T \  \B \T \O \T \  \B \        
          \     \     \  \     \O \  \T \O \R \O \  \U \O \R \O \  \U \       
           \     \     \  \     \N \  \T \N \_ \N \  \T \N \_ \N \  \T \      
            \     \     \  \     \  \  \O \  \P \  \  \T \  \P \  \  \T \     
             \     \     \  \     \  \  \N \  \R \  \  \O \  \R \  \  \O \    
              \     \     \  \     \  \  \  \  \E \  \  \N \  \E \  \  \N \   
               \     \     \  \     \  \  \  \  \S \  \  \  \  \S \  \  \  \  
                \     \     \  \     \  \  \  \  \S \  \  \  \  \S \  \  \  \ 
                 -------------------------------------------------------------
```

我的对应的程序是：
```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
ops = b''.join([
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.BACK_BUTTON,
    ev3.opUI_Button,
    ev3.WAIT_FOR_PRESS,
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.RIGHT_BUTTON,
    ev3.opUI_Button,
    ev3.WAIT_FOR_PRESS,
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.ENTER_BUTTON
])
my_ev3.send_direct_cmd(ops)
```

这真的关闭了EV3设备！

没有必要回复，但我是一个好奇的人。我的问题是：EV3会在它关闭之前回复还是不回复？

```
#!/usr/bin/env python3

import ev3

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1
my_ev3.sync_mode = ev3.SYNC
ops = b''.join([
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.BACK_BUTTON,
    ev3.opUI_Button,
    ev3.WAIT_FOR_PRESS,
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.RIGHT_BUTTON,
    ev3.opUI_Button,
    ev3.WAIT_FOR_PRESS,
    ev3.opUI_Button,
    ev3.PRESS,
    ev3.ENTER_BUTTON
])
my_ev3.send_direct_cmd(ops)
```

在我按下另一个按钮之前没有任何反应，然后它回复并关闭。这并不令人惊讶，这是不一致的。 关机和回复不合适一起，EV3 设备无法完成命令然后发送回复！

# 我们学到了什么

 * 直接命令由操作序列组成。当我们给 brick 发送直接命令时，一个操作接一个操作的执行。但它们彼此互相打断，如果想要它们等待的话需要特殊的操作。
 * 大多数操作需要参数，它们可以以格式 LC0，LC1，LC2 和 LC4 发送，这些都包含有符号整数，但具有不同的范围。另一种格式是 LCS，用于字符串。它以`0x|84|` 开头，然后是零终止的 ASCII 码串。
 * 局部变量（LV0，LV1，LV2 和 LV4）允许寻址保存中间数据的局部存储器。
 * 一些操作具有许多 CMD，它们使用不同的参数集定义不同的任务。
 * 我们已经看过很多操作并知道他们的参数的含义，但这只是 EV3 操作集的一小部分。

# 结论

我们关于直接命令的知识增长了，我们的类 EV3 也是。添加我们需要的所有常量需要一些耐心。随着操作数量的增加，直接命令的参考文档 *EV3 Firmware Developer Kit* 需要更加仔细地阅读。

这是我的函数和数据的实际状态：
```
Help on module ev3:

NAME
    ev3 - LEGO EV3 direct commands

CLASSES
    builtins.object
        EV3
    
    class EV3(builtins.object)
     ...

FUNCTIONS
    LCS(value:str) -> bytes
        pack a string into a LCS
    
    LCX(value:int) -> bytes
        create a LC0, LC1, LC2, LC4, dependent from the value
    
    LVX(value:int) -> bytes
        create a LV0, LV1, LV2, LV4, dependent from the value

DATA
    ASYNC = 'ASYNC'
    BACK_BUTTON = b'\x06'
    BLUETOOTH = 'Bluetooth'
    BMPFILE = b'\x1c'
    BREAK = b'\x00'
    ENTER_BUTTON = b'\x02'
    FILLWINDOW = b'\x13'
    LED = b'\x1b'
    LED_OFF = b'\x00'
    LED_GREEN = b'\x01'
    LED_GREEN_FLASH = b'\x04'
    LED_GREEN_PULSE = b'\x07'
    LED_ORANGE = b'\x03'
    LED_ORANGE_FLASH = b'\x06'
    LED_ORANGE_PULSE = b'\t'
    LED_RED = b'\x02'
    LED_RED_FLASH = b'\x05'
    LED_RED_PULSE = b'\x08'
    LINE = b'\x03'
    LOAD_IMAGE = b'\x08'
    PLAY = b'\x02'
    PRESS = b'\x53'
    REPEAT = b'\x02'
    RIGHT_BUTTON = b'\x04'
    SET_BRICKNAME = b'\x08'
    STD = 'STD'
    SYNC = 'SYNC'
    TONE = b'\x01'
    TOPLINE = b'\x12'
    USB = 'Usb'
    UPDATE = b'\x00'
    WAIT_FOR_PRESS = b'\x03'
    WIFI = 'Wifi'
    opCom_Set = b'\xd4'
    opFile = b'\xc0'
    opNop = b'\x01'
    opProgram_Start = b'\x03'
    opSound = b'\x94'
    opSound_Ready = b'\x96'
    opTimer_Wait = b'\x85'
    opTimer_Ready = b'\x86'
    opUI_Button = b'\x83'
    opUI_Draw = b'\x84'
    opUI_Write = b'\x82'
```

真正的机器人从其传感器读取数据并通过其电机进行运动。目前我们的 EV3 设备都没有。我也知道，存在更酷的声音或光效的电子设备。现在，你可以测试在 *EV3 Firmware Developer Kit* 中找到的其他一些操作了。

保持联系，下一课将是关于电机的。我希望，我们将更接近你真正感兴趣的话题。

[原文](http://ev3directcommands.blogspot.com/2016/01/ev3-direct-commands-lesson-02-pre.html)
