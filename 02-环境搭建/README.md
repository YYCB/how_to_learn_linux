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
