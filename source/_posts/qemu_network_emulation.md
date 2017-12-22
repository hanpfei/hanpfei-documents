---
title: QEMU 网络虚拟化
date: 2017-12-22 13:05:49
categories: 虚拟化
tags:
- 虚拟化
- Android开发
---

对于模拟器而言，让模拟器中的客户 Android 系统内的进程连接外部网络，与通过 adb forward 的方式，让外部网络的程序，连接模拟器的客户 Android 系统内的服务相比，网络拓扑结构有着非常大的不同。这种拓扑结构的差异，对模拟器内的客户 Android 系统中的进程与外部应用进程之间的网络连接的许多方面都有着非常大的影响，如连接的稳定性，性能等等。
<!--more-->
# 模拟器连接外部网络时的情况
首先来看模拟器的客户 Android 系统内部的进程与外部服务之间建立 TCP 连接的情况。

首先启动一个服务，让它监听 TCP 的 18960 端口，以接受连接。然后在模拟器的客户 Android 系统内启动一个进程，连接前面启动的 TCP 服务。查看此时主机上网络端口的打开情况，将看到如下的情形：

```
hanpfei0306@ThundeRobot:~/emu-2.4-release/external/qemu$ lsof -i | grep 18960
qemu-syst 13159 hanpfei0306   47u  IPv4 13995747      0t0  TCP 10.240.209.153:44658->10.240.209.153:18960 (ESTABLISHED)
EventServ 24086 hanpfei0306    4u  IPv4 13991611      0t0  TCP *:18960 (LISTEN)
EventServ 24086 hanpfei0306   10u  IPv4 13991620      0t0  TCP 10.240.209.153:18960->10.240.209.153:44658 (ESTABLISHED)
```

可以看到，qemu 进程与客户 Android 系统中的进程的连接目标建立了一条 TCP 连接。不难想象，在这种场景下，网络拓扑结构将像下面这样：

模拟器内客户 Android 系统内的进程 <------->  emulator 进程 <-------> 连接目标（事件服务器）

![186c66f2e78a6e286fdc75184fecf686.jpg](https://www.wolfcstech.com/images/1315506-3e5111e51506e566.jpg)

# 外部应用通过端口转发连入客户 Android 系统
通过 `adb forward` 做端口转发，将宿主机上的某个端口映射到客户 Android 系统内的某个端口上，如将宿主机上的 TCP 10977 端口，映射到客户 Android系统内的 TCP 10977 端口上。此时我们查看宿主机上与 10977 端口相关的网络连接，将如下面所示这样：

```
hanpfei0306@ThundeRobot:~/emu-2.4-release/external/qemu$ lsof -i | grep 10977
adb        3041 hanpfei0306   29u  IPv4 13726863      0t0  TCP localhost:10977 (LISTEN)
adb        3041 hanpfei0306   31u  IPv4 14035891      0t0  TCP localhost:10977->10.240.209.153:51168 (ESTABLISHED)
telnet    24980 hanpfei0306    3u  IPv4 14041367      0t0  TCP localhost:51168->localhost:10977 (ESTABLISHED)
hanpfei0306@ThundeRobot:~/emu-2.4-release/external/qemu$ lsof -i | grep adb | grep 10977
adb        3041 hanpfei0306   29u  IPv4 13726863      0t0  TCP localhost:10977 (LISTEN)
adb        3041 hanpfei0306   31u  IPv4 14035891      0t0  TCP localhost:10977->10.240.209.153:51168 (ESTABLISHED)
```

可以看到，端口转发实际上是让 adb 起一个 server socket，如上图，adb 作为外部进程和客户 Android 系统内的 server 进程通信的中转桥梁。

再通过 `adb shell` 查看客户 Android 系统内，与 10977 端口相关的网络连接。不难想象，在这种场景下，网络拓扑结构将像下面这样：

模拟器内客户 Android 系统内的进程 <-------> 模拟器内客户 Android 系统内的 adbd <-------> emulator 进程 <-------> 宿主系统中的 adb 守护进程 <-------> 连接目标（事件服务器）

![0e18daa8bfa88b1c827a344de7b43576.jpg](https://www.wolfcstech.com/images/1315506-2e7cd62090753629.jpg)

外部应用通过端口转发连入客户 Android 系统内进程的情形下，整个网络连接链路要不内部连接外部长得多，这么长的链路，难免会使这种连接的稳定性大为降低。端口转发还有一个比较严重的问题，无论外部应用连接的目标模拟器是哪一个，这些连接都必定要先经过 adb 守护进程，adb 守护进程难免常常会成为一个性能瓶颈；此外，连接的发起应用，对于连接状态的感知也将变得没有那么灵敏，如客户 Android 系统内连接的目标应用进程已经死掉，这种状态要通过客户 Android 系统内的 adbd，模拟器进程本身，以及 adb 守护进程三个环节才能传达给发起连接的应用。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
