# 15 — 内核调试与性能分析

> **学习目标**：掌握 Linux 内核调试与性能分析的完整工具链，从 printk 到 eBPF，
> 从静态分析到运行时 crash dump，能够定位内核 bug、量化性能瓶颈并实施优化。

---

## 目录

| 节 | 主题 |
|----|------|
| 15.1 | 调试工具全景 |
| 15.2 | printk 深入 |
| 15.3 | ftrace 基础 |
| 15.4 | ftrace 高级 |
| 15.5 | perf stat |
| 15.6 | perf record + report |
| 15.7 | FlameGraph 火焰图 |
| 15.8 | KASAN |
| 15.9 | KFENCE |
| 15.10 | KMSAN |
| 15.11 | UBSAN |
| 15.12 | lockdep |
| 15.13 | KCOV |
| 15.14 | kdump + crash |
| 15.15 | GDB + QEMU |
| 15.16 | bpftrace 调试技巧 |
| 15.17 | livepatch |
| 15.18 | 性能优化清单 |

---

## 15.1 调试工具全景

### 工具对比矩阵

| 工具 | 适用场景 | 运行时开销 | 内核配置 | 生产可用 |
|------|----------|----------|---------|---------|
| printk/dyndbg | 快速日志调试 | 低~中 | 内置 | ✅ |
| ftrace | 函数/事件追踪 | 低~中 | `CONFIG_FTRACE` | ✅ |
| perf | CPU/内存/IO 分析 | 低 | `CONFIG_PERF_EVENTS` | ✅ |
| eBPF/bpftrace | 动态安全探针 | 极低 | `CONFIG_BPF` | ✅ |
| KASAN | 内存安全 bug | **高(2x内存,慢2x)** | `CONFIG_KASAN` | ❌开发用 |
| KFENCE | UAF/OOB 检测 | **极低** | `CONFIG_KFENCE` | ✅ |
| KMSAN | 未初始化内存 | **极高** | `CONFIG_KMSAN` | ❌开发用 |
| UBSAN | 未定义行为 | 低~中 | `CONFIG_UBSAN` | ⚠️ |
| lockdep | 死锁检测 | 中 | `CONFIG_LOCKDEP` | ❌开发用 |
| KCOV | 代码覆盖率 | 中 | `CONFIG_KCOV` | ❌测试用 |
| kdump/crash | 崩溃分析 | 无（事后） | `CONFIG_KEXEC` | ✅ |
| GDB+QEMU | 源码级调试 | 极高（虚拟机）| vmlinux | ❌开发用 |
| livepatch | 热补丁 | 极低 | `CONFIG_LIVEPATCH` | ✅ |

### 工具选择决策树

```
遇到问题
    ├── 系统崩溃/Panic？
    │   └── kdump + crash → 分析 vmcore
    ├── 内存损坏/bug？
    │   ├── 开发环境 → KASAN（全面检测）
    │   └── 生产环境 → KFENCE（低开销）
    ├── 死锁/锁顺序问题？
    │   └── lockdep（开发内核）
    ├── 性能问题？
    │   ├── CPU 瓶颈 → perf stat + FlameGraph
    │   ├── 延迟问题 → ftrace + bpftrace
    │   └── IO 问题  → blktrace + bpftrace
    └── 行为追踪/理解代码？
        ├── 静态 → ftrace function tracer
        └── 动态 → bpftrace / eBPF
```

---

## 15.2 printk 深入

### 日志级别

```c
/* include/linux/kern_levels.h */
#define KERN_EMERG   "0"  /* 系统不可用，立即崩溃 */
#define KERN_ALERT   "1"  /* 必须立即处理 */
#define KERN_CRIT    "2"  /* 严重条件 */
#define KERN_ERR     "3"  /* 错误条件 */
#define KERN_WARNING "4"  /* 警告条件 */
#define KERN_NOTICE  "5"  /* 正常但值得注意 */
#define KERN_INFO    "6"  /* 信息性消息 */
#define KERN_DEBUG   "7"  /* 调试级别消息 */
#define KERN_DEFAULT "d"  /* 默认内核日志级别 */

/* 使用方式 */
pr_emerg("Out of memory: Kill process %d (%s)\n", pid, comm);
pr_err("Failed to allocate %zu bytes\n", size);
pr_warn("Deprecated feature used by %s\n", current->comm);
pr_info("Device %s registered\n", dev_name);
pr_debug("value = %d\n", val);  /* 仅在 DEBUG 宏定义时编译 */

/* 带设备前缀 */
dev_err(dev, "I2C transfer failed: %d\n", ret);
dev_info(dev, "Probed successfully\n");

/* 速率限制（避免日志洪水）*/
pr_err_ratelimited("DMA error %d\n", err);
printk_ratelimited(KERN_ERR "error: %d\n", err);
```

### dmesg 使用技巧

```bash
# 带时间戳查看
dmesg -T          # 人类可读时间
dmesg -t          # 无时间戳
dmesg --follow    # 实时跟踪

# 按级别过滤
dmesg -l err,crit,emerg     # 只显示错误
dmesg -l debug              # 只显示调试信息
dmesg --facility=kern       # 只显示内核消息

# 清空日志缓冲
dmesg -c

# 设置日志级别（内核输出到控制台的最低级别）
echo 7 > /proc/sys/kernel/printk   # 显示所有级别
# /proc/sys/kernel/printk 包含 4 个值：
# console_loglevel default_message_loglevel min_console_loglevel default_console_loglevel
cat /proc/sys/kernel/printk
# 7 4 1 7
```

### 动态调试 (dyndbg)

```bash
# CONFIG_DYNAMIC_DEBUG=y 时可运行时开关 pr_debug/dev_dbg

# 控制文件
ls /sys/kernel/debug/dynamic_debug/control

# 语法：
# 条件 格式标志 动作
# 动作：+/-  p=打印 f=函数名 l=行号 m=模块名 t=线程ID

# 启用特定文件的调试消息
echo "file net/ipv4/tcp.c +p" > /sys/kernel/debug/dynamic_debug/control

# 启用特定模块
echo "module e1000e +p" > /sys/kernel/debug/dynamic_debug/control

# 启用特定函数
echo "func tcp_sendmsg +p" > /sys/kernel/debug/dynamic_debug/control

# 启用某函数并显示行号
echo "func kmalloc +flp" > /sys/kernel/debug/dynamic_debug/control

# 内核命令行启用（早期调试）
dyndbg="file init/main.c +p"
dyndbg="module thermal +p"

# 查看当前启用的调试消息
cat /sys/kernel/debug/dynamic_debug/control | grep "=p"

# 禁用
echo "module e1000e -p" > /sys/kernel/debug/dynamic_debug/control
```

---

## 15.3 ftrace 基础

### tracefs 挂载与基本操作

```bash
# 挂载 tracefs（通常已挂载在 /sys/kernel/tracing）
mount -t tracefs tracefs /sys/kernel/tracing
# 或
mount -t debugfs debugfs /sys/kernel/debug
# tracefs 在 /sys/kernel/debug/tracing/

cd /sys/kernel/tracing   # 以下命令在此目录执行

# 查看可用 tracer
cat available_tracers
# blk function_graph wakeup_dl wakeup_rt wakeup function nop

# 查看当前 tracer
cat current_tracer

# 设置 function tracer
echo function > current_tracer

# 开始追踪
echo 1 > tracing_on

# 停止追踪
echo 0 > tracing_on

# 读取结果
cat trace | head -30
# 格式：进程名-PID  [CPU] 标志  时间戳:  函数名 <- 调用者
# bash-1234  [001] ....  1234.567890: kmalloc <- __kmalloc_node

# 清空 trace buffer
echo > trace
```

### function_graph tracer

```bash
# 显示函数调用图（入口+出口+执行时间）
echo function_graph > current_tracer

# 设置追踪深度
echo 5 > max_graph_depth

# 只追踪特定函数及其子调用
echo do_sys_open > set_graph_function
echo 1 > tracing_on
cat trace | head -50

# 示例输出：
# CPU DURATION      FUNCTION CALLS
# |   |   |         |   |   |   |
# 1)               | do_sys_open() {
# 1)               |   getname() {
# 1)   2.341 us    |     getname_flags();
# 1) + 5.234 us    |   } /* getname */
# 1)               |   alloc_fd() {
# 1)   0.891 us    |     __alloc_fd();
# 1)   1.234 us    |   } /* alloc_fd */
# 1) + 45.678 us   | } /* do_sys_open */
```

### set_ftrace_filter 过滤

```bash
# 只追踪特定函数
echo "kmalloc" > set_ftrace_filter
echo "kfree" >> set_ftrace_filter

# 使用通配符
echo "tcp_*" > set_ftrace_filter
echo "ext4_*" >> set_ftrace_filter

# 追踪模块函数
echo ':mod:e1000e' > set_ftrace_filter

# 排除某些函数（notrace filter）
echo "native_sched_clock" > set_ftrace_notrace

# 只追踪特定 PID
echo 1234 > set_ftrace_pid

# 查看可追踪的函数列表
cat available_filter_functions | wc -l   # 通常数万个

# trace-cmd 封装工具（更方便）
trace-cmd record -p function -l "tcp_*" sleep 5
trace-cmd report | head -50
trace-cmd hist
```

---

## 15.4 ftrace 高级

### 事件追踪 (trace_events)

```bash
# 查看所有可用事件
ls /sys/kernel/tracing/events/
# block  ext4  kmem  net  sched  signal  skb  sock  ...

# 查看某类别的事件
ls /sys/kernel/tracing/events/sched/
# sched_switch  sched_wakeup  sched_process_fork  ...

# 启用单个事件
echo 1 > /sys/kernel/tracing/events/sched/sched_switch/enable

# 启用整个类别
echo 1 > /sys/kernel/tracing/events/net/enable

# 查看事件格式
cat /sys/kernel/tracing/events/sched/sched_switch/format
# name: sched_switch
# field:unsigned short common_type;
# field:pid_t prev_pid;
# field:char prev_comm[16];
# field:int prev_prio;
# field:long prev_state;
# field:pid_t next_pid;
# field:char next_comm[16];

# 设置事件过滤器
echo "prev_comm == 'nginx'" > \
    /sys/kernel/tracing/events/sched/sched_switch/filter

# trace-cmd 方式（推荐）
trace-cmd record -e sched:sched_switch -e net:netif_rx \
    -f "comm == 'nginx'" sleep 10
trace-cmd report
```

### hist 触发器（内核直方图）

```bash
# 记录系统调用延迟直方图
echo 'hist:key=id.syscall:val=elapsed:sort=elapsed' > \
    /sys/kernel/tracing/events/raw_syscalls/sys_exit/trigger

sleep 10

cat /sys/kernel/tracing/events/raw_syscalls/sys_exit/hist
# 输出按延迟排序的系统调用直方图

# 记录调度延迟
echo 'hist:key=comm:val=hitcount:sort=hitcount' > \
    /sys/kernel/tracing/events/sched/sched_switch/trigger

# 更复杂：跟踪 sched_wakeup 到 sched_switch 的延迟
echo 'hist:key=pid:ts0=common_timestamp.usecs' > \
    /sys/kernel/tracing/events/sched/sched_wakeup/trigger
echo 'hist:key=next_pid:wakeup_lat=common_timestamp.usecs-$ts0:onmatch(sched.sched_wakeup).trace(sched_wakeup_latency,$wakeup_lat)' > \
    /sys/kernel/tracing/events/sched/sched_switch/trigger
```

### 合成事件与延迟测量

```bash
# 创建合成事件：测量块 IO 延迟
# 1. 定义合成事件
echo 'block_io_lat u64 sector; u64 lat_us' > \
    /sys/kernel/tracing/synthetic_events

# 2. 在 block_rq_issue 记录时间戳
echo 'hist:key=sector:ts0=common_timestamp.usecs' > \
    /sys/kernel/tracing/events/block/block_rq_issue/trigger

# 3. 在 block_rq_complete 计算延迟并触发合成事件
echo 'hist:key=sector:lat=common_timestamp.usecs-$ts0:
onmatch(block.block_rq_issue).block_io_lat(sector,$lat)' > \
    /sys/kernel/tracing/events/block/block_rq_complete/trigger

# 4. 监听合成事件
echo 1 > /sys/kernel/tracing/events/synthetic/block_io_lat/enable
cat /sys/kernel/tracing/trace
```

---

## 15.5 perf stat

### CPU 性能事件

```bash
# 基本统计：运行 5 秒全系统
perf stat -a sleep 5

# 输出解读：
# Performance counter stats for 'system wide':
#    10,234,567,890  cycles              #    3.214 GHz
#     8,901,234,567  instructions        #    0.87  insn per cycle  ← IPC
#        23,456,789  cache-misses        #    2.345 % of all cache refs
#         1,234,567  branch-misses       #    0.123 % of all branches
#               500  context-switches    #   50.000 /sec
#                12  cpu-migrations
#            45,678  page-faults

# IPC < 1：通常是内存/缓存延迟瓶颈
# IPC > 3：CPU 计算密集，运行良好
# cache-miss % > 5%：需要优化内存访问模式

# 追踪特定进程
perf stat -p $(pgrep nginx) sleep 10

# 追踪特定 CPU
perf stat -C 0,1,2,3 sleep 5

# 自定义事件
perf stat -e \
  cycles,\
  instructions,\
  cache-references,\
  cache-misses,\
  branch-instructions,\
  branch-misses,\
  L1-dcache-loads,\
  L1-dcache-load-misses,\
  LLC-loads,\
  LLC-load-misses \
  -- /usr/bin/stress --cpu 4 --timeout 10

# 性能公式
# IPC = instructions / cycles
# CPI = cycles / instructions （越低越好）
# cache miss rate = cache-misses / cache-references × 100%
# branch miss rate = branch-misses / branch-instructions × 100%
```

### perf list — 可用事件

```bash
# 列出所有硬件事件
perf list hw

# 软件事件
perf list sw

# 追踪点（tracepoints）
perf list tracepoint | wc -l   # 通常 1000+ 个

# PMU（处理器特定）事件
perf list pmu | head -30

# 按类别搜索
perf list | grep -i "cache"
perf list | grep -i "tlb"
perf list | grep -i "memory"

# Intel Top-Down 分析方法（需要 Intel CPU + perf topdown）
perf stat --topdown -a sleep 5
# Retiring: 40%  → 有效执行
# Bad Speculation: 15%  → 分支预测失误
# Frontend Bound: 25%  → 指令获取瓶颈
# Backend Bound: 20%  → 执行资源/内存瓶颈
```

---

## 15.6 perf record + report

### 采样原理

```
perf record 采样原理：
  1. 设置 PMU 溢出中断（每 N 次事件触发一次中断）
  2. 中断处理程序记录 RIP（当前指令地址）+ 调用栈
  3. 写入 perf.data 文件（mmap 环形缓冲区）
  4. perf report 对地址进行符号化

默认采样频率：-F 4000（4000 Hz 采样/秒）
默认事件：cycles（CPU 周期）

权衡：
  高频率 → 更精确 → 更高开销（>10000 Hz 慎用）
  低频率 → 低开销 → 统计误差更大
```

### 基本采样

```bash
# 全系统采样 30 秒
perf record -a -g -F 999 sleep 30
# -a: 所有 CPU
# -g: 记录调用图（call graph）
# -F 999: 每秒采样 999 次

# 采样特定进程
perf record -g -p $(pgrep -f "java.*MyApp") sleep 30

# 采样特定事件（LLC miss）
perf record -e LLC-load-misses -a -g sleep 10

# 运行命令并采样
perf record -g -- /path/to/my_program arg1 arg2

# 查看 perf.data 文件信息
perf report --header

# 基本报告
perf report --stdio | head -50
```

### 调用图采集方法对比

```bash
# 方法1：frame pointer（快速，但需要 -fno-omit-frame-pointer 编译）
perf record -g --call-graph=fp -a sleep 30

# 方法2：DWARF 调试信息（准确，开销较高）
perf record -g --call-graph=dwarf -a sleep 30

# 方法3：LBR（Last Branch Record，Intel 专有，最快）
perf record -g --call-graph=lbr -a sleep 30

# 实际推荐：
# 内核：fp（内核用 -fno-omit-frame-pointer 编译）
# 用户空间 C/C++：dwarf 或重新编译加 -fno-omit-frame-pointer
# Java/Python：需要额外 perf map agent

# 查看注释（按函数内的指令热点）
perf annotate --stdio kmalloc | head -40
```

---

## 15.7 FlameGraph 火焰图

### 生成步骤

```bash
# 1. 克隆 FlameGraph 工具
git clone https://github.com/brendangregg/FlameGraph.git
cd FlameGraph

# 2. 采样（使用 frame pointer 调用图）
perf record -F 99 -a -g -- sleep 60
# 或针对特定进程
perf record -F 99 -g -p $(pgrep nginx) sleep 30

# 3. 转换格式
perf script > out.perf

# 4. 折叠调用栈
./stackcollapse-perf.pl out.perf > out.folded

# 5. 生成 SVG
./flamegraph.pl out.folded > flamegraph.svg

# 一键命令
perf record -F 99 -a -g -- sleep 60 && \
    perf script | \
    ./stackcollapse-perf.pl | \
    ./flamegraph.pl > flamegraph.svg

# 打开查看
firefox flamegraph.svg
# 或
python3 -m http.server 8080  # 然后浏览器访问
```

### 读懂火焰图

```
火焰图解读规则：
  Y 轴（上下）= 调用栈深度（底部=被采样点，顶部=最深调用）
  X 轴（左右）= 时间宽度（宽=消耗更多 CPU 时间）
  颜色        = 随机（区分函数），无特殊含义

  ┌─────────────────────────────────────────────────┐
  │      handle_mm_fault      copy_page_range        │ ← 叶节点（最热）
  │        do_page_fault       do_mprotect           │
  │          page_fault       sys_mprotect           │
  │             entry_SYSCALL_64                     │
  │    nginx_worker_cycle    kernel_vsyscall         │
  └─────────────────────────────────────────────────┘
                      时间 →

  最宽的栈顶函数 = 最消耗 CPU 的热点
  宽大的平顶     = 可能的性能瓶颈
  窄但深的塔     = 递归或深调用链
```

### 差分火焰图（对比优化前后）

```bash
# 优化前采样
perf record -F 99 -a -g -- sleep 60
perf script > before.perf
./stackcollapse-perf.pl before.perf > before.folded

# 实施优化...

# 优化后采样
perf record -F 99 -a -g -- sleep 60
perf script > after.perf
./stackcollapse-perf.pl after.perf > after.folded

# 生成差分图（红=变慢/增多，蓝=变快/减少）
./difffolded.pl before.folded after.folded | \
    ./flamegraph.pl --colors=blue > diff.svg
```

---

## 15.8 KASAN (Kernel Address Sanitizer)

### 配置与原理

```bash
# 内核配置（GENERIC 与 HW_TAGS 互斥，二选一）
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y        # 软件实现（所有架构）
# 或（二选一，不可同时启用）
CONFIG_KASAN_HW_TAGS=y        # 硬件实现（ARM MTE，低开销，需要 ARMv8.5+）

# 开销：
# - 内存：每8字节对应1字节 shadow（内存×2）
# - CPU：约 1.5-2x 慢（每次访问检查 shadow）
# - 可检测：use-after-free, out-of-bounds, use-after-scope
```

### KASAN 报告解读

```
==================================================================
BUG: KASAN: slab-out-of-bounds in copy_from_user+0x.../...
Write of size 8 at addr ffff888012345678 by task kworker/0:1/234
                 ↑类型    ↑被访问地址          ↑任务名

CPU: 0 PID: 234 Comm: kworker/0:1
Hardware name: QEMU Standard PC

Call Trace:                          ← 谁触发的
 dump_stack+0x...
 kasan_report+0x...
 copy_from_user+0x...
 my_driver_write+0x3c/0x80          ← 问题代码位置
 vfs_write+0x...

Allocated by task 234:              ← 内存在哪里被分配
 kmalloc+0x...
 my_driver_probe+0x58/0x100

Freed by task 234:                  ← 内存在哪里被释放（UAF时）
 kfree+0x...
 my_driver_remove+0x...

The buggy address belongs to the object at ffff888012345600
 which belongs to the cache kmalloc-128 of size 128
The buggy address is located 120 bytes inside of
 128-byte region [ffff888012345600, ffff888012345680)
 ↑ 地址在 128 字节对象的第 120 字节处（越界 8 字节）
==================================================================
```

### 使用 KASAN 调试

```bash
# 在 KASAN 内核上运行目标程序
# 建议使用 syzkaller 或手动触发 bug 路径

# 查看 KASAN 统计
cat /sys/kernel/debug/kasan/stats 2>/dev/null

# KASAN 选项（内核命令行）
kasan=off            # 禁用（通常不需要）
kasan_multi_shot     # 每个 bug 报告多次（默认每次只报告一次）

# 编译内核时的 KASAN 选项
CONFIG_KASAN_INLINE=y   # 内联检查（更快）
CONFIG_KASAN_OUTLINE=y  # 函数调用检查（更小代码）
```

---

## 15.9 KFENCE (Kernel Electric Fence)

### 原理与配置

```c
/* KFENCE：低开销的内存安全检测，适合生产环境 */

/* 工作原理：
 * - 以固定概率（默认每 100ms 一次）将 slab 分配重定向到 KFENCE 保护池
 * - KFENCE 池中每个对象使用独立内存页，前后各有 Guard Page
 * - Guard Page 无访问权限，访问时触发 page fault
 * - 对象释放后，页面标记为不可访问（检测 UAF）
 */

/* 内核配置 */
// CONFIG_KFENCE=y
// CONFIG_KFENCE_SAMPLE_INTERVAL=100  (ms)
// CONFIG_KFENCE_NUM_OBJECTS=255

/* 内核命令行 */
// kfence.sample_interval=100   — 采样间隔（ms，0=禁用）
```

```bash
# 查看 KFENCE 统计
cat /sys/kernel/debug/kfence/stats
# total allocs:    12345
# total frees:     12340
# total bugs:          3

# 查看详细报告
dmesg | grep KFENCE

# 调整采样频率（越低=越多覆盖，越高开销）
echo 50 > /sys/module/kfence/parameters/sample_interval

# KFENCE vs KASAN 对比
# KFENCE：概率采样，开销极低，生产可用，覆盖率随时间积累
# KASAN：全量检测，开销高，开发专用，立即发现所有访问
```

---

## 15.10 KMSAN (Kernel Memory Sanitizer)

```c
/* KMSAN：检测内核中使用未初始化内存 */

/* 常见 bug：
 * - 栈变量未初始化就拷贝到用户空间（信息泄漏！）
 * - 联合体部分字段未初始化
 * - kmalloc 后未 memset 就读取
 */

/* 内核配置 */
// CONFIG_KMSAN=y
// 依赖：CONFIG_CC_IS_CLANG=y（需要 Clang 编译）

/* 示例 bug */
struct my_struct {
    int important;
    int padding;     /* 未初始化 */
};

void buggy_function(void) {
    struct my_struct s;
    s.important = 42;
    /* BUG: s.padding 未初始化 */
    if (copy_to_user(user_ptr, &s, sizeof(s)))
        return -EFAULT;
    /* KMSAN 会在这里报告：uninitialized memory copy to user */
}

/* 修复 */
struct my_struct s = {};  /* 零初始化 */
/* 或 */
memset(&s, 0, sizeof(s));
```

```bash
# 查看 KMSAN 报告
dmesg | grep "KMSAN"

# KMSAN 报告格式
# BUG: KMSAN: kernel-infoleak in copy_to_user+0x...
# Uninit was created at:
#   kmalloc+0x...
#   my_driver_alloc+0x...
```

---

## 15.11 UBSAN (Undefined Behavior Sanitizer)

```bash
# 内核配置
# CONFIG_UBSAN=y
# CONFIG_UBSAN_SANITIZE_ALL=y  — 检查所有代码
# CONFIG_UBSAN_TRAP=y          — 遇到 UB 时 trap（更严格）

# 检测的 UB 类型
# - 有符号整数溢出（signed overflow）
# - 移位越界（shift out of bounds）
# - 数组越界（array index out of bounds）
# - 空指针解引用（null pointer dereference）
# - 对齐违规（misaligned access）
# - 无效 bool 值

# UBSAN 报告示例
# UBSAN: signed-integer-overflow in kernel/time.c:123
# -2147483648 - 1 cannot be represented in type 'int'
# ...
# Call Trace:
#   ubsan_epilogue
#   handle_overflow
#   __ubsan_handle_sub_overflow

# 在代码中禁止特定 UBSAN 检查
__attribute__((no_sanitize("signed-integer-overflow")))
static int my_safe_function(int a, int b) {
    return a + b;  /* 此处溢出是预期行为 */
}
```

---

## 15.12 lockdep (死锁检测)

### 工作原理

```
lockdep 死锁检测原理：

1. 锁类（Lock Class）
   - 每个锁变量实例属于一个"锁类"（同一代码位置分配的锁）
   - 不追踪具体锁实例，追踪锁类之间的顺序关系

2. 锁顺序图（Lock Order Graph）
   - 记录：持有 A 时获取 B → 边 A→B
   - 检测：是否存在环（A→B→C→A = 死锁可能）

3. 检测时机
   - 每次 lock() 操作时即时检查
   - 不需要实际发生死锁，只要顺序可能导致死锁就报告
```

### lockdep 报告解读

```
=====================================================
WARNING: possible circular locking dependency detected
6.1.0 #1 SMP
-----------------------------------------------------
kworker/0:1/234 is trying to acquire lock:
ffffffff81234560 (&mm->mmap_lock){++++}, ...

but task is already holding lock:
ffffffff81567890 (&fs->lock){+.+.}, ...

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:

-> #1 (&fs->lock){+.+.}:       ← 锁 A
       lock_acquire
       __mutex_lock
       copy_fs_struct

-> #0 (&mm->mmap_lock){++++}:  ← 锁 B
       lock_acquire
       down_read
       dup_mm

other info that might help us debug this:
 Possible unsafe locking scenario:

       CPU0                    CPU1
       ----                    ----
  lock(&mm->mmap_lock);    lock(&fs->lock);
                             lock(&mm->mmap_lock);  ← 等待 CPU0
  lock(&fs->lock);  ← 等待 CPU1

 *** DEADLOCK ***
=====================================================
```

### lockdep 配置与工具

```bash
# 内核配置
# CONFIG_LOCKDEP=y
# CONFIG_PROVE_LOCKING=y
# CONFIG_DEBUG_LOCKDEP=y
# CONFIG_LOCK_STAT=y

# 查看锁统计
cat /proc/lock_stat | head -30
# class name    con-bounces    contentions  waittime-min  waittime-max  ...

# 重置统计
echo 0 > /proc/lock_stat

# 查看死锁报告
dmesg | grep -A50 "circular locking"

# 在代码中标注锁顺序（消除误报）
mutex_lock_nested(&child->lock, SINGLE_DEPTH_NESTING);

# 声明锁类（相同代码但不同实例）
static struct lock_class_key my_lock_key;
lockdep_set_class(&spinlock, &my_lock_key);
```

---

## 15.13 KCOV (内核代码覆盖率)

```c
/* KCOV：为 syzkaller 等 fuzzer 提供覆盖率反馈 */

/* 内核配置 */
// CONFIG_KCOV=y
// CONFIG_KCOV_ENABLE_COMPARISONS=y  — 比较值覆盖

/* 用户空间使用 KCOV */
#include <linux/kcov.h>

int fd = open("/sys/kernel/debug/kcov", O_RDWR);
ioctl(fd, KCOV_INIT_TRACE, COVER_SIZE);
uint64_t *cover = mmap(NULL, COVER_SIZE * sizeof(uint64_t),
                       PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

/* 开始收集覆盖率 */
ioctl(fd, KCOV_ENABLE, KCOV_TRACE_PC);
__atomic_store_n(&cover[0], 0, __ATOMIC_RELAXED);

/* 执行系统调用 */
read(some_fd, buf, size);

/* 停止收集 */
ioctl(fd, KCOV_DISABLE, 0);

uint64_t n = __atomic_load_n(&cover[0], __ATOMIC_RELAXED);
for (uint64_t i = 0; i < n; i++) {
    printf("0x%llx\n", cover[i + 1]);  /* 执行到的内核地址 */
}
```

```bash
# syzkaller 使用 KCOV 的基本配置
cat syz-manager.cfg
# {
#     "target": "linux/amd64",
#     "http": "0.0.0.0:56741",
#     "workdir": "/syzkaller/workdir",
#     "kernel_obj": "/linux",
#     "image": "/linux/stretch.img",
#     "sshkey": "/linux/stretch.id_rsa",
#     "syzkaller": "/syzkaller",
#     "procs": 8,
#     "type": "qemu",
#     "vm": {"count": 4, "kernel": "/linux/arch/x86/boot/bzImage"}
# }
```

---

## 15.14 kdump + crash

### 配置 kdump

```bash
# 1. 安装工具
apt install kdump-tools crash linux-crashdump
# 或
yum install kexec-tools crash kernel-debuginfo

# 2. 内核参数（/etc/default/grub）
GRUB_CMDLINE_LINUX="crashkernel=256M"
# 大内存系统建议：crashkernel=512M,high
# 自动计算：crashkernel=auto

update-grub && reboot

# 3. 验证配置
cat /proc/iomem | grep "Crash kernel"
# 1:00000000-3fffffff : Crash kernel  ← 预留的内存范围

# 4. 配置 kdump 目标
cat /etc/kdump.conf
# path /var/crash
# core_collector makedumpfile -l --message-level 1 -d 31
# default reboot
# 注：-d 31 = 丢弃不需要的页（zero/cache）以减小 dump 大小

# 5. 启动 kdump 服务
systemctl enable --now kdump
systemctl status kdump

# 6. 测试（!!! 会崩溃系统 !!!）
echo c > /proc/sysrq-trigger
```

### makedumpfile 过滤级别

```bash
# makedumpfile 过滤级别（-d 参数）
# 1  = 去除 zero 页
# 2  = 去除 cache 页（未使用）
# 4  = 去除 cache 页（私有）
# 8  = 去除用户数据页
# 16 = 去除 free 页
# 31 = 去除以上所有（只保留内核内存，大幅减小文件大小）

makedumpfile -l --message-level 31 -d 31 \
    /proc/vmcore /var/crash/vmcore.$(date +%s)
```

### crash 命令参考

```bash
# 启动 crash
crash /usr/lib/debug/boot/vmlinux-6.1.0-22 \
      /var/crash/202401010000/vmcore

# crash 内基本命令
crash> help        # 命令列表

# 进程和线程
crash> ps          # 所有进程
crash> ps -k       # 内核线程
crash> task 1234   # 查看指定 PID 的 task_struct
crash> thread_info 1234  # 线程信息

# 调用栈
crash> bt          # 当前（崩溃）进程的调用栈
crash> bt -a       # 所有 CPU 的调用栈
crash> bt -t 1234  # 指定 PID 的调用栈
crash> bt -l       # 显示行号
crash> bt -f       # 显示完整帧信息

# 内存
crash> vm          # 当前进程虚拟内存
crash> vm 1234     # 指定 PID 的 VMA
crash> kmem -i     # 内存使用概况
crash> kmem -s     # slab 分配器统计
crash> kmem -S kmalloc-128  # 特定 slab 信息

# 日志
crash> log         # 内核消息缓冲区（dmesg）
crash> log -m      # 带时间戳

# 文件
crash> files 1234  # 进程打开的文件
crash> net         # 网络统计
crash> net -s      # socket 信息

# 反汇编
crash> dis -l tcp_sendmsg    # 带行号反汇编
crash> dis -l 0xffffffff81234567  # 按地址

# 符号查找
crash> sym schedule       # 查找符号地址
crash> sym ffffffff81234567  # 地址转符号

# 查看数据结构
crash> struct task_struct 0xffff888012345600
crash> p init_task        # 打印变量值
crash> rd -64 0xffffffff81234567 20  # 读取内存（20个64位字）
```

---

## 15.15 GDB + QEMU 内核调试

### 环境搭建

```bash
# 编译调试内核（.config 选项）
# CONFIG_DEBUG_INFO=y
# CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT=y
# CONFIG_GDB_SCRIPTS=y
# CONFIG_FRAME_POINTER=y
# CONFIG_RANDOMIZE_BASE=n  (禁用 KASLR)
# CONFIG_KASAN=n           (可选，避免干扰)

# 启动 QEMU（暴露 GDB 服务器端口）
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -drive file=rootfs.img,format=raw \
    -append "root=/dev/sda console=ttyS0 nokaslr debug" \
    -serial stdio \
    -m 2G \
    -smp 4 \
    -nographic \
    -s \        # 监听 :1234 GDB 端口
    -S          # 暂停等待 GDB 连接（可选）
```

### GDB 连接与调试

```bash
# 启动 GDB
gdb vmlinux

# 连接 QEMU
(gdb) target remote localhost:1234

# 加载内核调试脚本（CONFIG_GDB_SCRIPTS=y 时可用）
(gdb) lx-symbols         # 加载所有模块符号
(gdb) lx-version         # 内核版本
(gdb) lx-ps              # 进程列表（类似 ps）
(gdb) lx-dmesg           # 内核消息缓冲区
(gdb) lx-lsmod           # 已加载模块

# 设置断点
(gdb) break sys_execve
(gdb) break net/ipv4/tcp.c:tcp_sendmsg
(gdb) hbreak do_page_fault  # 硬件断点（不修改代码）
(gdb) watch *(int*)0xffffffff81234567  # 内存监视点

# 继续执行
(gdb) continue
(gdb) c

# 单步
(gdb) step      # 进入函数
(gdb) next      # 跳过函数
(gdb) finish    # 执行到函数返回
(gdb) stepi     # 汇编单步

# 查看信息
(gdb) info registers             # 寄存器
(gdb) info breakpoints           # 断点列表
(gdb) backtrace                  # 调用栈
(gdb) frame 3                    # 切换到第 3 帧
(gdb) list                       # 显示源码
(gdb) print init_task.pid        # 打印变量
(gdb) print/x $rip               # 打印寄存器（十六进制）
(gdb) x/10i $rip                 # 查看当前指令

# lx-* 辅助命令
(gdb) lx-ps                      # 进程列表
(gdb) lx-task-by-pid 1234        # 按 PID 查找 task
(gdb) lx-thread-info 0xffff...   # 线程信息
(gdb) lx-per-cpu current_task 0  # CPU0 的当前任务
(gdb) lx-list-check init_task tasks  # 遍历进程链表

# 调试模块（加载后）
(gdb) add-symbol-file /path/to/module.ko 0xffffffffc0123000
```

---

## 15.16 bpftrace 调试技巧

### 追踪内存分配

```bash
# 安装
apt install bpftrace  # 或 dnf install bpftrace

# 1. 统计 kmalloc 调用大小分布
bpftrace -e '
kprobe:__kmalloc {
    @size_hist = hist(arg0);
}
interval:s:5 {
    print(@size_hist);
    clear(@size_hist);
}'

# 2. 找出 kmalloc 最多的调用方（前10）
bpftrace -e '
kprobe:__kmalloc {
    @[kstack()] = count();
}
END {
    print(@, 10);
}'

# 3. 追踪特定大小的分配（检测内存泄漏）
bpftrace -e '
kprobe:kmalloc {
    @allocs[arg0, retval] = count();
}
kprobe:kfree {
    delete(@allocs[0, arg0]);
}
interval:s:10 {
    print(@allocs);
}'

# 4. slab 分配器热点
bpftrace -e '
kprobe:kmem_cache_alloc {
    @cache[((struct kmem_cache *)arg0)->name] = count();
}
END { print(@cache, 20); }'
```

### 追踪调度延迟

```bash
# 1. 测量进程唤醒到运行的延迟
bpftrace -e '
tracepoint:sched:sched_wakeup,
tracepoint:sched:sched_wakeup_new {
    @wakeup_ts[args->pid] = nsecs;
}

tracepoint:sched:sched_switch {
    $ts = @wakeup_ts[args->next_pid];
    if ($ts) {
        $lat = (nsecs - $ts) / 1000;  /* 转换为微秒 */
        @latency_us = hist($lat);
        delete(@wakeup_ts[args->next_pid]);
    }
}

END { print(@latency_us); }'

# 2. 找出导致高调度延迟的进程
bpftrace -e '
tracepoint:sched:sched_wakeup {
    @wake[args->comm] = nsecs;
}
tracepoint:sched:sched_switch {
    $ts = @wake[args->next_comm];
    if ($ts && (nsecs - $ts) > 1000000) {  /* > 1ms */
        printf("HIGH LAT: %s lat=%dms on CPU%d\n",
               args->next_comm, (nsecs-$ts)/1000000, cpu);
    }
    delete(@wake[args->next_comm]);
}'
```

### 追踪文件 IO 延迟

```bash
# 1. 块设备 IO 延迟分布
bpftrace -e '
tracepoint:block:block_rq_issue {
    @start[args->dev, args->sector] = nsecs;
}
tracepoint:block:block_rq_complete {
    $key = (args->dev, args->sector);
    $ts = @start[$key];
    if ($ts) {
        $lat = (nsecs - $ts) / 1000;
        @io_lat_us = hist($lat);
        @io_lat_by_dev[args->dev] = hist($lat);
        delete(@start[$key]);
    }
}
END {
    print(@io_lat_us);
    print(@io_lat_by_dev);
}'

# 2. 追踪慢 read 系统调用（> 10ms）
bpftrace -e '
tracepoint:syscalls:sys_enter_read {
    @ts[tid] = nsecs;
    @fd[tid] = args->fd;
}
tracepoint:syscalls:sys_exit_read {
    $ts = @ts[tid];
    if ($ts && (nsecs - $ts) > 10000000) {
        printf("SLOW READ: pid=%d comm=%s fd=%d lat=%dms count=%d\n",
               pid, comm, @fd[tid],
               (nsecs - $ts) / 1000000,
               args->ret);
    }
    delete(@ts[tid]);
    delete(@fd[tid]);
}'

# 3. ext4 层延迟
bpftrace -e '
kprobe:ext4_file_read_iter { @[tid] = nsecs; }
kretprobe:ext4_file_read_iter {
    $lat = (nsecs - @[tid]) / 1000;
    if ($lat > 1000) {
        printf("ext4 slow read: %s lat=%dµs\n", comm, $lat);
    }
    @ext4_lat = hist($lat);
    delete(@[tid]);
}
END { print(@ext4_lat); }'
```

---

## 15.17 livepatch

### 热补丁原理

```c
/* kernel/livepatch/ — 无需重启修复内核 bug */

/* livepatch 流程：
 * 1. 编写补丁模块（替换有 bug 的函数）
 * 2. insmod 补丁模块
 * 3. 内核修改函数入口：添加 trampoline 跳转到新函数
 * 4. 一致性模型确保安全切换（所有 CPU 退出旧函数后才生效）
 */

/* 补丁模块示例 */
#include <linux/livepatch.h>

/* 替换函数（新实现）*/
static int livepatch_cmdline_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "%s\n", saved_command_line);
    /* 修复：添加了换行符 */
    return 0;
}

/* 描述要替换的函数 */
static struct klp_func funcs[] = {
    {
        .old_name = "cmdline_proc_show",     /* 旧函数名 */
        .new_func = livepatch_cmdline_proc_show,  /* 新函数 */
    }, { }
};

/* 描述包含该函数的对象 */
static struct klp_object objs[] = {
    {
        /* .name = NULL 表示 vmlinux 本体 */
        .funcs = funcs,
    }, { }
};

/* 补丁描述 */
static struct klp_patch patch = {
    .mod = THIS_MODULE,
    .objs = objs,
};

static int livepatch_init(void)
{
    return klp_enable_patch(&patch);
}

static void livepatch_exit(void)
{
    /* livepatch 不支持卸载（安全原因），但可以禁用 */
}

module_init(livepatch_init);
module_exit(livepatch_exit);
MODULE_INFO(livepatch, "Y");
MODULE_LICENSE("GPL");
```

### livepatch 管理

```bash
# 内核配置
# CONFIG_LIVEPATCH=y

# 加载补丁
insmod my_livepatch.ko

# 查看补丁状态
cat /sys/kernel/livepatch/my_livepatch/enabled
# 1 = 已启用

# 查看过渡状态（等待一致性）
cat /sys/kernel/livepatch/my_livepatch/transition
# 1 = 正在过渡中
# 0 = 完成

# 禁用补丁（回滚到原始函数）
echo 0 > /sys/kernel/livepatch/my_livepatch/enabled

# 确认状态
ls /sys/kernel/livepatch/
```

---

## 15.18 性能优化清单

### CPU 性能分析

```bash
# 1. 检查 CPU 使用率和负载
top -b -n1 | head -20
mpstat -P ALL 1 5          # 每秒每核统计
pidstat -u -p ALL 1 5      # 进程级 CPU

# 2. CPU 调度延迟（运行队列长度）
vmstat 1 10                # r 列 = 运行队列
sar -q 1 10               # 队列和负载

# 3. 中断分布均衡
watch -n1 'cat /proc/interrupts | sort -k2 -rn | head -10'

# 4. CPU 缓存命中率
perf stat -e cache-misses,cache-references -a sleep 5
# miss rate > 5% 需要优化内存访问局部性

# 5. CPU 频率和节流
cpupower frequency-info
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
# 频率低 = 可能被热节流
cat /sys/class/thermal/thermal_zone*/temp  # 温度

# 优化建议：
# - 设置 CPU 调速器为 performance：
#   cpupower frequency-set -g performance
# - 禁用 C-state 深睡眠（低延迟场景）：
#   cpupower idle-set -D 1
# - NUMA 绑定：numactl --cpunodebind=0 --membind=0 ./app
```

### 内存性能分析

```bash
# 1. 内存使用概况
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|Cached|Dirty|Writeback"

# 2. 内存分配器统计
cat /proc/slabinfo | sort -k3 -rn | head -20  # 最大的 slab
slabtop                                         # 实时 slab 视图

# 3. 页缓存压力
cat /proc/vmstat | grep -E "pgmajfault|pgfault|pswpin|pswpout"
# pgmajfault 高 = 大量缺页（可能 swap 抖动）

# 4. NUMA 统计
numastat -c
numastat -m | head -20     # 内存使用
# node_miss 高 = 大量跨节点访问，考虑 NUMA 优化

# 5. 内存带宽（Intel MLC 或 stream）
stream                     # 标准内存带宽测试

# 优化建议：
# - 大页：echo always > /sys/kernel/mm/transparent_hugepage/enabled
# - NUMA 感知分配：numactl, libnuma
# - 减少 dirty 页积压：
#   sysctl vm.dirty_ratio=5
#   sysctl vm.dirty_background_ratio=2
```

### IO 性能分析

```bash
# 1. 磁盘 IO 统计
iostat -x 1 10
# await 列：平均 IO 等待时间（ms）
# %util 列：设备利用率
# r_await/w_await：读写分别的等待时间

# 2. IO 延迟分布（bpftrace）
bpftrace -e '
tracepoint:block:block_rq_complete {
    @[args->rwbs] = hist(args->nr_sector * 512);
}' &
sleep 10; kill %1

# 3. IO 调度器查看
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# 4. 文件系统缓存命中率
bpftrace -e '
kprobe:__do_page_cache_readahead { @readahead = count(); }
kprobe:mark_page_accessed { @hits = count(); }
interval:s:5 {
    print(@readahead);
    print(@hits);
    clear(@readahead);
    clear(@hits);
}'

# 优化建议：
# - SSD 使用 none 或 mq-deadline 调度器：
#   echo mq-deadline > /sys/block/nvme0n1/queue/scheduler
# - 调整 readahead：
#   blockdev --setra 2048 /dev/sda  # 1MB readahead
# - 挂载选项：noatime,nodiratime,data=writeback（ext4）
# - io_uring 代替 epoll（高并发 IO）
```

### 网络性能分析

```bash
# 1. 网络吞吐和错误
sar -n DEV 1 10
ip -s link show eth0
ethtool -S eth0 | grep -E "error|drop|miss"

# 2. TCP 统计
ss -s                          # socket 摘要
netstat -s | grep -i "retransmit\|error\|fail"
cat /proc/net/netstat | \
    awk 'NR%2==0{for(i=1;i<=NF;i++) printf "%-30s %s\n", h[i], $i}
         NR%2==1{for(i=1;i<=NF;i++) h[i]=$i}' | \
    grep -E "Retrans|TCPLost|TCPSpuriousRTOs"

# 3. 网络中断分配
cat /proc/interrupts | grep eth

# 4. 软中断接收速率
watch -n1 'cat /proc/softirqs | grep NET'

# 优化建议：
# - 网卡多队列 + CPU 亲和性绑定：
#   ethtool -L eth0 combined 8
#   for i in $(seq 0 7); do
#     echo $i > /proc/irq/$(cat /sys/class/net/eth0/queues/rx-$i/rps_cpus)/smp_affinity_list
#   done
# - 增大 socket 缓冲区：
#   sysctl net.core.rmem_max=134217728
#   sysctl net.core.wmem_max=134217728
#   sysctl net.ipv4.tcp_rmem="4096 87380 134217728"
# - 启用 GRO/GSO：
#   ethtool -K eth0 gro on gso on tso on
# - 增大 backlog：
#   sysctl net.core.somaxconn=65535
#   sysctl net.ipv4.tcp_max_syn_backlog=65535
# - 减少 TIME_WAIT：
#   sysctl net.ipv4.tcp_tw_reuse=1
#   sysctl net.ipv4.tcp_fin_timeout=15
```

### 综合性能快速检查脚本

```bash
#!/bin/bash
# 快速性能快照

echo "=== CPU ===" 
uptime
mpstat -P ALL 1 1 | tail -5

echo "=== 内存 ==="
free -h
cat /proc/meminfo | grep -E "HugePages|Dirty|Writeback" 

echo "=== IO ==="
iostat -x 1 1 | tail -10

echo "=== 网络 ==="
ss -s
ip -s link | grep -A4 "eth0\|ens\|enp"

echo "=== 进程 TOP5 CPU ==="
ps aux --sort=-%cpu | head -6

echo "=== 进程 TOP5 内存 ==="
ps aux --sort=-%mem | head -6

echo "=== 最近内核错误 ==="
dmesg -T | grep -E "error|BUG|WARN|OOM" | tail -10

echo "=== 系统调用热点（5秒）==="
perf stat -e 'syscalls:sys_enter_*' -a sleep 5 2>&1 | \
    grep -v "0 " | sort -k1 -rn | head -10
```

---

## 参考资料

| 资源 | 链接/位置 |
|------|----------|
| perf wiki | `https://perf.wiki.kernel.org/` |
| FlameGraph | `https://github.com/brendangregg/FlameGraph` |
| bpftrace 参考 | `https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md` |
| KASAN 文档 | `Documentation/dev-tools/kasan.rst` |
| lockdep 设计 | `Documentation/locking/lockdep-design.rst` |
| ftrace 文档 | `Documentation/trace/ftrace.rst` |
| kdump 指南 | `Documentation/admin-guide/kdump/kdump.rst` |
| Brendan Gregg 博客 | `http://www.brendangregg.com/` |
| Linux 性能 | `http://www.brendangregg.com/linuxperf.html` |

```bash
# 调试工具快速安装
apt install -y \
    linux-tools-$(uname -r) \   # perf
    trace-cmd \                  # ftrace 前端
    bpftrace \                   # bpftrace
    bpfcc-tools \                # BCC tools
    crash \                      # kernel crash analyzer
    gdb \                        # 调试器
    rt-tests \                   # cyclictest
    sysstat \                    # iostat/mpstat/sar
    numactl                      # NUMA 工具
```
