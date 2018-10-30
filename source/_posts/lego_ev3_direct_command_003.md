---
title: EV3 直接命令 - 第 3 课 遥控车辆
date: 2018-10-30 20:15:49
categories: ROS
tags:
- ROS
- 翻译
---


移动电机是EV3的核心。机器人需要移动的能力，因此你需要知道它是如何完成的。请参阅文档 *EV3 Firmware Developer Kit* 中的第 4.9 部分，以获得与电机相对应的操作的第一印象。几乎所有这些都以相同的两个参数开头：

 * LAYER：如果组合多个 EV3 brick 作为单个机器，则声明其中一个是主机，最多3个额外 brick 作为主机从机。然后将操作发送到主机，主机将其发送给主机从机。这就是 LAYER 的含义。Layer 0x|00| 是主机，layers 0x|01|，0x|02| 和 0x|03| 是它的可选从机。在没有从机的标准情况下，我们将设置 LAYER = 0x|00|。我们的直接命令代码可以连接多个 EV3 设备。因此，我认为不需要不同的参数 LAYER 值。

 * NO 或 NOS：电机端口以字母 A，B，C 和 D 标识。在直接命令的层面，你可以通过数字命名它们，A = 0x|01|，B = 0x|02|，C = 0x|04| 和 D = 0x|08|。如果你将这些数字写为二进制表示，则 A = 0b 0000 0001，B = 0b 0000 0010，C = 0b 0000 0100 和 C = 0b 0000 1000，你看，后半个字节可以解释为一系列的四个标志。你可以组合这些标志。如果你处理端口 A 和 C 的电机操作，则设置 NOS = 0x|05| 或 0b 0000 0101。如果参数名称为 NO，则只能处理单个电机端口。NOS是说，你可以处理一个或多个端口。
<!--more-->
# 启动和停止电机

将一些电机插入你选择的端口，然后以全功率启动它们。这通过以下直接命令完成：
```
----------------------------------------------               
 \ len \ cnt \ty\ hd  \op\la\no\power\op\la\no\              
  ----------------------------------------------             
0x|0D:00|2A:00|80|00:00|A4|00|0F|81:64|A6|00|0F|             
  ----------------------------------------------             
   \ 13  \ 42  \no\ 0,0 \O \0 \a \ 100 \O \0 \a \             
    \     \     \  \     \u \  \l \     \u \  \l \            
     \     \     \  \     \t \  \l \     \t \  \l \           
      \     \     \  \     \p \  \  \     \p \  \  \          
       \     \     \  \     \u \  \  \     \u \  \  \         
        \     \     \  \     \t \  \  \     \t \  \  \        
         \     \     \  \     \_ \  \  \     \_ \  \  \       
          \     \     \  \     \P \  \  \     \S \  \  \      
           \     \     \  \     \o \  \  \     \t \  \  \     
            \     \     \  \     \w \  \  \     \a \  \  \    
             \     \     \  \     \e \  \  \     \r \  \  \   
              \     \     \  \     \r \  \  \     \t \  \  \  
               ----------------------------------------------
```

过了一段时间，你，你的父母，兄弟姐妹或孩子会对全动力马达的声音感到紧张。如果你是一个爱好和平的人，你应该发送以下直接命令：
```
----------------------------------              
 \ len \ cnt \ty\ hd  \op\la\no\br\             
  ----------------------------------            
0x|09:00|2A:00|00|00:00|A3|00|0F|00|            
  ----------------------------------            
   \ 9   \ 42  \re\ 0,0 \O \0 \a \no\            
    \     \     \  \     \u \  \l \  \           
     \     \     \  \     \t \  \l \  \          
      \     \     \  \     \p \  \  \  \         
       \     \     \  \     \u \  \  \  \        
        \     \     \  \     \t \  \  \  \       
         \     \     \  \     \_ \  \  \  \      
          \     \     \  \     \S \  \  \  \     
           \     \     \  \     \t \  \  \  \    
            \     \     \  \     \o \  \  \  \   
             \     \     \  \     \p \  \  \  \  
              ----------------------------------
```

我们看到了三个新操作：

  * opOutput_Power = 0x|A4|，参数是
    - LAYER
    - NOS
    - POWER：指定输出功率 [-100 – 100 %]（负值意味着相反的方向）

  * opOutput_Start = 0x|A6|，参数是
    - LAYER
    - NOS

  * opOutput_Stop = 0x|A3|，参数是
    - LAYER
    - NOS
    - BRAKE：指定制动级别 [0：浮动，1：制动]。这不是被动制动器，其中电动机固定在其位置上。它是一个主动的，电机试图回到原来的位置，这需要能量！

通过这三个操作，我们可以启动和停止电机。我们还应该讨论另一个，它设定了极性。通常，电动机的运动以前后方向描述，并且使用正数和负数。如果我们的机器结构导致极性相反，则有一个操作，可以改变它：

  * opOutput_Polarity = 0x|A7|，参数是
    - LAYER
    - NOS
    - POLARITY：指定极性 [-1, 0, 1]
        * -1：电机将向后运行
        * 0：电机将反方向运行
        * -1：电机将向前运行

定义速度而不是功率通常是更好的选择。幸运的是，乐高电机固定的电机，我们可以通过以下操作轻松定义电机运动的恒定速度：

  * opOutput_Speed = 0x|A5|，参数是
    - LAYER
    - NOS
    - SPEED：指定输出速度 [-100 – 100 %]（负值意味着相反的方向）

请构造一辆由两个电机驱动的车辆，一个用于左侧，另一个用于右侧。精心设计的例子是：

*   [http://robotsquare.com/2015/10/06/explor3r-building-instructions](http://robotsquare.com/2015/10/06/explor3r-building-instructions)
*   [http://robotsquare.com/wp-content/uploads/2013/10/45544_educator.pdf](http://robotsquare.com/wp-content/uploads/2013/10/45544_educator.pdf)
*   [http://www.lego.com/en-gb/mindstorms/build-a-robot/track3r](http://www.lego.com/en-gb/mindstorms/build-a-robot/track3r)
*   [http://www.lego.com/en-gb/mindstorms/build-a-robot/robodoz3r](http://www.lego.com/en-gb/mindstorms/build-a-robot/robodoz3r)

但一个简单的也将满足我们的需求。将右侧电机连接到端口 A，将左侧电机连接到端口 D，并发送以下直接命令：
```
-------------------------------------------               
 \ len \ cnt \ty\ hd  \op\la\no\sp\op\la\no\              
  -------------------------------------------             
0x|0C:00|2A:00|80|00:00|A5|00|09|3D|A6|00|09|             
  -------------------------------------------             
   \ 12  \ 42  \no\ 0,0 \O \0 \A \-3\O \0 \A \             
    \     \     \  \     \u \  \+ \  \u \  \+ \            
     \     \     \  \     \t \  \D \  \t \  \D \           
      \     \     \  \     \p \  \  \  \p \  \  \          
       \     \     \  \     \u \  \  \  \u \  \  \         
        \     \     \  \     \t \  \  \  \t \  \  \        
         \     \     \  \     \_ \  \  \  \_ \  \  \       
          \     \     \  \     \S \  \  \  \S \  \  \      
           \     \     \  \     \p \  \  \  \t \  \  \     
            \     \     \  \     \e \  \  \  \a \  \  \    
             \     \     \  \     \e \  \  \  \r \  \  \   
              \     \     \  \     \d \  \  \  \t \  \  \  
               -------------------------------------------
```

希望你的车辆以低速直行。使用上面的直接命令（opOutput_Stop）来停止它。

# 遥控车辆

现在是编写实际应用程序的时候了。使用遥控 EV3 车辆给你的朋友或孩子留下深刻印象。我们从一个简单的遥控器开始，并使用键盘箭头键来改变速度和方向。首先将新操作添加到 EV3 类，然后编写远程控制程序。像往常一样，我已经以python3 完成了它：
```
#!/usr/bin/env python3

import curses
import ev3

def move(speed: int, turn: int) -> None:
    global myEV3, stdscr
    stdscr.addstr(5, 0, 'speed: {}, turn: {}      '.format(speed, turn))
    if turn > 0:
        speed_right = speed
        speed_left  = round(speed * (1 - turn / 100))
    else:
        speed_right = round(speed * (1 + turn / 100))
        speed_left  = speed
    ops = b''.join([
        ev3.opOutput_Speed,
        ev3.LCX(0),                       # LAYER
        ev3.LCX(ev3.PORT_A),              # NOS
        ev3.LCX(speed_right),             # SPEED
        ev3.opOutput_Speed,
        ev3.LCX(0),                       # LAYER
        ev3.LCX(ev3.PORT_D),              # NOS
        ev3.LCX(speed_left),              # SPEED
        ev3.opOutput_Start,
        ev3.LCX(0),                       # LAYER
        ev3.LCX(ev3.PORT_A + ev3.PORT_D)  # NOS
    ])
    myEV3.send_direct_cmd(ops)

def stop() -> None:
    global myEV3, stdscr
    stdscr.addstr(5, 0, 'vehicle stopped                         ')
    ops = b''.join([
        ev3.opOutput_Stop,
        ev3.LCX(0),                       # LAYER
        ev3.LCX(ev3.PORT_A + ev3.PORT_D), # NOS
        ev3.LCX(0)                        # BRAKE
    ])
    myEV3.send_direct_cmd(ops)

def react(c):
    global speed, turn
    if c in [ord('q'), 27, ord('p')]:
        stop()
        return
    elif c == curses.KEY_LEFT:
        turn += 5
        turn = min(turn, 200)
    elif c == curses.KEY_RIGHT:
        turn -= 5
        turn = max(turn, -200)
    elif c == curses.KEY_UP:
        speed += 5
        speed = min(speed, 100)
    elif c == curses.KEY_DOWN:
        speed -= 5
        speed = max(speed, -100)
    move(speed, turn)

def main(window) -> None:
    global stdscr
    stdscr = window
    stdscr.clear()      # print introduction
    stdscr.refresh()
    stdscr.addstr(0, 0, 'Use Arrows to navigate your EV3-vehicle')
    stdscr.addstr(1, 0, 'Pause your vehicle with key <p>')
    stdscr.addstr(2, 0, 'Terminate with key <q>')

    while True:
        c = stdscr.getch()
        if c in [ord('q'), 27]:
            react(c)
            break
        elif c in [ord('p'),
                   curses.KEY_RIGHT, curses.KEY_LEFT, curses.KEY_UP, curses.KEY_DOWN]:
            react(c)

speed = 0
turn  = 0   
myEV3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
stdscr = None

# ops = opOutput_Polarity + b'\x00' + LCX(PORT_A + PORT_D) + LCX(-1)
# myEV3.send_direct_cmd(ops)

curses.wrapper(main)
```

一些说明：

 * 我使用模块 `curses` 来实现键盘事件的事件处理程序。
 * 方法 `wrapper` 控制资源键盘和终端。它改变终端的行为，创建一个窗口对象并调用函数 `main`。
 * 函数 `main` 将一些文本写入终端，然后等待键事件（方法 `getch`）。
 * 如果其中一个相关键命中，则调用函数 `react`，键 `<q>` 或 `<Ctrl-c>` 打破循环。
 * 变量 `speed` 定义了更快轮子的速度。
 * 变量 `turn` 定义方向。值 0 表示直行，正值表示左转，负值表示右转。转弯的绝对值大意味着转弯的半径小。最大值+200 和最小值 -200 意味着在原地盘旋。在下一课中，我们将回到驱动转弯的这个定义。
 * 我在第 5 步改变了 `speed` 和 `turn`。这似乎是精确度和反应速度之间的良好折衷。
 * 也许，你的车辆结构需要改变两个车轮的极性。这由注释掉的代码完成。
 * 右轮连接到端口 A，左侧连接到端口 D。
 * 对于低速，由于四舍五入到整数值，转弯半径的精度变得更差。

# 中断作为一种设计理念

上一课，我们将中断与不耐烦和乖巧的人进行了比较。本课程表明，这种行为恰恰是一种切实可行的设计理念。

遥控程序发送无限的命令，如果不被下一个命令中断，它将永久地移动电机。控制驾驶的人类希望纠正立即生效。这是绝对正确的，新命令会中断旧命令。

该设计概念的另一个积极方面是直接命令的执行时间短。它们不耗时。电机的运动持续很长时间，但是当电机获得新参数时，直接命令已经执行完成。控制立即返回，程序是自由的，以执行下一个任务。资源 EV3 设备未被阻塞，它可用于下一个直接命令。

你已经看到了如何编码远程控制的中断过程。中心部分是一个请求关键事件的循环。如果相关的按键事件发生，则调用一个函数，该函数更改运动的参数并向 EV3 设备发送直接命令。

当我们在 c''（上一课）中玩三合一时，我们避免了中断。我们有一个固定的所有音调时间表。我们的意见是，每次中断都会扰乱计划，需要加以防范。在绘制三角形时，我们已经讨论了两种时序选择，通过直接命令或本地程序。也可以使用本地程序中的调度并使用中断来播放三元组。这是获得正确时序的替代方法：

```
#!/usr/bin/env python3

import ev3, time

my_ev3 = ev3.EV3(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')

ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(262),
    ev3.LCX(0)
])
my_ev3.send_direct_cmd(ops)
time.sleep(0.5)
ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_GREEN,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(330),
    ev3.LCX(0)
])
my_ev3.send_direct_cmd(ops)
time.sleep(0.5)
ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(392),
    ev3.LCX(0)
])
my_ev3.send_direct_cmd(ops)
time.sleep(0.5)
ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_RED_FLASH,
    ev3.opSound,
    ev3.TONE,
    ev3.LCX(1),
    ev3.LCX(523),
    ev3.LCX(0)
])
my_ev3.send_direct_cmd(ops)
time.sleep(2)
ops = b''.join([
    ev3.opUI_Write,
    ev3.LED,
    ev3.LED_GREEN,
    ev3.opSound,
    ev3.BREAK
])
my_ev3.send_direct_cmd(ops)
```

此版本还实现了固定的调度，但时间发生在程序中而不是 EV3 设备上（在直接命令中）。同样，这有一定的优势，它不阻塞 EV3 设备。只要我们一次只执行一个任务，阻塞就无关紧要了。如果我们试图结合独立的声音和点击驱动，且并行执行任务，则本地时序和中断是必须的。

如果我们在上面的程序中设置 sync_mode == SYNC，数据流量会增长，但我们不会听到任何差异。值得反思的是，该程序同步运行，因为没有直接命令是耗时的。sync_mode == ASYNC 或 sync_mode == STD 是为异步执行设计的，但我们也可以使用这些设置编写同步执行代码。异步执行仅在直接命令耗时（时间由直接命令完成）且本地程序不等待直接命令结束时。中断有助于避免耗时的直接命令，并且是同步执行。

# 继承 EV3

到目前为止，我们直接在我们的程序中编写了操作。这需要封装和抽象。我想特化类，一个用于带两个驱动轮的车辆，一个用于音调和音乐等等。设计应该允许它们平并行使用。我的解决方案是大量的 EV3 类的子类，它们都与同一个 EV3 设备通信。这就是说，它们必须共享资源。为了实现这一点，我修改了 EV3 类的构造函数：

```
     |  __init__(self, protocol:str=None, host:str=None, ev3_obj=None)
     |      Establish a connection to a LEGO EV3 device
     |      
     |      Keyword Arguments (either protocol and host or ev3_obj):
     |      protocol: None, 'Bluetooth', 'Usb' or 'Wifi'
     |      host: None or mac-address of the LEGO EV3 (f.i. '00:16:53:42:2B:99')
     |      ev3_obj: None or an existing EV3 object (its connections will be used)
```

代码是：
```
class EV3:

    def __init__(self, protocol: str=None, host: str=None, ev3_obj=None):
        assert ev3_obj or protocol, \
            'Either protocol or ev3_obj needs to be given'
        if ev3_obj:
            assert isinstance(ev3_obj, EV3), \
                'ev3_obj needs to be instance of EV3'
            self._protocol = ev3_obj._protocol
            self._device = ev3_obj._device
            self._socket = ev3_obj._socket
        elif protocol:
            assert protocol in [BLUETOOTH, WIFI, USB], \
                'Protocol ' + protocol + 'is not valid'
            self._protocol = None
            self._device = None
            self._socket = None
            if protocol == BLUETOOTH:
                assert host, 'Protocol ' + protocol + 'needs host-id'
                self._connect_bluetooth(host)
            elif protocol == WIFI:
                self._connect_wifi()
            elif protocol == USB:
                self._connect_usb()
        self._verbosity = 0
        self._sync_mode = STD
        self._msg_cnt = 41
```

一些注解：

 * 有两种创建类 EV3 的新实例的方式：
    - 和以前一样，使用参数 `protocol` 和 `host` 调用构造函数以获取新连接。
    - 另一种方法是使用现有的 EV3 对象作为参数 `ev3_obj` 调用它，使用其中的连接。
 * 这不仅限于连接。对于将来的扩展，我们可以共享任何资源。
 * 每个类都有自己的 `_verbosity` 和 `_sync_mode`。这是 OK 的，但我们使消息计数器 `_msg_cnt` 成为一个类属性：
```
class EV3:
    _msg_cnt = 41
```

考虑到这一点，我编写了类 `TwoWheelVehicle`，这里是它的构造函数：
```
class TwoWheelVehicle(ev3.EV3):

    def __init__(
            self,
            protocol: str=None,
            host: str=None,
            ev3_obj: ev3.EV3=None
    ):
        super().__init__(protocol=protocol, host=host, ev3_obj=ev3_obj)
        self._polarity = 1
        self._port_left = ev3.PORT_D
        self._port_right = ev3.PORT_A
```

我添加了两个方法：
```
    def move(self, speed: int, turn: int) -> None:
        assert self._sync_mode != ev3.SYNC, "no unlimited operations allowed in sync_mode SYNC"
        assert isinstance(speed, int), "speed needs to be an integer value"
        assert -100 <= speed and speed <= 100, "speed needs to be in range [-100 - 100]"
        assert isinstance(turn, int), "turn needs to be an integer value"
        assert -200 <= turn and turn <= 200, "turn needs to be in range [-200 - 200]"
        if self._polarity == -1:
            speed *= -1
        if self._port_left < self._port_right:
            turn *= -1
        if turn > 0:
            speed_right = speed
            speed_left  = round(speed * (1 - turn / 100))
        else:
            speed_right = round(speed * (1 + turn / 100))
            speed_left  = speed
        ops = b''.join([
            ev3.opOutput_Speed,
            ev3.LCX(0),              # LAYER
            ev3.LCX(PORT_A),         # NOS
            ev3.LCX(speed_right),    # SPEED
            ev3.opOutput_Speed,
            ev3.LCX(0),              # LAYER
            ev3.LCX(PORT_D),         # NOS
            ev3.LCX(speed_left),     # SPEED
            ev3.opOutput_Start,
            ev3.LCX(0),              # LAYER
            ev3.LCX(PORT_A + PORT_D) # NOS
        ])
        self.send_direct_cmd(ops)
```

和
```
    def stop(self, brake: bool=False) -> None:
        assert isinstance(brake, bool), "brake needs to be a boolean value"
        if brake:
            br = 1
        else:
            br = 0
        ops_stop = b''.join([
            ev3.opOutput_Stop,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right), # NOS
            ev3.LCX(br)                                  # BRAKE
        ])
        self.send_direct_cmd(ops)
```

我们需要三个属性：
```
    @property
    def polarity(self):
        return self._polarity
    @polarity.setter
    def polarity(self, value: int):
        assert isinstance(value, int), "polarity needs to be of type int"
        assert value in [1, -1], "allowed polarity values are: -1 or 1"
        self._polarity = value

    @property
    def port_right(self):
        return self._port_right
    @port_right.setter
    def port_right(self, value: int):
        assert isinstance(value, int), "port needs to be of type int"
        assert value in [ev3.PORT_A, ev3.PORT_B, ev3.PORT_C, ev3.PORT_D], "value is not an allowed port"
        self._port_right = value

    @property
    def port_left(self):
        return self._port_left
    @port_left.setter
    def port_left(self, value: int):
        assert isinstance(value, int), "port needs to be of type int"
        assert value in [ev3.PORT_A, ev3.PORT_B, ev3.PORT_C, ev3.PORT_D], "value is not an allowed port"
        self._port_left = value
```

就是这样，我们有了一个新类 `TwoWheelVehicle`，它具有这样的 API：
```
Help on module ev3_vehicle:

NAME
    ev3_vehicle - EV3 vehicle

CLASSES
    ev3.EV3(builtins.object)
        TwoWheelVehicle
    
    class TwoWheelVehicle(ev3.EV3)
     |  ev3.EV3 vehicle with two drived Wheels
     |  
     |  Method resolution order:
     |      TwoWheelVehicle
     |      ev3.EV3
     |      builtins.object
     |  
     |  Methods defined here:
     |  
     |  __init__(self, protocol:str=None, host:str=None, ev3_obj:ev3.EV3=None)
     |      Establish a connection to a LEGO EV3 device
     |      
     |      Keyword Arguments (either protocol and host or ev3_obj):
     |      protocol
     |        BLUETOOTH == 'Bluetooth'
     |        USB == 'Usb'
     |        WIFI == 'Wifi'
     |      host: mac-address of the LEGO EV3 (f.i. '00:16:53:42:2B:99')
     |      ev3_obj: an existing EV3 object (its connections will be used)
     |  
     |  move(self, speed:int, turn:int) -> None
     |      Start unlimited movement of the vehicle
     |      
     |      Arguments:
     |      speed: speed in percent [-100 - 100]
     |        > 0: forward
     |        < 0: backward
     |      turn: type of turn [-200 - 200]
     |        -200: circle right on place
     |        -100: turn right with unmoved right wheel
     |         0  : straight
     |         100: turn left with unmoved left wheel
     |         200: circle left on place
     |  
     |  stop(self, brake:bool=False) -> None
     |      Stop movement of the vehicle
     |      
     |      Arguments:
     |      brake: flag if activating brake
     |  
     |  ----------------------------------------------------------------------
     |  Data descriptors defined here:
     |  
     |  polarity
     |      polarity of motor rotation (values: -1, 1, default: 1)
     |  
     |  port_left
     |      port of left wheel (default: PORT_D)
     |  
     |  port_right
     |      port of right wheel (default: PORT_A)
     |  
     |  ----------------------------------------------------------------------
     |  Methods inherited from ev3.EV3:
     |  
     |  __del__(self)
     |      closes the connection to the LEGO EV3
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
     |  Data descriptors inherited from ev3.EV3:
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
```

从现在开始，当我们使用两个驱动车轮移动车辆时，我们将使用`ClassWheelVehicle`。实际上这不仅仅是一个练习，而是一个真实的东西，但下一课，我们将为这个类添加功能。你可能会问，如果努力和利益真的是平衡的话。但是看，我们的程序变短了：
```
#!/usr/bin/env python3

import curses
import ev3, ev3_vehicle

def react(c):
    global speed, turn, my_vehicle
    if c in [ord('q'), 27, ord('p')]:
        my_vehicle.stop()
        return
    elif c == curses.KEY_LEFT:
        turn += 5
        turn = min(turn, 200)
    elif c == curses.KEY_RIGHT:
        turn -= 5
        turn = max(turn, -200)
    elif c == curses.KEY_UP:
        speed += 5
        speed = min(speed, 100)
    elif c == curses.KEY_DOWN:
        speed -= 5
        speed = max(speed, -100)
    stdscr.addstr(5, 0, 'speed: {}, turn: {}      '.format(speed, turn))
    my_vehicle.move(speed, turn)

def main(window) -> None:
    global stdscr
    stdscr = window
    stdscr.clear()      # print introduction
    stdscr.refresh()
    stdscr.addstr(0, 0, 'Use Arrows to navigate your EV3-vehicle')
    stdscr.addstr(1, 0, 'Pause your vehicle with key <p>')
    stdscr.addstr(2, 0, 'Terminate with key <q>')

    while True:
        c = stdscr.getch()
        if c in [ord('q'), 27]:
            react(c)
            break
        elif c in [ord('p'),
                   curses.KEY_RIGHT, curses.KEY_LEFT, curses.KEY_UP, curses.KEY_DOWN]:
            react(c)

speed = 0
turn  = 0   
my_vehicle = ev3_vehicle.TwoWheelVehicle(protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
stdscr = None

curses.wrapper(main)
```

如果你下载了 [ev3-python3](https://github.com/ChristophGaukel/ev3-python3) 的模块 `ev3_vehicle.py`，你需要以两个额外的参数 `radius_wheel` 和 `tread` 调用 `TwoWheelVehicle` 的构造函数。目前你可以将它们设置为任何正值。

# 结论

现在我们知道了，如何移动电机，我们获得了两个重要设计概念的经验，即中断和子类化。此外我们编写的东西像个真正的应用程序了，一个车辆的遥控器。

我希望，你的遥控器工作良好，并使你成为一个技术明星。我们即将结束本课程。浏览文档 *EV3 Firmware Developer Kit* 告诉您，移动电机存在许多额外的操作。其中大多数是为了精确，同步和平滑的运动。这将是我们下一课的主题。我希望能再次见到你。

[原文](http://ev3directcommands.blogspot.com/2016/01/ev3-direct-commands-lesson-03-pre.html)
