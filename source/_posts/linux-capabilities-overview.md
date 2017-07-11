---
title: Linux 权能综述
date: 2017-07-10 13:17:49
categories: 安全
tags:
- 安全
---

为了执行权限检查，传统的 UNIX 实现区分两种类型的进程：特权进程（其有效用户 ID 为0，称为超级用户或 root），和非特权用户（其有效 UID 非0）。特权进程绕过所有的内核权限检查，而非特权进程受基于进程的认证信息（通常是：有效 UID，有效 GID，和补充组列表）的完整权限检查的支配。

自内核 2.2 版本开始，Linux 将传统上与超级用户关联的特权分为几个单元，称为 capabilities （权能），它们可以被独立的启用或禁用。权能是每个线程的属性。

# 权能列表

下面的列表展示了 Linux 上实现的权能，以及每种权能允许的操作或行为：
  * **CAP_AUDIT_CONTROL**（自 Linux 2.6.11）
    启用和禁用内核审计；修改审计过滤器规则；提取审计状态和过滤规则。
  * **CAP_AUDIT_READ**（自 Linux 3.16）
    允许通过一个多播 netlink socket 读取审计日志。
  * **CAP_AUDIT_WRITE**（自 Linux 2.6.11）
    向内核审计日志写记录。
  * **CAP_BLOCK_SUSPEND**（自 Linux 3.5）
    可以阻塞系统挂起（**epoll**(7) **EPOLLWAKEUP**，*/proc/sys/wake_lock*）的特性。
  * **CAP_CHOWN**
    对文件的 UIDs 和 GIDs 做任意的修改（参考 **chown**(2)）。
  * **CAP_DAC_OVERRIDE**
    绕过文件的读，写，和执行权限检查。（DAC 是 "discretionary access control" 的缩写。）
  * **CAP_DAC_READ_SEARCH**
      * 绕过文件的读权限检查和目录的读和执行权限检查；
      * 调用 **open_by_handle_at**(2)。
  * **CAP_FOWNER**
      * 对于通常要求进程的文件系统 UID 与文件的 UID 匹配的操作，绕过权限检查 (比如，**chmod**(2)，**utime**(2))，除了那些包含在 **CAP_DAC_OVERRIDE** 和 **CAP_DAC_READ_SEARCH** 中的操作；
      * 为任意文件设置扩展文件属性(参考 **chattr**(1))；
      * 为任意文件设置访问控制表(ACLs)；
      * 对文件删除操作忽略目录的 sticky 位；
      * 在 **open**(2) 和 **fcntl**(2) 任意文件时设置 **O_NOATIME**。
  * **CAP_FSETID**
    当文件修改时不清除 set-user-ID 和 set-group-ID 模式位；为文件 GID 与调用进程的文件系统或补充 GIDs 不匹配的文件设置 set-group-ID 位。
  * **CAP_IPC_LOCK**
    锁定内存 (**mlock**(2)，**mlockall**(2)，**mmap**(2)，**shmctl**(2))。
  * **CAP_IPC_OWNER**
    绕过对 System V IPC 对象的操作的权限检查。
  * **CAP_KILL**
    绕过发送信号 (参考 **kill**(2)) 时的权限检查。这包括使用 **ioctl**(2) **KDSIGACCEPT** 操作。
  * **CAP_LEASE**（自 Linux 2.4）
    为任意文件建立租约 (参考 **fcntl**(2))。
  * **CAP_LINUX_IMMUTABLE**
    设置**FS_APPEND_FL** 和 **FS_IMMUTABLE_FL** inode 标记 (参考 **chattr**(1))。
  * **CAP_MAC_ADMIN**（自 Linux 2.6.25）
    覆盖强制访问控制 (Mandatory Access Control (MAC)).  为 Smack Linux 安全模块(Linux Security Module (LSM)) 而实现。
  * **CAP_MAC_OVERRIDE**（自 Linux 2.6.25）
    允许 MAC 配置或状态改变。为 Smack LSM 而实现。
  * **CAP_MKNOD**（自 Linux 2.4）
    使用 **mknod**(2) 创建特殊文件。
  * **CAP_NET_ADMIN**
    执行多种网络有关的操作：
      * 接口配置；
      * IP 防火墙，地址伪装，和账单管理；
      * 修改路由表；
      * 为透明代理绑定任何地址；
      * 设置服务类性 (type-of-service (TOS))；
      * 清理驱动统计资料；
      * 设置混杂模式；
      * 启用组播；
      * 使用 **setsockopt**(2) 设置下列 socket 选项：**SO_DEBUG**，**SO_MARK**，**SO_PRIORITY** (在0到6范围之外的优先级)，**SO_RCVBUFFORCE**，和 **SO_SNDBUFFORCE**。
  * **CAP_NET_BIND_SERVICE**
    将一个 socket 绑定到一个互联网域特权端口 (端口号小于 1024)。
  * **CAP_NET_BROADCAST**
    (未使用)  使 socket 发送组播，并监听组播。
  * **CAP_NET_RAW**
      * 使用 RAW 和 PACKET sockets；
      * 为透明代理绑定任何地址。
  * **CAP_SETGID**
    执行任意的进程 GIDs 和补充 GID 列表管理；当通过 UNIX 域 sockets 传递 socket 认证信息时伪造 GID；在一个用户命名空间 (参考 **user_namespaces**(7)) 中写入组 ID 映射。
  * **CAP_SYS_ADMIN**
      * 执行一系列系统管理操作，包括：**quotactl**(2)，**mount**(2)，**umount**(2)，**swapon**(2)，**swapoff**(2)，**sethostname**(2)，和 **setdomainname**(2)；
      * 执行特权 syslog(2) 操作 (自 Linux 2.6.37 开始，应该使用 CAP_SYSLOG 来允许这一操作)；
      * 执行 **VM86_REQUEST_IRQ vm86**(2) 命令；
      * 对任意 System V IPC 对象执行 IPC_SET 和 IPC_RMID 操作；
      * 覆盖 RLIMIT_NPROC 资源限制；
      * 执行 trusted 和 security Extended Attributes (see **xattr**(7)) 操作；
      * 使用 **lookup_dcookie**(2)；
      * 使用  ioprio_set(2) 来分配 IOPRIO_CLASS_RT 和 (Linux 2.6.25 之前) IOPRIO_CLASS_IDLE I/O 调度类别；
      * 当通过 UNIX 域 sockets 传递 socket 认证信息时伪装 PID；
      * 在系统调用打开文件 (比如，**accept**(2)，**execve**(2)，**open**(2)，**pipe**(2)) 时，超出 /proc/sys/fs/file-max，系统范围内打开文件数的限制；
      * 通过 **clone**(2) 和 **unshare**(2) 使用 ** CLONE_* ** 标记创建新的命名空间（但是，自从 Linux 3.8 开始，创建命名空间不需要任何权能）；
      * 调用 **perf_event_open**(2)；
      * 访问特权 perf 事件信息；
      * 调用 **setns**(2) (在目标命名空间中需要 CAP_SYS_ADMIN)；
      * 调用 **fanotify_init**(2)；
      * 调用 **bpf**(2)；
      * 执行 **KEYCTL_CHOWN** 和 **KEYCTL_SETPERM keyctl**(2) 操作；
      * 执行 **madvise**(2) **MADV_HWPOISON** 操作；
      * 使用 **TIOCSTI ioctl**(2) 向一个终端的输入队列中插入字符，而不是调用者的控制终端；
      * 使用废弃的 **nfsservctl **(2) 系统调用；
      * 使用废弃的 **bdflush **(2) 系统调用；
      * 执行各种特权的块设备 **ioctl**(2) 操作；
      * 执行各种特权的文件系统 **ioctl**(2) 操作；
      * 对许多设备驱动执行管理操作。
  * **CAP_SYS_BOOT**
    使用 **reboot**(2) 和 **kexec_load**(2)。
  * **CAP_SYS_CHROOT**
    使用 **chroot**(2)。
  * **CAP_SYS_MODULE**
    加载和卸载内核模块(参考 **init_module**(2) 和 **delete_module**(2))；在 2.6.25 之前的内核中：从系统范围内的权能边界集合中丢弃权能。
  * **CAP_SYS_NICE**
      * 触发进程 nice 值 (**nice**(2)，**setpriority**(2)) 和为任意进程改变 nice 值；
      * 为调用进程设置实时调度策略，及为任意进程设置调度策略和优先级 (**sched_setscheduler**(2)，**sched_setparam**(2)，**shed_setattr**(2))；
      * 为任意进程设置 CPU affinity (**sched_setaffinity**(2))；
      * 为任意进程设置 I/O 调度类别和优先级 (**ioprio_set**(2))；
      * 对任意进程应用 **migrate_pages**(2) 并允许进程被迁移到任意节点；
      * 对任意进程应用 **move_pages**(2)；
      * 在 **mbind**(2) 和 **move_pages**(2) 中使用 **MPOL_MF_MOVE_ALL** 标记。
  * **CAP_SYS_PACCT**
    使用 **acct**(2)。
  * **CAP_SYS_PTRACE**
      * 使用 **ptrace**(2) 追踪任意进程；
      * 对任意进程应用 **get_robust_list**(2)；
      * 使用 **process_vm_readv**(2) 和 **process_vm_writev**(2) 同任意进程的内存传输数据；
      * 使用 **kcmp**(2) 检查进程。
  * **CAP_SYS_RAWIO**
      * 执行 I/O 端口操作 (**iopl**(2) 和 **ioperm**(2))；
      * 访问 /proc/kcore；
      * 使用 **FIBMAP ioctl**(2) 操作；
      * 打开设备访问 x86 模式特有寄存器 (MSRs，参考 **msr**(4))；
      * 更新 /proc/sys/vm/mmap_min_addr；
      * 在地址低于 /proc/sys/vm/mmap_min_addr 的位置创建内存映射；
      * 在 /proc/bus/pci 中映射文件；
      * 打开 /dev/mem 和 /dev/kmem；
      * 执行各种 SCSI 设备命令；
      * 在 **hpsa**(4) 和 **cciss**(4) 设备上执行某一操作；
      * 在其它设备上执行一系列设备特有操作。
  * **CAP_SYS_RESOURCE**
      * 使用 ext2 文件系统上的预留空间；
      * 执行 ioctl(2) 调用控制  ext3 日志；
      * 覆盖磁盘配额限制；
      * 增加资源限制 (参考 **setrlimit**(2))；
      * 覆盖 RLIMIT_NPROC 资源限制；
      * 在终端分配上覆盖最大的终端数；
      * 覆盖最大的 keymaps 个数；
      * 允许实时时钟中断大于64 hz；
      * 触发一个 System V 消息队列的 msg_qbytes 限制超过 /proc/sys/kernel/msgmnb 中的限制 (参考 **msgop**(2) 和 **msgctl**(2))；
      * 当使用 **F_SETPIPE_SZ fcntl**(2) 命令设置一个管道的容量时覆盖 /proc/sys/fs/pipe-size-max 的限制；
      * 使用 **F_SETPIPE_SZ** 增加管道的容量超出 /proc/sys/fs/pipe-max-size 指定的限制；
      * 当创建 POSIX 消息队列 (参考 **mq_overview**(7)) 时覆盖  /proc/sys/fs/mqueue/queues_max 的限制；
      * 使用 **prctl**(2) **PR_SET_MM** 操作；
      * 设置 /proc/PID/oom_score_adj 为一个小于由一个具有 CAP_SYS_RESOURCE 的进程最近设置的值的值。
  * **CAP_SYS_TIME**
    设置系统时钟 (**settimeofday**(2)，**stime**(2)，**adjtimex**(2))；设置实时 (硬件) 时钟。
  * **CAP_SYS_TTY_CONFIG**
    使用 **vhangup**(2)；对虚拟终端使用各种特权 **ioctl**(2) 操作。
  * **CAP_SYSLOG** (自 Linux 2.6.37)
      * 执行特权 **syslog**(2) 操作。参考 **syslog**(2) 来获取哪些操作需要特权的信息；
      * 当 /proc/sys/kernel/kptr_restrict 值为 1 时，查看通过 /proc 和其它接口暴露
的内核地址。(参考 **proc**(5) 中 kptr_restrict 的讨论。)
  * **CAP_WAKE_ALARM** (自 Linux 3.0)
    触发将唤醒系统的东西 (设置 CLOCK_REALTIME_ALARM 和 CLOCK_BOOTTIME_ALARM 定时器)。

# 过去和当前的实现

权能的完整实现需要：
1. 对于所有的特权操作，内核必须检查线程是否在其有效集合中具有要求的权能。
2. 内核必须提供系统调用以允许设置和提取一个线程的权能。
3. 文件系统必须支持为一个可执行文件附接权能，以使文件被执行时进程获得那些权能。

在内核 2.6.24 之前，只有前两个要求能够满足；自内核 2.6.24 开始，所有三个要求都能满足。

# 线程权能集合
每个线程具有三个包含零个或多个上面的权能的权能集合：
 * 被允许的 (Permitted)：
    这是线程可以承担的有效权能的限制性超集。这也是在线程的有效集合中不包含 **CAP_SETPCAP** 权能时，可以被线程添加进可继承的集合的权能的限制性超集。

    如果一个线程从它的被允许集合中丢弃了一个权能，则它将永远无法重新获取该权能 (除非它 **execve**(2)s 一个 set-user-ID-root 程序，或一个关联的文件权能获取了该权能授权的程序)。

 * 可继承的 (Inheritable)：
    这是跨越一个 execve(2) 保留的权能的集合。当执行任何程序时可继承的权能保持可继承，且当执行一个在文件的可继承集合中设置了对应位的程序时可继承权能被添加进被允许的集合。

    由于可继承的权能在以非 root 用户运行时通常不跨 **execve**(2) 保留，希望以抬高的权能运行辅助程序的应用应该考虑使用外界的权能，在下面描述。

 * 有效地 (Effective)：
    这是内核用来为线程执行权限检查的权能集合。

 * 外界的 (Ambient) (自 Linux 4.3)：
    这是一个为非特权程序的跨 execve(2) 保留的权能集合。外界的权能集合服从不可变性，如果权能既不是被允许的也不是可继承的，则它也从不可能是外界的。

    外界的权能集合可以直接使用 **prctl**(2) 修改。如果对应的被允许的或可继承的权能被降低，外界的权能将自动地降低。

    执行一个由于 set-user-ID 或 set-group-ID 位而修改 UID 或 GID 的程序，或执行一个具有任何文件权能集合的程序将清除外界的集合。在调用 execve(2) 时外界的权能被添加进被允许的集合，并被赋值给有效集合。

A child created via fork(2) inherits copies of its parent's capability sets.  See below for a discussion of the treatment of capabilities during execve(2).

Using capset(2), a thread may manipulate its own capability sets (see below).

Since Linux 3.2, the file /proc/sys/kernel/cap_last_cap exposes the numerical value of the highest capability supported by the running kernel; this can be used to determine the highest bit that may be set in a capability set.

# 文件权能
Since kernel 2.6.24, the kernel supports associating capability sets with an executable file using  setcap(8).   The  file  capability  sets  are stored in an extended attribute (see setxattr(2)) named security.capability.  Writing to this extended attribute requires  the  CAP_SETFCAP  capability.   The  file capability  sets, in conjunction with the capability sets of the thread, determine the capabilities of a thread after an execve(2).

The three file capability sets are:

* Permitted (formerly known as forced):
    These capabilities are automatically permitted to the thread, regardless of the thread's  inheritable capabilities.

* Inheritable (formerly known as allowed):
    This  set  is ANDed with the thread's inheritable set to determine which inheritable capabilities are enabled in the permitted set of the thread after the execve(2).

* Effective:
    This is not a set, but rather just a single bit.  If this bit is set, then  during  an  execve(2) all  of  the  new permitted capabilities for the thread are also raised in the effective set.  If this bit is not set, then after an execve(2), none of the new permitted capabilities  is  in  the new effective set.

    Enabling  the  file effective capability bit implies that any file permitted or inheritable capability that causes a thread to acquire the corresponding permitted capability during an execve(2) (see the transformation rules described below) will also acquire that capability in its effective set.   Therefore,  when  assigning  capabilities   to   a   file   (setcap(8),   cap_set_file(3), cap_set_fd(3)),  if  we  specify the effective flag as being enabled for any capability, then the effective flag must also be specified as enabled for all other capabilities for which the  corresponding permitted or inheritable flags is enabled.

# execve() 期间的权能转换
During an execve(2), the kernel calculates the new capabilities of the process using the following algorithm:

 * P'(ambient) = (file is privileged) ? 0 : P(ambient)

 * P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset) | P'(ambient)

 * P'(effective) = F(effective) ? P'(permitted) : P'(ambient)

 * P'(inheritable) = P(inheritable)    [i.e., unchanged]

其中：

 * P         denotes the value of a thread capability set before the execve(2)

 * P'        denotes the value of a capability set after the execve(2)

 * F         denotes a file capability set

 * cap_bset  is the value of the capability bounding set (described below).

A privileged file is one that has capabilities or has the set-user-ID or set-group-ID bit set.

# root 的程序的权能和执行
In order to provide an all-powerful root using capability sets, during an execve(2):

1. If a set-user-ID-root program is being executed, or the real user ID of the process is 0 (root)  then the file inheritable and permitted sets are defined to be all ones (i.e., all capabilities enabled).

2. If  a  set-user-ID-root  program  is being executed, then the file effective bit is defined to be one (enabled).

The upshot of the above rules, combined with the capabilities transformations described above,  is  that when  a  process  execve(2)s  a  set-user-ID-root  program, or when a process with an effective UID of 0 execve(2)s a program, it gains all capabilities in its permitted and effective capability  sets,  except those  masked  out  by  the capability bounding set.  This provides semantics that are the same as those provided by traditional UNIX systems.

# 权能边界集合
The capability bounding set is a security mechanism that can be used to limit the capabilities that  can be gained during an execve(2).  The bounding set is used in the following ways:

 * During  an execve(2), the capability bounding set is ANDed with the file permitted capability set, and the result of this operation is assigned to the thread's permitted  capability  set.   The  capability bounding  set  thus  places a limit on the permitted capabilities that may be granted by an executable file.

 * (Since Linux 2.6.25) The capability bounding set acts as a limiting superset for the capabilities that a  thread  can  add to its inheritable set using capset(2).  This means that if a capability is not in the bounding set, then a thread can't add this capability to its inheritable set, even if  it  was  in its  permitted  capabilities,  and  thereby cannot have this capability preserved in its permitted set when it execve(2)s a file that has the capability in its inheritable set.

Note that the bounding set masks the file permitted capabilities, but not  the  inherited  capabilities. If  a  thread  maintains  a capability in its inherited set that is not in its bounding set, then it can still gain that capability in its permitted set by executing a file  that  has  the  capability  in  its inherited set.

Depending  on  the  kernel  version, the capability bounding set is either a system-wide attribute, or a per-process attribute.

## Linux 2.6.25 之前的权能边界集合

In kernels before 2.6.25, the capability bounding set  is  a  system-wide  attribute  that  affects  all threads  on  the system.  The bounding set is accessible via the file /proc/sys/kernel/cap-bound.  (Confusingly, this bit mask parameter is expressed as  a  signed  decimal  number  in  /proc/sys/kernel/capbound.)

Only  the  init  process may set capabilities in the capability bounding set; other than that, the superuser (more precisely: programs with the CAP_SYS_MODULE capability) may  only  clear  capabilities  from this set.

On a standard system the capability bounding set always masks out the CAP_SETPCAP capability.  To remove this restriction (dangerous!), modify the definition of CAP_INIT_EFF_SET  in  include/linux/capability.h and rebuild the kernel.

The system-wide capability bounding set feature was added to Linux starting with kernel version 2.2.11.

## Linux 2.6.25 之后的权能边界集合

From Linux 2.6.25, the capability bounding set is a per-thread attribute.  (There is no longer a systemwide capability bounding set.)

The bounding set is inherited at fork(2) from the thread's parent, and is preserved across an execve(2).

A thread may remove capabilities from its capability bounding set  using  the  prctl(2)  PR_CAPBSET_DROP operation,  provided  it  has  the  CAP_SETPCAP capability.  Once a capability has been dropped from the bounding set, it cannot be restored to that set.  A thread can determine  if  a  capability  is  in  its bounding set using the prctl(2) PR_CAPBSET_READ operation.

Removing capabilities from the bounding set is supported only if file capabilities are compiled into the kernel.  In kernels before Linux 2.6.33, file capabilities were an optional feature configurable via the CONFIG_SECURITY_FILE_CAPABILITIES option.  Since Linux 2.6.33, the configuration option has been removed and file capabilities are always part of the kernel.  When file capabilities are compiled into the  kernel, the init process (the ancestor of all processes) begins with a full bounding set.  If file capabilities are not compiled into the kernel, then init begins with a full  bounding  set  minus  CAP_SETPCAP, because this capability has a different meaning when there are no file capabilities.

Removing a capability from the bounding set does not remove it from the thread's inherited set.  However it does prevent the capability from being added back into the thread's inherited set in the future.

# Effect of user ID changes on capabilities
To preserve the traditional semantics for transitions between 0 and nonzero user IDs, the  kernel  makes the  following  changes  to a thread's capability sets on changes to the thread's real, effective, saved set, and filesystem user IDs (using setuid(2), setresuid(2), or similar):

1. If one or more of the real, effective or saved set user IDs was previously 0, and as a result of  the UID changes all of these IDs have a nonzero value, then all capabilities are cleared from the permitted and effective capability sets.

2. If the effective user ID is changed from 0 to nonzero, then all capabilities  are  cleared  from  the effective set.

3. If the effective user ID is changed from nonzero to 0, then the permitted set is copied to the effective set.

4. If the filesystem user ID is changed from 0 to nonzero (see setfsuid(2)), then the following capabilities   are  cleared  from  the  effective  set:  CAP_CHOWN,  CAP_DAC_OVERRIDE,  CAP_DAC_READ_SEARCH, CAP_FOWNER, CAP_FSETID, CAP_LINUX_IMMUTABLE (since Linux  2.6.30),  CAP_MAC_OVERRIDE,  and  CAP_MKNOD (since Linux 2.6.30).  If the filesystem UID is changed from nonzero to 0, then any of these capabilities that are enabled in the permitted set are enabled in the effective set.

If a thread that has a 0 value for one or more of its user IDs wants to prevent its permitted capability set  being cleared when it resets all of its user IDs to nonzero values, it can do so using the prctl(2) PR_SET_KEEPCAPS operation or the SECBIT_KEEP_CAPS securebits flag described below.

# 以编程方式调整权能集合
A thread can retrieve and change its capability sets using the capget(2)  and  capset(2)  system  calls. However,  the  use  of cap_get_proc(3) and cap_set_proc(3), both provided in the libcap package, is preferred for this purpose.  The following rules govern changes to the thread capability sets:

1. If the caller does not have the CAP_SETPCAP capability, the new inheritable set must be a  subset  of the combination of the existing inheritable and permitted sets.

2. (Since  Linux  2.6.25)  The  new  inheritable set must be a subset of the combination of the existing inheritable set and the capability bounding set.

3. The new permitted set must be a subset of the existing permitted set (i.e., it  is  not  possible  to acquire permitted capabilities that the thread does not currently have).

4. The new effective set must be a subset of the new permitted set.

# securebits 标志：建立一个仅限权能的环境
Starting  with kernel 2.6.26, and with a kernel in which file capabilities are enabled, Linux implements a set of per-thread securebits flags that can be used to disable special handling  of  capabilities  for UID 0 (root).  These flags are as follows:

 * SECBIT_KEEP_CAPS
    Setting  this flag allows a thread that has one or more 0 UIDs to retain its capabilities when it switches all of its UIDs to a nonzero value.  If this flag is not set, then  such  a  UID  switch causes  the thread to lose all capabilities.  This flag is always cleared on an execve(2).  (This flag provides the same functionality as the older prctl(2) PR_SET_KEEPCAPS operation.)

 * SECBIT_NO_SETUID_FIXUP
    Setting this flag stops the kernel from adjusting capability sets when  the  threads's  effective and  filesystem UIDs are switched between zero and nonzero values.  (See the subsection Effect of User ID Changes on Capabilities.)

 * SECBIT_NOROOT
    If this bit is set, then the kernel does not grant capabilities when a  set-user-ID-root  program is executed, or when a process with an effective or real UID of 0 calls execve(2).  (See the subsection Capabilities and execution of programs by root.)

 * SECBIT_NO_CAP_AMBIENT_RAISE
    Setting this flag disallows raising ambient capabilities via  the  prctl(2)  PR_CAP_AMBIENT_RAISE operation.

Each  of  the  above  "base"  flags has a companion "locked" flag.  Setting any of the "locked" flags is irreversible, and has the effect of preventing further changes to the corresponding  "base"  flag.   The locked  flags  are:  SECBIT_KEEP_CAPS_LOCKED,  SECBIT_NO_SETUID_FIXUP_LOCKED,  SECBIT_NOROOT_LOCKED, and SECBIT_NO_CAP_AMBIENT_RAISE.

The  securebits  flags  can  be  modified  and  retrieved  using  the  prctl(2)  PR_SET_SECUREBITS   and PR_GET_SECUREBITS operations.  The CAP_SETPCAP capability is required to modify the flags.

The  securebits  flags are inherited by child processes.  During an execve(2), all of the flags are preserved, except SECBIT_KEEP_CAPS which is always cleared.

An application can use the following call to lock itself, and all of its descendants, into  an  environment where the only way of gaining capabilities is by executing a program with associated file capabilities:
```
           prctl(PR_SET_SECUREBITS,
                   SECBIT_KEEP_CAPS_LOCKED |
                   SECBIT_NO_SETUID_FIXUP |
                   SECBIT_NO_SETUID_FIXUP_LOCKED |
                   SECBIT_NOROOT |
                   SECBIT_NOROOT_LOCKED);
```

# 与用户命名空间交互
For a discussion of the interaction of capabilities and user namespaces, see user_namespaces(7).

# 标准参考
No standards govern capabilities, but the Linux capability implementation  is  based  on  the  withdrawn POSIX.1e draft standard; see ⟨http://wt.tuxomania.net/publications/posix.1e/⟩.

# 备注
From  kernel  2.5.27  to  kernel  2.6.26,  capabilities  were  an  optional kernel component, and can be enabled/disabled via the CONFIG_SECURITY_CAPABILITIES kernel configuration option.

The /proc/PID/task/TID/status file  can  be  used  to  view  the  capability  sets  of  a  thread.   The /proc/PID/status  file shows the capability sets of a process's main thread.  Before Linux 3.8, nonexistent capabilities were shown as being enabled (1) in these sets.  Since Linux 3.8, all nonexistent capabilities (above CAP_LAST_CAP) are shown as disabled (0).

The  libcap  package provides a suite of routines for setting and getting capabilities that is more comfortable and less likely to change than the interface provided by capset(2) and capget(2).  This package also provides the setcap(8) and getcap(8) programs.  It can be found at ⟨http://www.kernel.org/pub/linux/libs/security/linux-privs⟩.

Before  kernel  2.6.24,  and from kernel 2.6.24 to kernel 2.6.32 if file capabilities are not enabled, a thread with the CAP_SETPCAP capability can manipulate the capabilities of  threads  other  than  itself. However,  this  is  only theoretically possible, since no thread ever has CAP_SETPCAP in either of these cases:

 * In the pre-2.6.25 implementation the system-wide capability bounding set,  /proc/sys/kernel/cap-bound, always  masks out this capability, and this can not be changed without modifying the kernel source and rebuilding.

 * If file capabilities are disabled in the current implementation, then init starts out with this  capability removed from its per-process bounding set, and that bounding set is inherited by all other processes created on the system.

# 另请参阅
capsh(1),  setpriv(1),   prctl(2),   setfsuid(2),   cap_clear(3),   cap_copy_ext(3),   cap_from_text(3), cap_get_file(3),   cap_get_proc(3),  cap_init(3),  capgetp(3),  capsetp(3),  libcap(3),  credentials(7), user_namespaces(7), pthreads(7), getcap(8), setcap(8)

include/linux/capability.h in the Linux kernel source tree

# 版本记录
This page is part of release 4.04 of the Linux man-pages project.  A description of the project,  information   about   reporting   bugs,   and   the   latest   version   of   this  page,  can  be  found  at http://www.kernel.org/doc/man-pages/.
