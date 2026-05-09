# Futex 深入分析

本文档对 glibc NPTL 中 futex（Fast Userspace muTEX）机制进行全面深入分析，
涵盖系统调用封装、三态锁协议、操作语义、各同步原语的 futex 使用模式、
PI（优先级继承）futex、robust（健壮）futex 及 AArch64 平台考量。

---

## 目录

1. [Futex 概述与设计哲学](#1-futex-概述与设计哲学)
2. [操作常量与系统调用封装](#2-操作常量与系统调用封装)
3. [三态锁协议（lll_lock）](#3-三态锁协议lll_lock)
4. [PRIVATE 标志优化](#4-private-标志优化)
5. [各同步原语的 futex 使用模式](#5-各同步原语的-futex-使用模式)
6. [PI Futex（优先级继承）](#6-pi-futex优先级继承)
7. [Robust Futex（健壮互斥锁）](#7-robust-futex健壮互斥锁)
8. [超时与时钟处理](#8-超时与时钟处理)
9. [futex2 / futex_waitv 支持现状](#9-futex2--futex_waitv-支持现状)
10. [AArch64 平台考量](#10-aarch64-平台考量)
11. [Futex 使用全景表](#11-futex-使用全景表)
12. [源文件快速索引](#12-源文件快速索引)

---

## 1. Futex 概述与设计哲学

### 1.1 什么是 Futex

Futex（Fast Userspace muTEX）是 Linux 内核提供的一种高效同步原语。其核心
设计思想是：

```
┌───────────────────────────────────────────────────────────────────┐
│                    Futex 设计哲学                                  │
│                                                                   │
│  "在无竞争的情况下，同步操作完全在用户空间完成（原子指令），        │
│   只有在存在竞争需要阻塞时，才陷入内核。"                          │
│                                                                   │
│  ┌────────────┐     无竞争      ┌────────────────────┐            │
│  │  用户空间   │ ──────────────► │ 原子 CAS / Exchange │            │
│  │  快速路径   │   ~5-10ns      │ 直接返回            │            │
│  └────────────┘                 └────────────────────┘            │
│                                                                   │
│  ┌────────────┐     有竞争      ┌────────────────────┐            │
│  │  内核慢路径 │ ──────────────► │ futex syscall       │            │
│  │  阻塞等待   │   ~200ns+      │ 进程挂起/唤醒       │            │
│  └────────────┘                 └────────────────────┘            │
└───────────────────────────────────────────────────────────────────┘
```

Futex 的关键组成：
- **Futex 字（futex word）**：用户空间中的一个 `unsigned int`，表示应用特定条件
- **内核状态**：与该 futex 字关联的等待队列

### 1.2 glibc 中的角色

glibc 的所有 NPTL 同步原语（mutex、condvar、rwlock、barrier、semaphore、
once、thread join）都建立在 futex 之上。glibc 提供了多层封装：

```
   应用层:  pthread_mutex_lock()  pthread_cond_wait()  sem_wait() ...
              │                      │                    │
              ▼                      ▼                    ▼
   NPTL层:  lll_lock()           __futex_abstimed_      futex_wait()
            lll_unlock()         wait_cancelable64()    futex_wake()
              │                      │                    │
              ▼                      ▼                    ▼
   包装层:  futex_wait()         futex_wait_simple()   futex_unlock_pi()
            futex_wake()         __futex_clocklock64()
              │                      │                    │
              ▼                      ▼                    ▼
   系统调用: lll_futex_timed_wait()  lll_futex_wake()   lll_futex_syscall()
              │
              ▼
   内核:    sys_futex / sys_futex_time64
```

---

## 2. 操作常量与系统调用封装

### 2.1 Futex 操作常量

所有 futex 操作常量定义在 `lowlevellock-futex.h` 中：

```c
/* lowlevellock-futex.h:27-44 */
#define FUTEX_WAIT              0     /* 等待 futex 字等于期望值 */
#define FUTEX_WAKE              1     /* 唤醒指定数量的等待者 */
#define FUTEX_REQUEUE           3     /* 将等待者从一个 futex 转移到另一个 */
#define FUTEX_CMP_REQUEUE       4     /* 带条件检查的 requeue */
#define FUTEX_WAKE_OP           5     /* 原子修改 + 条件唤醒 */
#define FUTEX_LOCK_PI           6     /* PI 协议锁定 */
#define FUTEX_UNLOCK_PI         7     /* PI 协议解锁 */
#define FUTEX_TRYLOCK_PI        8     /* PI 协议尝试锁定 */
#define FUTEX_WAIT_BITSET       9     /* 带位掩码的等待 */
#define FUTEX_WAKE_BITSET      10     /* 带位掩码的唤醒 */
#define FUTEX_WAIT_REQUEUE_PI  11     /* PI requeue 等待 */
#define FUTEX_CMP_REQUEUE_PI   12     /* PI 条件 requeue */
#define FUTEX_LOCK_PI2         13     /* PI 锁定 v2 (支持 CLOCK_MONOTONIC) */
#define FUTEX_PRIVATE_FLAG    128     /* 进程内私有标志 */
#define FUTEX_CLOCK_REALTIME  256     /* 使用 CLOCK_REALTIME */
#define FUTEX_BITSET_MATCH_ANY 0xffffffff  /* 匹配所有位 */
```

### 2.2 操作语义速查

| 操作 | 语义 | glibc 使用场景 |
|------|------|---------------|
| `FUTEX_WAIT` | 若 `*futex == expected` 则阻塞 | lll_lock 慢路径 |
| `FUTEX_WAKE` | 唤醒最多 N 个等待者 | lll_unlock、sem_post |
| `FUTEX_WAIT_BITSET` | 带位掩码的 WAIT + 支持绝对时间 | condvar wait、timed lock |
| `FUTEX_CMP_REQUEUE` | 唤醒 N 个 + 移动 M 个到另一 futex | condvar broadcast |
| `FUTEX_WAKE_OP` | 在第二个 futex 上原子操作 + 条件唤醒 | condvar signal (旧版) |
| `FUTEX_LOCK_PI` | 内核管理的优先级继承锁 | PI mutex |
| `FUTEX_UNLOCK_PI` | PI 锁释放 + 唤醒最高优先级等待者 | PI mutex unlock |
| `FUTEX_LOCK_PI2` | PI 锁 v2，支持 CLOCK_MONOTONIC | PI timed mutex |

### 2.3 核心封装函数

#### futex_wait — 基础等待

```c
/* futex-internal.h:123-144 */
static __always_inline int
futex_wait (unsigned int *futex_word, unsigned int expected, int private)
{
  int err = lll_futex_timed_wait (futex_word, expected, NULL, private);
  switch (err) {
  case 0:       return 0;        /* 被唤醒 */
  case -EAGAIN: return EAGAIN;   /* futex 字 != expected */
  case -EINTR:  return EINTR;    /* 被信号中断 */
  default:      futex_fatal_error ();  /* bug, 终止进程 */
  }
}
```

#### futex_wake — 唤醒等待者

```c
/* futex-internal.h:186-207 */
static __always_inline void
futex_wake (unsigned int *futex_word, int processes_to_wake, int private)
{
  int res = lll_futex_wake (futex_word, processes_to_wake, private);
  if (res >= 0) return;          /* 正常 */
  switch (res) {
  case -EFAULT:                  /* 内存可能被复用（合法情况） */
  case -EINVAL: return;          /* 对齐错误或 PI futex 复用 */
  default: futex_fatal_error (); /* 不可恢复错误 */
  }
}
```

#### futex_wait_simple — 忽略返回值的等待

```c
/* futex-internal.h:153-158 */
static __always_inline void
futex_wait_simple (unsigned int *futex_word, unsigned int expected, int private)
{
  ignore_value (futex_wait (futex_word, expected, private));
}
```

典型用法：在循环中反复检查条件 + futex_wait_simple，不需要区分唤醒原因。

---

## 3. 三态锁协议（lll_lock）

### 3.1 状态机定义

lll_lock（Low-Level Lock）使用三个状态实现高效互斥：

```c
/* lowlevellock.h:26-41 — 状态定义 */
/*  0:  未获取
 *  1:  已获取，无等待者 — 释放时无需 futex_wake
 *  >1: 已获取，可能有等待者 — 释放时必须 futex_wake
 */
```

```
                         ┌───────────────────────────────────┐
                         │         三态锁状态机              │
                         └───────────────────────────────────┘

       CAS(0→1) 成功                    CAS(0→1) 失败
     ┌─────────────┐               ┌─────────────────────────┐
     │             │               │                         │
     ▼             │               │    exchange(→2)          │
  ┌──────┐    ┌────┴────┐    ┌─────▼────┐              ┌─────────┐
  │  0   │    │   1     │    │    2     │              │ futex   │
  │ 空闲 │────│已锁/无  │────│已锁/有  │──────────────│ _wait   │
  │      │    │ 等待者  │    │ 等待者  │   *futex==2  │ 阻塞    │
  └──────┘    └────┬────┘    └────┬────┘              └────┬────┘
     ▲             │              │                        │
     │   exchange  │   exchange   │                        │
     │   (→0)      │   (→0)       │                        │
     │   oldval=1  │   oldval>1   │                        │
     │   无需wake  │   需要wake   │                        │
     │             │              │                        │
     └─────────────┘              └────────────────────────┘
                          futex_wake(1) 唤醒一个等待者
```

### 3.2 加锁路径

```c
/* lowlevellock.h:93-107 */
#define __lll_lock(futex, private)                          \
  ({                                                        \
    int *__futex = (futex);                                 \
    if (__glibc_unlikely                                    \
        (atomic_compare_and_exchange_bool_acq(__futex,1,0)))\
      {  /* CAS(0→1)失败，存在竞争 */                       \
        if (private == LLL_PRIVATE)                         \
          __lll_lock_wait_private(__futex);                  \
        else                                                \
          __lll_lock_wait(__futex, private);                 \
      }                                                     \
  })
```

慢路径实现（`lowlevellock.c:24-52`）：

```c
void __lll_lock_wait_private (int *futex)
{
  if (atomic_load_relaxed(futex) == 2)    /* 已是状态 2，直接等 */
    goto futex;

  while (atomic_exchange_acquire(futex, 2) != 0)  /* 设为 2 并检查 */
    {
    futex:
      futex_wait((unsigned int *)futex, 2, LLL_PRIVATE);
    }
    /* 循环退出时：exchange 返回 0，说明我们获得了锁，且状态为 2 */
}
```

**关键设计要点：**
1. 先尝试 CAS(0→1)，无竞争时一条原子指令即完成
2. 竞争时将状态设为 2（`exchange`），告知持锁者释放时必须唤醒
3. `futex_wait` 检查 `*futex == 2` 才阻塞，避免 lost wakeup
4. 循环：每次被唤醒重新 `exchange(→2)` 竞争锁

### 3.3 解锁路径

```c
/* lowlevellock.h:144-159 */
#define __lll_unlock(futex, private)                        \
  ({                                                        \
    int *__futex = (futex);                                 \
    int __private = (private);   /* 先读 private */         \
    int __oldval = atomic_exchange_release(__futex, 0);     \
    if (__glibc_unlikely(__oldval > 1))  /* 有等待者？ */   \
      {                                                     \
        __lll_lock_wake(__futex, __private);                 \
      }                                                     \
  })
```

唤醒实现（`lowlevellock.c:54-66`）：

```c
void __lll_lock_wake_private (int *futex)
{
  lll_futex_wake (futex, 1, LLL_PRIVATE);   /* 唤醒 1 个等待者 */
}
```

**设计要点：**
1. `exchange(→0)` 释放锁并获取旧值
2. 旧值 = 1：无等待者，无需系统调用 → **快速路径**
3. 旧值 > 1：可能有等待者，必须 `futex_wake` → **慢路径**
4. 先读 `private` 再释放锁，避免访问已被销毁的 mutex

### 3.4 条件变量特化版 lll_cond_lock

```c
/* lowlevellock.h:117-124 */
#define __lll_cond_lock(futex, private)                     \
  ({                                                        \
    int *__futex = (futex);                                 \
    if (__glibc_unlikely(atomic_exchange_acquire(__futex,2)!=0)) \
      __lll_lock_wait(__futex, private);                    \
  })
```

与普通 `lll_lock` 的区别：**总是设为 2**（不是先试 1 再设 2），因为条件
变量唤醒后通常有多个线程竞争同一把锁，保守地假设有等待者更高效。

---

## 4. PRIVATE 标志优化

### 4.1 设计原理

```c
/* lowlevellock-futex.h:46-54 */
#define LLL_PRIVATE  0                    /* 进程内私有 */
#define LLL_SHARED   FUTEX_PRIVATE_FLAG   /* 跨进程共享 (=128) */

#define __lll_private_flag(fl, private) \
  (((fl) | FUTEX_PRIVATE_FLAG) ^ (private))
```

**逻辑解读：**

| `private` 值 | 计算 `(fl | 128) ^ private` | 结果 |
|-------------|---------------------------|------|
| `LLL_PRIVATE=0` | `(fl | 128) ^ 0` = `fl | 128` | **带** FUTEX_PRIVATE_FLAG |
| `LLL_SHARED=128` | `(fl | 128) ^ 128` = `fl` | **不带** FUTEX_PRIVATE_FLAG |

这个看似"反直觉"的定义是因为：
- 大部分 futex 操作都是进程内私有的（默认值 0）
- 私有 futex 让内核跳过全局哈希表查找，直接使用进程本地哈希
- 性能差异在高竞争场景下可达 **2-3 倍**

### 4.2 使用模式

```c
/* futex-internal.h:66-67 */
#define FUTEX_PRIVATE LLL_PRIVATE   /* = 0 */
#define FUTEX_SHARED  LLL_SHARED    /* = 128 */
```

- **进程内 mutex/condvar/barrier**：使用 `FUTEX_PRIVATE`
- **进程间共享的同步对象**（`PTHREAD_PROCESS_SHARED`）：使用 `FUTEX_SHARED`
- 判断依据：`pthread_mutexattr_setpshared()` 设置的属性

---

## 5. 各同步原语的 futex 使用模式

### 5.1 pthread_mutex（互斥锁）

#### 普通 / 递归 / 错误检查互斥锁

```
pthread_mutex_lock()  [pthread_mutex_lock.c:70]
│
├─ type == TIMED_NP (普通):
│   └─ LLL_MUTEX_LOCK_OPTIMIZED(mutex)
│      = lll_lock(mutex->__data.__lock, private)
│      │
│      ├─ 快速路径: CAS(0→1) 成功 → 返回
│      └─ 慢路径: exchange(→2) + futex_wait(&__lock, 2)
│
├─ type == RECURSIVE_NP (递归):
│   ├─ __owner == self? → ++__count, 返回 (无 futex)
│   └─ 否则同普通路径
│
├─ type == ERRORCHECK_NP (错误检查):
│   ├─ __owner == self? → return EDEADLK (无 futex)
│   └─ 否则同普通路径
│
└─ type == ADAPTIVE_NP (自适应):
    ├─ 先自旋: LLL_MUTEX_TRYLOCK 循环
    │   指数退避 + 随机抖动 [pthread_mutex_lock.c:113-145]
    │   spin_count = exp_backoff + (jitter & (exp_backoff-1))
    └─ 自旋失败 → LLL_MUTEX_LOCK (标准 futex 慢路径)

pthread_mutex_unlock()  [pthread_mutex_unlock.c]
│
├─ 普通: lll_unlock(mutex->__data.__lock, private)
│   ├─ exchange(→0), oldval==1 → 无需 wake
│   └─ oldval>1 → futex_wake(&__lock, 1)
│
└─ Robust: atomic_exchange_release(&__lock, 0)
    └─ oldval & FUTEX_WAITERS → futex_wake(&__lock, 1)
       [pthread_mutex_unlock.c:164-166]
```

**Futex 字**：`mutex->__data.__lock`
**操作**：`FUTEX_WAIT` / `FUTEX_WAKE`

#### 自适应互斥锁的自旋策略

```c
/* pthread_mutex_lock.c:113-145 */
int cnt = 0;
int max_cnt = MIN(max_adaptive_count(), mutex->__data.__spins * 2 + 10);
int exp_backoff = 1;
unsigned int jitter = get_jitter();

do {
  spin_count = exp_backoff + (jitter & (exp_backoff - 1));
  cnt += spin_count;
  if (cnt >= max_cnt) {
    LLL_MUTEX_LOCK(mutex);    /* 放弃自旋，进入 futex */
    break;
  }
  do atomic_spin_nop(); while (--spin_count > 0);
  exp_backoff = get_next_backoff(exp_backoff);
} while (LLL_MUTEX_READ_LOCK(mutex) != 0 || LLL_MUTEX_TRYLOCK(mutex) != 0);

/* 自旋统计反馈: EWMA 更新 __spins */
mutex->__data.__spins += (cnt - mutex->__data.__spins) / 8;
```

### 5.2 pthread_cond（条件变量）

条件变量使用了复杂的 G1/G2 双组算法：

```
┌───────────────────────────────────────────────────────────────────┐
│              条件变量 G1/G2 组算法                                 │
│                                                                   │
│  __wseq:  等待者序列号 (全局递增)                                 │
│  __g_signals[2]: 两个组的信号计数                                 │
│  __g_size[2]:    两个组的等待者数                                 │
│                                                                   │
│  G1 (当前组): 正在接收信号的等待者组                               │
│  G2 (新组):   新加入的等待者组                                    │
│                                                                   │
│  signal → G1 有等待者? → 是: ++__g_signals[g1], --__g_size[g1]    │
│                          否: G2 有等待者? → 切换 G1←→G2           │
│                                                                   │
│  wait → 注册 wseq → 加入当前 G2 → futex_wait(__g_signals[g])     │
│  唤醒 → 检查 signals > 0 → 消费信号 → 重新获取 mutex             │
└───────────────────────────────────────────────────────────────────┘
```

```
pthread_cond_wait()  [pthread_cond_wait.c:329-445]
│
├─ 释放关联 mutex
├─ 注册序列号 (wseq)
├─ 确定所属组 g = (wseq & 1) ^ 1
│
├─ 循环:
│   ├─ 检查 __g_signals[g] > 0? → 消费信号, break
│   └─ __futex_abstimed_wait_cancelable64(
│        cond->__data.__g_signals + g,   /* futex 字 */
│        signals,                         /* 期望值 */
│        clockid, abstime, private)
│      [pthread_cond_wait.c:421-422]
│
└─ 重新获取 mutex

pthread_cond_signal()  [pthread_cond_signal.c:34-94]
│
├─ wrefs >> 3 == 0? → 无等待者，直接返回
├─ 获取内部锁
├─ g1 = 当前接收信号的组
├─ __g_size[g1] != 0? → ++__g_signals[g1], --__g_size[g1]
│  否则 → __condvar_switch_g1() 切换组
├─ 释放内部锁
└─ futex_wake(__g_signals + g1, 1, private)
   [pthread_cond_signal.c:91-92]
```

**Futex 字**：`cond->__data.__g_signals[g]`（g = 0 或 1）
**操作**：`FUTEX_WAIT_BITSET`（通过 `__futex_abstimed_wait_cancelable64`）/ `FUTEX_WAKE`

**Lost wakeup 防护：**
1. 等待者先注册序列号，再检查信号，最后才 futex_wait
2. 信号者在内部锁保护下递增信号计数后才 futex_wake
3. 取消的等待者若已消费信号，会发送替代信号（`pthread_cond_wait.c:136-143`）

### 5.3 pthread_rwlock（读写锁）

读写锁使用 **多个 futex 字** 分离不同类型的等待：

```
┌──────────────────────────────────────────────────────────────────┐
│              读写锁 Futex 字分工                                  │
│                                                                  │
│  __readers:        读者计数 + 标志位（极少用于 futex wait）       │
│  __wrphase_futex:  写阶段等待（读者等写阶段结束 / 写者等获取）   │
│  __writers_futex:  写者间接力（写者等待另一写者释放）             │
│                                                                  │
│  分离原因: __readers 变化太频繁，不适合作为 futex 等待字          │
└──────────────────────────────────────────────────────────────────┘
```

```
pthread_rwlock_rdlock()  [pthread_rwlock_common.c]
│
├─ 快速路径: atomic_add(&__readers) 增加读者计数
│   若在读阶段且无写者 → 直接返回（无 futex）
│
└─ 慢路径: 处于写阶段
    └─ __futex_abstimed_wait64(&__wrphase_futex, ...)
       [pthread_rwlock_common.c:454]

pthread_rwlock_wrlock()  [pthread_rwlock_common.c]
│
├─ CAS 获取写权限
│
├─ 等待读者清空:
│   └─ __futex_abstimed_wait64(&__wrphase_futex, ...)
│      [pthread_rwlock_common.c:823]
│
└─ 等待其他写者:
    └─ __futex_abstimed_wait64(&__writers_futex, ...)
       [pthread_rwlock_common.c:724]

pthread_rwlock_unlock()
│
├─ 写者释放 → futex_wake(&__wrphase_futex, INT_MAX)  唤醒所有等待读者
│   [pthread_rwlock_common.c:571]
│   + futex_wake(&__writers_futex, 1)  唤醒一个等待写者
│   [pthread_rwlock_common.c:577]
│
└─ 最后读者释放 → futex_wake(&__wrphase_futex, INT_MAX)
   [pthread_rwlock_common.c:266, 411]
```

### 5.4 pthread_barrier（屏障）

```
pthread_barrier_wait()  [pthread_barrier_wait.c:94-222]
│
├─ atomic_fetch_add_acq_rel(&bar->in, 1)  进入屏障
│
├─ 我是最后到达的线程？(i % count == 0 的回合)
│   ├─ YES: CAS 推进 current_round
│   │       futex_wake(&bar->current_round, INT_MAX, shared)
│   │       [pthread_barrier_wait.c:167]
│   │       return PTHREAD_BARRIER_SERIAL_THREAD
│   │
│   └─ NO:  等待回合完成
│           while (i > cr)
│             futex_wait_simple(&bar->current_round, cr, shared)
│             [pthread_barrier_wait.c:184]
│
└─ 重置逻辑 (最后离开者):
    atomic_store_release(&bar->in, 0)
    futex_wake(&bar->in, INT_MAX, shared)
    [pthread_barrier_wait.c:215-216]
```

**Futex 字**：`bar->current_round`（主等待）、`bar->in`（重置等待）
**操作**：`FUTEX_WAIT` / `FUTEX_WAKE`（唤醒所有 `INT_MAX`）

### 5.5 sem_wait / sem_post（信号量）

```
sem_post()  [sem_post.c:31-77]
│
├─ 64 位原子路径 (USE_64B_ATOMICS_ON_SEM_T):
│   ├─ CAS 循环: data += 1 (值在低 32 位, 等待者计数在高 32 位)
│   └─ (data >> SEM_NWAITERS_SHIFT) > 0?
│       → futex_wake(&data + SEM_VALUE_OFFSET, 1, private)
│       [sem_post.c:56]
│
└─ 32 位原子路径:
    ├─ CAS 循环: value += (1 << SEM_VALUE_SHIFT)
    └─ (value & SEM_NWAITERS_MASK) != 0?
        → futex_wake(&value, 1, private)
        [sem_post.c:73]

sem_wait()  [sem_waitcommon.c]
│
├─ 快速路径: 原子减 1 (若 > 0) → 成功
└─ 慢路径: futex_wait / __futex_abstimed_wait_cancelable64
```

**设计亮点：** 等待者计数嵌入在 futex 字中（64 位版本用高 32 位），
避免了额外的原子操作来维护等待者计数。

### 5.6 pthread_once（一次初始化）

```
pthread_once(once_control, init_routine)  [pthread_once.c:134-144]
│
├─ 快速路径:
│   val = atomic_load_acquire(once_control)
│   (val & __PTHREAD_ONCE_DONE) != 0 → return 0
│
└─ 慢路径: __pthread_once_slow()  [pthread_once.c:65-132]
    │
    ├─ CAS(val → __fork_generation | INPROGRESS)
    │
    ├─ CAS 成功 (我是执行者):
    │   ├─ 注册 cleanup handler (clear_once_control)
    │   ├─ 执行 init_routine()
    │   ├─ atomic_store_release(once_control, __PTHREAD_ONCE_DONE)
    │   └─ futex_wake(once_control, INT_MAX, FUTEX_PRIVATE)
    │      [pthread_once.c:127]
    │
    └─ CAS 失败 (其他线程正在执行):
        └─ futex_wait_simple(once_control, newval, FUTEX_PRIVATE)
           [pthread_once.c:105-106]
```

**状态机：**

```
   ┌──────────┐   CAS    ┌────────────────────┐   init完成   ┌─────────┐
   │    0     │ ────────► │  INPROGRESS |      │ ──────────► │  DONE   │
   │ 未初始化 │           │  fork_gen          │              │ 已完成  │
   └──────────┘           └────────────────────┘              └─────────┘
        ▲                        │                                 │
        │ cleanup (中断)         │ 其他线程                        │
        └────────────────────────┘ futex_wait                      │
                                                    load_acquire ──┘
                                                    (快速路径)
```

### 5.7 pthread_join（线程等待）

```
__pthread_clockjoin_ex()  [pthread_join_common.c:25-90]
│
├─ 快速路径:
│   atomic_load_acquire(&pd->joinstate) == THREAD_STATE_EXITED
│   → 直接返回
│
└─ 慢路径:
    while (state != THREAD_STATE_EXITED)
    │
    ├─ cancel 版本:
    │   __futex_abstimed_wait_cancelable64(&pd->joinstate, state, ...)
    │   [pthread_join_common.c:67-68]
    │
    └─ 非 cancel 版本:
        __futex_abstimed_wait64(&pd->joinstate, state, ...)
        [pthread_join_common.c:69-70]
```

**唤醒来源：** 内核通过 `CLONE_CHILD_CLEARTID` 标志，在线程退出时自动将
`joinstate` 清零并执行 `futex_wake`，无需 glibc 显式唤醒。

---

## 6. PI Futex（优先级继承）

### 6.1 问题：优先级反转

```
  优先级:  高 H ─────┐           低 L 持锁
                     │  H 阻塞等待    │
                     │  L 的锁        │  中 M 抢占 L
                     │  ◄─────────────┤  L 无法运行
                     │                │  H 间接被 M 阻塞!
                     │  这就是        │
                     │  优先级反转    │
                     ▼                ▼
```

### 6.2 PI Futex 解决方案

PI（Priority Inheritance）futex 让内核管理锁的持有者身份和优先级继承：

```c
/* futex-internal.c:144-205 */
int __futex_lock_pi64 (int *futex_word, clockid_t clockid,
                       const struct __timespec64 *abstime, int private)
{
  unsigned int clockbit = clockid == CLOCK_REALTIME
                          ? FUTEX_CLOCK_REALTIME : 0;
  int op_pi2 = __lll_private_flag(FUTEX_LOCK_PI2 | clockbit, private);

  /* 优先使用 FUTEX_LOCK_PI2 (支持 CLOCK_MONOTONIC) */
  /* 回退到 FUTEX_LOCK_PI (仅 CLOCK_REALTIME) */
  int op_pi1 = __lll_private_flag(FUTEX_LOCK_PI, private);
  int op_pi = abstime != NULL && clockid != CLOCK_REALTIME ? op_pi2 : op_pi1;

  err = INTERNAL_SYSCALL_CALL(futex_time64, futex_word, op_pi, 0, abstime);
  ...
}
```

**PI mutex 锁定流程**（`pthread_mutex_lock.c:340-495`）：

```
  ┌──────────────────────────────────────────────────────────────┐
  │  PI Mutex 工作原理                                           │
  │                                                              │
  │  futex 字 = owner_tid | FUTEX_WAITERS                       │
  │                                                              │
  │  锁定: FUTEX_LOCK_PI                                        │
  │    内核自动:                                                 │
  │    1. 若 *futex == 0 → 设为 caller_tid, 返回                │
  │    2. 若 *futex != 0 → 提升持锁者优先级到 max(waiters)      │
  │                        将 caller 加入优先级队列              │
  │                        阻塞                                  │
  │                                                              │
  │  解锁: FUTEX_UNLOCK_PI                                      │
  │    内核自动:                                                 │
  │    1. 恢复持锁者原始优先级                                   │
  │    2. 将锁传递给最高优先级等待者                             │
  │    3. 唤醒该等待者                                           │
  └──────────────────────────────────────────────────────────────┘
```

**PI 解锁**（`futex-internal.h:242-268`）：

```c
static __always_inline int
futex_unlock_pi (unsigned int *futex_word, int private)
{
  int err = lll_futex_timed_unlock_pi (futex_word, private);
  /* 处理多种合法返回值: EAGAIN, EINTR, ETIMEDOUT, ESRCH, EDEADLK, EPERM */
  ...
}
```

### 6.3 FUTEX_LOCK_PI vs FUTEX_LOCK_PI2

| 特性 | FUTEX_LOCK_PI (op=6) | FUTEX_LOCK_PI2 (op=13) |
|------|---------------------|----------------------|
| 时钟 | 仅 CLOCK_REALTIME | 支持 CLOCK_MONOTONIC |
| 内核版本 | 2.6.18+ | 5.14+ |
| glibc 策略 | 回退方案 | 优先使用 |

---

## 7. Robust Futex（健壮互斥锁）

### 7.1 问题：持锁者异常退出

普通互斥锁在持锁线程崩溃时会造成死锁。Robust mutex 利用内核的
`set_robust_list` 机制解决此问题。

### 7.2 工作原理

```
┌───────────────────────────────────────────────────────────────────┐
│                 Robust Futex 机制                                  │
│                                                                   │
│  线程启动时:                                                      │
│    set_robust_list(&pd->robust_head, sizeof(robust_list_head))   │
│    [dl-tls_init_tp.c:95-96]                                      │
│                                                                   │
│  加锁时:                                                         │
│    1. list_op_pending = &mutex->__data.__list.__next              │
│    2. CAS 获取锁 (__lock = tid | FUTEX_WAITERS)                  │
│    3. ENQUEUE_MUTEX(mutex) → 加入 robust_list                    │
│    4. list_op_pending = NULL                                      │
│    [pthread_mutex_lock.c:180-337]                                │
│                                                                   │
│  线程崩溃时 (内核处理):                                           │
│    1. 遍历线程的 robust_list                                     │
│    2. 对每个锁: *futex_word |= FUTEX_OWNER_DIED                  │
│    3. futex_wake() 唤醒等待者                                    │
│                                                                   │
│  等待者被唤醒:                                                    │
│    1. 检查 __lock & FUTEX_OWNER_DIED                             │
│    2. 若设置: 获取锁, __owner = INCONSISTENT, 返回 EOWNERDEAD    │
│    3. 调用者必须调用 pthread_mutex_consistent() 或销毁 mutex      │
│    [pthread_mutex_lock.c:208-253]                                │
└───────────────────────────────────────────────────────────────────┘
```

### 7.3 Futex 字的位域

```
┌──────────────────────────────────────────┐
│  Robust mutex 的 __lock 字段 (32位)      │
│                                          │
│  ┌───┬───┬──────────────────────────┐    │
│  │bit│bit│   bit 29-0               │    │
│  │31 │30 │                          │    │
│  │OD │WA │   TID (线程 ID)          │    │
│  └───┴───┴──────────────────────────┘    │
│                                          │
│  OD = FUTEX_OWNER_DIED  (0x40000000)     │
│  WA = FUTEX_WAITERS     (0x80000000)     │
│  TID_MASK              (0x3fffffff)      │
└──────────────────────────────────────────┘
```

### 7.4 解锁路径

```c
/* pthread_mutex_unlock.c:159-166 — Robust 解锁 */
private = PTHREAD_ROBUST_MUTEX_PSHARED(mutex);
if (__glibc_unlikely(
    (atomic_exchange_release(&mutex->__data.__lock, 0)
     & FUTEX_WAITERS) != 0))
  futex_wake((unsigned int *)&mutex->__data.__lock, 1, private);
```

---

## 8. 超时与时钟处理

### 8.1 超时封装层

glibc 提供统一的超时处理框架（`futex-internal.c:66-121`）：

```c
static int
__futex_abstimed_wait_common (unsigned int *futex_word,
                              unsigned int expected, clockid_t clockid,
                              const struct __timespec64 *abstime,
                              int private, bool cancel)
{
  /* 负超时直接返回 ETIMEDOUT */
  if (abstime != NULL && abstime->tv_sec < 0)
    return ETIMEDOUT;

  /* 选择时钟 */
  clockbit = (clockid == CLOCK_REALTIME) ? FUTEX_CLOCK_REALTIME : 0;

  /* 使用 FUTEX_WAIT_BITSET 实现绝对时间超时 */
  int op = __lll_private_flag(FUTEX_WAIT_BITSET | clockbit, private);

  /* 时间格式适配 (64位 vs 32位内核接口) */
  err = __futex_abstimed_wait_common64(...);
  ...
}
```

### 8.2 FUTEX_WAIT vs FUTEX_WAIT_BITSET

| 特性 | FUTEX_WAIT | FUTEX_WAIT_BITSET |
|------|-----------|-------------------|
| 超时类型 | 相对时间 | 绝对时间 |
| 时钟 | CLOCK_MONOTONIC | CLOCK_REALTIME / CLOCK_MONOTONIC |
| 位掩码 | 无 | FUTEX_BITSET_MATCH_ANY (0xffffffff) |
| 使用场景 | lll_lock 基础等待 | 所有带超时的等待 |

glibc 将 `FUTEX_WAIT_BITSET` + `FUTEX_BITSET_MATCH_ANY` 作为支持绝对
超时的标准等待操作，语义上等同于 `FUTEX_WAIT` 但超时为绝对时间。

### 8.3 可取消等待

```c
/* futex-internal.c:133-142 */
int __futex_abstimed_wait_cancelable64 (unsigned int *futex_word,
                                        unsigned int expected,
                                        clockid_t clockid,
                                        const struct __timespec64 *abstime,
                                        int private)
{
  return __futex_abstimed_wait_common(futex_word, expected, clockid,
                                     abstime, private, true /* cancel */);
}
```

`cancel=true` 时使用 `INTERNAL_SYSCALL_CANCEL`（而非 `INTERNAL_SYSCALL_CALL`），
使系统调用成为取消点，支持 `pthread_cancel` 中断等待。

### 8.4 __futex_clocklock64 — 三态锁 + 超时

```c
/* futex-internal.h:296-312 */
static __always_inline int
__futex_clocklock64 (int *futex, clockid_t clockid,
                     const struct __timespec64 *abstime, int private)
{
  if (__glibc_unlikely(
      atomic_compare_and_exchange_bool_acq(futex, 1, 0)))
    {
      while (atomic_exchange_acquire(futex, 2) != 0)
        {
          int err = __futex_abstimed_wait64(
            (unsigned int *)futex, 2, clockid, abstime, private);
          if (err == EINVAL || err == ETIMEDOUT || err == EOVERFLOW)
            return err;
        }
    }
  return 0;
}
```

这是三态锁协议 + 超时的组合：用于 `pthread_mutex_timedlock` 等场景。

---

## 9. futex2 / futex_waitv 支持现状

### 9.1 内核侧

Linux 5.16 引入了 `futex_waitv` 系统调用（`__NR_futex_waitv`），允许同时
等待多个 futex 字（类似 epoll 对 futex 的扩展）。

### 9.2 glibc 现状

glibc 2.43.9000 中：

- **有** `__NR_futex_waitv` 系统调用号定义（各架构的 `arch-syscall.h`）
- **有** `futex_waitv` 出现在 `syscall-names.list` 中
- **无** glibc 封装函数或内部使用

```
结论: glibc 目前仅注册了系统调用号，未提供 futex_waitv 的封装 API。
      应用程序可通过 syscall(__NR_futex_waitv, ...) 直接使用。
```

---

## 10. AArch64 平台考量

### 10.1 原子操作实现

AArch64 使用 LSE（Large System Extensions）原子指令或 LL/SC（Load-Linked /
Store-Conditional）实现 futex 所需的原子操作：

| 操作 | LSE 指令 | LL/SC 序列 |
|------|---------|-----------|
| CAS | `CAS` / `CASA` | `LDXR` + `STXR` 循环 |
| Exchange | `SWP` / `SWPA` | `LDXR` + `STXR` 循环 |
| Fetch-Add | `LDADD` | `LDXR` + `ADD` + `STXR` |

glibc 通过 IFUNC 在运行时选择最优实现（`HWCAP_ATOMICS` 检查）。

### 10.2 内存序

AArch64 是弱一致性（weakly ordered）架构，futex 操作中的内存序至关重要：

- `atomic_compare_and_exchange_bool_acq`: 使用 acquire 语义（`CASA`）
- `atomic_exchange_release`: 使用 release 语义（`SWPL`）
- `atomic_exchange_acquire`: 使用 acquire 语义（`SWPA`）

### 10.3 自旋 NOP

```c
/* AArch64 的 atomic_spin_nop() 使用 YIELD 指令 */
/* 提示处理器当前在自旋等待，允许核心降低功耗或让出资源给 SMT 同伴 */
```

### 10.4 无 AArch64 特化

Futex 的系统调用封装和三态锁协议在 glibc 中**没有 AArch64 特化实现**。
所有平台共享 `sysdeps/nptl/` 下的通用代码。架构差异仅体现在底层原子操作
指令的选择上。

---

## 11. Futex 使用全景表

| 同步原语 | Futex 字 | Wait 操作 | Wake 操作 | 快速路径 |
|----------|---------|----------|----------|---------|
| **lll_lock** | `*futex` (int) | `FUTEX_WAIT`, expected=2 | `FUTEX_WAKE`, n=1 | CAS(0→1) |
| **mutex (普通)** | `__lock` | `FUTEX_WAIT`, expected=2 | `FUTEX_WAKE`, n=1 | CAS(0→1) |
| **mutex (adaptive)** | `__lock` | 同上（自旋后） | 同上 | TRYLOCK 自旋循环 |
| **mutex (robust)** | `__lock` | `FUTEX_WAIT`, expected=oldval | `FUTEX_WAKE`, n=1 | CAS(0→tid) |
| **mutex (PI)** | `__lock` | `FUTEX_LOCK_PI` / `PI2` | `FUTEX_UNLOCK_PI` | CAS(0→tid) by kernel |
| **condvar** | `__g_signals[g]` | `FUTEX_WAIT_BITSET` | `FUTEX_WAKE`, n=1/INT_MAX | signals>0 消费 |
| **rwlock** | `__wrphase_futex` | `FUTEX_WAIT_BITSET` | `FUTEX_WAKE`, n=INT_MAX | atomic_add readers |
| **rwlock (写者)** | `__writers_futex` | `FUTEX_WAIT_BITSET` | `FUTEX_WAKE`, n=1 | CAS 获取写权限 |
| **barrier** | `current_round` | `FUTEX_WAIT` | `FUTEX_WAKE`, n=INT_MAX | — (无快速路径) |
| **barrier (重置)** | `in` | `FUTEX_WAIT` | `FUTEX_WAKE`, n=INT_MAX | — |
| **semaphore** | `data`/`value` | `FUTEX_WAIT` | `FUTEX_WAKE`, n=1 | 原子减>0 |
| **once** | `*once_control` | `FUTEX_WAIT` | `FUTEX_WAKE`, n=INT_MAX | load DONE 标志 |
| **join** | `pd->joinstate` | `FUTEX_WAIT_BITSET` | 内核 CLEARTID | load EXITED |

---

## 12. 源文件快速索引

| 文件 | 行号 | 内容 |
|------|------|------|
| lowlevellock-futex.h | 27-44 | futex 操作常量定义 |
| lowlevellock-futex.h | 46-54 | PRIVATE/SHARED 标志 + `__lll_private_flag` |
| lowlevellock-futex.h | 56-62 | `lll_futex_syscall` 宏 |
| lowlevellock.h | 26-61 | 三态锁协议文档注释 |
| lowlevellock.h | 67-70 | `lll_trylock` |
| lowlevellock.h | 93-107 | `lll_lock` 宏（CAS 快速路径 + 慢路径分派） |
| lowlevellock.h | 117-124 | `lll_cond_lock` 宏（条件变量特化） |
| lowlevellock.h | 144-159 | `lll_unlock` 宏（exchange + 条件 wake） |
| lowlevellock.c | 24-36 | `__lll_lock_wait_private` (exchange→2 + futex_wait 循环) |
| lowlevellock.c | 39-52 | `__lll_lock_wait` (共享版) |
| lowlevellock.c | 54-66 | `__lll_lock_wake_private` / `_wake` (futex_wake n=1) |
| futex-internal.h | 29-60 | futex 封装总体设计文档 |
| futex-internal.h | 62-70 | FUTEX_PRIVATE/SHARED 定义 |
| futex-internal.h | 123-144 | `futex_wait()` |
| futex-internal.h | 153-158 | `futex_wait_simple()` |
| futex-internal.h | 186-207 | `futex_wake()` |
| futex-internal.h | 242-268 | `futex_unlock_pi()` |
| futex-internal.h | 281-286 | `__futex_abstimed_wait_cancelable64` 声明 |
| futex-internal.h | 296-312 | `__futex_clocklock64` (三态锁+超时) |
| futex-internal.c | 66-121 | `__futex_abstimed_wait_common` (超时+时钟统一处理) |
| futex-internal.c | 123-131 | `__futex_abstimed_wait64` |
| futex-internal.c | 133-142 | `__futex_abstimed_wait_cancelable64` |
| futex-internal.c | 144-205 | `__futex_lock_pi64` (PI futex 锁定) |
| pthread_mutex_lock.c | 70-79 | 入口 — 类型分派 |
| pthread_mutex_lock.c | 83-89 | 普通 mutex — LLL_MUTEX_LOCK_OPTIMIZED |
| pthread_mutex_lock.c | 113-145 | 自适应 mutex — 指数退避自旋 |
| pthread_mutex_lock.c | 287-314 | Robust mutex — FUTEX_WAITERS + futex_wait |
| pthread_mutex_lock.c | 340-495 | PI mutex — __futex_lock_pi64 |
| pthread_mutex_unlock.c | 159-166 | Robust 解锁 — 条件 futex_wake |
| pthread_cond_wait.c | 421-422 | condvar wait — __futex_abstimed_wait_cancelable64 |
| pthread_cond_signal.c | 91-92 | condvar signal — futex_wake |
| pthread_rwlock_common.c | 454, 823 | rwlock — __futex_abstimed_wait64 on __wrphase_futex |
| pthread_rwlock_common.c | 724 | rwlock writer wait — __writers_futex |
| pthread_rwlock_common.c | 266, 411, 571, 577 | rwlock wake — futex_wake |
| pthread_barrier_wait.c | 167 | barrier — futex_wake(current_round, INT_MAX) |
| pthread_barrier_wait.c | 184 | barrier — futex_wait_simple(current_round) |
| pthread_barrier_wait.c | 215-216 | barrier reset — futex_wake(in, INT_MAX) |
| sem_post.c | 56 | semaphore 64b — futex_wake(&data, 1) |
| sem_post.c | 73 | semaphore 32b — futex_wake(&value, 1) |
| pthread_once.c | 105-106 | once — futex_wait_simple(once_control) |
| pthread_once.c | 127 | once — futex_wake(once_control, INT_MAX) |
| pthread_once.c | 38 | once cleanup — futex_wake(once_control, INT_MAX) |
| pthread_join_common.c | 46-70 | join — __futex_abstimed_wait on joinstate |
| dl-tls_init_tp.c | 95-96 | set_robust_list 注册 |
