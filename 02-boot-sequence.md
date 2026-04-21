# 第 2 章 - Linux 嵌入式启动流程

<link rel="stylesheet" href="../npu/assets/print-b5.css">

**理解板子从上电到 Shell 提示符出现，中间经历了什么。掌握 U-Boot 与 Kernel 的交互。**

---

## 📖 本章内容

1. 启动全景图 (Boot Chain)
2. 第一阶段：BootROM 与 SPL
3. 第二阶段：U-Boot 加载器
4. 第三阶段：Linux Kernel 解压
5. 第四阶段：Init 进程与用户空间

---

## 1. 启动全景图 (Boot Chain)

Linux 嵌入式系统的启动是一个接力赛过程：

```mermaid
graph LR
    PWR[上电] --> ROM[BootROM (固化代码)]
    ROM --> SPL[SPL (DRAM 初始化)]
    SPL --> UBOOT[U-Boot (加载 OS)]
    UBOOT --> KERNEL[Linux Kernel]
    KERNEL --> INIT[Init / Systemd]
    INIT --> SHELL[Shell / App]
```

每一阶段的目标都很简单：**把下一阶段加载到内存，然后跳过去执行**。

---

## 2. 第一阶段：BootROM 与 SPL

### 2.1 BootROM
- **位置**：CPU 内部的 Mask ROM，出厂固化，不可修改。
- **任务**：
  - 决定从哪个介质启动 (eMMC, SD 卡，SPI Flash, USB)。
  - 检查启动介质的合法性 (签名校验)。
  - 将 **SPL (Secondary Program Loader)** 加载到内部 SRAM。

### 2.2 SPL (Secondary Program Loader)
- **痛点**：芯片内部 SRAM 通常只有 100KB-500KB，装不下完整的 U-Boot，更无法操作外部大容量 DDR 内存。
- **任务**：
  - **初始化 DDR** (这是最关键的！)。
  - 加载完整的 U-Boot (FIT/u-boot.img) 到 DDR 内存中。
  - 跳转到 DDR 执行 U-Boot。

---

## 3. 第二阶段：U-Boot 加载器

U-Boot 是嵌入式世界的“通用引导程序”，支持多种架构和板子。

### 3.1 U-Boot 的核心工作
1. **硬件初始化**：时钟、串口 (打印 Log)、外设 (网口/存储)。
2. **环境变量加载**：从 Flash 读取配置 (启动参数、IP 地址等)。
3. **加载 Kernel**：根据环境变量，从存储设备读取 Kernel Image 和 Device Tree 到 RAM。
4. **启动 Kernel**：调用 `bootm` 命令，解压内核，传递设备树地址。

### 3.2 常用 U-Boot 命令

```bash
# 查看启动参数
printenv

# 修改启动参数
setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk0p2 rw'

# 查看文件
ls mmc 0:1

# 加载并启动内核
ext4load mmc 0:1 0x02000000 uImage
ext4load mmc 0:1 0x01000000 u.dtb
bootm 0x02000000 - 0x01000000
```

---

## 4. 第三阶段：Linux Kernel 解压

Kernel Image 通常是被压缩过的 (zImage 或 Image.gz)，以节省存储空间。

### 4.1 启动参数 (bootargs)
Kernel 启动时，U-Boot 会把配置好的启动参数传给它，告诉 Kernel：
- 串口在哪？(`console=ttyS0`)
- Rootfs 在哪？(`root=/dev/mmcblk0p2`)
- 文件系统类型？(`rootfstype=ext4`)

### 4.2 驱动初始化
Kernel 启动时，会遍历设备树 (DTS)，挂载平台下的驱动 (Platform Drivers)，如 NPU 驱动、DMA 驱动等。
**如果设备树没写对，这里驱动就不会 Probe (探测)。**

---

## 5. 第四阶段：Init 进程与用户空间

Kernel 加载完驱动后，启动第一个用户态进程 (PID 1)，通常是 `init` 或 `systemd`。

### 5.1 常见 Init 方案
- **BusyBox init**：极简，嵌入式最常用，只负责启动 `/etc/init.d/rcS` 脚本。
- **Systemd**：现代桌面系统标配，功能强大但在嵌入式中占用资源较多。
- **SysVinit**：传统的 `/etc/inittab` 方案，简单稳定。

### 5.2 启动完成
当 Init 脚本执行完毕，Shell 出现，你就可以登录板子，输入命令了。
此时，NPU 驱动通常已经加载到 `/dev/npu` 节点，等待用户态程序调用。

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
