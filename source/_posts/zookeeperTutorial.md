---
title: 使用ZooKeeper编程 - 快速教程
date: 2018-08-01 19:37:49
categories: 后台开发
tags:
- 后台开发
- 翻译
---

在这份教程中，我们展示使用 ZooKeeper 的 barriers 和生产者-消费者队列的简单实现。我们将各自的类称作 Barrier 和 Queue。这些例子假设你至少已经运行了一个 ZooKeeper 服务器。
<!--more-->
两个原语都使用下面的代码片段：
```
static ZooKeeper zk = null;
static final Object mutex = new Object();
 
String root;
 
SyncPrimitive(String address)
throws KeeperException, IOException {
    if(zk == null){
            System.out.println("Starting ZK:");
            zk = new ZooKeeper(address, 3000, this);
            System.out.println("Finished starting ZK: " + zk);
    }
}
 
public void process(WatcherEvent event) {
    synchronized (mutex) {
        mutex.notify();
    }
}
```

两个类都扩展 `SyncPrimitive`。通过这种方式，我们在 `SyncPrimitive` 的构造函数中执行所有原语共有的步骤。为了使例子保持简单，我们在第一次实例化一个 barrier 对象或 queue 对象时创建一个 `ZooKeeper` 对象，我们声明一个静态变量引用该对象。后续的 Barrier 和 Queue 实例检查 ZooKeeper 对象是否存在。此外，我们还可以让应用程序创建一个 ZooKeeper 对象，然后把它传给 Barrier 和 Queue 的构造函数。

我们使用 `process()` 方法处理由于观察点触发的通知。在下面的讨论中，我们将展示设置观察点的代码。观察点是内部结构，它可以让 ZooKeeper 通知客户端节点的改变。比如，如果客户端在等待其它客户端离开一个 barrier，则它可以设置一个观察点并等待可以表明等待的结束的特定节点的修改。一旦我们浏览了示例这点就变得很清晰。

## Barriers

Barrier 是能够让一组进程同步计算的开始和结束的原语。该实现的一般思想是让一个 barrier 节点成为独立的进程节点的父节点。假设我们称 barrier 节点为 "/b1"。然后每个进程 "p" 创建一个节点 "/b1/p"。一旦足够的进程创建了它们对应的节点，则加入的进程可以开始进行计算。

在这个例子中，每个进程实例化一个 Barrier 对象，且它的构造函数接受的参数如下：

 * ZooKeeper 服务器的地址(比如，"zoo1.foo.com:2181")；
 * ZooKeeper 上 barrier 节点的路径(比如 "/b1");
 * 进程组的大小。

Barrier 的构造函数把 ZooKeeper 服务器的地址传给它的父类的构造函数。如果还未创建的话，父类创建一个 ZooKeeper 的实例。然后 Barrier 的构造函数在 ZooKeeper 上创建一个 barrier 节点，它是所有进程节点的父节点，我们称为 root（注意：这不是 ZooKeeper 跟 "/"）。

```
      /**
******* Barrier constructor
*******
******* @param address
******* @param name
******* @param size
******* /
        Barrier(String address, String name, int size)
        throws KeeperException, InterruptedException, UnknownHostException {
            super(address);
            this.root = name;
            this.size = size;
 
            // Create barrier node
            if (zk != null) {
                    Stat s = zk.exists(root, false);
                    if (s == null) {
                        zk.create(root, new byte[0], Ids.OPEN_ACL_UNSAFE, 0);
                    }
            }
 
            // My node name
            name = new String(InetAddress.getLocalHost().getCanonicalHostName().toString());
        }
```

为了进入 barrier，进程调用 `enter()`。进程在 root 下创建一个节点来表示它，使用它自己的主机名来构造节点名。然后它开始等待，直到足够多的进程进入了 barrier。进程通过用 `getChildren()` 检查 root 节点的子节点个数，并在子节点数不足时继续等待通知来做到这一点。为了在 root 节点有变化时收到 通知，进程需要设置一个观察点，并通过调用 `getChildren()` 执行它。在这份代码中，我们的 `getChildren()` 接受两个参数。第一个说明了要读取的节点，第二个是让进程设置一个观察点的 boolean 标记。在这份代码中，标记为 true。

```
    /**
******* Join barrier
*******
******* @return
******* @throws KeeperException
******* @throws InterruptedException
******* /
 
        boolean enter() throws KeeperException, InterruptedException{
            zk.create(root + "/" + name, new byte[0], Ids.OPEN_ACL_UNSAFE,
                    CreateFlags.EPHEMERAL);
            while (true) {
                synchronized (mutex) {
                    ArrayList<String> list = zk.getChildren(root, true);
 
                    if (list.size() < size) {
                        mutex.wait();
                    } else {
                        return true;
                    }
                }
            }
        }
```

注意 `enter()` 抛出 KeeperException 和 InterruptedException，因此应用程序有责任捕获并处理这些异常。

一旦计算结束，进程调用 `leave()` 离开 barrier。它首先删除它对应的节点，然后获得 root 节点的子节点。如果有至少一个子节点，则它等待通知（注意：调用 `getChildren()` 的第二个参数是 true，意味着 ZooKeeper 将在 root 节点上设置一个观察点）。收到通知后，它再次检查根节点是否有任何子节点。

```
     /**
******* Wait until all reach barrier
*******
******* @return
******* @throws KeeperException
******* @throws InterruptedException
******* /
 
        boolean leave() throws KeeperException, InterruptedException{
            zk.delete(root + "/" + name, 0);
            while (true) {
                synchronized (mutex) {
                    ArrayList<String> list = zk.getChildren(root, true);
                        if (list.size() > 0) {
                            mutex.wait();
                        } else {
                            return true;
                        }
                    }
                }
        }
```

## 生产者-消费者队列

生产者-消费者队列是进程组用于生成和消费数据的分布式数据结构。生产者进程创建新的元素并把它们加进队列。消费者进程从列表中移除元素，并处理它们。在这个实现中，元素是简单的整数。队列由一个 root 节点表示，为了向队列中添加一个元素，生产者进程创建一个新的节点，根节点的子节点。

下面的代码片段是对象的构造函数。与 Barrier 一样，它首先调用父类 SyncPrimitive
 的构造函数，如果不存在的话其创建一个 ZooKeeper 对象。它然后验证队列的根节点是否存在，如果不存在的话就创建它。

```
     /**
******* Constructor of producer-consumer queue
*******
******* @param address
******* @param name
******* /
        Queue(String address, String name)
        throws KeeperException, InterruptedException {
            super(address);
            this.root = name;
            // Create ZK node name
            if (zk != null) {
                    Stat s = zk.exists(root, false);
                    if (s == null) {
                        zk.create(root, new byte[0], Ids.OPEN_ACL_UNSAFE, 0);
                    }
            }
        }
```

生产者进程调用 `produce()` 向队列中添加一个元素，并传递一个整数作为参数。为了向队列中添加一个元素，方法使用 `create()` 创建一个新节点，并使用 `SEQUENCE` 标记来指示 ZooKeeper 附加与根节点关联的 sequencer 计数器的值。通过这种方式，我们对队列的元素施加了一个总顺序，这保证了队列中最老的元素将是下一个被消费的。

```
    /**
******* Add element to the queue.
*******
******* @param i
******* @return
******* /
 
        boolean produce(int i) throws KeeperException, InterruptedException{
            ByteBuffer b = ByteBuffer.allocate(4);
            byte[] value;
 
            // Add child with value i
            b.putInt(i);
            value = b.array();
            zk.create(root + "/element", value, Ids.OPEN_ACL_UNSAFE,
                        CreateFlags.SEQUENCE);
 
            return true;
        }
```

为了消费元素，消费者进程获得根节点的子节点，读取计数器值最小的节点，并返回元素。注意如果存在冲突，则两个竞争进程中的一个将无法删除节点，且删除操作将抛出异常。

对 `getChildren()` 的调用以字典顺序返回子节点的列表。由于字典顺序不一定要与计数器值的数字顺序一致，则我们需要决定哪个元素是最小的。为了决定哪个具有最小的计数器值，我们遍历列表，然后从每个元素中移除 "element" 前缀。

```
     /**
******* Remove first element from the queue.
*******
******* @return
******* @throws KeeperException
******* @throws InterruptedException
******* /
        int consume() throws KeeperException, InterruptedException{
            int retvalue = -1;
            Stat stat = null;
 
            // Get the first element available
            while (true) {
                synchronized (mutex) {
                    ArrayList<String> list = zk.getChildren(root, true);
                    if (list.isEmpty()) {
                        System.out.println("Going to wait");
                        mutex.wait();
                    } else {
                        Integer min = new Integer(list.get(0).substring(7));
                        for(String s : list){
                            Integer tempValue = new Integer(s.substring(7));
                            if(tempValue < min) min = tempValue;
                        }
                        System.out.println("Temporary value: " + root + "/element" + min);
                        byte[] b = zk.getData(root + "/element" + min, false, stat);
                        zk.delete(root + "/element" + min, 0);
                        ByteBuffer buffer = ByteBuffer.wrap(b);
                        retvalue = buffer.getInt();
 
                        return retvalue;
                    }
                }
            }
        }
```

你还可以查看 Eurosys 2011 上的 [EurosysTutorial](https://cwiki.apache.org/confluence/display/ZOOKEEPER/EurosysTutorial)。

[原文](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Tutorial)
