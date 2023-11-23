---
title: Linux 内核音频数据传递主要流程
date: 2023-06-29 20:23:29
categories: Linux 内核
tags:
- Linux 内核
---

Linux 用户空间应用程序通过声卡驱动程序（一般牵涉到多个设备驱动程序）和 Linux 内核 ALSA 框架导出的 PCM 设备文件，如 `/dev/snd/pcmC0D0c` 和 `/dev/snd/pcmC0D0p` 等，与 Linux 内核音频设备驱动程序和音频硬件进行数据传递。PCM 设备文件的文件操作定义 (位于 *sound/core/pcm_native.c*) 如下：
```
const struct file_operations snd_pcm_f_ops[2] = {
	{
		.owner =		THIS_MODULE,
		.write =		snd_pcm_write,
		.write_iter =		snd_pcm_writev,
		.open =			snd_pcm_playback_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	},
	{
		.owner =		THIS_MODULE,
		.read =			snd_pcm_read,
		.read_iter =		snd_pcm_readv,
		.open =			snd_pcm_capture_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	}
};
```

大多数情况下，音频设备会同时提供播放和录制功能，用于播放和录制的 PCM 设备文件是一起导出的，播放和录制的 PCM 设备文件的文件操作也是一起定义的，其中索引为 `SNDRV_PCM_STREAM_PLAYBACK`，也就是 0 的文件操作用于播放，索引为 `SNDRV_PCM_STREAM_CAPTURE`，也就是 1 的文件操作用于录制。

Linux 用户空间有 *alsa-lib* 和 *tinyalsa* 等库可用于与 Linux 内核 ALSA 框架交互。*alsa-lib* 和 *tinyalsa* 等库主要通过 `ioctl` 命令与 Linux 内核 ALSA 框架交互。PCM 设备文件的 `ioctl` 操作 `snd_pcm_ioctl()` 函数定义 (位于 *sound/core/pcm_native.c*) 如下：
```
static int snd_pcm_common_ioctl(struct file *file,
				 struct snd_pcm_substream *substream,
				 unsigned int cmd, void __user *arg)
{
	struct snd_pcm_file *pcm_file = file->private_data;
	int res;

	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;

	res = snd_power_wait(substream->pcm->card, SNDRV_CTL_POWER_D0);
	if (res < 0)
		return res;

	switch (cmd) {
	case SNDRV_PCM_IOCTL_PVERSION:
		return put_user(SNDRV_PCM_VERSION, (int __user *)arg) ? -EFAULT : 0;
	case SNDRV_PCM_IOCTL_INFO:
		return snd_pcm_info_user(substream, arg);
	case SNDRV_PCM_IOCTL_TSTAMP:	/* just for compatibility */
		return 0;
	case SNDRV_PCM_IOCTL_TTSTAMP:
		return snd_pcm_tstamp(substream, arg);
	case SNDRV_PCM_IOCTL_USER_PVERSION:
		if (get_user(pcm_file->user_pversion,
			     (unsigned int __user *)arg))
			return -EFAULT;
		return 0;
	case SNDRV_PCM_IOCTL_HW_REFINE:
		return snd_pcm_hw_refine_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_PARAMS:
		return snd_pcm_hw_params_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_FREE:
		return snd_pcm_hw_free(substream);
	case SNDRV_PCM_IOCTL_SW_PARAMS:
		return snd_pcm_sw_params_user(substream, arg);
	case SNDRV_PCM_IOCTL_STATUS32:
		return snd_pcm_status_user32(substream, arg, false);
	case SNDRV_PCM_IOCTL_STATUS_EXT32:
		return snd_pcm_status_user32(substream, arg, true);
	case SNDRV_PCM_IOCTL_STATUS64:
		return snd_pcm_status_user64(substream, arg, false);
	case SNDRV_PCM_IOCTL_STATUS_EXT64:
		return snd_pcm_status_user64(substream, arg, true);
	case SNDRV_PCM_IOCTL_CHANNEL_INFO:
		return snd_pcm_channel_info_user(substream, arg);
	case SNDRV_PCM_IOCTL_PREPARE:
		return snd_pcm_prepare(substream, file);
	case SNDRV_PCM_IOCTL_RESET:
		return snd_pcm_reset(substream);
	case SNDRV_PCM_IOCTL_START:
		return snd_pcm_start_lock_irq(substream);
	case SNDRV_PCM_IOCTL_LINK:
		return snd_pcm_link(substream, (int)(unsigned long) arg);
	case SNDRV_PCM_IOCTL_UNLINK:
		return snd_pcm_unlink(substream);
	case SNDRV_PCM_IOCTL_RESUME:
		return snd_pcm_resume(substream);
	case SNDRV_PCM_IOCTL_XRUN:
		return snd_pcm_xrun(substream);
	case SNDRV_PCM_IOCTL_HWSYNC:
		return snd_pcm_hwsync(substream);
	case SNDRV_PCM_IOCTL_DELAY:
	{
		snd_pcm_sframes_t delay;
		snd_pcm_sframes_t __user *res = arg;
		int err;

		err = snd_pcm_delay(substream, &delay);
		if (err)
			return err;
		if (put_user(delay, res))
			return -EFAULT;
		return 0;
	}
	case __SNDRV_PCM_IOCTL_SYNC_PTR32:
		return snd_pcm_ioctl_sync_ptr_compat(substream, arg);
	case __SNDRV_PCM_IOCTL_SYNC_PTR64:
		return snd_pcm_sync_ptr(substream, arg);
#ifdef CONFIG_SND_SUPPORT_OLD_API
	case SNDRV_PCM_IOCTL_HW_REFINE_OLD:
		return snd_pcm_hw_refine_old_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_PARAMS_OLD:
		return snd_pcm_hw_params_old_user(substream, arg);
#endif
	case SNDRV_PCM_IOCTL_DRAIN:
		return snd_pcm_drain(substream, file);
	case SNDRV_PCM_IOCTL_DROP:
		return snd_pcm_drop(substream);
	case SNDRV_PCM_IOCTL_PAUSE:
		return snd_pcm_pause_lock_irq(substream, (unsigned long)arg);
	case SNDRV_PCM_IOCTL_WRITEI_FRAMES:
	case SNDRV_PCM_IOCTL_READI_FRAMES:
		return snd_pcm_xferi_frames_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_WRITEN_FRAMES:
	case SNDRV_PCM_IOCTL_READN_FRAMES:
		return snd_pcm_xfern_frames_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_REWIND:
		return snd_pcm_rewind_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_FORWARD:
		return snd_pcm_forward_ioctl(substream, arg);
	}
	pcm_dbg(substream->pcm, "unknown ioctl = 0x%x\n", cmd);
	return -ENOTTY;
}

static long snd_pcm_ioctl(struct file *file, unsigned int cmd,
			  unsigned long arg)
{
	struct snd_pcm_file *pcm_file;

	pcm_file = file->private_data;

	if (((cmd >> 8) & 0xff) != 'A')
		return -ENOTTY;

	return snd_pcm_common_ioctl(file, pcm_file->substream, cmd,
				     (void __user *)arg);
}
```

*alsa-lib* 和 *tinyalsa* 等库主要通过 `SNDRV_PCM_IOCTL_WRITEI_FRAMES`、`SNDRV_PCM_IOCTL_READI_FRAMES`、`SNDRV_PCM_IOCTL_WRITEN_FRAMES` 和 `SNDRV_PCM_IOCTL_READN_FRAMES` 等四个 `ioctl` 命令与 Linux 内核 ALSA 框架交换数据。这几个 `ioctl` 命令主要的区别在于，传递的数据的格式不同，`SNDRV_PCM_IOCTL_READI_FRAMES` 和 `SNDRV_PCM_IOCTL_WRITEI_FRAMES` 命令用于读写 *interleaved* 格式的音频数据，`SNDRV_PCM_IOCTL_READN_FRAMES` 和 `SNDRV_PCM_IOCTL_WRITEN_FRAMES` 命令则用于读写 *noninterleaved* 格式的音频数据。

`SNDRV_PCM_IOCTL_READI_FRAMES` 和 `SNDRV_PCM_IOCTL_WRITEI_FRAMES` 命令由 `snd_pcm_xferi_frames_ioctl()` 函数处理，`SNDRV_PCM_IOCTL_READN_FRAMES` 和 `SNDRV_PCM_IOCTL_WRITEN_FRAMES` 命令由 `snd_pcm_xfern_frames_ioctl()` 函数处理，这两个函数定义 (位于 *sound/core/pcm_native.c*) 如下：
```
static int snd_pcm_xferi_frames_ioctl(struct snd_pcm_substream *substream,
				      struct snd_xferi __user *_xferi)
{
	struct snd_xferi xferi;
	struct snd_pcm_runtime *runtime = substream->runtime;
	snd_pcm_sframes_t result;

	if (runtime->status->state == SNDRV_PCM_STATE_OPEN)
		return -EBADFD;
	if (put_user(0, &_xferi->result))
		return -EFAULT;
	if (copy_from_user(&xferi, _xferi, sizeof(xferi)))
		return -EFAULT;
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK)
		result = snd_pcm_lib_write(substream, xferi.buf, xferi.frames);
	else
		result = snd_pcm_lib_read(substream, xferi.buf, xferi.frames);
	if (put_user(result, &_xferi->result))
		return -EFAULT;
	return result < 0 ? result : 0;
}

static int snd_pcm_xfern_frames_ioctl(struct snd_pcm_substream *substream,
				      struct snd_xfern __user *_xfern)
{
	struct snd_xfern xfern;
	struct snd_pcm_runtime *runtime = substream->runtime;
	void *bufs;
	snd_pcm_sframes_t result;

	if (runtime->status->state == SNDRV_PCM_STATE_OPEN)
		return -EBADFD;
	if (runtime->channels > 128)
		return -EINVAL;
	if (put_user(0, &_xfern->result))
		return -EFAULT;
	if (copy_from_user(&xfern, _xfern, sizeof(xfern)))
		return -EFAULT;

	bufs = memdup_user(xfern.bufs, sizeof(void *) * runtime->channels);
	if (IS_ERR(bufs))
		return PTR_ERR(bufs);
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK)
		result = snd_pcm_lib_writev(substream, bufs, xfern.frames);
	else
		result = snd_pcm_lib_readv(substream, bufs, xfern.frames);
	kfree(bufs);
	if (put_user(result, &_xfern->result))
		return -EFAULT;
	return result < 0 ? result : 0;
}
```

`snd_pcm_xferi_frames_ioctl()` 和 `snd_pcm_xfern_frames_ioctl()` 函数将参数复制到内核空间栈上，并通过 `snd_pcm_lib_write()`、`snd_pcm_lib_read()`、`snd_pcm_lib_writev()` 和 `snd_pcm_lib_readv()` 四个函数执行读写操作。这四个函数定义 (位于 *include/sound/pcm.h*) 如下：
```
static inline snd_pcm_sframes_t
snd_pcm_lib_write(struct snd_pcm_substream *substream,
		  const void __user *buf, snd_pcm_uframes_t frames)
{
	return __snd_pcm_lib_xfer(substream, (void __force *)buf, true, frames, false);
}

static inline snd_pcm_sframes_t
snd_pcm_lib_read(struct snd_pcm_substream *substream,
		 void __user *buf, snd_pcm_uframes_t frames)
{
	return __snd_pcm_lib_xfer(substream, (void __force *)buf, true, frames, false);
}

static inline snd_pcm_sframes_t
snd_pcm_lib_writev(struct snd_pcm_substream *substream,
		   void __user **bufs, snd_pcm_uframes_t frames)
{
	return __snd_pcm_lib_xfer(substream, (void *)bufs, false, frames, false);
}

static inline snd_pcm_sframes_t
snd_pcm_lib_readv(struct snd_pcm_substream *substream,
		  void __user **bufs, snd_pcm_uframes_t frames)
{
	return __snd_pcm_lib_xfer(substream, (void *)bufs, false, frames, false);
}
```

所有的音频数据传递操作最终都由 `__snd_pcm_lib_xfer()` 函数完成。`__snd_pcm_lib_xfer()` 函数定义 (位于 *sound/core/pcm_lib.c*) 如下：
```
typedef int (*pcm_transfer_f)(struct snd_pcm_substream *substream,
			      int channel, unsigned long hwoff,
			      void *buf, unsigned long bytes);

typedef int (*pcm_copy_f)(struct snd_pcm_substream *, snd_pcm_uframes_t, void *,
			  snd_pcm_uframes_t, snd_pcm_uframes_t, pcm_transfer_f);
 . . . . . .
/* sanity-check for read/write methods */
static int pcm_sanity_check(struct snd_pcm_substream *substream)
{
	struct snd_pcm_runtime *runtime;
	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;
	runtime = substream->runtime;
	if (snd_BUG_ON(!substream->ops->copy_user && !runtime->dma_area))
		return -EINVAL;
	if (runtime->status->state == SNDRV_PCM_STATE_OPEN)
		return -EBADFD;
	return 0;
}

static int pcm_accessible_state(struct snd_pcm_runtime *runtime)
{
	switch (runtime->status->state) {
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_RUNNING:
	case SNDRV_PCM_STATE_PAUSED:
		return 0;
	case SNDRV_PCM_STATE_XRUN:
		return -EPIPE;
	case SNDRV_PCM_STATE_SUSPENDED:
		return -ESTRPIPE;
	default:
		return -EBADFD;
	}
}

/* update to the given appl_ptr and call ack callback if needed;
 * when an error is returned, take back to the original value
 */
int pcm_lib_apply_appl_ptr(struct snd_pcm_substream *substream,
			   snd_pcm_uframes_t appl_ptr)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	snd_pcm_uframes_t old_appl_ptr = runtime->control->appl_ptr;
	int ret;

	if (old_appl_ptr == appl_ptr)
		return 0;

	runtime->control->appl_ptr = appl_ptr;
	if (substream->ops->ack) {
		ret = substream->ops->ack(substream);
		if (ret < 0) {
			runtime->control->appl_ptr = old_appl_ptr;
			return ret;
		}
	}

	trace_applptr(substream, old_appl_ptr, appl_ptr);

	return 0;
}

/* the common loop for read/write data */
snd_pcm_sframes_t __snd_pcm_lib_xfer(struct snd_pcm_substream *substream,
				     void *data, bool interleaved,
				     snd_pcm_uframes_t size, bool in_kernel)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	snd_pcm_uframes_t xfer = 0;
	snd_pcm_uframes_t offset = 0;
	snd_pcm_uframes_t avail;
	pcm_copy_f writer;
	pcm_transfer_f transfer;
	bool nonblock;
	bool is_playback;
	int err;

	err = pcm_sanity_check(substream);
	if (err < 0)
		return err;

	is_playback = substream->stream == SNDRV_PCM_STREAM_PLAYBACK;
	if (interleaved) {
		if (runtime->access != SNDRV_PCM_ACCESS_RW_INTERLEAVED &&
		    runtime->channels > 1)
			return -EINVAL;
		writer = interleaved_copy;
	} else {
		if (runtime->access != SNDRV_PCM_ACCESS_RW_NONINTERLEAVED)
			return -EINVAL;
		writer = noninterleaved_copy;
	}

	if (!data) {
		if (is_playback)
			transfer = fill_silence;
		else
			return -EINVAL;
	} else if (in_kernel) {
		if (substream->ops->copy_kernel)
			transfer = substream->ops->copy_kernel;
		else
			transfer = is_playback ?
				default_write_copy_kernel : default_read_copy_kernel;
	} else {
		if (substream->ops->copy_user)
			transfer = (pcm_transfer_f)substream->ops->copy_user;
		else
			transfer = is_playback ?
				default_write_copy : default_read_copy;
	}

	if (size == 0)
		return 0;

	nonblock = !!(substream->f_flags & O_NONBLOCK);

	snd_pcm_stream_lock_irq(substream);
	err = pcm_accessible_state(runtime);
	if (err < 0)
		goto _end_unlock;

	runtime->twake = runtime->control->avail_min ? : 1;
	if (runtime->status->state == SNDRV_PCM_STATE_RUNNING)
		snd_pcm_update_hw_ptr(substream);

	/*
	 * If size < start_threshold, wait indefinitely. Another
	 * thread may start capture
	 */
	if (!is_playback &&
	    runtime->status->state == SNDRV_PCM_STATE_PREPARED &&
	    size >= runtime->start_threshold) {
		err = snd_pcm_start(substream);
		if (err < 0)
			goto _end_unlock;
	}

	avail = snd_pcm_avail(substream);
	while (size > 0) {
		snd_pcm_uframes_t frames, appl_ptr, appl_ofs;
		snd_pcm_uframes_t cont;
		if (!avail) {
			if (!is_playback &&
			    runtime->status->state == SNDRV_PCM_STATE_DRAINING) {
				snd_pcm_stop(substream, SNDRV_PCM_STATE_SETUP);
				goto _end_unlock;
			}
			if (nonblock) {
				err = -EAGAIN;
				goto _end_unlock;
			}
			runtime->twake = min_t(snd_pcm_uframes_t, size,
					runtime->control->avail_min ? : 1);
			err = wait_for_avail(substream, &avail);
			if (err < 0)
				goto _end_unlock;
			if (!avail)
				continue; /* draining */
		}
		frames = size > avail ? avail : size;
		appl_ptr = READ_ONCE(runtime->control->appl_ptr);
		appl_ofs = appl_ptr % runtime->buffer_size;
		cont = runtime->buffer_size - appl_ofs;
		if (frames > cont)
			frames = cont;
		if (snd_BUG_ON(!frames)) {
			err = -EINVAL;
			goto _end_unlock;
		}
		if (!atomic_inc_unless_negative(&runtime->buffer_accessing)) {
			err = -EBUSY;
			goto _end_unlock;
		}
		snd_pcm_stream_unlock_irq(substream);
		err = writer(substream, appl_ofs, data, offset, frames,
			     transfer);
		snd_pcm_stream_lock_irq(substream);
		atomic_dec(&runtime->buffer_accessing);
		if (err < 0)
			goto _end_unlock;
		err = pcm_accessible_state(runtime);
		if (err < 0)
			goto _end_unlock;
		appl_ptr += frames;
		if (appl_ptr >= runtime->boundary)
			appl_ptr -= runtime->boundary;
		err = pcm_lib_apply_appl_ptr(substream, appl_ptr);
		if (err < 0)
			goto _end_unlock;

		offset += frames;
		size -= frames;
		xfer += frames;
		avail -= frames;
		if (is_playback &&
		    runtime->status->state == SNDRV_PCM_STATE_PREPARED &&
		    snd_pcm_playback_hw_avail(runtime) >= (snd_pcm_sframes_t)runtime->start_threshold) {
			err = snd_pcm_start(substream);
			if (err < 0)
				goto _end_unlock;
		}
	}
 _end_unlock:
	runtime->twake = 0;
	if (xfer > 0 && err >= 0)
		snd_pcm_update_state(substream, runtime);
	snd_pcm_stream_unlock_irq(substream);
	return xfer > 0 ? (snd_pcm_sframes_t)xfer : err;
}
EXPORT_SYMBOL(__snd_pcm_lib_xfer);
```

`__snd_pcm_lib_xfer()` 函数的执行过程如下：

1. 调用 `pcm_sanity_check()` 函数执行合理性检查，`pcm_sanity_check()` 函数检查 DMA buffer 和状态。音频设备驱动程序需要保证，在 `__snd_pcm_lib_xfer()` 函数执行时，DMA buffer 已经准备好。

2. 根据传入的参数，即数据格式是 *interleaved* 还是 *noninterleaved*，播放还是录制，数据缓冲区是否为空，操作是由内核空间代码发起还是由用户空间代码发起等，选择 `writer` 和 `transfer` 操作。`writer` 和 `transfer` 操作主要用于在传入的数据缓冲区和 DMA buffer 之间传递数据。

3. 检查 `runtime` 的状态，当状态不为 `SNDRV_PCM_STATE_PREPARED`、`SNDRV_PCM_STATE_RUNNING` 和 `SNDRV_PCM_STATE_PAUSED` 时，报错并返回。

4. 当 `runtime` 状态为 `SNDRV_PCM_STATE_RUNNING` 时，调用 `snd_pcm_update_hw_ptr(substream)` 函数更新 hw_ptr。

5. 对于音频录制，如果 `runtime` 状态为 `SNDRV_PCM_STATE_PREPARED`，且请求录制的帧数超过发起阈值，则调用 `snd_pcm_start(substream)` 函数触发录制。

6. 调用 `snd_pcm_avail(substream)` 函数获得 DMA buffer 中以帧为单位的可用空间大小。
用户空间应用程序和 Linux 内核设备驱动程序对 DMA buffer 的访问是典型的读者-写者模型。对于播放来说，用户空间应用程序向 DMA buffer 写入数据，Linux 内核设备驱动程序从 DMA buffer 读取数据并发送给硬件设备。对于录制来说，Linux 内核设备驱动程序从硬件设备获得数据并写入 DMA buffer，用户空间应用程序从 DMA buffer 读取数据并作进一步处理。
这里的可用空间大小是站在用户空间应用程序的视角来说的，即对于播放来说，可用空间大小指还可以向 DMA buffer 写入的数据量，对于录制来说，则指可以从 DMA buffer 读取的数据量。

7. 通过一个循环，处理所有的数据传递请求。
  (1). 当 DMA buffer 的可用空间为 0 时，则等待直到可用空间大于 0。
  (2). 计算拷贝的数据量大小，拷贝的目的内存在 DMA buffer 中的位置等。
  (3). 调用 `writer` 和 `transfer` 操作在 DMA buffer 和传入的缓冲区之间传递数据，`appl_ofs` 作为 `hwoff` 参数传给 `writer` 操作。
  (4). 检查 `runtime` 的状态。
  (5). 更新并调用 `pcm_lib_apply_appl_ptr()` 函数应用新的 appl_ptr。
  (6). 根据传递的帧数，更新偏移量、要传递的帧数、已经传递的帧数和 DMA buffer 中可用的空间大小。
  (7). 对于音频播放，如果 `runtime` 状态为 `SNDRV_PCM_STATE_PREPARED`，且 DMA buffer 中可以播放的数据量超过发起阈值，则调用 `snd_pcm_start(substream)` 函数触发播放。

不同情况下，用于在传入的数据缓冲区和 DMA buffer 之间传递数据的 `writer` 和 `transfer` 操作定义 (位于 *sound/core/pcm_lib.c*) 如下：
```
/* calculate the target DMA-buffer position to be written/read */
static void *get_dma_ptr(struct snd_pcm_runtime *runtime,
			   int channel, unsigned long hwoff)
{
	return runtime->dma_area + hwoff +
		channel * (runtime->dma_bytes / runtime->channels);
}

/* default copy_user ops for write; used for both interleaved and non- modes */
static int default_write_copy(struct snd_pcm_substream *substream,
			      int channel, unsigned long hwoff,
			      void *buf, unsigned long bytes)
{
	if (copy_from_user(get_dma_ptr(substream->runtime, channel, hwoff),
			   (void __user *)buf, bytes))
		return -EFAULT;
	return 0;
}

/* default copy_kernel ops for write */
static int default_write_copy_kernel(struct snd_pcm_substream *substream,
				     int channel, unsigned long hwoff,
				     void *buf, unsigned long bytes)
{
	memcpy(get_dma_ptr(substream->runtime, channel, hwoff), buf, bytes);
	return 0;
}

/* fill silence instead of copy data; called as a transfer helper
 * from __snd_pcm_lib_write() or directly from noninterleaved_copy() when
 * a NULL buffer is passed
 */
static int fill_silence(struct snd_pcm_substream *substream, int channel,
			unsigned long hwoff, void *buf, unsigned long bytes)
{
	struct snd_pcm_runtime *runtime = substream->runtime;

	if (substream->stream != SNDRV_PCM_STREAM_PLAYBACK)
		return 0;
	if (substream->ops->fill_silence)
		return substream->ops->fill_silence(substream, channel,
						    hwoff, bytes);

	snd_pcm_format_set_silence(runtime->format,
				   get_dma_ptr(runtime, channel, hwoff),
				   bytes_to_samples(runtime, bytes));
	return 0;
}

/* default copy_user ops for read; used for both interleaved and non- modes */
static int default_read_copy(struct snd_pcm_substream *substream,
			     int channel, unsigned long hwoff,
			     void *buf, unsigned long bytes)
{
	if (copy_to_user((void __user *)buf,
			 get_dma_ptr(substream->runtime, channel, hwoff),
			 bytes))
		return -EFAULT;
	return 0;
}

/* default copy_kernel ops for read */
static int default_read_copy_kernel(struct snd_pcm_substream *substream,
				    int channel, unsigned long hwoff,
				    void *buf, unsigned long bytes)
{
	memcpy(buf, get_dma_ptr(substream->runtime, channel, hwoff), bytes);
	return 0;
}

/* call transfer function with the converted pointers and sizes;
 * for interleaved mode, it's one shot for all samples
 */
static int interleaved_copy(struct snd_pcm_substream *substream,
			    snd_pcm_uframes_t hwoff, void *data,
			    snd_pcm_uframes_t off,
			    snd_pcm_uframes_t frames,
			    pcm_transfer_f transfer)
{
	struct snd_pcm_runtime *runtime = substream->runtime;

	/* convert to bytes */
	hwoff = frames_to_bytes(runtime, hwoff);
	off = frames_to_bytes(runtime, off);
	frames = frames_to_bytes(runtime, frames);
	return transfer(substream, 0, hwoff, data + off, frames);
}

/* call transfer function with the converted pointers and sizes for each
 * non-interleaved channel; when buffer is NULL, silencing instead of copying
 */
static int noninterleaved_copy(struct snd_pcm_substream *substream,
			       snd_pcm_uframes_t hwoff, void *data,
			       snd_pcm_uframes_t off,
			       snd_pcm_uframes_t frames,
			       pcm_transfer_f transfer)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	int channels = runtime->channels;
	void **bufs = data;
	int c, err;

	/* convert to bytes; note that it's not frames_to_bytes() here.
	 * in non-interleaved mode, we copy for each channel, thus
	 * each copy is n_samples bytes x channels = whole frames.
	 */
	off = samples_to_bytes(runtime, off);
	frames = samples_to_bytes(runtime, frames);
	hwoff = samples_to_bytes(runtime, hwoff);
	for (c = 0; c < channels; ++c, ++bufs) {
		if (!data || !*bufs)
			err = fill_silence(substream, c, hwoff, NULL, frames);
		else
			err = transfer(substream, c, hwoff, *bufs + off,
				       frames);
		if (err < 0)
			return err;
	}
	return 0;
}
```

`transfer` 操作用于执行数据拷贝。根据发起数据传递的来源是内核还是用户空间应用程序，以及是播放还是录制，`transfer` 操作有四个默认实现，分别为 `default_write_copy_kernel()`、`default_read_copy_kernel()`、`default_write_copy()` 和 `default_read_copy()`，这几个函数根据 `channel` 和 `hwoff` 通过 `get_dma_ptr()` 函数获得 DMA 指针，并通过内核提供的 `copy_from_user()`、`memcpy()` 和 `copy_to_user()` 等函数在传入的数据缓冲区和 DMA buffer 之间传递数据。

从 `get_dma_ptr()` 函数可以看到，当数据格式为 `noninterleaved` 时，在 DMA buffer 中，各个 channel 的数据的布局。

`writer` 操作用于为数据拷贝做准备，它将帧为单位的偏移量和大小等数据拷贝参数转为以字节为单位，并调用 `transfer` 操作执行数据拷贝。`writer` 操作有两个默认实现，分别为 `interleaved_copy()` 和 `noninterleaved_copy()`。当数据格式为 `noninterleaved` 时，`writer` 操作分别拷贝各个 channel 的数据，传入的音频数据保存在多个不同的缓冲区中，每个通道一个，音频数据通过指针数组的方式传递。

**`writer` 和 `transfer` 操作中用到的指向 DMA buffer 的 `hwoff` 偏移量，由 `runtime->control->appl_ptr` 计算而来，在关于 DMA buffer 访问的读者-写者模型中，对于播放，`runtime->control->appl_ptr` 是写指针，对于录制，它是读指针。**

`snd_pcm_avail()` 函数用于获得 DMA buffer 中，用户空间应用程序视角的，以帧为单位的可用空间大小，Linux 内核还提供了另一个函数 `snd_pcm_hw_avail()`，用于获得音频硬件设备驱动视角的，以帧为单位的可用空间大小，这两个函数定义 (位于 *sound/core/pcm_local.h*) 如下：
```
static inline snd_pcm_uframes_t
snd_pcm_avail(struct snd_pcm_substream *substream)
{
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK)
		return snd_pcm_playback_avail(substream->runtime);
	else
		return snd_pcm_capture_avail(substream->runtime);
}

static inline snd_pcm_uframes_t
snd_pcm_hw_avail(struct snd_pcm_substream *substream)
{
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK)
		return snd_pcm_playback_hw_avail(substream->runtime);
	else
		return snd_pcm_capture_hw_avail(substream->runtime);
}
```

这两个函数根据流的类型是播放还是录制，将操作分派给另外四个函数 `snd_pcm_playback_avail()`、`snd_pcm_capture_avail()`、`snd_pcm_playback_hw_avail()` 和 `snd_pcm_capture_hw_avail()`。`snd_pcm_playback_avail()` 等函数定义 (位于 *include/sound/pcm.h*) 如下：
```
/**
 * snd_pcm_playback_avail - Get the available (writable) space for playback
 * @runtime: PCM runtime instance
 *
 * Result is between 0 ... (boundary - 1)
 */
static inline snd_pcm_uframes_t snd_pcm_playback_avail(struct snd_pcm_runtime *runtime)
{
	snd_pcm_sframes_t avail = runtime->status->hw_ptr + runtime->buffer_size - runtime->control->appl_ptr;
	if (avail < 0)
		avail += runtime->boundary;
	else if ((snd_pcm_uframes_t) avail >= runtime->boundary)
		avail -= runtime->boundary;
	return avail;
}

/**
 * snd_pcm_capture_avail - Get the available (readable) space for capture
 * @runtime: PCM runtime instance
 *
 * Result is between 0 ... (boundary - 1)
 */
static inline snd_pcm_uframes_t snd_pcm_capture_avail(struct snd_pcm_runtime *runtime)
{
	snd_pcm_sframes_t avail = runtime->status->hw_ptr - runtime->control->appl_ptr;
	if (avail < 0)
		avail += runtime->boundary;
	return avail;
}

/**
 * snd_pcm_playback_hw_avail - Get the queued space for playback
 * @runtime: PCM runtime instance
 */
static inline snd_pcm_sframes_t snd_pcm_playback_hw_avail(struct snd_pcm_runtime *runtime)
{
	return runtime->buffer_size - snd_pcm_playback_avail(runtime);
}

/**
 * snd_pcm_capture_hw_avail - Get the free space for capture
 * @runtime: PCM runtime instance
 */
static inline snd_pcm_sframes_t snd_pcm_capture_hw_avail(struct snd_pcm_runtime *runtime)
{
	return runtime->buffer_size - snd_pcm_capture_avail(runtime);
}
```

关于 DMA buffer 访问的读者-写者模型中，对于播放，`runtime->status->hw_ptr` 是读指针，(`runtime->control->appl_ptr` - `runtime->status->hw_ptr`) 是已经写入，但还未播放的数据量，对于录制，`runtime->status->hw_ptr` 是写指针。

`__snd_pcm_lib_xfer()` 函数调用 `snd_pcm_start()` 函数触发数据传递开始执行，这个函数定义 (位于 *sound/core/pcm_native.c*) 如下：
```
/*
 * start callbacks
 */
static int snd_pcm_pre_start(struct snd_pcm_substream *substream,
			     snd_pcm_state_t state)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	if (runtime->status->state != SNDRV_PCM_STATE_PREPARED)
		return -EBADFD;
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK &&
	    !snd_pcm_playback_data(substream))
		return -EPIPE;
	runtime->trigger_tstamp_latched = false;
	runtime->trigger_master = substream;
	return 0;
}

static int snd_pcm_do_start(struct snd_pcm_substream *substream,
			    snd_pcm_state_t state)
{
	if (substream->runtime->trigger_master != substream)
		return 0;
	return substream->ops->trigger(substream, SNDRV_PCM_TRIGGER_START);
}

static void snd_pcm_undo_start(struct snd_pcm_substream *substream,
			       snd_pcm_state_t state)
{
	if (substream->runtime->trigger_master == substream)
		substream->ops->trigger(substream, SNDRV_PCM_TRIGGER_STOP);
}

static void snd_pcm_post_start(struct snd_pcm_substream *substream,
			       snd_pcm_state_t state)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	snd_pcm_trigger_tstamp(substream);
	runtime->hw_ptr_jiffies = jiffies;
	runtime->hw_ptr_buffer_jiffies = (runtime->buffer_size * HZ) / 
							    runtime->rate;
	runtime->status->state = state;
	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK &&
	    runtime->silence_size > 0)
		snd_pcm_playback_silence(substream, ULONG_MAX);
	snd_pcm_timer_notify(substream, SNDRV_TIMER_EVENT_MSTART);
}

static const struct action_ops snd_pcm_action_start = {
	.pre_action = snd_pcm_pre_start,
	.do_action = snd_pcm_do_start,
	.undo_action = snd_pcm_undo_start,
	.post_action = snd_pcm_post_start
};

/**
 * snd_pcm_start - start all linked streams
 * @substream: the PCM substream instance
 *
 * Return: Zero if successful, or a negative error code.
 * The stream lock must be acquired before calling this function.
 */
int snd_pcm_start(struct snd_pcm_substream *substream)
{
	return snd_pcm_action(&snd_pcm_action_start, substream,
			      SNDRV_PCM_STATE_RUNNING);
}
```

这里调用音频硬件设备驱动程序的 `trigger` 操作，触发发起硬件的数据传输。

Done.
