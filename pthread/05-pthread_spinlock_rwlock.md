# pthread_spin 与 pthread_rwlock 深度分析

## 概述

自旋锁 (spinlock) 和读写锁 (rwlock) 是 pthread 互斥锁之外的两种重要同步原语。
自旋锁提供无系统调用的忙等待加锁；读写锁允许多读者并发，单写者独占。

---

## 第一部分: 自旋锁 (Spinlock)

---

## 1. 实现原理

glibc 自旋锁是 **纯用户态 CAS 忙等待**，不使用 futex，永不进入内核。

### 数据结构

```c
typedef volatile int pthread_spinlock_t;
// 状态: 0 = 未锁, 1 = 已锁
```

---

## 2. 各操作实现

### 2.1 pthread_spin_init

```c
// pthread_spin_init.c:25-27
int pthread_spin_init(pthread_spinlock_t *lock, int pshared) {
    atomic_store_relaxed(lock, 0);
    return 0;
}
```

- pshared 参数不影响实现（spinlock 本质上可跨进程使用）
- 仅将锁字设为 0

### 2.2 pthread_spin_lock

```c
// pthread_spin_lock.c:28-61
int pthread_spin_lock(pthread_spinlock_t *lock) {
    // 快速路径: 尝试 0→1
    if (atomic_exchange_acquire(lock, 1) == 0)
        return 0;

    // 慢速路径: 自旋
    do {
        // 先用 relaxed load 等待锁释放（避免总线流量）
        do {
            atomic_spin_nop();  // pause 指令
        } while (atomic_load_relaxed(lock) != 0);

        // 再尝试获取
    } while (atomic_compare_exchange_weak_acquire(lock, 0, 1) != 0);

    return 0;
}
```

**优化要点**:
- 内层循环用 **relaxed load** 监视，不产生总线写流量
- `atomic_spin_nop()` 插入 PAUSE 指令（x86）或 YIELD（ARM），减少流水线浪费
- 外层用 CAS 尝试获取

### 2.3 pthread_spin_trylock

```c
// pthread_spin_trylock.c:27-30
int pthread_spin_trylock(pthread_spinlock_t *lock) {
    return (atomic_exchange_acquire(lock, 1) == 0) ? 0 : EBUSY;
}
```

- 单次尝试，不自旋
- 失败立即返回 `EBUSY`

### 2.4 pthread_spin_unlock

```c
// pthread_spin_unlock.c:26-30
int pthread_spin_unlock(pthread_spinlock_t *lock) {
    atomic_store_release(lock, 0);
    return 0;
}
```

- release 语义保证临界区内存操作对后续 acquire 可见

### 2.5 pthread_spin_destroy

```c
// pthread_spin_destroy.c:22-25
int pthread_spin_destroy(pthread_spinlock_t *lock) {
    return 0;  // 无操作
}
```

---

## 3. 内存序语义

| 操作 | 内存序 | 说明 |
|------|--------|------|
| init | relaxed | 初始化时无需同步 |
| lock/trylock | **acquire** | 保证看到前一个持有者的写入 |
| unlock | **release** | 保证本次写入对下一个持有者可见 |

acquire-release 对构成 **happens-before** 关系。

---

## 4. 自旋锁 vs 互斥锁

| 特性 | 自旋锁 | 互斥锁 |
|------|--------|--------|
| 等待方式 | CPU 忙等待 | futex 内核阻塞 |
| 系统调用 | 无 | 竞争时有 |
| 大小 | 4 字节 | ~40 字节 |
| 递归 | 不支持 | 可选支持 |
| 优先级继承 | 不支持 | 可选支持 |
| 错误检测 | 无 | 可选 |
| 健壮性 | 无 | 可选 |
| 适用场景 | 极短临界区，无I/O | 通用 |
| 持有时是否可抢占 | 可抢占（浪费CPU） | 阻塞者让出CPU |

### 何时使用自旋锁

✅ 适用:
- 临界区极短（几十条指令以内）
- 竞争概率低
- 实时系统需要确定性延迟
- 中断上下文（内核态）

❌ 不适用:
- 临界区包含 I/O 或系统调用
- 竞争激烈
- 线程可能在临界区被抢占

---

## 第二部分: 读写锁 (Read-Write Lock)

---

## 5. 核心数据结构

```c
// sysdeps/nptl/bits/struct_rwlock.h:29-51
struct __pthread_rwlock_arch_t {
    unsigned int __readers;       // 主状态字（读者计数+标志位）
    unsigned int __writers;       // 写者相关状态
    unsigned int __wrphase_futex; // 阶段变化 futex
    unsigned int __writers_futex; // 写者等待 futex
    unsigned int __pad3;
    unsigned int __pad4;
    int __cur_writer;             // 当前写者 TID（用于 unlock 区分读/写）
    int __shared;                 // 共享标志
    unsigned int __flags;         // 偏好策略标志
    // ... padding
};
```

### __readers 字段位编码

```
┌────────────────────────────────────────────────┐
│ 读者计数 (高位)  │ WRHANDOVER │ WRLOCKED │ ... │
│ >> READER_SHIFT  │            │          │     │
└────────────────────────────────────────────────┘
```

| 标志 | 说明 |
|------|------|
| PTHREAD_RWLOCK_WRLOCKED | 写锁已持有 |
| PTHREAD_RWLOCK_WRHANDOVER | 写者间直接交接 |
| PTHREAD_RWLOCK_RWAITING | 有读者在等待 |
| PTHREAD_RWLOCK_FUTEX_USED | futex 可能有等待者 |

---

## 6. 读写偏好策略

| 策略 | 宏 | 行为 |
|------|-----|------|
| 读者优先（默认） | `PTHREAD_RWLOCK_PREFER_READER_NP` | 只要无写锁，新读者可立即进入 |
| 写者优先 | `PTHREAD_RWLOCK_PREFER_WRITER_NP` | 有写者等待时阻止新读者 |
| 写者优先（非递归） | `PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP` | 真正阻止新读者 |

### 读者优先的问题

```
读者1 持有 → 写者等待 → 读者2 加入 → 读者3 加入 → ...
                ↑
            写者可能永远等不到！（写者饥饿）
```

### 写者优先的解决

```
读者1 持有 → 写者等待 → 读者2 阻塞（看到有写者等待）
                              │
                              └── 读者1 释放 → 写者获得锁
```

---

## 7. 读锁流程

```
pthread_rwlock_rdlock(rwlock)
    │
    ├── 1. atomic_fetch_add(&__readers, 1 << READER_SHIFT)
    │       注册为读者
    │
    ├── 2. 检查: 是否有写锁或需要等待写者?
    │       ├── 无写锁 → 成功返回
    │       └── 有写锁或写者优先模式下有写者等待:
    │
    ├── 3. 设置 RWAITING 标志
    │
    ├── 4. futex_wait 等待阶段变化
    │       (在 __wrphase_futex 上等待)
    │
    └── 5. 被唤醒后重新检查，成功则返回
```

---

## 8. 写锁流程

```
pthread_rwlock_wrlock(rwlock)
    │
    ├── 1. 尝试 CAS: __readers 无读者无写者 → 设置 WRLOCKED
    │       ├── 成功 → __cur_writer = self; return 0
    │       └── 失败 → 进入等待
    │
    ├── 2. 注册为等待写者
    │
    ├── 3. 等待循环:
    │       ├── 检查是否有 WRHANDOVER（直接交接）
    │       │   └── 是 → CAS 获取，成功返回
    │       ├── 检查读者是否全部退出
    │       │   └── 是 → CAS 获取
    │       └── futex_wait 在 __writers_futex 上等待
    │
    └── 4. 设置 __cur_writer = self
```

---

## 9. 解锁流程

```
pthread_rwlock_unlock(rwlock)
    │
    ├── 判断: __cur_writer == self?
    │       ├── 是 → 写解锁路径
    │       └── 否 → 读解锁路径
    │
    ├── 【读解锁】:
    │   └── CAS: __readers -= (1 << READER_SHIFT)
    │       └── 若变为 0 且有写者等待 → futex_wake 写者
    │
    └── 【写解锁】:
        ├── 清除 __cur_writer
        ├── CAS: 清除 WRLOCKED
        │   ├── 若有读者等待 → futex_wake 读者 (INT_MAX)
        │   └── 若有写者等待 → WRHANDOVER 交接
        └── futex_wake 相应 futex word
```

---

## 10. 写者直接交接 (WRHANDOVER)

为避免写锁释放后被新读者抢占（写者饥饿），实现了直接交接:

```
写者 A 释放锁
    │
    ├── 检测到有写者 B 在等待
    │
    ├── 设置 WRHANDOVER 标志（不清除 WRLOCKED）
    │
    └── futex_wake 写者 B
            │
            └── 写者 B: CAS 清除 WRHANDOVER，获得锁
```

这保证了写者间的高效转移，无需经过"完全释放→重新竞争"的过程。

---

## 11. 三个 Futex Word

| futex word | 等待者 | 唤醒时机 |
|-----------|--------|----------|
| `__wrphase_futex` | 等待阶段变化的读者 | 写锁释放，进入读阶段 |
| `__writers_futex` | 等待的写者 | 所有读者退出 / WRHANDOVER |
| `__readers` | 特殊情况的读者等待 | 写者优先模式下读者等待 |

为什么需要独立的 futex word（而不直接在 `__readers` 上等待）:
- `__readers` 变化太频繁（每次读加锁/解锁都改变）
- 频繁变化导致 futex_wait 的预期值频繁不匹配
- 独立 futex word 只在状态转换时改变，减少虚假唤醒

---

## 12. 取消处理

> **POSIX 允许但不要求 rwlock 获取为取消点。glibc 的 rwlock 不支持取消。**

- 等待期间不检查取消请求
- 不设置取消清理函数
- 这简化了实现并避免了复杂的恢复路径

---

## 13. 超时等待

```c
// pthread_rwlock_timedrdlock / pthread_rwlock_timedwrlock
// pthread_rwlock_clockrdlock / pthread_rwlock_clockwrlock
```

- 超时直接传递给 futex_wait 的 timeout 参数
- 超时返回 `ETIMEDOUT`
- 支持 CLOCK_REALTIME 和 CLOCK_MONOTONIC

---

## 14. 内存序

| 操作 | 内存序 |
|------|--------|
| 读者加锁 (fetch_add) | acquire |
| 读者解锁 (CAS) | release |
| 写者加锁 (CAS) | acquire |
| 写者解锁 (CAS/store) | release |
| 阶段变化通知 | 带 fence |

---

## 15. 对比总结

| 特性 | spinlock | mutex | rwlock |
|------|----------|-------|--------|
| 并发读 | ❌ | ❌ | ✅ |
| 内核阻塞 | ❌ | ✅ | ✅ |
| 适用场景 | 极短临界区 | 通用互斥 | 读多写少 |
| 数据结构大小 | 4B | ~40B | ~56B |
| 取消支持 | ❌ | ❌ | ❌ |
| 超时支持 | ❌ | ✅ | ✅ |
| 递归 | ❌ | 可选 | ❌ |
| 进程共享 | ✅(天然) | 可选 | 可选 |

---

## 16. 源码位置速查

### 自旋锁

| 内容 | 文件:行号 |
|------|-----------|
| spin_init | pthread_spin_init.c:25-27 |
| spin_lock | pthread_spin_lock.c:24-63 |
| spin_trylock | pthread_spin_trylock.c:25-30 |
| spin_unlock | pthread_spin_unlock.c:24-30 |
| spin_destroy | pthread_spin_destroy.c:22-25 |

### 读写锁

| 内容 | 文件:行号 |
|------|-----------|
| 数据结构 | bits/struct_rwlock.h:29-51 |
| rwlock_init | pthread_rwlock_init.c:23-55 |
| 核心算法 | pthread_rwlock_common.c:30-944 |
| 偏好策略说明 | pthread_rwlock_common.c:30-58 |
| 读者计数编码 | pthread_rwlock_common.c:65-96 |
| futex 设计说明 | pthread_rwlock_common.c:97-168 |
| 读解锁 | pthread_rwlock_common.c:219-270 |
| 读加锁 | pthread_rwlock_common.c:296-349 |
| 写解锁 | pthread_rwlock_common.c:525-577 |
| 写加锁 | pthread_rwlock_common.c:616-693 |
| WRHANDOVER | pthread_rwlock_common.c:139-147, 830-859 |
| unlock 入口 | pthread_rwlock_unlock.c:29-43 |
