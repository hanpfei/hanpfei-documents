---
title: futex
date: 2023-10-05 21:08:29
categories: Linux
tags:
- Linux
---

## 摘要
 futex - 快速的用户空间锁定

```
       #include <linux/futex.h>
       #include <stdint.h>
       #include <sys/time.h>

       long futex(uint32_t *uaddr, int futex_op, uint32_t val,
                 const struct timespec *timeout,   /* or: uint32_t val2 */
                 uint32_t *uaddr2, uint32_t val3);
```

**注意：** futex 系统调用没有 glibc 包装器。应用程序需要通过 `syscall()` 函数调用 futex 系统调用，如：
```
#include <unistd.h>
#include <sys/syscall.h>

static int futex(uint32_t* uaddr,
                 int futex_op,
                 uint32_t val,
                 const struct timespec* timeout,
                 uint32_t* uaddr2,
                 uint32_t val3) {
  return syscall(SYS_futex, uaddr, futex_op, val, timeout, uaddr2, val3);
}
```

## 描述

`futex()` 系统调用为应用程序提供了一种**等待直到某个条件为真的方法**。它通常用作共享内存同步上下文中的阻塞构件。当使用 futex 时，同步操作主要在用户空间执行。用户空间程序只有在可能阻塞较长时间直到条件为真时，才使用 `futex()` 系统调用。其它 `futex()` 操作可用于唤醒等待特定条件的任何进程或线程。

futex 是一个 32 位的值 (下文称为 futex 字)，其地址提供给 `futex()` 系统调用。(在所有平台上 futex 的大小都是 32 位，包括 64 位系统。) 所有 futex 操作都借助于这个值管理。为了在进程之间共享 futex，该 futex 应该被放置在使用 (比如) mmap(2) 或 shmat(2) 创建的共享内存区域中。(这样，该 futex 字在不同进程中可能具有不同的虚拟地址，但这些地址都指向物理内存中的相同位置。) 在多线程程序中，把 futex 字放在所有线程共享的全局变量中就足够了。

当执行需要阻塞线程的 futex 操作时，内核只有在 futex 字的值与调用线程提供的 futex 字的期望值 (作为 `futex()` 调用的一个参数) 相同时才会阻塞。加载 futex 字的值，该值与期望值的比较，以及实际的阻塞将原子地发生，且将作为整体与其它线程在相同的 futex 字上并发地执行的操作排序。这样，futex 字用于将用户空间的同步和内核的阻塞实现连接起来。类似于原子的比较与交换 (compare-and-exchange) 操作可能会改变共享内存，通过 futex 阻塞是一个原子的比较与阻塞 (compare-and-block) 操作。

futex 的一个使用场景是实现锁。锁的状态 (比如已获得或未获得) 可以表示为共享内存中原子地访问的标记。在没有锁竞争时，线程可以使用原子指令访问或修改锁的状态，比如使用原子的比较与交换 (compare-and-exchange) 指令原子地把它从未获得状态变为已获得状态。(这种指令完全在用户模式下执行，内核无需维护关于锁状态的信息。)另一方面，一个线程可能由于另一个线程已经获得了锁而无法获得锁。它可能将锁的标记作为 futex 字，以及表示已获得状态的值作为期望值传给 `futex()` 等待操作。这个 `futex()` 操作当且仅当锁依然处于已获得状态时才会阻塞 (比如，futex 字中的值依然是“已获得状态”)。当释放锁时，线程必须首先把锁状态复位为未获得，然后执行 futex 操作，唤醒阻塞在用作 futex 字的锁标记上的线程 (这可以进一步优化，以避免不必要的唤醒)。请参阅 futex(7) 了解关于如何使用 futex 的更多细节。

除了基本的等待和唤醒 futex 功能，futex 还有更多的操作，旨在支持更复杂的使用场景。

注意，使用 futex 时无需显式地初始化或销毁；内核只有在像下文描述的 FUTEX_WAIT 这样的操作，在对特定的 futex 字执行时才维护一个 futex (内核内部实现所用的结构)。

### 参数

*uaddr* 参数指向 futex 字。在所有平台上， futex 都是四字节的整数，它必须对齐到四字节边界上。对 futex 执行的操作在 *futex_op* 参数中指定；*val* 值的含义和目的取决于 *futex_op*。

其余的参数 (*timeout*，*uaddr2*，和 *val3*) 只有在下文描述的某些 futex 操作中才需要。其中不需要的参数被忽略。

对于一些阻塞操作，*timeout* 参数是一个指向 *timespec* 结构体的指针，它描述了操作的超时时间。然而，尽管函数原型中，这个参数如上面展示的那样，但对于某些操作，这个参数的最低有效四字节被用作一个整数，其含义由操作决定。对于这些操作，内核首先把 *timeout* 值转换为 *unsigned long*，然后转为 *uint32_t*，在本页的剩余部分，当以这种方式来解释时，这个参数被称为 *val2*。

当需要时，*uaddr2* 参数是指向操作使用的第二个 futex 字的指针。

最后的整型参数的解释，*val3*，依赖于操作。

### Futex 操作

*futex_op* 参数由两部分组成：指定了要执行的操作的命令，与 0 个或多个选项按位或以改变操作的行为。可能包含在 *futex_op* 中的选项如下：

**FUTEX_PRIVATE_FLAG** (从 Linux 2.6.22 开始支持)
这个选项位可以用于所有 futex 操作。它告诉内核，该 futex 是进程私有的，而不与其它进程共享 (比如，它仅被用于相同进程中线程之间的同步)。它允许内核进行一些其它的性能优化。

为方便起见，*<linux/futex.h>* 定义了一系列带有 **_PRIVATE** 后缀的常量，它们等价于下面列出的操作，但常量中包含了 **FUTEX_PRIVATE_FLAG** 的按位或。这样就有了 **FUTEX_WAIT_PRIVATE**，**FUTEX_WAKE_PRIVATE**，等操作。

**FUTEX_CLOCK_REALTIME** (从 Linux 2.6.28 开始支持)
这个选项位只能用于 **FUTEX_WAIT_BITSET**，**FUTEX_WAIT_REQUEUE_PI**，(从 Linux 4.5 开始支持) **FUTEX_WAIT**，和 (从 Linux 5.14 开始支持) **FUTEX_LOCK_PI2** 操作。

如果设置了这个选项，内核将根据 **CLOCK_REALTIME** 时钟测量超时。

如果没有设置这个选项，内核将根据 **CLOCK_MONOTONIC** 时钟测量超时。

*futex_op* 中指定的操作是以下操作之一：

**FUTEX_WAIT** (从 Linux 2.6.0 开始支持)
这个操作测试地址 *uaddr* 指向的 futex 字中的值是否依然包含预期值 *val*，如果是，则休眠等待 futex 字上的 **FUTEX_WAKE** 操作。futex 字中的值的加载是一个原子的内存访问 (比如，使用各架构的原子的机器指令)。加载 futex 字的值，与期望值比较，以及启动休眠将原子地执行，且将作为整体与相同 futex 字上的其它 futex 操作排序。如果线程开始休眠，则它被称为这个 futex 字上的等待者。如果 futex 值与 *val* 不匹配，则调用将立即失败，并返回 **EAGAIN**。

与期望值比较的目的是为了防止丢失唤醒。如果调用线程决定基于之前的值阻塞，而另一个线程之后修改了 futex 字的值，而且如果另一个线程在修改值之后，且在这个 **FUTEX_WAIT** 操作之前，执行了 **FUTEX_WAKE** 操作 (或类似的唤醒)，则调用线程将观察到值的变化，并且不会开始休眠。

如果 *timeout* 不是 NULL，则它指向的结构指定了等待的超时值。(此间隔将四舍五入到系统时钟粒度，并保证不会过早过期。) 超时默认根据 **CLOCK_MONOTONIC** 时钟测量，但从 Linux 4.5 开始，可以通过在 *futex_op* 中指定 **FUTEX_CLOCK_REALTIME** 来选择 **CLOCK_REALTIME** 时钟。如果 *timeout* 为 NULL，则调用将无限阻塞。

*注意*：对于 **FUTEX_WAIT**，*timeout* 解释为一个 *相对* 值。这不同于其它的 futex 操作，其中 *timeout* 解释为绝对值。要获得与 **FUTEX_WAIT** 等价的绝对超时，使用**FUTEX_WAIT_BITSET**，同时 *val3* 指定为 **FUTEX_BITSET_MATCH_ANY**。

忽略参数 `uaddr2` 和 `val3`。

**FUTEX_WAKE** (从 Linux 2.6.0 开始支持)
这个操作唤醒最多 *val* 个等待在 (比如，在 **FUTEX_WAIT** 中) 地址 *uaddr* 处的 futex 字上的等待者。通常，*val* 指定为 1 (唤醒单个等待者) 或 INT_MAX (唤醒所有的等待者)。不能保证哪些等待者会被唤醒 (比如，具有更高调度优先级的等待者不能保证比具有更低优先级的等待者优先被唤醒)。

忽略参数 `timeout`、`uaddr2` 和 `val3`。

**FUTEX_FD** (从 Linux 2.6.0 到 Linux 2.6.25 (包括) 支持)
这个操作创建一个与位于 *uaddr* 的 futex 关联的文件描述符。调用者必须在使用之后关闭返回的文件描述符。当另一个进程或线程在 futex 字上执行 **FUTEX_WAKE** 时，通过 [select(2)](https://man7.org/linux/man-pages/man2/select.2.html)，[poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)，和 [epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html) 文件描述符可以指示其可读。

文件描述符可用于获取异步的通知：如果 `val` 非零，则当另一个进程或线程执行 **FUTEX_WAKE** 时，调用者将收到以传入的 **val** 为信号号的信号。

忽略参数 `timeout`、`uaddr2` 和 `val3`。

因为 **FUTEX_FD** 本身就是不安全的，所以它已经从 Linux 2.6.26 以后的版本中删除了。

**FUTEX_REQUEUE** (从 Linux 2.6.0 开始支持)
该操作执行与 **FUTEX_CMP_REQUEUE** 相同的任务 (见下文)，只是不使用 *val3* 中的值进行检查。(参数 *val3* 被忽略。)

**FUTEX_CMP_REQUEUE** (从 Linux 2.6.7 开始支持)
该操作首先检查位置 *uaddr* 是否仍然包含值 *val3*。如果没有，操作失败，并报错 **EAGAIN**。否则，该操作将唤醒在 *uaddr* 处的 futex 等待的最多 *val* 个等待者。如果有超过 *val* 个等待者，则将剩余的等待者从位于 *uaddr* 的源 futex 的等待队列中删除，并将其添加到位于 *uaddr2* 的目标 futex 的等待队列中。参数 *val2* 指定了在位于 *uaddr2* 处的 futex 中重新入队的等待者数量的上限。

从 *uaddr* 加载是一个原子的内存访问 (比如，使用各架构的原子的机器指令)。此加载，与 *val3* 的比较，以及所有等待者的重新入队原子地执行，且将作为整体与相同 futex 字上的其它操作排序。

为 *val* 指定的典型值为 0 和 1。(指定为 **INT_MAX** 没什么用，因为它将使 **FUTEX_CMP_REQUEUE** 操作等价于 **FUTEX_WAKE** 操作。) 通过 *val2* 指定的限制值通常为 1 或 **INT_MAX**。(把这个参数指定为 0 没什么用，因为它将使 **FUTEX_CMP_REQUEUE** 操作等价于 **FUTEX_WAIT** 操作。)

添加 **FUTEX_CMP_REQUEUE** 操作作为早期的 **FUTEX_REQUEUE** 的替代品。差别在于对位于 *uaddr* 的值的检查可用于确保重新入队仅在某些条件下发生，这允许在某些使用场景中避免竞态条件。

**FUTEX_REQUEUE** 和 **FUTEX_CMP_REQUEUE** 都可以用来避免在使用 **FUTEX_WAKE** 时可能发生的 “雷群” 唤醒，在这种情况下，所有被唤醒的等待者都需要获取另一个futex。考虑下面的场景，其中多个等待者线程等待在 B 上，使用一个 futex 来实现等待队列：
```
                  lock(A)
                  while (!check_value(V)) {
                      unlock(A);
                      block_on(B);
                      lock(A);
                  };
                  unlock(A);
```

如果唤醒者线程使用 **FUTEX_WAKE**，则等待在 B 上的所有等待者将被唤醒，且它们都将尝试获得锁 A。然而，以这种方式唤醒所有线程是毫无意义的，因为除了其中的一个线程外，其它线程都将立即再次阻塞在锁 A 上。相比之下，一个重新入队操作仅唤醒一个等待者，并把其它等待者移到锁 A 上，则当被唤醒的等待者解锁 A 时，下一个等待者可以进行处理。

**FUTEX_WAKE_OP** (从 Linux 2.6.14 开始支持)
添加这个操作是为了支持一些用户空间使用场景，其中多个 futex 必须同时处理。最值得注意的例子是 **pthread_cond_signal**(3) 的实现，它需要操作两个 futex，一个用于实现互斥量，一个用于实现与条件变量关联的等待队列。**FUTEX_WAKE_OP** 允许在不导致高频率的争用和上下文切换的情况下实现这样的场景。

**FUTEX_WAKE_OP** 操作等价于原子地执行如下代码，且作为整体与所提供的两个 futex 字上的任何其它 futex 操作排序：

```
                  uint32_t oldval = *(uint32_t *) uaddr2;
                  *(uint32_t *) uaddr2 = oldval op oparg;
                  futex(uaddr, FUTEX_WAKE, val, 0, 0, 0);
                  if (oldval cmp cmparg)
                      futex(uaddr2, FUTEX_WAKE, val2, 0, 0, 0);
```

换句话说，**FUTEX_WAKE_OP** 做如下这些事：

 * 保存位于 *uaddr2* 的 futex 字的原始值，并执行一个操作来修改位于 *uaddr2* 的 futex 的值；这是一个原子的读-修改-写内存访问 (比如，使用各架构的原子的机器指令)

 * 唤醒最多 *val* 个位于 *uaddr* 的 futex 字的 futex 上的等待者；以及

 * 依赖于位于 *uaddr2* 的 futex 字的原始值的测试结果，唤醒最多 *val2* 个位于 *uaddr2* 的 futex 字的 futex 上的等待者。

操作和要执行的比较被编码进参数 *val3* 的位中。图像上，编码是：
```
                  +---+---+-----------+-----------+
                  |op |cmp|   oparg   |  cmparg   |
                  +---+---+-----------+-----------+
                    4   4       12          12    <== # of bits
```

用代码来表达，编码是：
```
                  #define FUTEX_OP(op, oparg, cmp, cmparg) \
                                  (((op & 0xf) << 28) | \
                                  ((cmp & 0xf) << 24) | \
                                  ((oparg & 0xfff) << 12) | \
                                  (cmparg & 0xfff))
```

在上面的代码中，*op* 和 *cmp* 分别是下面列出的代码之一。*oparg* 和 *cmparg* 组件都是数值字面量，但如下所述除外。

*op* 组件有以下值之一：
```
                  FUTEX_OP_SET        0  /* uaddr2 = oparg; */
                  FUTEX_OP_ADD        1  /* uaddr2 += oparg; */
                  FUTEX_OP_OR         2  /* uaddr2 |= oparg; */
                  FUTEX_OP_ANDN       3  /* uaddr2 &= ~oparg; */
                  FUTEX_OP_XOR        4  /* uaddr2 ^= oparg; */
```

此外，将以下值按位或入 *op* 会导致 *(1 << oparg)* 被用作操作数：
```
                  FUTEX_OP_ARG_SHIFT  8  /* Use (1 << oparg) as operand */
```

*cmp* 字段是下列值之一：
```
                  FUTEX_OP_CMP_EQ     0  /* if (oldval == cmparg) wake */
                  FUTEX_OP_CMP_NE     1  /* if (oldval != cmparg) wake */
                  FUTEX_OP_CMP_LT     2  /* if (oldval < cmparg) wake */
                  FUTEX_OP_CMP_LE     3  /* if (oldval <= cmparg) wake */
                  FUTEX_OP_CMP_GT     4  /* if (oldval > cmparg) wake */
                  FUTEX_OP_CMP_GE     5  /* if (oldval >= cmparg) wake */
```

**FUTEX_WAKE_OP** 的返回值是唤醒的 futex *uaddr* 上的等待者加上唤醒的 futex *uaddr2* 上的等待者的总和。

**FUTEX_WAIT_BITSET** (从 Linux 2.6.25 开始支持)
这个操作与 **FUTEX_WAIT** 类似，除了 *val3* 用于给内核提供了一个 32 位的位掩码。这个位掩码必须设置至少一个位，存储在等待者的内核内部状态中。详细信息请参见 **FUTEX_WAKE_BITSET** 的描述。

如果 *timeout* 不是 NULL，则它指向的结构指定了等待操作的绝对超时。如果 *timeout* 为NULL，则该操作可以无限期阻塞。

忽略 `uaddr2` 参数。

**FUTEX_WAKE_BITSET** (从 Linux 2.6.25 开始支持)
这个操作与 **FUTEX_WAKE** 一样，除了 *val3* 参数用于给内核提供了一个 32 位的位掩码。这个位掩码必须设置至少一个位，用于选择应该唤醒哪个等待者。选择是通过 “唤醒” 位掩码 (即 *val3* 中的值) 和存储在等待者的内核内部状态中的位掩码 (使用 **FUTEX_WAIT_BITSET** 设置的 “等待” 位掩码) 的位与运算来完成的。所有与运算结果为非零值的等待者都被唤醒；剩下的等待者依然休眠。

**FUTEX_WAIT_BITSET** 和 **FUTEX_WAKE_BITSET** 的作用是允许在同一个 futex 上阻塞的多个等待者之间选择性唤醒。然而，请注意，根据使用场景，在 futex 上使用这个位掩码多路复用功能可能比简单地使用多个 futex 效率低，因为使用位掩码多路复用需要内核检查 futex 上的所有等待者，包括那些对被唤醒不感兴趣的等待者 (即，它们没有在它们的 “等待” 位掩码中设置相关的位)。

常量 **FUTEX_BITSET_MATCH_ANY**，对应于位掩码中的所有 32 个位都被设置，可以用作 **FUTEX_WAIT_BITSET** 和 **FUTEX_WAKE_BITSET** 的 *val3* 参数。除了处理 *timeout* 参数的不同之外，**FUTEX_WAIT** 操作等价于 *val3* 指定为 **FUTEX_BITSET_MATCH_ANY** 的 **FUTEX_WAIT_BITSET**；即，允许任何唤醒者的唤醒。**FUTEX_WAKE** 操作等价于 *val3* 指定为 **FUTEX_BITSET_MATCH_ANY** 的 **FUTEX_WAKE_BITSET**；即，唤醒任何等待者。

忽略 `uaddr2` 和 `timeout` 参数。

### 优先级继承 futex

Linux 支持优先级继承 (PI) futex，以便处理普通 futex 锁可能遇到的优先级反转问题。优先级反转是当高优先级任务被阻塞等待获得由低优先级任务持有的锁时发生的问题，而中等优先级的任务不断地从 CPU 抢占低优先级任务。结果，低优先级的任务无法朝着释放锁的方向推进，而高优先级的任务仍然处于阻塞状态。

优先级继承是一种处理优先级反转问题的机制。通过这种机制，当高优先级任务被低优先级任务持有的锁阻塞时，低优先级任务的优先级将暂时提升到高优先级任务的优先级，这样它就不会被任何中等优先级的任务抢占，从而可以朝着释放锁的方向推进。要使优先级继承机制生效，优先级继承必须是传递的，这意味着，如果高优先级的任务阻塞在一个低优先级任务持有的锁上，而低优先级任务本身阻塞在另一个中等优先级任务持有的锁上 (如此这样，形成任意长度的链)，则所有这些任务 (或更一般地，在锁链中的所有任务) 的优先级都提升为与高优先级任务相同。

从用户空间的角度来看，使 futex PI 感知的是用户空间和内核之间关于 futex 字的值的策略协议 (如下所述)，以及下面描述的 PI-futex 操作的使用。(与上面描述的其它 futex 操作不同，PI-futex 操作是为实现非常特定的 IPC 机制而设计的。)

下面描述的 PI-futex 操作与其它 futex 操作的不同之处在于，它们对 futex 字的值的使用施加了策略：

 * 如果未获得锁，则 futex 字的值应为 0。

 * 如果获得锁，futex 字的值应该是所属线程的线程ID (TID；参见 [gettid(2)](https://man7.org/linux/man-pages/man2/gettid.2.html))。

 * 如果锁被获得，并且有线程在争用，那么 futex 字的值应该设置 **FUTEX_WAITERS** 位；换句话说，这个值是：
```
              FUTEX_WAITERS | TID
```

(注意，一个 PI futex 字没有设置所有者和 **FUTEX_WAITERS** 位是无效的。)

通过适当地使用这个策略，用户空间应用程序可以使用在用户模式下执行的原子指令 (例如比较-交换操作，如 x86 架构上的 *cmpxchg*) 来获取未获取的锁或释放锁。获取锁只需要使用比较-交换来原子地将 futex 字的值设置为调用者的 TID，如果它的前一个值为 0 的话。如果前一个值是预期的 TID，则释放锁即使用比较-交换将 futex 字的值设置为0。

如果一个 futex 已经被获取 (也就是说，有一个非零值)，等待者必须使用 **FUTEX_LOCK_PI** 或 **FUTEX_LOCK_PI2** 操作来获取锁。如果其它线程正在等待锁，则应该设置 futex 值中的 **FUTEX_WAITERS** 位；这种情况下，锁的拥有者必须使用 **FUTEX_UNLOCK_PI** 操作释放锁。

在调用者被迫进入内核的情况下 (即，必须执行 **futex()** 调用)，它们然后直接处理所谓的 RT-mutex，这是一种实现了所需的优先级继承语义的内核锁定机制。在获得 RT-mutex 之后，在调用线程返回用户空间之前，将相应地更新 futex 的值。

需要注意的是，内核将在返回用户空间之前更新 futex 字的值。(这可以防止 futex 字的值以无效状态结束的可能性，比如具有所有者但值为 0，或者具有等待者但没有设置 **FUTEX_WAITERS** 位。)

如果 futex 在内核中有一个与之关联的 RT-mutex (即，有阻塞的等待者)，且 futex/RT-mutex 的所有者意外死亡，则内核清理 RT-mutex 并把它交接给下一个等待者。这反过来要求相应地更新用户空间值。为了表明这是必需的，内核将 futex 字中的 **FUTEX_OWNER_DIED** 位与新所有者的线程 ID 一起设置。用户空间可以通过 **FUTEX_OWNER_DIED** 位检测到这种情况，然后负责清理死亡所有者留下的陈旧状态。

通过在 *futex_op* 中指定下面列出的值之一来操作 PI futex。请注意，PI futex 操作必须作为成对操作使用，并受到一些额外要求的约束：

 * **FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，和 **FUTEX_TRYLOCK_PI** 与 **FUTEX_UNLOCK_PI** 配对。**FUTEX_UNLOCK_PI** 必须在调用线程拥有的 futex 上调用，如值策略所定义，否则会导致错误 EPERM。

 * **FUTEX_WAIT_REQUEUE_PI** 与 **FUTEX_CMP_REQUEUE_PI** 配对。这必须从一个非 PI futex 执行到一个不同的 PI futex (否则报错误 EINVAL 结果)。此外，*val* (要唤醒的等待者的个数) 必须为 1 (否则报错误 EINVAL 结果)。

PI futex 的操作如下：

**FUTEX_LOCK_PI** (从 Linux 2.6.18 开始支持)
这个操作在尝试通过用户模式的原子指令，由于 futex 字具有非 0 值而获得锁失败之后使用 - 具体地说，由于它包含锁拥有者的 TID (PID-namespace-specific)。

这个操作检查位于地址 *uaddr* 的 futex 字的值。如果值为 0，则内核尝试原子地设置 futex 的值为调用者的 TID。如果 futex 字的值非 0，内核原子地设置 **FUTEX_WAITERS** 位，这通知 futex 拥有者，它不能在用户空间通过原子地将 futex 值设置为 0 来解锁。之后，内核将：

(1). 尝试找到与拥有者 TID 关联的线程。

(2). 代表所有者创建或重用内核状态。(如果这是第一个等待者，则这个 futex 还没有内核状态，于是通过锁定 RT-mutex 创建内核状态，并使 futex 所有者成为 RT-mutex 的所有者。如果已经有了等待者，则重用已有的状态。)

(3). 把等待者绑在 futex 上 (即，将等待者入队 RT-mutex 的等待者列表)。

如果存在多个等待者，则等待者的入队按优先级降序排列。(关于优先级排序的信息，参考 [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html) 中 **SCHED_DEADLINE**，**SCHED_FIFO**，和 **SCHED_RR** 调度策略的讨论。) 拥有者继承等待者的 CPU 带宽 (如果等待者以 **SCHED_DEADLINE** 策略调度) 或等待者的优先级 (如果等待者以 **SCHED_RR** 或 **SCHED_FIFO** 策略调度)。在嵌套锁定的情况下此继承遵循锁链，并执行死锁检测。

*timeout* 参数为锁定尝试提供了超时值。如果 *timeout* 不是 NULL，则它指向的结构指定了绝对超时，根据 **CLOCK_REALTIME** 时钟测量。如果 *timeout* 是 NULL，则该操作将无限期阻塞。

*uaddr2*，*val*，和 *val3* 参数被忽略。

**FUTEX_LOCK_PI2** (从 Linux 5.14 开始支持)
这个操作与 **FUTEX_LOCK_PI** 相同，除了测量 *timeout* 所依据的时钟是可选的。默认情况下，*timeout* 中指定的 (绝对) 超时根据 **CLOCK_MONOTONIC** 时钟测量，但如果在 *futex_op* 中指定了 **FUTEX_CLOCK_REALTIME** 标记，则超时将根据 **CLOCK_REALTIME** 时钟测量。

**FUTEX_TRYLOCK_PI** (从 Linux 2.6.18 开始支持)
这个操作尝试获得位于 *uaddr* 的锁。当用户空间原子的获取由于 futex 字不是 0 而没有成功时调用它。

由于相对于用户空间，内核有权限访问更多状态信息，在 futex 字 (即，状态信息对于用户空间可访问) 包含陈旧状态 (**FUTEX_WAITERS** 和/或 **FUTEX_OWNER_DIED**) 的情况下，如果由内核执行，获取锁可能会成功。这可能发生在 futex 的所有者死亡时。用户空间无法以无竞争的方式处理这种情况，但内核可以修复它并获得锁。

*uaddr2*，*val*，*timeout*，和 *val3* 参数被忽略。

**FUTEX_UNLOCK_PI** (从 Linux 2.6.18 开始支持)
这个操作唤醒在 *uaddr* 参数提供的地址处的 futex 上，等待在 **FUTEX_LOCK_PI** 或 **FUTEX_LOCK_PI2** 中的最高优先级的等待者。

当位于 *uaddr* 的用户空间值无法原子地从一个 (所有者的) TID 变为 0 时调用。

*uaddr2*，*val*，*timeout*，和 *val3* 参数被忽略。

**FUTEX_CMP_REQUEUE_PI** (从 Linux 2.6.31 开始支持)
这个操作是 **FUTEX_CMP_REQUEUE** 的 PI 变体。它将通过 **FUTEX_WAIT_REQUEUE_PI** 阻塞在 *uaddr* 的等待者从一个非 PI 源 futex (*uaddr*) 重新入队到一个 PI 目标 futex (*uaddr2*)。

与 **FUTEX_CMP_REQUEUE** 一样，这个操作唤醒等待在位于 *uaddr* 处的 futex 上的最多 *val* 个等待者。然而，对于 **FUTEX_CMP_REQUEUE_PI**，*val* 必须是 1 (因为主要是为了避免惊群效应)。剩余的等待者从位于 *uaddr* 处的源 futex 的等待队列移除，并添加进位于 *uaddr2* 处的目标 futex 的等待队列中。

*val2* 和 *val3* 参数服务于与 **FUTEX_CMP_REQUEUE** 中的对应参数相同的目的。

**FUTEX_WAIT_REQUEUE_PI** (从 Linux 2.6.31 开始支持)
等待在位于 *uaddr* 处的非 PI futex 上，并可能重新入队 (通过另一个任务中的 **FUTEX_CMP_REQUEUE_PI** 操作) 在位于 *uaddr2* 处的 PI futex 上。在 *uaddr* 上的等待操作与 **FUTEX_WAIT** 相同。

等待者可能通过另一个任务中的 **FUTEX_WAIT** 操作，从 *uaddr* 上的等待移除，而没有入队 *uaddr2* 上。在这种情况下，**FUTEX_WAIT_REQUEUE_PI** 操作以错误 **EAGAIN** 失败。

如果 *timeout* 不是 NULL，则它指向的结构指定了等待操作的绝对超时值。如果 *timeout* 是 NULL，则该操作可能无限期阻塞。

忽略 `val3` 参数。

添加 **FUTEX_WAIT_REQUEUE_PI** 和 **FUTEX_CMP_REQUEUE_PI** 以支持一个相当特定的使用场景：支持优先级继承的 POSIX 线程条件变量。想法是这些操作应该总是配对的，以确保用户空间和内核保持同步。这样，在 **FUTEX_WAIT_REQUEUE_PI** 操作中，用户空间应用程序预指定，在 **FUTEX_CMP_REQUEUE_PI** 操作中发生的重新入队的目标。

## 返回值
发生错误时 (假设通过 [syscall(2)](https://man7.org/linux/man-pages/man2/syscall.2.html) 调用 **futex**())，所有的操作返回 -1，并设置 *[errno](https://man7.org/linux/man-pages/man3/errno.3.html)* 来指示错误。

成功时的返回值依赖于具体的操作，如下面的列表所述：

**FUTEX_WAIT**
如果调用者被唤醒，则返回0。注意，唤醒也可能是由之前碰巧使用过 futex 字的内存位置的不相关代码中的常见 futex 使用模式引起的 (例如，典型的基于 futex 的 Pthreads 互斥锁实现在某些情况下会导致这种情况)。因此，调用者应该始终保守地假设返回值为 0 可能意味着虚假唤醒，并使用 futex 字的值 (即用户空间同步方案) 来决定是否继续阻塞。

**FUTEX_WAKE**
返回被唤醒的等待者的个数。

**FUTEX_FD**
返回与 futex 关联的新文件描述符。

**FUTEX_REQUEUE**
返回被唤醒的等待者的个数。

**FUTEX_CMP_REQUEUE**
返回位于 *uaddr2* 的 futex 字的被唤醒或重新入队到 futex 的等待者的个数。如果这个值比 *val* 大，则差值是位于 *uaddr2* 的 futex 字的重新入队到 futex 的等待者的个数。

**FUTEX_WAKE_OP**
返回被唤醒的等待者的总个数。这是位于 *uaddr* 和 *uaddr2* 的 futex 字的两个 futex 上被唤醒的等待者的总和。

**FUTEX_WAIT_BITSET**
如果调用者被唤醒则返回 0。参考 **FUTEX_WAIT** 来了解在实践中如何正确地解释它。

**FUTEX_WAKE_BITSET**
返回被唤醒的等待者的个数。

**FUTEX_LOCK_PI**
如果成功锁定 futex 则返回 0。

**FUTEX_LOCK_PI2**
如果成功锁定 futex 则返回 0。

**FUTEX_TRYLOCK_PI**
如果成功锁定 futex 则返回 0。

**FUTEX_UNLOCK_PI**
如果成功解锁 futex 则返回 0。

**FUTEX_CMP_REQUEUE_PI**
返回位于 *uaddr2* 的 futex 字的被唤醒或重新入队到 futex 的等待者的个数。如果这个值比 *val* 大，则差值是位于 *uaddr2* 的 futex 字的重新入队到 futex 的等待者的个数。

**FUTEX_WAIT_REQUEUE_PI**
如果调用者成功入队到位于 *uaddr2* 的 futex 字的 futex 则返回 0。

## 错误

**EACCES** 没有对 futex 字的内存的读权限。

**EAGAIN** (**FUTEX_WAIT**，**FUTEX_WAIT_BITSET**， **FUTEX_WAIT_REQUEUE_PI**) 调用时，`uaddr` 指向的值不等于期望值 `val`。

**注意**：在 Linux 上，符号名 **EAGAIN** 和 **EWOULDBLOCK** (它们出现在内核 futex 代码的不同部分) 具有相同的值。

**EAGAIN** (**FUTEX_CMP_REQUEUE**，**FUTEX_CMP_REQUEUE_PI**) `uaddr` 指向的值不等于期望值 `val3`。

**EAGAIN** (**FUTEX_LOCK_PI**，**FUTEX_TRYLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**) `uaddr` (对于 **FUTEX_CMP_REQUEUE_PI** 是：`uaddr2`) 的 futex 所有者线程 ID 即将退出，但还没有处理内部的状态清理。再次尝试。

**EDEADLK** (**FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，**FUTEX_TRYLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**) 位于 `uaddr` 的 futex 字已经被调用者锁定。

**EDEADLK** (**FUTEX_CMP_REQUEUE_PI**) 将一个等待者重新入队位于 `uaddr2` 的 futex 字的 PI futex 时，内核检测到了死锁。

**EFAULT** 必须的指针参数 (如 *uaddr*，*uaddr2*，或 *timeout*) 没有指向一个有效的用户空间地址。

**EINTR** **FUTEX_WAIT** 或 **FUTEX_WAIT_BITSET** 操作被信号 (参见 [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)) 中断。在 Linux 2.6.22 之前，这个错误也可能因为虚假唤醒而返回；从 Linux 2.6.22 开始，这种情况不再发生。

**EINVAL** *futex_op* 中的操作是使用了超时值的操作中的一个，但提供的 *timeout* 参数是无效的 (*tv_sec* 小于 0，或 *tv_nsec* 不小于 1,000,000,000)。

**EINVAL** *futex_op* 中指定的操作使用了 *uaddr* 和 *uaddr2* 中的一个或两个指针，但它们中的一个没有指向一个有效的对象 —— 即，地址不是四字节对齐的。

**EINVAL** (**FUTEX_WAIT_BITSET**，**FUTEX_WAKE_BITSET**) *val3* 中提供的位掩码是 0。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) *uaddr* 等于 *uaddr2* (比如，尝试重新入队到相同的 futex)。

**EINVAL** (**FUTEX_FD**) *val* 中提供的信号号是无效的。

**EINVAL** (**FUTEX_WAKE**，**FUTEX_WAKE_OP**，**FUTEX_WAKE_BITSET**，**FUTEX_REQUEUE**，**FUTEX_CMP_REQUEUE**) 内核探测到了一个位于 *uaddr* 的用户空间状态和内核状态的不一致 —— 即，它探测到 *uaddr* 上有等待者等待在 **FUTEX_LOCK_PI** 或 **FUTEX_LOCK_PI2** 中。

**EINVAL** (**FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，**FUTEX_TRYLOCK_PI**，**FUTEX_UNLOCK_PI**) 内核探测到了一个位于 *uaddr* 的用户空间状态和内核状态的不一致。这表示发生了状态损坏，或内核发现了一个 *uaddr* 上的等待者通过 **FUTEX_WAIT** 或 **FUTEX_WAIT_BITSET** 在等待。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) 内核探测到了一个位于 *uaddr2* 的用户空间状态和内核状态的不一致；即，内核探测到了一个 *uaddr2* 上的等待者通过 **FUTEX_WAIT** 或 **FUTEX_WAIT_BITSET** 在等待。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) 内核探测到了一个位于 *uaddr* 的用户空间状态和内核状态的不一致；即，内核探测到了一个 *uaddr* 上的等待者通过 **FUTEX_WAIT** 或 **FUTEX_WAIT_BITSET** 在等待。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) 内核探测到了一个位于 *uaddr* 的用户空间状态和内核状态的不一致；即，内核探测到了一个通过 **FUTEX_LOCK_PI** 或 **FUTEX_LOCK_PI2** (而不是 **FUTEX_WAIT_REQUEUE_PI**) 等待在 *uaddr* 上的等待者。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) 尝试将一个等待者重新入队到的 futex，不是该等待者匹配的 **FUTEX_WAIT_REQUEUE_PI** 调用指定的那个。

**EINVAL** (**FUTEX_CMP_REQUEUE_PI**) *val* 参数不是 1。

**EINVAL** 无效参数。

**ENFILE** (**FUTEX_FD**) 已达到打开文件总数的全系统限制。

**ENOMEM** (**FUTEX_LOCK_PI**，**FUTEX_TRYLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**) 内核无法分配内存来保存状态信息。

**ENOSYS** *futex_op* 中指定的操作无效。

**ENOSYS** *futex_op* 中指定了 *FUTEX_CLOCK_REALTIME* 选项，但伴随的操作不是 **FUTEX_WAIT**，**FUTEX_WAIT_BITSET**，**FUTEX_WAIT_REQUEUE_PI**，或 **FUTEX_LOCK_PI2**。

**ENOSYS** (**FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，**FUTEX_TRYLOCK_PI**，**FUTEX_UNLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**，**FUTEX_WAIT_REQUEUE_PI**) 运行时检查确认操作不可用。PI-futex 操作不是在所有的架构上都实现了的，且在一些 CPU 变体上不支持。

**EPERM** (**FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，**FUTEX_TRYLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**) 调用者不被允许把它自己连接到位于 *uaddr* 的 futex (对于 **FUTEX_CMP_REQUEUE_PI**：futex 位于 *uaddr2*)。(这可能是由于用户空间的状态损坏引起的)

**EPERM**  (**FUTEX_UNLOCK_PI**) 调用者不拥有 futex 字表示的锁。

**ESRCH** (**FUTEX_LOCK_PI**，**FUTEX_LOCK_PI2**，**FUTEX_TRYLOCK_PI**，**FUTEX_CMP_REQUEUE_PI**) 位于 *uaddr* 处的 futex 字中的线程 ID 不存在。

**ESRCH** (**FUTEX_CMP_REQUEUE_PI**) 位于 *uaddr2* 处的 futex 字中的线程 ID 不存在。

**ETIMEDOUT** *futex_op* 中的操作使用了 *timeout* 中指定的超时值，但在操作完成之前超时值到期。

## 版本

Futex 最初是在 Linux 2.6.0 的稳定内核版本中提供的。

最初的 futex 支持合入了 Linux 2.5.7，但它具有与这里描述的不同的语义。具有这里描述的语义的 4 参数系统调用是在 Linux 2.5.40 中引入的。第 5 个参数是在 Linux 2.5.70 中添加的，第 6 个参数是在 Linux 2.6.7 中添加的。

## 兼容于

这个系统调用是 Linux 特有的。

## 注意

Glibc 没有提供这个系统调用的包装器；使用 **syscall**(2) 来调用它。

一些高级编程抽象通过 futex 实现，包括 POSIX 信号量和各种 POSIX 线程同步机制 (互斥量，条件变量，读写锁，和屏障)。

## 示例

下面的程序演示了如何在程序中使用 futex，其中有一个父进程和一个子进程，它们使用位于共享匿名内存映射中的一对 futex 同步对共享资源的访问：终端。这两个进程中的每个进程都向终端写入 nloops (命令行参数，没有指定时默认取 5) 条消息，并使用一个同步协议以确保它们交替地写消息。运行这个程序时，我们将看到如下的输出：

```
           $ ./futex_demo
           Parent (18534) 0
           Child  (18535) 0
           Parent (18534) 1
           Child  (18535) 1
           Parent (18534) 2
           Child  (18535) 2
           Parent (18534) 3
           Child  (18535) 3
           Parent (18534) 4
           Child  (18535) 4
```

### 程序源码
```
       /* futex_demo.c

          Usage: futex_demo [nloops]
                           (Default: 5)

          Demonstrate the use of futexes in a program where parent and child
          use a pair of futexes located inside a shared anonymous mapping to
          synchronize access to a shared resource: the terminal. The two
          processes each write 'num-loops' messages to the terminal and employ
          a synchronization protocol that ensures that they alternate in
          writing messages.
       */
       #define _GNU_SOURCE
       #include <stdio.h>
       #include <errno.h>
       #include <stdatomic.h>
       #include <stdlib.h>
       #include <unistd.h>
       #include <sys/wait.h>
       #include <sys/mman.h>
       #include <sys/syscall.h>
       #include <linux/futex.h>
       #include <sys/time.h>

       #define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                               } while (0)

       static int *futex1, *futex2, *iaddr;

       static int
       futex(int *uaddr, int futex_op, int val,
             const struct timespec *timeout, int *uaddr2, int val3)
       {
           return syscall(SYS_futex, uaddr, futex_op, val,
                          timeout, uaddr, val3);
       }

       /* Acquire the futex pointed to by 'futexp': wait for its value to
          become 1, and then set the value to 0. */

       static void
       fwait(int *futexp)
       {
           int s;

           /* atomic_compare_exchange_strong(ptr, oldval, newval)
              atomically performs the equivalent of:

                  if (*ptr == *oldval)
                      *ptr = newval;

              It returns true if the test yielded true and *ptr was updated. */

           while (1) {

               /* Is the futex available? */
               const int one = 1;
               if (atomic_compare_exchange_strong(futexp, &one, 0))
                   break;      /* Yes */

               /* Futex is not available; wait */

               s = futex(futexp, FUTEX_WAIT, 0, NULL, NULL, 0);
               if (s == -1 && errno != EAGAIN)
                   errExit("futex-FUTEX_WAIT");
           }
       }

       /* Release the futex pointed to by 'futexp': if the futex currently
          has the value 0, set its value to 1 and the wake any futex waiters,
          so that if the peer is blocked in fpost(), it can proceed. */

       static void
       fpost(int *futexp)
       {
           int s;

           /* atomic_compare_exchange_strong() was described in comments above */

           const int zero = 0;
           if (atomic_compare_exchange_strong(futexp, &zero, 1)) {
               s = futex(futexp, FUTEX_WAKE, 1, NULL, NULL, 0);
               if (s  == -1)
                   errExit("futex-FUTEX_WAKE");
           }
       }

       int
       main(int argc, char *argv[])
       {
           pid_t childPid;
           int j, nloops;

           setbuf(stdout, NULL);

           nloops = (argc > 1) ? atoi(argv[1]) : 5;

           /* Create a shared anonymous mapping that will hold the futexes.
              Since the futexes are being shared between processes, we
              subsequently use the "shared" futex operations (i.e., not the
              ones suffixed "_PRIVATE") */

           iaddr = mmap(NULL, sizeof(int) * 2, PROT_READ | PROT_WRITE,
                       MAP_ANONYMOUS | MAP_SHARED, -1, 0);
           if (iaddr == MAP_FAILED)
               errExit("mmap");

           futex1 = &iaddr[0];
           futex2 = &iaddr[1];

           *futex1 = 0;        /* State: unavailable */
           *futex2 = 1;        /* State: available */

           /* Create a child process that inherits the shared anonymous
              mapping */

           childPid = fork();
           if (childPid == -1)
               errExit("fork");

           if (childPid == 0) {        /* Child */
               for (j = 0; j < nloops; j++) {
                   fwait(futex1);
                   printf("Child  (%ld) %d\n", (long) getpid(), j);
                   fpost(futex2);
               }

               exit(EXIT_SUCCESS);
           }

           /* Parent falls through to here */

           for (j = 0; j < nloops; j++) {
               fwait(futex2);
               printf("Parent (%ld) %d\n", (long) getpid(), j);
               fpost(futex1);
           }

           wait(NULL);

           exit(EXIT_SUCCESS);
       }
```

## 另请参阅

**get_robust_list**(2)，**restart_syscall**(2)，**pthread_mutexattr_getprotocol**(3)，**futex**(7)，**sched**(7)。

下面的内核源文件：
  * Documentation/pi-futex.txt

  * Documentation/futex-requeue-pi.txt

  * Documentation/locking/rt-mutex.txt

  * Documentation/locking/rt-mutex-design.txt

  * Documentation/robust-futex-ABI.txt

Franke, H., Russell, R., and Kirwood, M., 2002.  Fuss, **Futexes and Furwocks: Fast Userlevel Locking in Linux** (from proceedings of the Ottawa Linux Symposium 2002), ⟨http://kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf⟩

Hart, D., 2009. **A futex overview and update**, ⟨http://lwn.net/Articles/360699/⟩

Hart, D. and Guniguntala, D., 2009.  **Requeue-PI: Making Glibc Condvars PI-Aware** (from proceedings of the 2009 Real-Time Linux Workshop), ⟨http://lwn.net/images/conf/rtlws11/papers/proc/p10.pdf⟩

Drepper, U., 2011. **Futexes Are Tricky**, ⟨http://www.akkadia.org/drepper/futex.pdf⟩

Futex 示例库，**futex-\*.tar.bz2 at ⟨ftp://ftp.kernel.org/pub/linux/kernel/people/rusty/⟩**

COLOPHON
       This page is part of release 5.10 of the Linux man-pages project.  A description of the project, information about reporting bugs, and the latest version of this page, can be found at https://www.kernel.org/doc/man-pages/.

[原文](https://man7.org/linux/man-pages/man2/futex.2.html)

Done.
