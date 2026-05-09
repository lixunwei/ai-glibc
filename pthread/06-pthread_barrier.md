# pthread_barrier — 屏障同步机制

## 概述

屏障（barrier）是一种集合同步原语：当 `count` 个线程都到达屏障时，它们同时被释放。glibc NPTL 的实现基于 **轮次（round）** 模型和 futex，支持进程共享、即时重用，且不需要额外的内部互斥锁。

## 数据结构

**源文件**: `sysdeps/nptl/internaltypes.h:110-119`

```c
struct pthread_barrier {
    unsigned int in;             // 已进入的线程计数
    unsigned int current_round;  // 最近完成的轮次位置
    unsigned int count;          // 每轮所需参与者数
    int shared;                  // FUTEX_PRIVATE 或 FUTEX_SHARED
    unsigned int out;            // 已确认退出的线程计数
};
```

### 字段语义

| 字段 | 作用 | 原子序 |
|------|------|--------|
| `in` | 累加进入计数，每个线程获得唯一位置 | acq_rel fetch_add |
| `current_round` | 标记已完成轮次的边界位置 | CAS (acq_rel) |
| `count` | 不变量，初始化后只读 | — |
| `shared` | futex 模式标志 | — |
| `out` | 跟踪退出线程数，用于重置/销毁 | release fetch_add |

## API 一览

| 函数 | 源文件 | 说明 |
|------|--------|------|
| `pthread_barrier_init` | `nptl/pthread_barrier_init.c:30-61` | 初始化屏障 |
| `pthread_barrier_wait` | `nptl/pthread_barrier_wait.c:94-221` | 等待所有参与者到达 |
| `pthread_barrier_destroy` | `nptl/pthread_barrier_destroy.c:24-59` | 销毁屏障 |

## 初始化

**源文件**: `nptl/pthread_barrier_init.c:30-61`

```
pthread_barrier_init(barrier, attr, count):
  if count == 0 || count >= BARRIER_IN_THRESHOLD:
    return EINVAL
  barrier->in = 0
  barrier->out = 0
  barrier->count = count
  barrier->current_round = 0
  barrier->shared = (pshared == PRIVATE) ? FUTEX_PRIVATE : FUTEX_SHARED
```

- `BARRIER_IN_THRESHOLD` 防止 `in` 计数器溢出

## 等待算法（核心）

**源文件**: `nptl/pthread_barrier_wait.c:94-221`

### 算法流程

```
┌─────────────────────────────────────────────────────────┐
│  Thread enters: i = atomic_fetch_add(&in, 1) + 1        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌── i > max_in_before_reset? ──┐                       │
│  │ YES: 等待 futex(in) 直到    │                       │
│  │      重置完成，然后重试      │                       │
│  └──────────────────────────────┘                       │
│                                                         │
│  读取 cr = current_round                                │
│                                                         │
│  ┌── cr + count <= i ? ─────────┐                       │
│  │ YES: 本线程可推进轮次         │                       │
│  │   newcr = i - (i % count)    │                       │
│  │   CAS(current_round, cr→newcr)│                      │
│  │   唤醒所有等待者              │                       │
│  └──────────────────────────────┘                       │
│                                                         │
│  ┌── i <= cr ? ─────────────────┐                       │
│  │ YES: 本轮已完成，直接离开     │                       │
│  │ NO:  futex_wait(current_round)│                      │
│  └──────────────────────────────┘                       │
│                                                         │
│  acquire fence                                          │
│  atomic_fetch_add(&out, 1)                              │
│                                                         │
│  ┌── out == max_in_before_reset? ┐                      │
│  │ YES (最后退出者): 执行重置    │                       │
│  │   current_round = 0           │                      │
│  │   out = 0                     │                      │
│  │   in = 0                      │                      │
│  │   futex_wake(&in, INT_MAX)    │                      │
│  └───────────────────────────────┘                      │
│                                                         │
│  return (i % count == 0) ?                              │
│         PTHREAD_BARRIER_SERIAL_THREAD : 0               │
└─────────────────────────────────────────────────────────┘
```

### 关键细节

1. **位置编号**: 每个线程通过 `fetch_add(&in, 1)` 获取唯一递增位置 `i`
2. **轮次推进**: 当某线程的 `i` 使得 `cr + count <= i` 时，该线程负责推进 `current_round`
3. **串行线程**: 位置是 `count` 整数倍的线程返回 `PTHREAD_BARRIER_SERIAL_THREAD`
4. **溢出保护**: `max_in_before_reset = threshold - threshold % count`，超出此范围的线程等待重置

### Futex 使用

| 等待条件 | futex 地址 | 唤醒时机 |
|----------|-----------|----------|
| 等待轮次完成 | `&current_round` | 轮次推进者唤醒 (`INT_MAX`) |
| 等待重置完成 | `&in` | 最后退出者重置后唤醒 (`INT_MAX`) |

## 重用语义

屏障天然可重用——**无需再次调用 init**：

1. 一轮完成后线程立即可以再次调用 `barrier_wait`
2. `in` 持续累加，轮次靠整除判断
3. 当 `in` 接近阈值时，最后一个退出的线程执行**重置**：
   - 将 `current_round`、`out`、`in` 全部归零
   - 唤醒所有因重置而阻塞的后来者
4. 重置期间新进入的线程在 `futex(&in)` 上等待

这种设计避免了传统 ABA 问题，同时支持无缝连续使用。

## 销毁

**源文件**: `nptl/pthread_barrier_destroy.c:24-59`

```
pthread_barrier_destroy(barrier):
  计算 max_in_before_reset
  人为推进 out 以触发重置路径
  如果还有线程未退出，等待 futex(&in) 直到重置完成
  acquire fence 确保销毁发生在所有使用之后
```

- 调用者必须保证没有新线程会再次调用 `barrier_wait`
- destroy 利用与正常退出相同的重置机制，保证内存安全

## 进程共享支持

- `pthread_barrierattr_setpshared(PTHREAD_PROCESS_SHARED)` 设置跨进程模式
- 初始化时存入 `shared = FUTEX_SHARED`
- 所有 futex 操作传递此标志，内核据此决定是否跨进程哈希

## 与其他实现的对比

| 特性 | glibc NPTL | FreeBSD | musl |
|------|-----------|---------|------|
| 无锁 | ✅ 纯原子操作 | ❌ 使用 mutex | ✅ |
| 即时重用 | ✅ 轮次模型 | ✅ generation | ✅ |
| 溢出保护 | ✅ 阈值重置 | ❌ | ❌ |
| 进程共享 | ✅ | ✅ | ✅ |

## 源文件速查

| 文件 | 内容 |
|------|------|
| `sysdeps/nptl/internaltypes.h:110-119` | `struct pthread_barrier` 定义 |
| `nptl/pthread_barrier_init.c:30-61` | 初始化 |
| `nptl/pthread_barrier_wait.c:94-221` | 等待算法核心 |
| `nptl/pthread_barrier_destroy.c:24-59` | 销毁 |
| `nptl/pthread_barrierattr.c` | 属性操作 |
