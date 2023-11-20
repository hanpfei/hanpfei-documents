---
title: Linux ALSA 核心简单分析
date: 2023-06-23 20:23:29
categories: Linux 内核
tags:
- Linux 内核
---

Linux 内核 ALSA 框架通过向用户空间导出多个设备文件，以使用户空间程序可以与内核的音频子系统交互，可以访问音频硬件设备。

## Linux 内核 ALSA 音频框架初始化

Linux 内核 ALSA 音频框架初始化时，注册字符设备驱动，并在 `/proc` 文件系统中，创建音频设备信息相关项。Linux 内核 ALSA 初始化和退出函数定义 (位于 `sound/core/sound.c`) 如下：
```
static int major = CONFIG_SND_MAJOR;
int snd_major;
EXPORT_SYMBOL(snd_major);
 . . . . . .
static const struct file_operations snd_fops =
{
	.owner =	THIS_MODULE,
	.open =		snd_open,
	.llseek =	noop_llseek,
};
 . . . . . .
static int __init alsa_sound_init(void)
{
	snd_major = major;
	snd_ecards_limit = cards_limit;
	if (register_chrdev(major, "alsa", &snd_fops)) {
		pr_err("ALSA core: unable to register native major device number %d\n", major);
		return -EIO;
	}
	if (snd_info_init() < 0) {
		unregister_chrdev(major, "alsa");
		return -ENOMEM;
	}
#ifndef MODULE
	pr_info("Advanced Linux Sound Architecture Driver Initialized.\n");
#endif
	return 0;
}

static void __exit alsa_sound_exit(void)
{
	snd_info_done();
	unregister_chrdev(major, "alsa");
}

subsys_initcall(alsa_sound_init);
module_exit(alsa_sound_exit);
```

音频设备文件的主设备号为 116，它们的文件操作为 `snd_fops`。

## 为声卡注册 ALSA 设备文件

各个具体的音频设备文件相关信息由 `struct snd_minor` 维护，这个结构体定义 (位于 `include/sound/core.h`) 如下：
```
struct snd_minor {
	int type;			/* SNDRV_DEVICE_TYPE_XXX */
	int card;			/* card number */
	int device;			/* device number */
	const struct file_operations *f_ops;	/* file operations */
	void *private_data;		/* private data for f_ops->open */
	struct device *dev;		/* device for sysfs */
	struct snd_card *card_ptr;	/* assigned card instance */
};
```

Linux 内核 ALSA 音频框架维护一个 `struct snd_minor` 对象指针的数组，数组中有 256 个元素，每个从设备号的设备文件对应数组中的一个元素，用来保存已经注册的音频设备文件相关信息。

根据功能和使用场合的不同，音频设备文件可分为多种不同的类型，如 **CONTROL**、**HWDEP**、**RAWMIDI**、**PCM_PLAYBACK**、**PCM_CAPTURE**、**SEQUENCER**、**TIMER** 和 **COMPRESS** 等。

Linux 内核 ALSA 音频框架的其它部分需要创建音频设备文件时，调用 `snd_register_device()` 函数为声卡注册 ALSA 设备文件，该函数定义 (位于 `sound/core/sound.c`) 如下：
```
/* this one holds the actual max. card number currently available.
 * as default, it's identical with cards_limit option.  when more
 * modules are loaded manually, this limit number increases, too.
 */
int snd_ecards_limit;
EXPORT_SYMBOL(snd_ecards_limit);

static struct snd_minor *snd_minors[SNDRV_OS_MINORS];
static DEFINE_MUTEX(sound_mutex);
 . . . . . .
#ifdef CONFIG_MODULES
static struct snd_minor *autoload_device(unsigned int minor)
{
	int dev;
	mutex_unlock(&sound_mutex); /* release lock temporarily */
	dev = SNDRV_MINOR_DEVICE(minor);
	if (dev == SNDRV_MINOR_CONTROL) {
		/* /dev/aloadC? */
		int card = SNDRV_MINOR_CARD(minor);
		struct snd_card *ref = snd_card_ref(card);
		if (!ref)
			snd_request_card(card);
		else
			snd_card_unref(ref);
	} else if (dev == SNDRV_MINOR_GLOBAL) {
		/* /dev/aloadSEQ */
		snd_request_other(minor);
	}
	mutex_lock(&sound_mutex); /* reacuire lock */
	return snd_minors[minor];
}
#else /* !CONFIG_MODULES */
#define autoload_device(minor)	NULL
#endif /* CONFIG_MODULES */
 . . . . . .
#ifdef CONFIG_SND_DYNAMIC_MINORS
static int snd_find_free_minor(int type, struct snd_card *card, int dev)
{
	int minor;

	/* static minors for module auto loading */
	if (type == SNDRV_DEVICE_TYPE_SEQUENCER)
		return SNDRV_MINOR_SEQUENCER;
	if (type == SNDRV_DEVICE_TYPE_TIMER)
		return SNDRV_MINOR_TIMER;

	for (minor = 0; minor < ARRAY_SIZE(snd_minors); ++minor) {
		/* skip static minors still used for module auto loading */
		if (SNDRV_MINOR_DEVICE(minor) == SNDRV_MINOR_CONTROL)
			continue;
		if (minor == SNDRV_MINOR_SEQUENCER ||
		    minor == SNDRV_MINOR_TIMER)
			continue;
		if (!snd_minors[minor])
			return minor;
	}
	return -EBUSY;
}
#else
static int snd_find_free_minor(int type, struct snd_card *card, int dev)
{
	int minor;

	switch (type) {
	case SNDRV_DEVICE_TYPE_SEQUENCER:
	case SNDRV_DEVICE_TYPE_TIMER:
		minor = type;
		break;
	case SNDRV_DEVICE_TYPE_CONTROL:
		if (snd_BUG_ON(!card))
			return -EINVAL;
		minor = SNDRV_MINOR(card->number, type);
		break;
	case SNDRV_DEVICE_TYPE_HWDEP:
	case SNDRV_DEVICE_TYPE_RAWMIDI:
	case SNDRV_DEVICE_TYPE_PCM_PLAYBACK:
	case SNDRV_DEVICE_TYPE_PCM_CAPTURE:
	case SNDRV_DEVICE_TYPE_COMPRESS:
		if (snd_BUG_ON(!card))
			return -EINVAL;
		minor = SNDRV_MINOR(card->number, type + dev);
		break;
	default:
		return -EINVAL;
	}
	if (snd_BUG_ON(minor < 0 || minor >= SNDRV_OS_MINORS))
		return -EINVAL;
	if (snd_minors[minor])
		return -EBUSY;
	return minor;
}
#endif

/**
 * snd_register_device - Register the ALSA device file for the card
 * @type: the device type, SNDRV_DEVICE_TYPE_XXX
 * @card: the card instance
 * @dev: the device index
 * @f_ops: the file operations
 * @private_data: user pointer for f_ops->open()
 * @device: the device to register
 *
 * Registers an ALSA device file for the given card.
 * The operators have to be set in reg parameter.
 *
 * Return: Zero if successful, or a negative error code on failure.
 */
int snd_register_device(int type, struct snd_card *card, int dev,
			const struct file_operations *f_ops,
			void *private_data, struct device *device)
{
	int minor;
	int err = 0;
	struct snd_minor *preg;

	if (snd_BUG_ON(!device))
		return -EINVAL;

	preg = kmalloc(sizeof *preg, GFP_KERNEL);
	if (preg == NULL)
		return -ENOMEM;
	preg->type = type;
	preg->card = card ? card->number : -1;
	preg->device = dev;
	preg->f_ops = f_ops;
	preg->private_data = private_data;
	preg->card_ptr = card;
	mutex_lock(&sound_mutex);
	minor = snd_find_free_minor(type, card, dev);
	if (minor < 0) {
		err = minor;
		goto error;
	}

	preg->dev = device;
	device->devt = MKDEV(major, minor);
	err = device_add(device);
	if (err < 0)
		goto error;

	snd_minors[minor] = preg;
 error:
	mutex_unlock(&sound_mutex);
	if (err < 0)
		kfree(preg);
	return err;
}
EXPORT_SYMBOL(snd_register_device);
```

`snd_register_device()` 函数的参数包含如下这些：

 * `type`：设备类型
 * `card`：声卡实例
 * `dev`：设备索引
 * `f_ops`：文件操作。用户空间程序对设备文件的各种操作，都将由这里传入的文件操作执行。
 * `private_data`：`f_ops->open()` 要用到的用户指针
 * `device`：要注册的设备

`snd_register_device()` 函数的执行过程如下：

1. 分配 `struct snd_minor` 对象，并初始化它；
2. 为要注册的设备寻找可用的从设备号。这有两种策略，一种是动态从设备号，另一种是静态从设备号。无论是哪种策略，**SEQUENCER** 和 **TIMER** 类型的设备的从设备号都是固定的 1 和 33。其它类型的设备，在动态从设备号策略中，顺序查找可用的从设备号；在静态从设备号策略中，根据声卡索引、设备类型和设备索引构造从设备号；
3. 构造包含主设备号和从设备号的设备号；
4. 向设备层次体系结构添加设备；
5. 将 `struct snd_minor` 对象指针保存在 `struct snd_minor` 对象指针数组中。

`snd_register_device()` 函数调用 `device_add()` 函数向设备层次体系结构添加设备，这个函数定义 (位于 `drivers/base/core.c`) 如下：
```
int device_add(struct device *dev)
{
	struct device *parent;
	struct kobject *kobj;
	struct class_interface *class_intf;
	int error = -EINVAL;
	struct kobject *glue_dir = NULL;

	dev = get_device(dev);
	if (!dev)
		goto done;

	if (!dev->p) {
		error = device_private_init(dev);
		if (error)
			goto done;
	}

	/*
	 * for statically allocated devices, which should all be converted
	 * some day, we need to initialize the name. We prevent reading back
	 * the name, and force the use of dev_name()
	 */
	if (dev->init_name) {
		dev_set_name(dev, "%s", dev->init_name);
		dev->init_name = NULL;
	}

	/* subsystems can specify simple device enumeration */
	if (!dev_name(dev) && dev->bus && dev->bus->dev_name)
		dev_set_name(dev, "%s%u", dev->bus->dev_name, dev->id);

	if (!dev_name(dev)) {
		error = -EINVAL;
		goto name_error;
	}

	pr_debug("device: '%s': %s\n", dev_name(dev), __func__);

	parent = get_device(dev->parent);
	kobj = get_device_parent(dev, parent);
	if (IS_ERR(kobj)) {
		error = PTR_ERR(kobj);
		goto parent_error;
	}
	if (kobj)
		dev->kobj.parent = kobj;

	/* use parent numa_node */
	if (parent && (dev_to_node(dev) == NUMA_NO_NODE))
		set_dev_node(dev, dev_to_node(parent));

	/* first, register with generic layer. */
	/* we require the name to be set before, and pass NULL */
	error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);
	if (error) {
		glue_dir = get_glue_dir(dev);
		goto Error;
	}

	/* notify platform of device entry */
	error = device_platform_notify(dev, KOBJ_ADD);
	if (error)
		goto platform_error;

	error = device_create_file(dev, &dev_attr_uevent);
	if (error)
		goto attrError;

	error = device_add_class_symlinks(dev);
	if (error)
		goto SymlinkError;
	error = device_add_attrs(dev);
	if (error)
		goto AttrsError;
	error = bus_add_device(dev);
	if (error)
		goto BusError;
	error = dpm_sysfs_add(dev);
	if (error)
		goto DPMError;
	device_pm_add(dev);

	if (MAJOR(dev->devt)) {
		error = device_create_file(dev, &dev_attr_dev);
		if (error)
			goto DevAttrError;

		error = device_create_sys_dev_entry(dev);
		if (error)
			goto SysEntryError;

		devtmpfs_create_node(dev);
	}

	/* Notify clients of device addition.  This call must come
	 * after dpm_sysfs_add() and before kobject_uevent().
	 */
	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_ADD_DEVICE, dev);

	kobject_uevent(&dev->kobj, KOBJ_ADD);

	/*
	 * Check if any of the other devices (consumers) have been waiting for
	 * this device (supplier) to be added so that they can create a device
	 * link to it.
	 *
	 * This needs to happen after device_pm_add() because device_link_add()
	 * requires the supplier be registered before it's called.
	 *
	 * But this also needs to happen before bus_probe_device() to make sure
	 * waiting consumers can link to it before the driver is bound to the
	 * device and the driver sync_state callback is called for this device.
	 */
	if (dev->fwnode && !dev->fwnode->dev) {
		dev->fwnode->dev = dev;
		fw_devlink_link_device(dev);
	}

	bus_probe_device(dev);
	if (parent)
		klist_add_tail(&dev->p->knode_parent,
			       &parent->p->klist_children);

	if (dev->class) {
		mutex_lock(&dev->class->p->mutex);
		/* tie the class to the device */
		klist_add_tail(&dev->p->knode_class,
			       &dev->class->p->klist_devices);

		/* notify any interfaces that the device is here */
		list_for_each_entry(class_intf,
				    &dev->class->p->interfaces, node)
			if (class_intf->add_dev)
				class_intf->add_dev(dev, class_intf);
		mutex_unlock(&dev->class->p->mutex);
	}
done:
	put_device(dev);
	return error;
 SysEntryError:
	if (MAJOR(dev->devt))
		device_remove_file(dev, &dev_attr_dev);
 DevAttrError:
	device_pm_remove(dev);
	dpm_sysfs_remove(dev);
 DPMError:
	bus_remove_device(dev);
 BusError:
	device_remove_attrs(dev);
 AttrsError:
	device_remove_class_symlinks(dev);
 SymlinkError:
	device_remove_file(dev, &dev_attr_uevent);
 attrError:
	device_platform_notify(dev, KOBJ_REMOVE);
platform_error:
	kobject_uevent(&dev->kobj, KOBJ_REMOVE);
	glue_dir = get_glue_dir(dev);
	kobject_del(&dev->kobj);
 Error:
	cleanup_glue_dir(dev, glue_dir);
parent_error:
	put_device(parent);
name_error:
	kfree(dev->p);
	dev->p = NULL;
	goto done;
}
EXPORT_SYMBOL_GPL(device_add);
```

`device_add()` 函数调用 `device_create_file()` 和 `devtmpfs_create_node()` 等函数在 sysfs 和 devtmpfs 文件系统中创建文件。

内核各模块通过 `devtmpfs_create_node()` 函数创建 devtmpfs 文件，这个函数定义 (位于 `drivers/base/devtmpfs.c`) 如下：
```
static int devtmpfs_submit_req(struct req *req, const char *tmp)
{
	init_completion(&req->done);

	spin_lock(&req_lock);
	req->next = requests;
	requests = req;
	spin_unlock(&req_lock);

	wake_up_process(thread);
	wait_for_completion(&req->done);

	kfree(tmp);

	return req->err;
}

int devtmpfs_create_node(struct device *dev)
{
	const char *tmp = NULL;
	struct req req;

	if (!thread)
		return 0;

	req.mode = 0;
	req.uid = GLOBAL_ROOT_UID;
	req.gid = GLOBAL_ROOT_GID;
	req.name = device_get_devnode(dev, &req.mode, &req.uid, &req.gid, &tmp);
	if (!req.name)
		return -ENOMEM;

	if (req.mode == 0)
		req.mode = 0600;
	if (is_blockdev(dev))
		req.mode |= S_IFBLK;
	else
		req.mode |= S_IFCHR;

	req.dev = dev;

	return devtmpfs_submit_req(&req, tmp);
}
```

这个函数通过 `device_get_devnode()` 函数获得 devtmpfs 设备文件的文件名，创建一个 devtmpfs 设备文件创建请求，并提交。在 `devtmpfs_submit_req()` 函数中，可以看到所有的请求由单链表维护，新的请求被放在单链表的头部。

`device_get_devnode()` 函数定义 (位于 `drivers/base/core.c`) 如下：
```
const char *device_get_devnode(struct device *dev,
			       umode_t *mode, kuid_t *uid, kgid_t *gid,
			       const char **tmp)
{
	char *s;

	*tmp = NULL;

	/* the device type may provide a specific name */
	if (dev->type && dev->type->devnode)
		*tmp = dev->type->devnode(dev, mode, uid, gid);
	if (*tmp)
		return *tmp;

	/* the class may provide a specific name */
	if (dev->class && dev->class->devnode)
		*tmp = dev->class->devnode(dev, mode);
	if (*tmp)
		return *tmp;

	/* return name without allocation, tmp == NULL */
	if (strchr(dev_name(dev), '!') == NULL)
		return dev_name(dev);

	/* replace '!' in the name with '/' */
	s = kstrdup(dev_name(dev), GFP_KERNEL);
	if (!s)
		return NULL;
	strreplace(s, '!', '/');
	return *tmp = s;
}
```

`device_get_devnode()` 函数按照一定的优先级，尝试从几个地方获得设备文件名：
1. 设备的设备类型 `struct device_type` 的 `devnode` 操作；
2. 设备的总线 `struct class` 的 `devnode` 操作；
3. 设备名字。

对于音频设备，我们在 `sound/sound_core.c` 文件中看到，其总线 `struct class` 的 `devnode` 操作定义如下：
```
static char *sound_devnode(struct device *dev, umode_t *mode)
{
	if (MAJOR(dev->devt) == SOUND_MAJOR)
		return NULL;
	return kasprintf(GFP_KERNEL, "snd/%s", dev_name(dev));
}
```

## 注销 ALSA 设备文件

当不再需要某个 ALSA 设备文件时，可以注销它，这通过 `snd_unregister_device()` 函数完成。`snd_unregister_device()` 函数定义 (位于 `sound/core/sound.c`) 如下：
```
int snd_unregister_device(struct device *dev)
{
	int minor;
	struct snd_minor *preg;

	mutex_lock(&sound_mutex);
	for (minor = 0; minor < ARRAY_SIZE(snd_minors); ++minor) {
		preg = snd_minors[minor];
		if (preg && preg->dev == dev) {
			snd_minors[minor] = NULL;
			device_del(dev);
			kfree(preg);
			break;
		}
	}
	mutex_unlock(&sound_mutex);
	if (minor >= ARRAY_SIZE(snd_minors))
		return -ENOENT;
	return 0;
}
EXPORT_SYMBOL(snd_unregister_device);
```

这个函数根据传入的设备，查找对应的 `struct snd_minor` 对象，找到时，则从系统中删除设备，这包括删除 devtmpfs 文件系统中的设备文件等，并释放 `struct snd_minor` 对象。

***在 `snd_register_device()` 函数中可以看到，是给设备计算了设备号的，这里不能获取设备号，并根据设备号在 `struct snd_minor` 对象指针数组中快速查找么？***

## 音频设备文件的文件操作

注册音频字符设备时，绑定的文件操作是 `snd_fops`，这个文件操作只定义了 `open` 和 `llseek` 两个操作，其中 `llseek` 操作 `noop_llseek` 的定义 (位于 `fs/read_write.c`) 如下：
```
loff_t noop_llseek(struct file *file, loff_t offset, int whence)
{
	return file->f_pos;
}
EXPORT_SYMBOL(noop_llseek);
```

这个操作基本上什么也没做。

`open` 操作 `snd_open` 的定义 (位于 `sound/core/sound.c`) 如下：
```
static int snd_open(struct inode *inode, struct file *file)
{
	unsigned int minor = iminor(inode);
	struct snd_minor *mptr = NULL;
	const struct file_operations *new_fops;
	int err = 0;

	if (minor >= ARRAY_SIZE(snd_minors))
		return -ENODEV;
	mutex_lock(&sound_mutex);
	mptr = snd_minors[minor];
	if (mptr == NULL) {
		mptr = autoload_device(minor);
		if (!mptr) {
			mutex_unlock(&sound_mutex);
			return -ENODEV;
		}
	}
	new_fops = fops_get(mptr->f_ops);
	mutex_unlock(&sound_mutex);
	if (!new_fops)
		return -ENODEV;
	replace_fops(file, new_fops);

	if (file->f_op->open)
		err = file->f_op->open(inode, file);
	return err;
}
```

这个函数：

1. 从 `struct inode` 中获得音频设备文件的从设备号；
2. 根据从设备号，在 `struct snd_minor` 对象指针数组中，找到对应的 `struct snd_minor` 对象；
3. 从 `struct snd_minor` 对象获得它的文件操作，即注册 ALSA 设备文件时传入的文件操作；
4. 将 `struct file` 的文件操作替换为获得的文件操作；
5. 执行新的文件操作的 `open` 操作。

Done.
