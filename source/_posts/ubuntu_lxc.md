---
title: Ubuntu LXC
date: 2017-12-04 11:05:49
categories: 虚拟化
tags:
- 虚拟化
- 翻译
---

容器是轻量级的虚拟化技术。它们更像增强的 chroot，而不是完整的虚拟化，比如 Qemu 或 VMware，因为它们不仿真硬件，且由于容器与主机共享相同的操作系统。容器与 Solaris zones 或 BSD  jails 类似。Linux-vserver 和 OpenVZ 是两种已经存在的，为 Linux 独立开发的类容器功能实现。事实上，容器是由 vserver 和 OpenVZ 功能升级的工作而产生的。
<!--more-->
有两种容器的用户空间实现，每个都利用了相同的内核功能。Libvirt 允许通过 'lxc:///' 借助于 LXC 驱动来使用容器。这可能是非常方便的，因为它支持像使用其它驱动那样使用。另一种实现，简单地称为 'LXC'，与 libvirt 不兼容，但是提供了更多的用户空间工具，因而更加灵活。在这两者之间来回切换也是可以的，尽管有些特点可能导致一些困扰。

在这份文档中我们将主要描述 *lxc* 包。由于缺少为 libvirt-lxc 容器的 Apparmor 保护，因而通常不建议使用 libvirt-lxc 。

在这份文档中，容器名字将显示为 CN，C1或C2。

# 安装

*lxc* 包可以使用如下命令进行安装：
```
$ sudo apt install lxc
```

这将拉取所需及建议的依赖，并建立一个网桥来给容器使用。如果你想使用非特权容器，你将需要确保用户具有足够的分配 subuids 和 subgids 的权限，并可能想要允许用户把容器连接到网桥（参考 [基本的非特权使用](https://help.ubuntu.com/lts/serverguide/lxc.html#lxc-unpriv "Basic unprivileged usage")）。

# 基本用法
LXC 可以以两种不同的方式使用 - 特权的，以 root 用户运行 lxc 命令；或非特权的，以非 root 用户。（root 用户启动非特权的容器也是可以的，但这里不描述这种情况。）非特权的容器有更多限制，比如不能创建设备节点或挂载块支持的文件系统。然而因为容器内的 root userid 被映射为主机系统上的非 root userid，它们对于主机系统的危险性更小。

## 基本的特权使用
要创建一个特权容器，你可以简单地这样做：
```
$ sudo lxc-create --template download --name u1
# or, abbreviated
$ sudo lxc-create -t download -n u1
```

这将交互式地询问要下载的容器根文件系统的类型 - 特别是发行版，发布版，和硬件架构。要非交互式地创建容器的话，可以在命令行中指定这些值：
```
$ sudo lxc-create -t download -n u1 -- --dist ubuntu --release xenial --arch amd64
# or
$ sudo lxc-create -t download -n u1 -- -d ubuntu -r xenial -a amd64
```

现在你可以使用 `lxc-ls` 来列出容器，使用 `lxc-info` 来获得详细的容器信息，使用 `lxc-start` 来启动及使用 `lxc-stop` 来停止容器了。`lxc-attach` 和 `lxc-console` 允许你进入一个容器，如果不能选择 ssh 的话。`lxc-destroy` 销毁容器，包括它的根文件系统。请参考手册页来获得关于每个命令更详细的信息。一个示例会话可能看起来像这样：
```
$ sudo lxc-ls --fancy
$ sudo lxc-start --name u1 --daemon
$ sudo lxc-info --name u1
$ sudo lxc-stop --name u1
$ sudo lxc-destroy --name u1
```

## 用户命名空间
非特权容器允许用户在没有任何 root 权限的情况下创建和管理容器。支撑这一点的功能称为用户名空间。用户命名空间是层次结构的，在父命名空间中的特权任务能够把它的 ids 映射到子命名空间。默认情况下，主机系统中的每个任务运行于初始用户命名空间，其中全部范围的 ID 映射到全范围。这可以通过 `/proc/self/uid_map` 和 `/proc/self/gid_map` 看出来，当在初始用户命名空间中读取时，它们都显示为 "0 0 4294967295"。对于 Ubuntu 14.04 来说，当创建新的用户时，它们默认被提供了一个 userids 范围。分配的 id 的列表可以在 `/etc/subuid` 和 `/etc/subgid` 文件中看到。请参考它们各自的 manpages 来了解更多信息。Subuids 和 subgids 通常从 id 100000 开始，以避免与系统用户发生冲突。

如果在更早的发布版上创建了用户，可以使用 `usermod` 为它授权一个 ID 范围，像下面这样：
```
$ sudo usermod -v 100000-200000 -w 100000-200000 user1
```

uidmap 包中的程序 `newuidmap` 和 `newgidmap` 是  setuid-root 程序，在 lxc 内部它们被用于把 subuids 和 subgids 从主机系统映射到非特权容器中。它们确保用户只映射主机系统配置中授权的 ID。

## 基本的非特权使用
要创建非特权容器，需要执行几个初始化的步骤。你需要创建一个默认的容器配置文件，指定你想要的 id 映射和网络设置，并配置主机系统以允许非特权用户 hook 主机系统的网络。下面的例子假设你的映射用户和组 ID 范围为100000-165536。检查你的实际用户和组 ID 范围并据此修改示例：
```
grep $USER /etc/subuid
grep $USER /etc/subgid
```

```
mkdir -p ~/.config/lxc
echo "lxc.id_map = u 0 100000 65536" > ~/.config/lxc/default.conf
echo "lxc.id_map = g 0 100000 65536" >> ~/.config/lxc/default.conf
echo "lxc.network.type = veth" >> ~/.config/lxc/default.conf
echo "lxc.network.link = lxcbr0" >> ~/.config/lxc/default.conf
echo "$USER veth lxcbr0 2" | sudo tee -a /etc/lxc/lxc-usernet
```

之后，你可以创建非特权容器，就像创建特权容器那样，只是无需 sudo：
```
lxc-create -t download -n u1 -- -d ubuntu -r xenial -a amd64
lxc-start -n u1 -d
lxc-attach -n u1
lxc-stop -n u1
lxc-destroy -n u1
```

## 嵌套
为了在容器内运行容器 - 被称为嵌套容器 - 必须向父容器的配置文件中添加如下两行：
```
lxc.mount.auto = cgroup
lxc.aa_profile = lxc-container-default-with-nesting
```

第一行将使得 cgroup 管理器 socket 被绑定容器中，以使容器内的 lxc 能够为它的嵌套容器管理 cgroups。第二行使容器以更松散的 Apparmor 规则运行，这使容器在启动容器时可以执行所需的挂载操作。注意，这个规则，当以特权容器使用时，相对于普通的规则或非特权容器要不安全得多。参考 [Apparmor](https://help.ubuntu.com/lts/serverguide/lxc.html#lxc-apparmor "Apparmor") 了解更多信息。

# 全局配置
LXC 会使用下面的配置文件。对于特权使用，它们在 `/etc/lxc` 下查找，而对于非特权使用则在 `~/.config/lxc` 下。
 * `lxc.conf` 可以为一些 lxc 设置指定可选的值，包括 lxcpath，默认的配置，要使用的 cgroups，cgroups 创建模式，以及 lvm 和 zfs 的存储后端设置。
 * `default.conf` 指定每个新创键的容器应该包含的配置。这通常至少包含一个网络部分，以及，对于非特权用户，id 映射部分。
 * `lxc-usernet.conf` 指定非特权用户可以如何把他们的容器连接到主机所有的网路。

`lxc.conf` 和 `default.conf` 都在 `/etc/lxc` 和 `$HOME/.config/lxc`下，而 `lxc-usernet.conf` 只在 host-wide。

默认情况下，对于 root 用户而言，容器位于 `/var/lib/lxc`，否则位于 `$HOME/.local/share/lxc`。可以为所有的 lxc 命令使用 "-P|--lxcpath" 参数来指定位置。

# 网络
默认情况下，LXC 为每个容器创建一个私有的网络命名空间，其中包含一个 2 层网络栈。容器通常通过一个物理 NIC 或一个传入容器的 veth 隧道端点连接到外部世界。LXC 创建一个 NAT 桥，lxcbr0，在主机启动的时候。使用默认配置创建的容器将有一个 veth NIC，其远程端点插入 lxcbr0 桥。NIC 一次只能出现在一个命名空间中，因此传入容器的物理 NIC 在主机上是不能用的。

也可以创建一个没有私有网络命名空间的容器。在这种情况下，容器将具有访问主机网络的权限，就像任何其它应用那样。注意，如果容器运行一个具有 upstart 的发行版的话，比如 Ubuntu，这特别危险，因为与 init 交互的程序，比如 `shutdown`，将通过抽象 Unix 域 socket 与主机的 upstart 交互，并关闭主机。

要给 lxcbr0 上的容器设置一个基于域名的固定 IP 地址的话，你可以向 `/etc/lxc/dnsmasq.conf` 写入像下面这样的内容：
```
dhcp-host=lxcmail,10.0.3.100
dhcp-host=ttrss,10.0.3.101
```

如果想要容器可以公开访问，有一些方式可以做到这一点。其中一个是使用 `iptables` 把主机端口转发到容器，比如：
```
 iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 587 -j DNAT \
 	--to-destination 10.0.3.100:587
```

另一种方法是桥接主机的网络接口（参考 Ubuntu 服务器指南的网络配置章节，[桥接](https://help.ubuntu.com/lts/serverguide/network-configuration.html#bridging)）。然后，在容器配置文件中指定主机的网桥代替 lxcbr0，比如：
```
lxc.network.type = veth
lxc.network.link = br0
```

最后，你可以请求 LXC 为容器的 NIC 使用 macvlan。注意这有一些限制，并依赖于配置，可能不允许容器与主机本身通信。因此另外的两个选项更好，且更常用。

有一些方式来决定容器的 IP 地址。首先，你可以使用 `lxc-ls --fancy`，它将打印出所有运行中的容器的 IP 地址，或者 `lxc-info -i -H -n C1`，其将打印 C1 的 IP 地址。如果已经在主机上安装了 dnsmasq，你也可以向 `/etc/dnsmasq.conf` 添加如下的内容：
```
server=/lxc/10.0.3.1
```

之后，dnsmasq 将本地解析 C1.lxc，因此你可以：
```
ping C1
ssh C1
```

更多信息，请参考 lxc.conf 以及位于 `/usr/share/doc/lxc/examples/`的示例网络配置。

# LXC 启动
LXC 没有长期运行的守护进程。然而，它确实有三个 upstart 任务：

 * `/etc/init/lxc-net.conf`：是一个可选的任务，只有当 `/etc/default/lxc-net` 指定 USE_LXC_BRIDGE 为（true 或 default）时运行。它建立一个 NAT 网桥来给容器使用。
 * `/etc/init/lxc.conf` 加载 lxc apparmor 配置，并可选地启动任何 autostart 容器。如果 `/etc/default/lxc` 中的 LXC_AUTO（默认为 true）被设置为 true，autostart 容器将忽略
 * `/etc/init/lxc.conf` 使用 `/etc/init/lxc-instance.conf` autostart 一个容器。

# 后备存储
LXC 支持一些容器根文件系统的后备存储。默认的是一个简单的目录后备存储，因为它不需要先前的主机定制，只要底层文件系统足够大。它也无需 root 权限来创建后备存储，因此它可以无缝地用于非特权场景。特权场景的
 rootfs 目录的后备容器（默认情况下）位于 `/var/lib/lxc/C1/rootfs`，而非特权容器的 rootfs 位于 `~/.local/share/lxc/C1/rootfs`。如果在 `lxc.system.com` 中指定了定制的 lxcpath，则容器的 rootfs 将位于 `$lxcpath/C1/rootfs`。

一个由目录支持的容器 C1 的快照克隆 C2，变成了一个 overlayfs 后备的容器，其 rootfs 称为 `overlayfs:/var/lib/lxc/C1/rootfs:/var/lib/lxc/C2/delta0`。其它的后备存储类型包括 loop，btrfs，LVM 和 zfs。

btrfs 后备的容器大多看起来像目录后备的容器，其根文件系统在相同的位置。然而跟文件系统包含一个子卷，因此快照克隆使用一个子卷快照创建。

LVM 后备的容器的根文件系统可以是任何分开的 LV。默认的 VG 名称可以由 `lxc.conf` 指定。文件系统类型和大小是每个容器使用 lxc-create 可配置的。

zfs 后备的容器的根文件系统是一个单独的 zfs 文件系统，挂载在传统的 `/var/lib/lxc/C1/rootfs` 位置下。zfsroot 可以在 lxc-create 处指定，默认的可以在 `lxc.system.conf` 中指定。

更多关于创建具有各种后备存储的容器的信息可以在 lxc-create 的手册页中找到。

# 模板
创建一个容器通常包含为容器创建一个根文件系统。`lxc-create` 将这一工作委托给 *templates*，它通常是发行版相关的。与 lxc 一起发布的 lxc 模板可以在 /usr/share/lxc/templates 下找到，包括创建 Ubuntu，Debian，Fedora，Oracle，centos，和 gentoo 容器及其它的模板。

创建发行版镜像在大多数情况下需要创建设备节点的能力，常常需要在其它发行版中不可用的工具，且通常是相当耗时的。然而 lxc 自带一个特殊的 *download* 模板，它从一个中央 lxc 服务器下载预编译的容器镜像。最重要的使用场景是允许非 root 用户简单的创建非特权容器，这些用户可能，比如无法简单的运行 `debootstrap` 命令。

当运行 `lxc-create` 时，-- 之后的所有选项都是传给模板的。在下面的命令中，--name，--template 和 --bdev 是传给 `lxc-create` 的，而 --release 是传给模板的：
```
$ lxc-create --template ubuntu --name c1 --bdev loop -- --release xenial
```

你可以通过把 `--help` 和模板名字传给 `lxc-create` 来获得任何特定容器支持的选项的帮助信息。比如，要获得 download 模板的帮助信息，
```
$ lxc-create --template download --help
```

# Autostart
LXC 支持把容器标记为在系统启动时启动。在 Ubuntu 14.04 之前，这通过使用位于 `/etc/lxc/auto` 目录下的符号链接完成。自 Ubuntu 14.04 开始，它通过容器配置文件完成。如下这样的配置
```
lxc.start.auto = 1
lxc.start.delay = 5
```

将意味着容器应该在启动时起动，且系统应该在起动下一个容器时等待 5 秒。LXC 也支持容器的排序和分组，并通过 autostart 分组 重启和关闭。请参考 `lxc-autostart` 和 `lxc.container.conf` 的手册页来获得更多信息。

# Apparmor
LXC 带有一个默认的 Apparmor 配置，用于保护主机免遭容器内特权的偶然误用的破坏。比如，容器将无法写入 `/proc/sysrq-trigger` 或大多数 `/sys` 文件。

`usr.bin.lxc-start` 配置通过运行 `lxc-start` 进入。这个配置主要用于防止 `lxc-start` 在容器的根文件系统之外挂载新的文件系统。在执行容器的 `init` 之前，LXC 请求切换容器的配置。默认情况下，这个配置是 `lxc-container-default` 规则，它定义在 `/etc/apparmor.d/lxc/lxc-default` 中。这个配置防止容器访问许多危险的路径，以及挂载大多数文件系统。

容器内的程序不能被进一步的限制 - 比如，MySQL 运行在容器的配置下（保护主机）但将不能进入 MySQL 配置（为了保护容器）。

`lxc-execute` 不进入 Apparmor 配置，但它派生的容器将收到限制。

## 定制容器的规则
如果你发现 `lxc-start` 由于合法的访问被它的 Apparmor 规则禁止而失败，你可以通过如下命令禁用 `lxc-start` 配置：
```
$ sudo apparmor_parser -R /etc/apparmor.d/usr.bin.lxc-start
$ sudo ln -s /etc/apparmor.d/usr.bin.lxc-start /etc/apparmor.d/disabled/
```

这将使 `lxc-start` 的运行毫无限制，但继续限制容器本身。如果你也想禁用容器的限制，然后还要禁用 `usr.bin.lxc-start` 配置，你必须给容器的配置文件添加如下内容：
```
lxc.aa_profile = unconfined
```

LXC 为容器自带了一些规则。如果你想要在容器内运行容器（嵌套），然后你可以使用 `lxc-container-default-with-nesting` 配置，通过向容器的配置文件添加如下的行
```
lxc.aa_profile = lxc-container-default-with-nesting
```

如果你想在容器内使用 libvirt，则你将需要通过取消注释如下的行，来编辑那个规则（定义在 `/etc/apparmor.d/lxc/lxc-default-with-nesting` 中）
```
mount fstype=cgroup -> /sys/fs/cgroup/**,
```

然后重新加载规则。

注意，特权容器中的嵌套规则远不如默认规则安全，因为它允许容器在非标准的位置重新挂载 `/sys` 和 `/proc`，绕过 apparmor 保护。非特权容器没有这个缺陷，因为容器的 root 无法写入 root 所拥有的 `proc` 和 `sys` 文件。

另一个与 lxc 一起发布的配置允许容器挂载像 ext4 这种类型的块文件系统。在像 maas 配置这样的情况下这可能很有用，但这通常被认为是不安全的，因为内核中的超级块处理器还没有被审计，以安全地处理不可信输入。

如果你需要以一个定制的配置运行容器，你可以在 `/etc/apparmor.d/lxc/` 下创建一个新的配置。其名称必须以 `lxc-` 开头，以便允许 `lxc-start` 转换到该配置文件。`lxc-default` 配置包含可复用的抽象文件 `/etc/apparmor.d/abstractions/lxc/container-base`。创建一个新的配置的简单方式就是基于相同的文件，然后在你的规则的底部添加额外的权限。

创建了规则之后，使用如下命令加载它：
```
$ sudo apparmor_parser -r /etc/apparmor.d/lxc-containers
```

配置将在重启后自动地加载，因为它由 `/etc/apparmor.d/lxc-containers` sourced。最后，要使容器 CN 使用这个新的 `lxc-CN-profile`，则在它的配置文件中添加如下的行：
```
lxc.aa_profile = lxc-CN-profile
```

# Control Groups
Control groups (cgroups) 是一个内核功能，它提供了层次式的任务分组，以及每个分组的资源管理和限制。在容器中它们被用于限制块和字符设备访问，及冻结（挂起）容器。它们可以被进一步用于限制内存使用及块 I/O，保证最小的 CPU 占用并把容器锁定到特定的 CPUs 上。

默认情况下，特权容器 CN 将被分配给称为 `/lxc/CN` 的 `cgroup`。在名字冲突的情况下（当使用定制的 lxcpath 的情况下可能发生）后缀 "-n"，其中 n 是一个从 0 开始的整数，将被添加到 cgroup 名称后面。

默认情况下，特权容器 CN 将被分配给启动容器的任务的 cgroup 之下称为 `CN` 的 `cgroup`，比如 `/usr/1000.user/1.session/CN`。容器的 root 用户将被赋予分组的目录的所有权（但不是所有文件），因此允许它创建新的子分组。

从 Ubuntu 14.04 开始，LXC 使用 cgroup 管理器（cgmanager）来管理 cgroup。cgroup 管理器通过 Unix socket `/sys/fs/cgroup/cgmanager/sock` 接收 D-Bus 请求。为了方便使用安全的嵌套容器，下面的行
```
lxc.mount.auto = cgroup
```

可以被添加到容器的配置中，以使 `/sys/fs/cgroup/cgmanager` 目录被绑定挂载到容器中。容器则应该启动 cgroup 管理代理（如果 cgmanager 包已经在容器内安装了则默认完成），其将把 `/sys/fs/cgroup/cgmanager` 目录移动到 `/sys/fs/cgroup/cgmanager.lower`，然后在它自己的 socket `/sys/fs/cgroup/cgmanager/sock` 上启动监听到代理的请求。主机的 cgmanager 将确保嵌套的容器无法逃脱它们被分配的 cgroups ，或者创建它们未被授权的请求。

# 克隆
为了快速配置，你可能希望根据你的需求定制规范容器，然后制作多个副本。这可以通过 `lxc-clone` 程序完成。

克隆是快照或另一个容器的拷贝。拷贝是从原始容器拷贝而得到的一个新的容器，其消耗的主机磁盘上的空间像原始的那个一样多。快照利用了底层后备存储的快照能力，来创建一个写时复制容器引用第一个。快照可以从 btrfs，LVM，zfs，和目录后备容器创建。每个后备存储具有它自己的特点 - 比如，未配置精简池的 LVM 容器不支持快照的快照；具有快照的 zfs 容器无法被移除，直到所有的快照都被释放；LVM 容器必须更小心的计划，因为底层的文件系统可能不支持增长；btrfs 没有任何这些缺点，但它的 fsync 性能降低了，从而导致 dpkg 和 apt 更慢。

目录打包容器的快照使用 overlay 文件系统创建。比如，一个特权目录打包的容器 C1 将具有它自己的根文件系统，位于 `/var/lib/lxc/C1/rootfs`。C1 的克隆快照称为 C2，将以位于 `/var/lib/lxc/C2/delta0` 的只读挂载的 C1 根文件系统启动。重要地是，在这种情况下，当 C2 运行时，不应该允许 C1 运行或移除。建议将 C1 视为一个规范的基本容器，并仅使用其快照。

给定一个已有的容器，称为 C1，可以使用如下命令创建一份拷贝：
```
$ sudo lxc-clone -o C1 -n C2
```

可以使用如下命令创建快照：
```
$ sudo lxc-clone -s -o C1 -n C2
```

参考 `lxc-clone` 手册页了解更多信息。

## 快照
为了更容易地支持使用快照克隆进行迭代的容器开发，LXC 支持 *snapshots*。在使用容器 C1 时，在进行具有潜在危险性或难以还原的更改之前，可以创建快照
```
sudo lxc-snapshot -n C1
```

其是一个位于 `/var/lib/lxcsnaps` 或 `$HOME/.local/share/lxcsnaps` 称为 'snap0' 的快照-克隆。下一个快照称为 'snap1',，以此类推。可以使用 `lxc-snapshot -L -n C1` 列出已有的快照，且快照可以被恢复 - 擦除当前的 C1 容器 - 使用 `lxc-snapshot -r snap1 -n C1`。恢复命令之后，snap1 快照依然存在，而之前的 C1 被擦除并被 snap1 快照替代。

btrfs，lvm，zfs，和 overlayfs 容器支持快照。如果在一个目录后备的容器上调用 `lxc-snapshot`，将打印一条错误日志，而快照将被创建为一个拷贝-克隆。这么做的理由是，如果用户创建了一个目录后备的容器的 overlayfs 快照，然后对目录后备的容器做了一些修改，然后原始容器的改变将部分地反映在快照中。如果需要目录后备的容器 C1 的快照，则应该创建 C1 的 overlayfs 克隆，而不应该再动 C1，且可以随意编辑和快照 overlayfs 克隆，如
```
lxc-clone -s -o C1 -n C2
lxc-start -n C2 -d # make some changes
lxc-stop -n C2
lxc-snapshot -n C2
lxc-start -n C2 # etc
```

## 短暂容器
快照对于镜像长期的增量开发，短暂容器利用快照来获得快速，一次性使用可丢掉的容器。 给定一个基本容器 C1，可以使用如下命令启动一个临时容器
```
$ lxc-start-ephemeral -o C1
```

容器以 C1 的快照开始。登录到容器的说明将被打印到控制台。关闭后，短暂容器将被销毁。 请参阅 `lxc-start-ephemeral` 手册页以获取更多选项。

# 生命周期管理 hooks

自 Ubuntu 12.10 开始，可以定义 hooks 在容器的生命周期中的特定时间点执行：

 1. Pre-start hooks 在容器 ttys，consoles 或 mounts 启动之前在主机的命名空间中运行。如果有任何挂载动作在这个 hook 中完成，则它们应该在 post-stop hook 中被清理掉。

 2. Pre-mount hooks 在容器的命名空间中运行，但在根文件系统被挂载之前。在这个 hook 中完成的挂载动作将在容器关闭时被自动地清理掉。

 3. Mount hooks 在容器文件系统已经被挂载之后运行，但是在容器调用 `pivot_root` 以改变它的根文件系统之前。

 4. Start hooks 在执行容器的 init 之前运行。由于这些是在切换到容器的文件系统之后执行的，被执行的命令必须被拷贝到容器的文件系统中。

 5. Post-stop hooks 在容器被关闭之后执行。

如果任何 hook 返回了错误，则容器的运行将被停止。任何 *post-stop* hook 将依然被执行。脚本产生的任何输出将以 debug 优先级被记录日志。

请参考 `lxc.container.conf` 手册页来了解配置文件格式，其中详细描述了 hooks。伴随 lxc 包一起发布的还包括一些示例 hooks，可以作为如何编写和使用这样的 hooks 的例子。

# 控制台
容器具有个数可配置的控制台。一个控制台总是位于容器的 `/dev/console`。这将被展示在你运行 `lxc-start` 的终端上，除非指定了 *-d* 选项。`/dev/console` 上的输出可以通过对 `lxc-start` 使用 `-c console-file` 选项被重定向到一个文件。额外的控制台的个数通过 `lxc.tty` 变量指定，且它通常被设置为 4。那些控制台展示在 `/dev/ttyN` (for 1 <= N <= 4) 上。要在主机上登录到控制台 3，则使用：
```
$ sudo lxc-console -n container -t 3
```

或者如果没有指定 *-t N* 选项，则将自动地选择一个未使用的控制台。要退出控制台，则使用转义序列 Ctrl-a q。注意转义序列在不带 *-d* 选项的 `lxc-start` 产生的控制台中不工作。

每个容器控制台实际上是主机的（不是客户的） pty mount 中的一个 Unix98 pty，通过客户的 `/dev/ttyN` 和 `/dev/console` 绑定挂载。然而，如果客户卸载了那些，或否则尝试访问实际的字符设备 *4:N*，它将不会为 LXC 控制台提供 getty 服务。（在默认的设置下，容器将无法访问那个字符设备，且 getty 则将失败。）当启动脚本盲目挂载新的 `/dev` 时，这很容易发生。

# 故障排除
## 打日志
如果在启动容器时某个地方出错了，则第一步应该是从 LXC 得到完整的日志输出：
```
$ sudo lxc-start -n C1 -l trace -o debug.out
```

这将使得 lxc 以最详细的级别进行日志记录，并将日志信息输出到名为 'debug.out' 的文件。如果文件 `debug.out` 已经存在了，则新的日志信息将被追加。

## 监视容器状态
有两个命令可以用来监视容器的状态改变。`lxc-monitor` 监视一个或多个容器的任何状态改变。它通常以 *-n* 选项接收一个容器名字，但在这种情况下容器的名字可以是 POSIX 正则表达式以允许监视想要的容器集合。`lxc-monitor` 持续运行打印容器的改变。`lxc-wait` 等待一个特定的状态改变然后退出。比如，
```
$ sudo lxc-monitor -n cont[0-5]*
```

将打印匹配列出的正则表达式的任何容器所有的状态改变，然而
```
$ sudo lxc-wait -n cont1 -s 'STOPPED|FROZEN'
```

将等待直到容器 cont1 进入状态 `STOPPED` 或状态 `FROZEN` 然后退出。

## 附加
从 Ubuntu 14.04 开始，可以附加到容器的命名空间。最简单的情况是简单地执行
```
$ sudo lxc-attach -n C1
```

它将启动一个附加到 C1 的命名空间的 shell，或者，在容器内部有效。附加功能是非常灵活的，它允许附加到容器命名空间的一个子集和安全上下文。参考手册页来了解更多信息。

## 容器初始化详细程度
如果 LXC 完成了容器启动，但容器 init 失败了（比如，没有显示登录提示符），则从 init 进程请求额外的详细信息可能是非常有用的。对于一个 upstart 容器，这可以是
```
$ sudo lxc-start -n C1 /sbin/init loglevel=debug
```

你也可以启动一个完全不同的程序来代替 init，比如
```
$ sudo lxc-start -n C1 /bin/bash
$ sudo lxc-start -n C1 /bin/sleep 100
$ sudo lxc-start -n C1 /bin/cat /proc/1/status
```

# LXC API
现在大多数 LXC 功能可以通过 liblxc 导出的 API 访问，其还有多种编程语言的绑定，包括 Python，lua，ruby，和 go。

下面是使用 python 绑定（可以用在 *python3-lxc* 包中）的一个例子，它创建并启动一个容器，然后等待它关闭：
```
# sudo python3
Python 3.2.3 (default, Aug 28 2012, 08:26:03)
[GCC 4.7.1 20120814 (prerelease)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import lxc
__main__:1: Warning: The python-lxc API isn't yet stable and may change at any p
oint in the future.
>>> c=lxc.Container("C1")
>>> c.create("ubuntu")
True
>>> c.start()
True
>>> c.wait("STOPPED")
True
```

# 安全性
命名空间把 ids 映射为资源。通过不给容器提供引用资源的任何 id，资源可以得到保护。这是为容器用户提供的一些安全性的基础。比如，IPC 命名空间是完全被隔离的。其它的命名空间，然而，具有各种 *leaks*，这使得特权被不适当地从一个容器施加到另一个容器或主机系统。

默认情况下，LXC 容器在一个 Apparmor 规则之下启动以限制一些行为。AppArmor 与 lxc 集成的细节在 [Apparmor](https://help.ubuntu.com/lts/serverguide/lxc.html#lxc-apparmor "Apparmor") 一节。非特权容器进一步把容器内的 root 映射为非特权的主机 userid。这样可以防止访问表示主机资源的 `/proc` 和 `/sys` 文件，以及主机上由 root 用户拥有的任何其他文件。

## 可利用的系统调用
容器与主机系统共享同一个内核是容器的一个核心功能。然而如果内核包含任何可利用的系统调用，容器也可以利用它们。一旦容器控制了内核它可以完全控制主机已知的任何资源。

自 Ubuntu 12.10 (Quantal) 开始，容器也可以被一个 seccomp filter 限制。Seccomp 是一个新的内核功能，它过滤一个任务及其子任务可以使用的系统调用。预计在不久的将来将会改进和简化规则管理，但当前的规则由一个简单的系统调用号白名单组成。规则文件以第一行的版本号（必须是 1）开始，然后第二行是一个规则类型（必须是 'whitelist'）。它后面是一个数字列表，每个一行。

通常运行一个完整的发行版容器，将需要大量的系统调用。然而，对于应用容器而言，把可用的系统调用个数减小到一点点也是可能的。即使是对于运行一个完整发行版的系统容器，可能也可以增强安全性，比如在 64 位容器中通过移除 32 位的兼容性系统调用。参考 `lxc.container.conf` 的手册页来了解如何配置一个容器使用 seccomp 的细节。默认情况下，不加载 seccomp 规则。

# 资源
*   The DeveloperWorks article [LXC: Linux container tools](https://www.ibm.com/developerworks/linux/library/l-lxc-containers/) was an early introduction to the use of containers.

*   The [Secure Containers Cookbook](http://www.ibm.com/developerworks/linux/library/l-lxc-security/index.html) demonstrated the use of security modules to make containers more secure.

*   Manual pages referenced above can be found at:
    [capabilities](http://manpages.ubuntu.com/manpages/en/man7/capabilities.7.html)
    [lxc.conf](http://manpages.ubuntu.com/manpages/en/man5/lxc.conf.5.html)

*   The upstream LXC project is hosted at [linuxcontainers.org](http://linuxcontainers.org/).

*   LXC security issues are listed and discussed at [the LXC Security wiki page](http://wiki.ubuntu.com/LxcSecurity)

*   For more on namespaces in Linux, see: S. Bhattiprolu, E. W. Biederman, S. E. Hallyn, and D. Lezcano. Virtual Servers and Check- point/Restart in Mainstream Linux. SIGOPS Operating Systems Review, 42(5), 2008.

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://help.ubuntu.com/lts/serverguide/lxc.html)
