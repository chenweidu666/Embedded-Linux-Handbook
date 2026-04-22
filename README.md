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
| **Chapter 5** | 05-kernel-build.md | 内核配置编译、模块加载、内核裁剪、DT Overlay、Panic 排错 | 3-4 hours |
| **Chapter 6** | 06-char-device-driver.md | file_operations、ioctl、copy_from_user、中断处理、NPU 驱动实战 | 4-5 hours |
| **Chapter 7** | 07-bus-device-model.md | Bus/Device/Driver 模型、Platform Bus、devm_* 资源管理 | 3-4 hours |
| **Chapter 8** | 08-memory-dma.md | kmalloc/vmalloc、ioremap、DMA 映射、CMA、零拷贝 mmap | 4-5 hours |
| **Chapter 9** | 09-rootfs-build.md | Rootfs 组成、BusyBox、Buildroot、Yocto、NPU SDK 集成 | 3-4 hours |
| **Chapter 10** | 10-network-debug.md | 网络配置、SSH、gdbserver、strace、/proc /sys 调试、日志系统 | 2-3 hours |
| **Chapter 11** | 11-npu-stack.md | NPU 架构、模型编译 (ONNX→RKNN)、SDK 集成、性能优化 | 4-6 hours |
| **Chapter 12** | 12-production-deploy.md | CPU 调频、Watchdog、systemd、OTA 升级、生产监控 | 2-3 hours |

**总计**: 12 章，36-49 小时完成（纯 Linux 嵌入式硬核内容）

---

## 🔗 进阶学习
- **硬件理论**: 移步 `npu/basics/` (NPU 计算原理、内存架构)
- **实操与调试**: 移步 `npu/debug/` (NPU 运行排错、工具使用)
- **C 语言基础**: 移步 [Embedded-C-Basics](https://github.com/chenweidu666/Embedded-C-Basics) (类型、指针、数据结构、并发)