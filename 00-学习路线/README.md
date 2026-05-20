# 00 — Linux 内核学习路线

> 本章给出一条从零基础到能够阅读并修改内核源码的完整路线，
> 包含各阶段目标、推荐时间、学习方法与检验标准。

![Linux 内核架构总览](../assets/diagrams/arch-overview.svg)

---

## 总览：四个阶段

```
阶段一            阶段二              阶段三              阶段四
基础准备      经典版本精读        现代内核对比         实战 & 贡献
(4~6 周)      (8~12 周)          (8~12 周)           (持续进行)
   │               │                  │                   │
C语言复习  ──► Linux 0.11 全读  ──► Linux 2.6 核心  ──► 写驱动/模块
汇编入门       调试环境实验        与现代内核对比       提交 patch
OS 理论        子系统笔记          性能分析工具        参与 LKML
```

---

## 阶段一：基础准备（4~6 周）

### 1.1 C 语言与内核编程习惯

| 知识点 | 要求 |
|--------|------|
| 指针、函数指针 | **必须熟练**，内核大量使用 |
| 位操作、宏 | 理解 `#define`、`do { } while(0)` 等惯用法 |
| 内联汇编（GCC AT&T 语法） | 能读懂 `asm volatile` |
| 链接脚本（vmlinux.lds） | 了解段布局即可 |

```c
/* 内核中的典型宏写法示例 */
#define container_of(ptr, type, member) ({          \
    const typeof(((type *)0)->member) *__mptr = (ptr); \
    (type *)((char *)__mptr - offsetof(type, member)); })
```

### 1.2 操作系统理论

重点掌握以下概念（对照书本 + 自己画图）：

- **进程 vs 线程**：PCB 结构、状态机
- **虚拟内存**：页表、TLB、缺页中断
- **调度算法**：FIFO、Round-Robin、CFS 思想
- **文件系统**：inode、目录树、VFS 抽象
- **同步原语**：互斥量、信号量、条件变量

### 1.3 工具链熟悉

```bash
# 必须会用的工具
gcc / clang       # 编译
gdb               # 调试（含 remote target）
make / kbuild     # 内核构建系统
objdump / nm      # 符号与反汇编
readelf           # ELF 格式分析
strace / ltrace   # 系统调用跟踪
perf              # 性能分析
```

---

## 阶段二：Linux 0.11 精读（8~12 周）

### 为什么选 Linux 0.11？

| 特性 | Linux 0.11 | 现代内核 |
|------|-----------|---------|
| 代码行数 | ~14,000 行 | ~3,000 万行 |
| 架构 | 仅 x86 | 多架构 |
| 调度器 | 简单时间片 | CFS + 实时 |
| 内存管理 | 段页式 | 纯分页 + NUMA |
| 文件系统 | Minix FS | ext4/btrfs/... |
| 学习曲线 | **平缓** | 陡峭 |

Linux 0.11 是 **包含现代 Linux 所有核心机制的最小完整实现**，
非常适合初学者完整读完并在脑海中形成整体图像。

### 0.11 源码目录结构

```
linux-0.11/
├── boot/           # 启动代码（bootsect.s, setup.s, head.s）
├── init/           # main.c — 内核入口
├── kernel/         # 核心：进程、调度、信号、系统调用
│   ├── sched.c     # 调度器
│   ├── fork.c      # 进程创建
│   ├── exit.c      # 进程退出
│   ├── signal.c    # 信号处理
│   └── sys.c       # 系统调用实现
├── mm/             # 内存管理
│   ├── memory.c    # 物理内存与页表
│   └── page.s      # 缺页中断汇编入口
├── fs/             # 文件系统（Minix FS）
│   ├── inode.c
│   ├── namei.c
│   └── buffer.c
├── lib/            # 内核库函数
├── include/        # 头文件
└── Makefile
```

### 8 周精读计划

| 周次 | 内容 | 对应源文件 |
|------|------|-----------|
| 第 1 周 | 启动流程：BIOS → 实模式 → 保护模式 | `boot/bootsect.s`, `boot/setup.s`, `boot/head.s` |
| 第 2 周 | 内核入口与初始化 | `init/main.c` |
| 第 3 周 | 内存管理：段页式 + 缺页 | `mm/memory.c`, `mm/page.s` |
| 第 4 周 | 进程创建与切换 | `kernel/fork.c`, `kernel/sched.c` |
| 第 5 周 | 系统调用机制 | `kernel/system_call.s`, `kernel/sys.c` |
| 第 6 周 | 信号与进程间通信 | `kernel/signal.c`, `kernel/exit.c` |
| 第 7 周 | 文件系统：Minix FS | `fs/` 全部文件 |
| 第 8 周 | 设备驱动：磁盘、终端 | `kernel/blk_drv/`, `kernel/chr_drv/` |

---

## 阶段三：Linux 2.6.0 核心精读（8~12 周）

### 为什么选 2.6.0？

- 2003 年发布，是**现代内核架构成型的里程碑版本**
- 引入了 O(1) 调度器（后被 CFS 替代，但结构类似）
- 引入了 `kobject`/`sysfs` 设备模型
- 代码量约 600 万行，子系统边界清晰
- 与现代内核（5.x/6.x）差异可对比学习

### 重点子系统对比学习路径

```
Linux 0.11                    Linux 2.6.0
─────────                     ───────────
sched.c (简单轮转)    ──►   kernel/sched.c (O1 调度器)
                                    │
                                    ▼
                            Linux 5.x: kernel/sched/fair.c (CFS)

fork.c               ──►   kernel/fork.c (引入线程、命名空间)

mm/memory.c          ──►   mm/memory.c + mm/slab.c + mm/vmalloc.c

fs/ (Minix FS)       ──►   fs/ext2/ + fs/vfs/ (完整 VFS 层)
```

---

## 阶段四：实战与贡献

### 4.1 写内核模块

```c
/* 最简单的 Hello World 内核模块 */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, Kernel World!\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, Kernel World!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

### 4.2 参与社区

1. 订阅 LKML（Linux Kernel Mailing List）
2. 从修复文档错误、Checkpatch 警告开始
3. 找到感兴趣的子系统，跟踪其 git log
4. 提交 trivial fix，学习邮件 patch 格式

---

## 每日学习建议

```
┌─────────────────────────────────────────────┐
│  每天 2~3 小时的建议安排                       │
│                                             │
│  0:00 ─ 0:30  复习昨天笔记，提炼关键概念       │
│  0:30 ─ 1:30  精读源码（不超过 200 行/天）     │
│  1:30 ─ 2:00  画出当天阅读部分的数据结构图     │
│  2:00 ─ 2:30  在 QEMU 中验证（打印、断点）     │
└─────────────────────────────────────────────┘
```

> **核心原则**：不要只读，要 **画图 + 实验**。
> 每读完一个函数，先画出它操作的数据结构，
> 再在 QEMU 中用 GDB 跟踪验证自己的理解。

---

## 推荐书单与资源

### 📚 书籍（按学习阶段排序）

| 书名 | 作者 | 定位 | 说明 |
|------|------|------|------|
| **Understanding the Linux Kernel** | Bovet & Cesati | 核心参考 | 最全面的内核原理书，覆盖 2.6 版本；建议配合源码阅读 |
| **Linux Kernel Development** | Robert Love | 入门进阶 | 文字友好，覆盖所有主要子系统，是入门 2.6 的最佳伴侣 |
| **Linux Device Drivers** | Corbet, Rubini, Kroah-Hartman | 驱动开发 | 驱动开发圣经，**免费在线版**：[lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/) |
| **Professional Linux Kernel Architecture** | Wolfgang Mauerer | 深度进阶 | 覆盖面极广，适合有一定基础后系统性补全 |
| **The Linux Programming Interface** | Michael Kerrisk | 系统编程 | 从用户空间角度理解系统调用，与内核知识互补 |
| **Linux内核完全注释** | 赵炯 | 中文入门 | 专门针对 Linux 0.11，逐行注释，非常适合初学者 |

### 🌐 在线资源

| 资源 | 链接 | 必读程度 |
|------|------|---------|
| **kernel.org 官方文档** | [kernel.org/doc](https://www.kernel.org/doc/html/latest/) | ⭐⭐⭐ 必读 |
| **LWN.net** | [lwn.net](https://lwn.net) | ⭐⭐⭐ **必读**，内核开发最权威的新闻/技术文章站 |
| **kernelnewbies.org** | [kernelnewbies.org](https://kernelnewbies.org) | ⭐⭐ 推荐，每个版本的变更摘要极有价值 |
| **Elixir Cross Referencer** | [elixir.bootlin.com](https://elixir.bootlin.com) | ⭐⭐⭐ 必备，在线代码索引，支持跨版本符号跳转 |
| **Linux Kernel Map** | [makelinux.github.io/kernel/map](https://makelinux.github.io/kernel/map/) | ⭐⭐ 直观的内核结构可视化地图 |
| **LKML（内核邮件列表）** | [lkml.org](https://lkml.org) | ⭐⭐ 了解真实开发讨论 |

### 🎬 视频课程

| 课程 | 说明 |
|------|------|
| **MIT 6.828 Operating System Engineering** | 最顶级的操作系统课，xv6 实验贯穿全课，配合内核学习效果极佳 |
| **David Beazley — Python Concurrency From the Ground Up** | 虽然是 Python 演讲，但对并发、GIL、内核调度的讲解极有深度 |
| **Linux Foundation 培训课程** | LFD420（内核内部原理）适合有基础后系统学习 |
| **Bootlin 内核培训材料** | [bootlin.com/training](https://bootlin.com/training/) 的 PDF 免费下载，质量极高 |

---

## 检验学习效果的问题清单

> 能独立、准确地回答以下问题，说明你已达到**专家级理解**。
> 建议每完成一个阶段后，尝试不查资料口述答案。

### 进程管理
1. `fork()` 返回两次的原理是什么？内核是如何让父子进程分别返回不同值的？（提示：`pt_regs.eax`）
2. 线程和进程在 Linux 内核中的**本质区别**是什么？`clone()` 的哪些 flag 决定了"线程"？
3. `task_struct` 中的 `thread_info` 存放在哪里？为什么能用 `esp & ~0x1FFF` 快速找到它？
4. 僵尸进程（Zombie）是如何产生的？为什么 `wait()` 必须被调用？孤儿进程如何处理？

### 内存管理
5. x86_64 四级页表的结构是什么？一次虚拟地址翻译需要几次内存访问？TLB 的作用是什么？
6. 伙伴系统解决什么问题？Slab 分配器又解决什么问题？两者如何协作？
7. 缺页中断（Page Fault）有哪几种情况？内核分别如何处理：匿名页、文件映射页、写时复制？
8. OOM Killer 是如何选择"牺牲"进程的？`/proc/PID/oom_score` 的计算方式是什么？

### 文件系统
9. VFS 的四大核心对象（`super_block`/`inode`/`dentry`/`file`）分别代表什么？它们的生命周期有何不同？
10. `dcache`（目录项缓存）的作用是什么？路径查找 `/home/user/file` 需要几次磁盘 IO？
11. `fsync()` 和 `fdatasync()` 的区别是什么？ext4 三种 journal 模式（journal/ordered/writeback）各有什么代价？
12. `io_uring` 相比传统 `read()`/`write()` 的核心优势是什么？SQE 和 CQE 各代表什么？

### 系统调用
13. `int 0x80` 和 `sysenter` 两种系统调用机制的开销差异在哪里？vDSO 如何绕过内核态切换？
14. `seccomp BPF` 的工作原理是什么？容器运行时（如 Docker）如何利用它限制系统调用？
15. 系统调用返回用户态前，内核会检查哪些"待办事项"？（信号、调度、TIF 标志）

### 网络子系统
16. `sk_buff` 的 `head/data/tail/end` 四个指针的含义是什么？为什么要有 headroom？
17. 一个 TCP 数据包从网卡到用户进程 `recv()` 返回，经过了哪些内核函数？（完整调用链）
18. netfilter 的五个 hook 点分别在网络栈的哪个位置？`iptables` 的 `PREROUTING/INPUT/OUTPUT` 各对应哪个 hook？

### 同步机制
19. `spin_lock` 和 `mutex` 的本质区别是什么？在中断上下文为什么不能使用 `mutex`？
20. RCU 的"宽限期"（Grace Period）是如何定义的？`synchronize_rcu()` 和 `call_rcu()` 的区别是什么？在什么场景下选择哪个？
