# POSIX 信号量与杂项 pthread API

## 概述

本文档分析以下组件：

1. **POSIX 信号量**（`sem_*`）：进程内/跨进程的计数信号量
2. **pthread_atfork**：fork 前后的回调注册
3. **pthread_setname_np / getname_np**：线程命名
4. **pthread_setaffinity_np / getaffinity_np**：CPU 亲和性

---

## 一、POSIX 信号量

### 内部数据结构

**源文件**: `sysdeps/nptl/internaltypes.h:161-187`

#### 64 位平台

```c
struct new_sem {
    uint64_t data;    // 低 32 位 = value（计数），高 32 位 = nwaiters
    int private;      // FUTEX_PRIVATE 或 FUTEX_SHARED
};
```

#### 32 位平台

```c
struct new_sem {
    unsigned int value;     // 信号量计数 + SEM_NWAITERS_MASK 位
    unsigned int nwaiters;  // 等待者计数
    int private;
};
```

### API 一览

| 函数 | 源文件 | 说明 |
|------|--------|------|
| `sem_init` | `nptl/sem_init.c:26-52` | 初始化无名信号量 |
| `sem_destroy` | `nptl/sem_destroy.c:24-30` | 销毁（当前为 no-op） |
| `sem_wait` | `nptl/sem_waitcommon.c:123-313` | 等待（P 操作） |
| `sem_post` | `nptl/sem_post.c:31-77` | 释放（V 操作） |
| `sem_timedwait` | `nptl/sem_timedwait.c:25-42` | 带超时等待 |
| `sem_trywait` | — | 非阻塞尝试 |
| `sem_getvalue` | `nptl/sem_getvalue.c:25-43` | 读取当前值 |
| `sem_open` | `sysdeps/pthread/sem_open.c:32-203` | 打开命名信号量 |
| `sem_unlink` | `sysdeps/pthread/sem_unlink.c:27-42` | 删除命名信号量 |

### sem_init

**源文件**: `nptl/sem_init.c:26-52`

```
sem_init(sem, pshared, value):
  if value > SEM_VALUE_MAX:
    return EINVAL
  sem->data = value  // 64位：低32位=value, 高32位=0(nwaiters)
  sem->private = pshared ? FUTEX_SHARED : FUTEX_PRIVATE
```

### sem_wait 算法

**源文件**: `nptl/sem_waitcommon.c:123-313`

```
┌─────────────────────────────────┐
│  快速路径 (sem_wait_fast)       │
│  relaxed load data              │
│  if value > 0:                  │
│    acquire CAS (value → value-1)│
│    return 0                     │
├─────────────────────────────────┤
│  慢路径                          │
│  atomic_add(nwaiters, 1)        │
│  loop:                          │
│    if value > 0:                │
│      CAS(value → value-1)      │
│      → 成功则 break             │
│    futex_wait(&data/&value, ..) │
│  atomic_sub(nwaiters, 1)        │
│  return 0                       │
└─────────────────────────────────┘
```

**关键设计**:
- 快速路径：无等待者时，单次 CAS 即可完成（无系统调用）
- 慢路径：先注册等待者计数，再 futex 等待
- 取消点：`sem_wait` 是 POSIX 取消点

### sem_post 算法

**源文件**: `nptl/sem_post.c:31-77`

```
sem_post(sem):
  release CAS: value → value + 1
  if nwaiters > 0 或 waiter_bit 设置:
    futex_wake(&data/&value, 1, private)  // 只唤醒一个
  return 0
```

- 只在有等待者时才执行 `futex_wake`（优化无竞争场景）
- 唤醒数量为 1（不是 INT_MAX），避免惊群

### sem_timedwait

**源文件**: `nptl/sem_timedwait.c:25-42`

- 验证超时参数有效性
- 先尝试快速路径
- 失败后进入带超时的 futex_wait（使用 `CLOCK_REALTIME`）
- 超时返回 `ETIMEDOUT`

### sem_getvalue

**源文件**: `nptl/sem_getvalue.c:25-43`

- relaxed load 读取当前计数
- 注释说明该操作非完全线性化（仅为快照）

### sem_destroy

**源文件**: `nptl/sem_destroy.c:24-30`

- 当前实现为 **no-op**（不做任何事）
- 调用者需确保无线程在等待

### 命名信号量

**源文件**: `sysdeps/pthread/sem_open.c:32-203`

```
sem_open(name, oflag, mode, value):
  打开/创建 /dev/shm/sem.<name> 文件
  mmap 为 sem_t*
  如果是新创建: sem_init(sem, 1, value)  // pshared=1
  返回 sem 指针
```

- 底层使用 POSIX 共享内存（`/dev/shm/`）
- 始终以 `FUTEX_SHARED` 模式初始化
- `sem_unlink` 即 `unlink("/dev/shm/sem.<name>")`

### 64 位 vs 32 位差异

| 特性 | 64 位 | 32 位 |
|------|-------|-------|
| value + nwaiters | 打包在一个 uint64_t | 分开两个 uint32_t |
| 原子操作 | 单次 64-bit CAS | 需要 waiter-bit 协调 |
| futex 地址 | `&data`（低 32 位） | `&value` |

---

## 二、pthread_atfork

### 注册机制

**源文件**: `sysdeps/pthread/pthread_atfork.c:43-57`

```c
int pthread_atfork(prepare, parent, child) {
    return __register_atfork(prepare, parent, child, __dso_handle);
}
```

**实际实现**: `posix/register-atfork.c:35-64`

```c
struct fork_handler {
    void (*prepare)(void);
    void (*parent)(void);
    void (*child)(void);
    void *dso_handle;
    uint64_t id;           // 注册顺序编号
};
```

- 注册在 `atfork_lock` 保护下追加到列表
- `dso_handle` 用于在动态库卸载时移除对应回调

### Fork 执行顺序

**源文件**: `posix/register-atfork.c:112-217`，`posix/fork.c:39-138`

```
fork() 执行流程:

  1. prepare 回调: 逆序执行（后注册的先调用）
     ↓
  2. 加锁内部资源: NSS / stdio / malloc
     ↓
  3. clone() 系统调用
     ↓
  ┌─── 父进程 ──────────────────┐  ┌─── 子进程 ──────────────────┐
  │ 4a. 释放内部锁              │  │ 4b. 重置内部锁              │
  │ 5a. parent 回调: 正序执行   │  │ 5b. child 回调: 正序执行    │
  └─────────────────────────────┘  └─────────────────────────────┘
```

### 安全性保证

- prepare 逆序：确保锁的获取顺序与注册顺序的反序一致（避免死锁）
- parent/child 正序：确保锁的释放顺序正确
- 回调执行期间释放 `atfork_lock`：允许回调内注册/注销新的 atfork handler
- fork 期间新注册的 handler 不会在当前 fork 中执行（通过 ID 比较跳过）

### glibc 内部 fork 锁

**源文件**: `posix/fork.c:39-138`

fork 时 glibc 自动加锁/解锁的内部资源：
- `_IO_list_lock`（stdio 缓冲区列表）
- malloc arenas 锁
- NSS 数据库锁

---

## 三、pthread_setname_np / pthread_getname_np

### 实现

**源文件**: `nptl/pthread_setname.c:30-63`，`nptl/pthread_getname.c:29-68`

```
pthread_setname_np(thread, name):
  if strlen(name) >= TASK_COMM_LEN(16):
    return ERANGE
  if thread == self:
    prctl(PR_SET_NAME, name)
  else:
    写入 /proc/self/task/<tid>/comm

pthread_getname_np(thread, buf, len):
  if len < TASK_COMM_LEN(16):
    return ERANGE
  if thread == self:
    prctl(PR_GET_NAME, buf)
  else:
    读取 /proc/self/task/<tid>/comm
```

### 限制

- 线程名最长 15 字节（加 `\0` = 16 = `TASK_COMM_LEN`）
- 设置他人线程名需要 `/proc` 文件系统可用
- `_np` 后缀表示非 POSIX 标准（GNU 扩展）

---

## 四、pthread_setaffinity_np / pthread_getaffinity_np

### 实现

**源文件**: `nptl/pthread_setaffinity.c:26-45`，`nptl/pthread_getaffinity.c:28-62`

```
pthread_setaffinity_np(thread, cpusetsize, cpuset):
  return sched_setaffinity(thread->tid, cpusetsize, cpuset)

pthread_getaffinity_np(thread, cpusetsize, cpuset):
  ret = sched_getaffinity(thread->tid, cpusetsize, cpuset)
  // 内核可能写入少于 cpusetsize 的字节
  // 尾部多余字节填零
  memset(remaining_bytes, 0, ...)
  return ret
```

### 使用说明

```c
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(0, &cpuset);
CPU_SET(2, &cpuset);
pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset);
```

- 直接封装 `sched_setaffinity` / `sched_getaffinity` 系统调用
- `cpusetsize` 必须至少能容纳系统 CPU 数的位图
- get 操作会将内核未写入的尾部字节清零（兼容处理）

---

## 五、性能特征对比

| 操作 | 快速路径 | 慢路径 |
|------|---------|--------|
| sem_wait（无竞争） | 1 次 CAS | — |
| sem_wait（有竞争） | — | futex_wait 系统调用 |
| sem_post（无等待者） | 1 次 CAS | — |
| sem_post（有等待者） | 1 次 CAS + futex_wake | — |
| setname（自己） | 1 次 prctl | — |
| setname（他人） | — | /proc 文件 I/O |
| setaffinity | — | 1 次系统调用 |

---

## 六、源文件速查

| 文件 | 内容 |
|------|------|
| `sysdeps/nptl/internaltypes.h:161-187` | `struct new_sem` |
| `nptl/sem_init.c:26-52` | 信号量初始化 |
| `nptl/sem_waitcommon.c:123-313` | wait 快慢路径 |
| `nptl/sem_post.c:31-77` | post + 唤醒逻辑 |
| `nptl/sem_timedwait.c:25-42` | 带超时等待 |
| `nptl/sem_getvalue.c:25-43` | 读取值 |
| `nptl/sem_destroy.c:24-30` | 销毁（no-op） |
| `sysdeps/pthread/sem_open.c:32-203` | 命名信号量 |
| `sysdeps/pthread/pthread_atfork.c:43-57` | atfork 入口 |
| `posix/register-atfork.c:35-217` | atfork 注册/执行 |
| `posix/fork.c:39-138` | fork 完整流程 |
| `nptl/pthread_setname.c:30-63` | 设置线程名 |
| `nptl/pthread_getname.c:29-68` | 获取线程名 |
| `nptl/pthread_setaffinity.c:26-45` | 设置 CPU 亲和性 |
| `nptl/pthread_getaffinity.c:28-62` | 获取 CPU 亲和性 |
