---
title: Linux 和设备树
date: 2023-05-13 19:33:29
categories: Linux 内核
tags:
- Linux 内核
---

设备树数据的 Linux 使用模型

本文描述了 Linux 如何使用设备树。可以在 devicetree.org 的设备树使用页找到设备树数据格式的概述。

[https://www.devicetree.org/specifications/](https://www.devicetree.org/specifications/)

“开放固件设备树”，或简称为 Devicetree (DT)，是一种用于描述硬件的数据结构和语言。更具体地说，它是操作系统可读的一种硬件描述，有了它操作系统就不需要硬编码机器的细节了。

从结构上来说，DT 是命名节点的树或无环图，节点可以具有任意数量的命名属性来封装任意数据。还存在一种机制，可以在自然的树结构之外创建从一个节点到另一个节点的任意链接。

从概念上讲，一组通用的使用约定，称为“绑定”，定义树中的数据格式，来描述典型的硬件特性，包括数据总线、中断线、GPIO 连接和外围设备。

尽可能使用已有的绑定来描述硬件，以最大限度地利用现有的支持代码，但由于属性和节点名称只是简单的文本字符串，因此通过定义新节点和属性来扩展已有的绑定或创建新的绑定也很容易。然而，创建新的绑定而不首先做功课了解已有的绑定，则还是要小心。目前有两种不同的不兼容的 i2c 总线绑定，就是由于新的绑定在创建时，没有首先研究现有系统中是如何枚举 i2c 设备的。

## 1. 历史

DT 最初是由开放固件 (Open Firmware) 创建的，它是从开放固件向客户端程序 (比如操作系统) 传递数据的通信方法的一部分。操作系统使用设备树在运行时发现硬件的拓扑结构，从而支持大多数可用的硬件，而无需硬编码信息 (假设所有设备的驱动程序都可用)。

由于开放固件 (Open Firmware) 通常用于 PowerPC 和 SPARC 平台，因此对那些体系结构的 Linux 支持使用设备树已经有很长时间了。

在 2005 年，当 PowerPC Linux 开始进行重大清理并合并 32 位和 64 位支持时，决定要求在所有 PowerPC 平台上支持 DT，而不管它们是否使用 Open Firmware。为此，创建了称为平坦设备树 (Flattened Device Tree，FDT) 的 DT 表示，它可以以二进制块的形式传给内核，而无需真正的 Open Firmware 实现。U-Boot、kexec 和其它引导加载程序被修改为既支持传递设备树二进制文件 (dtb)，也支持在启动时修改 dtb。DT 还被添加到 PowerPC 启动包装器 (*arch/ PowerPC /boot/**) 中，这样 dtb 就可以与内核镜像包装在一起，以支持启动现有的不支持 DT 的固件。

一段时间后，FDT 基础架构被推广为可用于所有体系结构。在撰写本文时，有 6 种主线体系结构 (arm，microblaze，mips，powerpc，sparc，和 x86) 及 1 种非主线体系结构 (nios) 都具有一定程度的 DT 支持。

## 数据模型

如果你还没有读过 [设备树使用](https://www.devicetree.org/specifications/) 规范文档，现在立即去读。没关系的，我可以等....

### 高层视图

要理解的最重要的事情是，DT 只是一个描述硬件的数据结构。它没有什么神奇之处，它不会神奇地使所有硬件配置问题消失。它所做的是提供一种语言，将硬件配置从 Linux 内核 (或任何其它操作系统) 的板子和设备驱动程序支持中解耦出来。使用它可以让板子和设备支持成为数据驱动的。使设置决策的做出基于传入内核的数据，而不是基于每台机器的硬编码选择。

理想情况下，数据驱动的平台设置应该减少重复代码，并且使用单个内核镜像支持各种硬件也更加容易。

Linux 使用 DT 数据有三个主要目的：
1. 平台识别，
2. 运行时配置，和
3. device population。

### 2.2 平台识别

首先，内核将使用 DT 中的数据来识别特定的机器。在完美的情况下，特定的平台应该与内核无关，因为设备树将以一致和可靠的方式完美地描述所有平台细节。但是硬件并不完美，因此内核必须在早期启动时识别机器，以便有机会运行特定于机器的修复程序。

在大多数情况下，机器识别是无关紧要的，内核将根据机器的核心 CPU 或 SoC 选择设置代码。以 ARM 为例，*arch/arm/kernel/setup.c* 中的 `setup_arch()` 将调用 *arch/arm/kernel/devtree.c* 中的 `setup_machine_fdt()`，后者搜索 `machine_desc` 表并选择与设备树数据最匹配的 machine_desc。它通过查看根设备树节点的 'compatible' 属性，并将其与 struct machine_desc (如果你好奇的话，其定义位于 *arch/arm/include/asm/mach/arch.h*) 的 dt_compat 列表进行比较来决定最佳匹配。

'compatible' 属性包含一个字符串的有序列表，其中各字符串以机器的名字开头，后面是一个可选的兼容的板子列表，其中各板子从最兼容到最小兼容排序。

例如，TI BeagleBoard 及后继型号和 BeagleBoard xM 板子的根 'compatible' 属性可能分别如下：
```
compatible = "ti,omap3-beagleboard", "ti,omap3450", "ti,omap3";
compatible = "ti,omap3-beagleboard-xm", "ti,omap3450", "ti,omap3";
```

其中 “ti,omap3-beagleboard-xm” 指定了确切的型号，它还声明它与 OMAP 3450 SoC和 omap3 系列 SoC 兼容。你会注意到该列表从最具体 (确切的板子) 到最不具体 (SoC 系列) 排序。

精明的读者可能会指出，Beagle xM 也可以声明与原始的 Beagle 板子兼容。然而，在板子级这样做应该谨慎，因为从一块板子到另一块板子通常会有很大的变化，即使是在同一条产品线中，而且当一块板子声称与另一块板子兼容时，很难确切地确定是什么意思。从顶层来看，最好谨慎行事，不要声称一块板子与另一块板子兼容。

关于 'compatible' 值还要注意一点。在 'compatible' 属性中使用的任何字符串都必须提供文档，来说明它表示什么。在 *Documentation/devicetree/bindings* 中为 ‘compatible’ 字符串添加文档。

再次以 ARM 为例，对于每个 machine_desc，内核查看是否有任何 dt_compat 列表项出现在 'compatible' 属性中。如果出现了一次，则这个 machine_desc 就是驱动机器的候选者。在搜索整个 machine_descs 表之后，`setup_machine_fdt()` 基于 'compatible' 属性中的条目与各个 machine_desc 的匹配情况返回 **最兼容的** machine_desc。如果没找到匹配的 machine_desc，则返回 NULL。

这个方案背后的原因是观察到，在大多数情况下，单个 machine_desc 可以支持大量的板子，如果它们都使用相同的 SoC，或相同的 SoC 系列。然而，总是会有一些例外，其中特定的板子需要一般情况下没什么用的特殊设置代码。特殊情况可通过在一般代码中显式地检查有问题的板子来处理，但这样做很快就变得很丑，且不可维护，如果有许多类似的情况的话。

相反，'compatible' 列表允许通用的 machine_desc 通过在 dt_compat 列表中指定 “不太兼容” 的值来提供对广泛的通用板子集合的支持。在上面的例子中，通用的板子支持可以声明与 "ti,omap3" 或 "ti,omap3450" 兼容。如果在最初的 beagleboard 上发现了需要在早期启动时使用的特殊 workaround 代码的 bug，则可以添加一个实现 workaround 的新 machine_desc，且只与 "ti,omap3-beagleboard" 匹配。

PowerPC 使用了一个稍有不同的方案，它从每个 machine_desc 调用 `.probe()` hook，并使用第一个返回的 TRUE 的那个。然而，这种方法没有考虑到 'compatible' 列表的优先级，且对于新体系结构的支持可能应该避免。

### 运行时配置

在大多数情况下，DT 将是将数据从固件传输到内核的唯一方法，因此也用于传递运行时和配置数据，比如内核参数字符串和 initrd 镜像的位置。

大多数这种数据都包含在 `/chosen` 节点中，当启动 Linux 时，它将看起来像下面这样：
```
chosen {
        bootargs = "console=ttyS0,115200 loglevel=8";
        initrd-start = <0xc8000000>;
        initrd-end = <0xc8200000>;
};
```

`bootargs` 属性包含内核参数，`initrd-*` 属性定义 initrd 块的地址和大小。注意，`initrd-end` 是 initrd 镜像之后的第一个地址，因此它与 `struct resource` 的通常的语义不匹配。`chosen` 节点还可以包含任意数量的附加属性，用于特定于平台的配置数据。

在早期启动期间，体系结构设置代码以不同的辅助回调多次调用 `of_scan_flat_dt()`，在 paging 设置之前解析设备树数据。`of_scan_flat_dt()` 代码扫描设备树，并使用辅助回调提取早期启动期间所需的信息。通常，`early_init_dt_scan_chosen()` 辅助回调用于解析 `chosen` 节点包含内核参数，`early_init_dt_scan_root()` 用于初始化 DT 地址空间模型，`early_init_dt_scan_memory()` 用于确定可用 RAM 的大小和位置。

在 ARM 上，其 `setup_machine_fdt()` 函数负责在选择正确的支持该板子的 machine_desc 之后，对设备树进行早期扫描。

### Device population

在识别了板子，并解析了早期配置数据之后，内核初始化就可以按照正常的方式进行了。在这个过程中的某个时刻，将调用 `unflatten_device_tree()` 把数据转换为更高效的运行时表示。这也是特定于机器的设置 hooks 被调用的时刻，比如 ARM 上的 machine_desc `.init_early()`，`.init_irq()` 和 `.init_machine()` hooks。本节的其余部分使用来自 ARM 实现的示例，但是当使用 DT 时，所有体系结构都将做几乎相同的事情。

从名字就能猜到，`.init_early()` 用于任何需要在启动过程早期执行的特定于机器的设置，而 `.init_irq()` 用于设置中断处理。使用 DT 并不会实质性地改变这两个函数的行为。如果提供了 DT，则 `.init_early()` 和 `.init_irq()` 都能够调用任何 DT 查询函数 (*include/linux/of*.h* 中的 `of_*`) 来获得关于平台的额外数据。

DT 上下文中最有趣的 hook 是 `.init_machine()`，它主要负责用关于平台的数据填充 Linux 设备模型。从历史上看，在嵌入式平台上已经通过在板子支持 *.c* 文件中定义一组静态的时钟结构体、platform_devices 和其他数据，并在 `.init_machine()` 中批量注册来实现。当使用 DT 时，不需要为各个平台硬编码静态的设备，设备列表可以通过解析 DT，并动态分配设备结构来获得。

最简单的情况是，`.init_machine()` 只负责注册 platform_devices 块。platform_device 是 Linux 为内存或 I/O 映射设备，以及组合('composite') 或虚拟 ('virtual') 设备 (稍后会详细介绍) 使用的概念，它们无法通过硬件探测。同时，DT 没有 “平台设备” ('platform device') 的术语，平台设备大致对应于设备树的根处的设备节点，和简单内存映射总线节点的子节点。

现在是举个例子的好时机。以下是 NVIDIA Tegra 板子的部分设备树：
```
/{
      compatible = "nvidia,harmony", "nvidia,tegra20";
      #address-cells = <1>;
      #size-cells = <1>;
      interrupt-parent = <&intc>;

      chosen { };
      aliases { };

      memory {
              device_type = "memory";
              reg = <0x00000000 0x40000000>;
      };

      soc {
              compatible = "nvidia,tegra20-soc", "simple-bus";
              #address-cells = <1>;
              #size-cells = <1>;
              ranges;

              intc: interrupt-controller@50041000 {
                      compatible = "nvidia,tegra20-gic";
                      interrupt-controller;
                      #interrupt-cells = <1>;
                      reg = <0x50041000 0x1000>, < 0x50040100 0x0100 >;
              };

              serial@70006300 {
                      compatible = "nvidia,tegra20-uart";
                      reg = <0x70006300 0x100>;
                      interrupts = <122>;
              };

              i2s1: i2s@70002800 {
                      compatible = "nvidia,tegra20-i2s";
                      reg = <0x70002800 0x100>;
                      interrupts = <77>;
                      codec = <&wm8903>;
              };

              i2c@7000c000 {
                      compatible = "nvidia,tegra20-i2c";
                      #address-cells = <1>;
                      #size-cells = <0>;
                      reg = <0x7000c000 0x100>;
                      interrupts = <70>;

                      wm8903: codec@1a {
                              compatible = "wlf,wm8903";
                              reg = <0x1a>;
                              interrupts = <347>;
                      };
              };
      };

      sound {
              compatible = "nvidia,harmony-sound";
              i2s-controller = <&i2s1>;
              i2s-codec = <&wm8903>;
      };
};
```

在 `.init_machine()` 时，Tegra 板子支持代码将需要查看这个 DT，并决定为哪些节点创建 platform_devices。但是，看一下这个树，每个节点代表什么类型的设备并不明显，甚至节点是否代表设备也不明显。`/chosen`、`/aliases` 和 `/memory` 节点是不描述设备的信息节点(尽管可以认为内存可以被视为设备)。`/soc` 节点的子节点是内存映射设备，但是 **codec@1a** 是一个 i2c 设备，而 **sound** 节点不表示设备，而是其它设备如何连接在一起以创建音频子系统。我知道每个设备是什么，是因为我熟悉板子的设计，但内核如何知道要怎么处理各个节点呢？

技巧在于内核是从设备树的根开始，寻找具有 'compatible' 属性的节点。首先，通常假设任何具有 'compatible' 属性的节点都表示某种设备，其次，可以假设设备树的根处的任何节点要么直接连接到处理器总线，要么是一个不能以任何其它方式描述的杂项系统设备。对于这些节点中的每一个，Linux 分配并注册一个 platform_device，而这个设备又可以绑定到一个 platform_driver。

为什么为这些节点使用 platform_device 是一个安全的假设? 对于 Linux 建模设备的方式，几乎所有的 bus_types 都假定它的设备是总线控制器的子设备。比如，每个 i2c_client 都是一个 i2c_master 的子设备。每个 spi_device 都是一个 SPI 总线的子设备。USB，PCI，MDIO 等等也类似。在 DT 中也可以找到相同的层次结构，其中 I2C 设备节点仅作为 I2C 总线节点的子节点出现。这同样适用于 SPI，MDIO，USB 等。唯一不需要特定类型的父设备的设备是 platform_devices (和 amba_devices，稍后会详细介绍)，它将愉快地驻留在 Linux /sys/devices 树的底部。因此，如果 DT 节点位于设备树的根，那么最好将其注册为 platform_device。

Linux 板子支持代码调用 `of_platform_populate(NULL, NULL, NULL, NULL)` 来启动在设备树的根处发现设备。参数全都是 NULL，因为从设备树的根开始时，不需要提供起始节点 (第一个 NULL)，父 [`struct device`](https://docs.kernel.org/driver-api/infrastructure.html#c.device "device")，且我们 (也) 不使用匹配表。 对于只需要注册设备的板子，`.init_machine()` 完全可以是空的，除了 [`of_platform_populate()`](https://docs.kernel.org/devicetree/kernel-api.html#c.of_platform_populate "of_platform_populate") 调用。

在 Tegra 的例子中，这解释了 `/soc` 和 `/sound` 节点，但是 SoC 节点的子节点呢? 难道它们不应该被注册为平台设备吗? 对于 Linux DT 支持，一般的行为是父设备驱动程序在驱动程序 `.probe()` 时注册子设备。因此，i2c 总线设备驱动程序将为各个子节点注册一个 i2c_client，SPI 总线驱动程序将注册它的 spi_device 子设备，其它 bus_types 也类似。根据该模型，可以编写一个绑定到 SoC 节点的驱动程序，并简单地为其每个子设备注册 platform_devices。板子支持代码将分配并注册 SoC 设备，(理论上) SoC 设备驱动程序可以绑定到 SoC 设备，并在其 `.probe()` hook 中为 */soc/interrupt-controller*，*/soc/serial*，*/soc/i2s*，和 */soc/i2c* 注册 platform_devices。容易，对吧？

实际上，将一些 platform_devices 的子设备注册为更多的 platform_devices 是一种常见的模式，设备树支持代码反映了这一点，并使上面的示例更简单。[`of_platform_populate()`](https://docs.kernel.org/devicetree/kernel-api.html#c.of_platform_populate "of_platform_populate") 的第二个参数是 of_device_id 表，与该表中的条目匹配的任何节点也将注册其子节点。

在 Tegra 的情况下，代码看起来像这样：
```
static void __init harmony_init_machine(void)
{
      /* ... */
      of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL);
}
```

在 [设备树规范](https://www.devicetree.org/specifications/) 中，"simple-bus" 被定义为一个属性，意思是一个简单的内存映射总线，因此可以编写 `of_platform_populate()` 代码，假设总是要遍历与简单总线兼容的节点。但是，我们将其作为参数传递，以便板子支持代码始终可以覆盖默认行为。

[需要添加对于添加 i2c/spi/等 子设备的讨论]。

## 附录 A：AMBA 设备

ARM Primecells 是一种附加在 ARM AMBA 总线上的设备，它支持硬件检测和电源管理。在 Linux 中，struct amba_device 和 amba_bus_type 用于表示 Primecell 设备。然而，棘手的是 AMBA 总线上并非所有设备都是 Primecells，对于 Linux 来说，amba_device 和 platform_device 实例都是同一总线段的兄弟实例是很典型的。

当使用 DT 时，这会给 [`of_platform_populate()`](https://docs.kernel.org/devicetree/kernel-api.html#c.of_platform_populate "of_platform_populate") 制造一个问题，因为它必须决定是把各个节点注册为 platform_device  还是 amba_device。不幸的是，这使设备创建模型变得有点复杂，但解决方案并不太具有侵入性。如果一个节点与 "arm,amba-primecell" 兼容，则 [`of_platform_populate()`](https://docs.kernel.org/devicetree/kernel-api.html#c.of_platform_populate "of_platform_populate") 将把它注册为 amba_device，而不是 platform_device。

[原文](https://docs.kernel.org/devicetree/usage-model.html)

