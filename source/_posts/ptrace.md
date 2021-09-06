---
title: ptrace 系统调用
date: 2021-09-05 11:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

ptrace 是 Linux 环境下，许许多多分析调试工具，如 `strace`，`ltrace`，特别是 `gdb` 等，所使用的最核心的系统调用。ptrace，即 process trace，指进程追踪。它能让一个进程控制及影响另一个进程的行为，比如让另外一个进程停止运行，获取另外一个进程内存地址空间中的值，设置另外一个进程内存中的值，获取另外一个进程的寄存器的值，设置另外一个进程的寄存器的值，等等等等。
<!--more-->
ptrace 系统调用的 C 库接口原型如下：
```
       #include <sys/ptrace.h>

       long ptrace(enum __ptrace_request request, pid_t pid,
                   void *addr, void *data);
```

后面更详细地描述 ptrace 系统调用。

## 描述

**`ptrace()`** 提供了一种方法，让一个进程（称为 "tracer"）可以观测及控制另外一个进程（称为 "tracee"）的执行，让 tracer 检测和修改 tracee 的内存和寄存器。它主要用于实现断点调试和系统调用追踪，前者即 GDB 这种调试器，后者即 strace 等系统调用追踪工具。

Tracee 首先需要被 attached 到 tracer 上。Attachment 和后续的命令是针对特定线程的：在一个多线程进程中，每个线程可以被单独 attached 到一个（有可能不同的） tracer，或者不被 attached，即不被调试。因此，"tracee" 总是意味着 "(一个) 线程"，而不是 "一个 (可能是多线程的) 进程"。Ptrace 命令总是用下面形式的调用发送给一个特定的 tracee：
```
    ptrace(PTRACE_foo, pid, ...)
```
其中 *pid* 是对应 Linux 线程的线程 ID。

（注意，在本页中，一个 “多线程进程” 意味着使用 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 的 **CLONE_THREAD** 标记创建的线程构成的线程组。 ）

进程可以通过调用 [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)，并让生成的子进程执行 **PTRACE_TRACEME**，以初始化一个追踪，然后（典型地）执行一个 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)。另外，一个进程可以使用 **PTRACE_ATTACH** 或 **PTRACE_SEIZE** 开启追踪另一个进程。

在被追踪时，tracee 将会在每次传递信号时停下来，即使信号被忽略。（**信号的具体含义是什么？**）（一个例外是 **SIGKILL**，它的行为就像通常的那样。）***信号既包括通过用 kill 工具或系统同名系统调用向进程发送的信号，也包括系统调用开始或结束这样的事件。*** Tracer 将在它下次调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 时得到通知（或一个与 "wait" 相关的系统调用）；那个调用将返回一个 *status* 值，其中包含了指示 tracee 断下来的原因的信息。当 tracee 停止时，tracer 可以使用各种 ptrace 请求调查和修改 tracee。随后 tracer 可以让 tracee 继续执行，或者也可以忽略传递过来的信号（或者甚至可以传递一个不同的信号给 tracee）。

如果 **PTRACE_O_TRACEEXEC** 选项没有设置，被追踪的进程所有对于 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 的成功调用将使得它发送一个 **SIGTRAP ** 信号，这给了父进程一个在新程序开始执行前获得控制权的机会。

当 tracer 结束了追踪，它可以通过 **PTRACE_DETACH** 使 tracee 继续以正常的、未被追踪的模式执行。

**从 API 设计的层面来看，ptrace 就像是用一个 API 向用户暴露了一个库的能力一样。除了需要用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 获得被追踪的 tracee 进程的信号通知之外，其它的追踪功能基本上都可以通过 ptrace 系统调用完成。**

*request* 的值决定了要执行的行为：

**PTRACE_TRACEME**
 > 表示这个进程将被它的父进程追踪。如果父进程不希望追踪它的话，则进程可能不应该发起这个请求。（*pid*，*addr*，和 *data* 被忽略。）
 > **PTRACE_TRACEME** 请求只被用于 tracee；其余的请求只被用于 tracer。在下面的请求中，*pid* 指定了将在其上执行的 tracee 的 线程 ID。除 **PTRACE_ATTACH**，**PTRACE_SEIZE**，**PTRACE_INTERRUPT**，和 **PTRACE_KILL** 请求外，tracee ***必须被停下来***。

> ----------------------------------------------------------------------------------------------------------
> *被追踪的进程不调用 **PTRACE_TRACEME** 的话，其它进程是否可以对它成功调用 **PTRACE_ATTACH** 和 **PTRACE_SEIZE** 等？还是说这个请求主要被用于协调被追踪的进程和其父进程的行动？* 
> Tracee 进程在执行这个调用时不会停下来等待父进程 attach，即 tracee 进程在执行这个调用之后，会直接执行后面的语句。

**PTRACE_PEEKTEXT**，**PTRACE_PEEKDATA**
 > 在 tracee 的内存中的地址 *addr* 处读取一个字，返回该字的值作为 **ptrace()** 调用的返回值。Linux 没有单独的 text 和 data 地址空间，因此这两个请求目前是等价的。（*data* 被忽略；请参考 NOTES。）

**PTRACE_PEEKUSER**
 > 在 tracee 的 USER 区域的 *addr* 偏移处读取一个字，USER 区域包含关于进程的寄存器和其它信息（请参考 *<sys/user.h>*）。字的值被作为 **ptrace()** 调用的结果返回。典型地，偏移量必须是字对齐的，尽管这可能因架构不同而异。请参考 NOTES。（*data* 被忽略；请参考 NOTES。）

> ----------------------------------------------------------------------------------------------------------
> USER 区域是内核维护的一段特殊内存区域，其中保存了进程的寄存器值等信息。这个调用和 PTRACE_GETREGS 有什么区别？两者的功能是否有一定的重合？这个 USER 区域的格式在什么地方定义？或者说，我怎么知道可以在哪个便宜处获得什么值呢？

> ----------------------------------------------------------------------------------------------------------
> USER 区域的格式定义，如 *<sys/user.h>* 文件中定义的 `struct user` 。

**PTRACE_POKETEXT**，**PTRACE_POKEDATA**
 > 把字 *data* 拷贝到 tracee 的内存中的地址 *addr* 处。就像 **PTRACE_PEEKTEXT** 和 **PTRACE_PEEKDATA** 那样，这两个请求目前也是等价的。

**PTRACE_POKEUSER**
 > 把字 *data* 拷贝到 tracee 的 USER 区域的偏移量 *addr* 处。就像 **PTRACE_PEEKUSER** 那样，典型情况下，偏移量必须是字对齐的。为了维护内核的完整性，对于 USER 区域的一些修改是不被允许的。

**PTRACE_GETREGS**，**PTRACE_GETFPREGS**
 > 分别拷贝 tracee 的通用或浮点寄存器到 tracer 中的地址 *data* 处。请参考 *<sys/user.h>* 来了解关于这个数据的格式的信息。（*addr* 被忽略。）注意 SPARC 系统中，*data* 和 *addr* 的含义不同；即 *data* 被忽略，寄存器被拷贝到地址 *addr* 处。**PTRACE_GETREGS** 和 **PTRACE_GETFPREGS** 并不是存在于所有的体系结构中的。

**PTRACE_GETREGSET** (自 Linux 2.6.34 始)
 > 读取 tracee 的寄存器。*addr* 以一种机器相关的方式指定要读取的寄存器的类型。**NT_PRSTATUS** （具有数字值 1）通常指定读取通用寄存器。如果 CPU 有，比如，浮点和/或向量寄存器，它们可以通过把 *addr* 设置为对应的 **NT_foo** 常量来提取。*data* 指向一个 *struct iovec*，其描述了目标缓冲区的位置和长度。返回时，内核修改 **iov.len** 来表示实际返回的字节数。

**PTRACE_SETREGS**，**PTRACE_SETFPREGS**
 > 分别用 tracer 中地址 *data* 处的数据修改 tracee 的通用或浮点寄存器。至于 **PTRACE_POKEUSER** ，一些通用寄存器修改可能是不被允许的。（*addr* 被忽略。）注意 SPARC 系统中 *data* 和 *addr* 参数的含义是相反的；即 *data* 被忽略，而寄存器的值是从地址 *addr* 处拷贝而来的。**PTRACE_SETREGS** 和 **PTRACE_SETFPREGS** 并不是存在于所有的体系结构中的。

**PTRACE_SETREGSET** (自 Linux 2.6.34 始)
 > 修改 tracee 的寄存器。*data* 和 *addr* 参数的含义与 **PTRACE_GETREGSET** 的相同。

**PTRACE_GETSIGINFO** (自 Linux 2.3.99-pre6 始)
 > 提取导致进程停止的信号的信息。把一个 *siginfo_t* 结构（请参考 [sigaction(2)](https://man7.org/linux/man-pages/man2/sigaction.2.html)）从 tracee 拷贝到 tracer 的地址 *data* 处。（*addr* 被忽略。）

**PTRACE_SETSIGINFO** (自 Linux 2.3.99-pre6 始)
 > 设置信号信息：把一个 *siginfo_t* 结构从 tracer 的地址 *data* 处拷贝到 tracee 中。这只会影响到那些通常将被传送给 tracee，但被 tracer 捕获的信号。从 **ptrace()** 本身生成的合成信号中区分出这些普通的信号可能比较困难。（*addr* 被忽略。）

**PTRACE_PEEKSIGINFO** (自 Linux 3.10 始)
 > 提取 *siginfo_t* 结构而不将信号从队列中移除。*addr* 指向一个 *ptrace_peeksiginfo_args* 结构，后者描述了拷贝的信号应该开始的顺序位置，要拷贝的信号的个数。*siginfo_t* 结构被拷贝到 *data* 指向的缓冲区。返回值包含拷贝的信号的个数（零表示没有信号对应于指定的顺序位置）。在返回的 *siginfo_t* 结构中，*si_code* 字段包含没有公开给用户空间的信息 (__SI_CHLD, __SI_FAULT, 等等)。
```
           struct ptrace_peeksiginfo_args {
               u64 off;    /* Ordinal position in queue at which
                              to start copying signals */
               u32 flags;  /* PTRACE_PEEKSIGINFO_SHARED or 0 */
               s32 nr;     /* Number of signals to copy */
           };
```
 > 目前，只有一个标记， **PTRACE_PEEKSIGINFO_SHARED**，用于从进程范围的信号队列转储信号。如果没有设置这个标记，将从指定线程的每个线程队列读取信号。

**PTRACE_GETSIGMASK** (自 Linux 3.11 始)
 > 把阻塞的信号的掩码（参考 [sigprocmask(2)](https://man7.org/linux/man-pages/man2/sigprocmask.2.html)）的一份拷贝放进 *data* 指向的缓冲区中，后者应该是一个指向 *sigset_t* 类型缓冲区的指针。*addr* 参数包含了 *data* 指向的缓冲区的大小（比如，*sizeof(sigset_t)*）。

**PTRACE_SETSIGMASK** (自 Linux 3.11 始)
 > 修改阻塞的信号的掩码（参考 [sigprocmask(2)](https://man7.org/linux/man-pages/man2/sigprocmask.2.html)）为 *data* 指向的缓冲区中指定的值，后者应该是一个指向 *sigset_t* 类型缓冲区的指针。*addr* 参数包含了 *data* 指向的缓冲区的大小（比如，*sizeof(sigset_t)*）。

**PTRACE_SETOPTIONS** (自 Linux 2.4.6 始；有关注意事项，请参阅BUGS)
 > 从 *data* 设置 ptrace 选项。（*addr* 被忽略。）*data* 被解释为选项的位掩码，选项由下面的标记指定：

   * **PTRACE_O_EXITKILL** (自 Linux 3.8 始)
      如果 tracer 退出就给 tracee 发送一个 **SIGKILL** 信号。这个选项对于想要确保 tracee 永远无法逃脱 tracer 控制的 ptrace jailers 是有用的。

   * **PTRACE_O_TRACECLONE** (自 Linux 2.5.46 始)
      在下一次调用 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 时停止 tracee，并自动地启动追踪新创建的进程，它将以一个 **SIGSTOP** 信号启动，或者如果使用了 **PTRACE_SEIZE** 则以 **PTRACE_EVENT_STOP**。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_CLONE<<8))` 。新进程的 PID 可以通过 **PTRACE_GETEVENTMSG** 提取。
      这个选项可能无法在所有情况下捕获 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 调用。如果 tracee 以 **CLONE_VFORK** 标记调用 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) ，则如果 **PTRACE_O_TRACEVFORK** 被设置将传递 **PTRACE_EVENT_VFORK** 来替代；否则如果 tracee 以退出信号设置为 SIGCHLD 调用 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html)，则如果 **PTRACE_O_TRACEFORK** 被设置将传递 **PTRACE_EVENT_FORK**。

   * **PTRACE_O_TRACEEXEC** (自 Linux 2.5.46 始)
      在下一次调用 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 时停止 tracee。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_EXEC<<8))` 。如果执行线程不是线程组的 leader，则在停止前线程 ID 将被重置为线程组的 leader 的 ID。自 Linux 3.0 开始，前者的线程 ID 可以通过 **PTRACE_GETEVENTMSG** 提取。

   * **PTRACE_O_TRACEEXIT** (自 Linux 2.5.60 始)
      在 tracee 退出时停止它。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_EXIT<<8))` 。Tracee 的退出状态可以通过 **PTRACE_GETEVENTMSG** 提取。
      Tracee 在进程退出时提前停止，当寄存器依然可用时，允许 tracer 查看退出发生在哪里，然而正常的退出通知将在进程完成退出之后完成。即使上下文依然可用，tracer 依然不能阻止退出发生在这个点。

   * **PTRACE_O_TRACEFORK** (自 Linux 2.5.46 始)
      在下一次调用 [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html) 时停止 tracee，并自动地启动追踪新 forked 的进程，它将以一个 **SIGSTOP** 信号启动，或者如果使用了 **PTRACE_SEIZE** 则以 **PTRACE_EVENT_STOP**。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_FORK<<8))` 。新进程的 PID 可以通过 **PTRACE_GETEVENTMSG** 提取。

   * **PTRACE_O_TRACESYSGOOD** (自 Linux 2.4.6 始)
      在传递系统调用陷入时，设置信号值的位 7（比如传递 *SIGTRAP|0x80*）。这可以让 tracer 非常容易地从系统调用导致的陷入中区分出普通的陷入。

   * **PTRACE_O_TRACEVFORK** (自 Linux 2.5.46 始)
      在下一次调用 [vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html) 时停止 tracee，并自动地启动追踪新 vforked 的进程，它将以一个 **SIGSTOP** 信号启动，或者如果使用了 **PTRACE_SEIZE** 则以 **PTRACE_EVENT_STOP**。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_VFORK<<8))` 。新进程的 PID 可以通过 **PTRACE_GETEVENTMSG** 提取。

   * **PTRACE_O_TRACEVFORKDONE** (自 Linux 2.5.60 始)
      在 tracee 完成下一次 [vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html) 时停止它。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_VFORK_DONE<<8))` 。新进程的 PID 可以（自 Linux 2.6.18 始）通过 **PTRACE_GETEVENTMSG** 提取。

   * **PTRACE_O_TRACESECCOMP** (自 Linux 3.5 始)
      在一个 [seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html) **SECCOMP_RET_TRACE** 规则被触发时停止 tracee。Tracer 调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 将返回一个这样的 *status* 值：`status>>8 == (SIGTRAP | (PTRACE_EVENT_SECCOMP<<8))` 。尽管这触发一个 **PTRACE_EVENT** 停止，但它与 syscall-enter-stop 类似。更多详情，请参考下面关于 **PTRACE_EVENT_SECCOMP** 的说明。Seccomp 事件消息数据（来自于 seccomp 过滤器规则的 **SECCOMP_RET_DATA** 部分）可以通过 **PTRACE_GETEVENTMSG** 提取。

   * **PTRACE_O_SUSPEND_SECCOMP** (自 Linux 4.3 始)
      挂起 tracee 的 seccomp 保护。这适用于任何模式，并且可被用在 tracee 还没有安装 seccomp 过滤器时。即，一个有效的使用场景是在 seccomp 保护被 tracee 安装之前挂起 seccomp 保护，让 tracee 安装过滤器，然后在过滤器应该被恢复时清除这个标记。设置这个选项需要 tracer 具有 **CAP_SYS_ADMIN** capability，没有任何 seccomp 保护已经安装，它本身还没有被设置 **PTRACE_O_SUSPEND_SECCOMP**。

**PTRACE_GETEVENTMSG** (自 Linux 2.5.46 始)
 > 提取一条关于刚刚发生的 ptrace 事件的消息（以一个 *unsigned long* 的形式），把它放在 tracer 中的地址 *data* 处。对于 **PTRACE_EVENT_EXIT**，这是 tracee 的退出状态。对于  **PTRACE_EVENT_FORK**， **PTRACE_EVENT_VFORK**， **PTRACE_EVENT_VFORK_DONE**， 和 **PTRACE_EVENT_CLONE**，这是新进程的 PID。对于 **PTRACE_EVENT_SECCOMP**，这是与触发的规则关联的 [seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html) 过滤器的 **SECCOMP_RET_DATA**。（*addr* 被忽略。）

**PTRACE_CONT**
 > 重启停止的 tracee 进程。如果 *data* 非零，则它被解释为传递给 tracee 的信号值；否则，不传递信号。这样，比如，tracer 可以控制发送给 tracee 的信号是否被传递。

**PTRACE_SYSCALL**，**PTRACE_SINGLESTEP**
 > 像 **PTRACE_CONT** 一样重启被停止的 tracee 进程，但分别地安排 tracee 在下一次进入或退出一个系统调用时，或在执行一条指令之后停下来。（Tracee 也将，像通常情况那样，在收到一个信号时停下来。）站在 tracer 的视角，tracee 将似乎已经被收到的一个 **SIGTRAP** 停下来。因此，对于 **PTRACE_SYSCALL** ，比如，其思想是在第一次停止时检查系统调用的参数，然后执行另一次 **PTRACE_SYSCALL**，并在第二次停止时检查系统调用的返回值。*data* 参数被按照 **PTRACE_CONT** 一样对待。（*addr* 被忽略。）

**PTRACE_SET_SYSCALL** (自 Linux 2.6.16 始)
 > 当处于 syscall-enter-stop 状态时，修改将要被执行的系统调用号为 *data* 参数指定的值。*addr* 参数被忽略。这个请求当前只在 arm （及 arm64，尽管只是为了向后兼容）平台上才支持，但大多数其它体系结构都有其他方法来实现这一点（通常是通过修改用户代码传入系统调用号的寄存器）。

**PTRACE_SYSEMU**，**PTRACE_SYSEMU_SINGLESTEP ** (自 Linux 2.6.14 始)
 > 对于 **PTRACE_SYSEMU**，继续并在进入下一个系统调用入口处停止，但系统调用将不会被执行。请参考下面关于 syscall-stops 的文档。对于 **PTRACE_SYSEMU_SINGLESTEP**，执行相同的操作但是是单步，但如果不是系统调用，也执行单步操作。这个调用被像用户模式 Linux 这样的想要模拟所有 tracee 的系统调用的程序使用。*data* 参数被按照 **PTRACE_CONT** 一样对待。*addr* 参数被忽略。这些参数当前只在 x86 上支持。

**PTRACE_LISTEN** (自 Linux 3.4 始)
 > 重新启动停止的 tracee，但阻止它执行。Tracee 最终的状态与被 **SIGSTOP** 信号（或其它停止信号）停止的进程的状态类似。参考 "group-stop" 部分来获得其它信息。**PTRACE_LISTEN** 只在通过 **PTRACE_SEIZE** attached 的 tracee 上才工作。

**PTRACE_KILL**
 > 给 tracee 发送一个 **SIGKILL** 信号来终止它。（*addr* 和 *data* 被忽略。）
 > *这个操作已经被废弃；不要使用它！* 相反，直接使用 [kill(2)](https://man7.org/linux/man-pages/man2/kill.2.html) 或 [tgkill(2)](https://man7.org/linux/man-pages/man2/tgkill.2.html) 发送 **SIGKILL** 信号。**PTRACE_KILL** 的问题是，它要求 tracee 处于 signal-delivery-stop 状态，否则它可能不工作（比如，可能成功完成但不会杀掉 tracee）。相比之下，直接发送 **SIGKILL** 信号没有这种限制。

**PTRACE_INTERRUPT** (自 Linux 3.4 始)
 > 停止 tracee。如果 tracee 正在运行或在内核空间休眠且 **PTRACE_SYSCALL** 生效，系统调用将被打断并报告 syscall-exit-stop 状态。（被中断的系统调用将在 tracee 被重启时重启。）如果 tracee 已经被一个信号停止，且被发了 **PTRACE_LISTEN**， 则 tracee 将以 **PTRACE_EVENT_STOP** 停止且 `WSTOPSIG(status)` 返回停止信号。如果同时生成了任何 ptrace-stop （比如，有一个信号发送给了 tracee），则 ptrace-stop 发生。如果上面的都不适用（比如，如果 tracee 正运行在用户空间），它将以 **PTRACE_EVENT_STOP** 停止，且 `WSTOPSIG(status) == SIGTRAP`。**PTRACE_INTERRUPT** 只在通过 **PTRACE_SEIZE** attached 的 tracee 上才工作。

**PTRACE_ATTACH**
 > Attach 到 *pid* 指定的进程，使它成为调用进程的一个 tracee。Tracee 被发送一个 **SIGSTOP** 信号，但不一定在调用完成时停止；使用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 来等待 tracee 停止。请参考 "Attaching 和 detaching" 小节来了解更多信息。（*addr* 和 *data* 被忽略。）
 > 执行 **PTRACE_ATTACH** 的权限由 ptrace 访问模式 **PTRACE_MODE_ATTACH_REALCREDS** 检查管理；请参考后文。

**PTRACE_SEIZE** (自 Linux 3.4 始)
 > Attach 到 *pid* 指定的进程，使它成为调用进程的一个 tracee。不像 **PTRACE_ATTACH**，**PTRACE_SEIZE** 不会停止 tracee。Group-stops 通过 **PTRACE_EVENT_STOP** 报告，且 *WSTOPSIG(status)* 返回停止信号。自动 attched 的子进程将以 **PTRACE_EVENT_STOP** 停止，*WSTOPSIG(status)* 返回 **SIGTRAP** 而不是把 **SIGSTOP** 信号传递给它们。[execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 不传递额外的 **SIGTRAP**。只有 **PTRACE_SEIZE** 的进程可以接受 **PTRACE_INTERRUPT** 和 **PTRACE_LISTEN** 命令。使用 **PTRACE_O_TRACEFORK**， **PTRACE_O_TRACEVFORK** 和 **PTRACE_O_TRACECLONE** 自动地 attached 的子进程会继承刚刚描述的 "seized" 行为。*addr* 必须是零。*data* 包含一个要立即激活的 ptrace 选项的位掩码。
 > 执行 **PTRACE_SEIZE** 的权限由 ptrace 访问模式 **PTRACE_MODE_ATTACH_REALCREDS** 检查管理；请参考后文。

**PTRACE_SECCOMP_GET_FILTER** (自 Linux 4.4 始)
 > 这个操作允许 tracer 转储 tracee 的经典 BPF 过滤器。
 > *addr* 是一个整数，指定了要转储的过滤器的索引。最近安装的过滤器索引为 0。如果 *addr* 比安装的过滤器的个数大，则操作将以错误 **ENOENT** 失败。
 > *data* 是一个足够保存 BPF 程序的指向 *struct sock_filter* 数组的指针，或在不保存程序时是 NULL。
 > 成功时，返回值是 BPF 程序中的指令数。如果 *data* 是 NULL，则这个返回值可以被用于正确地确定传递给后续调用的 *struct sock_filter* 数组的大小。
 > 如果调用者不具有 **CAP_SYS_ADMIN** 能力，或调用者处于 strict 或 filter seccomp 模式，则这个操作将以错误 **EACCES** 失败。如果 *addr* 引用的过滤器不是一个经典过滤器，则这个操作将以错误 **EMEDIUMTYPE** 失败。
 > 如果内核同时配置了 **CONFIG_SECCOMP_FILTER** 和 **CONFIG_CHECKPOINT_RESTORE** 选项这个操作可用。

**PTRACE_DETACH**
 > 像 **PTRACE_CONT** 一样重启被停止的 tracee，但首先从它 detach。在 Linux 下，tracee 可以通过这种方式被 detached，而无论初始化追踪所用的是何种方法。（*addr* 被忽略。）

**PTRACE_GET_THREAD_AREA** (自 Linux 2.6.0 始)
 > 这个操作执行了一个与 [get_thread_area(2)](https://man7.org/linux/man-pages/man2/get_thread_area.2.html) 类似的任务。它读取 GDT 中的 TLS 项，其中索引由 *addr* 给出，并在 *data* 指向的 *struct user_desc* 中放入一份该项的拷贝。（相比通过 [get_thread_area(2)](https://man7.org/linux/man-pages/man2/get_thread_area.2.html) 时，*struct user_desc* 的 *entry_number* 被忽略。）


**PTRACE_SET_THREAD_AREA** (自 Linux 2.6.0 始)
 > 这个操作执行了一个与 [set_thread_area(2)](https://man7.org/linux/man-pages/man2/set_thread_area.2.html)
 类似的任务。它设置 GDT 中的 TLS 项，其中索引由 *addr* 给出，把它的值设置为 *data* 指向的 *struct user_desc* 所提供的数据。（相比通过 [set_thread_area(2)](https://man7.org/linux/man-pages/man2/set_thread_area.2.html) 时，*struct user_desc* 的 *entry_number* 被忽略；换句话说，这个 ptrace 操作无法被用于分配一个空闲的 TLS 项。）

**PTRACE_GET_SYSCALL_INFO** (自 Linux 5.3 始)
 > 提取关于导致 tracee 停止的系统调用的信息。这些信息被放进 *data* 参数指向的缓冲区中，它应该是一个指向类型为 *struct ptrace_syscall_info* 的缓冲区的指针。*addr* 参数包含 *data* 参数指向的缓冲区的大小（比如，`sizeof(struct ptrace_syscall_info)`）。返回值包含可用的将被内核写入的字节数。如果将被内核写入的数据的大小超出了 *addr* 参数指定的大小，则输出的数据将被截断。
 > *ptrace_syscall_info* 结构包含如下字段：
```
                  struct ptrace_syscall_info {
                      __u8 op;        /* Type of system call stop */
                      __u32 arch;     /* AUDIT_ARCH_* value; see seccomp(2) */
                      __u64 instruction_pointer; /* CPU instruction pointer */
                      __u64 stack_pointer;    /* CPU stack pointer */
                      union {
                          struct {    /* op == PTRACE_SYSCALL_INFO_ENTRY */
                              __u64 nr;       /* System call number */
                              __u64 args[6];  /* System call arguments */
                          } entry;
                          struct {    /* op == PTRACE_SYSCALL_INFO_EXIT */
                              __s64 rval;     /* System call return value */
                              __u8 is_error;  /* System call error flag;
                                                 Boolean: does rval contain
                                                 an error value (-ERRCODE) or
                                                 a nonerror return value? */
                          } exit;
                          struct {    /* op == PTRACE_SYSCALL_INFO_SECCOMP */
                              __u64 nr;       /* System call number */
                              __u64 args[6];  /* System call arguments */
                              __u32 ret_data; /* SECCOMP_RET_DATA portion
                                                 of SECCOMP_RET_TRACE
                                                 return value */
                          } seccomp;
                      };
                  };
```

 > *op*，*arch*，*instruction_pointer*，和 *stack_pointer* 字段是所有类型的 ptrace 系统调用停止都定义了的。结构体的其余部分是一个联合体；读取的进程只应该读取那些对于 *op* 字段指定的系统调用停止类型有意义的字段。
*op* 字段可能的取值如下（在 *<linux/ptrace.h>* 中定义），这些值表示发生的停止是什么类型，及联合体的哪部分会被填充：

   * **PTRACE_SYSCALL_INFO_ENTRY**
      联合体的 *entry* 组件包含与系统调用进入停止有关的信息。

   * **PTRACE_SYSCALL_INFO_EXIT**
      联合体的 *exit* 组件包含与系统调用退出停止有关的信息。

   * **PTRACE_SYSCALL_INFO_SECCOMP**
      联合体的 *seccomp* 组件包含与 **PTRACE_EVENT_SECCOMP** 停止有关的信息。

   * **PTRACE_SYSCALL_INFO_NONE**
      没有联合体的组件包含相关的信息。

### ptrace 之下的死亡

当一个（可能是多线程的）进程收到一个杀进程信号（它的处理例程被设置为 **SIG_DFL **，且它的默认行为是杀掉进程），则所有线程退出。Tracee 向它们的 tracer 报告它们的死亡。这个事件的通知通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 传递。

注意杀进程信号将首先使 tracee 进入 signal-delivery-stop 状态（只在一个 tracee 中），且只要在它被 tracer 注入后（或它被派发给没有被追踪的线程后），多线程进程中的 *所有* tracee 中将发生信号死亡。（术语 "signal-delivery-stop" 在后面解释。）

**SIGKILL** 信号不产生 signal-delivery-stop，并因此 tracer 无法抑制它。**SIGKILL** 会杀死进程，甚至是在它正在执行系统调用时（在 **SIGKILL** 导致的死亡之前不生成 syscall-exit-stop）。最终效果是 **SIGKILL** 总是杀死进程（及它的所有线程），即使进程的某些线程在被 ptrace 追踪。

当 tracee 调用 [_exit(2)](https://man7.org/linux/man-pages/man2/_exit.2.html) 时，它向它的 tracer 报告它的死亡。其它线程不受影响。

当任何线程执行 [exit_group(2)](https://man7.org/linux/man-pages/man2/exit_group.2.html) 时，它的线程组的每个 tracee 向它的 tracer 报告它的死亡。

如果 **PTRACE_O_TRACEEXIT** 选项是打开的，则在实际的死亡之前将发生 **PTRACE_EVENT_EXIT**。这适用于通过 [exit(2)](https://man7.org/linux/man-pages/man2/exit.2.html) 和 [exit_group(2)](https://man7.org/linux/man-pages/man2/exit_group.2.html) 退出进程，及信号导致的死亡（除 **SIGKILL** 信号外，依赖于内核版本，请参考下面的 BUGS 部分），和在一个多线程进程中当执行 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 时导致的线程死亡。

Tracer 不能假设 ptrace-stop 的 tracee 退出。Tracee 处于停止状态时可能死掉的场景有很多（比如 **SIGKILL** 信号）。然而，tracer 必须准备好在任何 ptrace 操作上处理 **ESRCH** 错误。不幸的是，如果 tracee 退出但不处于 ptrace-stopped 状态（那些需要 tracee 处于停止状态的命令），或者如果它没有被发出 ptrace 调用的进程追踪会返回相同的错误。Tracer 需要跟踪 tracee 的停止/运行状态，并只在它确定已经观察到 tracee 进入 ptrace-stop 状态时才把 **ESRCH** 解释为 "tracee 意外退出"。注意如果一个 ptrace 操作返回 **ESRCH**，无法保证 *waitpid(WNOHANG)* 将可靠地报告 tracee 的死亡状态。*waitpid(WNOHANG)* 可能返回 0。换句话说，tracee 可能 "还没有完全死亡"，但已经开始拒绝 ptrace 请求。

Tracer 不能假设 tracee *总是* 通过报告 *WIFEXITED(status)* 或 *WIFSIGNALED(status)* 来结束它的生命；有些情况下这没有发生。比如，如果一个非线程组 leader 的线程执行了一个 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)，它将消失；它的 PID 将永远不会再被看到，且后续的所有 ptrace 停止将在线程组 leader 的 PID 下报告。

### 停止状态

一个 tracee 可以处于两种状态：运行或停止。不过对于 ptrace，阻塞在系统调用（比如，[read(2)](https://man7.org/linux/man-pages/man2/read.2.html)，[pause(2)](https://man7.org/linux/man-pages/man2/pause.2.html) 等等）中的 tracee 被认为处于运行状态，即使 tracee阻塞了很久。Tracee 在 **PTRACE_LISTEN** 之后的状态有点灰色区域：它不处于任何 ptrace-stop 状态（ptrace 命令在其上将不起作用，但它将传递 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 通知），但它也可以被认为是 "停止" 状态，因为它不执行指令（没有被调度），且如果它在 **PTRACE_LISTEN** 之前处于 group-stop，它将不对信号做出响应直到收到 **SIGCONT** 信号。

Tracee 被停止时有非常多种状态，在讨论中，它们经常被合并在一起。然而，使用精确的术语很重要。

在本手册中，tracee 所处于的为接收来自于 tracer 的 ptrace 命令做好准备的任何停止状态称作 *ptrace-stop*。Ptrace-stops 可以进一步被分成 *ignal-delivery-stop*，*group-stop*，*syscall-stop*，*PTRACE_EVENT stops*，等等。这些状态将在下面详细描述。

当正在运行的 tracee 进入 ptrace-stop 状态，它使用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 通知它的 tracer（或者其它 "wait" 系统调用中的一个）。本手册中的大部分页面假设 tracer 通过如下方式等待：
```
    pid = waitpid(pid_or_minus_1, &status, __WALL);
```

被 ptrace-stop 停掉的 tracee 被报告为 *pid* 大于 0 且 *WIFSTOPPED(status)* 为 true 的返回值。

**__WALL** 标记不包含 **WSTOPPED** 和 **WEXITED** 标记，但蕴含了它们的功能。

不建议在调用 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 时设置 **WCONTINUED** 标记："continued" 状态是 per-process 的，消费它可能会导致 tracee 真正的父进程混淆。使用 **WNOHANG** 标记可能导致 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回 0（“不等待最终结果可用”）即使 tracer 知道应该有一个通知。比如：
```
           errno = 0;
           ptrace(PTRACE_CONT, pid, 0L, 0L);
           if (errno == ESRCH) {
               /* tracee is dead */
               r = waitpid(tracee, &status, __WALL | WNOHANG);
               /* r can still be 0 here! */
           }
```

存在以下几种 ptrace-stops：signal-delivery-stops，group-stops，**PTRACE_EVENT** stops，syscall-stops。它们都通过 *WIFSTOPPED(status)* 为 true 的 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 报告。它们可以通过检查 *status>>8* 的值来区分，如果那个值有歧义，则通过查询 **PTRACE_GETSIGINFO**。（注意：*WSTOPSIG(status)* 宏不能被用于执行这种检查，因为它返回 *(status>>8) & 0xff* 的值。）

### Signal-delivery-stop

当一个进程（可能是多线程的）收到任何除 **SIGKILL** 之外的信号，内核选择一个处理信号的任意线程。（如果信号由 [tgkill(2)](https://man7.org/linux/man-pages/man2/tgkill.2.html) 生成，则目标线程可以由调用者显式地选择。）如果选择地线程在被追踪，则它进入 signal-delivery-stop 状态。此时，信号还没有被传递给进程，且可以被 tracer 抑制。如果 tracer 不抑制该信号，则它在下次 ptrace 重启请求中把该信号传给 tracee。信号传递的这第二步在本手册中被称为 *信号注入*。注意如果信号被阻塞，则 signal-delivery-stop 不会发生，直到该信号被解除阻塞，通常情况下 **SIGSTOP** 是不能被阻塞的。

Tracer 通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回 *WIFSTOPPED(status)* 为 true 观测到 signal-delivery-stop 状态，通过 *WSTOPSIG(status)* 获得返回的信号值。如果信号是 **SIGTRAP**，这可能是一种不同类型的 ptrace-stop；请参考下面的 "Syscall-stops" 和 "execve" 小节来了解详细信息。如果 *WSTOPSIG(status)* 返回一个停止信号，这可能是一个 group-stop；请参考下面的内容。

### 信号注入和抑制

Tracer 观测到 signal-delivery-stop 之后，tracer 应该通过这个调用重启 tracee：
```
           ptrace(PTRACE_restart, pid, 0, sig)
```
其中 **PTRACE_restart** 是重启 ptrace 请求中的一个。如果 *sig* 是 0，则信号不被传递。否则，信号 *sig* 被传递。这个操作在本手册中被称为 *信号注入*，以将它从 signal-delivery-stop 中区分出来。

*sig* 值可能不同于来自于 *WSTOPSIG(status)* 的值：tracer 可能导致一个不同的信号被注入。

注意，一个受抑制的信号依然导致系统调用提前返回。在这种情况下，系统调用将被重启：如果 tracer 使用 **PTRACE_SYSCALL** 则 tracer 将观察到 tracee 重新执行被中断的系统调用（或者一些系统调用使用不同的机制来重启，使用 [restart_syscall(2)](https://man7.org/linux/man-pages/man2/restart_syscall.2.html) 重启系统调用）。即使是在信号之后不能重启的系统调用（比如 [poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)）在信号被抑制之后也会重启；然而，内核缺陷的出现导致一些系统调用以 **EINTR** 失败，即使没有可观察的信号被注入进 tracee。

处于 ptrace-stops 而不是 signal-delivery-stop 状态中，重启 ptrace 命令发出的信号不保证会被注入，即使 *sig* 非零。没有错误会报告；非零的 *sig* 可能被简单地忽略。Ptrace 用户不应该尝试通过这种方式 "创建一个新信号"：使用 [tgkill(2)](https://man7.org/linux/man-pages/man2/tgkill.2.html) 替代。

当重启处于不是 signal-delivery-stops 的 ptrace 停止状态之后的 tracee 时，信号注入请求可能被忽略是 ptrace 用户感到困惑的一个原因。一个典型的场景是 tracer 观察到 group-stop，将它误认作 signal-delivery-stop，然后通过如下调用重启 tracee：
```
           ptrace(PTRACE_restart, pid, 0, stopsig)
```
期望注入 *stopsig*，但 *stopsig* 被忽略，且 tracee 继续执行。

**SIGCONT** 信号有一个副作用，即唤醒（所有）一个处于 group-stop 状态的进程。这个副作用发生在 signal-delivery-stop 之前。Tracer 无法抑制这种副作用（它只能抑制信号注入，即如果安装了这种处理器的话，它导致在 tracee 中的 **SIGCONT** 处理程序不被执行）。事实上，从 group-stop 唤醒后可能跟的是 signal-delivery-stop 的信号*而不是* **SIGCONT**，如果在 **SIGCONT** 被传递时它们正在挂起的话。换句话说，**SIGCONT** 可能不是 tracee 被发送信号之后观察到的第一个信号。

停止信号导致（所有线程）进程进入 group-stop 状态。这个副作用发生在信号注入之后，因而可以被 tracer 抑制。

在 Linux 2.4 及更早的版本中，**SIGSTOP** 不能被注入。

**PTRACE_GETSIGINFO** 可以被用于提取一个对应于传递的信号的 *siginfo_t* 结构。**PTRACE_SETSIGINFO** 可以被用于修改它。如果 **PTRACE_SETSIGINFO** 已经被用于改变 *siginfo_t*，则 *si_signo* 字段和重启命令中的 *sig* 参数必须匹配，否则结果是未定义的。

### Group-stop

当一个（可能是多线程的）进程收到一个停止信号，所有线程都停止。如果一些线程在被追踪，则它们进入 group-stop 状态。注意，停止信号将首先导致 signal-delivery-stop （只在一个 tracee 上），且只有在它被 tracer 注入后（或者在它被分派给一个没有被追踪的线程后），则在多线程进程内 group-stop 将在 *所有* tracee 上被初始化。像往常一样，每个 tracee 分别报告它的 group-stop 给对应的 tracer。

Tracer 通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回 *WIFSTOPPED(status)* 为 true 观测到 group-stop 状态，通过 *WSTOPSIG(status)* 获得返回的信号值。其它一些类型的 ptrace-stop 也会返回相同的结果，然而建议的实践是执行如下调用
```
           ptrace(PTRACE_GETSIGINFO, pid, 0, &siginfo)
```

如果信号不是 **SIGSTOP**，**SIGTSTP**，**SIGTTIN**，或 **SIGTTOU**，则这个调用可以避免；只有这四个信号是停止信号。如果 tracer 看到了其它的，则它不可能是 group-stop。否则，tracer 需要调用 **PTRACE_GETSIGINFO**。如果 **PTRACE_GETSIGINFO** 以 **EINVAL** 失败，那它绝对是一个 group-stop。（其它的错误码是可能的，比如如果一个 **SIGKILL** 杀掉了 tracee 则是 **ESRCH**（"没有这个进程"）。）

如果 tracee 是使用 **PTRACE_SEIZE** 被 attach 的，则 group-stop 由 **PTRACE_EVENT_STOP** 指示：`status>>16 == PTRACE_EVENT_STOP`。这允许探测 group-stop 而无需一个额外的 **PTRACE_GETSIGINFO** 调用。

自从 Linux 2.6.38，在 tracer 看到 tracee 进入 ptrace-stop 之后，直到它重启或杀掉它，则 tracee 将不会运行，且将不会给 tracer 发送通知（除了 **SIGKILL** 死亡之外），即使 tracer 进入了另一个 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 调用。

前面段落描述的内核行为导致了对停止信号的透明处理的问题。如果 tracer 在 group-stop 之后重启了 tracee，则停止信号实际上被忽略了 ——— tracee 不保持停止状态，它运行了。如果 tracer 在进入下一个 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 之前不重启 tracee，则未来 **SIGCONT** 信号将不会被报告给 tracer；这将导致 **SIGCONT** 信号对 tracee 无效。

自从 Linux 3.4 开始，有一个方法来克服这个问题：不使用 **PTRACE_CONT**，**PTRACE_LISTEN** 命令可以被用于重启 tracee，以一种它不执行，但等待一个新的它可以通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 报告的事件（比如当它被 **SIGCONT** 重启时）。

### PTRACE_EVENT 停止

如果 tracer 设置了 **PTRACE_O_TRACE_*** 选项，则 tracee 将进入称为 **PTRACE_EVENT** 停止的 ptrace-stops。

Tracer 通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回 *WIFSTOPPED(status)* 为 true，且 *WSTOPSIG(status)* 返回 **SIGTRAP** （或者对于  **PTRACE_EVENT_STOP**，如果 tracee 正处于 group-stop 状态则返回停止信号）观测到 **PTRACE_EVENT** 停止。状态字高位字节中的一个额外位被设置：`status>>8` 的值将是 
```
           ((PTRACE_EVENT_foo<<8) | SIGTRAP).
```

存在以下事件：

**PTRACE_EVENT_VFORK**
> 在以 **CLONE_VFORK** 标记从 [vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html) 或 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 返回之前停止。在这个停止之后 tracee 被继续时，它将在继续它的执行之前等待子进程进入 exit/exec（换句话说，[vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html)  上的通常行为）。

**PTRACE_EVENT_FORK**
> 在从 [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html) 或 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 返回之前停止时会把退出信号设置为 SIGCHLD。

**PTRACE_EVENT_CLONE**
> 在从 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 返回之前停止。

**PTRACE_EVENT_VFORK_DONE**
> 在以 **CLONE_VFORK** 标记从 [vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html) 或 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 返回之前停止，但在子进程通过退出或执行解除阻塞这个 tracee 之后。

对于上面描述的所有四种停止，停止发生在父进程（比如，tracee），而不是在新创建的线程中。**PTRACE_GETEVENTMSG** 可以被用于提取新线程的 ID。

**PTRACE_EVENT_EXEC**
> 在从 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 返回之前停止。自 Linux 3.0 起，**PTRACE_GETEVENTMSG** 返回之前的线程 ID。

**PTRACE_EVENT_EXIT**
> 在退出（包括从 [exit_group(2)](https://man7.org/linux/man-pages/man2/exit_group.2.html) 死亡），信号死亡，或者在多线程进程中的由 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 引起的退出之前停止。**PTRACE_GETEVENTMSG** 返回退出状态。可以检查寄存器（不像在 “真正的” 退出发生时那样）。Tracee 依然是活着的，它需要被 **PTRACE_CONT** 或 **PTRACE_DETACH** 来结束退出。

**PTRACE_EVENT_STOP**
> 在一个新的子进程被 attach 时（只有使用 **PTRACE_SEIZE** 时 attach）由 **PTRACE_INTERRUPT** 命令，或 group-stop 或初始化 ptrace-stop 引起的停止。

**PTRACE_EVENT_SECCOMP**
> 当 tracer 已经设置了 **PTRACE_O_TRACESECCOMP** 时，由 tracee 系统调用项上 [seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html) 规则触发的停止。seccomp 事件消息数据（从 seccomp 过滤器规则的 **SECCOMP_RET_DATA** 部分）可以通过 **PTRACE_GETEVENTMSG** 提取。这种停止的语义在下面一个单独的小节详细描述。

**PTRACE_EVENT** 停止上的 **PTRACE_GETSIGINFO** 在 *si_signo* 中返回 **SIGTRAP**，同时设置 *si_code* 为 *(event<<8) | SIGTRAP*。

### 系统调用停止
如果 tracee 是由 **PTRACE_SYSCALL** 或 **PTRACE_SYSEMU** 重新启动的，则 tracee 将在进入任何系统调用（如果是使用 **PTRACE_SYSEMU** 重新启动的，则它们将不会被执行，无论此时对寄存器做了任何修改，或无论在这个停止之后 tracee 是如何被重新启动的）之前进入 syscall-enter-stop 状态。无论哪个方法导致了 syscall-enter-stop，如果 tracer 用 **PTRACE_SYSCALL** 重新启动 tracee，则当系统调用结束时，tracee 进入 syscall-exit-stop 状态，或如果它被一个信号中断。（即，signal-delivery-stop 从来不会在 syscall-enter-stop 和 syscall-exit-stop 之间发生；它发生在 syscall-exit-stop *之后*。）如果使用任何其它方法（包括 **PTRACE_SYSEMU**）使 tracee 继续执行，则将不会有 syscall-exit-stop 发生。注意所有提到的 **PTRACE_SYSEMU** 同样适用于 **PTRACE_SYSEMU_SINGLESTEP**。

然而，即使 tracee 是使用 **PTRACE_SYSCALL** 使继续的，也不保证下一次停止将是 syscall-exit-stop。Tracee 停止的其它可能性包括，**PTRACE_EVENT** 停止（包括 seccomp 停止），退出（如果它进入 [_exit(2)](https://man7.org/linux/man-pages/man2/_exit.2.html) 或 [exit_group(2)](https://man7.org/linux/man-pages/man2/exit_group.2.html)），被 **SIGKILL** 杀死，或安静地死亡（如果它是一个线程组领袖，[execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 发生在另一个线程中，且那个线程不由同一个 tracer 追踪；这种情况在后面讨论）。

Syscall-enter-stop 和 syscall-exit-stop 由 tracer 通过 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回 *WIFSTOPPED(status)* 为 true，且 *WSTOPSIG(status)* 的值为 **SIGTRAP** 来观察到。如果 tracer 设置了 **PTRACE_O_TRACESYSGOOD** 选项，则 *WSTOPSIG(status)* 的值将为 *(SIGTRAP | 0x80)*。

**SIGTRAP** 发生时，syscall-stops 可以通过查询 **PTRACE_GETSIGINFO** 的如下情况从 signal-delivery-stop 辨别出来：

*si_code <= 0*
   - **SIGTRAP** 的传递是一个用户空间行为的结果，比如，一个系统调用（[tgkill(2)](https://man7.org/linux/man-pages/man2/tgkill.2.html)，[kill(2)](https://man7.org/linux/man-pages/man2/kill.2.html)，[sigqueue(3)](https://man7.org/linux/man-pages/man3/sigqueue.3.html)，等等），一个 POSIX 定时器的超时，一个 POSIX 消息队列上的状态变化，或一个异步 I/O 请求的完成。

*si_code* == SI_KERNEL (0x80)
   - **SIGTRAP** 是由内核发送的。

*si_code* == SIGTRAP 或 *si_code* == (SIGTRAP|0x80)
   - 这是一个 syscall-stop。

然而，syscall-stop 发生的非常频繁（每个系统调用两次），且为每一次 syscall-stop 执行 **PTRACE_GETSIGINFO** 可能有点昂贵。

一些体系架构允许通过检查寄存器来辨别这种情况。比如，在 x86 上，syscall-enter-stop 中，*rax* == -**ENOSYS**。由于 **SIGTRAP**（像任何其它信号那样） 总是发生在 *syscall-exit-stop* 之后，且此时 *rax* 几乎从来不包含 -**ENOSYS**，则 **SIGTRAP** 看起来像 “不是 syscall-enter-stop 的 syscall-stop”；换句话说，它看起来像一个 “迷路的 syscall-exit-stop”，且可通过这种方式探测。但这种探测是脆弱的，且最好能够避免。

建议使用 **PTRACE_O_TRACESYSGOOD** 选项的方法来将 syscall-stops 从其它种类的 ptrace-stops 中辨别出来，因为它比较可靠且不会导致性能损失。

跟踪程序无法区分 syscall-enter-stop 和 syscall-exit-stop。跟踪程序需要追踪 ptrace-stops 序列以不把 syscall-enter-stop 误解释为 syscall-exit-stop ，反之亦然。一般来说，syscall-enter-stop 后面总是会跟着 syscall-exit-stop，**PTRACE_EVENT** 停止，或者 tracee 的死亡；没有其它种类的 ptrace-stop 可以发生在这两者之间。 然而，注意 seccomp 停止（参考下文）可能导致 syscall-exit-stops，没有前面的 syscall-entry-stops。如果使用了 seccomp ，则需要注意不要把这种停止误解释为 syscall-entry-stops。

如果 syscall-enter-stop 之后，追踪程序使用一个 **PTRACE_SYSCALL** 之外的其它的重起命令，则不会生成 syscall-exit-stop。

syscall-stops 上的 **PTRACE_GETSIGINFO** 在 *si_signo* 中返回 **SIGTRAP**，其 *si_code* 被设置为 **SIGTRAP** 或 *(SIGTRAP|0x80)*。

### PTRACE_EVENT_SECCOMP 停止（Linux 3.5 到 4.7）
**PTRACE_EVENT_SECCOMP** 停止的行为以及它们与其他类型的 ptrace 停止的交互在内核版本之间是变化的。这里记录了从引入到 Linux 4.7（包括）的行为。后续内核版本中的行为在下一节中记录。

每当触发 **SECCOMP_RET_TRACE** 规则时，就会发生 **PTRACE_EVENT_SECCOMP** 停止。这与使用哪种方法重新启动系统调用无关。值得注意的是，即使被跟踪者被使用 **PTRACE_SYSEMU** 重新启动并且这个系统调用被无条件地跳过，seccomp 仍然运行。

从此停止重新启动的行为就像停止发生在相关系统调用之前一样。特别是，**PTRACE_SYSCALL** 和 **PTRACE_SYSEMU** 通常都会导致后续的 syscall-entry-stop。但是，如果在 **PTRACE_EVENT_SECCOMP** 之后系统调用号为负，则 syscall-entry-stop 和系统调用本身都将被跳过。这意味着如果在 **PTRACE_EVENT_SECCOMP** 之后系统调用号为负并且被跟踪者被使用 **PTRACE_SYSCALL** 重新启动，下一个观察到的停止将是 syscall-exit-stop，而不是可能预期的 syscall-entry-stop。

### PTRACE_EVENT_SECCOMP 停止（从 Linux 4.8 开始）
从 Linux 4.8 开始，**TRACE_EVENT_SECCOMP** 停止被重新排序以发生在 syscall-entry-stop 和 syscall-exit-stop 之间。请注意，如果由于 **PTRACE_SYSEMU** 而跳过系统调用，则 seccomp 不再运行（并且不会报告 **PTRACE_EVENT_SECCOMP**）。

在功能上，**PTRACE_EVENT_SECCOMP** 停止的功能与 syscall-entry-stop 类似（即，继续使用 **PTRACE_SYSCALL** 将导致 syscall-exit-stops，系统调用号可能会更改，并且任何其它修改的寄存器对要执行的系统调用都是可见的）。请注意，可能有但不必有前导的 syscall-entry-stop。

在 **PTRACE_EVENT_SECCOMP** 停止后，seccomp 将重新运行，**SECCOMP_RET_TRACE** 规则现在的功能与 **SECCOMP_RET_ALLOW** 相同。具体来说，这意味着如果在 **PTRACE_EVENT_SECCOMP** 停止期间没有修改寄存器，则系统调用将被允许。

### PTRACE_SINGLESTEP 停止

[这些类型的停止的细节还有待记录。]

### 信息和重新启动的 ptrace 命令

大多数 ptrace 命令（除了 **PTRACE_ATTACH**、**PTRACE_SEIZE**、**PTRACE_TRACEME**、**PTRACE_INTERRUPT** 和 **PTRACE_KILL**）都要求被跟踪者处于 ptrace-stop 状态，否则它们会因 **ESRCH** 而失败。

当被跟踪者处于 ptrace-stop 时，跟踪者可以使用信息命令向被跟踪者读取和写入数据。这些命令使被跟踪者仍处于 ptrace-stopped 状态：
```
           ptrace(PTRACE_PEEKTEXT/PEEKDATA/PEEKUSER, pid, addr, 0);
           ptrace(PTRACE_POKETEXT/POKEDATA/POKEUSER, pid, addr, long_val);
           ptrace(PTRACE_GETREGS/GETFPREGS, pid, 0, &struct);
           ptrace(PTRACE_SETREGS/SETFPREGS, pid, 0, &struct);
           ptrace(PTRACE_GETREGSET, pid, NT_foo, &iov);
           ptrace(PTRACE_SETREGSET, pid, NT_foo, &iov);
           ptrace(PTRACE_GETSIGINFO, pid, 0, &siginfo);
           ptrace(PTRACE_SETSIGINFO, pid, 0, &siginfo);
           ptrace(PTRACE_GETEVENTMSG, pid, 0, &long_var);
           ptrace(PTRACE_SETOPTIONS, pid, 0, PTRACE_O_flags);
```

请注意，某些错误未报告。例如，设置信号信息（*siginfo*）在某些 ptrace-stops 中可能没有效果，但调用可能会成功（返回 0 而不设置 *errno*）；如果当前 ptrace-stop 不返回有意义的事件消息，则查询 **PTRACE_GETEVENTMSG** 可能会成功并返回一些随机值。

调用
```
           ptrace(PTRACE_SETOPTIONS, pid, 0, PTRACE_O_flags);
```
影响一个被跟踪者。被跟踪者的当前标志被替换。标志由通过激活的 **PTRACE_O_TRACEFORK**、**PTRACE_O_TRACEVFORK** 或 **PTRACE_O_TRACECLONE** 选项创建并 “自动附加” 的新被追踪者继承。

另一组命令使 ptrace-stopped 的 tracee 运行。它们具有以下形式：
```
           ptrace(cmd, pid, 0, sig);
```
其中 *cmd* 是 **PTRACE_CONT**、**PTRACE_LISTEN**、**PTRACE_DETACH**、**PTRACE_SYSCALL**、**PTRACE_SINGLESTEP**、**PTRACE_SYSEMU** 或 **PTRACE_SYSEMU_SINGLESTEP**。如果被跟踪者处于 signal-delivery-stop 状态，则 *sig* 是要注入的信号（如果它非零）。否则，可能会忽略 *sig*。（当从 signal-delivery-stop 以外的 ptrace-stop 重新启动被跟踪者时，推荐的做法是始终在 *sig* 中传递 0。）

### 附加和分离

可以使用如下调用将线程附加到跟踪器
```
    ptrace(PTRACE_ATTACH, pid, 0, 0);
```
或
```
    ptrace(PTRACE_SEIZE, pid, 0, PTRACE_O_flags);
```

**PTRACE_ATTACH** 向该线程发送 **SIGSTOP**。如果跟踪器希望这个 **SIGSTOP** 不起作用，它需要抑制它。请注意，如果在附加期间其它信号同时发送到此线程，则跟踪器可能会看到被跟踪者首先以其它信号进入 signal-delivery-stop 状态！通常的做法是重新注入这些信号，直到看到 **SIGSTOP**，然后抑制 **SIGSTOP** 注入。这里的设计错误是 ptrace 附加和并发传递的 **SIGSTOP** 可能会竞争，并发 **SIGSTOP** 可能会丢失。

由于附加发送 **SIGSTOP** 并且跟踪器通常会抑制它，因此这可能会导致从被跟踪者中当前正在执行的系统调用返回一个杂散的 **EINTR**，如 “信号注入和抑制” 部分所述。

从 Linux 3.4 开始，可以使用 **PTRACE_SEIZE** 代替 **PTRACE_ATTACH**。**PTRACE_SEIZE** 不会停止附加的进程。如果您需要在附加后（或在任何其他时间）停止它而不向它发送任何信号，**请使用 PTRACE_INTERRUPT** 命令。

请求
```
    ptrace(PTRACE_TRACEME, 0, 0, 0);
```
将调用线程变成被跟踪者。这个线程继续运行（不进入 ptrace-stop）。一种常见的做法是在 **PTRACE_TRACEME** 后面跟着
```
    raise(SIGSTOP);
```
并允许父进程（现在是我们的跟踪器）观察我们的 signal-delivery-stop。

如果 **PTRACE_O_TRACEFORK**、**PTRACE_O_TRACEVFORK** 或 **PTRACE_O_TRACECLONE** 选项有效，则分别由带有 **CLONE_VFORK** 标志的 [vfork(2)](https://man7.org/linux/man-pages/man2/vfork.2.html) 或 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html)，退出信号设置为 **SIGCHLD** 的 [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html) 或 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) ，和其它类型的 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) 创建的子进程，会自动附加到跟踪其父进程的同一个跟踪器。**SIGSTOP** 被传递给子进程，导致它们在退出创建它们的系统调用后进入 signal-delivery-stop 状态。

被跟踪者的分离是通过以下方式执行的：
```
    ptrace(PTRACE_DETACH, pid, 0, sig);
```
**PTRACE_DETACH** 是重启操作；因此它要求被跟踪者处于 ptrace-stop 状态。如果被跟踪者处于 signal-delivery-stop 状态，则可以注入信号。否则，可能会默默地忽略 *sig* 参数。

如果在跟踪器想要分离它时被跟踪者正在运行，通常的解决方案是发送 **SIGSTOP**（使用 [tgkill(2)](https://man7.org/linux/man-pages/man2/tgkill.2.html)，以确保它转到正确的线程），等待被跟踪者因 **SIGSTOP** 在 signal-delivery-stop 中停止，然后将其分离（抑制 **SIGSTOP** 注入）。一个设计错误是这可能与并发 **SIGSTOP** 竞争。另一个复杂的问题是 tracee 可能会进入其它 ptrace-stops，需要重新启动并再次等待，直到看到 **SIGSTOP**。另一个复杂的问题是要确保被跟踪者还没有被 ptrace 停止，因为在它停止时不会发生信号传递 —— 甚至 **SIGSTOP** 也不行。

如果跟踪器死亡，所有被追踪者都会自动分离并重新启动，除非它们处于 group-stop 状态。从 group-stop 重新启动的处理目前有问题，但“按计划”的行为是让被跟踪者停止并等待 **SIGCONT**。如果被跟踪者从 signal-delivery-stop 中重新启动，则注入挂起的信号。

### ptrace 下的 execve(2)
当多线程进程中的一个线程调用 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 时，内核会销毁进程中的所有其它线程，并将执行线程的线程 ID 重置为线程组 ID（进程 ID）。（或者，换句话说，当多线程进程执行 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 时，在调用完成时，看起来好像 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 发生在线程组领导中，无论哪个线程执行 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)。）线程 ID 的这种重置对于跟踪器来说看起来非常混乱：

 * 如果 **PTRACE_O_TRACEEXIT** 选项被打开，所有其它线程停止在 **PTRACE_EVENT_EXIT** 停止处。然后除线程组领导者之外的所有其它线程报告死亡，就好像它们通过 [_exit(2)](https://man7.org/linux/man-pages/man2/_exit.2.html) 退出，退出代码为 0 一样。

 * 执行的被追踪者在 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 中时更改其线程 ID。（请记住，在 ptrace 下，从 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 返回的“ pid” 或输入到 ptrace 调用中的 “pid” 是被跟踪者的线程 ID。）也就是说，tracee 的线程 ID 被重置为与其进程 ID 相同的进程 ID，即与线程组领导的线程 ID 相同。

 * 如果 **PTRACE_O_TRACEEXEC** 选项打开，则 **PTRACE_EVENT_EXEC** 停止发生。

 * 如果此时线程组领导者已报告其 **PTRACE_EVENT_EXIT** 停止，则跟踪器似乎看到死去的线程领导者“从什么地方重新出现”。（注意：线程组领导不会通过 *WIFEXITED(status)* 报告死亡，直到至少有一个其它活动线程。这消除了跟踪器看到它死亡然后重新出现的可能性。）如果线程组领导还活着，对于跟踪器来说，这可能看起来好像线程组领导从与它进入的系统调用不同的系统调用返回，或者甚至 “从系统调用返回，即使它不在任何系统调用中”。如果线程组领导者没有被跟踪（或被不同的跟踪器跟踪），那么在 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 期间它将看起来好像它已成为执行的被追踪者的跟踪器的被追踪者。

以上所有效果都是 tracee 中线程 ID 变化的产物。

**PTRACE_O_TRACEEXEC** 选项是处理这种情况的推荐工具。首先，它启用 **PTRACE_EVENT_EXEC** 停止，这发生在 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 返回之前。在此种停止中，跟踪器可以使用 **PTRACE_GETEVENTMSG** 来检索被跟踪者以前的线程 ID。（这个特性是在 Linux 3.0 中引入的。）其次，**PTRACE_O_TRACEEXEC** 选项在 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 上禁用旧的 **SIGTRAP** 生成。

当跟踪器收到 **PTRACE_EVENT_EXEC** 停止通知时，保证除了这个被跟踪者和线程组领导者之外，进程中没有其它线程是活着的。

在收到 **PTRACE_EVENT_EXEC** 停止通知时，跟踪器应清除其所有描述此进程的线程的内部数据结构，并仅保留一个数据结构——一个描述仍在运行的单个被追踪者的数据结构，即
```
    thread ID == thread group ID == process ID.
```

比如，两个线程同时调用 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)：

       *** we get syscall-enter-stop in thread 1: **
       PID1 execve("/bin/foo", "foo" <unfinished ...>
       *** we issue PTRACE_SYSCALL for thread 1 **
       *** we get syscall-enter-stop in thread 2: **
       PID2 execve("/bin/bar", "bar" <unfinished ...>
       *** we issue PTRACE_SYSCALL for thread 2 **
       *** we get PTRACE_EVENT_EXEC for PID0, we issue PTRACE_SYSCALL **
       *** we get syscall-exit-stop for PID0: **
       PID0 <... execve resumed> )             = 0

如果 **PTRACE_O_TRACEEXEC** 选项对正在执行的被跟踪者无效，并且如果被跟踪者是 **PTRACE_ATTACH**ed 而非 **PTRACE_SEIZE**d，则内核会在 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 返回后向被跟踪者发送一个额外的 **SIGTRAP**。这是一个普通信号（类似于可以通过 *kill -TRAP* 生成的信号），而不是一种特殊的 ptrace-stop。为此信号使用 **PTRACE_GETSIGINFO** 会返回设置为 0 (*SI_USER*) 的 *si_code*。该信号可能会被信号掩码阻塞，因此可能会（很晚）被传递。

通常，跟踪器（例如，[strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html)）不想向用户显示这个额外的 post-execve **SIGTRAP** 信号，并且会抑制它传递给被跟踪者（如果 **SIGTRAP** 设置为 **SIG_DFL**，则它是一个 killing 信号）。但是，确定要抑制哪个 **SIGTRAP** 并不容易。推荐的方法是设置 **PTRACE_O_TRACEEXEC** 选项或使用 **PTRACE_SEIZE** 从而抑制这种额外的 **SIGTRAP**。

### 真正的父进程
ptrace API 使(烂)用基于 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 的标准 UNIX 父/子进程信号。当子进程被其它进程跟踪时，这种用法导致进程的真正父进程停止接收几种 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 通知。

其中许多缺陷已得到修复，但截至 Linux 2.6.38，仍有几个缺陷存在； 请参阅下面的 BUGS。

从 Linux 2.6.38 开始，以下内容被认为可以正常工作：

 * 信号导致的退出/死亡首先报告给跟踪器，然后，当跟踪器使用了 [waitpid(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html) 的结果时，报告给真正的父进程（仅当整个多线程进程退出时才报告给真正的父进程）。如果跟踪器和真正的父进程是同一个进程，则报告只发送一次。

## 返回值

成功时，**PTRACE_PEEK*** 请求返回请求的数据（但请参阅 NOTES），**PTRACE_SECCOMP_GET_FILTER** 请求返回 BPF 程序中的指令数，**PTRACE_GET_SYSCALL_INFO** 请求返回内核可写入的字节数，其它请求返回 0。

出错时，所有请求都返回 -1，并且设置 *[errno](https://man7.org/linux/man-pages/man3/errno.3.html)* 以指示错误。由于成功的 **PTRACE_PEEK*** 请求返回的值可能是 -1，因此调用者必须在调用前清除*[errno](https://man7.org/linux/man-pages/man3/errno.3.html)*，然后再检查它以确定是否发生错误。

## 错误

**EBUSY** （仅限 i386）分配或释放调试寄存器时出错。

**EFAULT** 试图读取或写入跟踪器或被跟踪者内存中的无效区域，可能是因为该区域未映射或不可访问。不幸的是，在 Linux 下，此故障的不同变体或多或少会任意返回 **EIO** 或 **EFAULT**。

**EINVAL** 试图设置无效选项。

**EIO** *request* 参数无效，或者试图从跟踪器或被跟踪者的内存中读取或写入无效区域，或者存在字对齐违规，或者在重新启动请求期间指定了无效信号。

**EPERM** 无法追踪指定的进程。这可能是因为跟踪器没有足够的权限（所需的能力是 **CAP_SYS_PTRACE**）；出于显而易见的原因，非特权进程无法跟踪无法向其发送信号的进程或运行 set-user-ID/set-group-ID 程序的进程。或者，进程可能已经被跟踪，或者（在 2.6.26 之前的内核上）是 [init(1)](https://man7.org/linux/man-pages/man1/init.1.html) (PID 1)。

**ESRCH** 指定的进程不存在，或者当前没有被调用者跟踪，或者没有停止（对于需要停止被跟踪者的请求）。

## 符合

SVr4，4.3BSD。

## 注意

尽管 **ptrace()** 的参数根据给定的原型进行解释，但 glibc 目前将 **ptrace()** 声明为可变参数函数，仅 *request* 参数固定。建议始终提供四个参数，即使请求的操作不使用它们，将未使用/忽略的参数设置为 *0L* 或 *`(void *) 0`*。

在 2.6.26 之前的 Linux 内核中，可能无法跟踪 PID 为 1 的进程 [init(1)](https://man7.org/linux/man-pages/man1/init.1.html) 。

即使跟踪器调用 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)，tracees 父进程仍然是跟踪器。

内存内容和 USER 区域的布局非常特定于操作系统和体系结构。提供的偏移量和返回的数据可能与 *struct user* 的定义不完全匹配。

“字” 的大小由操作系统变体决定（例如，对于 32 位 Linux，它是 32 位）。

此页面记录了 **ptrace()** 调用当前在 Linux 中的工作方式。它的行为在其他版本的 UNIX 上有很大不同。在任何情况下，**ptrace()** 的使用都高度特定于操作系统和体系结构。

### Ptrace 访问模式检查

内核用户空间 API 的各个部分（不仅仅是 **ptrace()** 操作）需要所谓的 “ptrace 访问模式” 检查，其结果决定了某个操作是否被允许（或者，在某些情况下，会导致 “读取” 操作返回清理的数据）。这些检查是在一个进程可以检查有关另一个进程的敏感信息或在某些情况下修改另一个进程的状态的情况下执行的。检查基于以下因素，例如两个进程的凭据和能力、“目标”进程是否可转储，以及任何已启用的 Linux 安全模块 (LSM) 执行的检查结果 —— 例如 SELinux、Yama ，或 Smack —— 以及由 commoncap LSM（总是被调用）。

在 Linux 2.6.27 之前，所有访问检查都属于单一类型。从 Linux 2.6.27 开始，区分了两种访问模式级别：

**PTRACE_MODE_READ**
用于“读取”操作或其它危险性较低的操作，例如：[get_robust_list(2)](https://man7.org/linux/man-pages/man2/get_robust_list.2.html)；[kcmp(2)](https://man7.org/linux/man-pages/man2/kcmp.2.html)；读取 */proc/[pid]/auxv*，*/proc/[pid]/environ*，或 */proc/[pid]/stat*；或一个 */proc/[pid]/ns/** 文件的 [readlink(2)](https://man7.org/linux/man-pages/man2/readlink.2.html)。

**PTRACE_MODE_ATTACH**
用于“写”操作，或其它更危险的操作，例如：ptrace 附加 (**PTRACE_ATTACH**) 到另一个进程或调用 [process_vm_writev(2)](https://man7.org/linux/man-pages/man2/process_vm_writev.2.html)。（**PTRACE_MODE_ATTACH** 实际上是 Linux 2.6.27 之前的默认设置。）

从 Linux 4.5 开始，上述访问模式检查与以下修饰符之一结合（或）：

**PTRACE_MODE_FSCREDS**
使用调用者的文件系统 UID 和 GID（请参阅 [credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html)）或 LSM 检查的有效权能。

**PTRACE_MODE_REALCREDS**
使用调用者的真实 UID 和 GID 或允许的权能进行 LSM 检查。这实际上是 Linux 4.5 之前的默认设置。

由于将凭证修饰符之一与上述访问模式之一组合是很典型的，因此在内核源代码中为组合定义了一些宏：

**PTRACE_MODE_READ_FSCREDS**
定义为 **PTRACE_MODE_READ | PTRACE_MODE_FSCREDS**

**PTRACE_MODE_READ_REALCREDS**
定义为 **PTRACE_MODE_READ | PTRACE_MODE_REALCREDS**

**PTRACE_MODE_ATTACH_FSCREDS**
定义为 **PTRACE_MODE_ATTACH | PTRACE_MODE_FSCREDS**

**PTRACE_MODE_ATTACH_REALCREDS**
定义为 **PTRACE_MODE_ATTACH | PTRACE_MODE_REALCREDS**

还有另一个修饰符可以与访问模式进行 OR 运算：

**PTRACE_MODE_NOAUDIT**（从 Linux 3.3 开始）

不要审计此访问模式检查。此修饰符用于 ptrace 访问模式检查（例如读取 */proc/[pid]/stat* 时的检查），它只会导致输出被过滤或清理，而不是导致将错误返回给调用者。在这些情况下，访问文件不是安全违规，也没有理由生成安全审核记录。此修饰符禁止为特定访问检查生成此类审计记录。

请注意，本小节中描述的所有 **PTRACE_MODE_*** 常量都是内核内部的，对用户空间不可见。此处提到的常量名称是为了标记为各种系统调用和访问各种伪文件（例如，在 /proc 下）执行的各种 ptrace 访问模式检查。这些名称在其它手册页中用于提供标记不同的内核检查的简单速记。

ptrace 访问模式检查的所使用的算法决定了是否允许调用进程对目标进程执行相应的操作。（在打开 */proc/[pid]* 文件的场景下，“调用进程” 就是打开文件的进程，具有对应 PID 的进程就是 “目标进程”。）算法如下：

1. 如果调用线程和目标线程在同一个线程组中，则始终允许访问。

2. 如果访问模式指定了 **PTRACE_MODE_FSCREDS**，那么对于下一步的检查，使用调用者的文件系统 UID 和 GID。（如 [credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html) 中所述，文件系统 UID 和 GID 几乎总是与相应的有效 ID 具有相同的值。）
否则，访问模式指定 **PTRACE_MODE_REALCREDS**，因此使用调用者的真实 UID 和 GID 进行下一步的检查。（大多数检查调用者 UID 和 GID 的 API 使用有效 ID。由于历史原因，**PTRACE_MODE_REALCREDS** 检查使用真实 ID。）

3. 如果以下情况都不为真，则拒绝访问：
    * 目标的真实、有效和保存的用户 ID 与调用者的用户 ID 相匹配，目标的真实、有效和保存的组 ID 与调用者的组 ID 相匹配。
    * 调用者在目标的用户命名空间中具有 **CAP_SYS_PTRACE** 权能。

4. 如果目标进程 “dumpable” 属性的值不是 1（**SUID_DUMP_USER**；请参阅 [prctl(2)](https://man7.org/linux/man-pages/man2/prctl.2.html) 中 P**R_SET_DUMPABLE** 的讨论），并且调用者在目标进程的用户命名空间中不具有 **CAP_SYS_PTRACE** 权能，则拒绝访问。

5. 内核 LSM *security_ptrace_access_check()* 接口被调用以查看是否允许 ptrace 访问。结果取决于 LSM。这个接口在 commoncap LSM 中的实现执行以下步骤：
  a) 如果访问模式包含 **PTRACE_MODE_FSCREDS**，则在下面的检查中使用调用者的 *有效* 权能集合； 否则（访问模式指定 **PTRACE_MODE_REALCREDS**，所以）使用调用者的 *被允许的* 权能集合。
  b) 如果以下情况都不为真，则拒绝访问：
     * 调用者和目标进程在同一个用户命名空间中，调用者的权能是目标进程允许的权能的超集。
     * 调用者在目标进程的用户命名空间中具有 **CAP_SYS_PTRACE** 权能。
       请注意，commoncap LSM 不区分 **PTRACE_MODE_READ** 和 **PTRACE_MODE_ATTACH**。

6. 如果前面的任何步骤都没有拒绝访问，则允许访问。

### /proc/sys/kernel/yama/ptrace_scope

在安装了 Yama Linux 安全模块 (LSM) 的系统上（即，内核配置了 **CONFIG_SECURITY_YAMA**），*/proc/sys/kernel/yama/ptrace_scope* 文件（自 Linux 3.4 起可用）可用于限制通过 **ptrace()** 跟踪进程的能力（因此也能够使用诸如 [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) 和 [gdb(1)](https://man7.org/linux/man-pages/man1/gdb.1.html) 之类的工具）。此类限制的目的是防止攻击升级，从而受感染的进程可以通过 ptrace-attach 附加到用户拥有的其它敏感进程（例如，GPG 代理或 SSH 会话），以获得可能存在于内存中的额外凭据，从而扩大攻击范围。

更准确地说，Yama LSM 限制了两种类型的操作：

 * 任何执行 ptrace 访问模式 **PTRACE_MODE_ATTACH** 检查的操作——例如，**ptrace()** **PTRACE_ATTACH**。 （请参阅上面的 “Ptrace 访问模式检查” 的讨论。）
 * **ptrace()** **PTRACE_TRACEME**。

具有 **CAP_SYS_PTRACE** 权能的进程可以使用以下值之一更新 */proc/sys/kernel/yama/ptrace_scope* 文件：

0（“经典 ptrace 权限”）
对执行 **PTRACE_MODE_ATTACH** 检查的操作没有其它限制（除了由 commoncap 和其它 LSM 强加的限制）。
**PTRACE_TRACEME** 的使用不变。

1（“受限ptrace”）[默认值]
当执行需要 **PTRACE_MODE_ATTACH** 检查的操作时，调用进程必须在目标进程的用户命名空间中具有 **CAP_SYS_PTRACE** 权能，或者它必须与目标进程具有预定义的关系。默认情况下，预定义的关系是目标进程必须是调用者的后代。
目标进程可以使用 [prctl(2)](https://man7.org/linux/man-pages/man2/prctl.2.html) P**R_SET_PTRACER** 操作来声明一个额外的 PID，该 PID 允许在目标进程上执行 **PTRACE_MODE_ATTACH** 操作。有关更多详细信息，请参阅内核源文件 *Documentation/admin-guide/LSM/Yama.rst*（或 Linux 4.13 之前的 *Documentation/security/Yama.txt*）。
**PTRACE_TRACEME** 的使用不变。

2（“仅限管理员附加”）
只有在目标进程的用户命名空间中具有 **CAP_SYS_PTRACE** 权能的进程才能执行 **PTRACE_MODE_ATTACH** 操作或跟踪使用 **PTRACE_TRACEME** 的子进程。

3（“无附加”）
任何进程都不能执行 **PTRACE_MODE_ATTACH** 操作或跟踪使用 **PTRACE_TRACEME** 的子进程。
一旦此值写入文件，就无法更改。

关于值 1 和 2，请注意，创建新的用户命名空间有效地消除了 Yama 提供的保护。这是因为父进程用户命名空间中的有效 UID 与子进程命名空间（以及进一步删除的该命名空间的后代）的创建者的 UID 匹配的进程，在子进程用户命名空间中执行操作时具有所有权能（包括 **CAP_SYS_PTRACE**）。因此，当进程尝试使用用户命名空间对自身进行沙箱处理时，它会无意中削弱 Yama LSM 提供的保护。

### C 库/内核差异

在系统调用级别，**PTRACE_PEEKTEXT**、**PTRACE_PEEKDATA** 和 **PTRACE_PEEKUSER** 请求具有不同的 API：它们将结果存储在 *data* 参数指定的地址，返回值是错误标志。glibc 包装函数提供上面描述中给出的 API，结果通过函数返回值返回。

## 缺陷
在具有 2.6 内核头文件的主机上， **PTRACE_SETOPTIONS** 声明的值与 2.4 的值不同。这会导致使用 2.6 内核头文件编译的应用程序在 2.4 内核上运行时失败。这可以通过将 **PTRACE_SETOPTIONS** 重新定义为 **PTRACE_OLDSETOPTIONS**（如果已定义）来解决。

Group-stop 通知被发送给跟踪器，但不会发送给真正的父进程。 最后确认于 2.6.38.6。

如果线程组领导者被跟踪，且通过调用 [_exit(2)](https://man7.org/linux/man-pages/man2/_exit.2.html) 退出，则将为其发生 **PTRACE_EVENT_EXIT** 停止（如果请求的话），但在所有其它线程退出之前不会传递后续的 **WIFEXITED** 通知。如上面的解释，如果其它线程之一调用 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)，则 *永远不会* 报告线程组领导者的死亡。如果此跟踪器未跟踪执行 exec 的线程，则跟踪器将永远不会知道 [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) 发生了。一种可能的解决方法是 **PTRACE_DETACH** 线程组领导者，而不是在这种情况下重新启动它。最后确认于 2.6.38.6。

**SIGKILL** 信号可能仍会在实际的信号死亡之前导致 **PTRACE_EVENT_EXIT** 停止。这在未来可能会改变；**SIGKILL** 旨在总是立即终止任务，即使在 ptrace 下也一样。最后在 Linux 3.13 上确认。

如果将信号发送给被跟踪者，但跟踪器抑制了传递，则某些系统调用会返回 **EINTR**。（这是非常典型的操作：它通常由调试器在每次附加时完成，以免引入虚假的 **SIGSTOP**）。从 Linux 3.2.9 开始，以下系统调用受到影响（此列表可能不完整）：[epoll_wait(2)](https://man7.org/linux/man-pages/man2/epoll_wait.2.html)，及从一个 [inotify(7)](https://man7.org/linux/man-pages/man7/inotify.7.html) 文件描述符 [read(2)](https://man7.org/linux/man-pages/man2/read.2.html)。此缺陷的常见症状是，当你使用如下命令附加到静止进程时
```
    strace -p <process-ID>
```
然后，不像通常的和预期的那样的单行输出，例如
```
    restart_syscall(<... resuming interrupted call ...>_
```
或
```
    select(6, [5], NULL, [5], NULL_
```
（'_'表示光标位置），你观察到不止一行。 例如：
```
    clock_gettime(CLOCK_MONOTONIC, {15370, 690928118}) = 0
    epoll_wait(4,_
```
这里看不到的是，在 [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) 附加到它之前，该进程在 [epoll_wait(2)](https://man7.org/linux/man-pages/man2/epoll_wait.2.html) 中被阻塞。附加导致 [epoll_wait(2)](https://man7.org/linux/man-pages/man2/epoll_wait.2.html) 返回用户空间并返回错误 **EINTR**。在这个特殊的例子里，程序通过检查当前时间来对 **EINTR** 做出反应，然后再次执行 [epoll_wait(2)](https://man7.org/linux/man-pages/man2/epoll_wait.2.html)。（没有预料到此类 “杂散” **EINTR** 错误的程序可能会在 [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) 附加时以意外的方式运行。）

与正常规则相反，**ptrace()** 的 glibc 包装器可以将 *[errno](https://man7.org/linux/man-pages/man3/errno.3.html)* 设置为零。

参考资料：
[man 手册原文](https://man7.org/linux/man-pages/man2/ptrace.2.html)
[Linux ptrace系统调用详解：利用 ptrace 设置硬件断点](https://blog.csdn.net/Rong_Toa/article/details/112155847)
[strace实现原理：ptrace系统调用](https://rtoax.blog.csdn.net/article/details/109825818)
[How debuggers work: Part 1 - Basics](https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1)
[How debuggers work: Part 2 - Breakpoints](https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints)
[How debuggers work: Part 3 - Debugging information](https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information)
[GDB Internals](http://www.deansys.com/doc/gdbInternals/gdbint_toc.html)
[GDB原理之ptrace实现原理](https://mp.weixin.qq.com/s?__biz=MzA3NzYzODg1OA==&mid=2648464351&idx=1&sn=af5e3b8f97da2cfe92cab1330e1e0563&chksm=8766067ab0118f6cc37d8799b97aa1e5053e8dc2408ec2b8ea04311a35d62b62b652de6a7f03&mpshare=1&scene=1&srcid=1102wsQDCpS40e8hzkkaSGGC&sharer_sharetime=1604275938625&sharer_shareid=d21dd3bcaa72a23e9bad061d738dba43&key=0eba2d0597cf1f68faeccd6a77be1767c15e8f97d4539b68f375aeaaee51c4d78bbd3b7baeb3743ce9457180a0d9c7c76284edfc1aaa5260cc5669e1ca0472ad922daca498291f700f36a9a5e4b1f948594d4d279aa173cf76d77bb62879c3ceffa9810651f7681b01cc372034421da43de58216001df6a7597eb72a339530c4&ascene=1&uin=NjczMTY5MzU3&devicetype=Windows+10+x64&version=6300002f&lang=zh_CN&exportkey=A8dvjFWBwW0tDf75BQmT44E%3D&pass_ticket=vOx8u6ogNAJmNnT3EE0lOTdQ0qmiIGrghCQp5S16Nscf8vYHWbSpOmbPZoKX6ejj&wx_header=0)
[ptrace理解](https://www.cnblogs.com/mysky007/p/11047943.html)

Done.
