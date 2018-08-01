---
title: ZooKeeper 入门：用 ZooKeeper 协调分布式应用程序
date: 2018-07-30 19:37:49
categories: 后台开发
tags:
- 后台开发
- 翻译
---

本文档包含使你快速入门 ZooKeeper 的信息。它主要针对希望尝试 ZooKeeper 的开发人员，还包含单 ZooKeeper 服务器的简单安装指导，一些用以验证它正在运行的命令，以及一个简单的编程示例。最后，为方便起见，有一些部分涉及更复杂的安装，例如运行复制部署和优化事务日志。然而，对于商业部署的完整指导，请参考 [ZooKeeper 管理员指南](http://zookeeper.apache.org/doc/current/zookeeperAdmin.html)。
<!--more-->
## 先决条件

参考 Admin 指导中的 [系统要求](http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_systemReq)。

## 下载

要获得 ZooKeeper 发行版，请从一个 Apache 下载镜像下载最近的[稳定](http://zookeeper.apache.org/releases.html) 版本。

## 独立操作

在独立模式下设置ZooKeeper服务器非常简单。服务器包含在一个单独的 JAR 文件中，因此安装由创建一个配置构成。

一旦下载了稳定的 ZooKeeper 发行版，则解包它，并 `cd` 进根目录。

要启动 ZooKeeper，你需要一个配置文件。这里有一个样例，在 **conf/zoo.cfg** 创建它
```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```

注：`conf/` 目录下有一个示例配置文件 `zoo_sample.cfg`，其内容如下：
```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```

可以基于此文件创建需要的配置文件。

这个文件可以叫任何名字，但为了讨论方便，称它为 **conf/zoo.cfg**。修改 **dataDir** 的值指定一个已经存在的目录（以空开始）。这里是每个字段的含义：

**tickTime** - ZooKeeper使用的基本时间单位（以毫秒为单位）。它被用于执行心跳，最小的会话时间将是两倍的 `tickTime`。

**dataDir** - 存储内存数据库快照的位置，除非指定了其它值，对数据的更新的事务日志也位于此处。

**clientPort** - 用于客户端连接的监听的端口。

现在你创建了配置文件，你可以启动 ZooKeeper 了：
```
bin/zkServer.sh start
```

ZooKeeper 使用 log4j 志记消息 -- 更多细节可以在程序员指南的[日志](http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#Logging) 一节找到。你将看到志记消息出现在终端（默认）和/或一个依赖 log4j 配置的日志文件中。

此处列出的步骤以独立模式运行 ZooKeeper。没有复制，因此如果 ZooKeeper 进程失败，服务将停掉。这适用于大多数开发情况，但要以复制模式运行 ZooKeeper，请参考 [运行复制的 ZooKeeper](http://zookeeper.apache.org/doc/current/zookeeperStarted.html#sc_RunningReplicatedZooKeeper)。

## 管理 ZooKeeper 存储

对于长期运行的生产系统，必须在外部管理 ZooKeeper 存储（dataDir和logs）。请参考 [维护](http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance) 一节来了解更多细节。

## 连接到 ZooKeeper
```
$ bin/zkCli.sh -server 127.0.0.1:2181
```
这使你可以执行简单的，类似文件的操作。

一旦你连接成功了，你应该看到像下面这样：
```
Connecting to localhost:2181
log4j:WARN No appenders could be found for logger (org.apache.zookeeper.ZooKeeper).
log4j:WARN Please initialize the log4j system properly.
Welcome to ZooKeeper!
JLine support is enabled
[zkshell: 0]
```

在 shell 中，键入 `help` 获得可以在客户端执行的命令列表，如下：
```
[zkshell: 0] help
ZooKeeper host:port cmd args
        get path [watch]
        ls path [watch]
        set path data [version]
        delquota [-n|-b] path
        quit
        printwatches on|off
        createpath data acl
        stat path [watch]
        listquota path
        history
        setAcl path acl
        getAcl path
        sync path
        redo cmdno
        addauth scheme auth
        delete path [version]
        setquota -n|-b val path
```

自此你可以尝试一些简单的命令来获得这些简单的命令行接口的一些感觉。首先，从发出 list 命令开始，如在 `ls` 中，产生：
```
[zkshell: 8] ls /
[zookeeper]
```

接下来，通过运行 `create /zk_test my_data` 创建一个新 znode。这将创建一个新的 znode，并把它和字符串 "my_data" 关联起来。你应该看到：
```
[zkshell: 9] create /zk_test my_data
Created /zk_test
```

发出另一个 `ls /` 命令来查看目录的样子：
```
[zkshell: 11] ls /
[zookeeper, zk_test]
```

注意，现在已经创建了 `zk_test` 目录。

接下来，通过运行 `get` 命令验证与 znode 关联的数据，如下：
```
[zkshell: 12] get /zk_test
my_data
cZxid = 5
ctime = Fri Jun 05 13:57:06 PDT 2009
mZxid = 5
mtime = Fri Jun 05 13:57:06 PDT 2009
pZxid = 5
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0
dataLength = 7
numChildren = 0
```

我们可以通过发出 `set` 命令来改变与 `zk_test` 关联的数据，如下：
```
[zkshell: 14] set /zk_test junk
cZxid = 5
ctime = Fri Jun 05 13:57:06 PDT 2009
mZxid = 6
mtime = Fri Jun 05 14:01:52 PDT 2009
pZxid = 5
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0
dataLength = 4
numChildren = 0
[zkshell: 15] get /zk_test
junk
cZxid = 5
ctime = Fri Jun 05 13:57:06 PDT 2009
mZxid = 6
mtime = Fri Jun 05 14:01:52 PDT 2009
pZxid = 5
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0
dataLength = 4
numChildren = 0
```
（注意我们在设置数据之后执行了一个 `get`，且它执行了，确实，改变了。）

最后让我们发出 `delete` 命令来删除节点：
```
[zkshell: 16] delete /zk_test
[zkshell: 17] ls /
[zookeeper]
[zkshell: 18]
```

现在就是这样。要浏览更多，请继续本文档的其余部分，并参考  [程序员指南](http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html)。

## ZooKeeper 编程

ZooKeeper 具有 Java 绑定和 C 绑定。它们在功能上是一样的。C 绑定有两个变体：单线程的和多线程的。它们的不同仅在于消息循环如何完成。更多信息，请参考[ZooKeeper 程序员指南中的编程示例](http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#ch_programStructureWithExample) 中的使用不同 API 的示例代码。

## 运行复制的 ZooKeeper

在独立模式下运行ZooKeeper便于评估，开发和测试。但是在生产中，你应该以复制模式运行 ZooKeeper。相同应用程序中的服务器复制组被称为 *quorum*，并以复制模式，quorum 中的所有服务器都具有相同配置文件的拷贝。

注意：对于复制模式，最少需要 3 台服务器，强烈建议你具有奇数台服务器。如果你只有两台服务器，那么你将处于这样的情况：如果其中一台服务器出现故障，则没有足够的机器来构成多数仲裁。两台服务器相对于单台服务器固有的**更缺少**稳定性，因为有两个单点失败。

复制模式所需的 **conf/zoo.cfg** 文件与独立模式中使用的类似，但是有一些不同。这是一个例子：
```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

新的项 **initLimit** 是超时，ZooKeeper 用以限制 ZooKeeper 服务器在仲裁中不得不连接 leader的时间长度。**syncLimit** 项限制服务器与 leader 的过期时间。

这些超时中，你可以使用 **tickTime** 指定时间单位。在这个例子中，initLimit 的超时时间是 5 ticks，2000 毫秒一个 tick，或 10 秒。

*server.X* 形式的项列出了组成 ZooKeeper 服务的服务器。当服务器启动时，它通过查找数据目录中的 `myid` 文件来知道它自己是那台服务器。那个文件包含服务器号，以 ASCII 的形式。

最后，注意每个服务器名后面的两个端口号："2888" and "3888"。节点使用前面的端口连接其它节点。这个连接在节点间的通信中是必须的，比如，就更新的顺序达成一致。更具体地说，ZooKeeper服务器使用此端口将 follower 连接到 leader。当新的 leader 出现时，follower 使用此端口打开与 leader 的TCP连接。由于默认的 leader 选举也使用 TCP，我们目前需要另一个端口进行 leader 选举。

注意
如果你想在单台机器上测试多个服务器，则把 servername 指定为 localhost，然后为服务器的配置文件中的每个 server.X 设置唯一的 quorum & leader election 端口（比如，上例中的 2888:3888, 2889:3889, 2890:3890）。当然也需要把 *dataDirs* 和 *clientPorts * 分开（在上面是重复的示例中，在单个 localhost 上运行，你仍然会有三个配置文件）。

请注意在单台机器上设置多个服务器并不会产生任何冗余。如果发生了某些会使机器挂掉的事情，则所有的 zookeeper 服务器都将掉线。完整的冗余需要每个服务器有它自己的机器。它必须是一台完全分开的物理服务器。相同物理主机上的多台虚拟机器仍然容易受到该主机的完全故障的影响。

## 其它优化

有两个其它的配置参数可以有效地提升性能：

 * 为了在更新时获得低延迟，有一个专门的事务日志目录是很重要的。默认情况下，事务日志与数据快照和 *myid* 文件放在相同的目录下。dataLogDir参数指示用于事务日志的不同目录。
 * [tbd: 什么是其他配置参数？]


[原文](http://zookeeper.apache.org/doc/current/zookeeperStarted.html)
