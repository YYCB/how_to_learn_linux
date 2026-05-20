# 07 — 设备驱动

> 驱动程序是内核与硬件之间的桥梁。
> 本章从**设备模型 → 字符设备 → 块设备 → 驱动开发流程**，
> 对照 Linux 0.11（直接操作）与 Linux 2.6.0（统一设备模型）拆解。

---

## 1. 设备分类

```
Linux 设备分为三类：

┌──────────────────────────────────────────────────────────┐
│  字符设备（Character Device）                             │
│  · 以字节流方式访问                                       │
│  · 无缓冲（或少缓冲）                                     │
│  · 例：键盘(/dev/tty)、串口(/dev/ttyS0)、鼠标            │
│  · 设备文件: /dev/xxx  主设备号:次设备号                  │
├──────────────────────────────────────────────────────────┤
│  块设备（Block Device）                                   │
│  · 以固定大小的块（通常 512B 或 4KB）随机访问             │
│  · 有内核缓冲（页缓存）                                   │
│  · 例：硬盘(/dev/sda)、U盘、SSD                          │
├──────────────────────────────────────────────────────────┤
│  网络设备（Network Device）                               │
│  · 通过 socket 接口访问，不在 /dev 中出现                 │
│  · 例：eth0、lo、wlan0                                    │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Linux 0.11：直接操作方式

### 2.1 终端（TTY）驱动结构

```c
/* kernel/chr_drv/tty_io.c */

/* 每个终端对应一个 tty_struct */
struct tty_struct {
    struct termios termios;       /* 终端参数（波特率、回显等）*/
    int pgrp;                     /* 前台进程组 */
    int stopped;                  /* 是否停止输出 */
    void (*write)(struct tty_struct *tty);  /* 输出函数指针 */
    struct tty_queue read_q;      /* 读缓冲队列 */
    struct tty_queue write_q;     /* 写缓冲队列 */
    struct tty_queue secondary;   /* 经行规处理后的队列 */
};

/* 终端读写（tty_read/tty_write 调用此）*/
void con_write(struct tty_struct *tty)   /* 控制台写 */
void rs_write(struct tty_struct *tty)    /* 串口写 */
```

### 2.2 硬盘驱动（HD）

```c
/* kernel/blk_drv/hd.c — Linux 0.11 硬盘驱动 */

/* 请求队列（I/O 请求）*/
static struct request request[NR_REQUEST];

/* 提交 I/O 请求 */
void add_request(struct blk_dev_struct *dev, struct request *req)
{
    /* 将请求插入电梯排序队列 */
    /* 电梯算法：按磁道号排序，减少磁头移动距离 */
    ...
    if (!(tmp = dev->current_request)) {
        dev->current_request = req;
        (dev->request_fn)();     /* 立即执行 */
    } else {
        /* 插入合适位置（按磁道号） */
        for (; tmp->next; tmp = tmp->next)
            if ((IN_ORDER(tmp, req) || !IN_ORDER(tmp, tmp->next))
                && IN_ORDER(req, tmp->next))
                break;
        req->next = tmp->next;
        tmp->next = req;
    }
}

/* 硬盘中断处理 */
void hd_interrupt(void)
{
    void (*handler)(void) = do_hd;
    do_hd = NULL;
    if (!handler)
        handler = unexpected_hd_interrupt;
    handler();                    /* 调用当前操作的完成处理函数 */
    enable_hd_dma();
}
```

---

## 3. Linux 2.6.0：统一设备模型

### 3.1 设备模型的核心对象

```
kobject（内核对象基类）
  ├── kset（kobject 的集合）
  ├── ktype（kobject 的操作集合）
  └── sysfs 文件系统节点

所有设备都通过 kobject 嵌入：

struct device {
    struct kobject kobj;          /* ← 内嵌 kobject */
    struct device *parent;        /* 父设备 */
    struct bus_type *bus;         /* 所在总线 */
    struct device_driver *driver; /* 对应驱动 */
    void *driver_data;            /* 驱动私有数据 */
    ...
};

struct device_driver {
    const char *name;
    struct bus_type *bus;
    int (*probe)(struct device *dev);   /* 探测设备 */
    int (*remove)(struct device *dev);  /* 移除设备 */
    ...
};
```

### 3.2 总线 - 驱动 - 设备 匹配机制

```
总线（bus_type）
  ├── 设备链表：USB 键盘、USB 鼠标、USB 网卡 ...
  └── 驱动链表：usbhid 驱动、usb-storage 驱动 ...

当设备插入时：
  1. 设备注册到总线设备链表
  2. 遍历总线驱动链表
  3. 调用 bus->match(device, driver) 尝试匹配
  4. 匹配成功 → driver->probe(device) → 驱动初始化硬件

当驱动加载时（insmod）：
  1. 驱动注册到总线驱动链表
  2. 遍历总线设备链表，尝试 match
  3. 找到匹配设备 → probe()
```

### 3.3 sysfs — 设备模型的可视化

```bash
# sysfs 将设备树暴露到文件系统
ls /sys/bus/pci/devices/
ls /sys/class/block/
ls /sys/class/net/

# 每个设备目录包含：
ls /sys/class/block/sda/
# driver    → 链接到 sata_sil 驱动
# power/    → 电源管理
# queue/    → 请求队列参数
# size      → 设备大小（512B 扇区数）
```

---

## 4. 字符设备驱动开发（2.6.0）

### 4.1 完整驱动模板

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>

#define MYDEV_NAME "mychardev"
#define MYDEV_MAJOR 240           /* 主设备号（或动态分配）*/
#define MYDEV_MINOR 0

/* 驱动私有数据 */
static struct {
    struct cdev cdev;
    char buf[256];
    int buf_len;
} mydev_data;

/* 1. open：打开设备 */
static int mydev_open(struct inode *inode, struct file *filp)
{
    /* 将驱动私有数据关联到 file */
    filp->private_data = &mydev_data;
    printk(KERN_INFO "%s: opened\n", MYDEV_NAME);
    return 0;
}

/* 2. release：关闭设备 */
static int mydev_release(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "%s: closed\n", MYDEV_NAME);
    return 0;
}

/* 3. read：从设备读数据（内核 → 用户）*/
static ssize_t mydev_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *ppos)
{
    int len = min(count, (size_t)mydev_data.buf_len);
    
    if (copy_to_user(buf, mydev_data.buf, len))
        return -EFAULT;
    
    *ppos += len;
    return len;
}

/* 4. write：向设备写数据（用户 → 内核）*/
static ssize_t mydev_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *ppos)
{
    int len = min(count, sizeof(mydev_data.buf) - 1);
    
    if (copy_from_user(mydev_data.buf, buf, len))
        return -EFAULT;
    
    mydev_data.buf[len] = '\0';
    mydev_data.buf_len = len;
    return len;
}

/* 5. ioctl：设备控制命令 */
static int mydev_ioctl(struct inode *inode, struct file *filp,
                        unsigned int cmd, unsigned long arg)
{
    switch (cmd) {
    case 0:  /* 自定义命令 0：清空缓冲 */
        mydev_data.buf_len = 0;
        break;
    default:
        return -EINVAL;
    }
    return 0;
}

/* file_operations：VFS 与驱动的接口 */
static struct file_operations mydev_fops = {
    .owner   = THIS_MODULE,
    .open    = mydev_open,
    .release = mydev_release,
    .read    = mydev_read,
    .write   = mydev_write,
    .ioctl   = mydev_ioctl,
};

/* 模块加载：注册设备 */
static int __init mydev_init(void)
{
    dev_t devno = MKDEV(MYDEV_MAJOR, MYDEV_MINOR);
    
    /* 注册设备号 */
    if (register_chrdev_region(devno, 1, MYDEV_NAME) < 0)
        return -ENODEV;
    
    /* 初始化并注册 cdev */
    cdev_init(&mydev_data.cdev, &mydev_fops);
    mydev_data.cdev.owner = THIS_MODULE;
    if (cdev_add(&mydev_data.cdev, devno, 1) < 0) {
        unregister_chrdev_region(devno, 1);
        return -ENODEV;
    }
    
    printk(KERN_INFO "%s: registered (major=%d)\n", MYDEV_NAME, MYDEV_MAJOR);
    return 0;
}

/* 模块卸载：注销设备 */
static void __exit mydev_exit(void)
{
    cdev_del(&mydev_data.cdev);
    unregister_chrdev_region(MKDEV(MYDEV_MAJOR, MYDEV_MINOR), 1);
    printk(KERN_INFO "%s: unregistered\n", MYDEV_NAME);
}

module_init(mydev_init);
module_exit(mydev_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My simple character device");
```

### 4.2 编译与测试

```bash
# Makefile
obj-m += mychardev.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

# 编译
make

# 加载模块
sudo insmod mychardev.ko

# 创建设备节点
sudo mknod /dev/mychardev c 240 0
sudo chmod 666 /dev/mychardev

# 测试
echo "Hello, Driver!" > /dev/mychardev
cat /dev/mychardev
# 输出：Hello, Driver!

# 查看内核日志
dmesg | tail

# 卸载模块
sudo rmmod mychardev
```

---

## 5. 驱动与内核交互机制

### 5.1 中断处理

```
硬件触发中断
    │
    ▼
CPU 查 IDT → 跳转到 request_irq() 注册的处理函数
    │
    ├─ 上半部（interrupt handler）：快速、不可休眠
    │     关键操作：清除中断状态、读写硬件寄存器
    │     保存数据到 tasklet/workqueue
    │
    └─ 下半部（bottom half）：可以稍微慢一些
          tasklet：软中断上下文，不可休眠
          workqueue：进程上下文，可以休眠

/* 注册中断处理函数 */
int request_irq(unsigned int irq,
                irqreturn_t (*handler)(int, void *, struct pt_regs *),
                unsigned long flags,
                const char *devname,
                void *dev_id);

/* 处理函数示例 */
irqreturn_t my_irq_handler(int irq, void *dev_id, struct pt_regs *regs)
{
    /* 读取硬件状态 */
    int status = inb(MY_STATUS_PORT);
    
    /* 调度 tasklet 处理数据 */
    tasklet_schedule(&my_tasklet);
    
    return IRQ_HANDLED;
}
```

### 5.2 DMA（直接内存访问）

```
不用 DMA：
  CPU 逐字节从硬件读数据到内存 → CPU 占用率高

用 DMA：
  1. CPU 设置 DMA 控制器（源、目标、长度）
  2. DMA 硬件自主搬运数据（CPU 可做其他事）
  3. DMA 完成后触发中断通知 CPU

/* 申请 DMA 缓冲区（需要物理连续内存）*/
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
/* cpu_addr: 内核虚拟地址，dma_handle: DMA 总线地址 */

/* 启动 DMA 传输 */
writel(dma_handle, dev_base + DMA_ADDR_REG);
writel(size, dev_base + DMA_SIZE_REG);
writel(DMA_START, dev_base + DMA_CTRL_REG);
```

---

## 6. 驱动调试技巧

```c
/* 内核打印（比 printf 更丰富）*/
printk(KERN_EMERG   "系统即将崩溃\n");  /* 0: 最高优先级 */
printk(KERN_ALERT   "需要立即处理\n");  /* 1 */
printk(KERN_CRIT    "严重错误\n");      /* 2 */
printk(KERN_ERR     "错误\n");          /* 3 */
printk(KERN_WARNING "警告\n");          /* 4 */
printk(KERN_NOTICE  "注意\n");          /* 5 */
printk(KERN_INFO    "信息\n");          /* 6 */
printk(KERN_DEBUG   "调试\n");          /* 7 */

/* /proc 接口（暴露驱动内部状态）*/
static int mydev_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "buffer: %s\n", mydev_data.buf);
    seq_printf(m, "length: %d\n", mydev_data.buf_len);
    return 0;
}
/* 注册：proc_create("mychardev", 0, NULL, &mydev_proc_fops) */
```

```bash
# 动态调试（Linux 2.6.30+）
echo "module mychardev +p" > /sys/kernel/debug/dynamic_debug/control

# 用 oops 分析崩溃
dmesg | grep "Oops"
# 使用 addr2line 或 gdb 定位崩溃位置
addr2line -e vmlinux 0xc01234ab
```
