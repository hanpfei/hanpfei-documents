---
title: Linux 内核设备树时钟绑定
date: 2023-06-17 17:33:29
categories: Linux 内核
tags:
- Linux 内核
---

这种绑定依然处于开发中，并且基于 benh[1] 的一些实验性工作。

时钟信号源可以由设备树中的任何节点表示。这些节点被指定为时钟提供者。时钟消费者节点使用 `phandle` 和时钟指示符对将时钟提供者输出连接到时钟输入。与 gpio 指示符类似，时钟指示符是 0 个、1 个或多个标识设备上的时钟输出的元素的数组。时钟指示符的长度由时钟提供者节点中的 `#clock-cells` 属性值定义。

[1] https://patchwork.ozlabs.org/patch/31551/

## 时钟提供者

必需的属性：
`#clock-cells`：时钟指示符中的元素个数；通常，具有单个时钟输出的节点为 0，具有多个时钟输出的节点为 1。

可选的属性：
`clock-output-names`：建议为，由时钟指示符中的首个元素索引的时钟输出信号名称的字符串的列表。然而，`clock-output-names` 的含义特定于时钟提供者的域，***并且只是为了鼓励对大多数时钟提供者使用相同的含义***。这种格式可能不适用于使用复杂时钟指示符格式的时钟提供者。在这些情况下，建议省略此属性，并创建特定于绑定的名称属性。

时钟消费者节点不得直接引用提供者的 `clock-output-names` 属性。

例如：
```
    oscillator {
        #clock-cells = <1>;
        clock-output-names = "ckil", "ckih";
    };
```

- 这个节点定义了一个具有两个时钟输出的设备，第一个的名称为  **"ckil"**，第二个的名称为 **"ckih"**。消费者节点总是通过索引引用时钟。名称应该反映设备的时钟输出信号名称。

`clock-indices`：如果节点中时钟的标识号不是从 0 开始线性增长的，这个属性可以将标识符映射到 `clock-output-names` 数组。

例如，如果我们有 `<&oscillator 1>` 和 `<&oscillator 3>` 两个时钟：
```
	oscillator {
		compatible = "myclocktype";
		#clock-cells = <1>;
		clock-indices = <1>, <3>;
		clock-output-names = "clka", "clkb";
	}
```

这确保我们在 `clock-output-names` 中没有任何空字符串。

## 时钟消费者

必需的属性：

`clocks`：`phandle` 和时钟指示符对的列表，设备的每个时钟输入一对。注意：如果时钟提供者指定其 `#clock-cells` 为 0，则只需要对的 `phandle` 部分。

可选的属性：
`clock-names`：时钟输入名称字符串的列表，***按与时钟属性相同的顺序排序***。消费者驱动程序将使用 `clock-names` 来匹配时钟输入名称和时钟指示符。

`clock-ranges`：空属性表示子节点可以从该节点继承命名时钟。用于总线节点向其子节点提供时钟。

例如：
```
    device {
        clocks = <&osc 1>, <&ref 0>;
        clock-names = "baud", "register";
    };
```

这表示具有两个时钟输入的设备，两个时钟输入名称为 **"baud"** 和 **"register"**。**"baud"** 时钟连接到 `&osc` 设备的输出 1，**"register"** 时钟连接到 `&ref` 设备的输出 0。

## 示例
```
    /* external oscillator */
    osc: oscillator {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency  = <32678>;
        clock-output-names = "osc";
    };

    /* phase-locked-loop device, generates a higher frequency clock
     * from the external oscillator reference */
    pll: pll@4c000 {
        compatible = "vendor,some-pll-interface"
        #clock-cells = <1>;
        clocks = <&osc 0>;
        clock-names = "ref";
        reg = <0x4c000 0x1000>;
        clock-output-names = "pll", "pll-switched";
    };

    /* UART, using the low frequency oscillator for the baud clock,
     * and the high frequency switched PLL output for register
     * clocking */
    uart@a000 {
        compatible = "fsl,imx-uart";
        reg = <0xa000 0x1000>;
        interrupts = <33>;
        clocks = <&osc 0>, <&pll 1>;
        clock-names = "baud", "register";
    };
```

这个 DT 片段定义了三个设备：一个提供低频参考时钟的外部振荡器，一个生成更高频率时钟信号的 PLL 设备，和一个 UART。

* 振荡器是固定频率的，它提供了一个时钟输出，名为 **"osc"**。
* PLL 同时是时钟提供者和时钟消费者。它使用外部振荡器生成的时钟信号，并提供两个输出信号 (**"pll"** 和 **"pll-switched"**)。
* UART 把它的 **"baud"** 时钟连接到外部振荡器，把它的 **"register"** 时钟连接到 PLL 时钟 (**"pll-switched"** 信号)。

## 分配时钟父节点和速率

某些平台可能需要初始配置默认父时钟和时钟频率。这样的配置可以在设备树节点中，通过 `assigned-clocks`，`assigned-clock-parents` 和 `assigned-clock-rates` 属性指定。`assigned-clock-parents` 属性应该以 `phandle` 和时钟指示符对的形式包含父时钟的列表，`assigned-clock-rates` 属性应该包含一个以 Hz 为单位的频率的列表。这两个属性应该对应于 `assigned-clocks` 属性中列出的时钟。

若要跳过设置时钟的父时钟或频率，则应将其对应的项设置为 0，如果其后没有任何非零项，则可以省略该项。
```
    uart@a000 {
        compatible = "fsl,imx-uart";
        reg = <0xa000 0x1000>;
        ...
        clocks = <&osc 0>, <&pll 1>;
        clock-names = "baud", "register";

        assigned-clocks = <&clkcon 0>, <&pll 2>;
        assigned-clock-parents = <&pll 2>;
        assigned-clock-rates = <0>, <460800>;
    };
```

在这个例子中，`<&pll 2>` 时钟被设置为时钟 `<&clkcon 0>` 的父时钟，且给 `<&pll 2>` 时钟分配了频率值 460800 Hz。

通过消费时钟的设备节点配置时钟的父时钟和频率，只能用于具有单个用户的时钟。禁止在多个消费者节点中为共享的时钟指定冲突的父时钟或频率配置。

公共时钟的配置，将会影响多个消费者设备，可以类似地在时钟提供者节点中指定。

## 受保护的时钟

某些平台或固件可能不会将所有时钟完全暴露给 OS ，例如在这些时钟由运行在 ARM 安全执行级别的驱动程序所使用的情况下。在设备树中，这种配置可以通过 `protected-clocks` 属性以时钟指示符列表的形式来指定。这种属性只应该在提供受保护时钟的节点中指定。

```
   clock-controller@a000f000 {
        compatible = "vendor,clk95;
        reg = <0xa000f000 0x1000>
        #clocks-cells = <1>;
        ...
        protected-clocks = <UART3_CLK>, <SPI5_CLK>;
   };
```

[原文](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/devicetree/bindings/clock/clock-bindings.txt?h=v5.10.186)