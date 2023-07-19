---
title: Linux 内核设备树和平台音频设备驱动程序
date: 2023-05-28 14:51:29
categories: Linux 内核
tags:
- Linux 内核
---

## 解析设备树

解析设备树，并将设备树保存在全局变量 `of_root` 中，这主要通过 `unflatten_device_tree()` 函数完成，该函数定义 (位于 `kernel/drivers/of/fdt.c`) 如下：
```
void __init unflatten_device_tree(void)
{
	__unflatten_device_tree(initial_boot_params, NULL, &of_root,
				early_init_dt_alloc_memory_arch, false);

	/* Get pointer to "/chosen" and "/aliases" nodes for use everywhere */
	of_alias_scan(early_init_dt_alloc_memory_arch);

	unittest_unflatten_overlay_base();
}
```

这最终将创建 `struct device_node` 的树形结构。`unflatten_device_tree()` 函数主要通过 `__unflatten_device_tree()` 函数来创建 `struct device_node` 树，`__unflatten_device_tree()` 函数定义 (位于 `kernel/drivers/of/fdt.c`) 如下：
```
static bool of_fdt_device_is_available(const void *blob, unsigned long node)
{
	const char *status = fdt_getprop(blob, node, "status", NULL);

	if (!status)
		return true;

	if (!strcmp(status, "ok") || !strcmp(status, "okay"))
		return true;

	return false;
}

static void *unflatten_dt_alloc(void **mem, unsigned long size,
				       unsigned long align)
{
	void *res;

	*mem = PTR_ALIGN(*mem, align);
	res = *mem;
	*mem += size;

	return res;
}

static void populate_properties(const void *blob,
				int offset,
				void **mem,
				struct device_node *np,
				const char *nodename,
				bool dryrun)
{
	struct property *pp, **pprev = NULL;
	int cur;
	bool has_name = false;

	pprev = &np->properties;
	for (cur = fdt_first_property_offset(blob, offset);
	     cur >= 0;
	     cur = fdt_next_property_offset(blob, cur)) {
		const __be32 *val;
		const char *pname;
		u32 sz;

		val = fdt_getprop_by_offset(blob, cur, &pname, &sz);
		if (!val) {
			pr_warn("Cannot locate property at 0x%x\n", cur);
			continue;
		}

		if (!pname) {
			pr_warn("Cannot find property name at 0x%x\n", cur);
			continue;
		}

		if (!strcmp(pname, "name"))
			has_name = true;

		pp = unflatten_dt_alloc(mem, sizeof(struct property),
					__alignof__(struct property));
		if (dryrun)
			continue;

		/* We accept flattened tree phandles either in
		 * ePAPR-style "phandle" properties, or the
		 * legacy "linux,phandle" properties.  If both
		 * appear and have different values, things
		 * will get weird. Don't do that.
		 */
		if (!strcmp(pname, "phandle") ||
		    !strcmp(pname, "linux,phandle")) {
			if (!np->phandle)
				np->phandle = be32_to_cpup(val);
		}

		/* And we process the "ibm,phandle" property
		 * used in pSeries dynamic device tree
		 * stuff
		 */
		if (!strcmp(pname, "ibm,phandle"))
			np->phandle = be32_to_cpup(val);

		pp->name   = (char *)pname;
		pp->length = sz;
		pp->value  = (__be32 *)val;
		*pprev     = pp;
		pprev      = &pp->next;
	}

	/* With version 0x10 we may not have the name property,
	 * recreate it here from the unit name if absent
	 */
	if (!has_name) {
		const char *p = nodename, *ps = p, *pa = NULL;
		int len;

		while (*p) {
			if ((*p) == '@')
				pa = p;
			else if ((*p) == '/')
				ps = p + 1;
			p++;
		}

		if (pa < ps)
			pa = p;
		len = (pa - ps) + 1;
		pp = unflatten_dt_alloc(mem, sizeof(struct property) + len,
					__alignof__(struct property));
		if (!dryrun) {
			pp->name   = "name";
			pp->length = len;
			pp->value  = pp + 1;
			*pprev     = pp;
			pprev      = &pp->next;
			memcpy(pp->value, ps, len - 1);
			((char *)pp->value)[len - 1] = 0;
			pr_debug("fixed up name for %s -> %s\n",
				 nodename, (char *)pp->value);
		}
	}

	if (!dryrun)
		*pprev = NULL;
}

static bool populate_node(const void *blob,
			  int offset,
			  void **mem,
			  struct device_node *dad,
			  struct device_node **pnp,
			  bool dryrun)
{
	struct device_node *np;
	const char *pathp;
	unsigned int l, allocl;

	pathp = fdt_get_name(blob, offset, &l);
	if (!pathp) {
		*pnp = NULL;
		return false;
	}

	allocl = ++l;

	np = unflatten_dt_alloc(mem, sizeof(struct device_node) + allocl,
				__alignof__(struct device_node));
	if (!dryrun) {
		char *fn;
		of_node_init(np);
		np->full_name = fn = ((char *)np) + sizeof(*np);

		memcpy(fn, pathp, l);

		if (dad != NULL) {
			np->parent = dad;
			np->sibling = dad->child;
			dad->child = np;
		}
	}

	populate_properties(blob, offset, mem, np, pathp, dryrun);
	if (!dryrun) {
		np->name = of_get_property(np, "name", NULL);
		if (!np->name)
			np->name = "<NULL>";
	}

	*pnp = np;
	return true;
}
. . . . . .
/**
 * unflatten_dt_nodes - Alloc and populate a device_node from the flat tree
 * @blob: The parent device tree blob
 * @mem: Memory chunk to use for allocating device nodes and properties
 * @dad: Parent struct device_node
 * @nodepp: The device_node tree created by the call
 *
 * It returns the size of unflattened device tree or error code
 */
static int unflatten_dt_nodes(const void *blob,
			      void *mem,
			      struct device_node *dad,
			      struct device_node **nodepp)
{
	struct device_node *root;
	int offset = 0, depth = 0, initial_depth = 0;
#define FDT_MAX_DEPTH	64
	struct device_node *nps[FDT_MAX_DEPTH];
	void *base = mem;
	bool dryrun = !base;

	if (nodepp)
		*nodepp = NULL;

	/*
	 * We're unflattening device sub-tree if @dad is valid. There are
	 * possibly multiple nodes in the first level of depth. We need
	 * set @depth to 1 to make fdt_next_node() happy as it bails
	 * immediately when negative @depth is found. Otherwise, the device
	 * nodes except the first one won't be unflattened successfully.
	 */
	if (dad)
		depth = initial_depth = 1;

	root = dad;
	nps[depth] = dad;

	for (offset = 0;
	     offset >= 0 && depth >= initial_depth;
	     offset = fdt_next_node(blob, offset, &depth)) {
		if (WARN_ON_ONCE(depth >= FDT_MAX_DEPTH))
			continue;

		if (!IS_ENABLED(CONFIG_OF_KOBJ) &&
		    !of_fdt_device_is_available(blob, offset))
			continue;

		if (!populate_node(blob, offset, &mem, nps[depth],
				   &nps[depth+1], dryrun))
			return mem - base;

		if (!dryrun && nodepp && !*nodepp)
			*nodepp = nps[depth+1];
		if (!dryrun && !root)
			root = nps[depth+1];
	}

	if (offset < 0 && offset != -FDT_ERR_NOTFOUND) {
		pr_err("Error %d processing FDT\n", offset);
		return -EINVAL;
	}

	/*
	 * Reverse the child list. Some drivers assumes node order matches .dts
	 * node order
	 */
	if (!dryrun)
		reverse_nodes(root);

	return mem - base;
}

/**
 * __unflatten_device_tree - create tree of device_nodes from flat blob
 *
 * unflattens a device-tree, creating the
 * tree of struct device_node. It also fills the "name" and "type"
 * pointers of the nodes so the normal device-tree walking functions
 * can be used.
 * @blob: The blob to expand
 * @dad: Parent device node
 * @mynodes: The device_node tree created by the call
 * @dt_alloc: An allocator that provides a virtual address to memory
 * for the resulting tree
 * @detached: if true set OF_DETACHED on @mynodes
 *
 * Returns NULL on failure or the memory chunk containing the unflattened
 * device tree on success.
 */
void *__unflatten_device_tree(const void *blob,
			      struct device_node *dad,
			      struct device_node **mynodes,
			      void *(*dt_alloc)(u64 size, u64 align),
			      bool detached)
{
	int size;
	void *mem;

	pr_debug(" -> unflatten_device_tree()\n");

	if (!blob) {
		pr_debug("No device tree pointer\n");
		return NULL;
	}

	pr_debug("Unflattening device tree:\n");
	pr_debug("magic: %08x\n", fdt_magic(blob));
	pr_debug("size: %08x\n", fdt_totalsize(blob));
	pr_debug("version: %08x\n", fdt_version(blob));

	if (fdt_check_header(blob)) {
		pr_err("Invalid device tree blob header\n");
		return NULL;
	}

	/* First pass, scan for size */
	size = unflatten_dt_nodes(blob, NULL, dad, NULL);
	if (size < 0)
		return NULL;

	size = ALIGN(size, 4);
	pr_debug("  size is %d, allocating...\n", size);

	/* Allocate memory for the expanded device tree */
	mem = dt_alloc(size + 4, __alignof__(struct device_node));
	if (!mem)
		return NULL;

	memset(mem, 0, size);

	*(__be32 *)(mem + size) = cpu_to_be32(0xdeadbeef);

	pr_debug("  unflattening %p...\n", mem);

	/* Second pass, do actual unflattening */
	unflatten_dt_nodes(blob, mem, dad, mynodes);
	if (be32_to_cpup(mem + size) != 0xdeadbeef)
		pr_warn("End of tree marker overwritten: %08x\n",
			be32_to_cpup(mem + size));

	if (detached && mynodes) {
		of_node_set_flag(*mynodes, OF_DETACHED);
		pr_debug("unflattened tree is detached\n");
	}

	pr_debug(" <- unflatten_device_tree()\n");
	return mem;
}
```

在创建 `struct device_node` 树的过程中，会检查设备树中节点的状态，即其 **"status"** 属性的值，如果这个属性值不是 **"ok"** 或 **"okay"** (大小写敏感)，则不会为该设备树节点创建 `struct device_node` 对象。

在 `populate_node()` 函数中，为 `struct device_node` 对象分配内存空间，并解析设备树节点的属性。

## 平台设备驱动程序核心初始化

平台设备驱动程序核心初始化时，会为从设备树解析获得的设备节点创建平台设备对象，并将这些设备挂在平台总线类型上，这个过程大体如下：
```
[    5.138804] device_add+0x180/0x970
[    5.143238] of_device_add+0x3c/0x50
[    5.147742] of_platform_device_create_pdata+0xa8/0x108
[    5.154072] of_platform_bus_create+0x198/0x21c
[    5.159615] of_platform_populate+0x78/0xd8
[    5.164804] of_platform_default_populate_init+0xb0/0xd0
[    5.171204] do_one_initcall+0x98/0x2dc
[    5.176046] do_initcall_level+0xa8/0x160
[    5.181078] do_initcalls+0x58/0x9c
[    5.185555] do_basic_setup+0x28/0x38
[    5.190188] kernel_init_freeable+0xf4/0x16c
[    5.195506] kernel_init+0x18/0x190
[    5.199913] ret_from_fork+0x10/0x30
```

这个过程的入口点是 `of_platform_default_populate_init()`，该函数定义 (位于 `kernel/drivers/of/platform.c`) 如下：
```
/**
 * of_platform_bus_create() - Create a device for a node and its children.
 * @bus: device node of the bus to instantiate
 * @matches: match table for bus nodes
 * @lookup: auxdata table for matching id and platform_data with device nodes
 * @parent: parent for new device, or NULL for top level.
 * @strict: require compatible property
 *
 * Creates a platform_device for the provided device_node, and optionally
 * recursively create devices for all the child nodes.
 */
static int of_platform_bus_create(struct device_node *bus,
				  const struct of_device_id *matches,
				  const struct of_dev_auxdata *lookup,
				  struct device *parent, bool strict)
{
	const struct of_dev_auxdata *auxdata;
	struct device_node *child;
	struct platform_device *dev;
	const char *bus_id = NULL;
	void *platform_data = NULL;
	int rc = 0;

	/* Make sure it has a compatible property */
	if (strict && (!of_get_property(bus, "compatible", NULL))) {
		pr_debug("%s() - skipping %pOF, no compatible prop\n",
			 __func__, bus);
		return 0;
	}

	/* Skip nodes for which we don't want to create devices */
	if (unlikely(of_match_node(of_skipped_node_table, bus))) {
		pr_debug("%s() - skipping %pOF node\n", __func__, bus);
		return 0;
	}

	if (of_node_check_flag(bus, OF_POPULATED_BUS)) {
		pr_debug("%s() - skipping %pOF, already populated\n",
			__func__, bus);
		return 0;
	}

	auxdata = of_dev_lookup(lookup, bus);
	if (auxdata) {
		bus_id = auxdata->name;
		platform_data = auxdata->platform_data;
	}

	if (of_device_is_compatible(bus, "arm,primecell")) {
		/*
		 * Don't return an error here to keep compatibility with older
		 * device tree files.
		 */
		of_amba_device_create(bus, bus_id, platform_data, parent);
		return 0;
	}

	dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
	if (!dev || !of_match_node(matches, bus))
		return 0;

	for_each_child_of_node(bus, child) {
		pr_debug("   create child: %pOF\n", child);
		rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	of_node_set_flag(bus, OF_POPULATED_BUS);
	return rc;
}
 . . . . . .
/**
 * of_platform_populate() - Populate platform_devices from device tree data
 * @root: parent of the first level to probe or NULL for the root of the tree
 * @matches: match table, NULL to use the default
 * @lookup: auxdata table for matching id and platform_data with device nodes
 * @parent: parent to hook devices from, NULL for toplevel
 *
 * Similar to of_platform_bus_probe(), this function walks the device tree
 * and creates devices from nodes.  It differs in that it follows the modern
 * convention of requiring all device nodes to have a 'compatible' property,
 * and it is suitable for creating devices which are children of the root
 * node (of_platform_bus_probe will only create children of the root which
 * are selected by the @matches argument).
 *
 * New board support should be using this function instead of
 * of_platform_bus_probe().
 *
 * Returns 0 on success, < 0 on failure.
 */
int of_platform_populate(struct device_node *root,
			const struct of_device_id *matches,
			const struct of_dev_auxdata *lookup,
			struct device *parent)
{
	struct device_node *child;
	int rc = 0;

	root = root ? of_node_get(root) : of_find_node_by_path("/");
	if (!root)
		return -EINVAL;

	pr_debug("%s()\n", __func__);
	pr_debug(" starting at: %pOF\n", root);

	device_links_supplier_sync_state_pause();
	for_each_child_of_node(root, child) {
		rc = of_platform_bus_create(child, matches, lookup, parent, true);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	device_links_supplier_sync_state_resume();

	of_node_set_flag(root, OF_POPULATED_BUS);

	of_node_put(root);
	return rc;
}
EXPORT_SYMBOL_GPL(of_platform_populate);

int of_platform_default_populate(struct device_node *root,
				 const struct of_dev_auxdata *lookup,
				 struct device *parent)
{
	return of_platform_populate(root, of_default_bus_match_table, lookup,
				    parent);
}
EXPORT_SYMBOL_GPL(of_platform_default_populate);

#ifndef CONFIG_PPC
static const struct of_device_id reserved_mem_matches[] = {
	{ .compatible = "qcom,rmtfs-mem" },
	{ .compatible = "qcom,cmd-db" },
	{ .compatible = "ramoops" },
	{}
};

static int __init of_platform_default_populate_init(void)
{
	struct device_node *node;

	device_links_supplier_sync_state_pause();

	if (!of_have_populated_dt())
		return -ENODEV;

	/*
	 * Handle certain compatibles explicitly, since we don't want to create
	 * platform_devices for every node in /reserved-memory with a
	 * "compatible",
	 */
	for_each_matching_node(node, reserved_mem_matches)
		of_platform_device_create(node, NULL, NULL);

	node = of_find_node_by_path("/firmware");
	if (node) {
		of_platform_populate(node, NULL, NULL, NULL);
		of_node_put(node);
	}

	/* Populate everything else. */
	of_platform_default_populate(NULL, NULL, NULL);

	return 0;
}
arch_initcall_sync(of_platform_default_populate_init);
```

这个过程中，在 `of_platform_bus_create()` 函数中，会检查设备树节点的属性，只有设备树节点具有 **"compatible"** 属性时，才会为它创建平台设备对象 `struct platform_device`。

`of_platform_bus_create()` 函数通过 `of_platform_device_create_pdata()` 函数分配并初始化 `struct platform_device` 对象，并把平台设备添加进平台总线的设备列表，`of_platform_device_create_pdata()` 函数定义 (位于 `kernel/drivers/of/platform.c`) 如下：
```
struct platform_device *of_device_alloc(struct device_node *np,
				  const char *bus_id,
				  struct device *parent)
{
	struct platform_device *dev;
	int rc, i, num_reg = 0, num_irq;
	struct resource *res, temp_res;

	dev = platform_device_alloc("", PLATFORM_DEVID_NONE);
	if (!dev)
		return NULL;

	/* count the io and irq resources */
	while (of_address_to_resource(np, num_reg, &temp_res) == 0)
		num_reg++;
	num_irq = of_irq_count(np);

	/* Populate the resource table */
	if (num_irq || num_reg) {
		res = kcalloc(num_irq + num_reg, sizeof(*res), GFP_KERNEL);
		if (!res) {
			platform_device_put(dev);
			return NULL;
		}

		dev->num_resources = num_reg + num_irq;
		dev->resource = res;
		for (i = 0; i < num_reg; i++, res++) {
			rc = of_address_to_resource(np, i, res);
			WARN_ON(rc);
		}
		if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
			pr_debug("not all legacy IRQ resources mapped for %pOFn\n",
				 np);
	}

	dev->dev.of_node = of_node_get(np);
	dev->dev.fwnode = &np->fwnode;
	dev->dev.parent = parent ? : &platform_bus;

	if (bus_id)
		dev_set_name(&dev->dev, "%s", bus_id);
	else
		of_device_make_bus_id(&dev->dev);

	return dev;
}
EXPORT_SYMBOL(of_device_alloc);

/**
 * of_platform_device_create_pdata - Alloc, initialize and register an of_device
 * @np: pointer to node to create device for
 * @bus_id: name to assign device
 * @platform_data: pointer to populate platform_data pointer with
 * @parent: Linux device model parent device.
 *
 * Returns pointer to created platform device, or NULL if a device was not
 * registered.  Unavailable devices will not get registered.
 */
static struct platform_device *of_platform_device_create_pdata(
					struct device_node *np,
					const char *bus_id,
					void *platform_data,
					struct device *parent)
{
	struct platform_device *dev;

	if (!of_device_is_available(np) ||
	    of_node_test_and_set_flag(np, OF_POPULATED))
		return NULL;

	dev = of_device_alloc(np, bus_id, parent);
	if (!dev)
		goto err_clear_flag;

	dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);
	if (!dev->dev.dma_mask)
		dev->dev.dma_mask = &dev->dev.coherent_dma_mask;
	dev->dev.bus = &platform_bus_type;
	dev->dev.platform_data = platform_data;
	of_msi_configure(&dev->dev, dev->dev.of_node);

	if (of_device_add(dev) != 0) {
		platform_device_put(dev);
		goto err_clear_flag;
	}

	return dev;

err_clear_flag:
	of_node_clear_flag(np, OF_POPULATED);
	return NULL;
}
```

在 `of_device_alloc()` 函数中，在分配 `struct platform_device` 对象外，它还会初始化设备节点的资源，这主要包括内存映射的寄存器资源，和中断资源。

`of_platform_device_create_pdata()` 函数调用 `of_device_add()` 函数添加设备，这个函数定义 (位于 `kernel/drivers/of/device.c`) 如下：
```
int of_device_add(struct platform_device *ofdev)
{
	BUG_ON(ofdev->dev.of_node == NULL);

	/* name and id have to be set so that the platform bus doesn't get
	 * confused on matching */
	ofdev->name = dev_name(&ofdev->dev);
	ofdev->id = PLATFORM_DEVID_NONE;

	/*
	 * If this device has not binding numa node in devicetree, that is
	 * of_node_to_nid returns NUMA_NO_NODE. device_add will assume that this
	 * device is on the same node as the parent.
	 */
	set_dev_node(&ofdev->dev, of_node_to_nid(ofdev->dev.of_node));

	return device_add(&ofdev->dev);
}
```

平台设备不仅仅可以通过解析设备树获得，但大多数情况下都是通过解析设备树获得的。

具体的平台设备驱动程序在向平台总线类型注册时，会通过设备 ID 的 `compatible` 字段和平台总线类型上挂的平台设备的 **"compatible"** 属性匹配，匹配成功时，则执行设备驱动程序的 `probe()` 函数。以 Rockchip I2S TDM 平台设备驱动程序为例，这个过程大体如下：
```
[    8.886192] rockchip_i2s_tdm_probe+0x44/0x7a0
[    8.886202] platform_drv_probe+0x9c/0xc4
[    8.886211] really_probe+0x204/0x510
[    8.886218] driver_probe_device+0x80/0xc0
[    8.886227] device_driver_attach+0x70/0xb4
[    8.886235] __driver_attach+0xc8/0x150
[    8.886242] bus_for_each_dev+0x80/0xd0
[    8.886250] driver_attach+0x28/0x38
[    8.886257] bus_add_driver+0x108/0x1e8
[    8.886265] driver_register+0x7c/0x118
[    8.886274] __platform_driver_register+0x48/0x58
[    8.886284] rockchip_i2s_tdm_driver_init+0x20/0x30
[    8.886291] do_one_initcall+0x98/0x2dc
[    8.886301] do_initcall_level+0xa8/0x160
[    8.886309] do_initcalls+0x58/0x9c
[    8.886318] do_basic_setup+0x28/0x38
[    8.886327] kernel_init_freeable+0xf4/0x16c
[    8.886336] kernel_init+0x18/0x190
[    8.886344] ret_from_fork+0x10/0x30
```

相同的驱动程序可能匹配到多个平台设备，对于每个匹配到的平台设备，`probe()` 函数都会被调用。

## 音频设备驱动程序

音频设备驱动程序会向用户空间导出多个设备文件。

#### ALSA 核心初始化
Linux 内核中 ALSA 核心初始化时，即会注册 timer 设备，向用户空间导出 timer 设备文件，这个过程大体如下：
```
[    8.882015] snd_register_device+0x40/0x194
[    8.882026] alsa_timer_init+0x1f8/0x244
[    8.882033] do_one_initcall+0x98/0x2dc
[    8.882044] do_initcall_level+0xa8/0x160
[    8.882052] do_initcalls+0x58/0x9c
[    8.882061] do_basic_setup+0x28/0x38
[    8.882069] kernel_init_freeable+0xf4/0x16c
[    8.882079] kernel_init+0x18/0x190
[    8.882086] ret_from_fork+0x10/0x30
```

#### 声卡驱动注册

I2S 一般是 ASoC 中的 CPU DAI 设备。声卡驱动程序，也就是 ASoC 的 machine 驱动程序，同样是平台设备驱动程序，它负责把 I2S 和 Codec 等设备驱动程序，连接进 ALSA 和 ASoC 框架，向用户空间导出适当的设备文件，将设备文件的用户空间操作和驱动程序的回调连接起来。

当定义了匹配的声卡设备时，声卡驱动程序的 `probe()` 函数被调用。

1. 创建 Control 音频设备

```
[    8.890096] snd_device_new+0x54/0x138
[    8.890106] snd_ctl_create+0x64/0x94
[    8.890114] snd_card_new+0x2c4/0x394
[    8.890122] snd_soc_bind_card+0x354/0xad0
[    8.890130] snd_soc_register_card+0xf8/0x114
[    8.890138] devm_snd_soc_register_card+0x48/0x90
[    8.890147] rk_hdmi_probe+0x394/0x400
[    8.890156] platform_drv_probe+0x9c/0xc4
[    8.890164] really_probe+0x204/0x510
[    8.890172] driver_probe_device+0x80/0xc0
[    8.890180] device_driver_attach+0x70/0xb4
[    8.890188] __driver_attach+0xc8/0x150
[    8.890195] bus_for_each_dev+0x80/0xd0
[    8.890203] driver_attach+0x28/0x38
[    8.890210] bus_add_driver+0x108/0x1e8
[    8.890218] driver_register+0x7c/0x118
[    8.890226] __platform_driver_register+0x48/0x58
[    8.890236] rockchip_sound_driver_init+0x20/0x30
[    8.890243] do_one_initcall+0x98/0x2dc
[    8.890252] do_initcall_level+0xa8/0x160
[    8.890260] do_initcalls+0x58/0x9c
[    8.890268] do_basic_setup+0x28/0x38
[    8.890277] kernel_init_freeable+0xf4/0x16c
[    8.890286] kernel_init+0x18/0x190
[    8.890293] ret_from_fork+0x10/0x30
```

2. 创建 Jack 音频设备
```
[    8.890718] snd_device_new+0x54/0x138
[    8.890725] snd_jack_new+0x124/0x22c
[    8.890733] snd_soc_card_jack_new+0xa0/0x118
[    8.890741] rk_dailink_init+0xdc/0x140
[    8.890749] snd_soc_link_init+0x28/0x84
[    8.890757] snd_soc_bind_card+0x6b4/0xad0
[    8.890765] snd_soc_register_card+0xf8/0x114
[    8.890773] devm_snd_soc_register_card+0x48/0x90
[    8.890780] rk_hdmi_probe+0x394/0x400
[    8.890789] platform_drv_probe+0x9c/0xc4
[    8.890797] really_probe+0x204/0x510
[    8.890805] driver_probe_device+0x80/0xc0
[    8.890813] device_driver_attach+0x70/0xb4
[    8.890821] __driver_attach+0xc8/0x150
[    8.890828] bus_for_each_dev+0x80/0xd0
[    8.890836] driver_attach+0x28/0x38
[    8.890843] bus_add_driver+0x108/0x1e8
[    8.890851] driver_register+0x7c/0x118
[    8.890859] __platform_driver_register+0x48/0x58
[    8.890868] rockchip_sound_driver_init+0x20/0x30
[    8.890875] do_one_initcall+0x98/0x2dc
[    8.890884] do_initcall_level+0xa8/0x160
[    8.890892] do_initcalls+0x58/0x9c
[    8.890900] do_basic_setup+0x28/0x38
[    8.890908] kernel_init_freeable+0xf4/0x16c
[    8.890917] kernel_init+0x18/0x190
[    8.890924] ret_from_fork+0x10/0x30
```

3. 创建 PCM 音频设备
```
[    8.891016] snd_device_new+0x54/0x138
[    8.891024] _snd_pcm_new+0x114/0x190
[    8.891031] snd_pcm_new+0x1c/0x2c
[    8.891039] soc_new_pcm+0x4b0/0x560
[    8.891046] snd_soc_bind_card+0x750/0xad0
[    8.891053] snd_soc_register_card+0xf8/0x114
[    8.891062] devm_snd_soc_register_card+0x48/0x90
[    8.891069] rk_hdmi_probe+0x394/0x400
[    8.891078] platform_drv_probe+0x9c/0xc4
[    8.891086] really_probe+0x204/0x510
[    8.891094] driver_probe_device+0x80/0xc0
[    8.891102] device_driver_attach+0x70/0xb4
[    8.891109] __driver_attach+0xc8/0x150
[    8.891117] bus_for_each_dev+0x80/0xd0
[    8.891124] driver_attach+0x28/0x38
[    8.891131] bus_add_driver+0x108/0x1e8
[    8.891139] driver_register+0x7c/0x118
[    8.891148] __platform_driver_register+0x48/0x58
[    8.891156] rockchip_sound_driver_init+0x20/0x30
[    8.891163] do_one_initcall+0x98/0x2dc
[    8.891172] do_initcall_level+0xa8/0x160
[    8.891180] do_initcalls+0x58/0x9c
[    8.891188] do_basic_setup+0x28/0x38
[    8.891196] kernel_init_freeable+0xf4/0x16c
[    8.891205] kernel_init+0x18/0x190
[    8.891212] ret_from_fork+0x10/0x30
```

4. 立即创建 PCM 设备文件
```
[    8.892003] snd_register_device+0x40/0x194
[    8.892010] snd_pcm_dev_register+0x60/0x1e0
[    8.892017] snd_device_register_all+0x60/0x8c
[    8.892023] snd_card_register+0x50/0x184
[    8.892031] snd_soc_bind_card+0x930/0xad0
[    8.892036] snd_soc_register_card+0xf8/0x114
[    8.892043] devm_snd_soc_register_card+0x48/0x90
[    8.892050] rk_hdmi_probe+0x394/0x400
[    8.892058] platform_drv_probe+0x9c/0xc4
[    8.892064] really_probe+0x204/0x510
[    8.892070] driver_probe_device+0x80/0xc0
[    8.892076] device_driver_attach+0x70/0xb4
[    8.892082] __driver_attach+0xc8/0x150
[    8.892088] bus_for_each_dev+0x80/0xd0
[    8.892094] driver_attach+0x28/0x38
[    8.892099] bus_add_driver+0x108/0x1e8
[    8.892105] driver_register+0x7c/0x118
[    8.892112] __platform_driver_register+0x48/0x58
[    8.892120] rockchip_sound_driver_init+0x20/0x30
[    8.892125] do_one_initcall+0x98/0x2dc
[    8.892133] do_initcall_level+0xa8/0x160
[    8.892139] do_initcalls+0x58/0x9c
[    8.892145] do_basic_setup+0x28/0x38
[    8.892152] kernel_init_freeable+0xf4/0x16c
[    8.892158] kernel_init+0x18/0x190
[    8.892164] ret_from_fork+0x10/0x30
```

5. 继续创建 Timer 音频设备
```
[    8.892402] snd_device_new+0x54/0x138
[    8.892408] snd_timer_new+0x120/0x17c
[    8.892414] snd_pcm_timer_init+0x6c/0x140
[    8.892421] snd_pcm_dev_register+0x78/0x1e0
[    8.892427] snd_device_register_all+0x60/0x8c
[    8.892434] snd_card_register+0x50/0x184
[    8.892441] snd_soc_bind_card+0x930/0xad0
[    8.892446] snd_soc_register_card+0xf8/0x114
[    8.892453] devm_snd_soc_register_card+0x48/0x90
[    8.892460] rk_hdmi_probe+0x394/0x400
[    8.892467] platform_drv_probe+0x9c/0xc4
[    8.892474] really_probe+0x204/0x510
[    8.892479] driver_probe_device+0x80/0xc0
[    8.892486] device_driver_attach+0x70/0xb4
[    8.892491] __driver_attach+0xc8/0x150
[    8.892497] bus_for_each_dev+0x80/0xd0
[    8.892503] driver_attach+0x28/0x38
[    8.892508] bus_add_driver+0x108/0x1e8
[    8.892514] driver_register+0x7c/0x118
[    8.892521] __platform_driver_register+0x48/0x58
[    8.892528] rockchip_sound_driver_init+0x20/0x30
[    8.892535] do_one_initcall+0x98/0x2dc
[    8.892542] do_initcall_level+0xa8/0x160
[    8.892549] do_initcalls+0x58/0x9c
[    8.892555] do_basic_setup+0x28/0x38
[    8.892561] kernel_init_freeable+0xf4/0x16c
[    8.892568] kernel_init+0x18/0x190
[    8.892574] ret_from_fork+0x10/0x30
```

6. 创建 Control 设备文件
```
[    8.892897] snd_register_device+0x40/0x194
[    8.892904] snd_ctl_dev_register+0x30/0x40
[    8.892911] snd_device_register_all+0x60/0x8c
[    8.892917] snd_card_register+0x50/0x184
[    8.892924] snd_soc_bind_card+0x930/0xad0
[    8.892930] snd_soc_register_card+0xf8/0x114
[    8.892937] devm_snd_soc_register_card+0x48/0x90
[    8.892944] rk_hdmi_probe+0x394/0x400
[    8.892952] platform_drv_probe+0x9c/0xc4
[    8.892958] really_probe+0x204/0x510
[    8.892964] driver_probe_device+0x80/0xc0
[    8.892970] device_driver_attach+0x70/0xb4
[    8.892976] __driver_attach+0xc8/0x150
[    8.892981] bus_for_each_dev+0x80/0xd0
[    8.892987] driver_attach+0x28/0x38
[    8.892993] bus_add_driver+0x108/0x1e8
[    8.892999] driver_register+0x7c/0x118
[    8.893005] __platform_driver_register+0x48/0x58
[    8.893013] rockchip_sound_driver_init+0x20/0x30
[    8.893019] do_one_initcall+0x98/0x2dc
[    8.893027] do_initcall_level+0xa8/0x160
[    8.893033] do_initcalls+0x58/0x9c
[    8.893051] do_basic_setup+0x28/0x38
[    8.893057] kernel_init_freeable+0xf4/0x16c
[    8.893064] kernel_init+0x18/0x190
[    8.893070] ret_from_fork+0x10/0x30
```

上面的这些过程是 `compatible` 为 **"rockchip,hdmi"** 的驱动程序触发的。这个声卡设备的设备树节点如下：
```
	hdmi0_sound: hdmi0-sound {
		status = "okay";
		compatible = "rockchip,hdmi";
		rockchip,mclk-fs = <128>;
		rockchip,card-name = "rockchip-hdmi0";
		rockchip,cpu = <&i2s5_8ch>;
		rockchip,codec = <&hdmi0>;
		rockchip,jack-det;
	};

	dp0_sound: dp0-sound {
		status = "okay";
		compatible = "rockchip,hdmi";
		rockchip,card-name= "rockchip,dp0";
		rockchip,mclk-fs = <512>;
		rockchip,cpu = <&spdif_tx2>;
		rockchip,codec = <&dp0 1>;
		rockchip,jack-det;
	};

	dp1_sound: dp1-sound {
		status = "okay";
		compatible = "rockchip,hdmi";
		rockchip,card-name= "rockchip,dp1";
		rockchip,mclk-fs = <512>;
		rockchip,cpu = <&spdif_tx5>;
		rockchip,codec = <&dp1 1>;
		rockchip,jack-det;
	};
```

下面是另一个声卡驱动的执行过程，其 `compatible` 为 **"rockchip,multicodecs-card"**，设备树节点如下：
```
	nau8822_sound: nau8822-sound {
		status = "okay";
		compatible = "rockchip,multicodecs-card";
		rockchip,card-name = "rockchip-nau8822";
		hp-det-gpio = <&gpio1 RK_PB2 GPIO_ACTIVE_HIGH>;
		io-channels = <&saradc 3>;
		io-channel-names = "adc-detect";
		keyup-threshold-microvolt = <1800000>;
		poll-interval = <100>;
//		spk-con-gpio = <&gpio1 RK_PD3 GPIO_ACTIVE_HIGH>;
//		hp-con-gpio = <&gpio1 RK_PD2 GPIO_ACTIVE_HIGH>;
		rockchip,format = "i2s";
		rockchip,mclk-fs = <256>;
		rockchip,cpu = <&i2s0_8ch>;
		rockchip,codec = <&nau8822>;
		rockchip,audio-routing =
			"Headphone", "RHP",
			"Headphone", "LHP",
			"Speaker", "LSPK",
			"Speaker", "RSPK",
			"RMICP", "Main Mic",
			"RMICN", "Main Mic",
			"RMICP", "Headset Mic",
			"RMICN", "Headset Mic",
			"LMICP", "Main Mic",
			"LMICN", "Main Mic",
			"LMICP", "Headset Mic",
			"LMICN", "Headset Mic";
		pinctrl-names = "default";
		pinctrl-0 = <&hp_det>;
		play-pause-key {
			label = "playpause";
			linux,code = <KEY_PLAYPAUSE>;
			press-threshold-microvolt = <2000>;
			poll-interval = <1000>;
		};
	};
```


1. 创建 Control 音频设备
```
[    8.893949] snd_device_new+0x54/0x138
[    8.893956] snd_ctl_create+0x64/0x94
[    8.893962] snd_card_new+0x2c4/0x394
[    8.893968] snd_soc_bind_card+0x354/0xad0
[    8.893974] snd_soc_register_card+0xf8/0x114
[    8.893981] devm_snd_soc_register_card+0x48/0x90
[    8.893987] rk_multicodecs_probe+0x86c/0x970
[    8.893995] platform_drv_probe+0x9c/0xc4
[    8.894001] really_probe+0x204/0x510
[    8.894007] driver_probe_device+0x80/0xc0
[    8.894013] device_driver_attach+0x70/0xb4
[    8.894019] __driver_attach+0xc8/0x150
[    8.894025] bus_for_each_dev+0x80/0xd0
[    8.894030] driver_attach+0x28/0x38
[    8.894036] bus_add_driver+0x108/0x1e8
[    8.894042] driver_register+0x7c/0x118
[    8.894049] __platform_driver_register+0x48/0x58
[    8.894056] rockchip_multicodecs_driver_init+0x20/0x30
[    8.894062] do_one_initcall+0x98/0x2dc
```

2. I2S 驱动程序注册的 DAI 驱动程序的 porbe 函数被调用
```
[    8.896726] rockchip_i2s_tdm_dai_probe+0x24/0x84
[    8.896732] snd_soc_pcm_dai_probe+0xb0/0xec
[    8.896738] snd_soc_bind_card+0x5d8/0xad0
[    8.896744] snd_soc_register_card+0xf8/0x114
[    8.896751] devm_snd_soc_register_card+0x48/0x90
[    8.896757] rk_multicodecs_probe+0x86c/0x970
[    8.896765] platform_drv_probe+0x9c/0xc4
[    8.896771] really_probe+0x204/0x510
[    8.896777] driver_probe_device+0x80/0xc0
[    8.896784] device_driver_attach+0x70/0xb4
[    8.896789] __driver_attach+0xc8/0x150
[    8.896795] bus_for_each_dev+0x80/0xd0
[    8.896801] driver_attach+0x28/0x38
[    8.896806] bus_add_driver+0x108/0x1e8
[    8.896812] driver_register+0x7c/0x118
[    8.896819] __platform_driver_register+0x48/0x58
[    8.896827] rockchip_multicodecs_driver_init+0x20/0x30
[    8.896833] do_one_initcall+0x98/0x2dc
[    8.896840] do_initcall_level+0xa8/0x160
[    8.896846] do_initcalls+0x58/0x9c
[    8.896852] do_basic_setup+0x28/0x38
[    8.896859] kernel_init_freeable+0xf4/0x16c
[    8.896865] kernel_init+0x18/0x190
[    8.896871] ret_from_fork+0x10/0x30
```

3. 创建 Jack 音频设备
```
[    8.896932] snd_device_new+0x54/0x138
[    8.896937] snd_jack_new+0x124/0x22c
[    8.896943] snd_soc_card_jack_new+0xa0/0x118
[    8.896950] rk_dailink_init+0x64/0x160
[    8.896956] snd_soc_link_init+0x28/0x84
[    8.896962] snd_soc_bind_card+0x6b4/0xad0
[    8.896968] snd_soc_register_card+0xf8/0x114
[    8.896974] devm_snd_soc_register_card+0x48/0x90
[    8.896980] rk_multicodecs_probe+0x86c/0x970
[    8.896986] platform_drv_probe+0x9c/0xc4
[    8.896992] really_probe+0x204/0x510
[    8.896998] driver_probe_device+0x80/0xc0
[    8.897004] device_driver_attach+0x70/0xb4
[    8.897010] __driver_attach+0xc8/0x150
[    8.897015] bus_for_each_dev+0x80/0xd0
[    8.897021] driver_attach+0x28/0x38
[    8.897026] bus_add_driver+0x108/0x1e8
[    8.897032] driver_register+0x7c/0x118
[    8.897038] __platform_driver_register+0x48/0x58
[    8.897045] rockchip_multicodecs_driver_init+0x20/0x30
[    8.897050] do_one_initcall+0x98/0x2dc
[    8.897056] do_initcall_level+0xa8/0x160
[    8.897062] do_initcalls+0x58/0x9c
[    8.897068] do_basic_setup+0x28/0x38
[    8.897075] kernel_init_freeable+0xf4/0x16c
[    8.897081] kernel_init+0x18/0x190
[    8.897087] ret_from_fork+0x10/0x30
```

4. I2S 驱动程序注册的 DAI 驱动程序的 set_fmt 操作被调用
```
[    8.897348] rockchip_i2s_tdm_set_fmt+0x34/0x2e0
[    8.897355] snd_soc_dai_set_fmt+0x30/0x88
[    8.897360] snd_soc_runtime_set_dai_fmt+0x140/0x1b4
[    8.897366] snd_soc_bind_card+0x6c8/0xad0
[    8.897372] snd_soc_register_card+0xf8/0x114
[    8.897377] devm_snd_soc_register_card+0x48/0x90
[    8.897383] rk_multicodecs_probe+0x86c/0x970
[    8.897390] platform_drv_probe+0x9c/0xc4
[    8.897396] really_probe+0x204/0x510
[    8.897402] driver_probe_device+0x80/0xc0
[    8.897408] device_driver_attach+0x70/0xb4
[    8.897414] __driver_attach+0xc8/0x150
[    8.897419] bus_for_each_dev+0x80/0xd0
[    8.897425] driver_attach+0x28/0x38
[    8.897430] bus_add_driver+0x108/0x1e8
[    8.897436] driver_register+0x7c/0x118
[    8.897442] __platform_driver_register+0x48/0x58
[    8.897449] rockchip_multicodecs_driver_init+0x20/0x30
[    8.897455] do_one_initcall+0x98/0x2dc
[    8.897461] do_initcall_level+0xa8/0x160
[    8.897467] do_initcalls+0x58/0x9c
[    8.897473] do_basic_setup+0x28/0x38
[    8.897479] kernel_init_freeable+0xf4/0x16c
[    8.897486] kernel_init+0x18/0x190
[    8.897491] ret_from_fork+0x10/0x30
```

5. 创建 PCM 音频设备
```
[    8.897609] snd_device_new+0x54/0x138
[    8.897614] _snd_pcm_new+0x114/0x190
[    8.897620] snd_pcm_new+0x1c/0x2c
[    8.897626] soc_new_pcm+0x4b0/0x560
[    8.897632] snd_soc_bind_card+0x750/0xad0
[    8.897637] snd_soc_register_card+0xf8/0x114
[    8.897643] devm_snd_soc_register_card+0x48/0x90
[    8.897649] rk_multicodecs_probe+0x86c/0x970
[    8.897655] platform_drv_probe+0x9c/0xc4
[    8.897661] really_probe+0x204/0x510
[    8.897667] driver_probe_device+0x80/0xc0
[    8.897673] device_driver_attach+0x70/0xb4
[    8.897678] __driver_attach+0xc8/0x150
[    8.897684] bus_for_each_dev+0x80/0xd0
[    8.897689] driver_attach+0x28/0x38
[    8.897695] bus_add_driver+0x108/0x1e8
[    8.897701] driver_register+0x7c/0x118
[    8.897708] __platform_driver_register+0x48/0x58
[    8.897714] rockchip_multicodecs_driver_init+0x20/0x30
[    8.897719] do_one_initcall+0x98/0x2dc
[    8.897725] do_initcall_level+0xa8/0x160
[    8.897731] do_initcalls+0x58/0x9c
[    8.897738] do_basic_setup+0x28/0x38
[    8.897744] kernel_init_freeable+0xf4/0x16c
[    8.897750] kernel_init+0x18/0x190
[    8.897756] ret_from_fork+0x10/0x30
```

6. 立即创建 PCM 设备文件
```
[    9.000061] snd_register_device+0x40/0x194
[    9.000080] snd_pcm_dev_register+0x60/0x1e0
[    9.000099] snd_device_register_all+0x60/0x8c
[    9.000116] snd_card_register+0x50/0x184
[    9.000135] snd_soc_bind_card+0x930/0xad0
[    9.000151] snd_soc_register_card+0xf8/0x114
[    9.000170] devm_snd_soc_register_card+0x48/0x90
[    9.000188] rk_multicodecs_probe+0x86c/0x970
[    9.000208] platform_drv_probe+0x9c/0xc4
[    9.000225] really_probe+0x204/0x510
[    9.000242] driver_probe_device+0x80/0xc0
[    9.000259] device_driver_attach+0x70/0xb4
[    9.000275] __driver_attach+0xc8/0x150
[    9.000292] bus_for_each_dev+0x80/0xd0
[    9.000308] driver_attach+0x28/0x38
[    9.000324] bus_add_driver+0x108/0x1e8
[    9.000342] driver_register+0x7c/0x118
[    9.000359] __platform_driver_register+0x48/0x58
[    9.000379] rockchip_multicodecs_driver_init+0x20/0x30
[    9.000396] do_one_initcall+0x98/0x2dc
[    9.000415] do_initcall_level+0xa8/0x160
[    9.000432] do_initcalls+0x58/0x9c
[    9.000449] do_basic_setup+0x28/0x38
[    9.000467] kernel_init_freeable+0xf4/0x16c
[    9.000484] kernel_init+0x18/0x190
[    9.000501] ret_from_fork+0x10/0x30
```

7. 继续创建 Timer 音频设备
```
[    9.001087] snd_device_new+0x54/0x138
[    9.001104] snd_timer_new+0x120/0x17c
[    9.001122] snd_pcm_timer_init+0x6c/0x140
[    9.001140] snd_pcm_dev_register+0x78/0x1e0
[    9.001158] snd_device_register_all+0x60/0x8c
[    9.001175] snd_card_register+0x50/0x184
[    9.001192] snd_soc_bind_card+0x930/0xad0
[    9.001209] snd_soc_register_card+0xf8/0x114
[    9.001227] devm_snd_soc_register_card+0x48/0x90
[    9.001244] rk_multicodecs_probe+0x86c/0x970
[    9.001263] platform_drv_probe+0x9c/0xc4
[    9.001279] really_probe+0x204/0x510
[    9.001297] driver_probe_device+0x80/0xc0
[    9.001313] device_driver_attach+0x70/0xb4
[    9.001330] __driver_attach+0xc8/0x150
[    9.001347] bus_for_each_dev+0x80/0xd0
[    9.001363] driver_attach+0x28/0x38
[    9.001379] bus_add_driver+0x108/0x1e8
[    9.001396] driver_register+0x7c/0x118
[    9.001414] __platform_driver_register+0x48/0x58
[    9.001432] rockchip_multicodecs_driver_init+0x20/0x30
[    9.001448] do_one_initcall+0x98/0x2dc
[    9.001466] do_initcall_level+0xa8/0x160
[    9.001484] do_initcalls+0x58/0x9c
[    9.001501] do_basic_setup+0x28/0x38
[    9.001518] kernel_init_freeable+0xf4/0x16c
[    9.001535] kernel_init+0x18/0x190
[    9.001551] ret_from_fork+0x10/0x30
```

8. 再次创建 PCM 设备文件和 Timer 音频设备
```
[    9.001742] snd_register_device+0x40/0x194
[    9.001759] snd_pcm_dev_register+0x130/0x1e0
[    9.001778] snd_device_register_all+0x60/0x8c
[    9.001794] snd_card_register+0x50/0x184
[    9.001811] snd_soc_bind_card+0x930/0xad0
[    9.001828] snd_soc_register_card+0xf8/0x114
[    9.001845] devm_snd_soc_register_card+0x48/0x90
[    9.001862] rk_multicodecs_probe+0x86c/0x970
[    9.001880] platform_drv_probe+0x9c/0xc4
[    9.001897] really_probe+0x204/0x510
[    9.001913] driver_probe_device+0x80/0xc0
[    9.001930] device_driver_attach+0x70/0xb4
[    9.001947] __driver_attach+0xc8/0x150
[    9.001963] bus_for_each_dev+0x80/0xd0
[    9.001980] driver_attach+0x28/0x38
[    9.001995] bus_add_driver+0x108/0x1e8
[    9.002012] driver_register+0x7c/0x118
[    9.002029] __platform_driver_register+0x48/0x58
[    9.002047] rockchip_multicodecs_driver_init+0x20/0x30
[    9.002063] do_one_initcall+0x98/0x2dc
[    9.002080] do_initcall_level+0xa8/0x160
[    9.002096] do_initcalls+0x58/0x9c
[    9.002113] do_basic_setup+0x28/0x38
[    9.002130] kernel_init_freeable+0xf4/0x16c
[    9.002147] kernel_init+0x18/0x190
[    9.002163] ret_from_fork+0x10/0x30
```

以及
```
[    9.002732] snd_device_new+0x54/0x138
[    9.002749] snd_timer_new+0x120/0x17c
[    9.002765] snd_pcm_timer_init+0x6c/0x140
[    9.002782] snd_pcm_dev_register+0x148/0x1e0
[    9.002800] snd_device_register_all+0x60/0x8c
[    9.002817] snd_card_register+0x50/0x184
[    9.002834] snd_soc_bind_card+0x930/0xad0
[    9.002851] snd_soc_register_card+0xf8/0x114
[    9.002868] devm_snd_soc_register_card+0x48/0x90
[    9.002885] rk_multicodecs_probe+0x86c/0x970
[    9.002903] platform_drv_probe+0x9c/0xc4
[    9.002920] really_probe+0x204/0x510
[    9.002936] driver_probe_device+0x80/0xc0
[    9.002953] device_driver_attach+0x70/0xb4
[    9.002970] __driver_attach+0xc8/0x150
[    9.002986] bus_for_each_dev+0x80/0xd0
[    9.003002] driver_attach+0x28/0x38
[    9.003019] bus_add_driver+0x108/0x1e8
[    9.003036] driver_register+0x7c/0x118
[    9.003053] __platform_driver_register+0x48/0x58
[    9.003071] rockchip_multicodecs_driver_init+0x20/0x30
[    9.003088] do_one_initcall+0x98/0x2dc
[    9.003105] do_initcall_level+0xa8/0x160
[    9.003122] do_initcalls+0x58/0x9c
[    9.003139] do_basic_setup+0x28/0x38
[    9.003157] kernel_init_freeable+0xf4/0x16c
[    9.003174] kernel_init+0x18/0x190
[    9.003190] ret_from_fork+0x10/0x30
```

9. 创建 Control 设备文件
```
[    9.003986] snd_register_device+0x40/0x194
[    9.004004] snd_ctl_dev_register+0x30/0x40
[    9.004022] snd_device_register_all+0x60/0x8c
[    9.004039] snd_card_register+0x50/0x184
[    9.004056] snd_soc_bind_card+0x930/0xad0
[    9.004072] snd_soc_register_card+0xf8/0x114
[    9.004089] devm_snd_soc_register_card+0x48/0x90
[    9.004107] rk_multicodecs_probe+0x86c/0x970
[    9.004124] platform_drv_probe+0x9c/0xc4
[    9.004141] really_probe+0x204/0x510
[    9.004157] driver_probe_device+0x80/0xc0
[    9.004174] device_driver_attach+0x70/0xb4
[    9.004190] __driver_attach+0xc8/0x150
[    9.004206] bus_for_each_dev+0x80/0xd0
[    9.004222] driver_attach+0x28/0x38
[    9.004237] bus_add_driver+0x108/0x1e8
[    9.004254] driver_register+0x7c/0x118
[    9.004272] __platform_driver_register+0x48/0x58
[    9.004289] rockchip_multicodecs_driver_init+0x20/0x30
[    9.004306] do_one_initcall+0x98/0x2dc
[    9.004323] do_initcall_level+0xa8/0x160
[    9.004340] do_initcalls+0x58/0x9c
[    9.004357] do_basic_setup+0x28/0x38
[    9.004374] kernel_init_freeable+0xf4/0x16c
[    9.004391] kernel_init+0x18/0x190
[    9.004407] ret_from_fork+0x10/0x30
```

上面的这些过程都是驱动程序初始化期间完成的。用户空间应用程序通过音频设备文件播放音频数据时，一些回调会被调到。

10. I2S 驱动程序注册的 DAI 驱动程序的 set_sysclk 操作被调用
```
[   20.693449] rockchip_i2s_tdm_set_sysclk+0x2c/0x8c
[   20.693453] snd_soc_dai_set_sysclk+0x44/0xb0
[   20.693457] rk_multicodecs_hw_params+0x74/0xd4
[   20.693461] snd_soc_link_hw_params+0x30/0x8c
[   20.693464] soc_pcm_hw_params+0x248/0x534
[   20.693469] snd_pcm_hw_params+0x1b4/0x4cc
[   20.693473] snd_pcm_ioctl_hw_params_compat+0xb8/0x274
[   20.693477] snd_pcm_ioctl_compat+0x484/0x494
[   20.693482] __arm64_compat_sys_ioctl+0x10c/0x160
[   20.693486] el0_svc_common+0xc0/0x23c
[   20.693489] do_el0_svc_compat+0x20/0x50
[   20.693492] el0_svc_compat+0x14/0x24
[   20.693496] el0_sync_compat_handler+0x7c/0xbc
[   20.693499] el0_sync_compat+0x1ac/0x1c0
```

11. I2S 驱动程序注册的 DAI 驱动程序的 hw_params 操作被调用
```
[   20.693817] rockchip_i2s_tdm_hw_params+0x4c/0xb84
[   20.693821] snd_soc_dai_hw_params+0x60/0xb8
[   20.693824] soc_pcm_hw_params+0x3c8/0x534
[   20.693828] snd_pcm_hw_params+0x1b4/0x4cc
[   20.693831] snd_pcm_ioctl_hw_params_compat+0xb8/0x274
[   20.693834] snd_pcm_ioctl_compat+0x484/0x494
[   20.693837] __arm64_compat_sys_ioctl+0x10c/0x160
[   20.693840] el0_svc_common+0xc0/0x23c
[   20.693843] do_el0_svc_compat+0x20/0x50
[   20.693846] el0_svc_compat+0x14/0x24
[   20.693849] el0_sync_compat_handler+0x7c/0xbc
[   20.693851] el0_sync_compat+0x1ac/0x1c0
```

11. I2S 驱动程序注册的 DAI 驱动程序的 trigger 操作被调用
`trigger` 操作用于触发向设备传输数据的启动和停止。
```
[   20.694843] rockchip_i2s_tdm_trigger+0x40/0x4e0
[   20.694850] snd_soc_pcm_dai_trigger+0xa4/0xec
[   20.694856] soc_pcm_trigger+0x74/0xb8
[   20.694862] snd_pcm_start+0x14c/0x1b0
[   20.694868] __snd_pcm_lib_xfer+0x598/0x6f0
[   20.694874] snd_pcm_ioctl_xferi_compat+0x250/0x47c
[   20.694880] snd_pcm_ioctl_compat+0x36c/0x494
[   20.694887] __arm64_compat_sys_ioctl+0x10c/0x160
[   20.694893] el0_svc_common+0xc0/0x23c
[   20.694899] do_el0_svc_compat+0x20/0x50
[   20.694904] el0_svc_compat+0x14/0x24
[   20.694907] el0_sync_compat_handler+0x7c/0xbc
[   20.694910] el0_sync_compat+0x1ac/0x1c0
```

```
[   25.618489] rockchip_i2s_tdm_trigger+0x40/0x4e0
[   25.618516] snd_soc_pcm_dai_trigger+0xa4/0xec
[   25.618543] soc_pcm_trigger+0x84/0xb8
[   25.618572] snd_pcm_stop+0xa4/0x148
[   25.618598] snd_pcm_release_substream+0x90/0x19c
[   25.618625] snd_pcm_release+0x44/0xa8
[   25.618652] __fput+0xdc/0x238
[   25.618678] ____fput+0x14/0x24
[   25.618706] task_work_run+0x90/0x148
[   25.618734] do_notify_resume+0x11c/0x218
[   25.618760] work_pending+0xc/0x5f8
```

另一个 `compatible` 为 **"simple-audio-card"** 的声卡驱动，其设备树节点为：
```
	hdmiin-sound {
	     compatible = "simple-audio-card";
	     simple-audio-card,format = "i2s";
	     simple-audio-card,name = "rockchip,hdmiin";
	     simple-audio-card,bitclock-master = <&dailink0_master>;
	     simple-audio-card,frame-master = <&dailink0_master>;
	     status = "okay";
	     simple-audio-card,cpu {
	         sound-dai = <&i2s7_8ch>;
	     };
	     dailink0_master: simple-audio-card,codec {
	         sound-dai = <&hdmiin_dc>;
	     };
	};
```

1. 创建 Control 音频设备
```
[    9.145160] snd_device_new+0x54/0x138
[    9.145169] snd_ctl_create+0x64/0x94
[    9.145178] snd_card_new+0x2c4/0x394
[    9.145188] snd_soc_bind_card+0x354/0xad0
[    9.145196] snd_soc_register_card+0xf8/0x114
[    9.145205] devm_snd_soc_register_card+0x48/0x90
[    9.145214] asoc_simple_probe+0x2a0/0x348
[    9.145223] platform_drv_probe+0x9c/0xc4
[    9.145232] really_probe+0x204/0x510
[    9.145240] driver_probe_device+0x80/0xc0
[    9.145248] __device_attach_driver+0x118/0x140
[    9.145256] bus_for_each_drv+0x84/0xd4
[    9.145265] __device_attach+0xc0/0x158
[    9.145273] device_initial_probe+0x18/0x28
[    9.145281] bus_probe_device+0x38/0xa0
[    9.145289] deferred_probe_work_func+0x80/0xe0
[    9.145298] process_one_work+0x1f4/0x490
[    9.145306] worker_thread+0x324/0x4dc
[    9.145315] kthread+0x13c/0x344
[    9.145323] ret_from_fork+0x10/0x30
```

2. I2S 驱动程序注册的 DAI 驱动程序的 porbe 函数被调用
```
[    9.145524] rockchip_i2s_tdm_dai_probe+0x24/0x84
[    9.145532] snd_soc_pcm_dai_probe+0xb0/0xec
[    9.145541] snd_soc_bind_card+0x5d8/0xad0
[    9.145549] snd_soc_register_card+0xf8/0x114
[    9.145557] devm_snd_soc_register_card+0x48/0x90
[    9.145565] asoc_simple_probe+0x2a0/0x348
[    9.145574] platform_drv_probe+0x9c/0xc4
[    9.145582] really_probe+0x204/0x510
[    9.145590] driver_probe_device+0x80/0xc0
[    9.145599] __device_attach_driver+0x118/0x140
[    9.145607] bus_for_each_drv+0x84/0xd4
[    9.145615] __device_attach+0xc0/0x158
[    9.145623] device_initial_probe+0x18/0x28
[    9.145631] bus_probe_device+0x38/0xa0
[    9.145639] deferred_probe_work_func+0x80/0xe0
[    9.145648] process_one_work+0x1f4/0x490
[    9.145655] worker_thread+0x324/0x4dc
[    9.145664] kthread+0x13c/0x344
[    9.145672] ret_from_fork+0x10/0x30
```

3. I2S 驱动程序注册的 DAI 驱动程序的 set_sysclk 操作被调用
```
[    9.145755] rockchip_i2s_tdm_set_sysclk+0x2c/0x8c
[    9.145764] snd_soc_dai_set_sysclk+0x44/0xb0
[    9.145772] asoc_simple_init_dai+0x3c/0xc4
[    9.145781] asoc_simple_dai_init+0x74/0x178
[    9.145789] snd_soc_link_init+0x28/0x84
[    9.145797] snd_soc_bind_card+0x6b4/0xad0
[    9.145805] snd_soc_register_card+0xf8/0x114
[    9.145814] devm_snd_soc_register_card+0x48/0x90
[    9.145821] asoc_simple_probe+0x2a0/0x348
[    9.145830] platform_drv_probe+0x9c/0xc4
[    9.145838] really_probe+0x204/0x510
[    9.145846] driver_probe_device+0x80/0xc0
[    9.145854] __device_attach_driver+0x118/0x140
[    9.145862] bus_for_each_drv+0x84/0xd4
[    9.145870] __device_attach+0xc0/0x158
[    9.145878] device_initial_probe+0x18/0x28
[    9.145886] bus_probe_device+0x38/0xa0
[    9.145895] deferred_probe_work_func+0x80/0xe0
[    9.145903] process_one_work+0x1f4/0x490
[    9.145910] worker_thread+0x324/0x4dc
[    9.145919] kthread+0x13c/0x344
[    9.145927] ret_from_fork+0x10/0x30
```

4. I2S 驱动程序注册的 DAI 驱动程序的 set_fmt 操作被调用
```
[    9.146010] rockchip_i2s_tdm_set_fmt+0x34/0x2e0
[    9.146019] snd_soc_dai_set_fmt+0x30/0x88
[    9.146027] snd_soc_runtime_set_dai_fmt+0x140/0x1b4
[    9.146036] snd_soc_bind_card+0x6c8/0xad0
[    9.146044] snd_soc_register_card+0xf8/0x114
[    9.146052] devm_snd_soc_register_card+0x48/0x90
[    9.146060] asoc_simple_probe+0x2a0/0x348
[    9.146069] platform_drv_probe+0x9c/0xc4
[    9.146077] really_probe+0x204/0x510
[    9.146085] driver_probe_device+0x80/0xc0
[    9.146093] __device_attach_driver+0x118/0x140
[    9.146101] bus_for_each_drv+0x84/0xd4
[    9.146109] __device_attach+0xc0/0x158
[    9.146117] device_initial_probe+0x18/0x28
[    9.146125] bus_probe_device+0x38/0xa0
[    9.146134] deferred_probe_work_func+0x80/0xe0
[    9.146142] process_one_work+0x1f4/0x490
[    9.146150] worker_thread+0x324/0x4dc
[    9.146158] kthread+0x13c/0x344
[    9.146166] ret_from_fork+0x10/0x30
```

5. 创建 PCM 音频设备
```
[    9.146323] snd_device_new+0x54/0x138
[    9.146332] _snd_pcm_new+0x114/0x190
[    9.146340] snd_pcm_new+0x1c/0x2c
[    9.146348] soc_new_pcm+0x4b0/0x560
[    9.146356] snd_soc_bind_card+0x750/0xad0
[    9.146365] snd_soc_register_card+0xf8/0x114
[    9.146373] devm_snd_soc_register_card+0x48/0x90
[    9.146381] asoc_simple_probe+0x2a0/0x348
[    9.146390] platform_drv_probe+0x9c/0xc4
[    9.146398] really_probe+0x204/0x510
[    9.146406] driver_probe_device+0x80/0xc0
[    9.146414] __device_attach_driver+0x118/0x140
[    9.146422] bus_for_each_drv+0x84/0xd4
[    9.146430] __device_attach+0xc0/0x158
[    9.146439] device_initial_probe+0x18/0x28
[    9.146447] bus_probe_device+0x38/0xa0
[    9.146455] deferred_probe_work_func+0x80/0xe0
[    9.146463] process_one_work+0x1f4/0x490
[    9.146470] worker_thread+0x324/0x4dc
[    9.146479] kthread+0x13c/0x344
[    9.146487] ret_from_fork+0x10/0x30
```

6. 立即创建 PCM 设备文件
```
[    9.147447] snd_register_device+0x40/0x194
[    9.147456] snd_pcm_dev_register+0x130/0x1e0
[    9.147465] snd_device_register_all+0x60/0x8c
[    9.147473] snd_card_register+0x50/0x184
[    9.147482] snd_soc_bind_card+0x930/0xad0
[    9.147490] snd_soc_register_card+0xf8/0x114
[    9.147499] devm_snd_soc_register_card+0x48/0x90
[    9.147507] asoc_simple_probe+0x2a0/0x348
[    9.147516] platform_drv_probe+0x9c/0xc4
[    9.147524] really_probe+0x204/0x510
[    9.147532] driver_probe_device+0x80/0xc0
[    9.147540] __device_attach_driver+0x118/0x140
[    9.147548] bus_for_each_drv+0x84/0xd4
[    9.147556] __device_attach+0xc0/0x158
[    9.147565] device_initial_probe+0x18/0x28
[    9.147573] bus_probe_device+0x38/0xa0
[    9.147581] deferred_probe_work_func+0x80/0xe0
[    9.147589] process_one_work+0x1f4/0x490
[    9.147597] worker_thread+0x324/0x4dc
[    9.147605] kthread+0x13c/0x344
[    9.147613] ret_from_fork+0x10/0x30
```

7. 继续创建 Timer 音频设备
```
[    9.147823] snd_device_new+0x54/0x138
[    9.147831] snd_timer_new+0x120/0x17c
[    9.147841] snd_pcm_timer_init+0x6c/0x140
[    9.147849] snd_pcm_dev_register+0x148/0x1e0
[    9.147858] snd_device_register_all+0x60/0x8c
[    9.147866] snd_card_register+0x50/0x184
[    9.147874] snd_soc_bind_card+0x930/0xad0
[    9.147882] snd_soc_register_card+0xf8/0x114
[    9.147891] devm_snd_soc_register_card+0x48/0x90
[    9.147899] asoc_simple_probe+0x2a0/0x348
[    9.147907] platform_drv_probe+0x9c/0xc4
[    9.147916] really_probe+0x204/0x510
[    9.147924] driver_probe_device+0x80/0xc0
[    9.147932] __device_attach_driver+0x118/0x140
[    9.147940] bus_for_each_drv+0x84/0xd4
[    9.147948] __device_attach+0xc0/0x158
[    9.147957] device_initial_probe+0x18/0x28
[    9.147965] bus_probe_device+0x38/0xa0
[    9.147973] deferred_probe_work_func+0x80/0xe0
[    9.147981] process_one_work+0x1f4/0x490
[    9.147989] worker_thread+0x324/0x4dc
[    9.147997] kthread+0x13c/0x344
[    9.148005] ret_from_fork+0x10/0x30
```

8. 创建 Control 设备文件
```
[    9.148092] snd_register_device+0x40/0x194
[    9.148101] snd_ctl_dev_register+0x30/0x40
[    9.148110] snd_device_register_all+0x60/0x8c
[    9.148118] snd_card_register+0x50/0x184
[    9.148126] snd_soc_bind_card+0x930/0xad0
[    9.148134] snd_soc_register_card+0xf8/0x114
[    9.148143] devm_snd_soc_register_card+0x48/0x90
[    9.148151] asoc_simple_probe+0x2a0/0x348
[    9.148160] platform_drv_probe+0x9c/0xc4
[    9.148168] really_probe+0x204/0x510
[    9.148176] driver_probe_device+0x80/0xc0
[    9.148184] __device_attach_driver+0x118/0x140
[    9.148192] bus_for_each_drv+0x84/0xd4
[    9.148200] __device_attach+0xc0/0x158
[    9.148208] device_initial_probe+0x18/0x28
[    9.148216] bus_probe_device+0x38/0xa0
[    9.148224] deferred_probe_work_func+0x80/0xe0
[    9.148233] process_one_work+0x1f4/0x490
[    9.148241] worker_thread+0x324/0x4dc
[    9.148250] kthread+0x13c/0x344
[    9.148257] ret_from_fork+0x10/0x30
```

Done.