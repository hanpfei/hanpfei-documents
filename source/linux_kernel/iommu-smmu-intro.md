---
title: IOMMU 和 ARM SMMU 介绍
date: 2023-10-17 21:37:29
categories: Linux 内核
tags:
- Linux 内核
---

## IOMMU 介绍

在计算中，输入-输出内存管理单元 (IOMMU) 是把能够执行直接内存存取 (DMA-capable) 的 I/O 总线连接到物理内存的内存管理单元 (MMU)。像传统的 MMU 一样，IOMMU 将设备可见的虚拟地址 (也称为 I/O 虚拟地址，IOVA) 映射到物理地址 (PA)。不同的平台具有不同的 IOMMU，比如 Intel IOMMU —— PCI Express 显卡使用的图形地址重映射表 (GART)，和 ARM 平台使用的系统内存管理单元 (SMMU)。

CPU 和设备以如下方式访问物理内存：
```
    +---------------------+
    |      Main Memory    |
    +---------------------+
               |
              pa
               |
         -------------
         |           |
    +--------+   +--------+
    | IOMMU  |   |  MMU   |
    +--------+   +--------+
         |           |
       iova          va
         |           |
    +--------+   +--------+
    | Device |   |  CPU   |
    +--------+   +--------+
```

相对于 DMA，IOMMU 具有如下的优势：

 * 可以分配大块内存区域，而无需在物理内存中连续。IOMMU 可以将连续的虚拟地址 (VAs) 映射到碎片化的物理地址 PA。

 * 不支持足够长的内存地址来寻址整个物理内存的设备仍然可以通过 IOMMU 寻址整个内存。比如，X86 计算机可以通过物理地址扩展 (PAE) 功能寻址超过 4GB 的内存。但普通的 32 位 PCI 设备无法寻址 4 GB 以上的内存。通过 IOMMU，设备可以寻址整个物理内存。

 * 内存受到保护，免受尝试 DMA 攻击的恶意设备和尝试错误内存传输的故障设备的影响，因为设备无法读取或写入映射的物理内存。

 * 在虚拟化中，Guest OS 可以使用不是专门为虚拟化设计的硬件。更高性能的硬件（例如显卡）使用 DMA 直接访问内存。在虚拟环境中，所有内存地址都会被虚拟化软件（例如 QEMU）重新映射，这会导致 guest OS 无法使用 DMA 访问内存。IOMMU 处理重映射，允许驱动在 guest OS 中使用 DMA 访问内存。

 * 在某些架构中 IOMMU 也可以以与地址重映射类似的方式，执行中断重映射。

 * IOMMU 可支持外设内存分页。使用 PCI-SIG PCIe 地址转换服务 (ATS) 页请求接口 (PRI) 扩展的外设可以检测内存管理器服务的需要并发出信号。

相对于 DMA，使用 IOMMU 的劣势是额外的性能和内存开销。地址转换和页错误处理增加性能开销。此外，IOMMU 需要在内存中为 I/O 页表分配空间。在某些情况下，IOMMU 和 CPU 共享页表，以避免这种内存开销。比如，设备和 CPU 共享虚拟地址。

## ARM SMMU 数据结构

SMMU 提供使用设备可见的 IOVA 访问物理内存的能力。在系统架构中，多个设备可以使用 IOVA 通过 IOMMU 访问物理内存。IOMMU 需要区分不同的设备，因此每个设备都被分配了一个流 ID (SID)，它指向对应的流表项 (STE)。所有 STE 以数组形式存在于内存中。SMMU 记录 STE 数组的起始地址。当扫描设备时，OS 给设备分配一个唯一的 SID。设备通过 IOMMU 访问内存的所有配置被写入 SID 对应的 STE。

流表如下：
```
                  +-------+
strtab_base ----- | STE 0 |
                  | STE 1 |
StreamID[n:0] ->  | STE 2 |
                  | STE 3 |
                  +-------+
```

STE 存储了从 IOVA 到 PA 的地址转换过程。为了适应虚拟化场景下的内存访问需求，SMMU 支持两级地址转换，类似于扩展页表（EPT）。第 1 阶段将 VA 转换为中间物理地址 (IPA)，第 2 阶段将 IPA 转换为 PA。

STE 如下：
```
Stream Table Entry (STE)
+-----------------------+
| Config | S1ContextPtr | -> CD -> Stage 1 translation tables
+-----------------------+
|  VMID  | S2TTB        | -> Stage 2 translation tables
+-----------------------+
| Other attributes,     |
| configuration         |
+-----------------------+
```

在非虚拟化场景下，如果设备使用 IOVA 通过 IOMMU 进行 DMA，则只需要第一阶段地址转换。因为多个设备可能使用一个设备，所以每个设备的 STE 还记录了上下文描述符（CD）表的信息，S1ContextPtr 指向内存中 CD 表的基地址。CD 表也是一个数组，并使用 SubstreamID（SSID 或 PASID）进行内存访问。PASID，与进程关联的ID，用于区分不同进程的 VA 空间。使用 PASID 找到 CD 条目后，找到第一阶段地址转换的 I/O 页表，并将其存储在 TTB0 和 TTB1 中。

CD 如下：
```
                  +-------+
S1ContextPtr ---- |  CD 0 |
                  |  CD 1 |
SubStreamID   ->  |  CD 2 |
                  |  CD 3 |
                  +-------+

Context Desctriptor (CD)
+-----------------------+
| Configuration | TTB0  |
+-----------------------+
|      ASID     | TTB1  |
+-----------------------+
```

需要说明的是，通常，该进程通过设备驱动程序，由内核态驱动程序分配设备进行 DMA 时所使用的 IOVA。当有多个进程时，内核态 IOVA 被映射到进程的 VA 空间，即不同进程在 DMA 时实际上使用相同的 IOVA 空间。因此，只需要 CD 0。一般情况下，CD 0 中只需要存储 I/O 页表的基地址。但当设备的 I/O 地址空间和进程的 VA 空间需要统一时，例如访问共享虚拟地址（SVA），则使用多个 CD 条目分别绑定不同进程的 VA 空间 。

设备通过 SMMU 转换地址是一个复杂的过程。第一步是通过使用设备 SID 定位 STE。STE 记录了第一阶段地址转换是否需要绕过。绕过意味着直接使用 PA 或 IPA。如果不绕过，则使用 SID 来定位关联的 CD，它记录了用于将 VA 转换为 IPA 的第一阶段地址转换的页表。如果在 STE 中配置了第二阶段页表转换，则 IPA 将被转换为 PA。如果没有配置第二阶段地址转换，则之前获取的 IPA 就是 PA。

SMMU 地址转换过程如下：
```
                         VA
                          |
                    -----------
                    |         |
+---------------------+       |
| Stage 1 translation |    Bypass
|        VA->IPA      |       |
+---------------------+       |
                    |         |
                    -----------
                          |
                         IPA
                          |
                    -----------
                    |         |
+---------------------+       |
| Stage 2 translation |    Bypass
|        IPA->VA      |       |
+---------------------+       |
                    |         |
                    -----------
                          |
                         PA
```

## ARM SMMUv3 初始化

所有 IOMMU 相关的驱动程序都保存在内核的 **drivers/iommu** 目录下。最新的 SMMUv3 驱动程序是 [arm-smmu-v3.c](https://elixir.bootlin.com/linux/v6.5.7/source/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c)。**struct arm_smmu_device** 结构体管理内存中关于 SMMU 平台设备的关键信息。内核通过填充这个结构体初始化 SMMU 设备。
```
/* An SMMUv3 instance */
struct arm_smmu_device {
	struct device			*dev;
	void __iomem			*base;
	void __iomem			*page1;

#define ARM_SMMU_FEAT_2_LVL_STRTAB	(1 << 0)
#define ARM_SMMU_FEAT_2_LVL_CDTAB	(1 << 1)
#define ARM_SMMU_FEAT_TT_LE		(1 << 2)
#define ARM_SMMU_FEAT_TT_BE		(1 << 3)
#define ARM_SMMU_FEAT_PRI		(1 << 4)
#define ARM_SMMU_FEAT_ATS		(1 << 5)
#define ARM_SMMU_FEAT_SEV		(1 << 6)
#define ARM_SMMU_FEAT_MSI		(1 << 7)
#define ARM_SMMU_FEAT_COHERENCY		(1 << 8)
#define ARM_SMMU_FEAT_TRANS_S1		(1 << 9)
#define ARM_SMMU_FEAT_TRANS_S2		(1 << 10)
#define ARM_SMMU_FEAT_STALLS		(1 << 11)
#define ARM_SMMU_FEAT_HYP		(1 << 12)
#define ARM_SMMU_FEAT_STALL_FORCE	(1 << 13)
#define ARM_SMMU_FEAT_VAX		(1 << 14)
#define ARM_SMMU_FEAT_RANGE_INV		(1 << 15)
#define ARM_SMMU_FEAT_BTM		(1 << 16)
#define ARM_SMMU_FEAT_SVA		(1 << 17)
#define ARM_SMMU_FEAT_E2H		(1 << 18)
#define ARM_SMMU_FEAT_NESTING		(1 << 19)
	u32				features;

#define ARM_SMMU_OPT_SKIP_PREFETCH	(1 << 0)
#define ARM_SMMU_OPT_PAGE0_REGS_ONLY	(1 << 1)
#define ARM_SMMU_OPT_MSIPOLL		(1 << 2)
#define ARM_SMMU_OPT_CMDQ_FORCE_SYNC	(1 << 3)
	u32				options;

	struct arm_smmu_cmdq		cmdq;
	struct arm_smmu_evtq		evtq;
	struct arm_smmu_priq		priq;

	int				gerr_irq;
	int				combined_irq;

	unsigned long			ias; /* IPA */
	unsigned long			oas; /* PA */
	unsigned long			pgsize_bitmap;

#define ARM_SMMU_MAX_ASIDS		(1 << 16)
	unsigned int			asid_bits;

#define ARM_SMMU_MAX_VMIDS		(1 << 16)
	unsigned int			vmid_bits;
	DECLARE_BITMAP(vmid_map, ARM_SMMU_MAX_VMIDS);

	unsigned int			ssid_bits;
	unsigned int			sid_bits;

	struct arm_smmu_strtab_cfg	strtab_cfg;

	/* IOMMU core code handle */
	struct iommu_device		iommu;

	struct rb_root			streams;
	struct mutex			streams_mutex;
};
```

驱动程序加载的入口是 **arm_smmu_device_probe()** 函数，该函数定义 (位于 *drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c*) 如下：
```
static int arm_smmu_device_probe(struct platform_device *pdev)
{
	int irq, ret;
	struct resource *res;
	resource_size_t ioaddr;
	struct arm_smmu_device *smmu;
	struct device *dev = &pdev->dev;
	bool bypass;

	smmu = devm_kzalloc(dev, sizeof(*smmu), GFP_KERNEL);
	if (!smmu)
		return -ENOMEM;
	smmu->dev = dev;

	if (dev->of_node) {
		ret = arm_smmu_device_dt_probe(pdev, smmu);
	} else {
		ret = arm_smmu_device_acpi_probe(pdev, smmu);
		if (ret == -ENODEV)
			return ret;
	}

	/* Set bypass mode according to firmware probing result */
	bypass = !!ret;

	/* Base address */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (!res)
		return -EINVAL;
	if (resource_size(res) < arm_smmu_resource_size(smmu)) {
		dev_err(dev, "MMIO region too small (%pr)\n", res);
		return -EINVAL;
	}
	ioaddr = res->start;

	/*
	 * Don't map the IMPLEMENTATION DEFINED regions, since they may contain
	 * the PMCG registers which are reserved by the PMU driver.
	 */
	smmu->base = arm_smmu_ioremap(dev, ioaddr, ARM_SMMU_REG_SZ);
	if (IS_ERR(smmu->base))
		return PTR_ERR(smmu->base);

	if (arm_smmu_resource_size(smmu) > SZ_64K) {
		smmu->page1 = arm_smmu_ioremap(dev, ioaddr + SZ_64K,
					       ARM_SMMU_REG_SZ);
		if (IS_ERR(smmu->page1))
			return PTR_ERR(smmu->page1);
	} else {
		smmu->page1 = smmu->base;
	}

	/* Interrupt lines */

	irq = platform_get_irq_byname_optional(pdev, "combined");
	if (irq > 0)
		smmu->combined_irq = irq;
	else {
		irq = platform_get_irq_byname_optional(pdev, "eventq");
		if (irq > 0)
			smmu->evtq.q.irq = irq;

		irq = platform_get_irq_byname_optional(pdev, "priq");
		if (irq > 0)
			smmu->priq.q.irq = irq;

		irq = platform_get_irq_byname_optional(pdev, "gerror");
		if (irq > 0)
			smmu->gerr_irq = irq;
	}
	/* Probe the h/w */
	ret = arm_smmu_device_hw_probe(smmu);
	if (ret)
		return ret;

	/* Initialise in-memory data structures */
	ret = arm_smmu_init_structures(smmu);
	if (ret)
		return ret;

	/* Record our private device structure */
	platform_set_drvdata(pdev, smmu);

	/* Check for RMRs and install bypass STEs if any */
	arm_smmu_rmr_install_bypass_ste(smmu);

	/* Reset the device */
	ret = arm_smmu_device_reset(smmu, bypass);
	if (ret)
		return ret;

	/* And we're up. Go go go! */
	ret = iommu_device_sysfs_add(&smmu->iommu, dev, NULL,
				     "smmu3.%pa", &ioaddr);
	if (ret)
		return ret;

	ret = iommu_device_register(&smmu->iommu, &arm_smmu_ops, dev);
	if (ret) {
		dev_err(dev, "Failed to register iommu\n");
		iommu_device_sysfs_remove(&smmu->iommu);
		return ret;
	}

	return 0;
}
```

**`arm_smmu_device_probe()`** 函数执行如下的操作：

 * 从 DTS 的 SMMU 节点或 ACPI 的 SMMU 配置表读取诸如 SMMU 中断这样的属性。
 * 使用 **struct resource** 从设备和重映射 I/O 获取资源信息。
 * 探测 SMMU 的硬件特性。
 * 初始化中断和事件队列。
 * 创建一个 STE 表。
 * 复位设备。
 * 向 IOMMU 注册 SMMU。

### 1. 读取 DTS 的 SMMU 节点信息

**`arm_smmu_device_probe()`** 函数从 **`smmu->dev->of_node`** 读取属性，并把属性记录在 **`smmu->options`** 中。此外，该函数还检查 DMA 是否支持一致性。如果 DMA 支持一致性，则该函数设置一致性功能。
```
	if (of_dma_is_coherent(dev->of_node))
		smmu->features |= ARM_SMMU_FEAT_COHERENCY;
```

### 2. 获取设备资源信息并执行 I/O 重映射
**`struct resource`** 存储获取的 SMMU 设备的资源信息，包括 I/O 基地址和使用**`smmu->base`** 重映射 I/O 后的基地址。之后，可以通过使用 **`smmu->base`** 加上偏移量来读取和写入 SMMU 硬件的寄存器。
```
	/* Base address */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (!res)
		return -EINVAL;
	if (resource_size(res) < arm_smmu_resource_size(smmu)) {
		dev_err(dev, "MMIO region too small (%pr)\n", res);
		return -EINVAL;
	}
	ioaddr = res->start;

	/*
	 * Don't map the IMPLEMENTATION DEFINED regions, since they may contain
	 * the PMCG registers which are reserved by the PMU driver.
	 */
	smmu->base = arm_smmu_ioremap(dev, ioaddr, ARM_SMMU_REG_SZ);
	if (IS_ERR(smmu->base))
		return PTR_ERR(smmu->base);

	if (arm_smmu_resource_size(smmu) > SZ_64K) {
		smmu->page1 = arm_smmu_ioremap(dev, ioaddr + SZ_64K,
					       ARM_SMMU_REG_SZ);
		if (IS_ERR(smmu->page1))
			return PTR_ERR(smmu->page1);
	} else {
		smmu->page1 = smmu->base;
	}
```

### 3. 探测 SMMU 的硬件特性
**`arm_smmu_device_hw_probe()`** 函数通过读取 SMMU 的寄存器获取 SMMU 的硬件特性。

IDR0 寄存器：
```
	/* IDR0 */
	reg = readl_relaxed(smmu->base + ARM_SMMU_IDR0);

	/* 2-level structures */
	if (FIELD_GET(IDR0_ST_LVL, reg) == IDR0_ST_LVL_2LVL)
		smmu->features |= ARM_SMMU_FEAT_2_LVL_STRTAB;

	if (reg & IDR0_CD2L)
		smmu->features |= ARM_SMMU_FEAT_2_LVL_CDTAB;

	/*
	 * Translation table endianness.
	 * We currently require the same endianness as the CPU, but this
	 * could be changed later by adding a new IO_PGTABLE_QUIRK.
	 */
	switch (FIELD_GET(IDR0_TTENDIAN, reg)) {
	case IDR0_TTENDIAN_MIXED:
		smmu->features |= ARM_SMMU_FEAT_TT_LE | ARM_SMMU_FEAT_TT_BE;
		break;
#ifdef __BIG_ENDIAN
	case IDR0_TTENDIAN_BE:
		smmu->features |= ARM_SMMU_FEAT_TT_BE;
		break;
#else
	case IDR0_TTENDIAN_LE:
		smmu->features |= ARM_SMMU_FEAT_TT_LE;
		break;
#endif
	default:
		dev_err(smmu->dev, "unknown/unsupported TT endianness!\n");
		return -ENXIO;
	}

	/* Boolean feature flags */
	if (IS_ENABLED(CONFIG_PCI_PRI) && reg & IDR0_PRI)
		smmu->features |= ARM_SMMU_FEAT_PRI;

	if (IS_ENABLED(CONFIG_PCI_ATS) && reg & IDR0_ATS)
		smmu->features |= ARM_SMMU_FEAT_ATS;

	if (reg & IDR0_SEV)
		smmu->features |= ARM_SMMU_FEAT_SEV;

	if (reg & IDR0_MSI) {
		smmu->features |= ARM_SMMU_FEAT_MSI;
		if (coherent && !disable_msipolling)
			smmu->options |= ARM_SMMU_OPT_MSIPOLL;
	}

	if (reg & IDR0_HYP) {
		smmu->features |= ARM_SMMU_FEAT_HYP;
		if (cpus_have_cap(ARM64_HAS_VIRT_HOST_EXTN))
			smmu->features |= ARM_SMMU_FEAT_E2H;
	}

	/*
	 * The coherency feature as set by FW is used in preference to the ID
	 * register, but warn on mismatch.
	 */
	if (!!(reg & IDR0_COHACC) != coherent)
		dev_warn(smmu->dev, "IDR0.COHACC overridden by FW configuration (%s)\n",
			 coherent ? "true" : "false");

	switch (FIELD_GET(IDR0_STALL_MODEL, reg)) {
	case IDR0_STALL_MODEL_FORCE:
		smmu->features |= ARM_SMMU_FEAT_STALL_FORCE;
		fallthrough;
	case IDR0_STALL_MODEL_STALL:
		smmu->features |= ARM_SMMU_FEAT_STALLS;
	}

	if (reg & IDR0_S1P)
		smmu->features |= ARM_SMMU_FEAT_TRANS_S1;

	if (reg & IDR0_S2P)
		smmu->features |= ARM_SMMU_FEAT_TRANS_S2;

	if (!(reg & (IDR0_S1P | IDR0_S2P))) {
		dev_err(smmu->dev, "no translation support!\n");
		return -ENXIO;
	}

	/* We only support the AArch64 table format at present */
	switch (FIELD_GET(IDR0_TTF, reg)) {
	case IDR0_TTF_AARCH32_64:
		smmu->ias = 40;
		fallthrough;
	case IDR0_TTF_AARCH64:
		break;
	default:
		dev_err(smmu->dev, "AArch64 table format not supported!\n");
		return -ENXIO;
	}

	/* ASID/VMID sizes */
	smmu->asid_bits = reg & IDR0_ASID16 ? 16 : 8;
	smmu->vmid_bits = reg & IDR0_VMID16 ? 16 : 8;
```

 * 记录是否支持两级 STE 表和两级 CD 表。
 * 记录是否支持 PRI、ATS、SEV、MSI、HYP、STALL、第一阶段和第二阶段。
 * 获取 ias 长度，**`asid_bits`** 和 **`vmid_bits`**。

IDR1 寄存器：
```
	/* IDR1 */
	reg = readl_relaxed(smmu->base + ARM_SMMU_IDR1);
	if (reg & (IDR1_TABLES_PRESET | IDR1_QUEUES_PRESET | IDR1_REL)) {
		dev_err(smmu->dev, "embedded implementation not supported\n");
		return -ENXIO;
	}

	/* Queue sizes, capped to ensure natural alignment */
	smmu->cmdq.q.llq.max_n_shift = min_t(u32, CMDQ_MAX_SZ_SHIFT,
					     FIELD_GET(IDR1_CMDQS, reg));
	if (smmu->cmdq.q.llq.max_n_shift <= ilog2(CMDQ_BATCH_ENTRIES)) {
		/*
		 * We don't support splitting up batches, so one batch of
		 * commands plus an extra sync needs to fit inside the command
		 * queue. There's also no way we can handle the weird alignment
		 * restrictions on the base pointer for a unit-length queue.
		 */
		dev_err(smmu->dev, "command queue size <= %d entries not supported\n",
			CMDQ_BATCH_ENTRIES);
		return -ENXIO;
	}

	smmu->evtq.q.llq.max_n_shift = min_t(u32, EVTQ_MAX_SZ_SHIFT,
					     FIELD_GET(IDR1_EVTQS, reg));
	smmu->priq.q.llq.max_n_shift = min_t(u32, PRIQ_MAX_SZ_SHIFT,
					     FIELD_GET(IDR1_PRIQS, reg));

	/* SID/SSID sizes */
	smmu->ssid_bits = FIELD_GET(IDR1_SSIDSIZE, reg);
	smmu->sid_bits = FIELD_GET(IDR1_SIDSIZE, reg);
	smmu->iommu.max_pasids = 1UL << smmu->ssid_bits;

	/*
	 * If the SMMU supports fewer bits than would fill a single L2 stream
	 * table, use a linear table instead.
	 */
	if (smmu->sid_bits <= STRTAB_SPLIT)
		smmu->features &= ~ARM_SMMU_FEAT_2_LVL_STRTAB;
```

 * 获取 **`evtq`** 和 **`priq`** 队列的长度，及 **`ssid_bits`** 和 **`sid_bits`**。

IDR5 寄存器：
```
	/* IDR3 */
	reg = readl_relaxed(smmu->base + ARM_SMMU_IDR3);
	if (FIELD_GET(IDR3_RIL, reg))
		smmu->features |= ARM_SMMU_FEAT_RANGE_INV;

	/* IDR5 */
	reg = readl_relaxed(smmu->base + ARM_SMMU_IDR5);

	/* Maximum number of outstanding stalls */
	smmu->evtq.max_stalls = FIELD_GET(IDR5_STALL_MAX, reg);

	/* Page sizes */
	if (reg & IDR5_GRAN64K)
		smmu->pgsize_bitmap |= SZ_64K | SZ_512M;
	if (reg & IDR5_GRAN16K)
		smmu->pgsize_bitmap |= SZ_16K | SZ_32M;
	if (reg & IDR5_GRAN4K)
		smmu->pgsize_bitmap |= SZ_4K | SZ_2M | SZ_1G;

	/* Input address size */
	if (FIELD_GET(IDR5_VAX, reg) == IDR5_VAX_52_BIT)
		smmu->features |= ARM_SMMU_FEAT_VAX;

	/* Output address size */
	switch (FIELD_GET(IDR5_OAS, reg)) {
	case IDR5_OAS_32_BIT:
		smmu->oas = 32;
		break;
	case IDR5_OAS_36_BIT:
		smmu->oas = 36;
		break;
	case IDR5_OAS_40_BIT:
		smmu->oas = 40;
		break;
	case IDR5_OAS_42_BIT:
		smmu->oas = 42;
		break;
	case IDR5_OAS_44_BIT:
		smmu->oas = 44;
		break;
	case IDR5_OAS_52_BIT:
		smmu->oas = 52;
		smmu->pgsize_bitmap |= 1ULL << 42; /* 4TB */
		break;
	default:
		dev_info(smmu->dev,
			"unknown output address size. Truncating to 48-bit\n");
		fallthrough;
	case IDR5_OAS_48_BIT:
		smmu->oas = 48;
	}

	if (arm_smmu_ops.pgsize_bitmap == -1UL)
		arm_smmu_ops.pgsize_bitmap = smmu->pgsize_bitmap;
	else
		arm_smmu_ops.pgsize_bitmap |= smmu->pgsize_bitmap;

	/* Set the DMA mask for our table walker */
	if (dma_set_mask_and_coherent(smmu->dev, DMA_BIT_MASK(smmu->oas)))
		dev_warn(smmu->dev,
			 "failed to set DMA mask for table walker\n");

	smmu->ias = max(smmu->ias, smmu->oas);

	if (arm_smmu_sva_supported(smmu))
		smmu->features |= ARM_SMMU_FEAT_SVA;
```

 * 获取 evtq stalls 的最大个数。
 * 记录是否支持 VAX、oas 长度，和 **`pgsize_bitmap`**。

### 4. 初始化中断和事件队列
**`arm_smmu_init_structures()`** 函数初始化内存中的数据结构，包括三个队列：**`evtq`**、**`priq`** 和 **`cmdq`**。SMMU 驱动程序使用 **`cmdq`** 队列给硬件发送命令，比如刷新 TLB 和写入 CD。挂载到 SMMU 的平台设备使用 **`evtq`** 队列给 SMMU 驱动程序发送异常消息。**`priq`** 队列的功能与 **`evtq`** 队列类似，除了前者由挂载的 PCI 设备使用。**`evtq`** 和 **`priq`** 队列有自己的中断 ID 来通知异常事件。另外，**`gerror`** 中断 ID 用于报告不可恢复的严重错误，这些错误会被直接中断，而不添加到队列中。
```
	/* Interrupt lines */

	irq = platform_get_irq_byname_optional(pdev, "combined");
	if (irq > 0)
		smmu->combined_irq = irq;
	else {
		irq = platform_get_irq_byname_optional(pdev, "eventq");
		if (irq > 0)
			smmu->evtq.q.irq = irq;

		irq = platform_get_irq_byname_optional(pdev, "priq");
		if (irq > 0)
			smmu->priq.q.irq = irq;

		irq = platform_get_irq_byname_optional(pdev, "gerror");
		if (irq > 0)
			smmu->gerr_irq = irq;
	}
 . . . . . .
	/* Initialise in-memory data structures */
	ret = arm_smmu_init_structures(smmu);
	if (ret)
		return ret;
```

当驱动程序复位 SMMU 设备时，**`arm_smmu_setup_unique_irqs()`** 函数注册对应的事件处理程序。**`evtq`** 和 **`priq`** 队列注册一个内核线程来完成事件的处理。对于不可恢复的错误，**`arm_smmu_setup_unique_irqs()`** 函数直接注册它完成了中断处理。

### 5. 创建 STE 表
基于 SMMU 的配置，可以创建两级或线性 STE 表。相比于线性 STE 表，两级的 STE 表不需要在一开始就创建所有的 STE，而仅分配第一级的目录项。对于线性 STE 表，根据 **`sid_bits`** 和 STE 的值，通过使用 DMA 分配内存中的连续区域。基地址记录在配置中，所有 STE 都设置为默认的旁路模式。
```
static int arm_smmu_init_strtab_linear(struct arm_smmu_device *smmu)
{
	void *strtab;
	u64 reg;
	u32 size;
	struct arm_smmu_strtab_cfg *cfg = &smmu->strtab_cfg;

	size = (1 << smmu->sid_bits) * (STRTAB_STE_DWORDS << 3);
	strtab = dmam_alloc_coherent(smmu->dev, size, &cfg->strtab_dma,
				     GFP_KERNEL);
	if (!strtab) {
		dev_err(smmu->dev,
			"failed to allocate linear stream table (%u bytes)\n",
			size);
		return -ENOMEM;
	}
	cfg->strtab = strtab;
	cfg->num_l1_ents = 1 << smmu->sid_bits;

	/* Configure strtab_base_cfg for a linear table covering all SIDs */
	reg  = FIELD_PREP(STRTAB_BASE_CFG_FMT, STRTAB_BASE_CFG_FMT_LINEAR);
	reg |= FIELD_PREP(STRTAB_BASE_CFG_LOG2SIZE, smmu->sid_bits);
	cfg->strtab_base_cfg = reg;

	arm_smmu_init_bypass_stes(strtab, cfg->num_l1_ents, false);
	return 0;
}

static int arm_smmu_init_strtab(struct arm_smmu_device *smmu)
{
	u64 reg;
	int ret;

	if (smmu->features & ARM_SMMU_FEAT_2_LVL_STRTAB)
		ret = arm_smmu_init_strtab_2lvl(smmu);
	else
		ret = arm_smmu_init_strtab_linear(smmu);

	if (ret)
		return ret;

	/* Set the strtab base address */
	reg  = smmu->strtab_cfg.strtab_dma & STRTAB_BASE_ADDR_MASK;
	reg |= STRTAB_BASE_RA;
	smmu->strtab_cfg.strtab_base = reg;

	/* Allocate the first VMID for stage-2 bypass STEs */
	set_bit(0, smmu->vmid_map);
	return 0;
}
```

### 6. 复位设备
**`arm_smmu_device_reset()`** 函数复位设备。该函数根据获得的设备寄存器，将队列内存属性等信息写入控制寄存器 CR1 和 CR2，将 STE 表的基地址和配置信息写入 STRTAB_BASE 寄存器，并将内存中三个队列的基地址、队列头和队列尾写入对应的寄存器中。然后，**`arm_smmu_device_reset()`** 函数通过调用**`arm_smmu_setup_irqs()`** 函数来注册中断事件处理程序。

### 7. 向 IOMMU 注册 SMMU

在 Linux 内核中，不同平台上的 IOMMU 设备具有统一的 IOMMU 接口。在 SMMU 初始化期间，向 **`sys`** 目录注册了 **`smmu->iommu`** 设备节点，且向设备及支持 SMMU 的总线类型注册了 **`arm_smmu_ops`**。通过这种方式，当 IOMMU 公共接口被使用时，SMMU 提供的函数被调用。更多细节，请参考 **`arm_smmu_ops`** 提供的各种 IOMMU 接口的实现。
```
	/* And we're up. Go go go! */
	ret = iommu_device_sysfs_add(&smmu->iommu, dev, NULL,
				     "smmu3.%pa", &ioaddr);
	if (ret)
		return ret;

	ret = iommu_device_register(&smmu->iommu, &arm_smmu_ops, dev);
	if (ret) {
		dev_err(dev, "Failed to register iommu\n");
		iommu_device_sysfs_remove(&smmu->iommu);
		return ret;
	}
```

**`arm_smmu_device_reset()`** 函数定义 (位于 `drivers/iommu/iommu.c`) 如下：
```
static struct bus_type * const iommu_buses[] = {
	&platform_bus_type,
#ifdef CONFIG_PCI
	&pci_bus_type,
#endif
#ifdef CONFIG_ARM_AMBA
	&amba_bustype,
#endif
#ifdef CONFIG_FSL_MC_BUS
	&fsl_mc_bus_type,
#endif
#ifdef CONFIG_TEGRA_HOST1X_CONTEXT_BUS
	&host1x_context_device_bus_type,
#endif
#ifdef CONFIG_CDX_BUS
	&cdx_bus_type,
#endif
};
 . . . . . .
/**
 * iommu_device_register() - Register an IOMMU hardware instance
 * @iommu: IOMMU handle for the instance
 * @ops:   IOMMU ops to associate with the instance
 * @hwdev: (optional) actual instance device, used for fwnode lookup
 *
 * Return: 0 on success, or an error.
 */
int iommu_device_register(struct iommu_device *iommu,
			  const struct iommu_ops *ops, struct device *hwdev)
{
	int err = 0;

	/* We need to be able to take module references appropriately */
	if (WARN_ON(is_module_address((unsigned long)ops) && !ops->owner))
		return -EINVAL;
	/*
	 * Temporarily enforce global restriction to a single driver. This was
	 * already the de-facto behaviour, since any possible combination of
	 * existing drivers would compete for at least the PCI or platform bus.
	 */
	if (iommu_buses[0]->iommu_ops && iommu_buses[0]->iommu_ops != ops)
		return -EBUSY;

	iommu->ops = ops;
	if (hwdev)
		iommu->fwnode = dev_fwnode(hwdev);

	spin_lock(&iommu_device_lock);
	list_add_tail(&iommu->list, &iommu_device_list);
	spin_unlock(&iommu_device_lock);

	for (int i = 0; i < ARRAY_SIZE(iommu_buses) && !err; i++) {
		iommu_buses[i]->iommu_ops = ops;
		err = bus_iommu_probe(iommu_buses[i]);
	}
	if (err)
		iommu_device_unregister(iommu);
	return err;
}
EXPORT_SYMBOL_GPL(iommu_device_register);
```

## IOMMU 和 DMA

IOMMU 的主要功能之一是防止设备在以 DMA 模式访问内存时直接使用 PA。因此，产生了 IOVA。当 **`dma_alloc()`** 分配内存时，首先在 I/O 地址空间中分配一个 IOVA，然后，在 IOMMU 管理的页表中建立 **`dma_alloc()`**  分配的 IOVA 与 PA 之间的映射。当外设执行 DMA 时，只需要使用 IOVA。

当调用 **`dma_alloc()`** 系列函数分配内存时，会调用 **`iommu_dma_alloc()`** 函数。**`iommu_dma_alloc()`** 函数分配 IOVA 和实际的物理内存，并使用 **`iommu_map()`** 函数建立 IOVA 和 物理内存之间的映射，即，定位到对应于设备的 STE，定位到 CD (通常是 CD 0)，定位到内存中对应的页表，并将 IOVA 和物理内存之间的映射写入页表。与 ARM SMMU 相关的页表操作在 **drivers/iommu/io-pgtable.c**/**drivers/iommu/dma-iommu.c** 中执行。
```
static dma_addr_t __iommu_dma_map(struct device *dev, phys_addr_t phys,
		size_t size, int prot, u64 dma_mask)
{
	struct iommu_domain *domain = iommu_get_dma_domain(dev);
	struct iommu_dma_cookie *cookie = domain->iova_cookie;
	struct iova_domain *iovad = &cookie->iovad;
	size_t iova_off = iova_offset(iovad, phys);
	dma_addr_t iova;

	if (static_branch_unlikely(&iommu_deferred_attach_enabled) &&
	    iommu_deferred_attach(dev, domain))
		return DMA_MAPPING_ERROR;

	size = iova_align(iovad, size + iova_off);

	iova = iommu_dma_alloc_iova(domain, size, dma_mask, dev);
	if (!iova)
		return DMA_MAPPING_ERROR;

	if (iommu_map(domain, iova, phys - iova_off, size, prot, GFP_ATOMIC)) {
		iommu_dma_free_iova(cookie, iova, size, NULL);
		return DMA_MAPPING_ERROR;
	}
	return iova + iova_off;
}
 . . . . . .
static void *iommu_dma_alloc(struct device *dev, size_t size,
		dma_addr_t *handle, gfp_t gfp, unsigned long attrs)
{
	bool coherent = dev_is_dma_coherent(dev);
	int ioprot = dma_info_to_prot(DMA_BIDIRECTIONAL, coherent, attrs);
	struct page *page = NULL;
	void *cpu_addr;

	gfp |= __GFP_ZERO;

	if (gfpflags_allow_blocking(gfp) &&
	    !(attrs & DMA_ATTR_FORCE_CONTIGUOUS)) {
		return iommu_dma_alloc_remap(dev, size, handle, gfp,
				dma_pgprot(dev, PAGE_KERNEL, attrs), attrs);
	}

	if (IS_ENABLED(CONFIG_DMA_DIRECT_REMAP) &&
	    !gfpflags_allow_blocking(gfp) && !coherent)
		page = dma_alloc_from_pool(dev, PAGE_ALIGN(size), &cpu_addr,
					       gfp, NULL);
	else
		cpu_addr = iommu_dma_alloc_pages(dev, size, &page, gfp, attrs);
	if (!cpu_addr)
		return NULL;

	*handle = __iommu_dma_map(dev, page_to_phys(page), size, ioprot,
			dev->coherent_dma_mask);
	if (*handle == DMA_MAPPING_ERROR) {
		__iommu_dma_free(dev, size, cpu_addr);
		return NULL;
	}

	return cpu_addr;
}
```

可以通过多种方式绕过 IOMMU。Linux 提供了 **`iommu.passthrough`** 模式。你可以将 DMA 配置为不使用 IOMMU，而是使用软件输入输出转换后备缓冲区 (SWIOTLB) 技术来访问内存。此外，SMMUv3 驱动程序提供了绕过 SMMU 的参数。除此之外，你不在 ACPI 或 DTS 中配置 SMMU 节点，这样系统加载时就不会探测到相应的 SMMU。

## 总结

IOMMU 的主要功能是为设备访问物理内存提供从 IOVA 到 PA 的映射。这样设备就不会直接使用 PA 来访问内存，这样更安全。ARM SMMUv3 作为 IOMMU 的具体实现，为 IOMMU 提供了接口。IOMMU 相关操作包括设备执行的 DMA 操作，以及通过 VFIO 直通安全地暴露给用户模式的设备硬件功能，用于建立设备标识的 IOVA 与实际 PA 之间的映射。

## 参考文档

[Learn the Architecture - SMMU Software Guide](https://developer.arm.com/documentation/109242/0100?lang=en)

[原文](https://www.openeuler.org/en/blog/wxggg/2020-11-21-iommu-smmu-intro.html)

Done.
