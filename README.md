# Linux 嵌入式开发 - 章节导览

<link rel="stylesheet" href="../npu/assets/print-b5.css">

**专注于 ARM/RISC-V Linux 环境下的系统开发。从交叉编译到内核驱动，打通 NPU 软件栈的底层链路。**

---

## 📚 章节列表

| Chapter | Document | Content | Time |
|---------|----------|---------|------|
| **Chapter 1** | 01-cross-compilation.md | Host/Target 概念、工具链命名、Sysroot、编译排错 | 2-3 hours |
| **Chapter 2** | 02-boot-sequence.md | 启动流程、U-Boot 命令、Kernel 引导参数、Init 进程 | 3-4 hours |
| **Chapter 3** | 03-device-tree.md | DTS 语法、Compatible 匹配、NPU 节点写法、中断与时钟 | 3-4 hours |
| **Chapter 4** | 04-platform-driver.md | 驱动框架、Probe 机制、IO Remap、注册字符设备 (cdev) | 3-4 hours |

**总计**: 4 章，11-15 小时完成（纯 Linux 嵌入式硬核内容）

---

## 🔗 进阶学习
- **硬件理论**: 移步 `npu/basics/` (NPU 计算原理、内存架构)
- **实操与调试**: 移步 `npu/debug/` (NPU 运行排错、工具使用)
