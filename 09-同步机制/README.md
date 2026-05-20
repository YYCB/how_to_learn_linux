# 09 — 同步机制

> 多处理器（SMP）环境下，多个 CPU 可能同时访问共享数据。
> 本章从**为什么需要同步 → 各种锁的实现原理 → 使用场景**，
> 对照 Linux 0.11（单处理器，禁中断）与 Linux 2.6.0（SMP，丰富锁原语）拆解。

---

## 1. 并发场景与竞争条件

```
场景：两个 CPU 同时对计数器 count++ 操作

CPU0                    CPU1
────                    ────
读 count = 5
                        读 count = 5
count + 1 = 6
                        count + 1 = 6
写 count = 6
                        写 count = 6    ← 应该是 7，实际是 6！

根本原因：count++ 不是原子操作
  汇编展开为：
    MOV eax, [count]   ; 读
    ADD eax, 1         ; 加
    MOV [count], eax   ; 写
  三条指令可被中断/并发破坏
```

---

## 2. Linux 0.11：关中断（单核时代）

```c
/* 单核系统：只需关闭中断即可防止并发 */

#define cli() __asm__ ("cli"::)   /* 关中断 */
#define sti() __asm__ ("sti"::)   /* 开中断 */

/* 典型用法 */
void foo(void)
{
    cli();          /* 关中断 → 不会被抢占 */
    /* ... 访问共享数据 ... */
    sti();          /* 开中断 */
}
```

**局限**：SMP（多核）系统中，关中断只能防止**本 CPU** 的中断，
无法防止**其他 CPU** 的并发访问。

---

## 3. 原子操作（Atomic Operations）

原子操作由 CPU 硬件保证，不需要锁：

```c
/* include/asm-i386/atomic.h — Linux 2.6.0 */
typedef struct { volatile int counter; } atomic_t;

/* 原子读写 */
#define atomic_read(v)     ((v)->counter)
#define atomic_set(v, i)   (((v)->counter) = (i))

/* 原子加减 */
static inline void atomic_add(int i, atomic_t *v)
{
    __asm__ __volatile__(
        LOCK "addl %1,%0"     /* LOCK 前缀：总线锁定，多核安全 */
        :"=m" (v->counter)
        :"ir" (i), "m" (v->counter));
}

static inline int atomic_inc_and_test(atomic_t *v)
{
    unsigned char c;
    __asm__ __volatile__(
        LOCK "incl %0; sete %1"
        :"=m" (v->counter), "=qm" (c)
        :"m" (v->counter) : "memory");
    return c != 0;
}

/* 使用场景：引用计数 */
atomic_t ref_count = ATOMIC_INIT(1);
atomic_inc(&ref_count);           /* 增加引用 */
if (atomic_dec_and_test(&ref_count))  /* 减少引用，若为0则释放 */
    kfree(object);
```

**`LOCK` 前缀的作用**：

```
单核：LOCK 为空（不需要）
SMP：  LOCK → 在总线事务期间锁定内存总线
              （或 cache line 锁，更现代的实现）
```

---

## 4. 自旋锁（Spinlock）

### 原理

```
获取锁失败时，不休眠，而是"自旋"（busy-wait）不断重试：

      尝试获取锁
      ┌──────────────────┐
      │ locked == 0?     │
      │ 是 → 设为1，返回 │
      │ 否 → 继续等待    │
      └──────────────────┘
           ↑  ↓（自旋）
           └──┘
      ← 其他 CPU 释放锁 →
```

### Linux 2.6.0 实现

```c
/* include/asm-i386/spinlock.h */
typedef struct {
    volatile unsigned int lock;
} spinlock_t;

#define SPIN_LOCK_UNLOCKED (spinlock_t) { 1 }

static inline void spin_lock(spinlock_t *lock)
{
    __asm__ __volatile__(
        spin_lock_string    /* 原子比较并交换，或 xchg */
        :"=m" (lock->lock) : : "memory");
}

static inline void spin_unlock(spinlock_t *lock)
{
    __asm__ __volatile__(
        spin_unlock_string
        :"=m" (lock->lock) : : "memory");
}
```

**x86 自旋锁的本质（test-and-set）**：

```asm
/* 获取锁：原子地将 lock 置 0（0=locked, 1=unlocked）*/
spin_lock:
    lock decb (%eax)    ; 原子减1
    jns spin_acquired   ; 若结果 >= 0（原来 > 0），获取成功
spin_retry:
    pause               ; 提示 CPU 在自旋（节省电力，优化超线程）
    cmpb $0, (%eax)    ; 检查锁是否释放
    jle spin_retry      ; 若仍锁定，继续等待
    lock decb (%eax)
    jns spin_acquired
    jmp spin_retry
spin_acquired:
    ret

/* 释放锁 */
spin_unlock:
    movb $1, (%eax)     ; 写 1 表示解锁（不需要 LOCK 前缀，store 已是原子）
    ret
```

### 使用规则

```c
spinlock_t my_lock = SPIN_LOCK_UNLOCKED;

/* 中断上下文安全版本（禁止本 CPU 中断）*/
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);
/* ... 临界区 ... */
spin_unlock_irqrestore(&my_lock, flags);

/* 普通版本（不禁中断，仅用于非中断上下文）*/
spin_lock(&my_lock);
/* ... 临界区 ... */
spin_unlock(&my_lock);
```

**适用场景**：临界区**极短**（几条指令），不能休眠（中断上下文）。

---

## 5. 互斥量（Mutex / Semaphore）

### 信号量（Semaphore）

```c
/* include/asm-i386/semaphore.h */
struct semaphore {
    atomic_t count;           /* 信号量计数 */
    int sleepers;             /* 等待中的进程数 */
    wait_queue_head_t wait;   /* 等待队列 */
};

/* 初始化为互斥量（count=1）*/
static DECLARE_MUTEX(my_mutex);  /* count = 1 */

/* 获取（P 操作）：count-- */
void down(struct semaphore *sem)
{
    /* 如果 count > 0，成功减1返回 */
    /* 如果 count == 0，将进程加入等待队列，休眠 */
}

/* 释放（V 操作）：count++ */
void up(struct semaphore *sem)
{
    /* count++ */
    /* 如果有等待的进程，唤醒一个 */
}
```

**与自旋锁的区别**：

```
             自旋锁                互斥量（信号量）
等待方式     忙等（自旋）           休眠（让出 CPU）
适用场景     临界区极短，中断上下文  临界区可能较长，进程上下文
开销         自旋期间浪费 CPU        上下文切换开销
可以休眠?    否                      是
```

### 读写信号量（rwsem）

```c
struct rw_semaphore rwsem = __RWSEM_INITIALIZER(rwsem);

/* 多个读者可以同时持有 */
down_read(&rwsem);
/* ... 读操作 ... */
up_read(&rwsem);

/* 写者需要独占 */
down_write(&rwsem);
/* ... 写操作 ... */
up_write(&rwsem);
```

---

## 6. RCU（Read-Copy-Update）

RCU 是 Linux 2.6.0 中引入的一种**无锁**并发技术，
专门为**读多写少**场景优化：

### 核心思想

```
RCU 的核心约束：
  · 读者：绝对不阻塞（也不需要加任何锁）
  · 写者：修改时创建副本，不影响正在读的读者

读者                 写者
──────               ──────
rcu_read_lock()      old_ptr = rcu_dereference(global_ptr)
data = rcu_dereference(ptr)  new_ptr = kmalloc(...)
/* 使用 data，不加锁 */      /* 修改 new_ptr */
rcu_read_unlock()    rcu_assign_pointer(global_ptr, new_ptr)
                     /* 等待所有读者完成当前临界区 */
                     synchronize_rcu()   ← 等待"宽限期"过去
                     kfree(old_ptr)      ← 安全释放旧数据

宽限期（Grace Period）：
  等待所有 CPU 都经历过一次上下文切换
  此后可保证没有读者还在使用旧数据
```

### 内核使用示例

```c
/* 链表的 RCU 遍历（无锁读）*/
rcu_read_lock();
list_for_each_entry_rcu(entry, &my_list, list) {
    /* 安全读取 entry，无锁 */
    do_something(entry);
}
rcu_read_unlock();

/* 安全删除链表元素（写者）*/
spin_lock(&list_lock);
list_del_rcu(&entry->list);
spin_unlock(&list_lock);
synchronize_rcu();    /* 等待所有读者完成 */
kfree(entry);         /* 现在可以安全释放 */
```

---

## 7. 死锁与调试

### 常见死锁场景

```
场景一：嵌套加锁顺序不一致

CPU0                    CPU1
────                    ────
lock(A)                 lock(B)
lock(B)  ← 等待B        lock(A)  ← 等待A
   ↑                         ↑
   └─────── 死锁! ────────────┘

避免方法：所有地方按相同顺序加锁（A 总是先于 B）

场景二：中断上下文与进程上下文

进程：spin_lock(&lock)
    此时发生中断
中断处理：spin_lock(&lock)  ← 自旋等待，但进程永远无法释放锁！

避免方法：进程中使用 spin_lock_irqsave()
```

### 调试工具

```bash
# lockdep：内核内置死锁检测器
# 编译时开启：CONFIG_PROVE_LOCKING=y

# 当检测到潜在死锁时，内核打印警告：
# [ BUG: possible circular locking dependency detected ]

# ftrace：跟踪锁获取/释放
echo function > /sys/kernel/debug/tracing/current_tracer
echo spin_lock > /sys/kernel/debug/tracing/set_ftrace_filter
cat /sys/kernel/debug/tracing/trace

# perf：分析锁竞争热点
perf lock record ls
perf lock report
```

---

## 8. 各同步机制对比总结

```
┌─────────────────┬──────────────┬─────────────┬──────────────────┐
│   机制           │  读性能      │  写性能      │  适用场景         │
├─────────────────┼──────────────┼─────────────┼──────────────────┤
│ 关中断（0.11）   │  最快        │  最快        │  单核，短临界区   │
│ 原子操作        │  快          │  快          │  单变量计数/标志  │
│ 自旋锁          │  快（无竞争）│  快（无竞争）│  短临界区，中断上下文│
│ 互斥量/信号量   │  慢（可休眠）│  慢（可休眠）│  长临界区，进程上下文│
│ 读写锁（rwlock）│  快（并发读）│  慢（独占写）│  读多写少         │
│ RCU             │  极快（无锁）│  中（写+等待）│  读极多写极少    │
└─────────────────┴──────────────┴─────────────┴──────────────────┘
```

> **选择原则**：
> 1. 能用原子操作就不用锁
> 2. 中断上下文用自旋锁
> 3. 进程上下文且临界区短用自旋锁，长用互斥量
> 4. 读远多于写用 RCU 或读写锁
