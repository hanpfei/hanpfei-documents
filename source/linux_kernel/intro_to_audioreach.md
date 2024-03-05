---
title: 高通 AudioReach 框架简介
date: 2024-03-02 11:08:29
categories: Linux 内核
tags:
- Linux 内核
---

AudioReach 是高通 SoC DSP 的信号处理框架，它本身运行于 DSP。AudioReach 是高通下一代音频 SDK 的必要组成部分，并将被部署在后续的高通芯片中。在 Linux/Android 端，创建 ASoC 驱动程序对 AudioReach 框架进行配置。AudioReach 及其 ASoC 驱动程序中，利用 ASoC Topology 技术将音频处理组件拓扑结构图加载进 DSP 中，随后拓扑结构图由 AudioReach 内的 APM（Audio Processing Manager，音频处理管理器）服务管理，来 prepare/start/stop。

AudioReach 及其 ASoC 驱动程序的简化高层块图如下：
```
___________________________________________________________
|                 CPU (Application Processor)               |
|  +---------+          +---------+         +---------+     |
|  |  q6apm  |          |  q6apm  |         | q6afe   |     |
|  |   dais  | <------> |         | <-----> | bedais  |     |
|  +---------+          +---------+         +---------+     |
|                            ^  ^                           |
|                            |  |           +---------+     |
|  +---------+               v  +---------->|topology |     |
|  | q6prm   |          +---------+         |         |     |
|  |         |<-------->|   GPR   |         +---------+     |
|  +---------+          +---------+                         |
|                            ^                              |
|____________________________|______________________________|
                             |  
                             | RPMSG (IPC over GLINK)              
 ____________________________|______________________________
|                            |                              |
|    +-----------------------+                              |
|    |                       |                              |
|    v                       v              q6 (Audio DSP)  |
|+-----+    +----------------------------------+            |
|| PRM |    | APM (Audio Processing Manager)   |            |
|+-----+    |  . Graph Management              |            |  
|           |  . Command Handing               |            |  
|           |  . Event Management              |            |  
|           |  ...                             |            |  
|           +----------------------------------+            |  
|                            ^                              |
|____________________________|______________________________|
                             |  
                             |   LPASS AIF
 ____________________________|______________________________
|                            |            Audio I/O         |
|                            v                              |
|   +--------------------------------------------------+    |
|    |                Audio devices                     |   |
|    | CODEC | HDMI-TX | PCM  | SLIMBUS | I2S |MI2S |...|   |
|    |                                                  |   |
|    +--------------------------------------------------+   |
|___________________________________________________________|
```

AudioReach 具有子图、容器和模块结构。每个子图可以有 N 个容器，每个容器可以有 N 个模块，且它们之间的连接可以是线性的或非线性的。一个音频功能可以被实现为一个或多个连接的子图。模块之间还存在控制/事件路径，可以在构建图时将其连接起来，以实现模块之间的各种控制机制。子图、容器和模块的这些概念在 ASoC 拓扑中表示。

这是一个简单的 I2S 图，在单个子图 (1) 中包含一个写入共享内存和一个音量控制模块，其中包含一个容器 (1) 和 5 个模块。
```
____________________________________________________________
 |                        Sub-Graph [1]                       |
 |  _______________________________________________________   |
 | |                       Container [1]                   |  |
 | | [WR_SH] -> [PCM DEC] -> [PCM CONV] -> [VOL]-> [I2S-EP]|  |
 | |_______________________________________________________|  |
 |____________________________________________________________|
```

现在该图被分成两个子图以实现 dpcm，如下所示：
```
 ________________________________________________    _________________
|                Sub-Graph [1]                   |  |  Sub-Graph [2]  |
|  ____________________________________________  |  |  _____________  |
| |              Container [1]                 | |  | |Container [2]| |
| | [WR_SH] -> [PCM DEC] -> [PCM CONV] -> [VOL]| |  | |   [I2S-EP]  | |
| |____________________________________________| |  | |_____________| |
|________________________________________________|  |_________________|

                                                      _________________
                                                    |  Sub-Graph [3]  |
                                                    |  _____________  |
                                                    | |Container [3]| |
                                                    | |  [DMA-EP]   | |
                                                    | |_____________| |
                                                    |_________________|

```

在最新的 Linux 内核代码 (v6.6.7) 中 AudioReach 的 Linux ASoC 驱动代码 sound/soc/qcom/qdsp6。AudioReach 的 Linux ASoC 驱动仍然在不断的发展完善中，如 2023 的 patch 添加了对 compress-offload 接口的支持。

从实现上，AudioReach 的 Linux ASoC 驱动是 AudioReach 能力的代理，或者适配器，它通过核间通信机制和 DSP 上运行的 AudioReach 通信，对于 Linux 内核，它实现 ASoC 的 PCM 和 compress 接口，向用户空间提供访问音频能力必不可少设备文件等接口。

除了运行于 DSP 的 AudioReach 和它的 Linux ASoC 驱动程序，拓扑配置是整个子系统运行的另一个组成部分，Audioreach 的拓扑配置文件可以参考 [Audioreach Topology](https://git.codelinaro.org/neil.armstrong/audioreach-topology)。

**参考文档**：
[ASoC: qcom: Add AudioReach support](https://lwn.net/Articles/858668/)
[ASoC: qcom: Add AudioReach support](https://lore.kernel.org/lkml/20210714153039.28373-12-srinivas.kandagatla@linaro.org/T/)
[Forums - Audio development platform with AudioReach support](https://developer.qualcomm.com/forum/qdn-forums/hardware/hardware-development-kits-hdks/71404)
[ASoC: qcom: audioreach: add compress offload support](https://lore.kernel.org/lkml/20230609145407.18774-1-srinivas.kandagatla@linaro.org/)
