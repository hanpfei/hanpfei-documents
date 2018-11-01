---
title: EV3 直接命令 - 第 4 课 用两个驱动轮精确地移动小车
date: 2018-10-31 20:15:49
categories: ROS
tags:
- ROS
- 翻译
---


# 简介

上一课，我们编写了 `TwoWheelVehicle` 类，一个 EV3 的子类。它的方法是 `move` 和 `stop`，但它不仅仅是围绕操作 `opOutput_Speed`，`opOutput_Start` 和 `opOutput_Stop` 的薄薄的封装。本节课的最后，类 `TwoWheelVehicle` 将具有实质的内容。如大多数软件那样，它一步步增长，本节课不会是最后一次，我们将继续使用这个类。
<!--more-->
上一课的主题是一个遥控小车。我们编写了无限移动的代码，它们由新命令中断。我们已经看到了这一设计理念的好处，它不阻塞 EV3 设备。精确移动的第一个版本（本节课的结果）将阻塞 EV3 设备，且它将耗费我们进一步的工作来找到一个不阻塞的方案。

# 同步的电机移动

希望上一节课的车辆仍然存在。我们再次需要它。你还记得，右轮连接到端口 A，左边连接到端口 D。我们的遥控解决方案存在缺陷：移动越慢，转向的精度越差。我们需要的是一个操作，两个电机的速度可以被设置为定义的比率。如你可能期待的那样，这种操作是存在的。请给你的小车发送如下的直接命令：
```
-------------------------------------------------------------                   
 \ len \ cnt \ty\ hd  \op\la\no\sp\ tu  \ step   \br\op\la\no\                  
  -------------------------------------------------------------                 
0x|12:00|2A:00|80|00:00|B0|00|09|3B|81:32|82:D0:02|00|A6|00|09|                 
  -------------------------------------------------------------                 
   \ 18  \ 42  \no\ 0,0 \O \0 \A \-5\ 50  \ 720    \0 \O \0 \A \                 
    \     \     \  \     \u \  \+ \  \     \        \  \u \  \+ \                
     \     \     \  \     \t \  \D \  \     \        \  \t \  \D \               
      \     \     \  \     \p \  \  \  \     \        \  \p \  \  \              
       \     \     \  \     \u \  \  \  \     \        \  \u \  \  \             
        \     \     \  \     \t \  \  \  \     \        \  \t \  \  \            
         \     \     \  \     \_ \  \  \  \     \        \  \_ \  \  \           
          \     \     \  \     \S \  \  \  \     \        \  \S \  \  \          
           \     \     \  \     \t \  \  \  \     \        \  \t \  \  \         
            \     \     \  \     \e \  \  \  \     \        \  \a \  \  \        
             \     \     \  \     \p \  \  \  \     \        \  \r \  \  \       
              \     \     \  \     \_ \  \  \  \     \        \  \t \  \  \      
               \     \     \  \     \S \  \  \  \     \        \  \  \  \  \     
                \     \     \  \     \y \  \  \  \     \        \  \  \  \  \    
                 \     \     \  \     \n \  \  \  \     \        \  \  \  \  \   
                  \     \     \  \     \c \  \  \  \     \        \  \  \  \  \  
                   -------------------------------------------------------------
```

点击 A 旋转 720°，点击 D 移动 360°。小车左转，两个电机都以低速转动且良好的同步。新的操作是：

 * opOutput_Step_Sync = 0xB0，参数是：
    - LAYER
    - NOS：两个输出端口，命令不是对称的，它区别了低端口和高端口。在我们的情况中，低端口是 PORT_A，右轮，高端口是 PORT_D，左轮。
    - SPEED
    - TURN：转向比率，[-200 - 200]
    - STEP：转速脉冲以度为单位，值 0 代表无限运动（如果TURN > 0，则 STEP 限制低端口，如果 TURN < 0，则限制较高端口）。正值，参数 SPEED 的符号确定方向。
    - BRAKE

参数 TURN 需要一些解释。但你处在一个很好的位置，你已经知道了它的含义，因为我们在第 3 课的远程控制程序中使用了它。如你可能记得的那样，我们计算两轮的速度：
```
    if turn > 0:
        speed_right  = speed
        speed_left = round(speed * (1 - turn / 100))
    else:
        speed_right  = round(speed * (1 + turn / 100))
        speed_left = speed
```

到整数值的舍入是低速下糟糕的精度的原因！但现在我们很高兴，操作 `opOutput_Step_Sync` 计算时不执行舍入：
```
    if turn > 0:
        speed_right  = speed
        speed_left = speed * (1 - turn / 100)
    else:
        speed_right  = speed * (1 + turn / 100)
        speed_left = speed
```

速度和步数是成比例的，我们也可以写为：
```
    if turn > 0:
        step_right  = math.copysign(1, speed) * step
        step_left = math.copysign(1, speed) * step * (1 - turn / 100)
    else:
        step_right  = math.copysign(1, speed) * step * (1 + turn / 100)
        step_left = math.copysign(1, speed) * step
```

操作 `opOutput_Step_Sync` 对于具有两个驱动轮的小车是完美的！就像是专门为它设计的一样。请通过把两个 `opOutput_Speed` 操作替换为一个 `opOutput_Step_Sync` 来提升你的远程控制程序。你将看到，它工作的更好，特别是低速下。我把我的 `move` 函数的代码修改为了：
```
    def move(self, speed: int, turn: int) -> None:
        assert self._sync_mode != ev3.SYNC, 'no unlimited operations allowed in sync_mode SYNC'
        assert isinstance(speed, int), "speed needs to be an integer value"
        assert -100 <= speed and speed <= 100, "speed needs to be in range [-100 - 100]"
        assert isinstance(turn, int), "turn needs to be an integer value"
        assert -200 <= turn and turn <= 200, "turn needs to be in range [-200 - 200]"
        if self._polarity == -1:
            speed *= -1
        if self._port_left < self._port_right:
            turn *= -1
        ops = b''.join([
            ev3.opOutput_Step_Sync,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right), # NOS
            ev3.LCX(speed),
            ev3.LCX(turn),
            ev3.LCX(0),                                  # STEPS
            ev3.LCX(0),                                  # BRAKE
            ev3.opOutput_Start,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right)  # NOS
        ])
        self.send_direct_cmd(ops)
```

一些说明：

 * 车辆遵从相同的转弯，与速度无关。这是 `opOutput_Step_Sync` 相对于使用两个 `opOutput_Speed` 操作的主要提升。
 * 如果你使用了 `opOutput_Polarity`，你将发现 ，你无法把它和 `opOutput_Step_Sync` 结合。你不得不手动完成。修改两个电机的极性很简单，只是反转参数 SPEED 即可。
 * 如果你交换了电机的连接，以使你的左边电机连接在较低端口上，它也很简单，反转 TURN。
 * 仅更改一个电机的极性非常棘手。

下表可能有助于理清你已经看到的移动。当给出以下条件时，它描述了依赖于 TURN 的运动类型：

 * 两个电机具有相同的极性，
 * 右轮连接到更低的端口，左边的更高。

| TURN | 移动 | 描述 |
|----------|--------|-------|
| 0 | 直行 | 两个电机以相同的速度朝相同的方向移动 |
| [0 - 100] | 左转 | 两个电机朝相同的方向旋转，左边的以更低的速度 |
| 100 | 绕左轮左转 | 只有右边的电机移动 |
| [100 - 200] | 向左转弯 | 两个电机朝相反的方向旋转，左边的以更低的速度旋转 |
| 200 | 向左转圈 | 两个电机朝相同的方向旋转，但速度相同 |
| [0 - -100] | 右转 | 两个电机朝相同的方向旋转，右边的以更低的速度 |
| -100 | 绕右轮右转 | 只有左边的电机移动 |
| [-100 - -200] | 向右转弯 | 两个电机朝相反的方向旋转，右边的以更低的速度旋转 |
| -200 | 向右转圈 | 两个电机朝相同的方向旋转，但速度相同 |

当端口连接不同时，左轮为较低端口，右轮为较高端口时，请反映情况。

欢迎你进一步提升远程控制工程。你可以使用游戏杆而不是你的键盘的方向键。或者你可以使用智能手机的陀螺仪传感器。但这是你的项目，我们将远程控制留在了后面（至少目前为止）。

# 具有两个驱动轮的车辆的定义明确且可预测的运动

我们仍然专注于具有两个驱动轮的车辆及其运动的准确性。远程控制是一种非常特殊的情况，如果需要，人类的头脑会监督车辆的运动并立即进行一些修正。这种情况不需要像转弯半径那样的单位。这就像驾驶汽车一样，没有必要确切知道转弯半径是如何从方向盘的位置决定的。修正是相对和直观的。机器人在没有外部控制的情况下移动，它们的算法需要知道参数的确切依赖性。我们将编写控制车辆运动的程序。我们从最难的变体开始，不存在任何校正机制。这意味着，我们需要能够预测和精确描述车辆运动的函数。我想到以下几点：

 * drive_straight(speed:int, distance: float=None)，其中
    - speed 的符号描述了方向（向前或向后）
    - speed 的绝对值描述了速度。我们更喜欢每秒 SI 单位计，但它是百分比。
    - distance 为 None 或正的，它以 SI 单位 meter 给出。如果它是 None，则运动是无限的。

 * drive_turn(speed:int, radius_turn:float, angle:float=None, right_turn:bool=False)，其中
    - speed 的符号描述了方向（向前或向后）
    - speed 的绝对值描述了速度。
    - radius_turn 是转弯的半径，单位为 `meter`。我们取两个驱动轮的中间作为我们的参考点。
    - `radius_turn` 的符号描述了转弯的方向。正值表示左转，正转向，负值代表顺时针转动。
    - `angle` 为 None 或正的，它是以度为单位的圆弧段 (90° 为四分之一圆，720° 为两个完整的圆)。如果它是 None，则表示无限移动。
    - `right_turn` 是一个用于特殊情景的标记。如果我们打开 place，属性`radius_turn` 为零并且没有符号。在这种情况下，左转是默认值，属性`right_turn` 是相反方向的标志。

正如你可能想象的那样，我们并没有走太远。我们将使用操作 `opOutput_Ready`，`opOutput_Start` 和 `opOutput_Speed_Sync`。这是说我们将不使用中断。从这个角度来看，我们可以回到第二课的知识。

## 确定车辆的尺寸

们需要车辆的某些尺寸来将上述功能的 SI 单位转换为参数 `turn` 和操作`opOutput_Speed_Sync` 的步骤。 车辆的尺寸为：

 * 驱动轮的半径：`radius_wheel`。
 * 车辆 `tread`。

使用尺子量出（对于我的车辆）：

 * radius_wheel = 0.021 m
 * tread = 0.16 m

具有更高精度的替代方案是测量车辆的运动。要获得驱动轮的半径，可以使用以下直接命令：
```
----------------------------------------------------------                   
 \ len \ cnt \ty\ hd  \op\la\no\sp\tu\ step   \br\op\la\no\                  
  ----------------------------------------------------------                 
0x|11:00|2A:00|80|00:00|B0|00|09|14|00|82:10:0E|01|A6|00|09|                 
  ----------------------------------------------------------                 
   \ 17  \ 42  \no\ 0,0 \O \0 \A \20\0 \ 3600   \1 \O \0 \A \                 
    \     \     \  \     \u \  \+ \  \  \        \  \u \  \+ \                
     \     \     \  \     \t \  \D \  \  \        \  \t \  \D \               
      \     \     \  \     \p \  \  \  \  \        \  \p \  \  \              
       \     \     \  \     \u \  \  \  \  \        \  \u \  \  \             
        \     \     \  \     \t \  \  \  \  \        \  \t \  \  \            
         \     \     \  \     \_ \  \  \  \  \        \  \_ \  \  \           
          \     \     \  \     \S \  \  \  \  \        \  \S \  \  \          
           \     \     \  \     \t \  \  \  \  \        \  \t \  \  \         
            \     \     \  \     \e \  \  \  \  \        \  \a \  \  \        
             \     \     \  \     \p \  \  \  \  \        \  \r \  \  \       
              \     \     \  \     \_ \  \  \  \  \        \  \t \  \  \      
               \     \     \  \     \S \  \  \  \  \        \  \  \  \  \     
                \     \     \  \     \y \  \  \  \  \        \  \  \  \  \    
                 \     \     \  \     \n \  \  \  \  \        \  \  \  \  \   
                  \     \     \  \     \c \  \  \  \  \        \  \  \  \  \  
                   ----------------------------------------------------------
```

拿出尺子并测量你的小车移动的距离（如果你的小车没有直行，寻找轮子的最佳结合，而不是外观，这似乎是相同的尺寸，真的是）。轮子的一个完整旋转的距离计算公式为 2 * pi * radius_wheel。3,600 度为 10 个完整旋转，因此下面的计算给出了你的轮子的 `radius_wheel`：
```
   radius_wheel = distance / (20 * pi)
```

这将我的轮子的半径修正为 radius_wheel = 0.02128 m。接下来，我们让车辆旋转，并计数 N，车辆转数。为此，我们发送直接命令：
```
----------------------------------------------------------------                   
 \ len \ cnt \ty\ hd  \op\la\no\sp\ tu     \ step   \br\op\la\no\                  
  ----------------------------------------------------------------                 
0x|13:00|2A:00|80|00:00|B0|00|09|14|82:C8:00|82:50:46|01|A6|00|09|                 
  ----------------------------------------------------------------                 
   \ 19  \ 42  \no\ 0,0 \O \0 \A \20\ 200    \ 18000  \1 \O \0 \A \                 
    \     \     \  \     \u \  \+ \  \        \        \  \u \  \+ \                
     \     \     \  \     \t \  \D \  \        \        \  \t \  \D \               
      \     \     \  \     \p \  \  \  \        \        \  \p \  \  \              
       \     \     \  \     \u \  \  \  \        \        \  \u \  \  \             
        \     \     \  \     \t \  \  \  \        \        \  \t \  \  \            
         \     \     \  \     \_ \  \  \  \        \        \  \_ \  \  \           
          \     \     \  \     \S \  \  \  \        \        \  \S \  \  \          
           \     \     \  \     \t \  \  \  \        \        \  \t \  \  \         
            \     \     \  \     \e \  \  \  \        \        \  \a \  \  \        
             \     \     \  \     \p \  \  \  \        \        \  \r \  \  \       
              \     \     \  \     \_ \  \  \  \        \        \  \t \  \  \      
               \     \     \  \     \S \  \  \  \        \        \  \  \  \  \     
                \     \     \  \     \y \  \  \  \        \        \  \  \  \  \    
                 \     \     \  \     \n \  \  \  \        \        \  \  \  \  \   
                  \     \     \  \     \c \  \  \  \        \        \  \  \  \  \  
                   ----------------------------------------------------------------
```

我计算了我的车辆 N = 15.2 完整旋转。两个轮子都旋转了 18, 000°，这是50 个完整的旋转，或者距离 50 * 2 * pi * `radius_wheel`。旋转的半径为 0.5 * tread（我们定义轮子的中间为我们的参考点）。这就是说 N * 2 * pi * 0.5 * tread = 50 * 2 * pi * radius_wheel 或者：
```
   tread = radius_wheel * 100 / N
```

这修正了 `tread` 的尺寸（在我的情况中：tread = 0.1346 m）。稍后我们将执行一些额外的运动，也许这将再次修正 `tread` 的尺寸。

## 获取参数 STEP 和 TURN 的数学转换

OK，我们知道了车的尺寸。接下来我们需要把我们的方法 `drive_straight` 和 `drive_turn` 的参数转为操作 `opOutput_Step_Sync` 的参数 STEP 和 TURN。这需要一些数学的东西。如果你对细节不感兴趣，你可以只看本小节最后的结果。但是大多数人都想知道细节，细节在这里。

给定我们的小车需要旋转的 `angle` 和 `radius_turn`。在转弯中，两个轮子移动不同的距离。外面的轮子的距离为：2 * pi * radius_wheel * STEP / 360。可以根据转弯的几何形状和车辆踏板的知识计算相同的距离：2 * pi * (radius_turn + 0.5 * tread) * angle / 360。这两个相同距离的描述给出了以下等式：
```
      STEP = angle * (radius_turn + 0.5 * tread) / radius_wheel
```

OK，第一个参数计算出来了，但是我们仍然需要计算 TURN。关键的方法来自转弯的几何学，是说，两个车轮之间的速度比 speed_right / speed_left 与外面和里面的车轮的距离比相同：
```
      speed_right / speed_left = (radius_turn + 0.5 * tread) / (radius_turn - 0.5 * tread)
```

你可能还记得，我们已经知道了两个轮子的速度：speed_right = SPEED 和 speed_left = SPEED * (1 - TURN / 100)。这给出等式：
```
      1 / (1 - TURN / 100) = (radius_turn + 0.5 * tread) / (radius_turn - 0.5 * tread)
```

转换这个等式得到：
```
      TURN  = 100 * (1 - (radius_turn - 0.5 * tread) / (radius_turn + 0.5 * tread))
```

就是这样，果给出转弯运动的尺寸 (angle, radius_turn) 和车辆的尺寸 (tread, radius_wheel)：

1. 变量 STEP 的计算方法为：
```
      STEP = angle * (radius_turn + 0.5 * tread) / radius_wheel
```

1. 变量 TURN 的计算方法为：
```
      TURN = 100 * (1 - (radius_turn - 0.5 * tread) / (radius_turn + 0.5 * tread))
```

在 Python 中，这可以写作：
```
        rad_outer = radius_turn + 0.5 * self._tread
        rad_inner = radius_turn - 0.5 * self._tread
        step = round(angle*rad_outer / self._radius_wheel)
        turn = round(100*(1 - rad_inner / rad_outer))
        if angle < 0:
            step *= -1
            turn *= -1
        if self._polarity == -1:
            speed *= -1
```

## 数学转换的控制

我们做一些合理性检查：

 * 以半径 radius_turn = 0.5 * tread 旋转得到 TURN = 100 或 TURN = -100，这是对的。
 * 以半径 radius_turn = 0 旋转得到 TURN = 200 或 TURN = -200，这也是OK 的。

现在，我们回到真正的测试。以 angle = 90° 和 radius_turn = 0.5 m 旋转。在我的情况中，由上面的计算给出 STEP = 2358 和 TURN = 23，这将得到命令：
```
----------------------------------------------------------                   
 \ len \ cnt \ty\ hd  \op\la\no\sp\tu\ step   \br\op\la\no\                  
  ----------------------------------------------------------                 
0x|11:00|2A:00|80|00:00|B0|00|09|14|17|82:36:09|01|A6|00|09|                 
  ----------------------------------------------------------                 
   \ 17  \ 42  \no\ 0,0 \O \0 \A \20\23\ 2358   \1 \O \0 \A \                 
    \     \     \  \     \u \  \+ \  \  \        \  \u \  \+ \                
     \     \     \  \     \t \  \D \  \  \        \  \t \  \D \               
      \     \     \  \     \p \  \  \  \  \        \  \p \  \  \              
       \     \     \  \     \u \  \  \  \  \        \  \u \  \  \             
        \     \     \  \     \t \  \  \  \  \        \  \t \  \  \            
         \     \     \  \     \_ \  \  \  \  \        \  \_ \  \  \           
          \     \     \  \     \S \  \  \  \  \        \  \S \  \  \          
           \     \     \  \     \t \  \  \  \  \        \  \t \  \  \         
            \     \     \  \     \e \  \  \  \  \        \  \a \  \  \        
             \     \     \  \     \p \  \  \  \  \        \  \r \  \  \       
              \     \     \  \     \_ \  \  \  \  \        \  \t \  \  \      
               \     \     \  \     \S \  \  \  \  \        \  \  \  \  \     
                \     \     \  \     \y \  \  \  \  \        \  \  \  \  \    
                 \     \     \  \     \n \  \  \  \  \        \  \  \  \  \   
                  \     \     \  \     \c \  \  \  \  \        \  \  \  \  \  
                   ----------------------------------------------------------
```

确实，我的小车几乎以半径 0.5 m 移动了完美的四分之圆。

# 增强类 TwoWheelVehicle

数学已经足够了，至少在目前，让我们的编码吧！作为我们的第一个任务，我们修改类 `TwoWheelVehicle` 的构造函数。我们添加两个尺寸 `radius_wheel` 和 `tread`，它们被确定为必要的：
```
    def __init__(
            self,
            radius_wheel: float,
            tread: float,
            protocol: str=None,
            host: str=None,
            ev3_obj: ev3.EV3=None
    ):
        super().__init__(protocol=protocol, host=host, ev3_obj=ev3_obj)
        self._radius_wheel = radius_wheel
        self._tread = tread
        self._polarity = 1
        self._port_left = ev3.PORT_D
        self._port_right = ev3.PORT_A
```

接下来我们编写方法 `_drive`，它与方法 `move` 非常接近。调用者传入参数 `speed`，`turn` 和 `step` 调用它。外部的世界用 `radius_turn` 和 `angle` 来思考，但它们必须被转为内部的参数 `turn` 和 `step`。这是说，方法 `drive_straight` 和 `drive_turn` 执行转换，然后它们调用内部的方法 `_drive`：
```
    def _drive(self, speed: int, turn: int, step: int) -> bytes:
        assert isinstance(speed, int), "speed needs to be an integer value"
        assert -100 <= speed and speed <= 100, "speed needs to be in range [-100 - 100]"
        if self._polarity == -1:
            speed *= -1
        if self._port_left < self._port_right:
            turn *= -1
        ops_ready = b''.join([
            ev3.opOutput_Ready,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right)  # NOS
        ])
        ops_start = b''.join([
            ev3.opOutput_Step_Sync,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right), # NOS
            ev3.LCX(speed),
            ev3.LCX(turn),
            ev3.LCX(step),
            ev3.LCX(0),                                  # BRAKE
            ev3.opOutput_Start,
            ev3.LCX(0),                                  # LAYER
            ev3.LCX(self._port_left + self._port_right)  # NOS
        ])
        if self._sync_mode == ev3.SYNC:
            return self.send_direct_cmd(ops_start + ops_ready)
        else:
            return self.send_direct_cmd(ops_ready + ops_start)
```

我们区分 SYNC 和 ASYNC 或 STD。在 ASYNC 或 STD 的情况中，我们在移动开始前等待，在 SYNC 的情况中直到它结束。如果你从 [ev3-python3](https://github.com/ChristophGaukel/ev3-python3) 下载了模块 `ev3_vehicle.py`，你将找不到方法 `_drive`。这是一个提示，我们将回到类 `TwoWheelVehicle` 实现中断。我们编写方法 `drive_turn` 的代码：
```
    def drive_turn(
            self,
            speed: int,
            radius_turn: float,
            angle: float=None,
            right_turn: bool=False
    ) -> None:
        assert isinstance(radius_turn, numbers.Number), "radius_turn needs to be a number"
        assert angle is None or isinstance(angle, numbers.Number), "angle needs to be a number"
        assert angle is None or angle > 0, "angle needs to be positive"
        assert isinstance(right_turn, bool), "right_turn needs to be a boolean"
        rad_right = radius_turn + 0.5 * self._tread
        rad_left = radius_turn - 0.5 * self._tread
        if radius_turn >= 0 and not right_turn:
            turn = round(100*(1 - rad_left / rad_right))
        else:
            turn = - round(100*(1 - rad_right / rad_left))
        if turn == 0:
            raise ValueError("radius_turn is too large")
        if angle is None:
            self.move(speed, turn)
        else:
            if turn > 0:
                step = round(angle*rad_right / self._radius_wheel)
            else:
                step = - round(angle*rad_left / self._radius_wheel)
            self._drive(speed, turn, step)
```

非常大的 `radius_turn` 值将导致一些问题。舍入到整数之后，它们导致直行。在这种情况下我们抛出一个错误。

方法 `drive_straight`：
```
    def drive_straight(self, speed: int, distance: float=None) -> None:
        assert distance is None or isinstance(distance, numbers.Number), \
            "distance needs to be a number"
        assert distance is None or distance > 0, \
            "distance needs to be positive"
        if distance is None:
            self.move(speed, 0)
        else:
            step = round(distance * 360 / (2 * math.pi * self._radius_wheel))
            self._drive(speed, 0, step)
```

放松，我们已经实现了一种可以预测车辆的工具。 请做一些测试！

## 了解车辆的位置和方向

抱歉，之前的一些内容我写了 *足够的数学知识*，但现在我来到了三角学。但我们处于一种真的值得努力的情境。想象一下，你驾驶你的车并经过了一系列的移动，你需要知道它的位置和方向。我们无需使用传感器就可以做到。相反，我们使用纯数学而不是魔法。

让我们做一些假设：

 * 当你创建类 TwoWheelVehicle 时你的车放置的位置将是你的坐标系统的原点 (0, 0)。
 * 此时指向正前方的方向是 x 轴的方向。
 * y 轴指向你的车的左手边。
 * 你的车的位置由 x 和 y 坐标描述。以米为单位。
 * 你的车的 `orientation` 是车的原始方向和它的实际方向的差值。以度数为单位。左转增加 `orientation`，右转减小它。

这次，我不会介绍数学。把它当作原样，或者把它作为一个谜语，你必须解决。但是请把如下的逻辑添加到你的类 `TwoWheelVehicle` 中：

 * 构造函数：
```
        self._orientation = 0.0
        self._pos_x = 0.0
        self._pos_y = 0.0
```

 * `drive_straight(speed, distance)`
```
        diff_x = distance * math.cos(math.radians(self._orientation))
        diff_y = distance * math.sin(math.radians(self._orientation))
        if speed > 0:
            self._pos_x += diff_x
            self._pos_y += diff_y
        else:
            self._pos_x -= diff_x
            self._pos_y -= diff_y
```

 * `drive_turn(speed, radius_turn, angle)`
```
            angle += 180
            angle %= 360
            angle -= 180
            fact = 2.0 * radius_turn * math.sin(math.radians(0.5 * angle))
            self._orientation += 0.5 * angle
            self._pos_x += fact * math.cos(math.radians(self._orientation))
            self._pos_y += fact * math.sin(math.radians(self._orientation))
            self._orientation += 0.5 * angle
            self._orientation += 180
            self._orientation %= 360
            self._orientation -= 180
```

# 完成类 TwoWheelVehicle

我们通过添加更多的功能来完成类 `TwoWheelVehicle`：

 * 我们添加一个方法 `rotate_to(speed: int, o: float)`，它做如下的事情：
    - 计算实际的和新的方向的距离
    - 以 `radius_turn` = 0 调用 `drive_turn` 旋转小车，以使其获得新方向。

 * 我们添加一个方法 `drive_to(self, speed: int, x: float, y: float)`，它做如下的事情：
    - 计算实际的位置和新位置的距离：
```
        diff_x = pos_x - self._pos_x
        diff_y = pos_y - self._pos_y
```
我们需要坐标和绝对值：
```
        distance = math.sqrt(diff_x**2 + diff_y**2)
```
    - 计算新位置的方向。这很棘手，你需要彻底了解三角学，因为你必须使用 `atan`，`tan` 的反函数。我给你一个提示：
```
        if abs(diff_x) > abs(diff_y):
            direction = math.degrees(math.atan(diff_y/diff_x))
        else:
            fract = diff_x / diff_y
            sign = math.copysign(1.0, fract)
            direction = sign * 90 - math.degrees(math.atan(fract))
        if diff_x < 0:
            direction += 180
        direction %= 360
```
    - 调用 `rotate_to`，以使 `orientation` 指向 `direction`。
    - 调用 `drive_straight` 移动小车到新的位置。

我们做一些测试：

 * 我们把小车发送到一些循环行程中，并编码一系列 `drive_to`，最终在 position = (0,0) 结束。我们最后添加一个以 orientation = 0 对 `rotate_to` 的调用。请评估，小车是否真的回到了它最初的位置和方向。
 * 我们给循环行程添加一些 `drive_turn` 并评估，`drive_turn` 的错误是大于还是小于 `drive_to` 的。

以大 `radius_turn` 旋转将有一个糟糕的精度。这也是舍入的结果。在这种情况中，将 TURN 舍入为整数值创造了错误。

如果你从 [ev3-python3](https://github.com/ChristophGaukel/ev3-python3) 下载了 `ev3_vehicle` 模块，当你调用它的方法 `drive_straight`，`drive_turn`，`rotate_to` 或 `drive_to` 时，你最后需要添加一个对 `stop` 的方法调用！

# 异步和同步运动
让我们仔细看看我们的车辆驾驶情况。这是我的循环行程的程序：
```
#!/usr/bin/env python3

import ev3, ev3_vehicle

my_vehicle = ev3_vehicle.TwoWheelVehicle(0.02128, 0.1346, protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_vehicle.verbosity = 1
speed = 25
my_vehicle.drive_straight(speed, 0.05)
my_vehicle.drive_turn(speed, -0.07, 65)
my_vehicle.drive_straight(speed, 0.35)
my_vehicle.drive_turn(speed, 0.20, 140)
my_vehicle.drive_straight(speed, 0.15)
my_vehicle.drive_turn(speed, -1.10, 55)
my_vehicle.drive_turn(speed, 0.35, 160)
my_vehicle.drive_to(speed, 0.0, 0.0)
my_vehicle.rotate_to(speed, 0.0)
```

这个程序的输出：
```
15:42:05.989592 Sent 0x|14:00|2A:00|80|00:00|AA:00:09:B0:00:09:19:00:82:87:00:00:A6:00:09|
15:42:05.990443 Sent 0x|15:00|2B:00|80|00:00|AA:00:09:B0:00:09:19:81:9E:82:A3:01:00:A6:00:09|
15:42:05.990925 Sent 0x|14:00|2C:00|80|00:00|AA:00:09:B0:00:09:19:00:82:AE:03:00:A6:00:09|
15:42:05.991453 Sent 0x|15:00|2D:00|80|00:00|AA:00:09:B0:00:09:19:81:32:82:DF:06:00:A6:00:09|
15:42:05.991882 Sent 0x|14:00|2E:00|80|00:00|AA:00:09:B0:00:09:19:00:82:94:01:00:A6:00:09|
15:42:05.992330 Sent 0x|14:00|2F:00|80|00:00|AA:00:09:B0:00:09:19:34:82:C9:0B:00:A6:00:09|
15:42:05.992766 Sent 0x|15:00|30:00|80|00:00|AA:00:09:B0:00:09:19:81:20:82:42:0C:00:A6:00:09|
15:42:05.993306 Sent 0x|16:00|31:00|80|00:00|AA:00:09:B0:00:09:19:82:38:FF:82:D0:01:00:A6:00:09|
15:42:05.993714 Sent 0x|14:00|32:00|80|00:00|AA:00:09:B0:00:09:19:00:82:3F:03:00:A6:00:09|
15:42:05.994202 Sent 0x|15:00|33:00|80|00:00|AA:00:09:B0:00:09:19:82:38:FF:81:69:00:A6:00:09|
```

在五毫秒内这个程序给 EV3 设备发送了所有的直接命令，它们被放入队列并等待执行。这个代码很简单，但是会阻塞 EV3 设备直到开始执行最后的命令。请遵循代码并反映出属性 pos_x，pos_y 和 orientation 的值，以及它们是否与我们车辆的实际位置和方向相对应。

这是异步行为！ 程序和 EV3 设备在不同的时间尺度上运行。

现在我们改为同步模式并比较行为：
```
#!/usr/bin/env python3

import ev3, ev3_vehicle

my_vehicle = ev3_vehicle.TwoWheelVehicle(0.02128, 0.1346, protocol=ev3.BLUETOOTH, host='00:16:53:42:2B:99')
my_vehicle.verbosity = 1
speed = 25
my_vehicle.sync_mode = ev3.SYNC
my_vehicle.drive_straight(speed, 0.05)
my_vehicle.drive_turn(speed, -0.07, 65)
my_vehicle.drive_straight(speed, 0.35)
my_vehicle.drive_turn(speed, 0.20, 140)
my_vehicle.drive_straight(speed, 0.15)
my_vehicle.drive_turn(speed, -1.10, 55)
my_vehicle.drive_turn(speed, 0.35, 160)
my_vehicle.drive_to(speed, 0.0, 0.0)
my_vehicle.rotate_to(speed, 0.0)
```

此版本产生以下输出：
```
15:46:19.859532 Sent 0x|14:00|2A:00|00|00:00|B0:00:09:19:00:82:87:00:00:A6:00:09:AA:00:09|
15:46:20.307045 Recv 0x|03:00|2A:00|02|
15:46:20.307760 Sent 0x|15:00|2B:00|00|00:00|B0:00:09:19:81:9E:82:A3:01:00:A6:00:09:AA:00:09|
15:46:22.100007 Recv 0x|03:00|2B:00|02|
15:46:22.100612 Sent 0x|14:00|2C:00|00|00:00|B0:00:09:19:00:82:AE:03:00:A6:00:09:AA:00:09|
15:46:24.597018 Recv 0x|03:00|2C:00|02|
15:46:24.597646 Sent 0x|15:00|2D:00|00|00:00|B0:00:09:19:81:32:82:DF:06:00:A6:00:09:AA:00:09|
15:46:29.141999 Recv 0x|03:00|2D:00|02|
15:46:29.142609 Sent 0x|14:00|2E:00|00|00:00|B0:00:09:19:00:82:94:01:00:A6:00:09:AA:00:09|
15:46:30.221994 Recv 0x|03:00|2E:00|02|
15:46:30.222626 Sent 0x|14:00|2F:00|00|00:00|B0:00:09:19:34:82:C9:0B:00:A6:00:09:AA:00:09|
15:46:44.779922 Recv 0x|03:00|2F:00|02|
15:46:44.780364 Sent 0x|15:00|30:00|00|00:00|B0:00:09:19:81:20:82:42:0C:00:A6:00:09:AA:00:09|
15:46:46.938901 Recv 0x|03:00|30:00|02|
15:46:46.939701 Sent 0x|16:00|31:00|00|00:00|B0:00:09:19:82:38:FF:82:D0:01:00:A6:00:09:AA:00:09|
15:46:55.695901 Recv 0x|03:00|31:00|02|
15:46:55.696508 Sent 0x|14:00|32:00|00|00:00|B0:00:09:19:00:82:3F:03:00:A6:00:09:AA:00:09|
15:47:05.061856 Recv 0x|03:00|32:00|02|
15:47:05.062504 Sent 0x|15:00|33:00|00|00:00|B0:00:09:19:82:38:FF:81:69:00:A6:00:09:AA:00:09|
15:47:05.097692 Recv 0x|03:00|33:00|02|
```

车辆的运动是相同的，但现在程序发送一个直接命令并等待它完成，然后它发送下一个。sync_mode = SYNC 如它设计那样工作，它同步程序和EV3 设备的时间尺度。另一个好处是，它控制每个直接命令的成功并直接作出反应，如果出现意外情况。

这两个版本（异步和同步）都会阻塞 EV3 设备。

# 结论

类 `TwoWheelVehicle` 通过两个驱动轮控制小车的运动。如果我们知道课程的几何形状，我们就可以编写一个程序，通过它来驱动我们的车辆。

我们已经看到了同步和异步模式之间的区别。目前，我们更倾向于 SYNC。但我们的目标是一个解决方案，它可以同步驱动车辆并且不会阻塞EV3 设备。这将打开多任务的大门。

我的 `EV3TwoWheelVehicle` 类实际上有以下状态：
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
     |  __init__(self, radius_wheel:float, tread:float, protocol:str=None, host:str=None, ev3_obj:ev3.EV3=None)
     |      Establish a connection to a LEGO EV3 device
     |      
     |      Arguments:
     |      radius_wheel: radius of the wheels im meter
     |      tread: the vehicles tread in meter
     |      
     |      Keyword Arguments (either protocol and host or ev3_obj):
     |      protocol
     |        BLUETOOTH == 'Bluetooth'
     |        USB == 'Usb'
     |        WIFI == 'Wifi'
     |      host: mac-address of the LEGO EV3 (f.i. '00:16:53:42:2B:99')
     |      ev3_obj: an existing EV3 object (its connections will be used)
     |  
     |  drive_straight(self, speed:int, distance:float=None) -> None
     |      Drive the vehicle straight forward or backward.
     |      
     |      Attributes:
     |      speed: in percent [-100 - 100] (direction depends on its sign)
     |          positive sign: forwards
     |          negative sign: backwards
     |      
     |      Keyword Attributes:
     |      distance: in meter, needs to be positive
     |                if None, unlimited movement
     |  
     |  drive_to(self, speed:int, pos_x:float, pos_y:float) -> None
     |      Drive the vehicle to the given position.
     |      
     |      Attributes:
     |      speed: in percent [-100 - 100] (direction depends on its sign)
     |          positive sign: forwards
     |          negative sign: backwards
     |      x: x-coordinate of target position
     |      y: y-coordinate of target position
     |  
     |  drive_turn(self, speed:int, radius_turn:float, angle:float=None, right_turn:bool=False) -> None
     |      Drive the vehicle a turn with given radius.
     |      
     |      Attributes:
     |      speed: in percent [-100 - 100] (direction depends on its sign)
     |          positive sign: forwards
     |          negative sign: backwards
     |      radius_turn: in meter
     |          positive sign: turn to the left side
     |          negative sign: turn to the right side
     |      
     |      Keyword Attributes:
     |      angle: absolute angle (needs to be positive)
     |             if None, unlimited movement
     |      right_turn: flag of turn right (only in case of radius_turn == 0)
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
     |  rotate_to(self, speed:int, orientation:float) -> None
     |      Rotate the vehicle to the given orientation.
     |      Chooses the direction with the smaller movement.
     |      
     |      Attributes:
     |      speed: in percent [-100 - 100] (direction depends on its sign)
     |      orientation: in degrees [-180 - 180]
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
     |  orientation
     |      actual orientation of the vehicle in degree, range [-180 - 180]
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
     |  pos_x
     |      actual x-component of the position in meter
     |  
     |  pos_y
     |      actual y-component of the position in meter
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

我希望，你感觉到，我们现在正在做实际的事情。 对我来说，这是一个亮点。 只有在极少数情况下，人们才能用这么少的努力获得这么多。

[原文](http://ev3directcommands.blogspot.com/2016/01/ev3-direct-commands-lesson-04-pre.html)
