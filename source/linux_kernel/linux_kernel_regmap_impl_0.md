---
title: Linux 内核设备驱动程序的IO寄存器访问 (上)
date: 2023-08-25 19:03:29
categories: Linux 内核
tags:
- Linux 内核
---

Linux 内核提供了一套可缓存的设备 IO 寄存器访问机制，即 **regmap**。**regmap** 机制支持以统一的接口，访问多种不同类型的设备 IO 寄存器，如内存映射的设备 IO 寄存器，需要通过 I2C、I3C、SPI、AC97 和 SLIMBUS 等总线访问的设备 IO 寄存器等。内存映射和 I2C 总线是嵌入式系统中比较常见的访问设备 IO 寄存器的方式，**regmap** 机制对这些方式提供了良好的支持。

## 内存映射设备 IO 寄存器访问

内存映射设备 IO 寄存器是将设备的 IO 寄存器映射到物理内存地址空间，软件访问设备 IO 寄存器就像访问普通的物理内存一样。设备的 IO 寄存器在物理内存地址空间中的具体区域，由整个硬件系统的设计决定。

由于内存管理单元 MMU 及虚拟内存的应用，在 Linux 内核设备驱动程序中，一般无法直接通过物理内存地址访问设备的 IO 寄存器。在 Linux 内核设备驱动程序中，访问内存映射设备 IO 寄存器的方法如下：

1. 在设备树文件的设备节点定义中说明内存映射设备 IO 寄存器在物理内存地址空间中的范围，这包括起始物理地址和地址空间大小，如 *arch/arm64/boot/dts/rockchip/rk3399.dtsi* 文件中 I2S0 设备节点的定义：
```
	i2s0: i2s@ff880000 {
		compatible = "rockchip,rk3399-i2s", "rockchip,rk3066-i2s";
		reg = <0x0 0xff880000 0x0 0x1000>;
 . . . . . .
	};
```

`reg = <0x0 0xff880000 0x0 0x1000>;` 行说明了这个设备的内存映射设备 IO 寄存器在物理内存地址空间中的范围，起始地址为 0xff880000，地址空间大小为 0x1000，即 4KB。

2. 创建寄存器映射配置，如 *sound/soc/rockchip/rockchip_i2s.c* 文件中的寄存器映射配置：
```
static bool rockchip_i2s_wr_reg(struct device *dev, unsigned int reg)
{
	switch (reg) {
	case I2S_TXCR:
	case I2S_RXCR:
	case I2S_CKR:
	case I2S_DMACR:
	case I2S_INTCR:
	case I2S_XFER:
	case I2S_CLR:
	case I2S_TXDR:
		return true;
	default:
		return false;
	}
}

static bool rockchip_i2s_rd_reg(struct device *dev, unsigned int reg)
{
	switch (reg) {
	case I2S_TXCR:
	case I2S_RXCR:
	case I2S_CKR:
	case I2S_DMACR:
	case I2S_INTCR:
	case I2S_XFER:
	case I2S_CLR:
	case I2S_TXDR:
	case I2S_RXDR:
	case I2S_FIFOLR:
	case I2S_INTSR:
		return true;
	default:
		return false;
	}
}

static bool rockchip_i2s_volatile_reg(struct device *dev, unsigned int reg)
{
	switch (reg) {
	case I2S_INTSR:
	case I2S_CLR:
	case I2S_FIFOLR:
	case I2S_TXDR:
	case I2S_RXDR:
		return true;
	default:
		return false;
	}
}

static bool rockchip_i2s_precious_reg(struct device *dev, unsigned int reg)
{
	switch (reg) {
	case I2S_RXDR:
		return true;
	default:
		return false;
	}
}

static const struct reg_default rockchip_i2s_reg_defaults[] = {
	{0x00, 0x0000000f},
	{0x04, 0x0000000f},
	{0x08, 0x00071f1f},
	{0x10, 0x001f0000},
	{0x14, 0x01f00000},
};

static const struct regmap_config rockchip_i2s_regmap_config = {
	.reg_bits = 32,
	.reg_stride = 4,
	.val_bits = 32,
	.max_register = I2S_RXDR,
	.reg_defaults = rockchip_i2s_reg_defaults,
	.num_reg_defaults = ARRAY_SIZE(rockchip_i2s_reg_defaults),
	.writeable_reg = rockchip_i2s_wr_reg,
	.readable_reg = rockchip_i2s_rd_reg,
	.volatile_reg = rockchip_i2s_volatile_reg,
	.precious_reg = rockchip_i2s_precious_reg,
	.cache_type = REGCACHE_FLAT,
};
```

寄存器映射配置由 `struct regmap_config` 结构体来描述，它告诉 **regmap** 机制设备寄存器地址的位宽，设备寄存器的值的位宽，最大的设备寄存器地址，有特殊默认值的设备寄存器的默认值，设备寄存器的读写属性，及设备寄存器访问的缓存类型等。

Linux 设备驱动程序通过寄存器映射配置，为 **regmap** 机制提供缓存访问、访问控制和访问操作执行等所需的信息。

可能有很多人觉得，由于设备 IO 寄存器的特殊性，对设备 IO 寄存器的访问都是不缓存的。实际上，CPU 不缓存对设备 IO 寄存器的访问，但 Linux 内核的 **regmap** 机制，在内存中开辟了一块区域来做设备 IO 寄存器的缓存访问。

3. 在设备驱动程序的 `probe` 操作中，获得 IO 重映射资源，初始化内存映射 IO 的 regmap，创建 `struct regmap` 结构对象，如 *sound/soc/rockchip/rockchip_i2s.c* 文件中的 `rockchip_i2s_probe()` 操作：
```
static int rockchip_i2s_probe(struct platform_device *pdev)
{
	struct device_node *node = pdev->dev.of_node;
	const struct of_device_id *of_id;
	struct rk_i2s_dev *i2s;
	struct snd_soc_dai_driver *soc_dai;
	struct resource *res;
	void __iomem *regs;
 . . . . . .
	regs = devm_platform_get_and_ioremap_resource(pdev, 0, &res);
	if (IS_ERR(regs)) {
		ret = PTR_ERR(regs);
		goto err_clk;
	}

	i2s->regmap = devm_regmap_init_mmio(&pdev->dev, regs,
					    &rockchip_i2s_regmap_config);
	if (IS_ERR(i2s->regmap)) {
		dev_err(&pdev->dev,
			"Failed to initialise managed register map\n");
		ret = PTR_ERR(i2s->regmap);
		goto err_clk;
	}
 . . . . . .
	return ret;
}
```

这里看到 `devm_platform_get_and_ioremap_resource()` 函数，获得在设备树文件中定义的 IO 重映射资源。它的返回值是类型为 `void __iomem *` 的内存映射设备 IO 寄存器地址空间基地址，在内核内存地址空间的虚拟地址，并通过传出参数 `struct resource` 返回内存映射设备 IO 寄存器的物理内存地址空间。

通过 `devm_platform_get_and_ioremap_resource()` 函数返回的指针，Linux 设备驱动程序可以像访问普通内存一样访问设备 IO 寄存器，如：
```
	u32 val;
 . . . . . .
	val = readl(regs + I2C_REG_STATUS);
 . . . . . .
	writel(regs + I2C_REG_CONFIGURE, 1);
```

通过内核提供的 `readl()` IO访问函数，像访问普通内存那样读取 32 位的设备 IO 寄存器的值。

相对于直接访问设备 IO 寄存器，**regmap** 机制可以提供的更多，如方便的读、写和更新操作函数，缓存的访问，及通过 debugfs 分析调试的能力等。调用 `devm_regmap_init_mmio()` 函数创建 `struct regmap` 是明智的，使用 **regmap** 机制是明智的。

4. 通过 **regmap** 机制提供的读、写和更新等操作函数访问设备 IO 寄存器，如 *sound/soc/rockchip/rockchip_i2s.c* 文件中的如下代码片段：
```
		if (!i2s->rx_start) {
			regmap_update_bits(i2s->regmap, I2S_XFER,
					   I2S_XFER_TXS_START |
					   I2S_XFER_RXS_START,
					   I2S_XFER_TXS_STOP |
					   I2S_XFER_RXS_STOP);

			udelay(150);
			regmap_update_bits(i2s->regmap, I2S_CLR,
					   I2S_CLR_TXC | I2S_CLR_RXC,
					   I2S_CLR_TXC | I2S_CLR_RXC);

			regmap_read(i2s->regmap, I2S_CLR, &val);

			/* Should wait for clear operation to finish */
			while (val) {
				regmap_read(i2s->regmap, I2S_CLR, &val);
				retry--;
				if (!retry) {
					dev_warn(i2s->dev, "fail to clear\n");
					break;
				}
			}
		}
```

可用以访问设备 IO 寄存器的函数有很多，如：
```
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
int regmap_write_async(struct regmap *map, unsigned int reg, unsigned int val);
int regmap_raw_write(struct regmap *map, unsigned int reg,
		     const void *val, size_t val_len);

int regmap_read(struct regmap *map, unsigned int reg, unsigned int *val);
int regmap_raw_read(struct regmap *map, unsigned int reg,
		    void *val, size_t val_len);

int regmap_update_bits_base(struct regmap *map, unsigned int reg,
			    unsigned int mask, unsigned int val,
			    bool *change, bool async, bool force);

static inline int regmap_update_bits(struct regmap *map, unsigned int reg,
				     unsigned int mask, unsigned int val)
{
	return regmap_update_bits_base(map, reg, mask, val, NULL, false, false);
}

static inline int regmap_set_bits(struct regmap *map,
				  unsigned int reg, unsigned int bits)
{
	return regmap_update_bits_base(map, reg, bits, bits,
				       NULL, false, false);
}

static inline int regmap_clear_bits(struct regmap *map,
				    unsigned int reg, unsigned int bits)
{
	return regmap_update_bits_base(map, reg, bits, 0, NULL, false, false);
}

int regmap_test_bits(struct regmap *map, unsigned int reg, unsigned int bits);
```

访问内存映射的设备 IO 寄存器是嵌入式 Linux 系统中，很多内核设备驱动程序所需的最基本最简单，也最重要的操作。

## regmap 机制的实现

`struct regmap_config` 的定义 (位于 *include/linux/regmap.h*) 如下：
```
enum regcache_type {
	REGCACHE_NONE,
	REGCACHE_RBTREE,
	REGCACHE_COMPRESSED,
	REGCACHE_FLAT,
};
 . . . . . .
struct reg_default {
	unsigned int reg;
	unsigned int def;
};
 . . . . . .
struct regmap_range {
	unsigned int range_min;
	unsigned int range_max;
};

#define regmap_reg_range(low, high) { .range_min = low, .range_max = high, }
 . . . . . .
struct regmap_access_table {
	const struct regmap_range *yes_ranges;
	unsigned int n_yes_ranges;
	const struct regmap_range *no_ranges;
	unsigned int n_no_ranges;
};

typedef void (*regmap_lock)(void *);
typedef void (*regmap_unlock)(void *);
 . . . . . .
struct regmap_config {
	const char *name;

	int reg_bits;
	int reg_stride;
	int pad_bits;
	int val_bits;

	bool (*writeable_reg)(struct device *dev, unsigned int reg);
	bool (*readable_reg)(struct device *dev, unsigned int reg);
	bool (*volatile_reg)(struct device *dev, unsigned int reg);
	bool (*precious_reg)(struct device *dev, unsigned int reg);
	bool (*writeable_noinc_reg)(struct device *dev, unsigned int reg);
	bool (*readable_noinc_reg)(struct device *dev, unsigned int reg);

	bool disable_locking;
	regmap_lock lock;
	regmap_unlock unlock;
	void *lock_arg;

	int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
	int (*reg_write)(void *context, unsigned int reg, unsigned int val);

	bool fast_io;

	unsigned int max_register;
	const struct regmap_access_table *wr_table;
	const struct regmap_access_table *rd_table;
	const struct regmap_access_table *volatile_table;
	const struct regmap_access_table *precious_table;
	const struct regmap_access_table *wr_noinc_table;
	const struct regmap_access_table *rd_noinc_table;
	const struct reg_default *reg_defaults;
	unsigned int num_reg_defaults;
	enum regcache_type cache_type;
	const void *reg_defaults_raw;
	unsigned int num_reg_defaults_raw;

	unsigned long read_flag_mask;
	unsigned long write_flag_mask;
	bool zero_flag_mask;

	bool use_single_read;
	bool use_single_write;
	bool can_multi_write;

	enum regmap_endian reg_format_endian;
	enum regmap_endian val_format_endian;

	const struct regmap_range_cfg *ranges;
	unsigned int num_ranges;

	bool use_hwlock;
	unsigned int hwlock_id;
	unsigned int hwlock_mode;

	bool can_sleep;
};
```

在 `struct regmap_config` 中，驱动程序除了可以用 `writeable_reg` 等回调函数来描述设备 IO 寄存器的读写属性外，还可以用 `struct regmap_access_table` 来描述。如果具有相同读写属性的设备 IO 寄存器是连续的，则 `struct regmap_access_table` 要方便很多，否则用回调函数比较方便。

Linux 设备驱动程序，可以通过 `reg_read` 和 `reg_write` 字段配置设备 IO 寄存器的读写操作。对于 I2C、SPI、I3C、AC97 和 mmio 等标准总线，Linux 内核已经提供相应的设备 IO 寄存器的读写操作。对于比较特别的设备，设备驱动程序可以自行提供这些操作。

设备驱动程序通过 `devm_regmap_init_mmio()` 创建并初始化 `struct regmap`，`devm_regmap_init_mmio()` 定义 (位于 *include/linux/regmap.h*) 如下：
```
struct regmap *__devm_regmap_init_mmio_clk(struct device *dev,
					   const char *clk_id,
					   void __iomem *regs,
					   const struct regmap_config *config,
					   struct lock_class_key *lock_key,
					   const char *lock_name);
 . . . . . .
#ifdef CONFIG_LOCKDEP
#define __regmap_lockdep_wrapper(fn, name, ...)				\
(									\
	({								\
		static struct lock_class_key _key;			\
		fn(__VA_ARGS__, &_key,					\
			KBUILD_BASENAME ":"				\
			__stringify(__LINE__) ":"			\
			"(" name ")->lock");				\
	})								\
)
#else
#define __regmap_lockdep_wrapper(fn, name, ...) fn(__VA_ARGS__, NULL, NULL)
#endif
 . . . . . .
#define devm_regmap_init_mmio_clk(dev, clk_id, regs, config)		\
	__regmap_lockdep_wrapper(__devm_regmap_init_mmio_clk, #config,	\
				dev, clk_id, regs, config)

/**
 * devm_regmap_init_mmio() - Initialise managed register map
 *
 * @dev: Device that will be interacted with
 * @regs: Pointer to memory-mapped IO region
 * @config: Configuration for register map
 *
 * The return value will be an ERR_PTR() on error or a valid pointer
 * to a struct regmap.  The regmap will be automatically freed by the
 * device management code.
 */
#define devm_regmap_init_mmio(dev, regs, config)		\
	devm_regmap_init_mmio_clk(dev, NULL, regs, config)
```

创建并初始化 `struct regmap` 的工作由 `__regmap_init_mmio_clk()` 函数执行，这个函数定义 (位于 *drivers/base/regmap/regmap-mmio.c*) 如下：
```
struct regmap_mmio_context {
	void __iomem *regs;
	unsigned val_bytes;

	bool attached_clk;
	struct clk *clk;

	void (*reg_write)(struct regmap_mmio_context *ctx,
			  unsigned int reg, unsigned int val);
	unsigned int (*reg_read)(struct regmap_mmio_context *ctx,
			         unsigned int reg);
};

static int regmap_mmio_regbits_check(size_t reg_bits)
{
	switch (reg_bits) {
	case 8:
	case 16:
	case 32:
#ifdef CONFIG_64BIT
	case 64:
#endif
		return 0;
	default:
		return -EINVAL;
	}
}

static int regmap_mmio_get_min_stride(size_t val_bits)
{
	int min_stride;

	switch (val_bits) {
	case 8:
		/* The core treats 0 as 1 */
		min_stride = 0;
		return 0;
	case 16:
		min_stride = 2;
		break;
	case 32:
		min_stride = 4;
		break;
#ifdef CONFIG_64BIT
	case 64:
		min_stride = 8;
		break;
#endif
	default:
		return -EINVAL;
	}

	return min_stride;
}
 . . . . . .
static void regmap_mmio_write32le(struct regmap_mmio_context *ctx,
				  unsigned int reg,
				  unsigned int val)
{
	writel(val, ctx->regs + reg);
}

static void regmap_mmio_write32be(struct regmap_mmio_context *ctx,
				  unsigned int reg,
				  unsigned int val)
{
	iowrite32be(val, ctx->regs + reg);
}

#ifdef CONFIG_64BIT
static void regmap_mmio_write64le(struct regmap_mmio_context *ctx,
				  unsigned int reg,
				  unsigned int val)
{
	writeq(val, ctx->regs + reg);
}
#endif
 . . . . . .
static unsigned int regmap_mmio_read32le(struct regmap_mmio_context *ctx,
				         unsigned int reg)
{
	return readl(ctx->regs + reg);
}

static unsigned int regmap_mmio_read32be(struct regmap_mmio_context *ctx,
				         unsigned int reg)
{
	return ioread32be(ctx->regs + reg);
}

#ifdef CONFIG_64BIT
static unsigned int regmap_mmio_read64le(struct regmap_mmio_context *ctx,
				         unsigned int reg)
{
	return readq(ctx->regs + reg);
}
#endif
 . . . . . .
static struct regmap_mmio_context *regmap_mmio_gen_context(struct device *dev,
					const char *clk_id,
					void __iomem *regs,
					const struct regmap_config *config)
{
	struct regmap_mmio_context *ctx;
	int min_stride;
	int ret;

	ret = regmap_mmio_regbits_check(config->reg_bits);
	if (ret)
		return ERR_PTR(ret);

	if (config->pad_bits)
		return ERR_PTR(-EINVAL);

	min_stride = regmap_mmio_get_min_stride(config->val_bits);
	if (min_stride < 0)
		return ERR_PTR(min_stride);

	if (config->reg_stride < min_stride)
		return ERR_PTR(-EINVAL);

	ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
	if (!ctx)
		return ERR_PTR(-ENOMEM);

	ctx->regs = regs;
	ctx->val_bytes = config->val_bits / 8;
	ctx->clk = ERR_PTR(-ENODEV);

	switch (regmap_get_val_endian(dev, &regmap_mmio, config)) {
	case REGMAP_ENDIAN_DEFAULT:
	case REGMAP_ENDIAN_LITTLE:
#ifdef __LITTLE_ENDIAN
	case REGMAP_ENDIAN_NATIVE:
#endif
		switch (config->val_bits) {
		case 8:
			ctx->reg_read = regmap_mmio_read8;
			ctx->reg_write = regmap_mmio_write8;
			break;
		case 16:
			ctx->reg_read = regmap_mmio_read16le;
			ctx->reg_write = regmap_mmio_write16le;
			break;
		case 32:
			ctx->reg_read = regmap_mmio_read32le;
			ctx->reg_write = regmap_mmio_write32le;
			break;
#ifdef CONFIG_64BIT
		case 64:
			ctx->reg_read = regmap_mmio_read64le;
			ctx->reg_write = regmap_mmio_write64le;
			break;
#endif
		default:
			ret = -EINVAL;
			goto err_free;
		}
		break;
	case REGMAP_ENDIAN_BIG:
#ifdef __BIG_ENDIAN
	case REGMAP_ENDIAN_NATIVE:
#endif
		switch (config->val_bits) {
		case 8:
			ctx->reg_read = regmap_mmio_read8;
			ctx->reg_write = regmap_mmio_write8;
			break;
		case 16:
			ctx->reg_read = regmap_mmio_read16be;
			ctx->reg_write = regmap_mmio_write16be;
			break;
		case 32:
			ctx->reg_read = regmap_mmio_read32be;
			ctx->reg_write = regmap_mmio_write32be;
			break;
		default:
			ret = -EINVAL;
			goto err_free;
		}
		break;
	default:
		ret = -EINVAL;
		goto err_free;
	}

	if (clk_id == NULL)
		return ctx;

	ctx->clk = clk_get(dev, clk_id);
	if (IS_ERR(ctx->clk)) {
		ret = PTR_ERR(ctx->clk);
		goto err_free;
	}

	ret = clk_prepare(ctx->clk);
	if (ret < 0) {
		clk_put(ctx->clk);
		goto err_free;
	}

	return ctx;

err_free:
	kfree(ctx);

	return ERR_PTR(ret);
}
 . . . . . .
struct regmap *__devm_regmap_init_mmio_clk(struct device *dev,
					   const char *clk_id,
					   void __iomem *regs,
					   const struct regmap_config *config,
					   struct lock_class_key *lock_key,
					   const char *lock_name)
{
	struct regmap_mmio_context *ctx;

	ctx = regmap_mmio_gen_context(dev, clk_id, regs, config);
	if (IS_ERR(ctx))
		return ERR_CAST(ctx);

	return __devm_regmap_init(dev, &regmap_mmio, ctx, config,
				  lock_key, lock_name);
}
EXPORT_SYMBOL_GPL(__devm_regmap_init_mmio_clk);
```

最终读写设备 IO 寄存器的操作由 context 提供，如这里的 `struct regmap_mmio_context`。可以将 context 看作是 `struct regmap_bus` 的实现。

`__regmap_init_mmio_clk()` 函数通过 `regmap_mmio_gen_context()` 函数分配并初始化 context，并调用 `__devm_regmap_init()` 函数分配并初始化 `struct regmap`。

`regmap_mmio_gen_context()` 函数检查寄存器映射配置，并根据配置的寄存器地址的位宽，值的位宽，及尾端是大尾端还是小尾端，选择适当的读写操作函数。大尾端不支持 64 位的寄存器读写。最终的读写操作，由各个硬件平台特有的 IO 操作完成。

`struct regmap_bus` 表示寄存器映射总线，这个结构体定义 (位于 *include/linux/regmap.h*) 如下：
```
struct regmap_async;

typedef int (*regmap_hw_write)(void *context, const void *data,
			       size_t count);
typedef int (*regmap_hw_gather_write)(void *context,
				      const void *reg, size_t reg_len,
				      const void *val, size_t val_len);
typedef int (*regmap_hw_async_write)(void *context,
				     const void *reg, size_t reg_len,
				     const void *val, size_t val_len,
				     struct regmap_async *async);
typedef int (*regmap_hw_read)(void *context,
			      const void *reg_buf, size_t reg_size,
			      void *val_buf, size_t val_size);
typedef int (*regmap_hw_reg_read)(void *context, unsigned int reg,
				  unsigned int *val);
typedef int (*regmap_hw_reg_write)(void *context, unsigned int reg,
				   unsigned int val);
typedef int (*regmap_hw_reg_update_bits)(void *context, unsigned int reg,
					 unsigned int mask, unsigned int val);
typedef struct regmap_async *(*regmap_hw_async_alloc)(void);
typedef void (*regmap_hw_free_context)(void *context);
 . . . . . .
struct regmap_bus {
	bool fast_io;
	regmap_hw_write write;
	regmap_hw_gather_write gather_write;
	regmap_hw_async_write async_write;
	regmap_hw_reg_write reg_write;
	regmap_hw_reg_update_bits reg_update_bits;
	regmap_hw_read read;
	regmap_hw_reg_read reg_read;
	regmap_hw_free_context free_context;
	regmap_hw_async_alloc async_alloc;
	u8 read_flag_mask;
	enum regmap_endian reg_format_endian_default;
	enum regmap_endian val_format_endian_default;
	size_t max_raw_read;
	size_t max_raw_write;
};
```

`struct regmap_bus` 基本上是各种设备寄存器 IO 操作的集合。对于 mmio，它的 `struct regmap_bus` 定义 (位于 *drivers/base/regmap/regmap-mmio.c*) 如下：
```
static int regmap_mmio_write(void *context, unsigned int reg, unsigned int val)
{
	struct regmap_mmio_context *ctx = context;
	int ret;

	if (!IS_ERR(ctx->clk)) {
		ret = clk_enable(ctx->clk);
		if (ret < 0)
			return ret;
	}

	ctx->reg_write(ctx, reg, val);

	if (!IS_ERR(ctx->clk))
		clk_disable(ctx->clk);

	return 0;
}
 . . . . . .
static int regmap_mmio_read(void *context, unsigned int reg, unsigned int *val)
{
	struct regmap_mmio_context *ctx = context;
	int ret;

	if (!IS_ERR(ctx->clk)) {
		ret = clk_enable(ctx->clk);
		if (ret < 0)
			return ret;
	}

	*val = ctx->reg_read(ctx, reg);

	if (!IS_ERR(ctx->clk))
		clk_disable(ctx->clk);

	return 0;
}

static void regmap_mmio_free_context(void *context)
{
	struct regmap_mmio_context *ctx = context;

	if (!IS_ERR(ctx->clk)) {
		clk_unprepare(ctx->clk);
		if (!ctx->attached_clk)
			clk_put(ctx->clk);
	}
	kfree(context);
}

static const struct regmap_bus regmap_mmio = {
	.fast_io = true,
	.reg_write = regmap_mmio_write,
	.reg_read = regmap_mmio_read,
	.free_context = regmap_mmio_free_context,
	.val_format_endian_default = REGMAP_ENDIAN_LITTLE,
};
```

在 **regmap** 机制中，`struct regmap_bus` 的角色即是完成对硬件设备 IO 寄存器的直接访问者。对于 mmio，`struct regmap_bus` 是通向 context `struct regmap_mmio_context` 的桥梁，其访问设备 IO 寄存器的动作由 context 完成。

`__devm_regmap_init()` 函数定义 (位于 *drivers/base/regmap/regmap.c*) 如下：
```
int regmap_attach_dev(struct device *dev, struct regmap *map,
		      const struct regmap_config *config)
{
	struct regmap **m;
	int ret;

	map->dev = dev;

	ret = regmap_set_name(map, config);
	if (ret)
		return ret;

	regmap_debugfs_exit(map);
	regmap_debugfs_init(map);

	/* Add a devres resource for dev_get_regmap() */
	m = devres_alloc(dev_get_regmap_release, sizeof(*m), GFP_KERNEL);
	if (!m) {
		regmap_debugfs_exit(map);
		return -ENOMEM;
	}
	*m = map;
	devres_add(dev, m);

	return 0;
}
EXPORT_SYMBOL_GPL(regmap_attach_dev);
 . . . . . .
struct regmap *__regmap_init(struct device *dev,
			     const struct regmap_bus *bus,
			     void *bus_context,
			     const struct regmap_config *config,
			     struct lock_class_key *lock_key,
			     const char *lock_name)
{
	struct regmap *map;
	int ret = -EINVAL;
	enum regmap_endian reg_endian, val_endian;
	int i, j;

	if (!config)
		goto err;

	map = kzalloc(sizeof(*map), GFP_KERNEL);
	if (map == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	ret = regmap_set_name(map, config);
	if (ret)
		goto err_map;

	ret = -EINVAL; /* Later error paths rely on this */

	if (config->disable_locking) {
		map->lock = map->unlock = regmap_lock_unlock_none;
		map->can_sleep = config->can_sleep;
		regmap_debugfs_disable(map);
	} else if (config->lock && config->unlock) {
		map->lock = config->lock;
		map->unlock = config->unlock;
		map->lock_arg = config->lock_arg;
		map->can_sleep = config->can_sleep;
	} else if (config->use_hwlock) {
		map->hwlock = hwspin_lock_request_specific(config->hwlock_id);
		if (!map->hwlock) {
			ret = -ENXIO;
			goto err_name;
		}

		switch (config->hwlock_mode) {
		case HWLOCK_IRQSTATE:
			map->lock = regmap_lock_hwlock_irqsave;
			map->unlock = regmap_unlock_hwlock_irqrestore;
			break;
		case HWLOCK_IRQ:
			map->lock = regmap_lock_hwlock_irq;
			map->unlock = regmap_unlock_hwlock_irq;
			break;
		default:
			map->lock = regmap_lock_hwlock;
			map->unlock = regmap_unlock_hwlock;
			break;
		}

		map->lock_arg = map;
	} else {
		if ((bus && bus->fast_io) ||
		    config->fast_io) {
			spin_lock_init(&map->spinlock);
			map->lock = regmap_lock_spinlock;
			map->unlock = regmap_unlock_spinlock;
			lockdep_set_class_and_name(&map->spinlock,
						   lock_key, lock_name);
		} else {
			mutex_init(&map->mutex);
			map->lock = regmap_lock_mutex;
			map->unlock = regmap_unlock_mutex;
			map->can_sleep = true;
			lockdep_set_class_and_name(&map->mutex,
						   lock_key, lock_name);
		}
		map->lock_arg = map;
	}

	/*
	 * When we write in fast-paths with regmap_bulk_write() don't allocate
	 * scratch buffers with sleeping allocations.
	 */
	if ((bus && bus->fast_io) || config->fast_io)
		map->alloc_flags = GFP_ATOMIC;
	else
		map->alloc_flags = GFP_KERNEL;

	map->format.reg_bytes = DIV_ROUND_UP(config->reg_bits, 8);
	map->format.pad_bytes = config->pad_bits / 8;
	map->format.val_bytes = DIV_ROUND_UP(config->val_bits, 8);
	map->format.buf_size = DIV_ROUND_UP(config->reg_bits +
			config->val_bits + config->pad_bits, 8);
	map->reg_shift = config->pad_bits % 8;
	if (config->reg_stride)
		map->reg_stride = config->reg_stride;
	else
		map->reg_stride = 1;
	if (is_power_of_2(map->reg_stride))
		map->reg_stride_order = ilog2(map->reg_stride);
	else
		map->reg_stride_order = -1;
	map->use_single_read = config->use_single_read || !bus || !bus->read;
	map->use_single_write = config->use_single_write || !bus || !bus->write;
	map->can_multi_write = config->can_multi_write && bus && bus->write;
	if (bus) {
		map->max_raw_read = bus->max_raw_read;
		map->max_raw_write = bus->max_raw_write;
	}
	map->dev = dev;
	map->bus = bus;
	map->bus_context = bus_context;
	map->max_register = config->max_register;
	map->wr_table = config->wr_table;
	map->rd_table = config->rd_table;
	map->volatile_table = config->volatile_table;
	map->precious_table = config->precious_table;
	map->wr_noinc_table = config->wr_noinc_table;
	map->rd_noinc_table = config->rd_noinc_table;
	map->writeable_reg = config->writeable_reg;
	map->readable_reg = config->readable_reg;
	map->volatile_reg = config->volatile_reg;
	map->precious_reg = config->precious_reg;
	map->writeable_noinc_reg = config->writeable_noinc_reg;
	map->readable_noinc_reg = config->readable_noinc_reg;
	map->cache_type = config->cache_type;

	spin_lock_init(&map->async_lock);
	INIT_LIST_HEAD(&map->async_list);
	INIT_LIST_HEAD(&map->async_free);
	init_waitqueue_head(&map->async_waitq);

	if (config->read_flag_mask ||
	    config->write_flag_mask ||
	    config->zero_flag_mask) {
		map->read_flag_mask = config->read_flag_mask;
		map->write_flag_mask = config->write_flag_mask;
	} else if (bus) {
		map->read_flag_mask = bus->read_flag_mask;
	}

	if (!bus) {
		map->reg_read  = config->reg_read;
		map->reg_write = config->reg_write;

		map->defer_caching = false;
		goto skip_format_initialization;
	} else if (!bus->read || !bus->write) {
		map->reg_read = _regmap_bus_reg_read;
		map->reg_write = _regmap_bus_reg_write;
		map->reg_update_bits = bus->reg_update_bits;

		map->defer_caching = false;
		goto skip_format_initialization;
	} else {
		map->reg_read  = _regmap_bus_read;
		map->reg_update_bits = bus->reg_update_bits;
	}

	reg_endian = regmap_get_reg_endian(bus, config);
	val_endian = regmap_get_val_endian(dev, bus, config);

	switch (config->reg_bits + map->reg_shift) {
	case 2:
		switch (config->val_bits) {
		case 6:
			map->format.format_write = regmap_format_2_6_write;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 4:
		switch (config->val_bits) {
		case 12:
			map->format.format_write = regmap_format_4_12_write;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 7:
		switch (config->val_bits) {
		case 9:
			map->format.format_write = regmap_format_7_9_write;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 10:
		switch (config->val_bits) {
		case 14:
			map->format.format_write = regmap_format_10_14_write;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 12:
		switch (config->val_bits) {
		case 20:
			map->format.format_write = regmap_format_12_20_write;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 8:
		map->format.format_reg = regmap_format_8;
		break;

	case 16:
		switch (reg_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_reg = regmap_format_16_be;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_reg = regmap_format_16_le;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_reg = regmap_format_16_native;
			break;
		default:
			goto err_hwlock;
		}
		break;

	case 24:
		if (reg_endian != REGMAP_ENDIAN_BIG)
			goto err_hwlock;
		map->format.format_reg = regmap_format_24;
		break;

	case 32:
		switch (reg_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_reg = regmap_format_32_be;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_reg = regmap_format_32_le;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_reg = regmap_format_32_native;
			break;
		default:
			goto err_hwlock;
		}
		break;

#ifdef CONFIG_64BIT
	case 64:
		switch (reg_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_reg = regmap_format_64_be;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_reg = regmap_format_64_le;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_reg = regmap_format_64_native;
			break;
		default:
			goto err_hwlock;
		}
		break;
#endif

	default:
		goto err_hwlock;
	}

	if (val_endian == REGMAP_ENDIAN_NATIVE)
		map->format.parse_inplace = regmap_parse_inplace_noop;

	switch (config->val_bits) {
	case 8:
		map->format.format_val = regmap_format_8;
		map->format.parse_val = regmap_parse_8;
		map->format.parse_inplace = regmap_parse_inplace_noop;
		break;
	case 16:
		switch (val_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_val = regmap_format_16_be;
			map->format.parse_val = regmap_parse_16_be;
			map->format.parse_inplace = regmap_parse_16_be_inplace;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_val = regmap_format_16_le;
			map->format.parse_val = regmap_parse_16_le;
			map->format.parse_inplace = regmap_parse_16_le_inplace;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_val = regmap_format_16_native;
			map->format.parse_val = regmap_parse_16_native;
			break;
		default:
			goto err_hwlock;
		}
		break;
	case 24:
		if (val_endian != REGMAP_ENDIAN_BIG)
			goto err_hwlock;
		map->format.format_val = regmap_format_24;
		map->format.parse_val = regmap_parse_24;
		break;
	case 32:
		switch (val_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_val = regmap_format_32_be;
			map->format.parse_val = regmap_parse_32_be;
			map->format.parse_inplace = regmap_parse_32_be_inplace;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_val = regmap_format_32_le;
			map->format.parse_val = regmap_parse_32_le;
			map->format.parse_inplace = regmap_parse_32_le_inplace;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_val = regmap_format_32_native;
			map->format.parse_val = regmap_parse_32_native;
			break;
		default:
			goto err_hwlock;
		}
		break;
#ifdef CONFIG_64BIT
	case 64:
		switch (val_endian) {
		case REGMAP_ENDIAN_BIG:
			map->format.format_val = regmap_format_64_be;
			map->format.parse_val = regmap_parse_64_be;
			map->format.parse_inplace = regmap_parse_64_be_inplace;
			break;
		case REGMAP_ENDIAN_LITTLE:
			map->format.format_val = regmap_format_64_le;
			map->format.parse_val = regmap_parse_64_le;
			map->format.parse_inplace = regmap_parse_64_le_inplace;
			break;
		case REGMAP_ENDIAN_NATIVE:
			map->format.format_val = regmap_format_64_native;
			map->format.parse_val = regmap_parse_64_native;
			break;
		default:
			goto err_hwlock;
		}
		break;
#endif
	}

	if (map->format.format_write) {
		if ((reg_endian != REGMAP_ENDIAN_BIG) ||
		    (val_endian != REGMAP_ENDIAN_BIG))
			goto err_hwlock;
		map->use_single_write = true;
	}

	if (!map->format.format_write &&
	    !(map->format.format_reg && map->format.format_val))
		goto err_hwlock;

	map->work_buf = kzalloc(map->format.buf_size, GFP_KERNEL);
	if (map->work_buf == NULL) {
		ret = -ENOMEM;
		goto err_hwlock;
	}

	if (map->format.format_write) {
		map->defer_caching = false;
		map->reg_write = _regmap_bus_formatted_write;
	} else if (map->format.format_val) {
		map->defer_caching = true;
		map->reg_write = _regmap_bus_raw_write;
	}

skip_format_initialization:

	map->range_tree = RB_ROOT;
	for (i = 0; i < config->num_ranges; i++) {
		const struct regmap_range_cfg *range_cfg = &config->ranges[i];
		struct regmap_range_node *new;

		/* Sanity check */
		if (range_cfg->range_max < range_cfg->range_min) {
			dev_err(map->dev, "Invalid range %d: %d < %d\n", i,
				range_cfg->range_max, range_cfg->range_min);
			goto err_range;
		}

		if (range_cfg->range_max > map->max_register) {
			dev_err(map->dev, "Invalid range %d: %d > %d\n", i,
				range_cfg->range_max, map->max_register);
			goto err_range;
		}

		if (range_cfg->selector_reg > map->max_register) {
			dev_err(map->dev,
				"Invalid range %d: selector out of map\n", i);
			goto err_range;
		}

		if (range_cfg->window_len == 0) {
			dev_err(map->dev, "Invalid range %d: window_len 0\n",
				i);
			goto err_range;
		}

		/* Make sure, that this register range has no selector
		   or data window within its boundary */
		for (j = 0; j < config->num_ranges; j++) {
			unsigned sel_reg = config->ranges[j].selector_reg;
			unsigned win_min = config->ranges[j].window_start;
			unsigned win_max = win_min +
					   config->ranges[j].window_len - 1;

			/* Allow data window inside its own virtual range */
			if (j == i)
				continue;

			if (range_cfg->range_min <= sel_reg &&
			    sel_reg <= range_cfg->range_max) {
				dev_err(map->dev,
					"Range %d: selector for %d in window\n",
					i, j);
				goto err_range;
			}

			if (!(win_max < range_cfg->range_min ||
			      win_min > range_cfg->range_max)) {
				dev_err(map->dev,
					"Range %d: window for %d in window\n",
					i, j);
				goto err_range;
			}
		}

		new = kzalloc(sizeof(*new), GFP_KERNEL);
		if (new == NULL) {
			ret = -ENOMEM;
			goto err_range;
		}

		new->map = map;
		new->name = range_cfg->name;
		new->range_min = range_cfg->range_min;
		new->range_max = range_cfg->range_max;
		new->selector_reg = range_cfg->selector_reg;
		new->selector_mask = range_cfg->selector_mask;
		new->selector_shift = range_cfg->selector_shift;
		new->window_start = range_cfg->window_start;
		new->window_len = range_cfg->window_len;

		if (!_regmap_range_add(map, new)) {
			dev_err(map->dev, "Failed to add range %d\n", i);
			kfree(new);
			goto err_range;
		}

		if (map->selector_work_buf == NULL) {
			map->selector_work_buf =
				kzalloc(map->format.buf_size, GFP_KERNEL);
			if (map->selector_work_buf == NULL) {
				ret = -ENOMEM;
				goto err_range;
			}
		}
	}

	ret = regcache_init(map, config);
	if (ret != 0)
		goto err_range;

	if (dev) {
		ret = regmap_attach_dev(dev, map, config);
		if (ret != 0)
			goto err_regcache;
	} else {
		regmap_debugfs_init(map);
	}

	return map;

err_regcache:
	regcache_exit(map);
err_range:
	regmap_range_exit(map);
	kfree(map->work_buf);
err_hwlock:
	if (map->hwlock)
		hwspin_lock_free(map->hwlock);
err_name:
	kfree_const(map->name);
err_map:
	kfree(map);
err:
	return ERR_PTR(ret);
}
EXPORT_SYMBOL_GPL(__regmap_init);

static void devm_regmap_release(struct device *dev, void *res)
{
	regmap_exit(*(struct regmap **)res);
}

struct regmap *__devm_regmap_init(struct device *dev,
				  const struct regmap_bus *bus,
				  void *bus_context,
				  const struct regmap_config *config,
				  struct lock_class_key *lock_key,
				  const char *lock_name)
{
	struct regmap **ptr, *regmap;

	ptr = devres_alloc(devm_regmap_release, sizeof(*ptr), GFP_KERNEL);
	if (!ptr)
		return ERR_PTR(-ENOMEM);

	regmap = __regmap_init(dev, bus, bus_context, config,
			       lock_key, lock_name);
	if (!IS_ERR(regmap)) {
		*ptr = regmap;
		devres_add(dev, ptr);
	} else {
		devres_free(ptr);
	}

	return regmap;
}
EXPORT_SYMBOL_GPL(__devm_regmap_init);
```

`__devm_regmap_init()` 函数通过 `__regmap_init()` 函数创建及初始化 `struct regmap` 对象，并创建 devres 来维护 `struct regmap` 对象的生命周期，使得它的生命周期可以随着 `struct device` 对象生命周期的结束而结束。

`__regmap_init()` 函数看上去比较长，但它做的事情比较清晰：

1. 分配 `struct regmap` 对象。
2. 根据寄存器映射配置选择 `lock`/`unlock` 操作。
3. 根据传入的寄存器映射配置、`regmap_bus` 和 `bus_context` 等初始化 `struct regmap` 对象。
4. 根据传入的寄存器映射配置和 `regmap_bus` 等选择设备 IO 寄存器的读写操作。
5. 根据传入的寄存器映射配置选择格式化的寄存器操作。
6. 处理寄存器映射范围。
7. 初始化寄存器映射缓存。
8. 连接 dev，这包括初始化 debugfs 等。

`regcache_init()` 函数初始化寄存器映射缓存，它的定义 (位于 *drivers/base/regmap/regcache.c*) 如下：
```
static const struct regcache_ops *cache_types[] = {
	&regcache_rbtree_ops,
#if IS_ENABLED(CONFIG_REGCACHE_COMPRESSED)
	&regcache_lzo_ops,
#endif
	&regcache_flat_ops,
};
 . . . . . .
int regcache_init(struct regmap *map, const struct regmap_config *config)
{
	int ret;
	int i;
	void *tmp_buf;

	if (map->cache_type == REGCACHE_NONE) {
		if (config->reg_defaults || config->num_reg_defaults_raw)
			dev_warn(map->dev,
				 "No cache used with register defaults set!\n");

		map->cache_bypass = true;
		return 0;
	}

	if (config->reg_defaults && !config->num_reg_defaults) {
		dev_err(map->dev,
			 "Register defaults are set without the number!\n");
		return -EINVAL;
	}

	for (i = 0; i < config->num_reg_defaults; i++)
		if (config->reg_defaults[i].reg % map->reg_stride)
			return -EINVAL;

	for (i = 0; i < ARRAY_SIZE(cache_types); i++)
		if (cache_types[i]->type == map->cache_type)
			break;

	if (i == ARRAY_SIZE(cache_types)) {
		dev_err(map->dev, "Could not match compress type: %d\n",
			map->cache_type);
		return -EINVAL;
	}

	map->num_reg_defaults = config->num_reg_defaults;
	map->num_reg_defaults_raw = config->num_reg_defaults_raw;
	map->reg_defaults_raw = config->reg_defaults_raw;
	map->cache_word_size = DIV_ROUND_UP(config->val_bits, 8);
	map->cache_size_raw = map->cache_word_size * config->num_reg_defaults_raw;

	map->cache = NULL;
	map->cache_ops = cache_types[i];

	if (!map->cache_ops->read ||
	    !map->cache_ops->write ||
	    !map->cache_ops->name)
		return -EINVAL;

	/* We still need to ensure that the reg_defaults
	 * won't vanish from under us.  We'll need to make
	 * a copy of it.
	 */
	if (config->reg_defaults) {
		tmp_buf = kmemdup(config->reg_defaults, map->num_reg_defaults *
				  sizeof(struct reg_default), GFP_KERNEL);
		if (!tmp_buf)
			return -ENOMEM;
		map->reg_defaults = tmp_buf;
	} else if (map->num_reg_defaults_raw) {
		/* Some devices such as PMICs don't have cache defaults,
		 * we cope with this by reading back the HW registers and
		 * crafting the cache defaults by hand.
		 */
		ret = regcache_hw_init(map);
		if (ret < 0)
			return ret;
		if (map->cache_bypass)
			return 0;
	}

	if (!map->max_register)
		map->max_register = map->num_reg_defaults_raw;

	if (map->cache_ops->init) {
		dev_dbg(map->dev, "Initializing %s cache\n",
			map->cache_ops->name);
		ret = map->cache_ops->init(map);
		if (ret)
			goto err_free;
	}
	return 0;

err_free:
	kfree(map->reg_defaults);
	if (map->cache_free)
		kfree(map->reg_defaults_raw);

	return ret;
}
```

这个函数根据传入的缓存类型配置，选择适当的 `struct regcache_ops`，并初始化缓存。前面我们看到的，寄存器映射配置中，有默认值的寄存器的默认值配置，在这里被复制一份并保存起来。不同缓存策略对于这些默认值的处理可能不同。如对于 `REGCACHE_FLAT` 缓存策略，这些默认值在缓存策略初始化过程中，会被用来初始化缓存，如下面代码 (位于 *drivers/base/regmap/regcache-flat.c*) 所示：
```
static int regcache_flat_init(struct regmap *map)
{
	int i;
	unsigned int *cache;

	if (!map || map->reg_stride_order < 0 || !map->max_register)
		return -EINVAL;

	map->cache = kcalloc(regcache_flat_get_index(map, map->max_register)
			     + 1, sizeof(unsigned int), GFP_KERNEL);
	if (!map->cache)
		return -ENOMEM;

	cache = map->cache;

	for (i = 0; i < map->num_reg_defaults; i++) {
		unsigned int reg = map->reg_defaults[i].reg;
		unsigned int index = regcache_flat_get_index(map, reg);

		cache[index] = map->reg_defaults[i].def;
	}

	return 0;
}
```

`struct regcache_ops` 定义 (位于 *drivers/base/regmap/internal.h*) 如下：
```
struct regcache_ops {
	const char *name;
	enum regcache_type type;
	int (*init)(struct regmap *map);
	int (*exit)(struct regmap *map);
#ifdef CONFIG_DEBUG_FS
	void (*debugfs_init)(struct regmap *map);
#endif
	int (*read)(struct regmap *map, unsigned int reg, unsigned int *value);
	int (*write)(struct regmap *map, unsigned int reg, unsigned int value);
	int (*sync)(struct regmap *map, unsigned int min, unsigned int max);
	int (*drop)(struct regmap *map, unsigned int min, unsigned int max);
};
```

`struct regcache_ops` 包含多个缓存操作。

`struct regmap` 定义 (位于 *drivers/base/regmap/internal.h*) 如下：
```
struct regmap {
	union {
		struct mutex mutex;
		struct {
			spinlock_t spinlock;
			unsigned long spinlock_flags;
		};
	};
	regmap_lock lock;
	regmap_unlock unlock;
	void *lock_arg; /* This is passed to lock/unlock functions */
	gfp_t alloc_flags;

	struct device *dev; /* Device we do I/O on */
	void *work_buf;     /* Scratch buffer used to format I/O */
	struct regmap_format format;  /* Buffer format */
	const struct regmap_bus *bus;
	void *bus_context;
	const char *name;

	bool async;
	spinlock_t async_lock;
	wait_queue_head_t async_waitq;
	struct list_head async_list;
	struct list_head async_free;
	int async_ret;

#ifdef CONFIG_DEBUG_FS
	bool debugfs_disable;
	struct dentry *debugfs;
	const char *debugfs_name;

	unsigned int debugfs_reg_len;
	unsigned int debugfs_val_len;
	unsigned int debugfs_tot_len;

	struct list_head debugfs_off_cache;
	struct mutex cache_lock;
#endif

	unsigned int max_register;
	bool (*writeable_reg)(struct device *dev, unsigned int reg);
	bool (*readable_reg)(struct device *dev, unsigned int reg);
	bool (*volatile_reg)(struct device *dev, unsigned int reg);
	bool (*precious_reg)(struct device *dev, unsigned int reg);
	bool (*writeable_noinc_reg)(struct device *dev, unsigned int reg);
	bool (*readable_noinc_reg)(struct device *dev, unsigned int reg);
	const struct regmap_access_table *wr_table;
	const struct regmap_access_table *rd_table;
	const struct regmap_access_table *volatile_table;
	const struct regmap_access_table *precious_table;
	const struct regmap_access_table *wr_noinc_table;
	const struct regmap_access_table *rd_noinc_table;

	int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
	int (*reg_write)(void *context, unsigned int reg, unsigned int val);
	int (*reg_update_bits)(void *context, unsigned int reg,
			       unsigned int mask, unsigned int val);

	bool defer_caching;

	unsigned long read_flag_mask;
	unsigned long write_flag_mask;

	/* number of bits to (left) shift the reg value when formatting*/
	int reg_shift;
	int reg_stride;
	int reg_stride_order;

	/* regcache specific members */
	const struct regcache_ops *cache_ops;
	enum regcache_type cache_type;

	/* number of bytes in reg_defaults_raw */
	unsigned int cache_size_raw;
	/* number of bytes per word in reg_defaults_raw */
	unsigned int cache_word_size;
	/* number of entries in reg_defaults */
	unsigned int num_reg_defaults;
	/* number of entries in reg_defaults_raw */
	unsigned int num_reg_defaults_raw;

	/* if set, only the cache is modified not the HW */
	bool cache_only;
	/* if set, only the HW is modified not the cache */
	bool cache_bypass;
	/* if set, remember to free reg_defaults_raw */
	bool cache_free;

	struct reg_default *reg_defaults;
	const void *reg_defaults_raw;
	void *cache;
	/* if set, the cache contains newer data than the HW */
	bool cache_dirty;
	/* if set, the HW registers are known to match map->reg_defaults */
	bool no_sync_defaults;

	struct reg_sequence *patch;
	int patch_regs;

	/* if set, converts bulk read to single read */
	bool use_single_read;
	/* if set, converts bulk write to single write */
	bool use_single_write;
	/* if set, the device supports multi write mode */
	bool can_multi_write;

	/* if set, raw reads/writes are limited to this size */
	size_t max_raw_read;
	size_t max_raw_write;

	struct rb_root range_tree;
	void *selector_work_buf;	/* Scratch buffer used for selector */

	struct hwspinlock *hwlock;

	/* if set, the regmap core can sleep */
	bool can_sleep;
};
```

`struct regmap` 是 `struct regmap_config`、`struct regmap_bus` 和 cache 三者的结合，它通过 `struct regmap_config` 连接具体设备的寄存器的信息，通过 `struct regmap_bus` 连接对具体设备的 IO 寄存器的读写操作，通过 cache 连接缓存。

Done.
