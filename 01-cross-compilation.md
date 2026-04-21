# 第 1 章 - 交叉编译与工具链

<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结

本章介绍了交叉编译工具链的配置、目标架构选择（arm/aarch64）、sysroot 使用和常见编译问题排查。


---

## 📖 本章内容

1. Host 与 Target 的概念
2. 为什么需要交叉编译？
3. 交叉编译工具链命名规则
4. 配置交叉编译环境 (PATH & Sysroot)
5. 常见编译错误排查

---

## 1. Host 与 Target 的概念

嵌入式开发涉及两个截然不同的机器：

- **Host (宿主机)**：你用来写代码、编译的电脑 (x86_64 PC, Ubuntu)。性能强大，工具链齐全。
- **Target (目标机)**：最终运行代码的硬件板 (ARM/RISC-V NPU Board)。资源受限，通常没有 GCC 编译器。

```mermaid
graph LR
    subgraph Host PC
    Editor[代码编辑器] --> Compiler[交叉编译器 arm-linux-gcc]
    end
    Compiler --> |生成 ARM ELF 文件| Binary(binary 文件)
    Binary --> |拷贝到 SD 卡/网络| Target[Target Board]
```

---

## 2. 为什么需要交叉编译？

### 2.1 性能与生态
- **编译速度慢**：在树莓派上编译 Linux Kernel 可能需要几小时，而在 PC 上只需几分钟。
- **缺失工具链**：精简的嵌入式文件系统 (Rootfs) 通常只包含运行时库 (libc)，不包含 `gcc` 和头文件，体积会暴增。

### 2.2 架构差异
Host 是 x86_64 架构，Target 是 ARM64/aarch32 架构。PC 上的 `gcc` 编译出来的指令集，板子根本无法执行。

---

## 3. 交叉编译工具链命名规则

工具链通常由三个部分组成：`arch-vendor-os-gcc`

```bash
arm-linux-gnueabihf-gcc
```

- **arm**: 目标架构 (arm, aarch64, riscv64)
- **linux**: 目标操作系统 (linux, none/裸机, uclinux)
- **gnueabihf**: 库格式 (gnu, musl) + EABI + **hf (Hard Float)**
  - `gnueabi`: 软件模拟浮点 (慢，不推荐)
  - `gnueabihf`: 硬件浮点 (快，NPU/ARM 板通用标准)

**注意**：NPU 芯片 (如 RK3399/RK3588) 通常运行 64 位 Linux，对应的工具链是 **`aarch64-linux-gnu-gcc`**。

---

## 4. 配置交叉编译环境

### 4.1 安装与解压

```bash
# 下载工具链 (示例)
tar -xvf gcc-arm-11.2-aarch64-linux-gnu.tar.xz -C /opt/

# 添加到环境变量
export PATH=$PATH:/opt/gcc-arm-11.2/bin
export CROSS_COMPILE=aarch64-linux-gnu-
```

### 4.2 Sysroot (系统根目录)

**Sysroot** 是目标机文件系统在 Host 上的映射（包含头文件和库）。如果不指定 Sysroot，编译器找不到 `<linux/ioctl.h>` 等头文件。

```bash
# 使用 --sysroot 参数
aarch64-linux-gnu-gcc hello.c -o hello \
    --sysroot=/opt/sysroot/aarch64
```

### 4.3 验证安装

```bash
$ aarch64-linux-gnu-gcc --version
aarch64-linux-gnu-gcc (GCC) 11.2.0

$ file hello
hello: ELF 64-bit LSB executable, ARM aarch64
# 确认架构是 ARM64，而不是 x86-64
```

---

## 5. 常见编译错误排查

### 5.1 找不到头文件 (fatal error: xxx.h: No such file)
- **原因**：未设置 Sysroot，或者 Sysroot 路径错误。
- **解决**：检查 `--sysroot` 是否指向了包含 `usr/include` 的目录。

### 5.2 找不到库文件 (cannot find -lxxx)
- **原因**：Sysroot 里的 `usr/lib` 没有对应的 `.so` 或 `.a` 文件。
- **解决**：确认库文件是否存在于 Sysroot 中，或使用 `-L` 指定额外库路径。

### 5.3 运行时报错：No such file or directory (但文件明明存在！)
- **原因**：这是最经典的坑。**ELF 解释器缺失**。编译时指定的动态链接器 (如 `/lib/ld-linux-aarch64.so.1`) 在板子上不存在，或者版本不匹配。
- **解决**：确保板子上的 `/lib` 目录下有对应架构的 `ld-linux`，或者静态编译 (`-static`)。

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
