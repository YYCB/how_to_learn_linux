# 02 — 环境搭建：QEMU + GDB 调试内核

> 本章介绍如何搭建一个可以**单步调试**内核的实验环境，
> 让你可以在 GDB 中打断点、查看内核变量、跟踪函数调用。

---

## 总体架构

```
┌─────────────────────────────────────────────┐
│           你的开发机（Host）                   │
│                                             │
│  ┌─────────────────┐    ┌────────────────┐  │
│  │   QEMU 虚拟机    │◄──►│  GDB 调试器    │  │
│  │                 │    │                │  │
│  │  Linux 内核运行  │    │ 设置断点       │  │
│  │  （被调试目标）  │    │ 单步执行       │  │
│  │                 │    │ 查看变量/寄存器 │  │
│  └─────────────────┘    └────────────────┘  │
│         ▲                      ▲            │
│         │   TCP :1234          │            │
│         └──────────────────────┘            │
│              GDB Remote Protocol            │
└─────────────────────────────────────────────┘
```

---

## 一、安装依赖

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y \
    qemu-system-x86 \
    gcc \
    make \
    gdb \
    git \
    build-essential \
    libncurses-dev \
    flex \
    bison \
    libssl-dev \
    libelf-dev

# macOS（使用 Homebrew）
brew install qemu gdb
```

---

## 二、调试 Linux 0.11

### 2.1 获取带调试符号的 0.11 镜像

```bash
# 克隆含 Makefile 的仓库
git clone https://github.com/karottc/linux-0.11
cd linux-0.11

# 在 Makefile 中添加 -g 调试标志（已在新版 Makefile 中支持）
# 直接编译
make
# 生成文件：Image（内核镜像）、tools/system（含符号的 ELF）
```

### 2.2 启动 QEMU 并等待 GDB

```bash
# 终端 1：启动 QEMU（-s 开启 GDB 服务端 :1234，-S 暂停等待 GDB）
qemu-system-i386 \
    -m 16M \
    -boot a \
    -fda Image \
    -hda hdc-0.11.img \
    -s -S

# 终端 2：启动 GDB 并连接
gdb tools/system

# 在 GDB 中执行：
(gdb) target remote :1234
(gdb) break main          # 在 init/main.c:main 设断点
(gdb) continue
```

### 2.3 常用 GDB 命令速查

```
命令                        说明
──────────────────────────────────────────────
b <函数名>                  在函数入口设断点
b <文件>:<行号>             在指定行设断点
info breakpoints            列出所有断点
delete <编号>               删除断点

n                           单步执行（不进入函数）
s                           单步执行（进入函数）
c                           继续运行
finish                      运行到当前函数返回

p <变量>                    打印变量值
p *<指针>                   打印指针指向的结构
p/x <变量>                  以十六进制打印
info registers              显示所有寄存器
x/10i $eip                  反汇编当前指令附近

bt                          打印调用栈
frame <编号>                切换栈帧
list                        显示当前源码
```

---

## 三、调试 Linux 2.6.0

### 3.1 编译内核

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/linux-2.6.0.tar.gz
tar xf linux-2.6.0.tar.gz
cd linux-2.6.0

# 生成最小配置（适合 QEMU x86）
make defconfig

# 开启调试选项（重要！）
# 编辑 .config，或使用 menuconfig：
make menuconfig
# 进入：Kernel hacking
#   [*] Compile the kernel with debug info
#   [*] Compile the kernel with frame pointers

# 编译（-j 并行）
make -j$(nproc)

# 生成文件：
#   arch/i386/boot/bzImage  （可启动内核）
#   vmlinux                  （含符号，用于 GDB）
```

### 3.2 制作最小根文件系统（BusyBox）

```bash
# 编译 BusyBox（静态链接）
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar xf busybox-1.36.0.tar.bz2
cd busybox-1.36.0
make defconfig
# 设置静态链接：CONFIG_STATIC=y
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc)
make install
# 安装到 _install/

# 创建 initramfs
cd _install
mkdir -p dev proc sys
mknod dev/console c 5 1
mknod dev/null c 1 3
# 创建 init 脚本
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Welcome to minimal Linux!"
exec /bin/sh
EOF
chmod +x init

# 打包为 initramfs
find . | cpio -o --format=newc | gzip > ../initramfs.img
```

### 3.3 启动并调试

```bash
# 终端 1：启动 QEMU
qemu-system-i386 \
    -kernel arch/i386/boot/bzImage \
    -initrd initramfs.img \
    -append "console=ttyS0 nokaslr" \
    -nographic \
    -s -S

# 终端 2：GDB
gdb vmlinux
(gdb) target remote :1234
(gdb) break start_kernel      # 内核 C 代码入口
(gdb) continue
```

---

## 四、推荐的调试工作流

### 4.1 跟踪进程创建（fork）

```gdb
(gdb) break sys_fork
(gdb) commands
> # 每次进入 sys_fork 时自动打印
> printf "fork() called, current pid=%d\n", current->pid
> bt
> continue
> end
(gdb) continue
```

### 4.2 查看页表结构

```gdb
# 在 Linux 0.11 中，查看进程内存映射
(gdb) p current->ldt
(gdb) p *current->mm   # 在 2.6.0 中

# 打印页目录项（Linux 2.6.0）
(gdb) p/x ((unsigned long *)0xc0101000)[0]
```

### 4.3 跟踪系统调用

```gdb
# 在系统调用分发处设断点（0.11）
(gdb) break system_call
# 在 2.6.0 中
(gdb) break do_syscall_trace
```

---

## 五、VS Code + GDB 图形化调试

如果你更喜欢 GUI，可以配置 VS Code：

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Linux Kernel",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/vmlinux",
            "miDebuggerPath": "/usr/bin/gdb",
            "miDebuggerServerAddress": "localhost:1234",
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "设置反汇编风格",
                    "text": "set disassembly-flavor intel"
                },
                {
                    "description": "关闭确认",
                    "text": "set confirm off"
                }
            ]
        }
    ]
}
```

安装 VS Code 扩展：`C/C++`（Microsoft）

---

## 六、常见问题

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| `SIGTRAP`后无法继续 | 内核断点处理 | 用 `signal 0` 忽略信号 |
| 符号找不到 | 未加 `-g` 编译 | 重新编译并确认 `CFLAGS += -g` |
| `bzImage` 无法在 QEMU 启动 | 内核配置问题 | 用 `make defconfig` 重新生成配置 |
| GDB 连接被拒绝 | QEMU 未启动 `-s` | 检查 QEMU 启动参数 |
| 断点不命中 | 地址重定位 | 加 `nokaslr` 内核启动参数 |

> **提示**：调试内核时务必加 `nokaslr` 参数（关闭地址随机化），
> 否则每次启动内核地址不同，GDB 符号表会对不上。

---

## 七、KGDB：真实硬件上的远程内核调试

QEMU 内置 GDB stub，但在真实硬件上需要使用 **KGDB**（内核内置的 GDB 代理）。

### 7.1 编译时开启 KGDB

```bash
make menuconfig
# 进入：Kernel hacking → Kernel debugging
#   [*] KGDB: kernel debugger
#   [*] KGDB: use kgdb over the serial console (kgdboc)
#   [*] KGDB: internal test suite
```

### 7.2 通过串口连接（两台机器）

```bash
# 被调试机（target）：在启动参数中加入
# /boot/grub/grub.cfg 或 /etc/default/grub：
GRUB_CMDLINE_LINUX="kgdboc=ttyS0,115200 kgdbwait"
# kgdbwait：启动时立即进入 KGDB 等待状态

# 触发断点（在运行中的内核）：
echo g > /proc/sysrq-trigger    # 通过 SysRq 进入 KGDB

# 调试机（host）：通过串口连接
gdb vmlinux
(gdb) set remotebaud 115200
(gdb) target remote /dev/ttyS0    # 串口设备
(gdb) target remote /dev/ttyUSB0  # USB 转串口

# 或通过网络（kgdb over ethernet，需 kgdboe 模块）
(gdb) target remote udp:192.168.1.100:6443
```

### 7.3 常用 KGDB 调试会话

```gdb
# 连接后查看所有 CPU 的调用栈
(gdb) thread apply all bt

# 切换到特定 CPU 上下文
(gdb) thread 2
(gdb) bt

# 打印内核链表（以进程链表为例）
(gdb) set $task = init_task
(gdb) set $task = (struct task_struct *)($task->tasks.next - \
                  (long)&((struct task_struct *)0)->tasks)
(gdb) printf "pid=%d comm=%s\n", $task->pid, $task->comm

# 动态断点（修改内核变量后继续）
(gdb) break tcp_v4_rcv
(gdb) condition 1 ((struct tcphdr*)skb->data)->dest == 0x5000  # port 80
(gdb) continue

# 查看 per-CPU 变量（偏移量方式）
(gdb) p/x __per_cpu_offset[0]
(gdb) p *(unsigned long*)(__per_cpu_offset[0] + (long)&nr_context_switches)
```

---

## 八、clangd：为内核源码配置智能 IDE 导航

`clangd` 是基于 Clang 的 LSP（语言服务器），为内核代码提供跳转、自动补全、引用查找。

### 8.1 生成 compile_commands.json

```bash
# 方法一：使用 bear（推荐）
sudo apt install bear
cd linux-5.15
bear -- make -j$(nproc) 2>&1 | tail -5
# 生成 compile_commands.json（约 1GB）

# 方法二：使用内核自带脚本（内核 5.x+）
make compile_commands.json
# 利用 scripts/clang-tools/gen_compile_commands.py

# 方法三：针对特定子系统（节省时间）
bear -- make drivers/net/ -j$(nproc)
```

### 8.2 VS Code 配置

```bash
# 安装 clangd 扩展（clangd language server）
# 在 VS Code 扩展市场搜索 "clangd"，安装 "clangd" by LLVM

# .vscode/settings.json
{
    "clangd.arguments": [
        "--background-index",        // 后台建立索引
        "--clang-tidy",              // 启用静态分析
        "--completion-style=detailed",
        "--header-insertion=never",
        "--query-driver=/usr/bin/gcc,/usr/bin/arm-linux-gnueabi-gcc"
    ],
    "clangd.path": "/usr/bin/clangd-14",
    "editor.semanticHighlighting.enabled": true
}
```

### 8.3 vim/neovim 配置（nvim-lspconfig）

```lua
-- ~/.config/nvim/init.lua
require('lspconfig').clangd.setup({
    cmd = {
        'clangd',
        '--background-index',
        '--clang-tidy',
        '--query-driver=/usr/bin/gcc*,/usr/bin/arm*',
    },
    root_dir = require('lspconfig.util').root_pattern(
        'compile_commands.json', 'Makefile'
    ),
})
-- 快捷键：gd(跳转定义) gr(查找引用) K(悬停文档)
```

### 8.4 常用导航技巧（内核特有）

```bash
# container_of 宏跳转：clangd 能正确解析
# 在 task_struct *t 处，跳转到 mm_struct 定义：直接 gd

# 查找所有调用 schedule() 的地方
# VS Code: 右键 → Find All References

# 解析 SYSCALL_DEFINE 宏展开
# clangd 能展开，直接跳转到 sys_read 的参数定义

# 注意：__attribute__ 和内联汇编可能导致部分误报，正常现象
```

---

## 九、内核编译优化技巧

### 9.1 make defconfig vs tinyconfig

```bash
# defconfig：生成针对当前架构的"合理默认"配置
# 适合：调试、学习、在 QEMU 中运行
make defconfig
# 编译时间：约 5~10 分钟（现代机器）
# 生成内核大小：约 8~12 MB (bzImage)

# tinyconfig：极度精简配置（几乎关闭所有功能）
# 适合：快速验证编译、CI 测试、最小内核实验
make tinyconfig
# 编译时间：约 30~60 秒
# 生成内核大小：约 500KB

# allmodconfig：尽可能多开模块（测试编译覆盖率用）
make allmodconfig

# 只重新编译修改的文件（增量编译）
make -j$(nproc)   # 第二次及后续编译，速度大幅提升

# 使用 ccache 加速重复编译
sudo apt install ccache
export CC="ccache gcc"
export HOSTCC="ccache gcc"
make -j$(nproc)
ccache -s   # 查看命中率
```

### 9.2 快速编译特定子系统

```bash
# 只编译 drivers/net/ 子系统
make drivers/net/ -j$(nproc)

# 只编译单个模块
make M=drivers/net/ethernet/intel/e1000/ -j$(nproc)

# 编译时只看警告/错误（过滤掉正常输出）
make 2>&1 | grep -E "^(.*error:|.*warning:)" | head -30

# 使用 LLVM/Clang 编译内核（5.x+ 正式支持）
make CC=clang LD=ld.lld -j$(nproc)
```

### 9.3 关键 Kconfig 选项（调试用）

```bash
# 在 .config 中启用或通过 menuconfig 设置：

# 必须开启（调试时）：
CONFIG_DEBUG_INFO=y            # 包含调试符号
CONFIG_FRAME_POINTER=y         # 保留帧指针（GDB 调用栈更准确）
CONFIG_KALLSYMS=y              # 内核符号表（oops 时显示函数名）
CONFIG_KALLSYMS_ALL=y          # 包含所有符号

# 推荐开启（检测问题）：
CONFIG_DEBUG_SLAB=y            # Slab 越界检测
CONFIG_KASAN=y                 # 内核地址消毒（类似 AddressSanitizer）
CONFIG_UBSAN=y                 # 未定义行为检测
CONFIG_LOCKDEP=y               # 死锁检测（性能影响大）
CONFIG_PROVE_LOCKING=y         # 锁依赖验证

# 性能分析：
CONFIG_FTRACE=y                # 函数追踪框架
CONFIG_PERF_EVENTS=y           # perf 支持
CONFIG_BPF_SYSCALL=y           # eBPF 系统调用
```

---

## 十、crash 工具：分析内核转储文件（vmcore）

`crash` 是分析 Linux 内核崩溃转储（`/proc/vmcore` 或离线 `vmcore` 文件）的权威工具。

### 10.1 安装与准备

```bash
# 安装 crash 工具
sudo apt install crash

# 安装 kdump（生成崩溃转储）
sudo apt install kdump-tools linux-crashdump
sudo systemctl enable kdump
# 在启动参数中预留内存：crashkernel=256M

# 触发测试崩溃（会重启！）
echo c > /proc/sysrq-trigger
# 重启后 vmcore 保存在 /var/crash/

# 也可以分析运行中的系统（实时调试）
sudo crash vmlinux /proc/kcore
```

### 10.2 分析 vmcore

```bash
# 打开崩溃转储
crash vmlinux /var/crash/2024-01-01/vmcore

# crash 常用命令
crash> bt              # 显示崩溃时的调用栈
crash> bt -a           # 所有 CPU 的调用栈
crash> ps             # 所有进程列表（类似 ps aux）
crash> ps | grep D    # 找所有 D 状态（uninterruptible）进程

# 查看内存分配
crash> kmem -i        # 内存使用概览
crash> kmem -s        # slab 缓存统计
crash> kmem -S task_struct  # 特定类型的 slab

# 查看进程详情
crash> task 1234      # 显示 PID 1234 的 task_struct
crash> files 1234     # PID 1234 的打开文件
crash> vm 1234        # PID 1234 的虚拟内存映射

# 查看内核日志（崩溃前的 dmesg）
crash> log            # 完整内核日志
crash> log | tail -50 # 最后 50 行

# 查看寄存器状态
crash> sys            # 系统信息
crash> mach           # 机器信息（CPU 数量、内存大小等）

# 定位崩溃地址
crash> dis -l ffffffff81234567    # 反汇编 + 源码行号
crash> gdb x/10i 0xffffffff81234567
```

### 10.3 典型崩溃分析流程

```
1. crash> bt        → 看崩溃时调用栈，找崩溃函数
2. crash> log       → 看崩溃前的内核消息（Oops 信息）
3. crash> dis -l <addr>  → 定位到具体源码行
4. crash> struct task_struct <addr>  → 查看崩溃时的数据结构
5. crash> kmem -s   → 检查是否有内存损坏（slab 错误）

# 示例：分析 NULL 指针解引用
crash> bt
 #0 [ffffffff81a00000] machine_kexec+0x...
 #1 [ffffffff81234abc] do_exit+0x...
 #2 [ffffffff81234def] my_driver_function+0x28  ← 崩溃点

crash> dis -l ffffffff81234def
0xffffffff81234def <my_driver_function+40>: mov 0x10(%rax),%rbx
# %rax 为 NULL → 解引用 NULL+0x10 → 崩溃
```
