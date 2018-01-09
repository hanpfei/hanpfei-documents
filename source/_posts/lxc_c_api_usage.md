---
title: LXC C API 使用
date: 2017-12-05 11:05:49
categories: 虚拟化
tags:
- 虚拟化
---

LXC 提供了稳定的 C API 以及大量不同语言的绑定。LXC 版本中的 liblxc1 API 的接口可能会增加，但不会在不调用 liblxc2 的情况下删除或更改现有符号。
<!--more-->
与稳定的 API 一起发布的第一个 LXC 版本是 1.0.0。只有 `lxccontainer.h` 头文件中列出的符号是 API 的一部分，所有其它的都是 LXC 的内部符号，且可能在任何时间点改变。

API 用法最好的示例是各语言的绑定的实现以及 LXC 工具本身。

官方同时提供了当前 git master 的代码所对应的最新的 [API 文档](https://linuxcontainers.org/lxc/apidoc/)。

这里是一个简单的示例，用于说明如何创建、启动、停止及销毁一个容器。
```
#include <stdio.h>

#include <lxc/lxccontainer.h>

int main() {
    struct lxc_container *c;
    int ret = 1;

    fprintf(stderr, "To setup container struct.\n");
    /* Setup container struct */
    c = lxc_container_new("apicontainer", NULL);
    if (!c) {
        fprintf(stderr, "Failed to setup lxc_container struct\n");
        goto out;
    }

    if (!c->is_defined(c)) {
        fprintf(stderr, "To create the container.\n");
        /* Create the container */
        if (!c->createl(c, "download", NULL, NULL, LXC_CREATE_QUIET,
                        "-d", "ubuntu", "-r", "trusty", "-a", "i386", NULL)) {
            fprintf(stderr, "Failed to create container rootfs\n");
            goto out;
        }
    } else {
        fprintf(stderr, "Container already exists\n");
    }

    fprintf(stderr, "To start the container\n");
    /* Start the container */
    if (!c->start(c, 0, NULL)) {
        fprintf(stderr, "Failed to start the container\n");
        goto out;
    }

    /* Query some information */
    printf("Container state: %s\n", c->state(c));
    printf("Container PID: %d\n", c->init_pid(c));

    /* Stop the container */
    if (!c->shutdown(c, 30)) {
        printf("Failed to cleanly shutdown the container, forcing.\n");
        if (!c->stop(c)) {
            fprintf(stderr, "Failed to kill the container.\n");
            goto out;
        }
    }

    /* Destroy the container */
    if (!c->destroy(c)) {
        fprintf(stderr, "Failed to destroy the container.\n");
        goto out;
    }

    ret = 0;
out:
    lxc_container_put(c);
    return ret;
}
```

在 LXC 中，用 `struct lxc_container` 结构描述一个容器，该结构封装了如容器名称，容器配置文件路径，容器配置等信息，以及可以对单个容器执行的所有操作。`struct lxc_container` 结构定义（位于 `lxccontainer.h` 头文件中，lxc-2.0.7 版）如下：
```
struct lxc_container {
	char *name;

	char *configfile;

	char *pidfile;

	struct lxc_lock *slock;

	struct lxc_lock *privlock;

	int numthreads;

	struct lxc_conf *lxc_conf;

	char *error_string;

	int error_num;

	bool daemonize;

	char *config_path;

	bool (*is_defined)(struct lxc_container *c);

	const char *(*state)(struct lxc_container *c);

	bool (*is_running)(struct lxc_container *c);

	bool (*freeze)(struct lxc_container *c);

	bool (*unfreeze)(struct lxc_container *c);

	pid_t (*init_pid)(struct lxc_container *c);

	bool (*load_config)(struct lxc_container *c, const char *alt_file);

	bool (*start)(struct lxc_container *c, int useinit, char * const argv[]);

	bool (*startl)(struct lxc_container *c, int useinit, ...);

	bool (*stop)(struct lxc_container *c);

	bool (*want_daemonize)(struct lxc_container *c, bool state);

	bool (*want_close_all_fds)(struct lxc_container *c, bool state);

	char *(*config_file_name)(struct lxc_container *c);

	bool (*wait)(struct lxc_container *c, const char *state, int timeout);

	bool (*set_config_item)(struct lxc_container *c, const char *key, const char *value);

	bool (*destroy)(struct lxc_container *c);

	bool (*save_config)(struct lxc_container *c, const char *alt_file);

	bool (*create)(struct lxc_container *c, const char *t, const char *bdevtype,
			struct bdev_specs *specs, int flags, char *const argv[]);

	bool (*createl)(struct lxc_container *c, const char *t, const char *bdevtype,
			struct bdev_specs *specs, int flags, ...);

	bool (*rename)(struct lxc_container *c, const char *newname);

	bool (*reboot)(struct lxc_container *c);

	bool (*shutdown)(struct lxc_container *c, int timeout);

	void (*clear_config)(struct lxc_container *c);

	bool (*clear_config_item)(struct lxc_container *c, const char *key);

	int (*get_config_item)(struct lxc_container *c, const char *key, char *retv, int inlen);

	char* (*get_running_config_item)(struct lxc_container *c, const char *key);

	int (*get_keys)(struct lxc_container *c, const char *key, char *retv, int inlen);

	char** (*get_interfaces)(struct lxc_container *c);

	char** (*get_ips)(struct lxc_container *c, const char* interface, const char* family, int scope);

	int (*get_cgroup_item)(struct lxc_container *c, const char *subsys, char *retv, int inlen);

	bool (*set_cgroup_item)(struct lxc_container *c, const char *subsys, const char *value);

	const char *(*get_config_path)(struct lxc_container *c);

	bool (*set_config_path)(struct lxc_container *c, const char *path);

	struct lxc_container *(*clone)(struct lxc_container *c, const char *newname,
			const char *lxcpath, int flags, const char *bdevtype,
			const char *bdevdata, uint64_t newsize, char **hookargs);

	int (*console_getfd)(struct lxc_container *c, int *ttynum, int *masterfd);

	int (*console)(struct lxc_container *c, int ttynum,
			int stdinfd, int stdoutfd, int stderrfd, int escape);

	int (*attach)(struct lxc_container *c, lxc_attach_exec_t exec_function,
			void *exec_payload, lxc_attach_options_t *options, pid_t *attached_process);

	int (*attach_run_wait)(struct lxc_container *c, lxc_attach_options_t *options, const char *program, const char * const argv[]);

	int (*attach_run_waitl)(struct lxc_container *c, lxc_attach_options_t *options, const char *program, const char *arg, ...);

	int (*snapshot)(struct lxc_container *c, const char *commentfile);

	int (*snapshot_list)(struct lxc_container *c, struct lxc_snapshot **snapshots);

	bool (*snapshot_restore)(struct lxc_container *c, const char *snapname, const char *newname);

	bool (*snapshot_destroy)(struct lxc_container *c, const char *snapname);

	bool (*may_control)(struct lxc_container *c);

	bool (*add_device_node)(struct lxc_container *c, const char *src_path, const char *dest_path);

	bool (*remove_device_node)(struct lxc_container *c, const char *src_path, const char *dest_path);

	bool (*attach_interface)(struct lxc_container *c, const char *dev, const char *dst_dev);

	bool (*detach_interface)(struct lxc_container *c, const char *dev, const char *dst_dev);

	bool (*checkpoint)(struct lxc_container *c, char *directory, bool stop, bool verbose);

	bool (*restore)(struct lxc_container *c, char *directory, bool verbose);

	bool (*destroy_with_snapshots)(struct lxc_container *c);

	bool (*snapshot_destroy_all)(struct lxc_container *c);

	int (*migrate)(struct lxc_container *c, unsigned int cmd, struct migrate_opts *opts, unsigned int size);
};
```

`struct lxc_container` 结构是 LXC C API 最主要的部分。除了 `struct lxc_container` 结构，LXC 还提供了如下的 API：
```
struct lxc_snapshot {
	char *name; /*!< Name of snapshot */
	char *comment_pathname; /*!< Full path to snapshots comment file (may be \c NULL) */
	char *timestamp; /*!< Time snapshot was created */
	char *lxcpath; /*!< Full path to LXCPATH for snapshot */

	/*!
	 * \brief De-allocate the snapshot.
	 * \param s snapshot.
	 */
	void (*free)(struct lxc_snapshot *s);
};

struct bdev_specs {
	char *fstype; /*!< Filesystem type */
	uint64_t fssize;  /*!< Filesystem size in bytes */
	struct {
		char *zfsroot; /*!< ZFS root path */
	} zfs;
	struct {
		char *vg; /*!< LVM Volume Group name */
		char *lv; /*!< LVM Logical Volume name */
		char *thinpool; /*!< LVM thin pool to use, if any */
	} lvm;
	char *dir; /*!< Directory path */
	struct {
		char *rbdname; /*!< RBD image name */
		char *rbdpool; /*!< Ceph pool name */
	} rbd;
};

enum {
	MIGRATE_PRE_DUMP,
	MIGRATE_DUMP,
	MIGRATE_RESTORE,
};

struct migrate_opts {
	/* new members should be added at the end */
	char *directory;
	bool verbose;

	bool stop; /* stop the container after dump? */
	char *predump_dir; /* relative to directory above */
	char *pageserver_address; /* where should memory pages be send? */
	char *pageserver_port;
};

struct lxc_container *lxc_container_new(const char *name, const char *configpath);

int lxc_container_get(struct lxc_container *c);

int lxc_container_put(struct lxc_container *c);

int lxc_get_wait_states(const char **states);

const char *lxc_get_global_config_item(const char *key);

const char *lxc_get_version(void);

int list_defined_containers(const char *lxcpath, char ***names, struct lxc_container ***cret);

int list_active_containers(const char *lxcpath, char ***names, struct lxc_container ***cret);

int list_all_containers(const char *lxcpath, char ***names, struct lxc_container ***cret);

void lxc_log_close(void);
```

`struct lxc_container` 结构的生命周期如下：
![](https://www.wolfcstech.com/images/1315506-9d97e6656c872449.png)

即通过 `lxc_container_new()` 函数创建并初始化，通过 `lxc_container_get()` 增加结构的引用计数，然后通过结构的成员函数指针执行针对特定容器的操作，最后通过 `lxc_container_put()` 减小引用计数或释放结构体。`lxc_container_put()` 与 `struct lxc_container` 结构的 `destroy()` 的主要差别在于，后者是销毁磁盘上与特定容器有关的所有数据，而前者只是释放在内存中用于表示特定容器的结构体，而不会对磁盘上容器的数据产生影响。

其中 `lxc_container_new()` 函数的实现（位于 `lxc/src/lxc/lxccontainer.c`）如下：
```
struct lxc_container *lxc_container_new(const char *name, const char *configpath)
{
	struct lxc_container *c;

	if (!name)
		return NULL;

	c = malloc(sizeof(*c));
	if (!c) {
		fprintf(stderr, "failed to malloc lxc_container\n");
		return NULL;
	}
	memset(c, 0, sizeof(*c));

	if (configpath)
		c->config_path = strdup(configpath);
	else
		c->config_path = strdup(lxc_global_config_value("lxc.lxcpath"));

	if (!c->config_path) {
		fprintf(stderr, "Out of memory\n");
		goto err;
	}

	remove_trailing_slashes(c->config_path);
	c->name = malloc(strlen(name)+1);
	if (!c->name) {
		fprintf(stderr, "Error allocating lxc_container name\n");
		goto err;
	}
	strcpy(c->name, name);

	c->numthreads = 1;
	if (!(c->slock = lxc_newlock(c->config_path, name))) {
		fprintf(stderr, "failed to create lock\n");
		goto err;
	}

	if (!(c->privlock = lxc_newlock(NULL, NULL))) {
		fprintf(stderr, "failed to alloc privlock\n");
		goto err;
	}

	if (!set_config_filename(c)) {
		fprintf(stderr, "Error allocating config file pathname\n");
		goto err;
	}

	if (file_exists(c->configfile) && !lxcapi_load_config(c, NULL))
		goto err;

	if (ongoing_create(c) == 2) {
		ERROR("Error: %s creation was not completed", c->name);
		container_destroy(c);
		lxcapi_clear_config(c);
	}
	c->daemonize = true;
	c->pidfile = NULL;

	// assign the member functions
	c->is_defined = lxcapi_is_defined;
	c->state = lxcapi_state;
	c->is_running = lxcapi_is_running;
	c->freeze = lxcapi_freeze;
	c->unfreeze = lxcapi_unfreeze;
	c->console = lxcapi_console;
	c->console_getfd = lxcapi_console_getfd;
	c->init_pid = lxcapi_init_pid;
	c->load_config = lxcapi_load_config;
	c->want_daemonize = lxcapi_want_daemonize;
	c->want_close_all_fds = lxcapi_want_close_all_fds;
	c->start = lxcapi_start;
	c->startl = lxcapi_startl;
	c->stop = lxcapi_stop;
	c->config_file_name = lxcapi_config_file_name;
	c->wait = lxcapi_wait;
	c->set_config_item = lxcapi_set_config_item;
	c->destroy = lxcapi_destroy;
	c->destroy_with_snapshots = lxcapi_destroy_with_snapshots;
	c->rename = lxcapi_rename;
	c->save_config = lxcapi_save_config;
	c->get_keys = lxcapi_get_keys;
	c->create = lxcapi_create;
	c->createl = lxcapi_createl;
	c->shutdown = lxcapi_shutdown;
	c->reboot = lxcapi_reboot;
	c->clear_config = lxcapi_clear_config;
	c->clear_config_item = lxcapi_clear_config_item;
	c->get_config_item = lxcapi_get_config_item;
	c->get_running_config_item = lxcapi_get_running_config_item;
	c->get_cgroup_item = lxcapi_get_cgroup_item;
	c->set_cgroup_item = lxcapi_set_cgroup_item;
	c->get_config_path = lxcapi_get_config_path;
	c->set_config_path = lxcapi_set_config_path;
	c->clone = lxcapi_clone;
	c->get_interfaces = lxcapi_get_interfaces;
	c->get_ips = lxcapi_get_ips;
	c->attach = lxcapi_attach;
	c->attach_run_wait = lxcapi_attach_run_wait;
	c->attach_run_waitl = lxcapi_attach_run_waitl;
	c->snapshot = lxcapi_snapshot;
	c->snapshot_list = lxcapi_snapshot_list;
	c->snapshot_restore = lxcapi_snapshot_restore;
	c->snapshot_destroy = lxcapi_snapshot_destroy;
	c->snapshot_destroy_all = lxcapi_snapshot_destroy_all;
	c->may_control = lxcapi_may_control;
	c->add_device_node = lxcapi_add_device_node;
	c->remove_device_node = lxcapi_remove_device_node;
	c->attach_interface = lxcapi_attach_interface;
	c->detach_interface = lxcapi_detach_interface;
	c->checkpoint = lxcapi_checkpoint;
	c->restore = lxcapi_restore;
	c->migrate = lxcapi_migrate;

	return c;

err:
	lxc_container_free(c);
	return NULL;
}
```

在这个函数中，为 `struct lxc_container` 结构分配内存，加载配置，并检查容器是不是创建了一半，如果是就移除创建了一半的容器。最最重要的还是初始化结构体的一大票函数指针。从这个函数中，我们可以非常清除地看到，`struct lxc_container` 结构的那些 API 函数，其实现是什么。如 `start()` 成员，其实现是 `lxcapi_start()` 函数。

前面的示例代码在编译时，需要链接 liblxc。

这些 C API 与相应的命令行工具作用相同。可以利用各个命令行工具，获得更多与 API 调用时发生的错误有关的信息。当容器创建成功时，也就是`lxc_container::createl()` 成功执行时，就可以通过命令行工具 `lxc-ls` 来查看创建的容器了，如：
```
$ sudo lxc-ls
apicontainer
```

通过 C API 创建的容器，同样可以用命令行工具来操作，如这样起动或停止容器：
```
$ sudo lxc-start --name apicontainer --logfile /home/hanpfei0306/lxc.log
$ sudo lxc-stop --name apicontainer --logfile /home/hanpfei0306/lxc.log
```

上面的示例代码，在执行之前，需要先创建名为 lxcbr0 的 br。创建方法如下：
```
$ sudo brctl show
[sudo] hanpfei0306 的密码： 
bridge name	bridge id		STP enabled	interfaces
anbox0		8000.000000000000	no		
virbr0		8000.000000000000	yes		
$ sudo brctl addbr lxcbr0
$ sudo brctl show
[sudo] hanpfei0306 的密码： 
bridge name	bridge id		STP enabled	interfaces
anbox0		8000.000000000000	no		
lxcbr0		8000.000000000000	no		
virbr0		8000.000000000000	yes		
```

否则，在启动容器时将失败。从命令行工具中可以看到如下的报错信息：
```
$ sudo lxc-start --name apicontainer --logfile /home/hanpfei0306/lxc.log
lxc-start: lxc_start.c: main: 344 The container failed to start.
lxc-start: lxc_start.c: main: 346 To get more details, run the container in foreground mode.
lxc-start: lxc_start.c: main: 348 Additional information can be obtained by setting the --logfile and --logpriority options.
```

在 lxc 的日志文件中可以看到这样的内容：
```
      lxc-start 20171204160718.133 ERROR    lxc_conf - conf.c:instantiate_veth:2594 - failed to attach 'vethUW1G7M' to the bridge 'lxcbr0': Operation not permitted
      lxc-start 20171204160718.159 ERROR    lxc_conf - conf.c:lxc_create_network:2871 - failed to create netdev
      lxc-start 20171204160718.159 ERROR    lxc_start - start.c:lxc_spawn:1066 - failed to create the network
      lxc-start 20171204160718.159 ERROR    lxc_start - start.c:__lxc_start:1329 - failed to spawn 'apicontainer'
```

### [打赏](https://www.wolfcstech.com/about/donate.html)

参考文档：
[Get started with lxc.](http://blog.csdn.net/dongsheng_yang/article/details/41832197)
[Linux 容器的建立和简单管理](https://www.ibm.com/developerworks/cn/linux/1312_caojh_linuxlxc/index.html)

Done.
