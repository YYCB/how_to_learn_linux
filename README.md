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

### 入门 & 准备
| # | 章节 | 核心内容 |
|---|------|----------|
| [00](./00-学习路线/index.html) | **学习路线** | 阶段规划、时间表、推荐资源 |
| [01](./01-经典版本选择/index.html) | **经典版本选择** | 0.11 / 2.6.0 / 5.x / 6.x 对比 |
| [02](./02-环境搭建/index.html) | **环境搭建** | QEMU + GDB + clangd + ftrace + perf |

### 核心子系统
| # | 章节 | 核心内容 |
|---|------|----------|
| [03](./03-进程管理/index.html) | **进程管理** | task_struct、fork/CoW、上下文切换、状态机 |
| [04](./04-内存管理/index.html) | **内存管理** | 多级页表、Buddy、Slab、NUMA、OOM、THP |
| [05](./05-文件系统/index.html) | **文件系统** | VFS 四对象、Page Cache、ext4、io_uring |
| [06](./06-系统调用/index.html) | **系统调用** | entry_SYSCALL_64、vDSO、seccomp、自定义 syscall |
| [07](./07-设备驱动/index.html) | **设备驱动** | 完整 char driver、设备树、中断上下半部 |
| [08](./08-网络子系统/index.html) | **网络子系统** | sk_buff、TCP 状态机、netfilter、XDP |
| [09](./09-同步机制/index.html) | **同步机制** | atomic/spinlock/mutex/RCU/percpu/futex/lockdep |

### 专家级深入
| # | 章节 | 核心内容 |
|---|------|----------|
| [10](./10-CFS调度器/index.html) | **CFS 调度器** | vruntime、红黑树、调度类、EAS、cgroup 调度 |
| [11](./11-容器与命名空间/index.html) | **容器与命名空间** | 8 种 NS、cgroups v2、OverlayFS、75 行 mini-docker |
| [12](./12-eBPF与可观测性/index.html) | **eBPF 与可观测性** | Verifier/JIT/Maps、kprobe/XDP/tc、bpftrace、Cilium |
| [13](./13-中断与异常/index.html) | **中断与异常** | IDT、APIC、softirq/workqueue/threaded IRQ、IPI |
| [14](./14-启动流程深入/index.html) | **启动流程** | UEFI → GRUB → initramfs → start_kernel → systemd |
| [15](./15-内核调试与性能/index.html) | **内核调试与性能** | ftrace/perf/KASAN/lockdep/bpftrace/kdump/livepatch |

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
