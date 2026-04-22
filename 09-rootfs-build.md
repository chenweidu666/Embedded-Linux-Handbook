# 第 9 章 - Rootfs 构建与定制
<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结
本章讲解 Linux Rootfs 的组成、BusyBox 配置与编译、Buildroot 完整流程、Yocto/Poky 简介、自定义 Rootfs 添加 NPU SDK，以及 Rootfs 挂载失败的排错方法。

---

## 📖 本章内容
1. Rootfs 的组成：bin/sbin/lib/etc/dev/proc/sys
2. BusyBox 配置与编译 (嵌入式瑞士军刀)
3. Buildroot 完整流程：交叉编译工具链 + 内核 + Rootfs 一气呵成
4. Yocto/Poky 简介 (大型项目的标准方案)
5. 自定义 Rootfs：添加 NPU SDK 库、配置文件、启动脚本
6. 排错：Rootfs 挂载失败、libc 版本不匹配、缺少关键设备节点

---

## 1. Rootfs 的组成

Rootfs (Root Filesystem) 是 Linux 启动后挂载的第一个文件系统，包含系统运行所需的所有文件。

### 1.1 标准目录结构

```
/
├── bin/      # 基础用户命令 (ls, cp, sh)
├── sbin/     # 系统管理命令 (init, mount, ifconfig)
├── lib/      # 共享库 (libc.so, ld-linux.so)
├── usr/      # 用户程序与数据
│   ├── bin/  # 非基础命令 (gcc, python)
│   ├── lib/  # 非基础库
│   └── sbin/ # 非基础系统命令
├── etc/      # 配置文件 (passwd, fstab, init.d/)
├── dev/      # 设备节点 (由 udev/mdev 动态创建)
├── proc/     # 进程信息 (虚拟文件系统)
├── sys/      # 设备/驱动信息 (虚拟文件系统)
├── tmp/      # 临时文件
└── var/      # 可变数据 (log, run)
```

### 1.2 设备节点与虚拟文件系统

```mermaid
graph TD
    Boot[Kernel 启动] --> MountRoot[挂载 Rootfs]
    MountRoot --> Init[启动 PID 1 (init/systemd)]
    Init --> MountVFS[挂载虚拟文件系统]
    MountVFS --> Proc[mount -t proc proc /proc]
    MountVFS --> Sys[mount -t sysfs sysfs /sys]
    MountVFS --> Dev[mount -t tmpfs tmpfs /dev]
    Dev --> Mdev[mdev/udev 创建设备节点]
```

---

## 2. BusyBox 配置与编译 (嵌入式瑞士军刀)

### 2.1 什么是 BusyBox？

BusyBox 将 300+ 个常用 Unix 命令集成到一个二进制文件中，通过硬链接或软链接实现多命令调用，极大节省 Flash 空间。

### 2.2 编译 BusyBox

```bash
# 下载源码
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1

# 配置 (使用默认配置)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

# 图形化配置
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

# 编译
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# 安装到目标目录
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CONFIG_PREFIX=/tmp/rootfs install
```

### 2.3 关键配置项

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `Settings → Build Options → Build BusyBox as a static binary` | 静态编译 (不依赖共享库) | ✅ 推荐 |
| `Coreutils → ls, cp, mv, rm` | 基础文件操作 | ✅ 必选 |
| `Linux System Utilities → mount, insmod, mdev` | 系统管理 | ✅ 必选 |
| `Networking Utilities → ifconfig, udhcpc, ping` | 网络工具 | ✅ 推荐 |
| `Shells → ash` | 默认 Shell | ✅ 必选 |

---

## 3. Buildroot 完整流程：交叉编译工具链 + 内核 + Rootfs 一气呵成

### 3.1 为什么使用 Buildroot？

手动编译工具链、内核、BusyBox、库文件极其繁琐。Buildroot 通过配置文件自动化整个流程。

### 3.2 Buildroot 快速上手

```bash
# 下载 Buildroot
git clone https://git.buildroot.net/buildroot
cd buildroot

# 查看可用板级配置
ls configs/ | grep rockchip

# 加载配置
make rockchip_rk3588_defconfig

# 图形化配置
make menuconfig

# 开始构建 (首次可能需要 1-2 小时下载编译)
make -j$(nproc)
```

### 3.3 构建输出

```
output/
├── images/
│   ├── rootfs.ext4      # EXT4 Rootfs 镜像
│   ├── rootfs.tar       # Rootfs 压缩包
│   └── Image            # 内核镜像
├── host/                # 宿主工具链
└── target/              # 目标 Rootfs 目录 (可直接查看)
```

### 3.4 添加自定义包 (NPU SDK)

```bash
# 在 package/ 目录下创建 npu-sdk/
mkdir -p package/npu-sdk

# 创建 Config.in
cat > package/npu-sdk/Config.in << 'EOF'
config BR2_PACKAGE_NPU_SDK
    bool "npu-sdk"
    help
      NPU SDK library for Rockchip RK3588
EOF

# 创建 npu-sdk.mk
cat > package/npu-sdk/npu-sdk.mk << 'EOF'
NPU_SDK_VERSION = 1.0.0
NPU_SDK_SOURCE = npu-sdk-$(NPU_SDK_VERSION).tar.gz
NPU_SDK_SITE = http://example.com/downloads

define NPU_SDK_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/lib/libnpu.so $(TARGET_DIR)/usr/lib/
    $(INSTALL) -D -m 0644 $(@D)/include/npu.h $(TARGET_DIR)/usr/include/
endef

$(eval $(generic-package))
EOF

# 在 menuconfig 中启用
make menuconfig → Target packages → npu-sdk → 选中
```

---

## 4. Yocto/Poky 简介 (大型项目的标准方案)

### 4.1 Yocto vs Buildroot

| 特性 | Buildroot | Yocto/Poky |
|------|-----------|------------|
| 学习曲线 | 简单 (1 天上手) | 陡峭 (1-2 周) |
| 灵活性 | 中等 | 极高 |
| 构建时间 | 快 | 慢 |
| 适用场景 | 中小项目、快速原型 | 大型产品、长期维护 |
| 包管理 | 无 (静态 Rootfs) | 支持 (opkg/rpm) |

### 4.2 Yocto 核心概念

- **Layer**: 功能模块层 (如 `meta-rockchip`, `meta-npu`)
- **Recipe**: 软件包编译配方 (`.bb` 文件)
- **Image**: 最终生成的系统镜像配置
- **BitBake**: 构建引擎 (类似 Make，但更强大)

**NPU 开发建议**：初期使用 Buildroot 快速验证，产品化阶段迁移到 Yocto。

---

## 5. 自定义 Rootfs：添加 NPU SDK 库、配置文件、启动脚本

### 5.1 手动定制 Rootfs

```bash
# 1. 解压基础 Rootfs
tar xpf rootfs.tar.gz -C /tmp/rootfs

# 2. 添加 NPU SDK
cp -r npu-sdk/lib/* /tmp/rootfs/usr/lib/
cp -r npu-sdk/bin/* /tmp/rootfs/usr/bin/

# 3. 添加配置文件
cat > /tmp/rootfs/etc/npu.conf << 'EOF'
clock=800000000
voltage=1.1
debug=0
EOF

# 4. 添加启动脚本
cat > /tmp/rootfs/etc/init.d/S99npu << 'EOF'
#!/bin/sh
echo "Starting NPU driver..."
insmod /lib/modules/npu.ko
mknod /dev/npu c 240 0
chmod 666 /dev/npu
echo "NPU driver started."
EOF
chmod +x /tmp/rootfs/etc/init.d/S99npu

# 5. 重新打包
cd /tmp/rootfs
tar czpf ../custom-rootfs.tar.gz .
```

### 5.2 使用 `chroot` 测试 Rootfs

```bash
# 在 x86 主机上通过 qemu-user-static 测试 ARM64 Rootfs
sudo apt install qemu-user-static
sudo cp /usr/bin/qemu-aarch64-static /tmp/rootfs/usr/bin/

sudo chroot /tmp/rootfs /bin/sh
# 现在可以在 Rootfs 中执行 ARM64 命令!
```

---

## 6. 排错：Rootfs 挂载失败、libc 版本不匹配、缺少关键设备节点

### 6.1 挂载失败：`VFS: Unable to mount root fs`

**原因**：
- `bootargs` 中的 `root=` 参数错误。
- 文件系统类型不匹配 (如 `rootfstype=ext4` 但实际是 `squashfs`)。
- 存储设备驱动未编译进内核 (非模块)。

**排查步骤**：
```bash
# 检查 U-Boot 启动参数
printenv bootargs
# 输出: console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw

# 确认分区存在
fdisk -l /dev/mmcblk0
```

### 6.2 libc 版本不匹配

**现象**：运行程序报 `No such file or directory`，但文件确实存在。
**原因**：程序编译时使用的 libc 版本与 Rootfs 中的 `ld-linux.so` 不匹配。

```bash
# 检查程序依赖的动态链接器
readelf -l my_app | grep interpreter
# 输出: [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]

# 检查 Rootfs 中的链接器
ls -l /tmp/rootfs/lib/ld-linux-aarch64.so*
```

**解决**：确保交叉编译工具链的 sysroot 与 Rootfs 使用相同的 libc 版本。

### 6.3 缺少关键设备节点

**现象**：驱动加载成功，但 `/dev/npu` 不存在。
**原因**：`mdev`/`udev` 未自动创建节点，或 `mknod` 权限不足。

```bash
# 手动创建节点 (临时)
mknod /dev/npu c 240 0
chmod 666 /dev/npu

# 永久解决：添加 udev 规则
cat > /etc/udev/rules.d/99-npu.rules << 'EOF'
KERNEL=="npu", MODE="0666"
EOF
```

---

## 🔧 实操练习

1. **编译 BusyBox Rootfs**: 使用 BusyBox 构建最小 Rootfs，添加 `ls`, `sh`, `insmod` 命令，通过 NFS 挂载到开发板验证启动。
2. **Buildroot 添加自定义包**: 按照第 3 节的步骤，为 Buildroot 添加 NPU SDK 包，编译生成包含 SDK 的 Rootfs 镜像。
3. **Rootfs 排错实战**: 故意制造 3 种启动失败场景 (错误的 `root=` 参数、libc 不匹配、缺少 `/dev/console`)，记录排错过程。

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)