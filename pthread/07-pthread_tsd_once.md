# pthread TSD 与 pthread_once — 线程特定数据和一次性初始化

## 概述

本文档分析两个相关但独立的机制：

1. **TSD（Thread-Specific Data）**：每个线程拥有私有的键值存储，支持析构函数
2. **pthread_once**：保证初始化函数在所有线程中恰好执行一次

两者都是无锁设计，依赖原子操作和序列号实现并发安全。

---

## 一、线程特定数据（TSD）

### 全局键表

**源文件**: `nptl/pthread_keys.c:21-23`

```c
struct pthread_key_struct __pthread_keys[PTHREAD_KEYS_MAX];
```

**键结构**: `sysdeps/nptl/internaltypes.h:132-153`

```c
struct pthread_key_struct {
    uintptr_t seq;            // 序列号（偶数=未使用，奇数=已分配）
    void (*destr)(void *);    // 析构函数
};
```

- `PTHREAD_KEYS_MAX = 1024`（`sysdeps/unix/sysv/linux/bits/local_lim.h`）
- `KEY_UNUSED`：seq 为偶数表示槽位空闲
- `KEY_USABLE`：防止 seq 溢出后重用的哨兵值

### 键的生命周期

```
         ┌──────────────────────────────────────┐
         │      __pthread_keys[0..1023]          │
         │  ┌─────┬─────┬─────┬───┬─────┐      │
         │  │ seq │ seq │ seq │...│ seq │      │
         │  │destr│destr│destr│   │destr│      │
         │  └─────┴─────┴─────┴───┴─────┘      │
         └──────────────────────────────────────┘
              ▲                        ▲
    key_create: CAS seq→seq+1     key_delete: CAS seq→seq+1
    (偶→奇=分配)                  (奇→偶=释放)
```

### pthread_key_create

**源文件**: `nptl/pthread_key_create.c:24-47`

```
遍历 __pthread_keys[0..1023]:
  读取 seq
  if seq 为偶数 (KEY_UNUSED):
    CAS(seq → seq+1)  // 偶→奇 = 分配成功
    if 成功:
      存储 destructor
      return 索引作为 key
return EAGAIN  // 所有槽位已满
```

- **无锁**: 使用 CAS 避免互斥锁
- **ABA 防护**: seq 单调递增，偶数/奇数交替标记使用状态

### pthread_key_delete

**源文件**: `nptl/pthread_key_delete.c:23-39`

```
CAS(__pthread_keys[key].seq → seq+1)  // 奇→偶 = 释放
```

- 不会清理各线程的 `specific` 数据（延迟到线程退出或 getspecific 时检测）

### 每线程存储结构

**源文件**: `nptl/descr.h`（struct pthread 成员）

```
struct pthread {
    ...
    struct pthread_key_data specific_1stblock[32];  // 前32个键的快速路径
    struct pthread_key_data *specific[32];          // 二级指针数组
    bool specific_used;                             // 是否使用过 TSD
    ...
};
```

**两级索引方案**:

```
key 0~31:   直接 → specific_1stblock[key]
key 32~1023:
  idx1st = key / 32    (0~31)
  idx2nd = key % 32    (0~31)
  → specific[idx1st][idx2nd]   (按需分配第二级块)
```

### pthread_setspecific

**源文件**: `nptl/pthread_setspecific.c:34-88`

```
验证 key < PTHREAD_KEYS_MAX
验证 __pthread_keys[key].seq 为奇数（已分配）

if key < 32:
  specific_1stblock[key] = {seq, data}
else:
  idx1st = key / 32
  idx2nd = key % 32
  if specific[idx1st] == NULL:
    分配 32 个条目的块（calloc）
  specific[idx1st][idx2nd] = {seq, data}

specific_used = true
```

### pthread_getspecific

**源文件**: `nptl/pthread_getspecific.c:27-63`

```
定位 entry（一级或二级）
if entry.seq == __pthread_keys[key].seq:
  return entry.data    // seq 匹配，数据有效
else:
  entry.data = NULL    // seq 不匹配，键已被 delete/recreate
  return NULL
```

- **seq 校验**是检测过期数据的核心机制

### 析构函数调用

**源文件**: `nptl/nptl_deallocate_tsd.c:33-86`

线程退出时执行：

```
for round = 0 to PTHREAD_DESTRUCTOR_ITERATIONS-1:
  found_nonzero = false
  for idx = 0 to PTHREAD_KEYS_MAX-1:
    entry = 定位 specific 条目
    if entry.data != NULL && entry.seq == __pthread_keys[idx].seq:
      destr = __pthread_keys[idx].destr
      entry.data = NULL
      if destr != NULL:
        destr(data)
        found_nonzero = true
  if !found_nonzero:
    break
```

- 最多执行 `PTHREAD_DESTRUCTOR_ITERATIONS`（通常为 4）轮
- 每轮按键索引升序扫描
- 析构函数中可以设置其他键的数据（因此需要多轮）
- 只有 seq 匹配的条目才调用析构函数

### 线程安全总结

| 操作 | 同步机制 | 说明 |
|------|----------|------|
| key_create | 无锁 CAS | seq 偶→奇 |
| key_delete | 无锁 CAS | seq 奇→偶 |
| setspecific | 线程本地写 | 只写自己的 specific |
| getspecific | 线程本地读 + seq 校验 | 检测过期 |
| 析构 | 线程退出时顺序执行 | 无并发 |

---

## 二、pthread_once

### 状态机

**源文件**: `nptl/pthread_once.c:42-61`（注释描述），`65-143`（实现）

```
┌───────────┐    CAS 成功     ┌──────────────────────┐
│  0 (初始) │───────────────→│ fork_gen | INPROGRESS │
└───────────┘                 └──────────────────────┘
      ▲                              │
      │ 取消/异常                      │ init_routine() 成功
      │ (cleanup handler)             ▼
      └─────────────────────── ┌───────────┐
                               │   DONE    │
                               └───────────┘
```

**状态编码**:
- `0`：未开始
- `__fork_generation | __PTHREAD_ONCE_INPROGRESS`：正在执行
- `__PTHREAD_ONCE_DONE`：已完成

### 执行流程

**源文件**: `nptl/pthread_once.c:65-143`（`__pthread_once_slow` + `___pthread_once`）

```
pthread_once(once_control, init_routine):
  // 快速路径
  if *once_control == DONE:
    return 0   // acquire 语义确保可见性

  // 慢路径
  newval = __fork_generation | INPROGRESS
  while true:
    oldval = atomic_load(once_control)
    if oldval == DONE:
      return 0
    if CAS(once_control, oldval → newval) 成功:
      break
    // CAS 失败或仍在进行中 → futex_wait
    futex_wait(once_control, oldval, FUTEX_PRIVATE)

  // 赢得执行权
  push cleanup_handler   // 处理取消/异常
  init_routine()
  pop cleanup_handler

  // 标记完成
  atomic_store_release(once_control, DONE)
  futex_wake(once_control, INT_MAX, FUTEX_PRIVATE)
  return 0
```

### 取消与异常安全

**源文件**: `nptl/pthread_once.c:27-39`（`clear_once_control`），`114-118`（push/pop）

```c
static void once_cleanup(void *arg) {
    pthread_once_t *once_control = arg;
    atomic_store_relaxed(once_control, 0);  // 重置为初始状态
    futex_wake(once_control, INT_MAX, FUTEX_PRIVATE);  // 唤醒等待者
}
```

- 如果执行线程被取消或抛出异常，cleanup handler 将状态重置为 0
- 其他等待线程被唤醒后可以重新竞争执行
- 测试用例: `nptl/tst-once5.cc:39-76`

### Fork 安全

**源文件**: `sysdeps/nptl/fork.h:31-36`

```
每次 fork() 后子进程中:
  __fork_generation += __PTHREAD_ONCE_FORK_GEN_INCR  // 递增 generation

判断逻辑:
  if (oldval & ~__PTHREAD_ONCE_DONE) != (__fork_generation | INPROGRESS):
    // generation 不匹配 → 视为中断，可重新执行
```

- fork 后子进程中，父进程正在执行但未完成的 once 会被自动重置
- 子进程可以安全地重新执行 init_routine

### Futex 使用

| 场景 | 操作 |
|------|------|
| 等待执行完成 | `futex_wait(once_control, oldval, PRIVATE)` |
| 执行完成唤醒 | `futex_wake(once_control, INT_MAX, PRIVATE)` |
| 取消后唤醒 | `futex_wake(once_control, INT_MAX, PRIVATE)` |

> 注意：pthread_once 仅支持 `FUTEX_PRIVATE`，不支持跨进程

---

## 三、性能特征

### TSD

- **getspecific**: 前 32 个键只需一次数组访问 + seq 比较（极快）
- **setspecific**: 首次使用高键号时需要 malloc（之后为 O(1)）
- **create/delete**: 线性扫描 1024 个槽位（最坏情况）

### pthread_once

- **已完成时**: 单次 acquire load（与 volatile 读取相当）
- **首次执行**: 一次 CAS + 函数调用 + release store + futex_wake
- **等待者**: 一次 futex_wait 系统调用

---

## 四、源文件速查

| 文件 | 内容 |
|------|------|
| `nptl/pthread_keys.c:21-23` | 全局键表定义 |
| `sysdeps/nptl/internaltypes.h:132-153` | `struct pthread_key_struct` |
| `nptl/pthread_key_create.c:24-47` | 键分配 |
| `nptl/pthread_key_delete.c:23-39` | 键释放 |
| `nptl/pthread_setspecific.c:34-88` | 设置线程数据 |
| `nptl/pthread_getspecific.c:27-63` | 获取线程数据 |
| `nptl/nptl_deallocate_tsd.c:33-86` | 析构函数调用 |
| `nptl/descr.h` | `struct pthread` 中 specific 字段 |
| `nptl/pthread_once.c:27-143` | once 完整实现（cleanup+slow+fast） |
| `sysdeps/nptl/fork.h:31-36` | fork generation 机制 |
