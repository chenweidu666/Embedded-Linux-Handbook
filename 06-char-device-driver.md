# 第 6 章 - 字符设备驱动深入
<link rel="stylesheet" href="../npu/assets/print-b5.css">

## 📝 本章总结
本章深入讲解 Linux 字符设备驱动开发：`file_operations` 结构体详解、`ioctl` 命令定义规范、用户态与内核态数据拷贝、同步与互斥机制、中断处理，以及编写一个完整的 NPU 推理字符设备驱动。

---

## 📖 本章内容
1. file_operations 结构体详解 (open/read/write/ioctl/release)
2. `ioctl` 命令定义规范 (`_IO`, `_IOR`, `_IOW`, `_IOWR`)
3. 用户态与内核态数据拷贝 (`copy_from_user` / `copy_to_user`)
4. 同步与互斥：mutex、spinlock、completion
5. 中断处理 (request_irq、顶半部/底半部、tasklet、workqueue)
6. 实战：编写一个完整的 NPU 推理字符设备驱动

---

## 1. file_operations 结构体详解

字符设备驱动通过实现 `file_operations` 结构体，将用户态的系统调用映射到内核态函数。

### 1.1 核心回调函数

```c
#include <linux/fs.h>

static const struct file_operations npu_fops = {
    .owner          = THIS_MODULE,
    .open           = npu_open,
    .release        = npu_release,
    .read           = npu_read,
    .write          = npu_write,
    .unlocked_ioctl = npu_ioctl,
    .mmap           = npu_mmap,
    .poll           = npu_poll,
};
```

### 1.2 各回调函数职责

| 回调 | 触发时机 | 典型用途 |
|------|----------|----------|
| `open` | 用户调用 `open("/dev/npu", ...)` | 初始化设备、分配资源、检查权限 |
| `release` | 用户调用 `close(fd)` | 释放资源、停止硬件 |
| `read` | 用户调用 `read(fd, buf, len)` | 从设备读取数据 (如推理结果) |
| `write` | 用户调用 `write(fd, buf, len)` | 向设备发送数据 (如输入图像) |
| `ioctl` | 用户调用 `ioctl(fd, cmd, arg)` | 设备控制命令 (配置、重置、查询状态) |
| `mmap` | 用户调用 `mmap(...)` | 将设备内存映射到用户空间 (零拷贝) |
| `poll` | 用户调用 `poll/select/epoll` | 非阻塞 I/O 等待 (数据就绪通知) |

---

## 2. `ioctl` 命令定义规范

`ioctl` 是用户态与驱动交互的"万能接口"。必须遵循内核规范的命令编码，避免冲突。

### 2.1 命令宏定义

```c
#include <linux/ioctl.h>

// 定义魔术数 (唯一标识该设备，通常取 ASCII 字符)
#define NPU_IOC_MAGIC  'N'

// 命令定义
#define NPU_IOC_RESET      _IO(NPU_IOC_MAGIC, 0)          // 无参数
#define NPU_IOC_SET_CLOCK  _IOW(NPU_IOC_MAGIC, 1, uint32_t) // 写参数
#define NPU_IOC_GET_STATUS _IOR(NPU_IOC_MAGIC, 2, uint32_t) // 读参数
#define NPU_IOC_RUN_TASK   _IOWR(NPU_IOC_MAGIC, 3, NpuTask_t) // 读写参数

#define NPU_IOC_MAXNR 3 // 最大命令号
```

### 2.2 命令类型说明

| 宏 | 方向 | 说明 |
|----|------|------|
| `_IO(type, nr)` | 无 | 仅触发操作，无数据传输 |
| `_IOR(type, nr, datatype)` | 内核 → 用户 | 从设备读取数据 |
| `_IOW(type, nr, datatype)` | 用户 → 内核 | 向设备写入数据 |
| `_IOWR(type, nr, datatype)` | 双向 | 先写入配置，后读取结果 |

### 2.3 ioctl 处理函数

```c
static long npu_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    void __user *argp = (void __user *)arg;
    
    switch (cmd) {
        case NPU_IOC_RESET:
            npu_hw_reset();
            return 0;
            
        case NPU_IOC_SET_CLOCK: {
            uint32_t clock;
            if (copy_from_user(&clock, argp, sizeof(clock)))
                return -EFAULT;
            return npu_set_clock(clock);
        }
        
        case NPU_IOC_GET_STATUS: {
            uint32_t status = npu_get_status();
            if (copy_to_user(argp, &status, sizeof(status)))
                return -EFAULT;
            return 0;
        }
        
        default:
            return -ENOTTY; // 未知命令
    }
}
```

---

## 3. 用户态与内核态数据拷贝

内核空间和用户空间使用不同的虚拟地址映射，**绝对不能直接解引用用户态指针**。

### 3.1 `copy_from_user` 与 `copy_to_user`

```c
// 用户态 → 内核态
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
// 返回值: 未能拷贝的字节数 (0 表示成功)

// 内核态 → 用户态
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
// 返回值: 未能拷贝的字节数 (0 表示成功)
```

### 3.2 错误处理

```c
int npu_write(struct file *filp, const char __user *buf, size_t count, loff_t *pos) {
    if (count > MAX_DATA_SIZE)
        return -EINVAL;
    
    char *kbuf = kmalloc(count, GFP_KERNEL);
    if (!kbuf)
        return -ENOMEM;
    
    // 安全拷贝
    if (copy_from_user(kbuf, buf, count)) {
        kfree(kbuf);
        return -EFAULT; // 用户态指针无效
    }
    
    // 处理数据...
    process_npu_data(kbuf, count);
    
    kfree(kbuf);
    return count; // 返回实际写入的字节数
}
```

**⚠️ 安全警告**：直接访问 `buf[i]` 会导致内核 Panic（页错误），因为用户态地址在内核页表中不存在。

---

## 4. 同步与互斥：mutex、spinlock、completion

### 4.1 Mutex (互斥锁)

适用于可能睡眠的场景（如等待硬件响应）。

```c
#include <linux/mutex.h>

static DEFINE_MUTEX(npu_mutex);

int npu_open(struct inode *inode, struct file *filp) {
    mutex_lock(&npu_mutex);
    if (npu_is_busy()) {
        mutex_unlock(&npu_mutex);
        return -EBUSY;
    }
    npu_mark_busy();
    mutex_unlock(&npu_mutex);
    return 0;
}
```

### 4.2 Spinlock (自旋锁)

适用于极短临界区、不能睡眠的场景（如中断处理函数）。

```c
#include <linux/spinlock.h>

static DEFINE_SPINLOCK(npu_lock);
static int npu_task_count = 0;

void npu_irq_handler(int irq, void *dev_id) {
    unsigned long flags;
    spin_lock_irqsave(&npu_lock, flags); // 关中断 + 加锁
    npu_task_count++;
    spin_unlock_irqrestore(&npu_lock, flags); // 解锁 + 恢复中断
}
```

### 4.3 Completion (完成量)

用于等待某个事件完成（如 DMA 传输结束）。

```c
#include <linux/completion.h>

static DECLARE_COMPLETION(npu_done);

// 用户态阻塞等待
ssize_t npu_read(struct file *filp, char __user *buf, size_t count, loff_t *pos) {
    wait_for_completion(&npu_done); // 阻塞直到 completion 被触发
    return copy_to_user(buf, result_buf, count) ? -EFAULT : count;
}

// 中断中触发
void npu_irq_handler(int irq, void *dev_id) {
    complete(&npu_done); // 唤醒等待的线程
}
```

**同步机制选择指南：**
```mermaid
graph TD
    NeedSync{需要同步？}
    NeedSync -->|否| NoLock[无锁]
    NeedSync -->|是| CanSleep{可能睡眠？}
    CanSleep -->|是| Wait{等待事件？}
    Wait -->|是| Comp[Completion]
    Wait -->|否| Mutex[Mutex]
    CanSleep -->|否 (中断/原子上下文)| Spin[Spinlock]
```

---

## 5. 中断处理 (request_irq、顶半部/底半部)

### 5.1 注册中断

```c
#include <linux/interrupt.h>

static irqreturn_t npu_irq_handler(int irq, void *dev_id) {
    uint32_t status = readl(npu_base + NPU_STATUS_REG);
    
    if (status & NPU_IRQ_DONE) {
        // 清除中断标志
        writel(NPU_IRQ_DONE, npu_base + NPU_IRQ_CLEAR_REG);
        // 触发完成量
        complete(&npu_done);
        return IRQ_HANDLED;
    }
    
    return IRQ_NONE; // 不是本设备的中断
}

// 在 probe 函数中注册
int npu_probe(struct platform_device *pdev) {
    int irq = platform_get_irq(pdev, 0);
    int ret = request_irq(irq, npu_irq_handler, IRQF_SHARED, "npu_driver", dev);
    if (ret) {
        dev_err(&pdev->dev, "Failed to request IRQ %d\n", irq);
        return ret;
    }
    return 0;
}
```

### 5.2 顶半部与底半部

中断处理函数（顶半部）必须快速执行，耗时操作应推迟到底半部。

| 底半部机制 | 适用场景 | 执行上下文 |
|------------|----------|------------|
| **Tasklet** | 轻量级、同类型中断序列化 | 软中断上下文 (原子) |
| **Workqueue** | 可能睡眠的操作 (如 `kmalloc`、I/O) | 进程上下文 (可睡眠) |
| **Threaded IRQ** | 现代推荐方案，简化顶/底半部编写 | 内核线程 |

**Workqueue 示例：**
```c
#include <linux/workqueue.h>

static struct workqueue_struct *npu_wq;
static DECLARE_WORK(npu_work, npu_work_func);

static irqreturn_t npu_irq_handler(int irq, void *dev_id) {
    // 顶半部：仅记录状态，调度底半部
    queue_work(npu_wq, &npu_work);
    return IRQ_HANDLED;
}

static void npu_work_func(struct work_struct *work) {
    // 底半部：处理耗时操作 (可睡眠)
    process_inference_results();
    complete(&npu_done);
}
```

---

## 6. 实战：编写一个完整的 NPU 推理字符设备驱动

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/mutex.h>

#define NPU_NAME "npu"
#define NPU_IOC_MAGIC 'N'
#define NPU_IOC_RUN_TASK _IOWR(NPU_IOC_MAGIC, 0, NpuTask_t)

typedef struct {
    uint32_t task_id;
    uint32_t input_addr;
    uint32_t output_addr;
    uint32_t status;
} NpuTask_t;

static dev_t npu_dev_num;
static struct cdev npu_cdev;
static struct class *npu_class;
static DEFINE_MUTEX(npu_mutex);
static DECLARE_COMPLETION(npu_done);

static long npu_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    NpuTask_t task;
    if (cmd != NPU_IOC_RUN_TASK) return -ENOTTY;
    if (copy_from_user(&task, (void __user *)arg, sizeof(task))) return -EFAULT;
    
    mutex_lock(&npu_mutex);
    npu_hw_start_task(&task);
    mutex_unlock(&npu_mutex);
    
    wait_for_completion(&npu_done); // 阻塞等待 NPU 完成
    task.status = npu_get_status();
    
    if (copy_to_user((void __user *)arg, &task, sizeof(task))) return -EFAULT;
    return 0;
}

static const struct file_operations npu_fops = {
    .owner = THIS_MODULE,
    .unlocked_ioctl = npu_ioctl,
};

static int __init npu_init(void) {
    alloc_chrdev_region(&npu_dev_num, 0, 1, NPU_NAME);
    cdev_init(&npu_cdev, &npu_fops);
    cdev_add(&npu_cdev, npu_dev_num, 1);
    npu_class = class_create(THIS_MODULE, NPU_NAME);
    device_create(npu_class, NULL, npu_dev_num, NULL, NPU_NAME);
    return 0;
}

static void __exit npu_exit(void) {
    device_destroy(npu_class, npu_dev_num);
    class_destroy(npu_class);
    cdev_del(&npu_cdev);
    unregister_chrdev_region(npu_dev_num, 1);
}

module_init(npu_init);
module_exit(npu_exit);
MODULE_LICENSE("GPL");
```

---

## 🔧 实操练习

1. **实现 ioctl 命令集**: 为 NPU 驱动添加 `NPU_IOC_RESET`、`NPU_IOC_GET_VERSION`、`NPU_IOC_LOAD_MODEL` 命令，并编写用户态测试程序。
2. **中断+Completion 同步**: 模拟 NPU 硬件中断（使用定时器），在驱动中使用 `wait_for_completion` 阻塞等待，验证用户态 `ioctl` 调用的同步行为。
3. **Workqueue 底半部改造**: 将第 6 节的驱动改为 Workqueue 底半部模式，验证中断处理函数执行时间缩短。

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)