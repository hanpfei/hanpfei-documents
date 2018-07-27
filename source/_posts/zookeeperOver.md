---
title: ZooKeeper：分布式应用程序的分布式协调服务
date: 2018-07-27 13:37:49
categories: 后台开发
tags:
- 后台开发
- 翻译
---

ZooKeeper 是一个分布式应用程序的分布式的，开源的协调服务。它提供了一组简单的原语，分布式应用程序可以基于它为同步，配置维护，分组和命名而实现更高层的服务。它被设计为易于编程的，它使用了我们熟悉的文件系统目录树结构风格的数据模型。它运行于 Java 中，但具有 Java 和 C 的绑定。

众所周知，协调服务很难实现。它们特别容易出现诸如竞态条件和死锁这样的错误。ZooKeeper 背后的动机是解除分布式应用程序从头开始实现协调服务的责任。
<!--more-->
## 设计目标

**ZooKeeper 是简单的。** ZooKeeper 允许分布式进程通过一个共享的与标准文件系统的组织方式类似的分层命名空间相互协作。命名空间由数据寄存器组成 - 在ZooKeeper的说法中称为 znodes - 且它们与文件和目录类似。不像为存储而设计的典型文件系统那样，ZooKeeper 在内存中保存数据，这意味着 ZooKeeper 可以实现高吞吐量和低延迟数量。

ZooKeeper 实现非常重视高性能，高可用性，严格有序的访问。ZooKeeper 的性能方面意味着它可以在大型分布式系统中使用。可靠性方面使其不会出现单点失败的问题。严格有序意味着可以在客户端实现复杂的同步原语。

**ZooKeeper 是冗余的。**如同它协调的分布式进程那样，ZooKeeper 本身旨在通过称为 `ensemble` 的一组主机间进行复制。

![ZooKeeper 服务](https://www.wolfcstech.com/images/1315506-c00ff2a00a1b35ba.jpg)

组成 ZooKeeper 服务的服务器必须彼此了解。它们维护内存中的状态镜像，以及持久性存储中的事务日志和快照。只要大部分服务器可用，ZooKeeper 服务就可用。

客户端连接到一个 ZooKeeper 服务器。客户端维护一个 TCP 连接，通过它发送请求，获得响应，获得观察事件，并发送心跳。如果与服务器的 TCP 连接断开了，客户端将连接一个不同的服务器。

**ZooKeeper 是有序的。** ZooKeeper 以一个可以反映在所有 ZooKeeper 事务中的顺序的数字标记每次更新。后续的操作可以使用顺序实现更高层的抽象，比如同步源语。

**ZooKeeper 是快速的。** 在 “读取为主导” 工作负载中特别快。ZooKeeper 应用程序运行于几千台机器上，在读取比写入更常见，比例大约为 10：1 的情况下它执行地最好。

## 数据模型和分层命名空间

ZooKeeper 提供的命名空间与标准的文件系统非常像。名字是由反斜线（/）分割的一系列路径元素。ZooKeeper 命名空间中的每个节点由一个路径标识。

![ZooKeeper 的分层命名空间](https://www.wolfcstech.com/images/1315506-fa12c3dfb178ab61.jpg)

## 节点和短暂节点

不像标准的文件系统，ZooKeeper 命名空间中的每个节点都可以有与其关联的数据和子节点。它就像一个允许文件同时是目录的文件系统。（ZooKeeper 被设计为用于存储协调数据：状态信息，配置，位置信息，等等，因此每个节点所存储的数据通常都很小，在字节到千字节范围内。）我们使用术语 *znode* 使我们讨论的 ZooKeeper 数据节点更清晰。

Znodes 维护一个状态结构，其中包含数据修改的版本号，ACL 修改，和时间戳，以允许缓存验证和协调更新。每次 znode 的数据修改了，其版本号都增加。例如，每当客户端检索数据时，它也接收数据的版本号。

命名空间中的每个 znode 所存储的数据的读和写都是原子的。读取获得与一个 znode 关联的所有数据字节，写入替换所有的数据。每个节点具有一个访问控制表（ACL），其限制了谁可以做什么。

ZooKeeper 还有临时节点的概念。这些 znodes 的生命期与创建 znode 的会话活跃期一样长。当会话结束时则 znode 被删除。当你想要实现 *[tbd]* 时临时节点很有用。

## 条件更新和观察点

ZooKeeper 支持 *观察点* 的概念。客户端可以在 znodes 上设置观察点。当 znode 修改时观察点将被触发并被移除。观察点被触发时，客户端收到一个包，说明 znode 被修改了。如果客户端和其中一个 ZooKeeper 服务器之间的连接断开了，客户端将收到一个本地通知。这些可以被用于 *[tbd]*。

## 保证

ZooKeeper 非常快速且非常简单。由于它的目标是成为构建更复杂的服务的基础设施，比如同步，它提供了一系列保证。包括：

 * 顺序一致性 - 来自于客户端的更新将以它们被发送的顺序应用。
 * 原子性 - 更新要么成功要么失败。没有部分结果。
 * 单系统镜像 - 客户端将看到服务相同的视图，无论它连接的是哪个服务器。
 * 可靠性 - 一旦应用了一个更新，则它将从那一刻持久化，直到客户端覆写了更新。
 * 及时性 - 系统的客户端视图被保证在一定的时间范围内保持最新。

欲了解更多关于此的信息，及它们如何使用，请参考 *[tbd]*。

## 简单的 API

ZooKeeper 的一个设计目标是提供非常简单的编程接口。因此，它仅支持以下操作：

`create` - 在树的特定位置创建一个节点
`delete` - 删除节点
`exists` - 测试特定位置节点是否存在
`get data` - 从节点读取数据
`set data` - 向节点写入数据
`get children` - 提取节点的子节点列表
`sync` - 等待数据传播

要获得关于这些内容更深入的讨论，及如何使用它们实现更高层的操作，请参考 *[tbd]*。

## 实现

ZooKeeper Components 展示了 ZooKeeper 服务的高层组件。除了请求处理器之外，组成  ZooKeeper 服务的每个服务器都复制其自己的每个组件的副本。

![ZooKeeper 组件](https://www.wolfcstech.com/images/1315506-24af7617b6c3af58.jpg)

复制的数据库是一个内存数据库，其包含了完整的数据树。为了可恢复性，更新被志记在磁盘中，在写操作被应用于内存数据库之前它们先被序列化进磁盘中。

每个 ZooKeeper 都为客户端提供服务。客户端连接到一个特定的服务器来提交请求。读取请求由每个服务器数据库的本地复制提供服务。改变服务的状态的请求，写请求，由一个一致性协议处理。

作为一致性协议的一部分，所有来自于客户端的写请求被转发给一个特定的服务器，称为 *leader*。其它的 ZooKeeper 服务器，称为 *followers*，接收来自 leader 的消息提议并同意消息传递。消息传递层负责替换失败的 leaders 并将 followers 与 leaders 同步。

ZooKeeper 使用一个定制的原子性消息协议。由于消息传递层是原子的，ZooKeeper 可以保证本地副本永远不会分叉。当 leader 接收到一个写请求时，它计算写操作应用时系统的状态，并把这转换为捕获新状态的事务。

## 使用

ZooKeeper 的编程接口很简单。然而通过它你可以实现更高级的顺序操作，比如同步原语，组成员身份，所有权等。一些分布式应用程序已经使用它：*[tbd: add uses from white paper and video presentation.]* 更多信息，请参考 *[tbd]*。

## 性能

ZooKeeper 的设计目标是高性能。但是做到了么？Yahoo! Research 的 ZooKeeper 开发团队的结果表明做到了。（参考 [ZooKeeper 吞吐量随读-写比例变化的变化](http://zookeeper.apache.org/doc/current/zookeeperOver.html#fg_zkPerfRW)。）在读取数量超过写入的应用程序中，它的性能特别高，因为写操作包括同步所有服务器的状态。（读取数量超过写入是典型的协调服务的场景。）

![ZooKeeper 吞吐量随读-写比例变化的变化](https://www.wolfcstech.com/images/1315506-151e8c18a0292b9f.jpg)

图 [ZooKeeper 吞吐量随读-写比例变化的变化](http://zookeeper.apache.org/doc/current/zookeeperOver.html#fg_zkPerfRW) 是 ZooKeeper 发行版 3.2 运行于双  2Ghz Xeon 和 SATA 15K RPM 驱动器的服务器上的吞吐量图。一个驱动器被用作专门的 ZooKeeper 志记设备。快照被写入 OS 驱动器。写请求是 1K 的写操作，读请求是 1K 的读操作。图中的 "Servers" 是 ZooKeeper 集群的大小，组成服务的服务器的数量。接近 30 台其它服务器被用于模拟客户端。ZooKeeper 集群被配置为 leaders 不允许接受来自于客户端的连接。

基准测试也展示了它是可靠的。 [错误出现时的可靠性](http://zookeeper.apache.org/doc/current/zookeeperOver.html#fg_zkPerfReliability) 展示了部署如何响应各种失败。图中标记的事件如下：

 1. 一个 follower 的失败和恢复
 2. 一个不同的 follower 的失败和恢复
 3. Leader 的失败
 4. 两个 follower 的失败和恢复
 5. 另一个 leader 的失败

## 可靠性

为了展示系统随着时间和失败的插入的行为，我们运行了一个由 7 台服务器组成的 ZooKeeper 服务。我们之前运行了相同的饱和基准测试，但是这次我们将写的百分比固定在 30%，这是我们预期工作负载的保守比率。

![zkperfreliability.jpg](https://www.wolfcstech.com/images/1315506-8535c80afa4a2122.jpg)

这张图中有一些重要的观察。首先，如果 followers 失败并快速恢复，则尽管有失败， ZooKeeper 依然能够承受很高的吞吐量。但可能更重要地是，leader  选举算法允许系统足够快地恢复过来，以防止吞吐量大幅下降。在我们的观察中，ZooKeeper 选举新 leader 耗费的时间小于 200ms。第三，至于 followers  恢复，一旦它们开始处理请求，ZooKeeper 可以再次提升吞吐量。

## ZooKeeper 项目

ZooKeeper 已经成功地被用于许多工业级的应用程序中了。在 Yahoo!，它被用作
 Yahoo! Message Broker 的协调和失败恢复服务，Yahoo! Message Broker 是一个高可扩展的发布-订阅系统，其为复制和数据传递管理着几千个 topics。Yahoo! crawler 的 Fetching Service 也使用了它，其中 ZooKeeper 也管理失败恢复。大量的 Yahoo! 广告系统服务也使用 ZooKeeper  来实现可靠的服务。

鼓励所有的用户和开发者加入社区并贡献他们的专业知识。参考 [Zookeeper Project on Apache ](http://zookeeper.apache.org/) 来了解更多信息。

[原文](http://zookeeper.apache.org/doc/current/zookeeperOver.html)
