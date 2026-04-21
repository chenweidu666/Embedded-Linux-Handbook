# 第 3 章 - 设备树 (Device Tree) 编写基础

<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结

本章介绍了设备树（DTS/DTB）语法、节点和属性定义、GPIO/时钟/中断配置和常见 DTS 修改方法。


---

## 📖 本章内容

1. 为什么需要设备树？
2. DTS 与 DTB 的关系
3. 设备树基本语法
4. NPU 节点配置示例
5. 常用属性解析 (Reg, Interrupts, Clocks)

---

## 1. 为什么需要设备树？

在 ARM 架构中，没有 PC 那样的 ACPI 标准或 BIOS。Kernel 怎么知道板子上挂载了什么硬件？

- **过去**：写死在 C 代码里 (Hard-coded in `arch/arm/mach-xxx/board-xxx.c`)。
  - **缺点**：每出一款新板子，Kernel 源码就要新增一个文件，杂乱且难以维护。
- **现在**：设备树 (Device Tree)。
  - **原理**：硬件描述与代码分离。用纯文本描述硬件连接，编译成二进制 (DTB)，由 Bootloader 传递给 Kernel。
  - **优势**：同一个 Kernel 镜像可以跑在不同板子上，只需加载不同的 DTB。

---

## 2. DTS 与 DTB 的关系

```text
.dts  (文本，人类可读) --> dtc (编译器) --> .dtb (二进制，机器可读) --> Bootloader 传给 Kernel
```

- **DTS (Device Tree Source)**：开发者编辑的文件。
- **DTB (Device Tree Blob)**：最终烧录的文件。
- **DTSI (Device Tree Source Include)**：公共部分 (SoC 级别的定义)，被 DTS 文件 `#include` 引用。

---

## 3. 设备树基本语法

### 3.1 节点 (Node) 与 属性 (Property)

设备树是一个树形结构，每个硬件设备是一个节点：

```dts
/ {  // 根节点
    model = "Verisilicon NPU Board";

    npu {  // 子节点
        compatible = "verisilicon,vip-npu";
        reg = <0x40000000 0x10000>;
        interrupts = <0 32 4>;
    };
};
```

### 3.2 关键属性：compatible

这是设备树匹配驱动的**身份证**。

- 驱动侧会声明一个 `of_device_id` 表，包含它支持的字符串。
- Kernel 启动时，会扫描 DTS 节点的 `compatible` 属性。
- 如果**匹配**，则调用驱动的 `probe()` 函数初始化设备。

```c
// 驱动里的匹配表
const struct of_device_id npu_ids[] = {
    { .compatible = "verisilicon,vip-npu" },
    { /* sentinel */ }
};
```

---

## 4. NPU 节点配置示例

在 NPU 开发中，你通常需要在 DTS 里添加或修改如下节点：

```dts
npu: npu@ff9a0000 {
    compatible = "verisilicon,vip-npu";
    reg = <0x0 0xff9a0000 0x0 0x10000>;  // 寄存器物理地址及长度
    interrupts = <GIC_SPI 65 IRQ_TYPE_LEVEL_HIGH>; // 中断号与触发类型
    clocks = <&cru 123>, <&cru 124>;     // 时钟控制器引用
    clock-names = "npu_clk", "npu_axi_clk";
    power-domains = <&power RK32_PD_NPU>; // 电源域
    status = "okay";                     // 启用该设备
};
```

### 4.1 属性拆解
- **reg**: 定义硬件在内存中的映射。前两个值是地址 (64 位分两段)，后两个是大小。
- **interrupts**: 定义中断控制器 (GIC) 的 ID。
- **clocks**: 引用 SoC 的时钟树，Kernel 会自动帮你在驱动里开启/关闭这些时钟。

---

## 5. 常见陷阱 (Pitfalls)

| 陷阱 | 现象 | 修复 |
|------|------|------|
| **Status = "disabled"** | 驱动 Probe 失败，打印 "probe deferred" | 改为 `"okay"`，或确保父节点也是 `okay` |
| **地址写错** | 访问硬件时触发 Data Abort (内核崩溃) | 仔细核对 Datasheet，确保地址和长度正确 |
| **电源域未配置** | 读寄存器全是 `0xFF` 或 `0x00` | 检查 Power Domain 是否已开启 |
| **Compatible 不匹配** | 驱动加载了，但 Probe 没跑 | 检查 DTS 和 代码里的 `compatible` 字符串是否一字不差 |

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
