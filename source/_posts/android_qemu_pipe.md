---
title: Android QEMU 高速管道
date: 2017-12-21 13:05:49
categories: 虚拟化
tags:
- 虚拟化
- Android开发
- 翻译
---

介绍
-------

Android 模拟器实现了一个特殊的虚拟设备，用于提供客户 Android 系统和模拟器本身 ***非常*** 快速的通信通道。
<!--more-->
在客户 Android 系统端，用法非常简单，如下：

  1/ 打开 /dev/qemu_pipe 设备文件来读和写
     注意：自 Linux 3.10 开始，设备被重命名为了 `/dev/goldfish_pipe`，但行为完全一样。

  2/ 写入描述你想要连接的服务，且以 0 结束的字符串。

  3/ 简单地使用 read() 和 write() 来与服务通信。

换句话说：
```
   fd = open("/dev/qemu_pipe", O_RDWR);
   const char* pipeName = "<pipename>";
   ret = write(fd, pipeName, strlen(pipeName)+1);
   if (ret < 0) {
       // error
   }
   ... ready to go
```

其中 `<pipename>` 是你想要使用的特定模拟器服务的名字。本文档在后面列出了支持的模拟器服务的名字。

实现细节
-------------

在模拟器的源码树中：
```
    ./hw/android/goldfish/pipe.c  实现虚拟驱动。
    ./hw/android/goldfish/pipe.h 提供了任何模拟器 pipe 服务必须提供的接口。
    ./android/hw-pipe-net.c 包含网络 pipe 服务（比如 'tcp' 和 'unix'）的实现。更多详情参考下文。
```

***模拟器根据所使用的 Android Linux 内核的不同，而有两个不同的版本，即使用 goldfish 内核的 QEMU1 和使用 ranchu 内核的 QEMU2。上面列出的这些文件，是用于 QEMU1 的相关文件。这些文件在 emulator 2.3 release 版本代码库中的具体位置分别为 `qemu/android/qemu1/hw/android/goldfish/pipe.c`，`qemu/android/qemu1/include/hw/android/goldfish/pipe.h`，而 `hw-pipe-net.c` 文件则已经被移除了。关于网络虚拟化的代码，分布于 `qemu/net/`、`qemu/hw/net/` 目录下。***


在内核源码树中：
```
    drivers/misc/qemupipe/qemu_pipe.c 包含在客户系统内，可通过 /dev/qemu_pipe
 访问的驱动源码。
```

***这文件当然是在模拟器版的 Linux 内核中，也就是从 `git clone https://android.googlesource.com/kernel/goldfish` clone 下来的代码中，且在 `android-goldfish-3.4` 分支中。如在前面提到的，自 3.10 版内核开始，设备被重命名为了 `/dev/goldfish_pipe`，实现该设备的驱动程序文件也变为了 `goldfish/drivers/platform/goldfish/goldfish_pipe.c`***

设备 / 驱动协议细节
--------------------------

设备和驱动使用一个 I/O 内存页和一个 IRQ 来通信。

  - 驱动写入不同的 I/O 寄存器来向设备发送命令。

  - 设备触发一个 IRQ 通知驱动某一事件发生了。

  - 驱动读取 I/O 寄存器获得它的最新命令的状态，或者在中断的情况下发生的事件的列表。

客户系统内每个打开的 `/dev/qemu_pipe` 的文件描述符对应一个由驱动分配的 32 位的 **'通道'** 值。

下面的内容描述了驱动向设备发送的不同命令。以 `REG_` 开头的可变的名字对应于 32 位的 I/O 寄存器：

  0/ 通道和地址值

   依赖于客户系统的 CPU 架构，每个通信通道由一个非零的 32 位或 64 位值标识。

   由内核向模拟器发送的通道值为：

```
        void write_channel(channel) {
        #if 64BIT_GUEST_CPU
          REG_CHANNEL_HIGH = (channel >> 32);
        #endif
          REG_CHANNEL = (channel & 0xffffffffU);
        }
```

  类似地，当给模拟器传递一个内核地址时：
```
        void write_address(buffer_address) {
        #if 64BIT_GUEST_CPU
          REG_ADDRESS_HIGH = (buffer_address >> 32);
        #endif
          REG_ADDRESS = (buffer_address & 0xffffffffU);
        }
```
  1/ 创建一个新的通道

   驱动用来表示客户系统打开了将由名称 '<channel>' 标识的 `/dev/qemu_pipe`：
```
        write_channel(<channel>)
        REG_CMD = CMD_OPEN
```

   重要：`<channel>` 从不应该为 0

  2/ 关闭一个通道

   驱动用来表示客户系统调用了通道文件描述符的 'close'。

```
        write_channel(<channel>)
        REG_CMD = CMD_CLOSE
```

  3/ 向通道写入数据

   对应于客户系统在通道的文件描述符上执行 `write()` 或 `writev()`。这个命令用于发送一个单独的内存缓冲区：

```
        write_channel(<channel>)
        write_address(<buffer-address>)
        REG_SIZE    = <buffer-size>
        REG_CMD     = CMD_WRITE_BUFFER

        status = REG_STATUS
```

  注意：`<buffer-address>` 是 ***客户系统*** 的缓冲区地址，而不是物理的/内核的。

  重要：通过这个命令发送的缓冲区 ***应该总是*** 被完整的包含在一个客户系统的内存页内。这是强制的，用以简化驱动程序和设备。
  如果一个 `write()` 跨越多个客户系统的内存页，驱动将连续触发多个，对客户端透明的 `CMD_WRITE_BUFFER` 命令。

  `REG_STATUS` 返回的值应该是：

   `> 0`  写入 pipe 的字节数
   `0`   表示流结束状态
   `< 0`  负值错误码（参考下文）。

  一个重要的错误码是 `PIPE_ERROR_AGAIN`，用于表示写操作还不能执行。参考 `CMD_WAKE_ON_WRITE` 了解更多内容。

  4/ 从通道读取数据

  对应于客户系统在通道的文件描述符上执行 `read()` 和 `readv()`。
```
        write_channel(<channel>)
        write_address(<buffer-address>)
        REG_SIZE    = <buffer-size>
        REG_CMD     = CMD_READ_BUFFER

        status = REG_STATUS
```

  有着相同的缓冲区地址/长度限制和相同的错误码集合。

  5/ 等待写能力

  如果 `CMD_WRITE_BUFFER` 返回 `PIPE_ERROR_AGAIN`，且文件描述符不是处于非阻塞模式，驱动必须把客户端任务放进一个等待队列中，直到 pipe 服务可以再次接受数据。

  在此之前，驱动将执行：
```
        write_channel(<channel>)
        REG_CMD     = CMD_WAKE_ON_WRITE
```

  以向虚拟设备表明，它正在等待，并应该在 pipe 再次变得可写时被唤醒。稍后将解释这是如何完成的。

  6/ 等待读能力

  这与 `CMD_WAKE_ON_WRITE` 一样，但要替换为读能力。

```
        write_channel(<channel>)
        REG_CMD     = CMD_WAKE_ON_READ
```

  7/ 轮询（polling）可读/可写状态

  下面的命令由驱动用于实现 `select()`，`poll()` 和 `epoll()` 系统调用，其中包含一个 pipe 通道。

```
        write_channel(<channel>)
        REG_CMD     = CMD_POLL
        mask = REG_STATUS
```

  REG_STATUS 返回的掩码值是自上次调用以来，哪些事件可用/已经发生的位标记的混合。参考 `PIPE_POLL_READ` / `PIPE_POLL_WRITE` / `PIPE_POLL_CLOSED`。

  8/ 向驱动通知事件

  设备可以通过生成它的 IRQ 来向驱动通知事件。驱动的中断处理器随后将读取一个以单独的 0 值结束的通道的 (channel,mask) 对的列表。

  换句话说，驱动的中断处理器将执行：
```
        for (;;) {
            channel = REG_CHANNEL
            if (channel == 0)  // END OF LIST
                break;

            mask = REG_WAKES  // BIT FLAGS OF EVENTS
            ... process events
        }
```

  通过这个列表报告的事件很简单：
```
       PIPE_WAKE_READ   :: 通道现在可读。
       PIPE_WAKE_WRITE  :: 通道现在可写。
       PIPE_WAKE_CLOSED :: pipe 服务关闭了连接
```

  如果 `CMD_WAKE_ON_READ` 或 `CMD_WAKE_ON_WRITE`（分别地）为给定通道发出，则 `PIPE_WAKE_READ` 和 `PIPE_WAKE_WRITE` 仅向它报告。

  `PIPE_WAKE_CLOSED` 可以在任何时间通知。

  9/ 通过参数块的更快的读/写

  最近的 Goldfish 内核实现了一种更快的方式来执行读和写，它每个操作执行一个单独的 I/O 写操作（当通过 KVM 或 HAX 模拟 x86 系统时很有用）。

  它使用下面同时为虚拟设备和内核所知的结构，在 `$QEMU/hw/android/goldfish/pipe.h` 中定义：

  32 位客户系统 CPU 的是：
```
        struct access_params {
            uint32_t channel;
            uint32_t size;
            uint32_t address;
            uint32_t cmd;
            uint32_t result;
            /* reserved for future extension */
            uint32_t flags;
        };
```

  64 位的是：
```
        struct access_params_64 {
            uint64_t channel;
            uint32_t size;
            uint64_t address;
            uint32_t cmd;
            uint32_t result;
            /* reserved for future extension */
            uint32_t flags;
        };
```

  这是把多个参数打包进一个单独的结构的简单的方式。起初的，比如，在启动时间，内核将分配一个这样的结构，并这样传递它的物理地址：
```
       PARAMS_ADDR_LOW  = (params & 0xffffffff);
       PARAMS_ADDR_HIGH = (params >> 32) & 0xffffffff;
```

  然后对于每个操作，它将做这样一些事情：
```
        params.channel = channel;
        params.address = buffer;
        params.size = buffer_size;
        params.cmd = CMD_WRITE_BUFFER (or CMD_READ_BUFFER)

        REG_ACCESS_PARAMS = <any>

        status = params.status
```

  向 `REG_ACCESS_PARAMS` 写入将触发操作，比如 QEMU 将读取参数块的内容，使用它的字段来执行操作，然后把返回值写回 `params.status`。

10/ v2 pipe: 通过命令缓冲区和缓冲区列表的更快的读/写

  批量 access_params 依然一次只能执行一个缓冲区传输。这对于每次传输只使用一个内存页的应用来说是 OK 的，但对于许多延迟/吞吐量敏感的应用，比如 OpenGL 和 ADB push/pull，每次操作一个缓冲区/页是不够的。

  理想情况下，如果客户系统想要传输缓冲区，那么应该尽可能地像在一个步骤中完成。

  但是，客户系统的内存页更可能是分散在多个位置，且不是物理上连续的，使得这变得很困难。

  v2 Goldfish pipe 驱动和设备修改了寄存器集合以及 device/pipe 结构，以允许 `goldfish_pipe_read_write` 的每个循环传输更多的缓冲区，为所有吞吐量受限的 pipe 用户提升性能。

  有两个关键的新结构要考虑。

  1. v2 pipe 添加了一个 `struct goldfish_pipe_command` 来表示在一个 I/O 事务中传输多个缓冲区：
```
    /* A per-pipe command structure, shared with the host */
    struct goldfish_pipe_command {
    	s32 cmd;		/* PipeCmdCode, guest -> host */
    	s32 id;			/* pipe id, guest -> host */
    	s32 status;		/* command execution status, host -> guest */
    	s32 reserved;	/* to pad to 64-bit boundary */
    	union {
    		/* Parameters for PIPE_CMD_{READ,WRITE} */
    		struct {
    			u64 ptrs[MAX_BUFFERS_PER_COMMAND]; 	/* buffer pointers, guest -> host */
    			u32 sizes[MAX_BUFFERS_PER_COMMAND];	/* buffer sizes, guest -> host */
    			u32 buffers_count;					/* number of buffers, guest -> host */
    			s32 consumed_size;					/* number of consumed bytes, host -> guest */
    		} rw_params;
    	};
    };
```

  对于每个 pipe fd，都有一个对应的 `goldfish_pipe_command` 结构，它可以持有最多 `MAX_BUFFERS_PER_COMMAND` （具体地说，目前为 10^2）个缓冲区。缓冲区中包含客户系统内在 rw_params 中追踪的数据。需要做些努力来使列表尽可能短（比如，合并物理上连续的页等）。

  2. 以前，对于主机触发的传输，我们遍历具有挂起的 actions (PIPE_WAKE_***) 的 pipe 的链表，称为 "signaled pipes" 列表，中断处理器在每次传输发生时将粗略地处理 1-3 个这样的 pipes。这不是很理想，因为 a) 我们在中断处理器中执行任务和 b) 在客户系统中运行内核驱动与在主机上运行虚拟设备之间可能有大量的事务（事务很慢）。

  因此，pipe v2 记录了一系列来自于主机 IRQ 触发的要处理的 pipe：
```
    /* Device-level set of buffers shared with the host */
    struct goldfish_pipe_dev_buffers {
    	struct open_command_param open_command_params;
    	struct signalled_pipe_buffer signalled_pipe_buffers[MAX_SIGNALLED_PIPES];
    };
```
  且每个中断处理器一次可以处理最多 MAX_SIGNALLED_PIPES 个。此外，唤醒被通知的 pipes 的工作被留给了一个 bottom-half 处理器，且不在中断上下文中完成。

  11/ goldfish_dma：从主机系统直接访问客户系统上的内存

  有时，对于同时需要非常高的吞吐量及非常低的延时的应用，比如视频回放，跳过主机/客户事务及收集非连续的缓冲区，而直接写入对主机系统也可见的内存可能是很有用的。

  为了做到这一点，`goldfish_dma` 允许任何 pipe fd 用 ioctls 在内核空间分配一块连续的物理区域，然后从客户系统的用户空间 mmap() 到那个物理区域。这些区域也可以跨进程共享，比如通过 gralloc。

  `goldfish_dma` 内核驱动将  `struct goldfish_dma_context` 添加到所有的 `struct goldfish_pipe` 实例中：
```
    struct goldfish_dma_context {
        struct kref kref; /* kref to safely free dma region */
        struct device* pdev_dev; /* pointer to feed to dma_***_coherent */
        void* dma_vaddr; /* kernel vaddr of dma region */
        dma_addr_t dma_bus_addr; /* kernel dma_addr_t of dma region */
        u64 dma_size; /* size of dma region */
    	u64 phys_begin; /* paddr of dma region */
    	unsigned long pfn; /* pfn of dma region */
    	u64 phys_end; /* paddr of dma region + dma_size */
    	bool locked; /* marks whether the region is currently in use */
        unsigned long refcount; /* For debugging purposes only. */
        struct goldfish_pipe_command* command_buffer; /* For safe freeing of DMA regions,
        we need another pipe command buffer that lives longer than the pipe instance */
    };

    Interface (guest side):
```

  客户系统用户在一个 goldfish pipe fd 上调用 goldfish_dma_alloc (ioctls) 以及 mmap()，这意味着它想要高速访问主机可见的内存。
    
  然后客户系统可以写入 `mmap()` 返回的指针，随后这些写入在主机上变得立即可见而无需 BQL 及上下为切换。

  主要的数据结构跟踪状态是 `struct goldfish_dma_context`，它作为一个额外的指针成员而被包含在 `struct goldfish_pipe` 中。每个这样的上下文可能与由一个物理地址和大小描述的分配的 DMA 区域关联，且对于每个 pipe fd 只允许有一次分配。更多的分配需要 pipe fd 的更多 `open()`。
    
  `dma_alloc_coherent()` 被用于获得连续的物理内存区域，然后我们在客户系统和主机系统上分配并与这一区域交互，通过如下的 ioctls：

  - LOCK：为数据访问锁定 DMA 区域。
  - UNLOCK：解锁 DMA 区域。这也可能在主机系统上通过 WAKE_ON_UNLOCK_DMA 完成。
  - CREATE_REGION：为一个 DMA 区域初始化大小信息。
  - GETOFF：发送物理地址给客户系统驱动。
  - (UN)MAPHOST：使用 `goldfish_pipe_cmd` 告诉主机系统，(un)map 与当前 dma 上下文关联的客户系统的物理地址。这使物理的连续内存对于主机系统 (不) 可见。
    
  客户系统用户空间使用 `mmap()` 获得一个指向 DMA 内存的指针，它也用 `dma_alloc_coherent()` 来延迟分配内存。（在最后一个 pipe `close()` 中，该区域被释放）。


  `mmap()` 映射的区域可以处理非常高带宽的传输，且 pipe 操作可与处理同步和命令通信一起使用。

  在内核驱动中虚拟设备有这些 hooks：

  1. UNLOCK_DMA：从主机系统中解锁 DMA 区域。
  2. MAP/UNMAP_HOST：这是一个 pipe 命令，用于告诉主机获得一个主机的 `void*` 指向特定的客户系统 RAM 物理地址。当通过 `ioctl()` 在客户系统的内核中分配一个新的 DMA 缓冲区时触发，以使主机系统也可以看到相同的区域。

  然后主机系统将调用 `cpu_physical_memory_(un)map` 并在一个全局的数据结构  (DmaMap) 中追踪所有当前分配的区域。如果我们拍快照，我们将需要 remap 那些区域，一些东西也由 `DmaMap` 处理。

  2. UNLOCK_DMA：这个 pipe 命令通知客户系统内核，主机系统完成了对 DMA 区域的处理。在大多数简单的用户案例中，客户系统将首先锁定它的 DMA 区域然后写入它，假设处理完成后 unlock 将由主机系统触发。UNLOCK_DMA 是主机如何触发这种解锁的方式。

可用的服务
---------------

  tcp:<port>

   打开一个到给定 localhost 端口的 TCP socket。它提供了一个不依赖于非常慢的内部模拟器 NAT 路由器的非常快速的通道。注意，你只能以 `read()` 和 `write()` 使用文件描述符，`send()` 和 `recv()` 将返回一个 `ENOTSOCK` 错误，任何 socket 的 `ioctl()` 也是。

   出于安全原因，无法连接非 localhost 端口。

  unix:<path>

   打开一个主机上的 Unix 域 socket。

  opengles

   连接 OpenGL ES 模拟进程。目前为止，其实现等价于 tcp:22468，但在未来这可能会改变。

  qemud

   连接模拟器内部的 QEMUD 服务。它替代了在老的 Android 平台发行版本上通过 `/dev/ttyS1` 执行的连接。详情请参考 `$QEMU/android/docs/ANDROID-QEMUD.TXT`。

### [打赏](https://www.wolfcstech.com/about/donate.html)

原文  ---- emu-2.4-release/external/qemu/android/docs/ANDROID-QEMU-PIPE.TXT

Done.
