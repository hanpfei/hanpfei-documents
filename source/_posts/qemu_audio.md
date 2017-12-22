---
title: QEMU 中音频模拟如何工作
date: 2017-12-22 19:05:49
categories: 虚拟化
tags:
- 虚拟化
- Android开发
- 翻译
---

事情有点棘手，但这里有一个粗略的描述：
<!--more-->
  QEMUSoundCard：建模一个给定的模拟的声卡
  SWVoiceOut：建模一个来自 QEMUSoundCard 的音频输出
  SWVoiceIn：建模一个来自 QEMUSoundCard 的音频输入

  HWVoiceOut：建模一个主机端的音频输出（后端）
  HWVoiceIn：建模一个主机端的音频输入（后端）

每个声音在采样大小，字节序，速率等方面都可以有自己的设置。

对于一个给定声卡的模拟典型的做法如下：

  1/ 创建一个 QEMUSoundCard 对象，然后用 `AUD_register_card()` 注册它
  2/ 对于每个模拟的输出，调用 `AUD_open_out()` 创建一个 `SWVoiceOut` 对象
  3/ 对于每个模拟的输入，调用 `AUD_open_in()` 创建一个 `SWVoiceIn` 对象

  注意你必须给 `AUD_open_out()` 和 `AUD_open_in()` 传递一个回调函数；后面有更多相关内容。

  每个 SWVoiceOut 与一个 HWVoiceOut 关联，每个 `SWVoiceIn` 与一个 `HWVoiceIn` 关联。

  然而你可以让多个 `SWVoiceOut` 与相同的 `HWVoiceOut` 关联（相同的事情也发生在 SWVoiceIn/HWVoiceIn 中）。

声音播放细节
=======================

每个 HWVoiceOut 也有以下这些：

  - 一个固定大小的立体声采样的循环缓冲区（用于立体声）。其采样的格式为 floats 或 int64_t（依赖于构建配置）。

  - 一个 `samples` 字段给出了（固定的）立体声缓冲区中采样对的个数。

  - 一个目标转换函数，称为 `clip()`，用于从立体声缓冲区中读取并写入一个平台特有的声音缓冲区（比如，Windows 上 WinWave 管理的缓冲区）。

  - 一个 `rpos`，它循环缓冲区的偏移，指明了从立体声缓冲区中的什么位置读取下一个采样数据，以通过 `clip` 用于下一次转换。

```
            |<----------------- samples ----------------------->|

            |                                                   |

            |       rpos                                        |
                    |
            |_______v___________________________________________|
            |       |                                           |
            |       |                                           |
            |_______|___________________________________________|
```

  - 一个 `run_out` 方法，每次调用以告知输出后端从立体声缓冲区发送采样数据到主机的声卡/声音服务。这个方法也应该修改 `rpos` 并返回 `played` 采样的数据的个数。下面有关于这个过程的一个更详细的描述。

  - 一个 `write` 方法回调用于从一个 `SWVoiceOut` 写入一个模拟的声音采样数据的缓冲区到立体声缓冲区。当前所有后端简单地调用通用函数 `audio_pcm_sw_write()` 来实现它。

    根据 malc，音频子系统的最初的作者，如果可以的话，这用于允许一个后端使用平台特有的函数来做相同的事情。

    (简单地说，所有的输入后端具有一个 `read` 方法，它简单地调用 `audio_pcm_sw_read`。)

每个 `SWVoiceOut` 有以下这些：

  - 一个 `conv()` 函数用于从模拟的声卡读取声音采样数据，并复制/混合它们到对应的 HWVoiceOut 的立体声缓冲区。

  - 一个 `total_hw_samples_mixed` 对应于已经被混合到目标 `HWVoiceOut` 立体声缓冲区的采样的个数（从 HWVoiceOut 的 `rpos` 偏移开始）。注意：这是 HWVoiceOut 立体声缓冲区中的采样的计数，而不是模拟的硬件声音采样，它们可能具有不同的属性（频率，大小，尾端）。

```
                                         ______________
                                        |              |
                                        |  SWVoiceOut2 |
                                        |______________|
                  ______________           |
                 |              |          |
                 |  SWVoiceOut1 |          |     thsm<N> := total_hw_samples_mixed
                 |______________|          |                for SWVoiceOut<N>
                           |               |
                           |               |
                    |<-----|------------thsm2-->|
                    |      |                    |
                    |<---thsm1-------->|        |
             _______|__________________v________|_______________ 
            |       |111111111111111111|        v               |
            |       |222222222222222222222222222|               |
            |_______|___________________________________________|
                    ^
                    |         HWVoiceOut stereo buffer
                    rpos
```

  - 一个 `ratio` 值，它是目标 HWVoiceOut 的频率与 SWVoiceOut 的频率的比值，乘以 (1 << 32) 的一个 64 位的整数。

    因此，如果 HWVoiceOut 的频率为 44kHz，而 SWVoiceOut 的频率为 11kHz，则 ratio 将是 (44/11*(1 << 32)) = 0x4_0000_0000。

  - 一个当 SWVoiceOut 创建时模拟的硬件提供的回调。这个函数用于混合 SWVoiceOut 的采样到目的 HWVoiceOut 立体声缓冲区（它也必须执行频率插值，音量调整，等等。。。）。

    这个回调通常调用音频子系统中的另一个辅助函数 (`AUD_write()`) 来从模拟的硬件采样缓冲区混合/调整音量。

这是一个小的图形，更好地解释了它：
```
   SWVoiceOut:  模拟的硬件声音缓冲区:
          |
          |   (通过 AUD_write() 混合，该函数在用户提供的回到中调用，
          |    回调本身在每次音频始终滴答时调用 ).
          |
          v
   HWVoiceOut: 立体声采样循环缓冲区
          |
          |   (通过 HWVoiceOut 的 `clip` 函数发送，它在 `run_out` 方法中调用，
          |    也在每次音频时钟滴答时调用)
          |
          v
   后端特有的声音缓冲区
```

audio/audio.c 中的 `audio_timer()` 函数被周期性地调用，且被用作一个执行声音缓冲区的传输和混合的脉冲。对于音频输出语音更具体的如下：

- 对于每个 HWVoiceOut，找到活跃的 SWVoiceOut 的个数，以及已经被写入缓冲区的 `total_hw_samples_mixed` 的最小个数。我们将这个值称为立体声缓冲区中 “live” 采样的数量。

- 如果 `live` 为 0，如果需要，调用每个活跃的 SWVoiceOut 的回调来填充立体声缓冲区，然后退出。

- 否则，调用 `HWVoiceOut` 对象的 `run_out` 方法。这将改变 `rpos` 的值，并返回播放的采样的个数。然后所有活跃的 `SWVoiceOuts` 的 `total_hw_samples_mixed` 减去 `played`，回调被调用以重新填充立体声缓冲区。

重要的是要注意SWVoiceOut回调：

- 接收一个 `free` 参数，其是可以被发送给硬件立体声缓冲区的立体声声音采样的个数（在比率调整前，比如，不是 SWVoiceOut 模拟的硬件声音缓冲区中声音采样的个数）。

- 必须调用 `AUD_write(sw, buff, count)`，其中 `buff` 指向模拟的声音采样，且它们的 `count`，必须 <= `free` 参数。

- `AUD_write()` 的实现将调用目标 `HWVoiceOut` 的 `write` 方法，这反过来调用函数 `audio_pcm_sw_write()`，其在混合转换到目标立体声缓冲区之前执行标准的 比率/音量 调整。它也增加 `SWVoiceOut` 的 `total_hw_samples_mixed` 值。

- `audio_pcm_sw_write()` 返回已经被混合入立体声缓冲区的声音采样的 *字节* 数，` AUD_write()` 也是如此。

因此，最后我们有伪代码：
```
    every sound timer ticks:
      for hw in list_HWVoiceOut:
         live = MIN([sw.total_hw_samples_mixed for sw in hw.list_SWVoiceOut ])
         if live > 0:
            played = hw.run_out(live)
            for sw in hw.list_SWVoiceOut:
                sw.total_hw_samples_mixed -= played

        for sw in hw.list_SWVoiceOut:
            free = hw.samples - sw.total_hw_samples_mixed
            if free > 0:
                sw.callback(sw, free)
```

录音详情
========================

情况类似但顺序相反，比如，HWVoiceIn 在它的立体声缓冲区中获取声音采样数据，且 SWVoiceIn 对象必须尽快消费它们。

原文 $QEMU/android/docs/AUDIO.TXT

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
