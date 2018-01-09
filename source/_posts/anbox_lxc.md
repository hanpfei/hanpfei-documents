---
title: Anbox LXC
date: 2017-12-06 11:05:49
categories: 虚拟化
tags:
- 虚拟化
---

# Anbox LXC 编译安装
在命令行中，通过 `anbox` 命令直接启动 Anbox 的容器管理器时，它将动态链接系统中安装的 liblxc。由 Anbox 项目的 snapcraft.yaml 文件，可以看到在创建 Anbox 的 snap 时，LXC 编译相关的选项：
<!--more-->
```
  lxc:
    source: https://github.com/lxc/lxc
    source-type: git
    source-tag: lxc-2.0.7
    build-packages:
      - libapparmor-dev
      - libcap-dev
      - libgnutls28-dev
      - libseccomp-dev
      - pkg-config
    plugin: autotools
    configflags:
      - --disable-selinux
      - --disable-python
      - --disable-lua
      - --disable-tests
      - --disable-examples
      - --disable-doc
      - --disable-api-docs
      - --disable-bash
      - --disable-cgmanager
      - --disable-apparmor
      - --disable-seccomp
      - --enable-capabilities
      - --with-rootfs-path=/var/snap/anbox/common/lxc/
      - --libexecdir=/snap/anbox/current/libexec/
    organize:
      snap/anbox/current/libexec: libexec
    prime:
      - lib/liblxc.so.1
      - lib/liblxc.so.1.2.0
      - libexec/lxc/lxc-monitord
      - bin/lxc-start
      - bin/lxc-stop
      - bin/lxc-info
      - bin/lxc-attach
      - bin/lxc-ls
      - bin/lxc-top
```

需要特别注意的是 `source-tag` 和 `configflags` 配置，其中前者指明了编译 LXC 的代码的 TAG，后者指明了配置时的选项。

要手动为系统编译及安装用于 Anbox 的 LXC 组件时，最好将代码切换到 TAG 为
 lxc-2.0.7 的版本，同时在配置编译时，加上上面 `configflags` 中除 `--libexecdir=/snap/anbox/current/libexec/` 的配置项，就像下面这样：
```
$ git clone https://github.com/lxc/lxc.git
$ cd lxc
$ git checkout lxc-2.0.7
$ ./autogen.sh
$ ./configure --disable-selinux --disable-python  --disable-lua  --disable-tests  --disable-examples  --disable-doc  --disable-api-docs  --disable-bash  --disable-cgmanager  --disable-apparmor  --disable-seccomp  --enable-capabilities  --with-rootfs-path=/var/snap/anbox/common/lxc/
$ make
$ sudo make install
```

liblxc 将被安装到 `/usr/local/lib/` 下。这样 Anbox 就可以像下面这样启动了，如容器管理器：
```
Debug/src/anbox container-manager --data-path=/var/snap/anbox/common/ --android-image=/snap/anbox/current/android.img
```

# 系统服务 Anbox
对于通过 snap 安装的 Anbox，可以通过移除 Anbox 项目的 snapcraft.yaml 文件中，`parts:` 部分如上面的那段代码，lxc 描述的相关内容，以及 `parts:` 部分的 `anbox:` 块中的如下内容：
```
    after:
      - lxc
```

以使打包 Anbox snap 时使用系统中安装的 LXC 组件。

当系统中安装的 LXC 做了修改更新，需要重新打包 Anbox snap 时，则应首先执行如下命令：
```
$ snapcraft clean
```

做一个清理。然后再执行如下命令打包：
```
$ snapcraft
```

启动 anbox 的容器管理器，无论是像下面这样通过系统服务：
```
$ sudo systemctl start snap.anbox.container-manager
```

还是直接在命令行中通过像上面那样的命令，容器启动过程中，Anbox 打印的日志位于 `/var/snap/anbox/common/logs/container.log`，而 LXC 日志则位于 `/var/snap/anbox/common/containers/lxc-monitord.log`，这些日志信息可以用于故障诊断。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。
