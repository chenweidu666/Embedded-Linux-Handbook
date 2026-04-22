# HCM C4000 (RISC-V) NPU 板端调试指南

## 1. 硬件与环境信息
* **板卡型号**: HCM C4000 (16-core)
* **处理器架构**: RISC-V 64-bit (`riscv64`)
* **目标 IP**: `192.168.3.118` (默认)
* **操作系统**: 嵌入式 Linux (BusyBox), Kernel `6.6.0`

---

## 2. 基础连接配置

### 2.1 串口登录
* **工具**: MobaXterm / SecureCRT
* **参数**: COM 口 (如 COM1), 波特率 `115200`, 数据位 `8`, 停止位 `1`, 无校验
* **状态**: 看到 `~ #` 提示符即为成功

### 2.2 网络配置
板子上电后，若 IP 未配置，需手动设置：
```bash
ifconfig eth0 192.168.3.118
```
> 注意：配置后请 ping 一下 Windows 主机，确认物理链路通畅。

---

## 3. 文件传输 (TFTP 方案)
由于板子是精简系统 (BusyBox)，推荐使用 TFTP 传输文件，速度快且兼容性好。

### 3.1 Windows 端配置
1. 打开 **Tftpd64** (或 Tftpd32)。
2. **Current Directory**: 设置为要传输文件所在的 Windows 目录。
3. **Server interfaces**: 选择与板子在同一网段的网卡 IP (如 `192.168.3.x`)。

### 3.2 板端下载命令
在板子上执行：
```bash
tftp -g -r 文件名 -l 本地保存名 <主机IP>
```
**⚠️ 重要避坑**：
* `-r` (Remote) 后面**只能写文件名** (如 `payload.tar.gz`)！
* **绝对不要**写 Windows 路径 (如 `D:\cw\payload.tar.gz`)，否则会报 `File not found`。

### 3.3 传输文件夹
TFTP 不支持直接传文件夹，必须打包。
1. Windows 端将文件夹压缩为 `payload.tar.gz`。
2. 板端下载并解压：
```bash
tar -xzf payload.tar.gz
```

---

## 4. NPU 驱动验证与加载

### 4.1 快速验证 (推荐)
出厂系统通常已预置 NPU 驱动和测试文件，路径通常为：
```bash
cd /app_bin/npu
./vpm_run -s sample_ok.txt
```
* **成功标志**: 终端打印 `Test output 0 passed` 且 `vpm run ret=0`。
* **设备节点**: 正常情况下 `ls /dev/vipcore` 应存在。

### 4.2 手动加载驱动 (进阶)
如果 `/dev/vipcore` 不存在，说明驱动未加载。

#### 情况 A: 正常加载
```bash
insmod vipcore.ko
ls -l /dev/vipcore  # 确认节点生成
```

#### 情况 B: 版本号不匹配 (常见坑)
**报错**: `version magic '6.6.0+' should be '6.6.0'`
**原因**: 驱动编译内核版本与板子内核版本仅差一个 `+` 号。
**解决**: BusyBox 的 `insmod` 不支持 `--force`，需使用 `modprobe -f` 强制加载。

```bash
# 1. 建立模块目录 (欺骗 modprobe 查找路径)
mkdir -p /lib/modules/$(uname -r)
cp vipcore.ko /lib/modules/$(uname -r)/

# 2. 强制加载
modprobe -f vipcore

# 3. 验证
ls -l /dev/vipcore
```

---

## 5. NPU 推理测试

### 5.1 基本运行命令
进入模型所在目录 (包含 `.nb` 文件和 `.so` 库)：
```bash
export LD_LIBRARY_PATH=$(pwd)
./vpm_run -s sample.txt -l 1
```

### 5.2 关键参数说明
| 参数 | 说明 | 示例 |
| :--- | :--- | :--- |
| `-s` | **必须**。指定样本配置文件 (包含 NBG 路径等) | `-s sample.txt` |
| `-l` | 循环推理次数 | `-l 10` |
| `-d` | 设备索引 (默认 0) | `-d 0` |
| `-c` | 核心索引 | `-c 0` |

### 5.3 运行日志解读
* **成功**: `Test output 0 passed.` (与 Golden 数据比对通过)
* **失败**:
    * `viphal_os_init: open device /dev/vipcore` -> 驱动没加载成功。
    * `failed to init vip` -> 驱动版本不对或库文件缺失。

---

## 6. 常见问题排查 (Troubleshooting)

### 6.1 脚本执行报错
* **现象**: `./cmd.sh: line 1: insmod vipcore.ko: No such file or directory`
* **原因**: Windows 编辑的脚本带有 `\r` (回车符)，Linux 不识别。
* **解决**:
```bash
sed -i 's/\r$//' cmd.sh
```

### 6.2 程序报 `not found`
* **现象**: 文件明明在，运行 `./vpm_run` 却说 `not found`。
* **原因**:
    1. **架构不匹配**: 板子是 RISC-V，不能用 x86 或 ARM 编译的程序。
    2. **缺动态库**: 缺少解释器或依赖的 `.so` 库。
* **解决**: 检查 `ldd vpm_run` (若有此命令) 或确认 SDK 版本是否为 RISC-V 版。

### 6.3 找不到 `file` 或 `unrar` 命令
* 精简系统通常不预装这些工具，请改用 `tar -xzf` 或查看文件十六进制头 (`od -A x -t x1 文件名`) 判断架构。
