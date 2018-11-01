---
title: EV3 直接命令 - 第 5 课 从 EV3 的传感器读取数据
date: 2018-10-31 21:15:49
categories: ROS
tags:
- ROS
- 翻译
---


# 读取传感器的类型和模式

我们从 EV3 设备的一些自反映开始并询问它：

1. 端口 16 上连接了什么类型的设备？
2. 端口 16 上的传感器的模式？
<!--more-->
请给你的 EV3 发送如下的直接命令：
```
----------------------------------------               
 \ len \ cnt \ty\ hd  \op\cd\la\no\ty\mo\              
  ----------------------------------------             
0x|0B:00|2A:00|00|02:00|99|05|00|10|60|61|             
  ----------------------------------------             
   \ 11  \ 42  \re\ 0,2 \I \G \0 \16\0 \1 \             
    \     \     \  \     \n \E \  \  \  \  \            
     \     \     \  \     \p \T \  \  \  \  \           
      \     \     \  \     \u \_ \  \  \  \  \          
       \     \     \  \     \t \T \  \  \  \  \         
        \     \     \  \     \_ \Y \  \  \  \  \        
         \     \     \  \     \D \P \  \  \  \  \       
          \     \     \  \     \e \E \  \  \  \  \      
           \     \     \  \     \v \M \  \  \  \  \     
            \     \     \  \     \i \O \  \  \  \  \    
             \     \     \  \     \c \D \  \  \  \  \   
              \     \     \  \     \e \E \  \  \  \  \  
               ----------------------------------------
```

一些说明：

 * 我们以 CMD `GET_TYPEMODE` 使用操作 `opInput_device`
 * 如果被用作传感器端口，电机端口 A 的编号为 16。
 * 我们将获得两个数作为应答，类型和模式。
 * 类型需要一个字节且将占据字节 0 的位置，模式也需要一个字节且被放在字节 1 的位置。

我获得了如下的回答：
```
----------------------    
 \ len \ cnt \rs\ty\mo\   
  ----------------------  
0x|05:00|2A:00|02|07|00|  
  ----------------------  
   \ 5   \ 42  \ok\7 \0 \  
    ----------------------
```

它是说：电机端口 A 处的传感器具有类型 7 且实际处于模式 0 中。如果你看一下文档 *EV3 Firmware Developer Kit* 的第 5 章，其标题为 *Device type list*，你发现类型 7 和模式 0 代表 *EV3-Large-Motor-Degree*。 [### EV3 Firmware Developer Kit](https://le-www-live-s.legocdn.com/sc/media/files/ev3-developer-kit/lego%20mindstorms%20ev3%20firmware%20developer%20kit-7be073548547d99f7df59ddfd57c0088.pdf?la=en-us)。

# 读取电机的实际位置

我们来到一个非常有趣的问题：电机端口 A 的电机实际位置是多少？我发送了这个命令：
```
----------------------------------------------                  
 \ len \ cnt \ty\ hd  \op\cd\la\no\ty\mo\va\v1\                 
  ----------------------------------------------                
0x|0D:00|2A:00|00|04:00|99|1C|00|10|07|00|01|60|                
  ----------------------------------------------                
   \ 13  \ 42  \re\ 0,4 \I \R \0 \16\E \D \1 \0 \                
    \     \     \  \     \n \E \  \  \V \e \  \  \               
     \     \     \  \     \p \A \  \  \3 \g \  \  \              
      \     \     \  \     \u \D \  \  \- \r \  \  \             
       \     \     \  \     \t \Y \  \  \L \e \  \  \            
        \     \     \  \     \_ \_ \  \  \a \e \  \  \           
         \     \     \  \     \D \R \  \  \r \  \  \  \          
          \     \     \  \     \e \A \  \  \g \  \  \  \         
           \     \     \  \     \v \W \  \  \e \  \  \  \        
            \     \     \  \     \i \  \  \  \- \  \  \  \       
             \     \     \  \     \c \  \  \  \M \  \  \  \      
              \     \     \  \     \e \  \  \  \o \  \  \  \     
               \     \     \  \     \  \  \  \  \t \  \  \  \    
                \     \     \  \     \  \  \  \  \o \  \  \  \   
                 \     \     \  \     \  \  \  \  \r \  \  \  \  
                  ----------------------------------------------
```

我得到了答复：
```
----------------------------    
 \ len \ cnt \rs\ degrees   \   
  ----------------------------  
0x|07:00|2A:00|02|00:00:00:00|  
  ----------------------------  
   \ 7   \ 42  \ok\ 0         \  
    ----------------------------
```

然后我用手移动电机并再次发送相同的直接命令。这次回复是：
```
----------------------------    
 \ len \ cnt \rs\ degrees   \   
  ----------------------------  
0x|07:00|2A:00|02|50:07:00:00|  
  ----------------------------  
   \ 7   \ 42  \ok\ 1872      \  
    ----------------------------
```

那是说：电机移动了 1, 872 度（5.2 周）。这似乎是对的！

# 技术细节

是时候看一下幕后的东西了！你需要理解：

 * 端口编号的系统，
 * 我们使用的操作的参数，和
 * 如何定位并解包全局内存。

## 端口编号的系统

传感器有四个端口，电机有四个端口。传感器端口的编号是 1 到 4：

 * 端口 1: PORT = 0x|00| 或 LCX(0)
 * 端口 2: PORT = 0x|01| 或 LCX(1)
 * 端口 3: PORT = 0x|02| 或 LCX(2)
 * 端口 4: PORT = 0x|03| 或 LCX(3)

这似乎有点滑稽，但计算机通常从数字 0 开始计数，人类从数字 1 开始计数。我们刚刚了解到，电机也是传感器，我们可以从中读取电机的实际位置。电机端口标为字母 A 到 D，但通过如下方式定位：

 * 端口 A: PORT = 0x|10| 或 LCX(16)
 * 端口 B: PORT = 0x|11| 或 LCX(17)
 * 端口 C: PORT = 0x|12| 或 LCX(18)
 * 端口 D: PORT = 0x|13| 或 LCX(19)

我在我的模块 ev3.py 中添加了一个小函数：
```
def port_motor_input(port_output: int) -> bytes:
    """
    get corresponding input motor port (from output motor port)
    """
    if port_output == PORT_A:
        return LCX(16)
    elif port_output == PORT_B:
        return LCX(17)
    elif port_output == PORT_C:
        return LCX(18)
    elif port_output == PORT_D:
        return LCX(19)
    else:
        raise ValueError("port_output needs to be one of the port numbers [1, 2, 4, 8]")
```

从电机输出端口转换到输入端口。

## 操作 opInput_Device

opInput_Device 的两个变体的简短描述，我们已经使用了：

 * opInput_Device = 0x|99| 的 CMD GET_TYPEMODE = 0x|05|：
参数
    - (Data8) LAYER：链 layer 号
    - (Data8) NO：端口编号

返回值
    - (Data8) TYPE：设备类型
    - (Data8) MODE：设备模式

 * opInput_Device = 0x|99| 的 CMD READY_RAW = 0x|1C|：
参数
    - (Data8) LAYER：链 layer 号
    - (Data8) NO：端口编号
    - (Data8) TYPE：设备类型
    - (Data8) MODE：设备模式
    - (Data8) VALUES：返回值的个数
返回值
    - (Data32) VALUE1：以特定模式从传感器接收的第一个值

这里 Data32 是说这是一个 32 位有符号整数。 返回的数据是值，但请记住，返回参数如 VALUE1 是引用。引用是局部或全局内存的地址。阅读下一部分了解详情。

## 寻址全局内存

在第 2 课中，我们介绍了常量参数和局部变量。你将记得，我们已经看到了 LCS，LC0，LC1，LC2，LC4，LV0，LV1，LV2 和 LV4，并写了三个函数 LCX(value:int)，LVX(value:int) 和 LCS(value:str)：
```
FUNCTIONS
    LCS(value:str) -> bytes
        pack a string into a LCS
    
    LCX(value:int) -> bytes
        create a LC0, LC1, LC2, LC4, dependent from the value
    
    LVX(value:int) -> bytes
        create a LV0, LV1, LV2, LV4, dependent from the value
```

我们讨论了标识字节，它定义了变量的类型和长度：

现在我们编写另一个函数 GVX，它返回全局内存的地址。如你已经知道的那样，标识字节的位 0 代表短格式或长格式：

 * `0b 0... ....` 短格式（只有一个字节，标识字节包含值）
 * `0b 1... ....` 长格式（标识字节不包含任何值的位）

如果位 1 和 2 是 `0b .11. ....`，它们代表全局变量，它们是全局内存的地址。

位 6 和 7 代表后续的值的长度

 * `0b .... ..00` 意味着可变长度，
 * `0b .... ..01` 意味着后面有一个字节，
 * `0b .... ..10` 是说，后面有两个字节，
 * `0b .... ..11` 是说，后面有四个字节。

现在我们写 4 个全局变量作为二进制掩码，我们不需要符号，因为地址总是正数。 V 代表地址（值）的一位。

 * `GV0`： `0b 011V VVVV`，5 位地址，范围：0 - 31，长度：1 字节，由前导位 011 标识。
 * `GV1`：`0b 1110 0001 VVVV VVVV`，8 位地址，范围：0 - 255，长度：2 字节，由前导字节 `0x|E1|` 标识。
 * `GV2`：`0b 1110 0010 VVVV VVVV VVVV VVVV`，16 位地址，范围：0 – 65.536，长度：3 字节，由前导字节 ` 0x|E2|` 标识。
 * `GV4`：`0b 1110 0011 VVVV VVVV VVVV VVVV VVVV VVVV VVVV VVVV`，32 位地址，范围：0 – 4,294,967,296，长度：5 字节，由前导字节 `0x|E3|` 标识。

一些说明：

 * 在直接命令中，不需要 GV4！你记得全局内存最多有 1019 个字节 (1024 - 5)。
 * 必须正确放置全局内存的地址。 如果将 4 字节值写入全局内存，则其地址必须为 0，4，8，......（4的倍数）。 对于 2 字节值也是一样，它们的地址必须是 2 的倍数。
 * 你将需要将全局内存拆分为所需长度的段，然后使用每个段的第一个字节的地址。我们的第一个例子中，我们需要两个段（类型和模式），每个段一个字节。因此我们使用 GV0(0) 和 GV0(1) 作为地址。
 * 头字节包含全局内存的总长度（有关详细内容，请参阅第 1 课）。在我们的例子中，这些是2个字节 resp. 4字节。不要忘记正确发送头字节！
 * 不要在段之间留下空隙！标准的工具诸如 `struct.unpack` 不喜欢它们。把 4 字节类型放在前面，然后是 2 字节类型以此类推。这使得对拆包进行编码比较方便。

## 一个新模块函数：GVX

请给你的 ev3 模块添加一个函数 `GVX(value)`，依赖于值，它返回 GV0，GV1，GV2或 GV4 中最短的类型。我已经完成了，现在我的模块 ev3 的文档如下：
```
FUNCTIONS
    GVX(value:int) -> bytes
        create a GV0, GV1, GV2, GV4, dependent from the value
    
    LCS(value:str) -> bytes
        pack a string into a LCS
    
    LCX(value:int) -> bytes
        create a LC0, LC1, LC2, LC4, dependent from the value
    
    LVX(value:int) -> bytes
        create a LV0, LV1, LV2, LV4, dependent from the value
    
    port_motor_input(port_output:int) -> bytes
        get corresponding input motor port (from output motor port)
```

## 解包全局内存

我已经提到，已经有了解包全局内存的好工具了。在 Python 3 中，这个工具是 [struct — Interpret bytes as packed binary data](https://docs.python.org/3.0/library/struct.html) 。

### 一字节无符号整数

我的从电机端口 A 读取模式和类型的程序：
```
#!/usr/bin/env python3

import ev3, struct

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opInput_Device,
    ev3.GET_TYPEMODE,
    ev3.LCX(0),                   # LAYER
    ev3.port_motor_input(PORT_A), # NO
    ev3.GVX(0),                   # TYPE
    ev3.GVX(1)                    # MODE
])
reply = my_ev3.send_direct_cmd(ops, global_mem=2)
(type, mode) = struct.unpack('BB', reply[5:])
print("type: {}, mode: {}".format(type, mode))
```

模式 'BB' 把全局内存分为两个 1 字节的无符号整数值。这个程序的输出是：
```
08:08:13.477998 Sent 0x|0B:00|2A:00|00|02:00|99:05:00:10:60:61|
08:08:13.558793 Recv 0x|05:00|2A:00|02|07:00|
type: 7, mode: 0
```

**四个字节的浮点数和四个字节的有符号整数**

我的读取端口 A 和端口 D 上的电机的电机位置的程序：
```
#!/usr/bin/env python3

import ev3, struct

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opInput_Device,
    ev3.READY_SI,
    ev3.LCX(0),                   # LAYER
    ev3.port_motor_input(PORT_A), # NO
    ev3.LCX(7),                   # TYPE
    ev3.LCX(0),                   # MODE
    ev3.LCX(1),                   # VALUES
    ev3.GVX(0),                   # VALUE1
    ev3.opInput_Device,
    ev3.READY_RAW,
    ev3.LCX(0),                   # LAYER
    ev3.port_motor_input(PORT_D), # NO
    ev3.LCX(7),                   # TYPE
    ev3.LCX(0),                   # MODE
    ev3.LCX(1),                   # VALUES
    ev3.GVX(4)                    # VALUE1
])
reply = my_ev3.send_direct_cmd(ops, global_mem=8)
(pos_a, pos_d) = struct.unpack('<fi', reply[5:])
print("positions in degrees (ports A and D): {} and {}".format(pos_a, pos_d))
```

格式 '<fi' 将全局内存分为一个 4 字节的浮点数和一个 4 字节的有符号整数，都是小尾端的。输出是：
```
08:32:32.865522 Sent 0x|15:00|2A:00|00|08:00|99:1D:00:10:07:00:01:60:99:1C:00:13:07:00:01:64|
08:32:32.949266 Recv 0x|0B:00|2A:00|02|00:80:6C:C4:54:04:00:00|
positions in degrees (ports A and D): -946.0 and 1108
```

**字符串**

我们读取 EV3 设备的名字：
```
#!/usr/bin/env python3

import ev3, struct

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opCom_Get,
    ev3.GET_BRICKNAME,
    ev3.LCX(16),                  # LENGTH
    ev3.GVX(0)                    # NAME
])
reply = my_ev3.send_direct_cmd(ops, global_mem=16)
(brickname,) = struct.unpack('16s', reply[5:])
brickname = brickname.split(b'\x00')[0]
brickname = brickname.decode("ascii")
print("Brickname:", brickname)
```

说明：

 * 格式 '16s' 描述了一个 16 字节的字符串。
 * brickname = brickname.split(b'\x00')[0] 占据了以 0 结尾的字符串的第一部分。你需要那样做是因为 EV3 设备不清除全局内存。在字符串的右端部分也许有一些垃圾。等一会儿，然后我将演示这个问题。
 * brickname = brickname.decode("ascii") 从字节类型创建一个字符串类型。

这个程序的输出是：
```
08:55:00.098825 Sent 0x|0A:00|2B:00|00|10:00|D3:0D:81:20:60|
08:55:00.138258 Recv 0x|13:00|2B:00|02|6D:79:45:56:33:00:00:00:00:00:00:00:00:00:00:00|
Brickname: myEV3
```

**带有垃圾的字符串**

我们发送两个直接命令，第二个读取一个字符串：
```
#!/usr/bin/env python3

import ev3, struct

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_ev3.verbosity = 1

ops = b''.join([
    ev3.opInput_Device,
    ev3.READY_SI,
    ev3.LCX(0),                   # LAYER
    ev3.port_motor_input(PORT_A), # NO
    ev3.LCX(7),                   # TYPE
    ev3.LCX(0),                   # MODE
    ev3.LCX(1),                   # VALUES
    ev3.GVX(0),                   # VALUE1
    ev3.opInput_Device,
    ev3.READY_RAW,
    ev3.LCX(0),                   # LAYER
    ev3.port_motor_input(PORT_D), # NO
    ev3.LCX(7),                   # TYPE
    ev3.LCX(0),                   # MODE
    ev3.LCX(1),                   # VALUES
    ev3.GVX(4)                    # VALUE1
])
reply = my_ev3.send_direct_cmd(ops, global_mem=8)
(pos_a, pos_d) = struct.unpack('<fi', reply[5:])
print("positions in degrees (ports A and D): {} and {}".format(pos_a, pos_d))

ops = b''.join([
    ev3.opCom_Get,
    ev3.GET_BRICKNAME,
    ev3.LCX(16),                  # LENGTH
    ev3.GVX(0)                    # NAME
])
reply = my_ev3.send_direct_cmd(ops, global_mem=16)
```

这个程序的输出是：
```
09:13:30.379771 Sent 0x|15:00|2A:00|00|08:00|99:1D:00:10:07:00:01:60:99:1C:00:13:07:00:01:64|
09:13:30.433495 Recv 0x|0B:00|2A:00|02|00:08:90:C5:FE:F0:FF:FF|
positions in degrees (ports A and D): -4609.0 and -3842
09:13:30.433932 Sent 0x|0A:00|2B:00|00|10:00|D3:0D:81:20:60|
09:13:30.502499 Recv 0x|13:00|2B:00|02|6D:79:45:56:33:00:FF:FF:00:00:00:00:00:00:00:00|
```

以 0 结尾的字符串 'myEV3' (0x|6D:79:45:56:33:00|) 的长度为 6 个字节。接下来的两个字节 (0x|FF:FF|) 是来自于第一个直接命令的垃圾。

# 最快的拇指

触屏传感器的类型编号为 16，且有两个模式，0: EV3-Touch 和 1: EV3-Bump。第一个测试，如果传感器实际被触摸了，第二个从上次清除传感器开始计算触摸。我们通过一个小程序演示这些模式。它计数，摸传感器在五秒钟内撞击的频率（请在端口 2 插入你的触摸传感器）：
```
#!/usr/bin/env python3

import ev3, struct, time

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')

def change_color(color) -> bytes:
    return b''.join([
        ev3.opUI_Write,
        ev3.LED,
        color
    ])

def play_sound(vol: int, freq: int, dur:int) -> bytes:
    return b''.join([
        ev3.opSound,
        ev3.TONE,
        ev3.LCX(vol),
        ev3.LCX(freq),
        ev3.LCX(dur)
    ])

def ready() -> None:
    ops = change_color(ev3.LED_RED)
    my_ev3.send_direct_cmd(ops)
    time.sleep(3)

def steady() -> None:
    ops_color = change_color(ev3.LED_ORANGE)
    ops_sound = play_sound(1, 200, 60)
    my_ev3.send_direct_cmd(ops_color + ops_sound)
    time.sleep(0.25)
    for i in range(3):
        my_ev3.send_direct_cmd(ops_sound)
        time.sleep(0.25)

def go() -> None:
    ops_clear = b''.join([
        ev3.opInput_Device,
        ev3.CLR_CHANGES,
        ev3.LCX(0),          # LAYER
        ev3.LCX(1)           # NO
    ])
    ops_color = change_color(ev3.LED_GREEN_FLASH)
    ops_sound = play_sound(10, 200, 100)
    my_ev3.send_direct_cmd(ops_clear + ops_color + ops_sound)
    time.sleep(5)

def stop() -> None:
    ops_read = b''.join([
        ev3.opInput_Device,
        ev3.READY_SI,
        ev3.LCX(0),          # LAYER
        ev3.LCX(1),          # NO
        ev3.LCX(16),         # TYPE - EV3-Touch
        ev3.LCX(0),          # MODE - Touch
        ev3.LCX(1),          # VALUES
        ev3.GVX(0),          # VALUE1
        ev3.opInput_Device,
        ev3.READY_SI,
        ev3.LCX(0),          # LAYER
        ev3.LCX(1),          # NO
        ev3.LCX(16),         # TYPE - EV3-Touch
        ev3.LCX(1),          # MODE - Bump
        ev3.LCX(1),          # VALUES
        ev3.GVX(4)           # VALUE1
    ])
    ops_sound = play_sound(10, 200, 100)
    reply = my_ev3.send_direct_cmd(ops_sound + ops_read, global_mem=8)
    (touched, bumps) = struct.unpack('<ff', reply[5:])
    if touched == 1:
        bumps += 0.5
    print(bumps, "bumps")

for i in range(3):
    ready()
    steady()
    go()
    stop()
ops_color = change_color(ev3.LED_GREEN)
my_ev3.send_direct_cmd(ops_color)
print("**** Game over ****")
```

我们使用了一个新操作：opInput_Device = 0x|99| 的 CMD CLR_CHANGES = 0x|1A|，有这些参数：
 * (Data8) LAYER：链 layer 号
 * (Data8) NO：端口编号

它清除传感器，所有它的内部数据被设置为初始值。

# 郁闷的长颈鹿

让我们编写一个程序，它使用类 `TwoWheelVehicle` 和红外传感器。红外传感器的类型编号为 33，它的模式 0 读取传感器前方的自由距离。我们使用它来探测小车前方的障碍和坑洞。转换你的小车并放置红外传感器，使其看向前方，但从上到下（向下约30 - 60°）。传感器读取小车前方的区域并在遇到意外状况时停止运动：
```
#!/usr/bin/env python3

import ev3, ev3_vehicle, struct, random

vehicle = ev3_vehicle.TwoWheelVehicle(
    0.02128,                 # radius_wheel
    0.1175,                  # tread
    protocol=ev3.BLUETOOTH,
    host='00:16:53:42:2B:99'
)

def distance() -> float:
    ops = b''.join([
        ev3.opInput_Device,
        ev3.READY_SI,
        ev3.LCX(0),          # LAYER
        ev3.LCX(0),          # NO
        ev3.LCX(33),         # TYPE - EV3-IR
        ev3.LCX(0),          # MODE - Proximity
        ev3.LCX(1),          # VALUES
        ev3.GVX(0)           # VALUE1
    ])
    reply = vehicle.send_direct_cmd(ops, global_mem=4)
    return struct.unpack('<f', reply[5:])[0]

speed = 25
vehicle.move(speed, 0)
for i in range(10):
    while True:
        dist = distance()
        if dist < 15 or dist > 20:
            break
    vehicle.stop()
    vehicle.sync_mode = ev3.SYNC
    angle = 135 + 45 * random.random()
    if random.random() > 0.5:
        vehicle.drive_turn(speed, 0, angle)
    else:
        vehicle.drive_turn(speed, 0, angle, right_turn=True)
    vehicle.sync_mode = ev3.STD
    speed -= 2
    vehicle.move(speed, 0)
vehicle.stop()
```

一些注释：

 * 如果你从 [ev3-python3](https://github.com/ChristophGaukel/ev3-python3) 下载了模块 ev3_vehicle.py，请消除属性 `sync_mode` 的设置(vehicle.sync_mode = ev3.SYNC 或 vehicle.sync_mode = ev3.STD)
 * 算法的核心部分是：
```
    while True:
        dist = distance()
        if dist < 15 or dist > 20:
            break
    vehicle.stop()
```

这个代码在自由距离小于 15 cm 或大于 20 cm 时（具体值依赖于对象的构造）停止运动。这是说：如果小车到了桌子的边缘（距离变大），它将停止，以及如果它到了一个障碍物处（小距离），它也将停止。
 * 停止后，车辆以随机方向及随机角度开启（范围在 135 到 180°）。sync_mode 设置为 SYNC，我们想要程序等待直到转弯完成：
```
    vehicle.sync_mode = ev3.SYNC
    angle = 135 + 45 * random.random()
    if random.random() > 0.5:
        vehicle.drive_turn(speed, 0, angle)
    else:
        vehicle.drive_turn(speed, 0, angle, right_turn=True)
    vehicle.sync_mode = ev3.STD
```
 * 然后速度减小，小车向前移动，循环再次开始：
```
    speed -= 2
    vehicle.move(speed, 0)
```
 * 循环数限制为10。
 * 我的传感器放在一个装配长颈鹿颈部的结构上。这个以及越来越慢的运动就成了这个名字。
 * 一个缺点是传感器直接向前聚焦。如果车辆以小角度移动到桌子的边缘或靠着障碍物，它将会识别它太晚。

技术上会发生什么？
 * vehicle.move(speed, 0) 启动一个无限的运动，它不阻塞 EV3 设备。
 * 这允许在小车运动时从传感器读取自由距离。
 * 与第 3 课和第 4 课的远程控制的相似性非常重要，传感器取代了人类的思维。
 * 仅有的阻塞 EV3 设备的行为是方法 `drive_turn`。这个命令需要 sync_mode = SYNC。幸运的是，在它执行时，我们不需要任何传感器数据。

现在是时候适配你的程序来满足你的需要和你的小车的构造了。我发现将小车放在桌面上，其中桌面的一部分被屏障隔开，是最令人印象深刻的。

# 导引头

红外传感器有另一种有趣的模式：`seeker`。这个模式读取 EV3 红外信标的方向和距离。信标允许在四个信号通道中选一个。请在端口 2 插入 IR 传感器，打开信标，选择一个通道，把它放在红外传感器的前方，然后运行这个程序：
```
#!/usr/bin/env python3

import ev3, struct

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')

ops_read = b''.join([
    ev3.opInput_Device,
    ev3.READY_RAW,
    ev3.LCX(0),                       # LAYER
    ev3.LCX(1),                       # NO
    ev3.LCX(33),                      # TYPE - IR
    ev3.LCX(1),                       # MODE - Seeker
    ev3.LCX(8),                       # VALUES
    ev3.GVX(0),                       # VALUE1 - heading   channel 1
    ev3.GVX(4),                       # VALUE2 - proximity channel 1
    ev3.GVX(8),                       # VALUE3 - heading   channel 2
    ev3.GVX(12),                      # VALUE4 - proximity channel 2
    ev3.GVX(16),                      # VALUE5 - heading   channel 3
    ev3.GVX(20),                      # VALUE6 - proximity channel 3
    ev3.GVX(24),                      # VALUE5 - heading   channel 4
    ev3.GVX(28)                       # VALUE6 - proximity channel 4
])
reply = my_ev3.send_direct_cmd(ops_read, global_mem=32)
(
    h1, p1,
    h2, p2,
    h3, p3,
    h4, p4,
) = struct.unpack('8i', reply[5:])
print("heading1: {}, proximity1: {}".format(h1, p1))
print("heading2: {}, proximity2: {}".format(h2, p2))
print("heading3: {}, proximity3: {}".format(h3, p3))
print("heading4: {}, proximity4: {}".format(h4, p4))
```

朝向的范围为 [-25 - 25]，负值代表左，0 代表直行，正的代表右边。接近性的范围为 [0 - 100]，且以 cm 计。这个操作读取所有 4 个通道，每个通道两个值。这个程序的输出是(seeker 通道是 2)：
```
heading1: 0, proximity1: -2147483648
heading2: -21, proximity2: 27
heading3: 0, proximity3: -2147483648
heading4: 0, proximity4: -2147483648
```

信标放在红外传感器的左前方，距离为 27 cm。通道 1，3，和 4 返回一个距离值 -2147483648，它是 0x|00:00:00:80|（小尾端，最高位为 1，所有其它的为 0），表示 *没信号*。

# PID 控制器

[PID 控制器](https://en.wikipedia.org/wiki/PID_controller) 持续计算错误值，作为所需设定值和测量过程变量之间的差值。控制器尝试通过控制变量的调整随时间最小化误差。这是一个伟大的算法，它修改一个过程的参数直到过程达到它的目的状态。最好的是，你不需要知道你的参数的精确依赖以及过程的状态。一个典型的例子是加热房间的暖气片。过程变量是房间的温度，控制器改变暖气片阀的位置直到房间温度稳定在设置的点。我们将使用 PID 控制器调整小车移动的参数 `speed` 和 `turn`。我们给模块 `ev3` 添加一个类 PID：
```
class PID():
    """
    object to implement a PID controller
    """
    def __init__(self,
                 setpoint: float,
                 gain_prop: float,
                 gain_der: float=None,
                 gain_int: float=None,
                 half_life: float=None
    ):
        self._setpoint = setpoint
        self._gain_prop = gain_prop
        self._gain_int = gain_int
        self._gain_der = gain_der
        self._half_life = half_life
        self._error = None
        self._time = None
        self._int = None
        self._value = None

    def control_signal(self, actual_value: float) -> float:
        if self._value is None:
            self._value = actual_value
            self._time = time.time()
            self._int = 0
            self._error = self._setpoint - actual_value
            return self._gain_prop * self._error
        else:
            time_act = time.time()
            delta_time = time_act - self._time
            self._time = time_act
            if self._half_life is None:
                self._value = actual_value
            else:
                fact1 = math.log(2) / self._half_life
                fact2 = math.exp(-fact1 * delta_time)
                self._value = fact2 * self._value + actual_value * (1 - fact2)
            error = self._setpoint - self._value
            if self._gain_int is None:
                signal_int = 0
            else:
                self._int += error * delta_time
                signal_int = self._gain_int * self._int
            if self._gain_der is None:
                signal_der = 0
            else:
                signal_der = self._gain_der * (error - self._error) / delta_time
            self._error = error
            return self._gain_prop * error + signal_int + signal_der
```

这实现了一个PID控制器，只有一个修改：`half_life`。实际值可能有噪声或通过离散步骤改变，我们对它们进行平滑，因为当实际值随机或离散变化时，导数部分将显示峰值。`half_life` 的维度 [s] 为时间，并且是阻尼的半衰期。但请记住：平滑控制器使其变得迟缓！

它的文档为：
```
    class PID(builtins.object)
     |  object to implement a PID controller
     |  
     |  Methods defined here:
     |  
     |  __init__(self, setpoint:float, gain_prop:float, gain_der:float=None, gain_int:float=None, half_life:float=None)
     |      Parametrizes a new PID controller
     |      
     |      Arguments:
     |      setpoint: ideal value of the process variable
     |      gain_prop: proportional gain,
     |                 high values result in fast adaption, but too high values produce oscillations or instabilities
     |      
     |      Keyword Arguments:
     |      gain_der: gain of the derivative part [s], decreases overshooting and settling time
     |      gain_int: gain of the integrative part [1/s], eliminates steady-state error, slower and smoother response
     |      half_life: used for discrete or noisy systems, smooths actual values [s]
     |  
     |  control_signal(self, actual_value:float) -> float
     |      calculates the control signal from the actual value
     |      
     |      Arguments:
     |      actual_value: actual measured process variable (will be compared to setpoint)
     |      
     |      Returns:
     |      control signal, which will be sent to the process
```

# 保持专注

请将红外传感器放在车辆上，水平放在前面。将其插入端口 2，选择信标通道 1，激活信标，然后启动这个程序：
```
#!/usr/bin/env python3

import ev3, ev3_vehicle, struct

vehicle = ev3_vehicle.TwoWheelVehicle(
    0.02128,       # radius_wheel
    0.1175,        # tread
    protocol=ev3.BLUETOOTH,
    host='00:16:53:42:2B:99'
)
ops_read = b''.join([
    ev3.opInput_Device,
    ev3.READY_RAW,
    ev3.LCX(0),    # LAYER
    ev3.LCX(1),    # NO
    ev3.LCX(33),   # TYPE - IR
    ev3.LCX(1),    # MODE - Seeker
    ev3.LCX(2),    # VALUES
    ev3.GVX(0),    # VALUE1 - heading   channel 1
    ev3.GVX(4)     # VALUE2 - proximity channel 1
])
speed_ctrl = ev3.PID(0, 2, half_life=0.1, gain_der=0.2)
while True:
    reply = vehicle.send_direct_cmd(ops_read, global_mem=8)
    (heading, proximity) = struct.unpack('2i', reply[5:])
    if proximity == -2147483648:
        print("**** lost connection ****")
        break
    turn = 200
    speed = round(speed_ctrl.control_signal(heading))
    speed = max(-100, min(100, speed))
    vehicle.move(speed, turn)
vehicle.stop()
```

说明：

 * 我们选择了通道 1，这只允许读取该通道的值。
 * 控制器不是一个 PID，它的 PD 带有平滑的值。
 * 如果你移动信标，你的小车将改变它的方向并保持信标在它的眼镜的焦点上。
 * 这个程序在关闭信标时停止。
 * 前进方向是过程变量，其设定值为 0（直行）。通过将车辆转动到位来完成调整。
 * 请改变 PD 控制器的参数以了解控制机制。
 * 没有稳定状态错误，因为 control_signal == 0 将进程保持在稳定状态并且是唯一的稳定状态。

# 跟我来

我们稍微改变了程序的代码，但从根本上改变了它的含义：
```
#!/usr/bin/env python3

import ev3, ev3_vehicle, struct

vehicle = ev3_vehicle.TwoWheelVehicle(
    0.02128,       # radius_wheel
    0.1175,        # tread
    protocol=ev3.BLUETOOTH,
    host='00:16:53:42:2B:99'
)
ops_read = b''.join([
    ev3.opInput_Device,
    ev3.READY_RAW,
    ev3.LCX(0),    # LAYER
    ev3.LCX(1),    # NO
    ev3.LCX(33),   # TYPE - IR
    ev3.LCX(1),    # MODE - Seeker
    ev3.LCX(2),    # VALUES
    ev3.GVX(0),    # VALUE1 - heading   channel 1
    ev3.GVX(4)     # VALUE2 - proximity channel 1
])
speed_ctrl =  ev3.PID(10, 4, half_life=0.1, gain_der=0.2)
turn_ctrl =  ev3.PID(0, 8, half_life=0.1, gain_der=0.3)
while True:
    reply = vehicle.send_direct_cmd(ops_read, global_mem=8)
    (heading, proximity) = struct.unpack('2i', reply[5:])
    if proximity == -2147483648:
        print("**** lost connection ****")
        break
    turn = round(turn_ctrl.control_signal(heading))
    turn = max(-200, min(200, turn))
    speed = round(-speed_ctrl.control_signal(proximity))
    speed = max(-100, min(100, speed))
    vehicle.move(speed, turn)
vehicle.stop()
```

这个程序使用 heading 来控制移动参数 `turn` 和 `proximity` 继而控制它的 `speed`。`speed_ctrl` 的设定点是一个距离 (10 cm)。如果距离增长，控制器增加小车的速度。你可以减少 10 厘米以下的距离，然后车辆向后移动。控制器总是试图保持或达到信标和红外传感器距离为 10 厘米的状态。请改变两个传感器的参数。

如果信标稳定向前移动并且车辆跟随信标会发生什么？这就像驾驶车队一样，可以研究稳态误差。则 speed = gain_prop * error，即 speed = gain_prop * (proximity - setpoint)。这是说：proximity = speed / gain_prop + setpoint。信标和传感器之间的稳定距离随着速度从 10 cm (speed == 0) 到 35 cm (speed == 100) 的增加而增加。如果我们模拟车辆车队，这正是我们想要的。

我们可以设置 `gain_int` 为一个正值。甚至非常小的值将消除稳态误差。巡航将保持在 10 cm 的距离，甚至在高速的情况下。

# 结论

这一课是关于传感器值的。我们已经看到，电机也是传感器，它允许我们读取实际的电机位置。我们写了一些小程序，使用红外传感器来控制带有两个驱动轮的车辆的运动。这是我们第一个真正的机器人程序。读取传感器值的机器，可以对环境作出反应。

我们获得了一些 PID 控制器方面的经验，PID 控制器是受控过程的行业标准。调整它们的参数取代了复杂算法的编码。我们的程序，使用PID控制器是惊人的紧凑和惊人的统一。PID控制器似乎功能强大且通用。

下一课将改进我们的类 `TwoWheelVehicle`，并为多任务做好准备。我期待着再次见到你。

[原文](http://ev3directcommands.blogspot.com/2016/03/lesson-5-reading-data-from-ev3s-sensors.html)
