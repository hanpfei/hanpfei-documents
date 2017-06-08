---
title: WiFi 热点共享设置
date: 2017-06-05 09:17:49
categories: 网络调试
tags:
- 网络调试
---

借助于 WiFi 热点共享，分析移动设备网络流量非常方便，但 WiFi 热点共享竟然这么简单就可以实现。

# Windows 平台

1. 以管理员身份运行 `cmd`
点击***开始***， 选择 ***附件***，找到命令提示符，右键单击选择 ***以管理员身份运行***。
![runcmd.png](https://www.wolfcstech.com/images/1315506-570d4caf85c32c89.png)
2. 设置 WLAN 模式
输入如下命令，并回车，将无线网卡设置为承载网络模式：
```
netsh wlan set hostednetwork mode=allow ssid="HanpfeiAP" key=“1234567890”
```
输出以下信息表示设置成功：
![](https://www.wolfcstech.com/images/1315506-2a74cb327bdf805b.png)
**ssid** 和 **key** 参数的值根据自己的情况进行设置。
3. 启动承载网络
输入如下命令并回车启动承载网络：
```
netsh wlan start hostednetwork
```
命令执行输出如下：
![](https://www.wolfcstech.com/images/1315506-779b15ef9da2c73a.png)
此时，在移动设备端，比如 Android，已经可以搜到共享的 WiFi AP 了。如下图：
![](https://www.wolfcstech.com/images/1315506-e8e0c8675f276f85.png)
在移动设备上像连接任何 WiFi Ap 那样连接共享的 WiFi AP。 
4. 共享网络连接给共享 WiFi AP
点击***开始***， 选择 ***控制面板*** 打开控制面板。点击  ***网络和 Internet*** -> ***网络和共享中心*** ->  ***网络和共享中心*** 。可以看到一块 **Microsoft Virtual WiFi Miniport Adapter** 虚拟网卡：
![](https://www.wolfcstech.com/images/1315506-9c0f484d70f926d5.png)
选中正在联网的网卡，如上图中的 **无线网络连接**，右键单击，选择 **属性** -> **共享**：
![](https://www.wolfcstech.com/images/1315506-43368b2ad75cd88c.png)
选中 **允许共享**，并选择虚拟网卡 **无线网络连接3**。如上图所示。
至此，无线网络共享打开成功。不过用这种方法设置 Win7 热点会出现一些问题，比如说每次开机都要设置一次（当然也可以批量处理解决这个问题）。
5. 停止承载网络
输入如下命令并回车，停止承载网络：
```
netsh wlan stop hostednetwork
```
输出如下：
![](https://www.wolfcstech.com/images/1315506-450e7c3948c7d33b.png)
移动设备将无法搜索到该 WiFi AP了。

# Ubuntu 16.04
Ubuntu 16.04 里面可以直接创建 WiFi 热点，Android 手机可以连接，而不用像以前的版本，还要其他辅助工具。
 
系统要求：带有线和无线网卡的笔记本电脑。
 
步骤：
1.打开网络选项：
![](https://www.wolfcstech.com/images/1315506-4b73e7860a815fbf.jpg)
 
2.添加一个新连接：
![](https://www.wolfcstech.com/images/1315506-265e320933c4b618.jpg)
 
3.配置新连接：
编辑wifi的名字：SSID；选择 Hotspot （热点）模式：
![](https://www.wolfcstech.com/images/1315506-807290a20e28ed47.jpg)
在 Wifi Security 页, 选择 WPA & WPA2 Personal选项并输入你要创建的wifi密码：
![](https://www.wolfcstech.com/images/1315506-61266029e69cbb32.jpg)
在 IPv4 设置页面, 选择 “Share to other computers”：
![](https://www.wolfcstech.com/images/1315506-c0900eff0b16617b.jpg)
保存。
 
4.Connect to Hidden Wi-Fi network 选择你刚刚创建的网络
![](https://www.wolfcstech.com/images/1315506-778b5ba91e3f7927.jpg)
 
最后，确保已连接了有线连接，完成。
 