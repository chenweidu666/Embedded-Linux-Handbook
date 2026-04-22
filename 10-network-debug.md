# 第 10 章 - 网络与远程调试
<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结
本章讲解嵌入式网络配置、串口调试 vs SSH、gdbserver 远程调试、strace/ltrace 用户态跟踪、/proc & /sys 文件系统调试技巧，以及日志系统配置。

---

## 📖 本章内容
1. 嵌入式网络配置：eth0/wlan0、静态 IP、DHCP
2. 串口调试 (minicom / picocom) vs 网络 SSH
3. gdbserver 远程调试：Host 上 gdb 调试 Target 进程
4. strace / ltrace 用户态跟踪
5. /proc & /sys 文件系统调试技巧
6. 日志系统：dmesg、journald、kernel log level 控制

---

## 1. 嵌入式网络配置：eth0/wlan0、静态 IP、DHCP

### 1.1 查看网络接口

```bash
# 查看所有网络接口
ip link show

# 输出示例:
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
# 2: eth0: <BROADCAST,MULTICAST> mtu 1500 ...
# 3: wlan0: <BROADCAST,MULTICAST> mtu 1500 ...
```

### 1.2 配置静态 IP

```bash
# 临时配置 (重启失效)
ip addr add 192.168.1.100/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.1.1

# 永久配置 (使用 /etc/network/interfaces)
cat > /etc/network/interfaces << 'EOF'
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8
EOF

# 重启网络服务
/etc/init.d/networking restart
```

### 1.3 DHCP 自动获取

```bash
# 使用 udhcpc (BusyBox 内置)
udhcpc -i eth0

# 使用 dhclient (完整版)
dhclient eth0
```

### 1.4 网络调试命令

```bash
ping 192.168.1.1          # 测试连通性
ifconfig                  # 查看 IP 地址 (旧版)
ip addr show              # 查看 IP 地址 (新版)
netstat -tunlp            # 查看监听端口
ss -tunlp                 # 查看监听端口 (新版，更快)
tcpdump -i eth0           # 抓包分析
```

---

## 2. 串口调试 (minicom / picocom) vs 网络 SSH

### 2.1 串口调试

**适用场景**：系统启动初期 (网络未就绪)、内核 Panic 调试、无网络环境。

```bash
# 使用 picocom (推荐，轻量)
sudo picocom -b 115200 /dev/ttyUSB0

# 使用 minicom
sudo minicom -D /dev/ttyUSB0 -b 115200

# 退出: Ctrl+A, 然后 X
```

### 2.2 SSH 远程登录

**适用场景**：系统正常运行后，日常开发调试。

```bash
# 目标板启用 SSH (Dropbear)
dropbear -R -p 22

# Host 端连接
ssh root@192.168.1.100
```

### 2.3 对比

| 特性 | 串口调试 | SSH |
|------|----------|-----|
| 启动阶段 | Bootloader 即可用 | 需网络就绪 |
| 速度 | 115200 bps (慢) | 网络速度 (快) |
| 日志捕获 | 完整内核启动日志 | 仅用户态日志 |
| 文件传输 | 困难 (需 xmodem/ymodem) | 简单 (scp/sftp) |
| 多窗口 | 单窗口 | 多窗口/并行 |

**建议**：开发阶段保留串口用于查看内核日志，日常操作使用 SSH。

---

## 3. gdbserver 远程调试：Host 上 gdb 调试 Target 进程

### 3.1 原理

```mermaid
graph LR
    Host[Host PC: aarch64-linux-gnu-gdb] <--TCP/串口--> Target[Target: gdbserver]
    Target --> Process[调试进程]
    
    style Host fill:#bbf,stroke:#333
    style Target fill:#fbb,stroke:#333
```

### 3.2 目标板启动 gdbserver

```bash
# 编译时添加调试信息
aarch64-linux-gnu-gcc -g -o my_app my_app.c

# 拷贝到目标板
scp my_app root@192.168.1.100:/root/

# 启动 gdbserver
gdbserver :1234 ./my_app
# 输出: Process ./my_app created; pid = 1234
# Listening on port 1234
```

### 3.3 Host 端连接调试

```bash
# 启动 gdb
aarch64-linux-gnu-gdb ./my_app

(gdb) target remote 192.168.1.100:1234
(gdb) break main
(gdb) continue
(gdb) print variable
(gdb) step
(gdb) quit
```

### 3.4 调试已运行进程

```bash
# 目标板附加到运行中的进程
gdbserver :1234 --attach 1234

# Host 端连接
aarch64-linux-gnu-gdb
(gdb) target remote 192.168.1.100:1234
```

---

## 4. strace / ltrace 用户态跟踪

### 4.1 strace：系统调用跟踪

```bash
# 跟踪程序的所有系统调用
strace ./my_app

# 跟踪特定系统调用
strace -e open,read,write ./my_app

# 附加到运行中的进程
strace -p 1234

# 输出示例:
# open("/etc/npu.conf", O_RDONLY) = 3
# read(3, "clock=800000000\n", 4096) = 16
# ioctl(3, NPU_IOC_RUN, 0xffffd5c0) = 0
```

### 4.2 ltrace：库函数调用跟踪

```bash
# 跟踪动态库函数调用
ltrace ./my_app

# 输出示例:
# malloc(1024) = 0xaaaae8c01000
# strcpy(0xaaaae8c01000, "hello") = 0xaaaae8c01000
# printf("Result: %d\n", 42) = 12
```

### 4.3 实战：定位程序卡死

```bash
# 程序卡住时，使用 strace 查看阻塞在哪个系统调用
strace -p 1234 -T -tt

# 输出:
# 14:32:15.123456 read(3,  <unfinished ...>
# 14:32:25.654321 <... read resumed> 0xffffd5c0, 4096) = ? ERESTARTSYS
```
→ 程序阻塞在 `read()` 调用，等待设备数据。

---

## 5. /proc & /sys 文件系统调试技巧

### 5.1 /proc：进程与内核信息

```bash
# 查看 CPU 信息
cat /proc/cpuinfo

# 查看内存使用
cat /proc/meminfo

# 查看某个进程的详细信息
cat /proc/1234/status
cat /proc/1234/maps      # 内存映射
cat /proc/1234/cmdline   # 命令行参数

# 动态调整内核参数
echo 1 > /proc/sys/net/ipv4/ip_forward  # 启用 IP 转发
```

### 5.2 /sys：设备与驱动信息

```bash
# 查看 NPU 设备信息
ls /sys/devices/platform/ffbc0000.npu/

# 查看驱动绑定
cat /sys/bus/platform/drivers/rk-npu/bind

# 查看时钟频率
cat /sys/kernel/debug/clk/clk_npu/clk_rate

# 查看 DMA 缓冲区统计
cat /sys/kernel/debug/dma_buf/bufinfo
```

### 5.3 实用调试命令

```bash
# 查看内核模块信息
lsmod
modinfo npu_driver

# 查看中断统计
cat /proc/interrupts | grep npu

# 查看内存分配 (slab)
cat /proc/slabinfo | grep kmalloc
```

---

## 6. 日志系统：dmesg、journald、kernel log level 控制

### 6.1 dmesg：内核日志

```bash
# 查看内核日志
dmesg

# 实时跟踪
dmesg -w

# 按级别过滤
dmesg --level=err,warn

# 清空日志
dmesg -c
```

### 6.2 内核日志级别

| 级别 | 宏 | 说明 |
|------|----|------|
| 0 | `KERN_EMERG` | 系统不可用 |
| 1 | `KERN_ALERT` | 必须立即行动 |
| 2 | `KERN_CRIT` | 严重错误 |
| 3 | `KERN_ERR` | 错误 |
| 4 | `KERN_WARNING` | 警告 |
| 5 | `KERN_NOTICE` | 正常但重要 |
| 6 | `KERN_INFO` | 信息 |
| 7 | `KERN_DEBUG` | 调试 |

### 6.3 控制控制台日志级别

```bash
# 查看当前级别
cat /proc/sys/kernel/printk
# 输出: 7 4 1 7 (当前 默认 最小 引导)

# 设置控制台只打印 ERROR 及以上
echo 3 > /proc/sys/kernel/printk

# 恢复打印所有级别
echo 8 > /proc/sys/kernel/printk
```

### 6.4 journald (systemd 系统)

```bash
# 查看系统日志
journalctl

# 查看内核日志
journalctl -k

# 实时跟踪
journalctl -f

# 查看特定服务日志
journalctl -u npu-service
```

---

## 🔧 实操练习

1. **gdbserver 远程调试**: 编译带 `-g` 标志的 NPU 测试程序，使用 gdbserver 远程调试，设置断点、查看变量、单步执行。
2. **strace 定位问题**: 编写一个会阻塞的程序，使用 `strace` 定位阻塞的系统调用，分析原因。
3. **/sys 文件系统探索**: 通过 `/sys` 查看 NPU 设备的时钟、电源、中断状态，尝试动态修改时钟频率。

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)