# Linux 环境常见问题 (FAQ)

<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结

本文档汇总了嵌入式 Linux 开发环境配置、主机交互（VMware/Ubuntu）及基础使用中的常见问题与解决方案。

## 📖 本章内容

1. [VMware 拖拽文件失效](#1-vmware-拖拽文件失效)
2. [VMware 共享文件夹不显示](#2-vmware-共享文件夹不显示)

---

## 1. VMware 拖拽文件失效

**现象**：在 VMware 设置中已勾选“启用拖放”，但无法将主机文件拖入 Ubuntu 24.04 虚拟机。

**原因**：
1.  缺失桌面版 VMware Tools 组件。
2.  Ubuntu 24.04 默认使用 **Wayland** 显示协议，与 VMware 拖拽功能兼容性差。

**解决方案**：

**步骤 1：重装桌面版工具**
打开终端，安装 `open-vm-tools-desktop`（注意后缀）：
```bash
sudo apt install --reinstall open-vm-tools-desktop -y
```
安装完成后重启服务或重启虚拟机：
```bash
sudo reboot
```

**步骤 2：切换到 Xorg (关键)**
如果安装工具后仍无效，说明是 Wayland 问题。
1.  **注销**当前用户（Logout），**不要重启**。
2.  在登录界面右下角点击**齿轮图标**。
3.  选择 **"Ubuntu on Xorg"** 登录。
4.  登录后拖拽功能通常即可恢复。

**备选方案**：
使用 VMware 共享文件夹。在设置中开启共享文件夹后，虚拟机内可通过 `/mnt/hgfs/` 直接访问主机目录，比拖拽更稳定。

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
