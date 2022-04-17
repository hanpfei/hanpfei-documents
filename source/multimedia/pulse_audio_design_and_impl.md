---
title: PulseAudio 设计和实现浅析
date: 2021-12-20 22:05:49
categories: 音视频开发
tags:
- 音视频开发
---

**PulseAudio** 是一个 POSIX 操作系统的音频服务器系统，它是我们的音频应用程序访问系统音频设备的代理。它是所有相关的现代 Linux 发行版的组成部分，并被多个供应商用在了各种各样的移动设备中。它在应用程序和硬件设备间传递音频数据时，可以对音频数据执行一些高级操作。比如，把音频数据传给不同的机器，修改样本格式或通道数，或者混音多路音频到一路输入/输出，这些用 PulseAudio 实现都很简单。**PulseAudio** [主页](https://www.freedesktop.org/wiki/Software/PulseAudio/)。
<!--more-->
很多学习了编程语言，操作系统，计算机网络和标准库及操作系统 API 的同学都会有一些疑问：自己好像已经学习了不少知识了，但这些知识能用来做些什么，基于这些知识怎么才能实现一个有一定价值的应用，应用程序究竟是如何访问硬件设备的，如音频设备、视频设备等。

通过了解 **PulseAudio** 系统的设计和实现，可以让我们看到许许多多 POSIX 系统，特别是 Linux 系统平台下，开发一个实用的软件，所需要了解的许许多多跟系统相关的点，如基于 udev 的设备发现和设备状态变化监听，各种各样的进程间通信方式，如基于 IPv4 和 IPv6 的 TCP，Unix 域 socket，D-Bus 等，访问音频设备的 alsa 架构等。

这里简单看一下 **PulseAudio** 的设计和实现。

## PulseAudio 的代码下载及编译

编译出调试版代码，并在调试器中把代码运行起来，可以为我们研究一个软件项目提供极大的便利。我们在 Ubuntu 20.04 上编译 **PulseAudio** 并把它跑起来。

我们可以通过 Git 下载代码，当前 **PulseAudio** 最新发布的版本为 14.2，我们切换到这个分支：
```
$ git clone https://github.com/pulseaudio/pulseaudio.git
$ git checkout -t origin/stable-14.x
```

**PulseAudio** 是一个通过 meson + ninja 来构建的工程，更多关于 meson 构建系统的信息，可以参考 [meson 主页](https://mesonbuild.com/)。在开始构建 **PulseAudio** 前，需要先下载 meson：
```
$ sudo apt install meson
```

还需要下载 **PulseAudio** 本身及测试用例编译的依赖库：
```
$ sudo apt-get install libltdl-dev libsndfile1-dev cmake libsbc-dev check libbluetooth-dev  libtdb-dev  libavahi-client-dev
```

编译 **PulseAudio**：
```
$ cd pulseaudio
$ mkdir build
$ cd build/
$ meson ..
$ ninja -C .
. . . . . . . . . 
    Database:                      tdb
    Legacy Database Entry Support: true
    module-stream-restore:
      Clear old devices:           false
    Running from build tree:       true
    System User:                   pulse
    System Group:                  pulse
    Access Group:                  pulse-access
meson.build:899: WARNING: 
You do not have speex support enabled. It is strongly recommended
that you enable speex support if your platform supports it as it is
the primary method used for audio resampling and is thus a critical
part of PulseAudio on that platform.
Build targets in project: 167

Found ninja-1.8.2 at ~/bin/depot_tools/ninja
```

**PulseAudio** 默认已经运行在 Ubuntu 20.04 中了。为了能够运行我们编译出来的 **PulseAudio** 服务，需要先暂停系统中已经存在的 pulseaudio 服务：
```
systemctl --user stop pulseaudio.socket
systemctl --user stop pulseaudio.service
```

注意，执行这些命令不需要超级用户权限 sudo。

调试完成之后，还可以通过如下命令重新启动系统的  pulseaudio 服务：
```
systemctl --user start pulseaudio.socket
systemctl --user start pulseaudio.service
```

通过如下命令，可以将 pulseaudio 服务运行起来：
```
~/data/opensource/pulseaudio$ build/src/daemon/pulseaudio -n -F build/src/daemon/default.pa -p $(pwd)/build/src/modules/ 
W: [pulseaudio] caps.c: Normally all extra capabilities would be dropped now, but that's impossible because PulseAudio was built without capabilities support.
N: [pulseaudio] daemon-conf.c: Detected that we are run from the build tree, fixing search path.
N: [pulseaudio] alsa-util.c: Disabling timer-based scheduling because running inside a VM.
E: [alsa-sink-ES1371/1] alsa-sink.c: ALSA 提醒我们在该设备中写入新数据，但实际上没有什么可以写入的！
E: [alsa-sink-ES1371/1] alsa-sink.c: 这很可能是 ALSA 驱动程序 'snd_ens1371' 中的一个 bug。请向 ALSA 开发人员报告这个问题。
E: [alsa-sink-ES1371/1] alsa-sink.c: 我们因 POLLOUT 被设置而唤醒 -- 但结果是 snd_pcm_avail() 返回 0 或者另一个小于最小可用值的数值。
N: [pulseaudio] alsa-util.c: Disabling timer-based scheduling because running inside a VM.
E: [pulseaudio] stdin-util.c: Unable to read or parse data from client.
E: [pulseaudio] module.c: Failed to load module "module-gsettings" (argument: ""): initialization failed.
```

## PulseAudio 的设计和实现

pulse audio 工程有许多测试用例，可以用于帮助我们了解这个工程设计与实现。其中许多测试用例是单元测试，主要用于测试 pulse audio 基础组件的 API，但也有一些测试用例会跑完整的录制和播放流程，可以让我们听到通过 pulse audio 播放出来的音频数据，这些测试用例有 `lo-latency-test` (`pulseaudio/src/tests/lo-latency-test.c`----`pulseaudio/build/src/tests/lo-latency-test`) 和 `sync-playback` (`pulseaudio/src/tests/sync-playback.c`----`pulseaudio/build/src/tests/sync-playback`) 等。

这里我们通过观察 `sync-playback` 和 pulseaudio 服务的交互来了解 PulseAudio 的实现。

从进程角度来看，pulse audio 是客户端/服务器架构的。pulse audio 提供一个常驻系统的服务进程，它接收音频相关的操作请求命令，管理音频设备，实现与音频设备间的数据交互等。各种接入 pulse audio 库，通过 pulse audio API 访问音频设备的应用，是 pulse audio 系统服务的客户端，它们通过 pulse audio 系统服务向音频播放设备写入数据将声音播放出来，通过 pulse audio 系统服务读取录制设备以获得采集的音频数据。

pulse audio 系统服务和它的客户端可以通过多种方式实现进程间通信。对于命令的传递，可以通过 UNIX 域 socket，IPv4 socket，IP v6 socket，Linux 系统的 D-Bus 等，对于大块的音频 PCM 数据，则通过共享内存来传递。

这里看一下 pulse audio 数据传输的基本过程。

1. 发生在 pulse audio 系统服务进程中的 socket 监听
```
#0  pa_socket_server_new_unix ()
    at ../src/pulsecore/socket-server.c:175
#1  module_native_protocol_unix_LTX_pa__init () at ../src/modules/module-protocol-stub.c:330
#2  pa_module_load ()
    at ../src/pulsecore/module.c:191
#3  pa_cli_command_load ()
    at ../src/pulsecore/cli-command.c:437
#4  pa_cli_command_execute_line_stateful () at ../src/pulsecore/cli-command.c:2141
#5  pa_cli_command_execute_file_stream ()
    at ../src/pulsecore/cli-command.c:2181
#6  pa_cli_command_execute_file ()
    at ../src/pulsecore/cli-command.c:2212
#7  pa_cli_command_execute_line_stateful () at ../src/pulsecore/cli-command.c:2106
#8  pa_cli_command_execute ()
    at ../src/pulsecore/cli-command.c:2238
#9  main () at ../src/daemon/main.c:1112
```

pulseaudio 系统服务在进程启动时，会执行传入的配置文件 `build/src/daemon/default.pa` 中的命令。如配置文件中的 `load-module module-native-protocol-unix` 行会使得 pulseaudio 系统服务去加载 `module-native-protocol-unix` 模块，在这个模块中会起动监听路径为 `/run/user/1000/pulse/native` 的 Unix 域 socket，详细的执行路径如上面的堆栈信息，并设置连接建立等事件发生时的回调 `socket_server_on_connection_cb()` 等。Unix 域 socket 由模块 `module-native-protocol-unix` 管理。

2. 客户端建立连接
```
#0  pa_socket_client_new_unix ()
    at ../src/pulsecore/socket-client.c:227
#1  pa_socket_client_new_string () at ../src/pulsecore/socket-client.c:459
#2  try_next_connection () at ../src/pulse/context.c:884
#3  pa_context_connect () at ../src/pulse/context.c:1046
#4  sync_playback_test () at ../src/tests/sync-playback.c:165
#5  main () at ../src/tests/sync-playback.c:184
```

客户端应用程序，在调用 `pa_context_connect()` 接口时，会去连接 pulseaudio 系统服务监听的 Unix 域 socket。

3. pulseaudio 系统服务将命令处理程序和 IO 通道关联

pulseaudio 系统服务在收到客户端的连接时，socket-server 在其回调中，根据新连接的文件描述符创建 `pa_iochannel`，并执行回调 `socket_server_on_connection_cb()`，这个回调通过 `pa_native_protocol_connect()` 函数为连接创建所需的资源，如 `pa_client`，`pa_native_connection` 和流 `pa_pstream` 等，为流 `pa_pstream` 设置各种事件的回调，并将命令处理程序关联到流上：
```
#0  pa_native_protocol_connect () at ../src/pulsecore/protocol-native.c:5194
#1  socket_server_on_connection_cb ()
    at ../src/modules/module-protocol-stub.c:202
#2  callback ()
    at ../src/pulsecore/socket-server.c:143
#3  dispatch_pollfds () at ../src/pulse/mainloop.c:655
#4  pa_mainloop_dispatch () at ../src/pulse/mainloop.c:896
#5  pa_mainloop_iterate () at ../src/pulse/mainloop.c:927
#6  pa_mainloop_run () at ../src/pulse/mainloop.c:942
#7  main () at ../src/daemon/main.c:1170
```

`pa_native_protocol_connect()` 函数的部分代码如下：
```
    c->pstream = pa_pstream_new(p->core->mainloop, io, p->core->mempool);
    pa_pstream_set_receive_packet_callback(c->pstream, pstream_packet_callback, c);
    pa_pstream_set_receive_memblock_callback(c->pstream, pstream_memblock_callback, c);
    pa_pstream_set_die_callback(c->pstream, pstream_die_callback, c);
    pa_pstream_set_drain_callback(c->pstream, pstream_drain_callback, c);
    pa_pstream_set_revoke_callback(c->pstream, pstream_revoke_callback, c);
    pa_pstream_set_release_callback(c->pstream, pstream_release_callback, c);

    c->pdispatch = pa_pdispatch_new(p->core->mainloop, true, command_table, PA_COMMAND_MAX);

    c->record_streams = pa_idxset_new(NULL, NULL);
    c->output_streams = pa_idxset_new(NULL, NULL);
```

4. 客户端发送命令请求 pulseaudio 系统服务创建资源及 pulseaudio 系统服务处理命令

客户端在连接建立完成后，通过新创建的连接发送几个命令，请求 pulseaudio 系统服务分配资源。这些命令的交互主要包括如下这些：
```
command_get_info
command_get_info
command_auth
command_register_memfd_shmid
command_set_client_name
command_get_info
command_get_info
command_enable_srbchannel
command_create_playback_stream
command_create_playback_stream
command_create_playback_stream
command_create_playback_stream
command_get_info
command_get_info
command_get_info
command_get_info
command_get_info
command_get_info
command_cork_playback_stream
command_get_info
command_get_info
command_get_info
```
忽略不是很重要的 `command_get_info` 命令。

(1). 如前面我们提到的，客户端和 pulseaudio 系统服务通过共享内存相互传递大块的音频数据。客户端的共享内存对象在创建 `pa_context` 对象时打开，如：
```
#0  sharedmem_create ()
    at ../src/pulsecore/shm.c:141
#1  pa_shm_create_rw () at ../src/pulsecore/shm.c:241
#2  pa_mempool_new () at ../src/pulsecore/memblock.c:849
#3  pa_context_new_with_proplist () at ../src/pulse/context.c:185
#4  pa_context_new () at ../src/pulse/context.c:103
#5  sync_playback_test () at ../src/tests/sync-playback.c:160
#6  main () at ../src/tests/sync-playback.c:184
```

(2). pulseaudio 系统服务是在客户端发送了 `PA_COMMAND_AUTH` 命令，在`PA_COMMAND_AUTH` 命令处理程序中打开的共享内存对象。客户端在连接建立成功时发送 `PA_COMMAND_AUTH` 命令：
```
#0  pa_pstream_send_packet () at ../src/pulsecore/pstream.c:442
#1  pa_pstream_send_tagstruct_with_ancil_data ()
    at ../src/pulsecore/pstream-util.c:45
#2  pa_pstream_send_tagstruct_with_creds ()
    at ../src/pulsecore/pstream-util.c:58
#3  setup_context () at ../src/pulse/context.c:650
#4  on_connection () at ../src/pulse/context.c:926
#5  do_call () at ../src/pulsecore/socket-client.c:159
#6  connect_defer_cb () at ../src/pulsecore/socket-client.c:172
#7  dispatch_defer () at ../src/pulse/mainloop.c:680
#8  pa_mainloop_dispatch () at ../src/pulse/mainloop.c:887
#9  pa_mainloop_iterate () at ../src/pulse/mainloop.c:927
#10  pa_mainloop_run () at ../src/pulse/mainloop.c:942
#11  sync_playback_test () at ../src/tests/sync-playback.c:170
#12  main () at ../src/tests/sync-playback.c:184
```

(3). pulseaudio 系统服务在 `PA_COMMAND_AUTH` 命令处理程序中，创建 `pa_srbchannel` 对象，并打开共享内存对象：
```
#0  sharedmem_create ()
    at ../src/pulsecore/shm.c:141
#1  pa_shm_create_rw () at ../src/pulsecore/shm.c:241
#2  pa_mempool_new (t) at ../src/pulsecore/memblock.c:849
#3  setup_srbchannel () at ../src/pulsecore/protocol-native.c:2502
#4  command_auth ()
    at ../src/pulsecore/protocol-native.c:2740
#5  pa_pdispatch_run ()
    at ../src/pulsecore/pdispatch.c:346
#6  pstream_packet_callback ()
    at ../src/pulsecore/protocol-native.c:5027
#7  do_read () at ../src/pulsecore/pstream.c:1020
#8  do_pstream_read_write () at ../src/pulsecore/pstream.c:260
#9  io_callback () at ../src/pulsecore/pstream.c:312
#10 callback ()
    at ../src/pulsecore/iochannel.c:158
#11 dispatch_pollfds () at ../src/pulse/mainloop.c:655
#12 pa_mainloop_dispatch () at ../src/pulse/mainloop.c:896
#13 pa_mainloop_iterate () at ../src/pulse/mainloop.c:927
#14 pa_mainloop_run () at ../src/pulse/mainloop.c:942
#15 main () at ../src/daemon/main.c:1170
```

pulseaudio 系统服务在 `PA_COMMAND_AUTH` 命令的处理中，创建共享内存对象之后，还会通过 `pa_pstream_register_memfd_mempool()` 将共享内存对象的文件描述符传给客户端，如下面的 `setup_srbchannel ()` 函数定义所示：
```
static void setup_srbchannel(pa_native_connection *c, pa_mem_type_t shm_type) {
    pa_srbchannel_template srbt;
    pa_srbchannel *srb;
    pa_memchunk mc;
    pa_tagstruct *t;
    int fdlist[2];

#ifndef HAVE_CREDS
    pa_log_debug("Disabling srbchannel, reason: No fd passing support");
    return;
#endif

    if (!c->options->srbchannel) {
        pa_log_debug("Disabling srbchannel, reason: Must be enabled by module parameter");
        return;
    }

    if (c->version < 30) {
        pa_log_debug("Disabling srbchannel, reason: Protocol too old");
        return;
    }

    if (!pa_pstream_get_shm(c->pstream)) {
        pa_log_debug("Disabling srbchannel, reason: No SHM support");
        return;
    }

    if (c->rw_mempool) {
        pa_log_debug("Ignoring srbchannel setup, reason: received COMMAND_AUTH "
                     "more than once");
        return;
    }

    if (!(c->rw_mempool = pa_mempool_new(shm_type, c->protocol->core->shm_size, true))) {
        pa_log_warn("Disabling srbchannel, reason: Failed to allocate shared "
                    "writable memory pool.");
        return;
    }

    if (shm_type == PA_MEM_TYPE_SHARED_MEMFD) {
        const char *reason;
        if (pa_pstream_register_memfd_mempool(c->pstream, c->rw_mempool, &reason)) {
            pa_log_warn("Disabling srbchannel, reason: Failed to register memfd mempool: %s", reason);
            goto fail;
        }
    }
    pa_mempool_set_is_remote_writable(c->rw_mempool, true);

    srb = pa_srbchannel_new(c->protocol->core->mainloop, c->rw_mempool);
    if (!srb) {
        pa_log_debug("Failed to create srbchannel");
        goto fail;
    }
    pa_log_debug("Enabling srbchannel...");
    pa_srbchannel_export(srb, &srbt);

    /* Send enable command to client */
    t = pa_tagstruct_new();
    pa_tagstruct_putu32(t, PA_COMMAND_ENABLE_SRBCHANNEL);
    pa_tagstruct_putu32(t, (size_t) srb); /* tag */
    fdlist[0] = srbt.readfd;
    fdlist[1] = srbt.writefd;
    pa_pstream_send_tagstruct_with_fds(c->pstream, t, 2, fdlist, false);

    /* Send ringbuffer memblock to client */
    mc.memblock = srbt.memblock;
    mc.index = 0;
    mc.length = pa_memblock_get_length(srbt.memblock);
    pa_pstream_send_memblock(c->pstream, 0, 0, 0, &mc);

    c->srbpending = srb;
    return;

fail:
    if (c->rw_mempool) {
        pa_mempool_unref(c->rw_mempool);
        c->rw_mempool = NULL;
    }
}
```

`setup_srbchannel ()` 函数还会向客户端发送 `PA_COMMAND_ENABLE_SRBCHANNEL` 命令并向共享内存中写入一段数据。

(4). 随后，客户端收到 pulseaudio 系统服务发过来的共享内存对象文件描述符，attach 到这块共享内存对象上：
```
#0  shm_attach () at ../src/pulsecore/shm.c:347
#1  pa_shm_attach ()
    at ../src/pulsecore/shm.c:424
#2  segment_attach ()
    at ../src/pulsecore/memblock.c:1123
#3  pa_memimport_attach_memfd () at ../src/pulsecore/memblock.c:1213
#4  pa_pstream_attach_memfd_shmid () at ../src/pulsecore/pstream.c:377
#5  pa_pstream_register_memfd_mempool ()
    at ../src/pulsecore/pstream-util.c:174
#6  setup_complete_callback ()
    at ../src/pulse/context.c:554
#7  run_action () at ../src/pulsecore/pdispatch.c:288
#8  pa_pdispatch_run ()
    at ../src/pulsecore/pdispatch.c:341
#9  pstream_packet_callback ()
    at ../src/pulse/context.c:353
#10 do_read () at ../src/pulsecore/pstream.c:1020
#11 do_pstream_read_write () at ../src/pulsecore/pstream.c:260
#12 io_callback () at ../src/pulsecore/pstream.c:312
#13 callback ()
    at ../src/pulsecore/iochannel.c:158
#14 dispatch_pollfds () at ../src/pulse/mainloop.c:655
#15 pa_mainloop_dispatch () at ../src/pulse/mainloop.c:896
#16 pa_mainloop_iterate () at ../src/pulse/mainloop.c:927
#17 pa_mainloop_run () at ../src/pulse/mainloop.c:942
#18 sync_playback_test () at ../src/tests/sync-playback.c:170
#19 main () at ../src/tests/sync-playback.c:184
```

(5). pulseaudio 系统服务创建的共享内存对象的文件描述符被传给客户端，客户端除了 attach 到这块共享内存对象上之外，还会通过发送 `PA_COMMAND_REGISTER_MEMFD_SHMID` 命令向 pulseaudio 系统服务注册它自己创建的共享内存对象的文件描述符：
```
#0  pa_pstream_send_packet () at ../src/pulsecore/pstream.c:442
#1  pa_pstream_send_tagstruct_with_ancil_data ()
    at ../src/pulsecore/pstream-util.c:45
#2  pa_pstream_send_tagstruct_with_fds ()
    at ../src/pulsecore/pstream-util.c:81
#3  pa_pstream_register_memfd_mempool ()
    at ../src/pulsecore/pstream-util.c:186
#4  setup_complete_callback ()
    at ../src/pulse/context.c:554
```

(6). pulseaudio 系统服务处理客户端发过来的注册共享内存对象文件描述符请求：
```
#0  shm_attach () at ../src/pulsecore/shm.c:347
#1  pa_shm_attach ()
    at ../src/pulsecore/shm.c:424
#2  segment_attach ()
    at ../src/pulsecore/memblock.c:1123
#3  pa_memimport_attach_memfd () at ../src/pulsecore/memblock.c:1213
#4  pa_pstream_attach_memfd_shmid () at ../src/pulsecore/pstream.c:377
#5  pa_common_command_register_memfd_shmid ()
    at ../src/pulsecore/native-common.c:67
#6  command_register_memfd_shmid ()
    at ../src/pulsecore/protocol-native.c:2750
#7  pa_pdispatch_run ()
    at ../src/pulsecore/pdispatch.c:346
```

(7). 客户端发送 `PA_COMMAND_ENABLE_SRBCHANNEL` 命令：
```
#0  handle_srbchannel_memblock () at ../src/pulse/context.c:359
#1  pstream_memblock_callback () at ../src/pulse/context.c:413
#2  do_read () at ../src/pulsecore/pstream.c:1066
#3  do_pstream_read_write () at ../src/pulsecore/pstream.c:260
#4  io_callback () at ../src/pulsecore/pstream.c:312
```

pulseaudio 系统服务处理了客户端注册的共享内存对象文件描述符之后，向共享内存中写入数据，客户端收到数据之后，向 pulseaudio 系统服务发送 `PA_COMMAND_ENABLE_SRBCHANNEL` 命令。

(8). pulseaudio 系统服务处理 `PA_COMMAND_ENABLE_SRBCHANNEL` 命令
```
#0  pa_pstream_set_srbchannel () at ../src/pulsecore/pstream.c:1272
#1  command_enable_srbchannel ()
    at ../src/pulsecore/protocol-native.c:2559
#2  pa_pdispatch_run ()
    at ../src/pulsecore/pdispatch.c:346
#3  pstream_packet_callback ()
    at ../src/pulsecore/protocol-native.c:5027
```

`PA_COMMAND_ENABLE_SRBCHANNEL` 命令处理函数中，将 srbchannel 与 pa_pstream 关联起来。

(9). 客户端发送 `PA_COMMAND_CREATE_PLAYBACK_STREAM` 命令：
```
#0  pa_pstream_send_packet () at ../src/pulsecore/pstream.c:442
#1  pa_pstream_send_tagstruct_with_ancil_data ()
    at ../src/pulsecore/pstream-util.c:45
#2  pa_pstream_send_tagstruct_with_creds () at ../src/pulsecore/pstream-util.c:61
#3  create_stream () at ../src/pulse/stream.c:1382
#4  pa_stream_connect_playback () at ../src/pulse/stream.c:1402
#5  context_state_callback () at ../src/tests/sync-playback.c:129
```

客户端创建 pa_stream 对象，并在调用 `pa_stream_connect_playback ()` 接口时向pulseaudio 系统服务发送 `PA_COMMAND_CREATE_PLAYBACK_STREAM` 命令，请求 pulseaudio 系统服务创建播放流。

(10). pulseaudio 系统服务处理 `PA_COMMAND_CREATE_PLAYBACK_STREAM` 命令
```
#0  playback_stream_new () at ../src/pulsecore/protocol-native.c:964
#1  command_create_playback_stream ()
    at ../src/pulsecore/protocol-native.c:2067
#2  pa_pdispatch_run )
    at ../src/pulsecore/pdispatch.c:346
```

pulseaudio 系统服务处理 `PA_COMMAND_CREATE_PLAYBACK_STREAM` 命令，创建 `playback_stream` 对象，这里会给流的 `sink_input` 设置许多回调，如：
```
    s->sink_input->parent.process_msg = sink_input_process_msg;
    s->sink_input->pop = sink_input_pop_cb;
    s->sink_input->process_underrun = sink_input_process_underrun_cb;
    s->sink_input->process_rewind = sink_input_process_rewind_cb;
    s->sink_input->update_max_rewind = sink_input_update_max_rewind_cb;
    s->sink_input->update_max_request = sink_input_update_max_request_cb;
    s->sink_input->kill = sink_input_kill_cb;
    s->sink_input->moving = sink_input_moving_cb;
    s->sink_input->suspend = sink_input_suspend_cb;
    s->sink_input->send_event = sink_input_send_event_cb;
    s->sink_input->userdata = s;

    start_index = ssync ? pa_memblockq_get_read_index(ssync->memblockq) : 0;

    fix_playback_buffer_attr(s);

    pa_sink_input_get_silence(sink_input, &silence);
    memblockq_name = pa_sprintf_malloc("native protocol playback stream memblockq [%u]", s->sink_input->index);
    s->memblockq = pa_memblockq_new(
            memblockq_name,
            start_index,
            s->buffer_attr.maxlength,
            s->buffer_attr.tlength,
            &sink_input->sample_spec,
            s->buffer_attr.prebuf,
            s->buffer_attr.minreq,
            0,
            &silence);
    pa_xfree(memblockq_name);
```

(11). 客户端发送 `PA_COMMAND_CORK_PLAYBACK_STREAM` 命令：
```
#0  pa_pstream_send_packet () at ../src/pulsecore/pstream.c:442
#1  pa_pstream_send_tagstruct_with_ancil_data ()
    at ../src/pulsecore/pstream-util.c:45
#2  pa_pstream_send_tagstruct_with_creds () at ../src/pulsecore/pstream-util.c:61
#3  pa_stream_cork () at ../src/pulse/stream.c:2293
#4  stream_state_callback () at ../src/tests/sync-playback.c:95
```

客户端调用 `pa_stream_cork ()` 接口向 pulseaudio 系统服务发送 `PA_COMMAND_CORK_PLAYBACK_STREAM` 命令。这个接口用于暂停或恢复播放或录制流。

(12). pulseaudio 系统服务处理 `PA_COMMAND_CORK_PLAYBACK_STREAM` 命令
```
#0  pa_sink_input_cork () at ../src/pulsecore/sink-input.c:1577
#1  command_cork_playback_stream ()
    at ../src/pulsecore/protocol-native.c:3988
#2  pa_pdispatch_run ()
    at ../src/pulsecore/pdispatch.c:346
#3  pstream_packet_callback ()
    at ../src/pulsecore/protocol-native.c:5027
```
pulseaudio 系统服务设置 sink_input 的状态。

此外，客户端还向 pulseaudio 系统服务发送了 `command_set_client_name` 命令和多个 `command_get_info` 命令 ，这里不再详述这些命令的发送和接收处理过程。

客户端和 pulseaudio 系统服务经过上面的这些命令交互，完成整个的协商过程，它们之间可以开始进行音频数据的互相发送了。

5. 客户端发送数据

客户端通过调用 `pa_stream_write ()` 向播放流中写入数据，将播放数据发送给 pulseaudio 系统服务：
```
#0  pa_pstream_send_memblock () at ../src/pulsecore/pstream.c:477
#1  pa_stream_write_ext_free () at ../src/pulse/stream.c:1549
#2  pa_stream_write () at ../src/pulse/stream.c:1616
#3  stream_state_callback () at ../src/tests/sync-playback.c:87
```

`pa_stream_write_ext_free ()` 将要发送的数据切成一个个的块，然后通过 `pa_pstream_send_memblock ()` 发送出去：
```
        while (t_length > 0) {
            pa_memchunk chunk;

            chunk.index = 0;

            if (free_cb && !pa_pstream_get_shm(s->context->pstream)) {
                chunk.memblock = pa_memblock_new_user(s->context->mempool, (void*) t_data, t_length, free_cb, free_cb_data, 1);
                chunk.length = t_length;
            } else {
                void *d;
                size_t blk_size_max;

                /* Break large audio streams into _aligned_ blocks or the
                 * other endpoint will happily discard them upon arrival. */
                blk_size_max = pa_frame_align(pa_mempool_block_size_max(s->context->mempool), &s->sample_spec);
                chunk.length = PA_MIN(t_length, blk_size_max);
                chunk.memblock = pa_memblock_new(s->context->mempool, chunk.length);

                d = pa_memblock_acquire(chunk.memblock);
                memcpy(d, t_data, chunk.length);
                pa_memblock_release(chunk.memblock);
            }

            pa_pstream_send_memblock(s->context->pstream, s->channel, t_offset, t_seek, &chunk);

            t_offset = 0;
            t_seek = PA_SEEK_RELATIVE;

            t_data = (const uint8_t*) t_data + chunk.length;
            t_length -= chunk.length;

            pa_memblock_unref(chunk.memblock);
        }
```

`pa_pstream_send_memblock ()` 发送数据是异步的，它将一个个块的数据再按需切成一个个 item，放进发送队列里：
```
void pa_pstream_send_memblock(pa_pstream*p, uint32_t channel, int64_t offset, pa_seek_mode_t seek_mode, const pa_memchunk *chunk) {
    size_t length, idx;
    size_t bsm;

    pa_assert(p);
    pa_assert(PA_REFCNT_VALUE(p) > 0);
    pa_assert(channel != (uint32_t) -1);
    pa_assert(chunk);

    if (p->dead)
        return;

    idx = 0;
    length = chunk->length;

    bsm = pa_mempool_block_size_max(p->mempool);

    while (length > 0) {
        struct item_info *i;
        size_t n;

        if (!(i = pa_flist_pop(PA_STATIC_FLIST_GET(items))))
            i = pa_xnew(struct item_info, 1);
        i->type = PA_PSTREAM_ITEM_MEMBLOCK;

        n = PA_MIN(length, bsm);
        i->chunk.index = chunk->index + idx;
        i->chunk.length = n;
        i->chunk.memblock = pa_memblock_ref(chunk->memblock);

        i->channel = channel;
        i->offset = offset;
        i->seek_mode = seek_mode;
#ifdef HAVE_CREDS
        i->with_ancil_data = false;
#endif

        pa_queue_push(p->send_queue, i);

        idx += n;
        length -= n;
    }

    p->mainloop->defer_enable(p->defer_event, 1);
}
```

在主事件循环中，通过 srbchannel 将数据发送出去：
```
#0  pa_memexport_put () at ../src/pulsecore/memblock.c:1455
#1  prepare_next_write_item () at ../src/pulsecore/pstream.c:664
#2  do_write () at ../src/pulsecore/pstream.c:751
#3  do_pstream_read_write () at ../src/pulsecore/pstream.c:266
#4  srb_callback () at ../src/pulsecore/pstream.c:295
#5  srbchannel_rwloop () at ../src/pulsecore/srbchannel.c:190
#6  semread_cb ()
    at ../src/pulsecore/srbchannel.c:210
#7  dispatch_pollfds () at ../src/pulse/mainloop.c:655
```

`pa_pstream_send_memblock ()` 函数是如何触发 srbchannel 的回调执行的呢？这是由于客户端通过如下过程创建 `pa_srbchannel`：
```
#0  pa_srbchannel_new_from_template () at ../src/pulsecore/srbchannel.c:291
#1  handle_srbchannel_memblock () at ../src/pulse/context.c:380
#2  pstream_memblock_callback () at ../src/pulse/context.c:413
#3  do_read () at ../src/pulsecore/pstream.c:1066
#4  do_pstream_read_write () at ../src/pulsecore/pstream.c:260
#5  io_callback () at ../src/pulsecore/pstream.c:312
#6  callback ()
    at ../src/pulsecore/iochannel.c:158
#7  dispatch_pollfds () at ../src/pulse/mainloop.c:655
```

且会为 srb_channel 设置 defer_event 回调：
```
#0  pa_srbchannel_set_callback () at ../src/pulsecore/srbchannel.c:340
#1  check_srbpending () at ../src/pulsecore/pstream.c:738
#2  do_write () at ../src/pulsecore/pstream.c:755
#3  do_pstream_read_write () at ../src/pulsecore/pstream.c:266
#4  0x00007ffff7bcd19e in io_callback () at ../src/pulsecore/pstream.c:312
```

`pa_srbchannel_set_callback ()` 函数实现如下：
```
void pa_srbchannel_set_callback(pa_srbchannel *sr, pa_srbchannel_cb_t callback, void *userdata) {
    if (sr->callback)
        pa_fdsem_after_poll(sr->sem_read);

    sr->callback = callback;
    sr->cb_userdata = userdata;

    if (sr->callback) {
        /* If there are events to be read already in the ringbuffer, we will not get any IO event for that,
           because that's how pa_fdsem works. Therefore check the ringbuffer in a defer event instead. */
        if (!sr->defer_event)
            sr->defer_event = sr->mainloop->defer_new(sr->mainloop, defer_cb, sr);
        sr->mainloop->defer_enable(sr->defer_event, 1);
    }
}
```

`pa_srbchannel_new_from_template ()` 函数的实现如下：
```
pa_srbchannel* pa_srbchannel_new_from_template(pa_mainloop_api *m, pa_srbchannel_template *t)
{
    int temp;
    struct srbheader *srh;
    pa_srbchannel* sr = pa_xmalloc0(sizeof(pa_srbchannel));

    sr->mainloop = m;
    sr->memblock = t->memblock;
    pa_memblock_ref(sr->memblock);
    srh = pa_memblock_acquire(sr->memblock);

    sr->rb_read.capacity = sr->rb_write.capacity = srh->capacity;
    sr->rb_read.count = &srh->read_count;
    sr->rb_write.count = &srh->write_count;

    sr->rb_read.memory = (uint8_t*) srh + srh->readbuf_offset;
    sr->rb_write.memory = (uint8_t*) srh + srh->writebuf_offset;

    sr->sem_read = pa_fdsem_open_shm(&srh->read_semdata, t->readfd);
    if (!sr->sem_read)
        goto fail;

    sr->sem_write = pa_fdsem_open_shm(&srh->write_semdata, t->writefd);
    if (!sr->sem_write)
        goto fail;

    pa_srbchannel_swap(sr);
    temp = t->readfd; t->readfd = t->writefd; t->writefd = temp;

#ifdef DEBUG_SRBCHANNEL
    pa_log("Enabling io event on fd %d", t->readfd);
#endif

    sr->read_event = m->io_new(m, t->readfd, PA_IO_EVENT_INPUT, semread_cb, sr);
    m->io_enable(sr->read_event, PA_IO_EVENT_INPUT);

    return sr;

fail:
    pa_srbchannel_free(sr);

    return NULL;
}
```

这里会给 io event （`read_event`）关联回调。

在 `pa_pstream_send_memblock ()` 函数中通过 `p->mainloop->defer_enable(p->defer_event, 1);` 唤醒主循环，这个回调的实际实现函数为 `mainloop_defer_enable()`：
```
static void mainloop_defer_enable(pa_defer_event *e, int b) {
    pa_assert(e);
    pa_assert(!e->dead);

    if (e->enabled && !b) {
        pa_assert(e->mainloop->n_enabled_defer_events > 0);
        e->mainloop->n_enabled_defer_events--;
    } else if (!e->enabled && b) {
        e->mainloop->n_enabled_defer_events++;
        pa_mainloop_wakeup(e->mainloop);
    }

    e->enabled = b;
}
```

`mainloop_defer_enable()` 函数更新状态 `n_enabled_defer_events`，并唤醒主事件循环。被唤醒的主事件循环中，`pa_mainloop_dispatch()` 在看到这个状态时，会将事件派发给各个 defer event：
```
static unsigned dispatch_defer(pa_mainloop *m) {
    pa_defer_event *e;
    unsigned r = 0;

    if (m->n_enabled_defer_events <= 0)
        return 0;

    PA_LLIST_FOREACH(e, m->defer_events) {

        if (m->quit)
            break;

        if (e->dead || !e->enabled)
            continue;

        pa_assert(e->callback);
        e->callback(&m->api, e, e->userdata);
        r++;
    }

    return r;
}
. . . . . .
int pa_mainloop_dispatch(pa_mainloop *m) {
    unsigned dispatched = 0;

    pa_assert(m);
    pa_assert(m->state == STATE_POLLED);

    if (m->quit)
        goto quit;

    if (m->n_enabled_defer_events)
        dispatched += dispatch_defer(m);
    else {
```

`mainloop_defer_enable()` 在出发 defer event 被回调之外，也会触发 srb_channel 的 read event 被调用。

6. 服务端接收数据

在 Linux 平台上，pulseaudio 系统服务通过 ALSA 与系统音频硬件交互。ALSA 项目的主页为 [ALSA](https://www.alsa-project.org/wiki/Main_Page)。ALSA 项目有一份文档 [A Tutorial on Using the ALSA Audio API](http://equalarea.com/paul/alsa-audio.html) 简单说明了 ALSA 库接口的用法。ALSA 库的接口还可以参考 [ALSA Library API](https://www.alsa-project.org/wiki/ALSA_Library_API) 和 [ALSA library API reference](http://www.alsa-project.org/alsa-doc/alsa-lib/)，特别是 [PCM interface](https://www.alsa-project.org/alsa-doc/alsa-lib/pcm.html) 部分。

有两种方法可以用来传输音频样本数据，第一种是标准的读写接口。第二种是使用直接音频缓冲区，来与设备通信。标准的读写接口包括 [snd_pcm_writei()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html#gabc748a500743713eafa960c7d104ca6f "Write interleaved frames to a PCM.") / [snd_pcm_readi()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html#ga4c2c7bd26cf221268d59dc3bbeb9c048 "Read interleaved frames from a PCM.") 和 [snd_pcm_writen()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html#gae599772ce3d0aa6a70de143abcf145e7 "Write non interleaved frames to a PCM.") / [snd_pcm_readn()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html#gafea175455f1a405f633a43484ded3d8a "Read non interleaved frames to a PCM.")。直接读写传输借助于 mmap 的区域来传输数据，这些接口包括 [snd_pcm_mmap_begin()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m___direct.html#ga6d4acf42de554d4d1177fb035d484ea4 "Application request to access a portion of direct (mmap) area.") / [snd_pcm_mmap_commit()](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m___direct.html#gac306bd13c305825aa39dd9180a3ad520 "Application has completed the access to area requested with snd_pcm_mmap_begin.")。pulse audio 中用这两种方式都支持，一般用的是直接读写传输。

alsa-sink 内部有一个线程，在需要的时候，会去取播放的数据并写入设备(`pulseaudio/src/modules/alsa/alsa-sink.c`)：
```
static int mmap_write(struct userdata *u, pa_usec_t *sleep_usec, bool polled, bool on_timeout) {
    bool work_done = false;
. . . . . .
        for (;;) {
            pa_memchunk chunk;
            void *p;
            int err;
            const snd_pcm_channel_area_t *areas;
            snd_pcm_uframes_t offset, frames;
            snd_pcm_sframes_t sframes;
            size_t written;

            frames = (snd_pcm_uframes_t) (n_bytes / u->frame_size);
/*             pa_log_debug("%lu frames to write", (unsigned long) frames); */

            if (PA_UNLIKELY((err = pa_alsa_safe_mmap_begin(u->pcm_handle, &areas, &offset, &frames, u->hwbuf_size, &u->sink->sample_spec)) < 0)) {

                if (!after_avail && err == -EAGAIN)
                    break;

                if ((r = try_recover(u, "snd_pcm_mmap_begin", err)) == 0)
                    continue;

                if (r == 1)
                    break;

                return r;
            }

            /* Make sure that if these memblocks need to be copied they will fit into one slot */
            frames = PA_MIN(frames, u->frames_per_block);

            if (!after_avail && frames == 0)
                break;

            pa_assert(frames > 0);
            after_avail = false;

            /* Check these are multiples of 8 bit */
            pa_assert((areas[0].first & 7) == 0);
            pa_assert((areas[0].step & 7) == 0);

            /* We assume a single interleaved memory buffer */
            pa_assert((areas[0].first >> 3) == 0);
            pa_assert((areas[0].step >> 3) == u->frame_size);

            p = (uint8_t*) areas[0].addr + (offset * u->frame_size);

            written = frames * u->frame_size;
            chunk.memblock = pa_memblock_new_fixed(u->core->mempool, p, written, true);
            chunk.length = pa_memblock_get_length(chunk.memblock);
            chunk.index = 0;

            pa_sink_render_into_full(u->sink, &chunk);
            pa_memblock_unref_fixed(chunk.memblock);

            if (PA_UNLIKELY((sframes = snd_pcm_mmap_commit(u->pcm_handle, offset, frames)) < 0)) {

                if ((int) sframes == -EAGAIN)
                    break;

                if ((r = try_recover(u, "snd_pcm_mmap_commit", (int) sframes)) == 0)
                    continue;

                if (r == 1)
                    break;

                return r;
            }
```

如上所示，alsa-sink 模块通过 `snd_pcm_mmap_commit()` 接口将接收的音频数据送给音频设备。

alsa-sink 线程取数据的过程如下：
```
#0  sink_input_pop_cb () at ../src/pulsecore/protocol-native.c:1504
#1  pa_sink_input_peek ()
    at ../src/pulsecore/sink-input.c:931
#2  fill_mix_info () at ../src/pulsecore/sink.c:1100
#3  pa_sink_render_into () at ../src/pulsecore/sink.c:1340
#4  pa_sink_render_into_full () at ../src/pulsecore/sink.c:1424
#5  mmap_write () at ../src/modules/alsa/alsa-sink.c:737
#6  thread_func () at ../src/modules/alsa/alsa-sink.c:1921
#7  internal_thread_func () at ../src/pulsecore/thread-posix.c:81
#8  start_thread ( at pthread_create.c:477
```

pulseaudio 的 ALSA 模块在 `sink_input_pop_cb()` 函数中请求音频数据：
```
/* Called from thread context */
static int sink_input_pop_cb(pa_sink_input *i, size_t nbytes, pa_memchunk *chunk) {
    playback_stream *s;

    pa_sink_input_assert_ref(i);
    s = PLAYBACK_STREAM(i->userdata);
    playback_stream_assert_ref(s);
    pa_assert(chunk);

#ifdef PROTOCOL_NATIVE_DEBUG
    pa_log("%s, pop(): %lu", pa_proplist_gets(i->proplist, PA_PROP_MEDIA_NAME), (unsigned long) pa_memblockq_get_length(s->memblockq));
#endif

    if (!handle_input_underrun(s, false))
        s->is_underrun = false;

    /* This call will not fail with prebuf=0, hence we check for
       underrun explicitly in handle_input_underrun */
    if (pa_memblockq_peek(s->memblockq, chunk) < 0)
        return -1;

    chunk->length = PA_MIN(nbytes, chunk->length);

    if (i->thread_info.underrun_for > 0)
        pa_asyncmsgq_post(pa_thread_mq_get()->outq, PA_MSGOBJECT(s), PLAYBACK_STREAM_MESSAGE_STARTED, NULL, 0, NULL, NULL);

    pa_memblockq_drop(s->memblockq, chunk->length);
    playback_stream_request_bytes(s);

    return 0;
}
```

`sink_input_pop_cb()` 函数先处理 underrun ，即缓冲区中数据不足的情况：
```
#0  playback_stream_request_bytes () at ../src/pulsecore/protocol-native.c:1116
#1  handle_input_underrun () at ../src/pulsecore/protocol-native.c:1488
#2  sink_input_pop_cb () at ../src/pulsecore/protocol-native.c:1516
#3  pa_sink_input_peek ()
    at ../src/pulsecore/sink-input.c:931
```

此时会通过 `playback_stream_request_bytes ()` 给客户端发消息让它发数据过来。随后，从内存块缓存队列中取一部分数据出来：
```
#0  pa_memblockq_peek () at ../src/pulsecore/memblockq.c:474
#1  sink_input_pop_cb () at ../src/pulsecore/protocol-native.c:1521
#2  pa_sink_input_peek ()
    at ../src/pulsecore/sink-input.c:931
```

最后`sink_input_pop_cb()` 函数还是会通过 `playback_stream_request_bytes ()` 给客户端发消息请求数据。`playback_stream_request_bytes ()` 函数定义如下：
```
static void playback_stream_request_bytes(playback_stream *s) {
    size_t m;

    playback_stream_assert_ref(s);

    m = pa_memblockq_pop_missing(s->memblockq);

    if (m <= 0)
        return;

#ifdef PROTOCOL_NATIVE_DEBUG
    pa_log("request_bytes(%lu)", (unsigned long) m);
#endif

    if (pa_atomic_add(&s->missing, (int) m) <= 0)
        pa_asyncmsgq_post(pa_thread_mq_get()->outq, PA_MSGOBJECT(s), PLAYBACK_STREAM_MESSAGE_REQUEST_DATA, NULL, 0, NULL, NULL);
}
```

这最终会导致 pulseaudio 服务向客户端发送一个请求数据的命令(pulseaudio/src/pulsecore/protocol-native.c)
```
static int playback_stream_process_msg(pa_msgobject *o, int code, void*userdata, int64_t offset, pa_memchunk *chunk) {
    playback_stream *s = PLAYBACK_STREAM(o);
    playback_stream_assert_ref(s);

    if (!s->connection)
        return -1;

    switch (code) {

        case PLAYBACK_STREAM_MESSAGE_REQUEST_DATA: {
            pa_tagstruct *t;
            int l = 0;

            for (;;) {
                if ((l = pa_atomic_load(&s->missing)) <= 0)
                    return 0;

                if (pa_atomic_cmpxchg(&s->missing, l, 0))
                    break;
            }

            t = pa_tagstruct_new();
            pa_tagstruct_putu32(t, PA_COMMAND_REQUEST);
            pa_tagstruct_putu32(t, (uint32_t) -1); /* tag */
            pa_tagstruct_putu32(t, s->index);
            pa_tagstruct_putu32(t, (uint32_t) l);
            pa_pstream_send_tagstruct(s->connection->pstream, t);
```

播放数据这块的处理是典型的生产者-消费者模型。前面这里看到的都是消费者的处理，不过那生产者是怎么把数据放进队列里的呢？

`sink_input_process_msg()` 函数在处理 `SINK_INPUT_MESSAGE_SEEK` 消息和 `SINK_INPUT_MESSAGE_POST_DATA` 消息时将读取的数据放进队列里，从 backtrace 可以看到，放数据的动作是在 IO 线程，也就是 alsa-sink 线程，里完成的：
```
#0  sink_input_process_msg () at ../src/pulsecore/protocol-native.c:1318
#1  pa_asyncmsgq_dispatch ()
    at ../src/pulsecore/asyncmsgq.c:323
#2  asyncmsgq_read_work () at ../src/pulsecore/rtpoll.c:566
#3  pa_rtpoll_run () at ../src/pulsecore/rtpoll.c:238
#4  thread_func () at ../src/modules/alsa/alsa-sink.c:2003
```

`SINK_INPUT_MESSAGE_POST_DATA` 和 `SINK_INPUT_MESSAGE_POST_DATA` 消息，读取到客户端发送过来的数据时发送，pstream.c 下的 `do_read()` 通过 `pa_memimport_get ()` 获得客户端发送过来的内存块：
```
#0  pa_memimport_get () at ../src/pulsecore/memblock.c:1231
#1  do_read () at ../src/pulsecore/pstream.c:1042
#2  do_pstream_read_write () at ../src/pulsecore/pstream.c:253
#3  srb_callback () at ../src/pulsecore/pstream.c:295
#4  srbchannel_rwloop () at ../src/pulsecore/srbchannel.c:190
#5  semread_cb ()
    at ../src/pulsecore/srbchannel.c:210
```

在 `pstream_memblock_callback()` 函数中，发送 `SINK_INPUT_MESSAGE_POST_DATA` 和 `SINK_INPUT_MESSAGE_POST_DATA` 消息出去：
```
#0  pstream_memblock_callback () at ../src/pulsecore/protocol-native.c:5033
#1  do_read () at ../src/pulsecore/pstream.c:1066
#2  do_pstream_read_write () at ../src/pulsecore/pstream.c:253
```

从 PulseAudio Git repo master 分支的 commit 历史可以看到，第一笔 commit 是在 2004 年提交的，从第一笔 commit 提交到现在已经有近 20 年了，PulseAudio 是一个经过了非常多年发展，非常有历史的一个项目，想必 PulseAudio 的很多代码也是经过了相当长的历史演化变成今天这个样子的。一天两天似乎是很难将这个项目完全吃透的。这里对于 PulseAudio 的说明还很粗浅。

PulseAudio 项目不仅仅是要做一个应用和音频设备之间的数据通道，它更想成为一个音频框架。这里的介绍只是了解 PulseAudio 设计和实现的一个角度。要想对 PulseAudio 有更深入的了解，对一些技术基础的了解不可或缺：

 * 进程间通信的库接口的详细用法，这包括 D-Bus，Unix 域 socket API，IPv4/IPv6 tcp socket 的 API，共享内存接口如 `memfd_create()` / `shm_open()` 等。
 * udev 库接口的详细用法及其相关机制
 * ALSA 库接口的详细用法
 * timerfd 和 eventfd 的用法

还有很多 PulseAudio 相关的内容这里还没有涉及：

 * PulseAudio 的线程模型及线程基础设施，如 `pa_mainloop`、`pa_mainloop_api` 和 `pa_threaded_mainloop` 等。
 * PulseAudio 创建的一些概念和抽象的语义，如 `pa_context`，`pa_stream`，`pa_iochannel`，`pa_pstream`，`pa_srbchannel`，`pa_sink`，`pa_sink_input`，`pa_source`，`pa_source_output`，`pa_module` 和 `pa_card` 等等等
 * pulseaudio 系统服务管理音频设备文件
 * pulseaudio 系统服务的模块化架构设计
 * 音频 pipeline 的搭建
 * 客户端和 pulseaudio 系统服务间消息和数据交换的详细设计
 * 等等等。

参考文档：
[如何暂时禁用PulseAudio？](https://qastack.cn/ubuntu/8425/how-to-temporarily-disable-pulseaudio)
[Instructions for building and installing the current development version](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/Developer/PulseAudioFromGit/)
[README.md](https://gitlab.freedesktop.org/pulseaudio/pulseaudio/) of pulseaudio 
