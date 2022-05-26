---
title: ALSA 音频 API 使用入门
date: 2022-05-25 21:05:49
categories: 音视频开发
tags:
- 音视频开发
---

本文尝试提供一些对 ALSA 音频 API 的介绍。它不是 ALSA  API 的完整参考手册，它也不包含更复杂的软件需要解决的许多特有问题。然而，它确实尝试为技能娴熟，但对 ALSA  API 不熟悉的程序员提供充足的背景和信息，来编写使用 ALSA  API 的简单程序。
<!--more-->
## 理解音频接口

让我们先回顾一下一个音频接口的基本设计。作为应用开发者，你不需要担心这个层面的操作 - 它全部由设备驱动程序（它们是 ALSA 提供的组件之一）处理。但是，如果你想写出高效且灵活的软件，你确实需要在概念层面理解发生了什么。

音频接口是一个设备，它允许一台计算机从外部世界接收数据，并向外部世界发送数据。在计算机内部，音频数据由一串比特流表示，就像其它种类的数据。然而，音频接口可以以模拟信号（随时间变化的电压）或数字信号（一些比特流）的形式发送或接收音频。在任一情况下，计算机用以表示特定声音的位的集合，在被传递给外部世界之前，将需要先进行转换，同样，接口接收的外部信号在计算机可以使用之前，也需要先做转换。这两种转换是音频接口存在的理由。

音频接口内有一个称为 “硬件缓冲区” 的区域。当音频信号从外界到达时，接口把它转换为计算机可用的比特流，并把它存储在用于向计算机发送数据的部分硬件缓冲区。当它在硬件缓冲区中收集到足够的数据时，接口会中断计算机，告诉它已经准备好数据。对于从计算机发送到外部世界的数据，类似的过程会反过来发生。接口中断计算机，告诉它硬件缓冲区中有空间了，计算机继续在那里存储数据。接口稍后把这些比特转换为将其传递到外界需要的任何形式，然后传递。理解接口把这块缓冲区用作 “循环缓冲区” 非常重要，当它到达缓冲区的末尾时，它会继续环绕到开头。

要使这个过程正常工作，需要配置许多变量。它们包括：

 * 当在计算机使用的比特流和外界使用的信号之间转换时，接口应该使用什么样的格式？
 * 样本应该以什么样的速率在接口和计算机之间移动？
 * 在设备中断计算机之前应该有多少数据（和/或空间）？
 * 硬件缓冲区应该有多大？

前两个问题是控制音频数据质量的基础。后两个问题影响音频信号的 “延迟”。这个术语是指如下两者之间的延迟

 1. 数据从外界到达音频接口，到它对计算机可用（“输入延迟”）
 2. 数据被传递给计算机，到它被传递给外界（“输出延迟”）

这两种延迟对多种音频软件都非常重要，尽管有些程序不需要关心这些问题。

## 典型的音频应用做了什么

典型的音频应用程序具有以下粗略结构：
```
      open_the_device();
      set_the_parameters_of_the_device();
      while (!done) {
           /* one or both of these */
           receive_audio_data_from_the_device();
	   deliver_audio_data_to_the_device();
      }
      close the device
```

接收音频数据的设备和把音频数据发送给外界的设备可能是同一个，也可能不是同一个。

一个音频应用程序可能只需要发送数据给外部世界，也就是播放音频数据，可能只需要从外部世界接收数据，即音频录制，也可能两者都需要。

## 最小的播放程序

这个程序打开一个用于播放的音频接口，将其配置为立体声，16 位，44.1 kHz，交错的常规读/写访问。然后它向它传递一大块随机数据，然后退出。它代表了 ALSA 音频 API 的最简单用法，并不是一个真正的程序。

```
#include <stdio.h>
#include <stdlib.h>
#include <alsa/asoundlib.h>
	      
int main(int argc, char *argv[]) {
  int i;
  int err;

  snd_pcm_t *playback_handle;
  snd_pcm_hw_params_t *hw_params;

  unsigned int sample_rate = 44100;
  const static int32_t kFrameLength = 44100 / 100 * 2;
  int16_t buf[kFrameLength];

  if ((err = snd_pcm_open(&playback_handle, argv[1], SND_PCM_STREAM_PLAYBACK, 0))
      < 0) {
    fprintf(stderr, "cannot open audio device %s (%s)\n", argv[1],
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_malloc(&hw_params)) < 0) {
    fprintf(stderr, "cannot allocate hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_any(playback_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot initialize hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_access(playback_handle, hw_params,
      SND_PCM_ACCESS_RW_INTERLEAVED)) < 0) {
    fprintf(stderr, "cannot set access type (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_format(playback_handle, hw_params,
      SND_PCM_FORMAT_S16_LE)) < 0) {
    fprintf(stderr, "cannot set sample format (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_rate_near(playback_handle, hw_params, &sample_rate,
      0)) < 0) {
    fprintf(stderr, "cannot set sample rate (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_channels(playback_handle, hw_params, 2))
      < 0) {
    fprintf(stderr, "cannot set channel count (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params(playback_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot set parameters (%s)\n", snd_strerror(err));
    exit(1);
  }

  snd_pcm_hw_params_free(hw_params);

  if ((err = snd_pcm_prepare(playback_handle)) < 0) {
    fprintf(stderr, "cannot prepare audio interface for use (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  for (i = 0; i < 10000; ++i) {
    if ((err = snd_pcm_writei(playback_handle, buf, kFrameLength)) != kFrameLength) {
      fprintf(stderr, "write to audio interface failed (%s)\n",
          snd_strerror(err));
      exit(1);
    }
  }

  snd_pcm_close(playback_handle);
  exit(0);
}
```

要把这个程序跑起来，需要把音频接口作为参数传给它。在 Linux 上可以通过 udev 提供的 API 来获取音频接口。

```
#include <libudev.h>

static const char* retrieve_device_name(struct userdata *u, struct udev_device *dev) {
  const char *path;
  const char *t;

  path = udev_device_get_devpath(dev);
  if (!(t = udev_device_get_property_value(dev, "PULSE_NAME"))) {
    if (!(t = udev_device_get_property_value(dev, "ID_ID"))) {
      if (!(t = udev_device_get_property_value(dev, "ID_PATH"))) {
        t = path_get_card_id(path);
      }
    }
  }

  return t;
}

void process_device(struct userdata *u, struct udev_device *dev) {
  const char *action, *ff;
  if (udev_device_get_property_value(dev, "PULSE_IGNORE")) {
    printf("Ignoring %s, because marked so.\n",
        udev_device_get_devpath(dev));
    return;
  }

  if ((ff = udev_device_get_property_value(dev, "SOUND_CLASS"))
      && streq(ff, "modem")) {
    printf("Ignoring %s, because it is a modem.\n",
        udev_device_get_devpath(dev));
    return;
  }

  action = udev_device_get_action(dev);

  if (action && streq(action, "remove")) {
    remove_card(u, dev);
  } else if ((!action || streq(action, "change"))
      && udev_device_get_property_value(dev, "SOUND_INITIALIZED")) {
    retrieve_device_name(u, dev);
  }
}

static void process_path(struct userdata *u, const char *path) {
  struct udev_device *dev;

  if (!path_get_card_id(path)) {
    return;
  }

  printf("process_path path %s\n", path);
  if (!(dev = udev_device_new_from_syspath(u->udev, path))) {
    printf("Failed to get udev device object from udev.\n");
    return;
  }

  process_device(u, dev);
  udev_device_unref(dev);
}

int setup_udev(struct userdata *u) {
  int fd = -1;
  struct udev_enumerate *enumerate = NULL;
  struct udev_list_entry *item = NULL, *first = NULL;

  if (!(u->udev = udev_new())) {
    printf("Failed to initialize udev library.\n");
    goto fail;
  }
  if (!(u->monitor = udev_monitor_new_from_netlink(u->udev, "udev"))) {
    printf("Failed to initialize monitor.\n");
    goto fail;
  }

  if (udev_monitor_filter_add_match_subsystem_devtype(u->monitor, "sound", NULL)
      < 0) {
    printf("Failed to subscribe to sound devices.\n");
    goto fail;
  }

  errno = 0;
  if (udev_monitor_enable_receiving(u->monitor) < 0) {
    printf("Failed to enable monitor: %s\n", strerror(errno));
    if (errno == EPERM)
      printf("Most likely your kernel is simply too old and "
          "allows only privileged processes to listen to device events. "
          "Please upgrade your kernel to at least 2.6.30.\n");
    goto fail;
  }

  if ((fd = udev_monitor_get_fd(u->monitor)) < 0) {
    printf("Failed to get udev monitor fd.");
    goto fail;
  }

  if (!(enumerate = udev_enumerate_new(u->udev))) {
    printf("Failed to initialize udev enumerator.");
    goto fail;
  }

  if (udev_enumerate_add_match_subsystem(enumerate, "sound") < 0) {
    printf("Failed to match to subsystem.");
    goto fail;
  }

  if (udev_enumerate_scan_devices(enumerate) < 0) {
    printf("Failed to scan for devices.");
    goto fail;
  }

  first = udev_enumerate_get_list_entry(enumerate);
  udev_list_entry_foreach(item, first) {
    process_path(u, udev_list_entry_get_name(item));
  }

  udev_enumerate_unref(enumerate);

  return 0;

  fail:
  if (enumerate)
    udev_enumerate_unref(enumerate);
  return -1;
}
```

通过 udev 的 API 可以获得音频设备在 sysfs 文件系统中的路径，类似于下面这样：
```
/sys/devices/pci0000:00/0000:00:11.0/0000:02:02.0/sound/card0
```

从设备在 sysfs 文件系统中的路径可以获得这个设备的设备 ID，设备 ID 也就是设备文件的文件名中 `"card"` 后面的数字。在这个例子中，设备 ID 为 0。根据设备 ID，在 procfs 文件系统中可以查到这个设备当前状态的一些信息，某个特定音频接口相关的信息位于 procfs 文件系统中的 `/proc/asound/card[id]` 目录下，其中 [id] 为设备 ID。如在这个例子中，这个音频接口的相关信息位于 `/proc/asound/card0` 目录下，这个目录的内容类似于下面这样：
```
$ ls /proc/asound/card0/
audiopci  codec97#0  id  midi0  pcm0c  pcm0p  pcm1p
```

通过 `/proc/asound/card[id]/pcm[XX]/sub[Y]/status` 文件可以查看音频接口的子设备的状态。如在设备没有打开时，这个文件的内容如下：
```
$ cat /proc/asound/card0/pcm0p/sub0/status 
closed
```

在通过上面的 ALSA 程序打开音频设备时，`/proc/asound/card[id]/pcm[XX]/sub[Y]/status` 文件的内容如下：
```
$ cat /proc/asound/card0/pcm0p/sub0/status 
state: RUNNING
owner_pid   : 180338
trigger_time: 1079770.211430254
tstamp      : 0.000000000
delay       : 16384
avail       : 0
avail_max   : 15758
-----
hw_ptr      : 129536
appl_ptr    : 145920
```

`/proc/asound/card[id]/pcm[XX]/sub[Y]` 目录下的其它几个文件也包含音频设备的一些重要信息。如 `info` 文件的内容如下：
```
$ cat /proc/asound/card0/pcm0p/sub0/info 
card: 0
device: 0
subdevice: 0
stream: PLAYBACK
id: ES1371/1
name: ES1371 DAC2/ADC
subname: subdevice #0
class: 0
subclass: 0
subdevices_count: 1
subdevices_avail: 0
```

`sw_params` 文件的内容如下：
```
$ cat /proc/asound/card0/pcm0p/sub0/sw_params 
tstamp_mode: NONE
period_step: 1
avail_min: 221
start_threshold: 1
stop_threshold: 16384
silence_threshold: 0
silence_size: 0
boundary: 4611686018427387904
```

`hw_params` 文件的内容如下：
```
$ cat /proc/asound/card0/pcm0p/sub0/hw_params 
access: RW_INTERLEAVED
format: S16_LE
subformat: STD
channels: 2
rate: 44100 (1445100000/32768)
period_size: 221
buffer_size: 16384
```

`/proc/asound/card[id]/pcm[XX]` 目录下，目录名以 `"pcm"` 开头的目录中，以 `"p"` 结尾的目录包含音频接口的播放子设备的信息，以 `"c"` 结尾的目录则包含音频接口的录制子设备的信息。

上面的 `retrieve_device_name()` 函数从 `struct udev_device` 结构中获取设备的设备名，类似于下面这样：
```
pci-0000:02:02.0
```

ALSA  API 接受的设备参数由设备 ID 根据一些模版构成。如 PulseAudio 在 `pulseaudio/src/modules/alsa/mixer/profile-sets/default.conf` 中定义的模版包括如下这些：
```
front:%f
iec958:%f
front:%f
front:%f
surround21:%f
surround40:%f
surround41:%f
surround50:%f
surround51:%f
surround71:%f
iec958:%f
a52:%f
a52:%f
dca:%f
hdmi:%f
hdmi:%f
hdmi:%f
dcahdmi:%f
hdmi:%f,1
hdmi:%f,1
hdmi:%f,1
dcahdmi:%f,1
hdmi:%f,2
hdmi:%f,2
hdmi:%f,2
dcahdmi:%f,2
hdmi:%f,3
hdmi:%f,3
hdmi:%f,3
dcahdmi:%f,3
hdmi:%f,4
hdmi:%f,4
hdmi:%f,4
dcahdmi:%f,4
hdmi:%f,5
hdmi:%f,5
hdmi:%f,5
dcahdmi:%f,5
hdmi:%f,6
hdmi:%f,6
hdmi:%f,6
dcahdmi:%f,6
hdmi:%f,7
hdmi:%f,7
hdmi:%f,7
dcahdmi:%f,7
front:%f
front:%f
```

将模板中的 `"%f"` 替换为设备 ID，即为 ALSA  API `snd_pcm_open()`  接受的设备名参数，如 `front:0`。

## 最小的采集程序

这个程序打开一个用于采集的音频接口，将其配置为立体声，16 位，44.1 kHz，交错的常规读/写访问。然后它从其中读取一大块随机数据，然后退出，它并不是一个真正的程序。
```
#include <stdio.h>
#include <stdlib.h>
#include <alsa/asoundlib.h>
	      
int main(int argc, char *argv[]) {
  int i;
  int err;
  snd_pcm_t *capture_handle;
  snd_pcm_hw_params_t *hw_params;

  unsigned int sample_rate = 44100;
  const static int32_t kFrameLength = 44100 / 100 * 2;
  int16_t buf[kFrameLength];

  if ((err = snd_pcm_open(&capture_handle, argv[1], SND_PCM_STREAM_CAPTURE, 0))
      < 0) {
    fprintf(stderr, "cannot open audio device %s (%s)\n", argv[1],
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_malloc(&hw_params)) < 0) {
    fprintf(stderr, "cannot allocate hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_any(capture_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot initialize hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_access(capture_handle, hw_params,
      SND_PCM_ACCESS_RW_INTERLEAVED)) < 0) {
    fprintf(stderr, "cannot set access type (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_format(capture_handle, hw_params,
      SND_PCM_FORMAT_S16_LE)) < 0) {
    fprintf(stderr, "cannot set sample format (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_rate_near(capture_handle, hw_params, &sample_rate,
      0)) < 0) {
    fprintf(stderr, "cannot set sample rate (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_channels(capture_handle, hw_params, 2))
      < 0) {
    fprintf(stderr, "cannot set channel count (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params(capture_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot set parameters (%s)\n", snd_strerror(err));
    exit(1);
  }

  snd_pcm_hw_params_free(hw_params);

  if ((err = snd_pcm_prepare(capture_handle)) < 0) {
    fprintf(stderr, "cannot prepare audio interface for use (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  for (i = 0; i < 1000; ++i) {
    if ((err = snd_pcm_readi(capture_handle, buf, kFrameLength)) != kFrameLength) {
      fprintf(stderr, "read from audio interface failed (%s)\n",
          snd_strerror(err));
      exit(1);
    }
  }

  snd_pcm_close(capture_handle);
  exit(0);
}
```

## 最小的中断驱动程序

这个程序打开一个音频接口用于播放，将其配置为立体声，16 位，44.1 kHz，交错的常规读/写访问。然后等待，直到接口为播放数据做好准备，并在那时把随机数据传递给它。这种设计使你的程序可以很容易地移植到依赖于回调驱动机制的系统，比如 [JACK](http://jackit.sf.net/)，[LADSPA](http://www.ladspa.org/)，CoreAudio，VST 和许多其它的。

```
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <poll.h>
#include <alsa/asoundlib.h>
	      
snd_pcm_t *playback_handle;
int playback_callback(snd_pcm_sframes_t nframes) {
  int err;
  short buf[4096];
  printf("playback callback called with %ld frames\n", nframes);

  /* ... fill buf with data ... */

  if ((err = snd_pcm_writei(playback_handle, buf, 3072)) < 0) {
    fprintf(stderr, "write failed (%s)\n", snd_strerror(err));
  }

  if (err > 0) {
    err = nframes;
  }
  return err;
}

int main(int argc, char *argv[]) {
  snd_pcm_hw_params_t *hw_params;
  snd_pcm_sw_params_t *sw_params;
  snd_pcm_sframes_t frames_to_deliver;
  int nfds;
  int err;
  struct pollfd *pfds;

  unsigned int sample_rate = 44100;
  if ((err = snd_pcm_open(&playback_handle, argv[1], SND_PCM_STREAM_PLAYBACK, 0))
      < 0) {
    fprintf(stderr, "cannot open audio device %s (%s)\n", argv[1],
        snd_strerror(err));
    exit(1);
  }
  fprintf(stderr, "open audio device %s (%p)\n", argv[1],
      playback_handle);

  if ((err = snd_pcm_hw_params_malloc(&hw_params)) < 0) {
    fprintf(stderr, "cannot allocate hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_any(playback_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot initialize hardware parameter structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_access(playback_handle, hw_params,
      SND_PCM_ACCESS_RW_INTERLEAVED)) < 0) {
    fprintf(stderr, "cannot set access type (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_format(playback_handle, hw_params,
      SND_PCM_FORMAT_S16_LE)) < 0) {
    fprintf(stderr, "cannot set sample format (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_rate_near(playback_handle, hw_params,
      &sample_rate, 0)) < 0) {
    fprintf(stderr, "cannot set sample rate (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params_set_channels(playback_handle, hw_params, 2))
      < 0) {
    fprintf(stderr, "cannot set channel count (%s)\n", snd_strerror(err));
    exit(1);
  }

  if ((err = snd_pcm_hw_params(playback_handle, hw_params)) < 0) {
    fprintf(stderr, "cannot set parameters (%s)\n", snd_strerror(err));
    exit(1);
  }

  snd_pcm_hw_params_free(hw_params);

  /* tell ALSA to wake us up whenever 4096 or more frames
   of playback data can be delivered. Also, tell
   ALSA that we'll start the device ourselves.
   */

  if ((err = snd_pcm_sw_params_malloc(&sw_params)) < 0) {
    fprintf(stderr, "cannot allocate software parameters structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }
  if ((err = snd_pcm_sw_params_current(playback_handle, sw_params)) < 0) {
    fprintf(stderr, "cannot initialize software parameters structure (%s)\n",
        snd_strerror(err));
    exit(1);
  }
  if ((err = snd_pcm_sw_params_set_avail_min(playback_handle, sw_params, 4096))
      < 0) {
    fprintf(stderr, "cannot set minimum available count (%s)\n",
        snd_strerror(err));
    exit(1);
  }
  if ((err = snd_pcm_sw_params_set_start_threshold(playback_handle, sw_params,
      0U)) < 0) {
    fprintf(stderr, "cannot set start mode (%s)\n", snd_strerror(err));
    exit(1);
  }
  if ((err = snd_pcm_sw_params(playback_handle, sw_params)) < 0) {
    fprintf(stderr, "cannot set software parameters (%s)\n", snd_strerror(err));
    exit(1);
  }

  /* the interface will interrupt the kernel every 4096 frames, and ALSA
   will wake up this program very soon after that.
   */

  if ((err = snd_pcm_prepare(playback_handle)) < 0) {
    fprintf(stderr, "cannot prepare audio interface for use (%s)\n",
        snd_strerror(err));
    exit(1);
  }

  while (1) {
    /* wait till the interface is ready for data, or 1 second
     has elapsed.
     */
    if ((err = snd_pcm_wait(playback_handle, 1000)) < 0) {
      fprintf(stderr, "poll failed (%s)\n", strerror(errno));
      break;
    }

    /* find out how much space is available for playback data */

    if ((frames_to_deliver = snd_pcm_avail_update(playback_handle)) < 0) {
      if (frames_to_deliver == -EPIPE) {
        fprintf(stderr, "an xrun occured\n");
        break;
      } else {
        fprintf(stderr, "unknown ALSA avail update return value (%d)\n",
            frames_to_deliver);
        break;
      }
    }

    frames_to_deliver = frames_to_deliver > 4096 ? 4096 : frames_to_deliver;

    /* deliver the data */
    if (playback_callback(frames_to_deliver) != frames_to_deliver) {
      fprintf(stderr, "playback callback failed\n");
      break;
    }
  }

  snd_pcm_close(playback_handle);
  exit(0);
}
```

## 最小的全双工程序

全双工可以通过结合上面展示的播放和采集设计来实现。尽管许多已有的 Linux 音频应用程序使用这种设计，但在作者看来，它存在严重缺陷。中断驱动的例子代表了在许多情况下从根本上更好的设计。然而，把它扩展到全双工相当复杂。这也就是我建议你忘记这里的一切的原因。

## 术语

*采集*
  从外界接收数据（与 “录制” 不同，它意味着把数据存储在某个地方，这不是 ASLA  API 的一部分）

*播放*
  将数据传递给外部世界，虽然不一定，但可能被听到。

*全双工*
  在同一个接口上同时进行采集和播放的情形。

*xrun*
  一旦音频接口开始运行，它就会持续运行，直到被告知停止。它将为计算机生成数据来用和/或从计算机发送数据到外部世界。由于各种原因，你的程序可能跟不上它。对于播放，这可能导致接口需要来自计算机的新数据，但它不存在，迫使它使用留在硬件缓冲区中的旧数据。这被称为 “underrun”。对于采集，接口可能有数据要传送到计算机，但无处存储，因此它必须覆盖包含计算机尚未接收到的数据的硬件缓冲区的一部分。这被称为 “overrun”。为了简化，我们使用通用的术语 “xrun” 来指代这两种情况。

*PCM*
  脉码调制。这个短语（和首字母缩写词）描述了一种以数字形式表示模拟信号的方法。它是几乎所有计算机音频接口使用的方法，在 ALSA API 中用作 “音频” 的简写。

*通道数 (channel)*

*帧 (frame)*
  样本是一个值，它描述了 *单个通道* 上单个时间点的音频信号幅度。当我们讨论使用数字音频时，我们经常想讨论在一个时间点代表所有通道的数据。这种是样本的集合，每个通道一个，通常称为 “帧”。当我们用帧来讨论时间的流逝时，它大致相当于人们用样本来衡量的，但更准确；更重要的是，当我们讨论在某个时间点表示所有通道所需的数据量时，它是唯一有意义的单位。几乎每个 ALSA 音频 API 函数都使用帧作为数据量的测量单位。

*交错的 (interleaved)*
  一种数据布局安排，其中将同时播放的每个通道的样本按顺序相互跟随。见 “非交错的”。

*非交错的 (non-interleaved)*
  一种数据布局安排，其中单个通道的样本按顺序相互跟随；另一个通道的样本要么在另一个缓冲区中，要么在此缓冲区的另一部分中。与 “交错的” 对比。

*采样时钟 (sample clock)*
  用于标记应向外部世界传递，和/或从外部世界接收样本的时间的计时源。一些音频接口允许你使用外部采样时钟，可以是 “字时钟” 信号（通常在许多工作室中使用），也可以是 “自动同步”，它使用传入数字数据中的时钟信号。所有音频接口都至少有一个接口本身存在的采样时钟源。通常是一个小型晶体时钟。有些接口不允许改变时钟的频率，有些接口的时钟实际上并不以你期望的频率运行（44.1kHz 等）。不可能期望两个采样时钟以完全相同的频率运行 - 如果你需要两个采样流保持彼此同步，它们必须从相同的采样时钟运行。

## 如何做 . . .

### 打开设别

ALSA 将采集和播放分开 ......

### 设置参数

我们在上面提到，需要设置许多参数才能使音频接口完成其工作。然而，由于你的程序实际上不直接与硬件交互，而是和控制硬件的设备驱动交互，实际上有两组不同的参数：

#### 硬件参数

这些是直接影响音频接口硬件的参数。

*采样率*
  如果接口有模拟 I/O，这将控制完成 A/D/D/A 转换的速率。对于全数字接口，它控制用于将数字音频数据移入/移出外部世界的时钟速度。在某些音频接口上，其它特定于设备的配置可能意味着你的程序无法控制这个值（例如，当接口被告知使用外部字时钟源来确定采样率时）。

*样本格式*
  这个参数控制用于将数据传输到接口和从接口传输数据的采样格式。它可能与硬件直接支持的格式对应，也可能不对应。

*通道数*
  希望这是不言自明的。

*数据访问和布局*
  这个参数控制程序向接口传递数据或从接口接收数据的方式。有两个参数由 4 种可能的设置控制。一个参数是是否将使用 “读/写” 模型，其中使用显式函数调用来传输数据。这里的另一个选项是使用 “mmap 模式”，其中数据通过在内存区域之间复制来传输，API 调用只需要注意它何时开始和结束。
  另一个参数是数据布局是交错的还是非交错的。

*中断间隔*
  这个参数决定了接口在每次完整遍历其硬件缓冲区时将产生多少中断。可以通过指定周期数、周期大小来设置它。由于这决定了在接口中断计算机之前必须累积的空间/数据的帧数，因而它是控制延迟的中心。

*缓冲区大小*
  这个参数决定了硬件缓冲区有多大。它可以以时间或帧数为单位来指定。

#### 软件参数

这些是控制设备驱动的操作而不是硬件本身的参数。大多数使用 ALSA 音频 API 的程序将不需要设置它们中的任何一个；有些应用程序可能需要设置它们中的一些参数。

*何时启动设备*
  当你打开音频接口时，ALSA 确保它不是活跃的 - 没有数据正在被移动到它的外部连接器，或从它的连接器移出。想必你希望在某个时候开始数据传输。要做到这一点有多个选项。
  这里的控制点时起动阈值，它定义了在自动地起动设备之前需要的空间/数据的帧数。如果为播放设置了某个非零值，则在设备起动之前需要先预填充播放缓冲区。如果设置为零，则写入设备的第一块数据（或首次尝试从采集流读取数据）将起动设备。
  你还可以使用 `snd_pcm_start` 显式地起动设备，但在播放流的情况下这需要预填充缓冲区。如果你尝试起动流，而没有先做这些，则你将得到返回码 `-EPIPE`，指示没有数据等待被传递给播放硬件缓冲区。

*xruns 时做什么*
  如果发生了 xrun，设备驱动可以，如果请求的话，采取一些措施来处理它。选项包括停止设备，或使用于播放的全部或部分硬件缓冲区静音。

    停止阈值

        如果可用的数据/空间的帧数达到或超过这个值，驱动将停止接口。

    静音阈值

        如果播放流的可用空间的帧数达到或超过这个值，驱动将用静音数据填充部分播放硬件缓冲区。

    静音大小

        当达到静音阈值级别时，这决定了将多少静音帧写入播放硬件缓冲区。

*唤醒的可用最小空间/数据*
  使用 `poll(2)` 或 `select(2)` 来确定何时可以将音频数据传输到接口或从接口传输的程序可以将其设置为控制相对于硬件缓冲区状态的哪个点，它们希望被唤醒。

*传输块大小*
  这个参数决定了向/从设备硬件缓冲区传输数据时使用的帧数。

还有一些其它的软件参数，但在这里我们不必关心它们。

## 为什么你可以忘掉这里的一切

一句话：[JACK](http://jackit.sf.net/)。

总之，最好不要使用 ALSA 这么底层的音频 API。开源社区有几个非常好的 Linux 音频服务实现：[JACK](http://jackit.sf.net/)，[PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/)，和 [PipeWire](https://pipewire.org/)，这些音频服务的接口使用起来更简单方便。

 **参考文档**
[A Tutorial on Using the ALSA Audio API](http://equalarea.com/paul/alsa-audio.html)

[PipeWire Late Summer Update 2020](https://blogs.gnome.org/uraeus/2020/09/04/pipewire-late-summer-update-2020/)

[How to Use PipeWire to replace PulseAudio in Ubuntu 22.04](https://ubuntuhandbook.org/index.php/2022/04/pipewire-replace-pulseaudio-ubuntu-2204/)

[Linux下录音和播放](https://zhuanlan.zhihu.com/p/58834651?from_voters_page=true)

[使用PipeWire取代PulseAudio](http://equalarea.com/paul/alsa-audio.html)

[用了200天的PipeWire到底好在哪？](https://lado.me/2022/04/16/whats-good-about-pipewire/)

[How use PulseAudio and JACK?](https://jackaudio.org/faq/pulseaudio_and_jack.html)

[Writing an ALSA driver: PCM Hardware Description](http://ben-collins.blogspot.com/2010/05/writing-alsa-driver-pcm-hardware.html)

[Analysis of Android ALSA Audio System Architecture (1) - - Understanding Audio from Loopback](https://programmer.help/blogs/5c30d5f30caf7.html)
