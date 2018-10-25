---
title: LEGO EV3 中执行 VSCode Python 代码过程分析
date: 2018-10-25 20:05:49
categories: ROS
tags:
- ROS
---

镜像为 ev3dev。

通过 SSH 连接 LEGO EV3 设备，默认密码为 `maker`：
```
$ ssh robot@ev3dev.local
Password: 
Linux ev3dev 4.14.61-ev3dev-2.2.2-ev3 #1 PREEMPT Mon Aug 6 14:22:31 CDT 2018 armv5tejl
             _____     _
   _____   _|___ /  __| | _____   __
  / _ \ \ / / |_ \ / _` |/ _ \ \ / /
 |  __/\ V / ___) | (_| |  __/\ V /
  \___| \_/ |____/ \__,_|\___| \_/

Debian stretch on LEGO MINDSTORMS EV3!
Last login: Wed Oct 24 07:42:33 2018 from 10.42.0.1
```
<!--more-->
登录之后，查看设备系统中运行的所有进程，剔除所有的内核进程，用户进程主要包括如下这些：
```
robot@ev3dev:~$ ps -alx
F   UID   PID  PPID PRI  NI    VSZ   RSS WCHAN  STAT TTY        TIME COMMAND
4     0     1     0  20   0  27960  5632 -      Ss   ?          0:10 /sbin/init
4     0   148     1  20   0  39620  7484 -      Ss   ?          0:05 /lib/systemd/systemd-journald
4     0   193     1  20   0  13660  2692 -      Ss   ?          0:03 /lib/systemd/systemd-udevd
4   107   358     1  20   0   6360  2964 -      Ss   ?          0:01 avahi-daemon: running [ev3dev.local]
4   106   359     1  20   0   6476  3268 -      Ss   ?          0:02 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
1   107   360   358  20   0   6360  1444 -      S    ?          0:00 avahi-daemon: chroot helper
4     0   362     1  20   0   7340  4224 -      Ss   ?          0:00 /lib/systemd/systemd-logind
4     0   365     1  20   0  17184  4600 -      Ssl  ?          0:04 /usr/sbin/brickd
4  1000   378     1  20   0  33756  4200 poll_s Ssl+ tty5       0:00 /usr/bin/conrun-server
1     0   393     1  20   0   1768   544 -      Ss   ?          0:00 /usr/sbin/ldattach 29 /dev/tty_ev3-ports:in1
4     0   427     1  20   0  10988  4588 -      Ss   ?          0:03 /usr/sbin/connmand -n
4     0   429     1  20   0   7344  3300 -      Ss   ?          0:00 /usr/lib/bluetooth/bluetoothd
4   103   510     1  20   0   8192  4444 -      Ss   ?          0:00 /lib/systemd/systemd-resolved
4     0   512     1  20   0   9900  3612 -      Ss   ?          0:00 /sbin/wpa_supplicant -u -s -O /run/wpa_supplicant
4     0   514     1  20   0  51356 11772 -      Ssl+ tty1       0:34 /usr/sbin/brickman
4     0   518     1  20   0  10268  5188 -      Ss   ?          0:00 /usr/sbin/sshd -D
5   105   535     1  20   0   8972  3856 -      Ssl  ?          0:01 /usr/sbin/ntpd -p /var/run/ntpd.pid -g -u 105:108
4     0   702   518  20   0  11524  5616 -      Ss   ?          0:00 sshd: robot [priv]
4  1000   709     1  20   0   9644  5564 SyS_ep Ss   ?          0:00 /lib/systemd/systemd --user
5  1000   712   709  20   0  29564  2540 -      S    ?          0:00 (sd-pam)
5  1000   719   702  20   0  11524  3680 -      R    ?          0:00 sshd: robot@pts/0
0  1000   722   719  20   0   5264  3008 wait   Ss   pts/0      0:00 -bash
0  1000   842   722  20   0   7200  2296 -      R+   pts/0      0:00 ps -alx
```

总共有 23 个用户进程。

以 ev3dev 官方提供的在 VSCode 中为 EV3 开发 Python 代码的示例工程 [vscode-hello-python](https://github.com/ev3dev/vscode-hello-python) 为例，在 VSCode 中以打开目录的方式打开下载的这个工程的源码目录，按下 F5 键在 EV3 中运行代码，此时查看 EV3 中进程的运行情况，所有运行的进程如下：
```
robot@ev3dev:~$ ps -alx
F   UID   PID  PPID PRI  NI    VSZ   RSS WCHAN  STAT TTY        TIME COMMAND
4     0     1     0  20   0  27960  5192 -      Ss   ?          0:10 /sbin/init
4     0   148     1  20   0  39620  6604 -      Ss   ?          0:05 /lib/systemd/systemd-journald
4     0   193     1  20   0  13660  2676 -      Ss   ?          0:03 /lib/systemd/systemd-udevd
4   107   358     1  20   0   6360  2772 -      Ss   ?          0:01 avahi-daemon: running [ev3dev.local]
4   106   359     1  20   0   6476  3028 -      Ss   ?          0:02 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
1   107   360   358  20   0   6360  1332 -      S    ?          0:00 avahi-daemon: chroot helper
4     0   362     1  20   0   7340  3672 -      Ss   ?          0:00 /lib/systemd/systemd-logind
4     0   365     1  20   0  17212  4444 -      Ssl  ?          0:05 /usr/sbin/brickd
4  1000   378     1  20   0  33756  3896 poll_s Ssl  tty5       0:00 /usr/bin/conrun-server
1     0   393     1  20   0   1768   492 -      Ss   ?          0:00 /usr/sbin/ldattach 29 /dev/tty_ev3-ports:in1
4     0   427     1  20   0  10988  3592 -      Ss   ?          0:03 /usr/sbin/connmand -n
4     0   429     1  20   0   7344  3084 -      Ss   ?          0:00 /usr/lib/bluetooth/bluetoothd
4   103   510     1  20   0   8192  3752 -      Ss   ?          0:00 /lib/systemd/systemd-resolved
4     0   512     1  20   0   9900  2712 -      Ss   ?          0:00 /sbin/wpa_supplicant -u -s -O /run/wpa_supplicant
4     0   514     1  20   0  51356  8764 -      Ssl+ tty1       0:38 /usr/sbin/brickman
4     0   518     1  20   0  10268  5016 -      Ss   ?          0:00 /usr/sbin/sshd -D
5   105   535     1  20   0   8972  3256 -      Ssl  ?          0:01 /usr/sbin/ntpd -p /var/run/ntpd.pid -g -u 105:108
4     0   702   518  20   0  11524  5616 -      Ss   ?          0:00 sshd: robot [priv]
4  1000   709     1  20   0   9644  5144 SyS_ep Ss   ?          0:00 /lib/systemd/systemd --user
5  1000   712   709  20   0  29564  2412 -      S    ?          0:00 (sd-pam)
5  1000   719   702  20   0  11524  3692 -      S    ?          0:04 sshd: robot@pts/0
0  1000   722   719  20   0   5264  3008 wait   Ss   pts/0      0:00 -bash
4     0   857   518  20   0  11524  5568 -      Ss   ?          0:01 sshd: robot [priv]
5  1000   868   857  20   0  11524  4800 -      S    ?          0:00 sshd: robot@notty
0  1000   870   868  20   0   2296  1448 -      Ss   ?          0:00 /usr/lib/openssh/sftp-server
0  1000   872   868  20   0  34148  4724 poll_s Ssl  ?          0:00 brickrun --directory=/home/robot/vscode-hello-python /home/robot/vscode-hello-python/hello.
0  1000   884   378  20   0   9100  6572 poll_s S+   tty5       0:02 python3 /home/robot/vscode-hello-python/hello.py
0  1000   907   722  20   0   7200  2300 -      R+   pts/0      0:00 ps -alx
```

这次总共有 28 个用户进程。此时多了如下 5 个进程：
```
4     0   857   518  20   0  11524  5568 -      Ss   ?          0:01 sshd: robot [priv]
5  1000   868   857  20   0  11524  4800 -      S    ?          0:00 sshd: robot@notty
0  1000   870   868  20   0   2296  1448 -      Ss   ?          0:00 /usr/lib/openssh/sftp-server
0  1000   872   868  20   0  34148  4724 poll_s Ssl  ?          0:00 brickrun --directory=/home/robot/vscode-hello-python /home/robot/vscode-hello-python/hello.
0  1000   884   378  20   0   9100  6572 poll_s S+   tty5       0:02 python3 /home/robot/vscode-hello-python/hello.py
```

过程似乎是，在 VSCode 需要将代码灌进 EV3 设备中执行时，EV3 设备的系统中启动了 sftp 服务器 `sftp-server` 用于传送代码，代码传送完成后执行 `brickrun --directory=/home/robot/vscode-hello-python /home/robot/vscode-hello-python/hello.py` 命令运行我们的代码。这个 `brickrun` 进程的父进程是进程 ID 为 868 的 ` sshd: robot@notty` 进程。

`brickrun` 并没有直接解释执行我们的 Python 代码，而是借助于进程号为 378 的 `conrun-server` 和 EV3 设备系统中的 Python3 解释器 `python3` 解释执行我们的代码。

 [vscode-hello-python](https://github.com/ev3dev/vscode-hello-python) 的代码所完成的功能主要是，在设备的屏幕上显示 “Hello World!”，并在终端上显示 “Hello VS Code!”。通过 VSCode 的插件将代码灌进设备并执行，无疑可以正确的完成这些功能。

如果我们用 SSH 登录进设备的系统中之后，直接在终端中执行 Python 代码，就像下面这样：
```
robot@ev3dev:~$ python3 /home/robot/vscode-hello-python/hello.py
Hello World!
Hello VS Code!
```

此时，无法看到字符串被显示在 LEGO EV3 设备的屏幕上，两条字符串都被显示在了终端中。

如果我们直接在终端中执行 `brickrun` 命令，如下面这样：
```
robot@ev3dev:~$ brickrun --directory=/home/robot/vscode-hello-python /home/robot/vscode-hello-python/hello.py
Hello VS Code!
```

此时代码依然可以如最初预期的那样运行。如果查看 LEGO EV3 设备中进程运行情况的话，可以看到多了如下这两个进程：
```
0  1000  1046   722  20   0  34148  4616 poll_s Sl+  pts/0      0:00 brickrun --directory=/home/robot/vscode-hello-python /home/robot/vscode-hello-python/hello.py
0  1000  1058   378  20   0  10696  6736 poll_s S+   tty5       0:02 python3 /home/robot/vscode-hello-python/hello.py
```

`brickrun` 进程由进程号为 722 号的我们正在运行的 `bash` 进程创建，Python3 解释器进程依然由进程号为 378 的 `conrun-server` 进程创建。

在 [搭建 LEGO EV3 的 PyCharm Python 开发环境](https://www.wolfcstech.com/2018/10/24/setting-up-python-pycharm/) 一文中，可以看到，用 Python 代码控制马达转动的代码，可以直接在终端中用 Python 解释器如我们预期的那样执行。

这就说明，直接在终端中执行 Python 解释器的 Python 运行环境，和 `brickrun`/`conrun-server` 创建的 Python 运行环境是不同的，主要的不同点应该在 Python 标准库中输入输出函数的行为上。

总体上来看，LEGO EV3 中执行 VSCode Python 代码的过程，似乎主要是一个远程代码执行的过程。

LEGO EV3 设备中的 ev3dev 镜像的官方网站为 [ev3dev](https://www.ev3dev.org/)，ev3dev 镜像中运行的定制的用户进程的代码基本上都是开源的。前面提到的几个命令的源码地址如下：

 * `brickrun` - **[brickrun](https://github.com/ev3dev/brickrun)**
 * `conrun-server` - **[console-runner](https://github.com/ev3dev/console-runner)**

ev3dev 的 VSCode 插件同样是开源的，其地址为 **[vscode-ev3dev-browser](https://github.com/ev3dev/vscode-ev3dev-browser)**。



