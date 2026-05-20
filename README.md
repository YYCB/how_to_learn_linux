# 🐧 如何学习 Linux 内核 — 系统性学习指南

> **目标**：通过阅读经典内核版本（Linux 0.11 & Linux 2.6.0）的源码，
> 从零开始理解现代 Linux 操作系统的核心框架与设计哲学，
> 并通过专家级章节深入 CFS、容器、eBPF、调试等现代主题。

---

## 🌐 推荐阅读方式：HTML 网站

本指南已升级为 **完整的 HTML 网站**，包含：

- 🎨 暗色主题 + 语法高亮的代码块
- 📊 30+ 个内嵌 SVG 架构图（页表、CFS 红黑树、TCP 状态机、容器结构…）
- 📑 16 章覆盖从入门到专家级的完整知识体系
- 🧭 左侧固定侧边栏，章节间一键跳转

**用浏览器直接打开 [`index.html`](./index.html) 即可开始学习。**

> 仍保留每章的 README.md 作为快速文本参考。

---

## 📚 目录

> 每章均提供两种格式：点击章节名在线阅读 **Markdown**（GitHub 直接渲染），或用本地浏览器打开 `index.html` 获得完整 SVG 图表体验。

### 入门 & 准备
| # | 章节 | 核心内容 | SVG 图表 |
|---|------|----------|---------|
| [00](./00-学习路线/README.md) | **学习路线** | 阶段规划、时间表、书单、20道自测题 | 架构总览 |
| [01](./01-经典版本选择/README.md) | **经典版本选择** | 0.11/1.0/2.4/2.6/3.10/4.19/5.15/6.1 全景对比 | 架构总览 |
| [02](./02-环境搭建/README.md) | **环境搭建** | QEMU + GDB + clangd + ftrace + perf + kdump | — |

### 核心子系统
| # | 章节 | 核心内容 | SVG 图表 |
|---|------|----------|---------|
| [03](./03-进程管理/README.md) | **进程管理** | task_struct、fork/CoW、上下文切换汇编、状态机 | 虚拟地址空间布局 |
| [04](./04-内存管理/README.md) | **内存管理** | 多级页表、Buddy/Slab/SLUB、NUMA、kswapd、OOM、THP | x86_64 四级页表 |
| [05](./05-文件系统/README.md) | **文件系统** | VFS 四对象、Page Cache 回写、ext4 journal、io_uring | VFS 对象模型 |
| [06](./06-系统调用/README.md) | **系统调用** | entry_SYSCALL_64 汇编、vDSO、seccomp BPF、新增 syscall | syscall 路径图 |
| [07](./07-设备驱动/README.md) | **设备驱动** | 完整字符驱动、kobject/sysfs、设备树、MSI/DMA | 设备模型层次 |
| [08](./08-网络子系统/README.md) | **网络子系统** | sk_buff、收包路径、TCP 状态机、netfilter 5hook、XDP | TCP 握手图 |
| [09](./09-同步机制/README.md) | **同步机制** | atomic/spinlock/mutex/seqlock/RCU/percpu/futex/lockdep | 同步机制全景 |

### 专家级深入
| # | 章节 | 核心内容 | SVG 图表 |
|---|------|----------|---------|
| [10](./10-CFS调度器/README.md) | **CFS 调度器** | vruntime 公式、红黑树、5调度类、EAS、cgroup 层次调度 | CFS 红黑树 |
| [11](./11-容器与命名空间/README.md) | **容器与命名空间** | 8 种 NS、cgroups v2、OverlayFS、seccomp、mini-docker | 容器内部结构 |
| [12](./12-eBPF与可观测性/README.md) | **eBPF 与可观测性** | Verifier/JIT/Maps、XDP、CO-RE、bpftrace、Cilium/Falco | eBPF 完整架构 |
| [13](./13-中断与异常/README.md) | **中断与异常** | IDT/APIC/MSI、softirq/workqueue/threaded IRQ、IPI、hrtimer | 中断处理路径 |
| [14](./14-启动流程深入/README.md) | **启动流程深入** | BIOS/UEFI、GRUB2、解压、head_64.S、start_kernel()、systemd | 启动全流程图 |
| [15](./15-内核调试与性能/README.md) | **内核调试与性能** | ftrace/perf/FlameGraph/KASAN/lockdep/kdump+crash/livepatch | 调试工具全景 |

---

## 🗺️ 总体架构一览

```
┌─────────────────────────────────────────────────────────┐
│                   用户空间 (User Space)                   │
│  应用程序  Shell  libc  系统工具  ……                       │
└──────────────────────┬──────────────────────────────────┘
                       │  系统调用接口 (syscall)
┌──────────────────────▼──────────────────────────────────┐
│                   内核空间 (Kernel Space)                  │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │ 进程管理  │  │ 内存管理  │  │ 文件系统  │  │ 网络栈  │  │
│  │ scheduler│  │  VMM/MM  │  │   VFS    │  │TCP/IP   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘  │
│       │              │              │              │       │
│  ┌────▼──────────────▼──────────────▼──────────────▼───┐  │
│  │              设备驱动 & 硬件抽象层 (HAL)               │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                     硬件 (Hardware)                       │
│   CPU   内存   磁盘   网卡   键盘/鼠标   ……                 │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 学习目标

完成本指南后，你将能够：

1. **读懂** Linux 内核源码中任意子系统的核心逻辑
2. **理解** 进程调度、虚拟内存、文件系统等核心机制
3. **调试** 内核模块，用 GDB + QEMU 单步跟踪内核执行流
4. **对比** Linux 0.11 与现代内核的演进路径
5. **编写** 简单的内核模块与字符设备驱动

---

## 📖 推荐书单

| 书名 | 作者 | 适用阶段 |
|------|------|---------|
| 《Linux内核完全注释》| 赵炯 | 入门（基于 0.11）|
| 《深入理解Linux内核》| Bovet & Cesati | 进阶（基于 2.6）|
| 《Linux设备驱动程序》| Corbet 等 | 驱动开发 |
| 《Linux内核设计与实现》| Robert Love | 综合理解 |
| 《深入Linux内核架构》| Mauerer | 深度参考 |
| 《操作系统：精髓与设计原理》| Stallings | 理论基础 |

---

## 🔗 重要资源

- 源码在线浏览：<https://elixir.bootlin.com/linux>
- Linux 0.11 源码：<https://github.com/karottc/linux-0.11>
- Linux 2.6.0 源码：<https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/linux-2.6.0.tar.gz>
- 官方文档：<https://www.kernel.org/doc/html/latest/>
- LKML 邮件列表：<https://lkml.org/>
