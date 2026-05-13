# pthread_mutex — 互斥锁深度分析

## 概述

pthread 互斥锁是最常用的线程同步原语，glibc 基于 Linux futex 系统调用
实现了高效的用户态快速路径和内核态慢速路径。支持普通、递归、检错、
自适应四种类型，以及健壮锁和优先级继承等高级特性。

---

## 1. 互斥锁类型

| 类型 | 宏 | 同线程重复加锁 | 非持有者解锁 | 适用场景 |
|------|-----|---------------|-------------|---------|
| NORMAL | `PTHREAD_MUTEX_NORMAL` | **死锁** | 未定义行为 | 默认，最快 |
| RECURSIVE | `PTHREAD_MUTEX_RECURSIVE` | 计数+1，正常 | `EPERM` | 递归函数 |
| ERRORCHECK | `PTHREAD_MUTEX_ERRORCHECK` | 返回 `EDEADLK` | 返回 `EPERM` | 调试 |
| ADAPTIVE | `PTHREAD_MUTEX_ADAPTIVE_NP` | **死锁** | 未定义行为 | 短临界区 |

---

## 2. 核心数据结构

```c
// 简化表示
typedef struct {
    int __lock;           // 锁状态: 0=空闲, 1=持有无等待, >1=持有有等待
    unsigned int __count; // 递归计数（仅 RECURSIVE）
    int __owner;          // 持有者 TID
    int __kind;           // 类型 + 标志位
    // ... 对齐填充
} pthread_mutex_t;
```

### __kind 标志位组合

```
__kind = base_type | ROBUST_BIT | PSHARED_BIT | PI_BIT | PP_BIT
```

---

## 3. Futex 锁状态机

### 3.1 锁状态（`__lock` 字段）

```
┌─────┐        CAS 0→1        ┌─────────────┐
│  0  │ ──────────────────────→│      1      │
│空闲 │                        │ 持有,无等待者│
└─────┘                        └──────┬──────┘
   ↑                                  │
   │     CAS/atomic_store 0           │ 竞争: 有其他线程尝试加锁
   │                                  ↓
   │                           ┌─────────────┐
   └───── futex_wake(1) ──────│     >1      │
                               │ 持有,有等待者│
                               └─────────────┘
                                      ↑↓
                              futex_wait 阻塞
```

### 3.2 快速路径 vs 慢速路径

**快速路径** (无竞争):
```c
// CAS: 0 → 1，成功则加锁完成
if (atomic_compare_exchange_weak_acquire(&mutex->__lock, 0, 1))
    return 0;  // 加锁成功，无系统调用
```

**慢速路径** (有竞争):
```c
// 将状态提升为 >1，表示有等待者
int oldval = atomic_exchange(&mutex->__lock, 2);
while (oldval != 0) {
    futex_wait(&mutex->__lock, 2);  // 内核阻塞
    oldval = atomic_exchange(&mutex->__lock, 2);
}
```

**解锁**:
```c
int prev = atomic_exchange(&mutex->__lock, 0);
if (prev > 1)  // 有等待者
    futex_wake(&mutex->__lock, 1);  // 唤醒一个
```

---

## 4. 各类型加锁/解锁详细流程

### 4.1 NORMAL 类型

**lock** (pthread_mutex_lock.c:83-89):
1. 单线程优化: 若进程只有一个线程，跳过原子操作
2. `lll_lock(&__lock)`: CAS 快速路径 → futex 慢速路径
3. 设置 `__owner = 当前 TID`

**unlock** (pthread_mutex_unlock.c:55-71):
1. 清除 `__owner`
2. `lll_unlock(&__lock)`: 若有等待者则 `futex_wake(1)`

### 4.2 RECURSIVE 类型

**lock** (pthread_mutex_lock.c:90-112):
```
if (__owner == self) {
    if (__count == MAX) return EAGAIN;
    __count++;
    return 0;
}
// 否则同 NORMAL
lll_lock(&__lock);
__owner = self;
__count = 0;
```

**unlock** (pthread_mutex_unlock.c:72-81):
```
if (__count > 0) {
    __count--;
    return 0;
}
// count==0，真正释放
__owner = 0;
lll_unlock(&__lock);
```

### 4.3 ERRORCHECK 类型

**lock** (pthread_mutex_lock.c:148-155):
```
if (__owner == self)
    return EDEADLK;  // 检测到死锁
// 否则同 NORMAL
```

**unlock** (pthread_mutex_unlock.c:85-93):
```
if (__owner != self)
    return EPERM;  // 非持有者不能解锁
// 否则正常释放
```

### 4.4 ADAPTIVE 类型

**lock** (pthread_mutex_lock.c:113-145):
```
if (trylock() == 0) return 0;  // 快速尝试

// 自旋阶段：指数退避
int spins = 0;
while (spins++ < max_adaptive_count) {
    atomic_spin_nop();
    if (trylock() == 0) return 0;
}

// 自旋失败，进入 futex 等待
lll_lock(&__lock);
```

特点: 短临界区下避免系统调用开销，退避后才进入内核。

---

## 5. 健壮互斥锁 (Robust Mutex)

### 5.1 问题场景

持有互斥锁的线程意外死亡（如 `SIGKILL`），其他等待线程将永远阻塞。

### 5.2 机制

- `__lock` 字段直接存储持有者 **TID**
- 内核通过 `set_robust_list()` 维护每线程的健壮锁列表
- 线程死亡时内核将 `FUTEX_OWNER_DIED` 位写入锁
- 下一个获取者收到 `EOWNERDEAD` 错误

### 5.3 流程

```
线程 A 持有锁 → 线程 A 被 kill
                    │
                    ▼ 内核设置 FUTEX_OWNER_DIED
                    │
线程 B 尝试加锁 ←──┘
    │
    ├── 返回 EOWNERDEAD（锁状态不一致）
    │
    ├── 线程 B 修复共享数据
    │
    └── pthread_mutex_consistent(&mutex)  ← 标记已修复
        然后正常使用
```

### 5.4 未修复就解锁

如果收到 `EOWNERDEAD` 后不调用 `pthread_mutex_consistent()` 就解锁:
- 锁变为 `PTHREAD_MUTEX_NOTRECOVERABLE`
- 后续所有加锁尝试返回 `ENOTRECOVERABLE`
- 必须 `pthread_mutex_destroy()` + `pthread_mutex_init()` 重建

---

## 6. 优先级继承互斥锁 (PI Mutex)

### 6.1 问题: 优先级反转

```
高优先级线程 H ──→ 等待锁（被 L 持有）
中优先级线程 M ──→ 抢占 L 的 CPU
低优先级线程 L ──→ 持有锁但无法运行
```

结果: H 被 M 间接阻塞（优先级反转）。

### 6.2 解决: 优先级继承

```
L 持有锁 → H 等待 → 内核提升 L 的优先级到 H 的级别
                     → L 不会被 M 抢占
                     → L 释放锁后优先级恢复
```

### 6.3 实现

- 初始化时检测内核 PI futex 支持
- 加锁: 用户态 CAS 尝试 → 失败调用 `futex_lock_pi`
- 解锁: CAS 尝试清零 → 若有等待者调用 `futex_unlock_pi`
- 完全由内核维护优先级继承链

### 6.4 API

```c
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
```

---

## 7. 优先级天花板互斥锁 (PP Mutex)

### 7.1 机制

- 锁有固定的优先级天花板值
- 任何持有锁的线程优先级被提升到天花板值
- 若线程优先级高于天花板，加锁失败 (`EINVAL`)

### 7.2 API

```c
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_PROTECT);
pthread_mutexattr_setprioceiling(&attr, priority);
```

---

## 8. 属性 (pthread_mutexattr_t)

| 属性 | 设置函数 | 选项 |
|------|----------|------|
| 类型 | `pthread_mutexattr_settype` | NORMAL/RECURSIVE/ERRORCHECK/ADAPTIVE |
| 进程共享 | `pthread_mutexattr_setpshared` | PRIVATE/SHARED |
| 协议 | `pthread_mutexattr_setprotocol` | NONE/INHERIT/PROTECT |
| 健壮性 | `pthread_mutexattr_setrobust` | STALLED/ROBUST |
| 天花板 | `pthread_mutexattr_setprioceiling` | 优先级值 |

### 组合规则

- Robust + PI 可以同时使用
- Robust + PP 不能同时使用（某些实现限制）
- PSHARED 要求锁在共享内存中

---

## 9. 私有 vs 共享 Futex

| 模式 | futex 标志 | 场景 |
|------|-----------|------|
| Private | `FUTEX_PRIVATE_FLAG` | 同进程内线程（默认） |
| Shared | 0 | 跨进程（共享内存中的锁） |

Private futex 性能更高（内核跳过全局哈希表查找）。

---

## 10. 源码位置速查

| 内容 | 文件:行号 |
|------|-----------|
| mutex_init | pthread_mutex_init.c:50-152 |
| mutex_lock (NORMAL) | pthread_mutex_lock.c:83-89 |
| mutex_lock (RECURSIVE) | pthread_mutex_lock.c:90-112 |
| mutex_lock (ADAPTIVE) | pthread_mutex_lock.c:113-145 |
| mutex_lock (ERRORCHECK) | pthread_mutex_lock.c:148-155 |
| mutex_lock (ROBUST) | pthread_mutex_lock.c:176-338 |
| mutex_lock (PI) | pthread_mutex_lock.c:344-496 |
| mutex_unlock | pthread_mutex_unlock.c:47-352 |
| mutex_trylock | pthread_mutex_trylock.c:25-443 |
| mutex_timedlock | pthread_mutex_timedlock.c:31-570 |
| lowlevellock 状态 | sysdeps/nptl/lowlevellock.h:26-45 |
| lll_lock 实现 | sysdeps/nptl/lowlevellock.h:64-93 |
| futex 慢路径 | nptl/lowlevellock.c:24-50 |
