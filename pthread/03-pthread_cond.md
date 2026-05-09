# pthread_cond — 条件变量深度分析

## 概述

条件变量允许线程在某个条件不满足时挂起等待，并在条件可能满足时被唤醒。
glibc 的条件变量实现使用了精巧的 **双组 (two-group)** 序列号机制，
配合 futex 系统调用实现高效的等待/唤醒。

---

## 1. 内部数据结构

```c
// sysdeps/nptl/bits/thread-shared-types.h:94-104
struct __pthread_cond_s {
    __atomic_wide_counter __wseq;        // 等待者序列号（64位）
    __atomic_wide_counter __g1_start;    // G1 组起始序号
    unsigned int __g_size[2];            // 两个组的剩余等待者数
    unsigned int __g1_orig_size;         // G1 初始大小 + 内部锁位
    unsigned int __wrefs;                // 引用计数 + 标志位
    unsigned int __g_signals[2];         // 两个组的可消费信号数
};
```

### 字段详解

| 字段 | 说明 |
|------|------|
| `__wseq` | 全局递增序号，每个 waiter 分配唯一值。最低位标识当前 G2 索引 |
| `__g1_start` | 当前 G1 组的起始序号边界 |
| `__g_size[g]` | 组 g 中尚未收到 signal 的等待者数量 |
| `__g1_orig_size` | G1 关闭时的原始大小，含内部锁位 |
| `__wrefs` | bit0=共享标志, bit1=时钟标志, bit2=销毁中, bit3+=引用计数 |
| `__g_signals[g]` | 组 g 中可供消费的唤醒信号数量（futex word） |

---

## 2. 双组 (Two-Group) 机制

### 2.1 为什么需要两个组？

单纯的计数器方案存在 **丢失唤醒 (lost wakeup)** 问题:
- signal 在 waiter 进入 futex_wait 之前到达
- futex word 只有 32 位，无法编码所有状态

双组方案将等待者分为 **G1**（可被唤醒组）和 **G2**（新到达组）:

```
时间轴 →
                 ┌─── G1 (可被 signal 唤醒) ───┐
                 │  waiter_1  waiter_2  ...     │
                 └─────────────────────────────┘
                 ┌─── G2 (新来的 waiter) ──────┐
                 │  waiter_N  waiter_N+1  ...   │
                 └─────────────────────────────┘
```

### 2.2 组切换

当 G1 所有 waiter 都已被唤醒（`__g_size[g1] == 0`）:
1. 关闭旧 G1: `__g1_start += old_orig_size`
2. 翻转 `__wseq` 最低位（切换 G2 索引）
3. 旧 G2 变为新 G1
4. 新来的 waiter 进入新 G2

### 2.3 序号判定

每个 waiter 通过比较自己的序号和 `__g1_start` 判断所在组:
- `seq < __g1_start` → 在已关闭的旧 G1 中（直接返回）
- 否则根据 `__wseq` 最低位判断在 G1 还是 G2

---

## 3. pthread_cond_wait 流程

```
pthread_cond_wait(cond, mutex)
    │
    ├── 1. 获取序号: seq = atomic_fetch_add(&__wseq, 2)
    │       注册为等待者
    │
    ├── 2. 增加引用: __wrefs += 8
    │       防止 cond 被提前销毁
    │
    ├── 3. 释放互斥锁: mutex_unlock(mutex)
    │       ※ 此时其他线程可看到 waiter 已注册
    │
    ├── 4. 等待循环:
    │       while (true) {
    │           检查 seq 是否 < g1_start (组已关闭 → 退出)
    │           检查 __g_signals[g] > 0 (有信号 → CAS消费)
    │           futex_wait(&__g_signals[g], 0, timeout)
    │       }
    │
    ├── 5. 确认唤醒: __condvar_confirm_wakeup()
    │       减少引用计数
    │
    └── 6. 重新加锁: mutex_lock(mutex)
            返回
```

### 关键: "原子释放并等待" 语义

虽然 unlock 和 futex_wait 不是真正的单原子操作，但正确性由以下保证:
1. waiter 先注册序号（对 signaler 可见）
2. 再释放 mutex
3. signaler 持有 mutex 时能看到已注册的 waiter

因此 signal 不会丢失。

---

## 4. pthread_cond_signal 流程

```
pthread_cond_signal(cond)
    │
    ├── 1. 快速检查: if (__wrefs >> 3 == 0) return  // 无等待者
    │
    ├── 2. 获取内部锁: __condvar_acquire_lock()
    │
    ├── 3. 确定当前 G1
    │
    ├── 4. if (G1 有等待者) {
    │       __g_signals[g1]++     // 增加可消费信号
    │       __g_size[g1]--        // 减少剩余等待者
    │       futex_wake(&__g_signals[g1], 1)  // 唤醒一个
    │   }
    │
    ├── 5. if (G1 空了) {
    │       尝试 __condvar_switch_g1()  // 组切换
    │       对新 G1 发 signal
    │   }
    │
    └── 6. 释放内部锁
```

---

## 5. pthread_cond_broadcast 流程

```
pthread_cond_broadcast(cond)
    │
    ├── 1. 快速检查: 无等待者则返回
    │
    ├── 2. 获取内部锁
    │
    ├── 3. 唤醒当前 G1 全部:
    │       __g_signals[g1] += __g_size[g1]
    │       __g_size[g1] = 0
    │       futex_wake(&__g_signals[g1], INT_MAX)
    │
    ├── 4. 组切换: __condvar_switch_g1()
    │
    ├── 5. 唤醒新 G1 全部:
    │       __g_signals[new_g1] += __g_size[new_g1]
    │       __g_size[new_g1] = 0
    │       futex_wake(&__g_signals[new_g1], INT_MAX)
    │
    └── 6. 释放内部锁
```

---

## 6. 时钟选择

### 初始化时设置

```c
// pthread_cond_init.c:40-46
pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
// 内部: __wrefs |= __PTHREAD_COND_CLOCK_MONOTONIC_MASK (bit1)
```

### timedwait 时使用

```c
clockid_t clockid = (__wrefs & CLOCK_MONOTONIC_MASK)
                    ? CLOCK_MONOTONIC : CLOCK_REALTIME;
```

| 时钟 | 特点 | 适用场景 |
|------|------|----------|
| CLOCK_REALTIME | 可被用户/NTP 调整 | 需要绝对时间的场景 |
| CLOCK_MONOTONIC | 单调递增，不受调整 | 推荐用于超时等待 |

---

## 7. 取消处理 (Cancellation)

`pthread_cond_wait` 是一个 **取消点**。取消发生时的清理流程:

```
取消/超时发生
    │
    ├── __condvar_cancel_waiting(cond, seq, g)
    │       ├── 若在已关闭的 G1: 消耗了一个 signal → 需要补发
    │       ├── 若在 G2: 减少 __g_size[g]
    │       └── 溢出保护: 必要时 broadcast 清理
    │
    ├── __condvar_confirm_wakeup()
    │       减少 __wrefs 引用计数
    │
    ├── futex_wake(..., 1)  // 保守补发，防止唤醒丢失
    │
    └── 重新加锁 mutex（即使被取消也要重新获取）
```

**关键**: 被取消的线程必须重新获取 mutex，才能安全执行清理处理函数。

---

## 8. 进程共享 (pshared)

| 设置 | `__wrefs` bit0 | futex 模式 |
|------|----------------|-----------|
| PTHREAD_PROCESS_PRIVATE | 0 | FUTEX_PRIVATE |
| PTHREAD_PROCESS_SHARED | 1 | FUTEX_SHARED |

使用进程共享条件变量时:
- 条件变量必须在共享内存中（`mmap(MAP_SHARED)` 或 POSIX shm）
- 配套的 mutex 也必须是 `PTHREAD_PROCESS_SHARED`
- futex 使用共享模式（通过全局哈希表定位）

---

## 9. 虚假唤醒 (Spurious Wakeups)

POSIX 标准明确允许虚假唤醒。glibc 实现中:

- futex_wait 可能因信号、内核调度等原因返回
- waiter 被唤醒后必须重新检查条件
- 应用程序 **必须** 使用 while 循环:

```c
// 正确用法
pthread_mutex_lock(&mutex);
while (!condition)  // ← while，不是 if
    pthread_cond_wait(&cond, &mutex);
// 条件成立，处理...
pthread_mutex_unlock(&mutex);
```

---

## 10. pthread_cond_destroy 注意事项

销毁条件变量时:
1. 检查 `__wrefs` 引用计数
2. 如果还有 waiter，设置销毁标志并 futex_wait 等待所有 waiter 退出
3. 所有 waiter 离开后才能安全销毁

**规则**: 不能销毁还有线程在等待的条件变量（未定义行为）。

---

## 11. 与 futex 的关系

| 用途 | futex word | 操作 |
|------|-----------|------|
| waiter 阻塞 | `__g_signals[g]` | futex_wait |
| signal/broadcast 唤醒 | `__g_signals[g]` | futex_wake |
| destroy 等待 | `__wrefs` | futex_wait |

futex 是"睡眠/唤醒载体"，真正的正确性由序列号和双组机制保证。

---

## 12. 源码位置速查

| 内容 | 文件:行号 |
|------|-----------|
| struct __pthread_cond_s | bits/thread-shared-types.h:94-104 |
| cond_init | pthread_cond_init.c:26-49 |
| cond_wait 主循环 | pthread_cond_wait.c:329-445 |
| cond_wait 算法说明 | pthread_cond_wait.c:173-328 |
| cond_signal | pthread_cond_signal.c:37-95 |
| cond_broadcast | pthread_cond_broadcast.c:37-87 |
| cond_destroy | pthread_cond_destroy.c:41-58 |
| 组切换逻辑 | pthread_cond_common.c:192-258 |
| 取消处理 | pthread_cond_wait.c:77-171 |
