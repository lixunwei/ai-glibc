# pthread_create — 线程创建深度分析

## 概述

`pthread_create` 是 POSIX 线程创建的核心接口，其内部流程涉及栈分配、
线程控制块（TCB）初始化、clone 系统调用和子线程启动等多个阶段。

---

## 1. 函数签名

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

---

## 2. 整体流程

```
pthread_create (pthread_create.c:649)
    │
    ├── 1. late_init() — 首次调用时初始化
    │       安装 SIGSETXID 处理器，解除信号阻塞
    │
    ├── 2. 获取线程属性
    │       attr == NULL → 使用默认属性
    │
    ├── 3. allocate_stack() — 分配栈和 TCB
    │       ├── 尝试复用缓存栈 (get_cached_stack)
    │       └── mmap 新栈 + 设置 guard page
    │
    ├── 4. 初始化 struct pthread (TCB)
    │       设置 start_routine, arg, flags, schedparam...
    │
    ├── 5. 准备信号掩码
    │       阻塞所有信号，写入 pd->sigmask
    │
    ├── 6. create_thread() — 执行 clone 系统调用
    │       __clone_internal(&args, &start_thread, pd)
    │
    ├── 7. 恢复父线程信号掩码
    │
    └── 8. 错误处理 / 成功返回
```

---

## 3. 栈分配 (allocatestack.c)

### 3.1 栈布局

```
低地址                                              高地址
┌──────────┬─────────────────────────┬────────────────┐
│ Guard    │      可用栈空间          │ struct pthread │
│ Page(s)  │      (用户代码使用)      │ (TCB + TLS)   │
│ PROT_NONE│      ↑ 栈增长方向        │               │
└──────────┴─────────────────────────┴────────────────┘
│←─────── stackblock_size ──────────────────────────→│
```

### 3.2 分配策略

**优先复用缓存栈**:
1. 从 `dl_stack_cache` 链表查找大小匹配的空闲栈
2. 重置 TCB/TLS 状态
3. 调用 `_dl_allocate_tls_init()` 重新初始化 TLS

**新建栈**:
1. `mmap(MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK)` 分配整块内存
2. 设置 guard page:
   - 优先使用 `madvise(MADV_GUARD_INSTALL)`（Linux 6.x+）
   - 回退到 `mprotect(PROT_NONE)`
3. `struct pthread` 放在映射的**高地址端**（与 TLS 静态区紧邻）
4. 将栈块加入 `dl_stack_used` 链表

### 3.3 用户自定义栈

如果通过 `pthread_attr_setstack()` 指定了栈:
- 不分配 guard page
- 按 TLS 对齐要求计算 `pd` 位置
- 加入 `dl_stack_user` 链表（退出时不回收）

### 3.4 栈大小

- 默认栈大小: 由 `__default_pthread_attr` 决定（通常 8MB）
- 最小栈: `PTHREAD_STACK_MIN`（通常 16KB）
- 实际大小会加上 TLS 静态区 + guard 后对齐

---

## 4. TCB 初始化 (struct pthread)

### 4.1 关键字段

| 字段 | 初始化值 | 说明 |
|------|----------|------|
| `header.self` | pd | 自引用（TLS 访问用） |
| `start_routine` | 用户函数 | 线程入口 |
| `arg` | 用户参数 | 传给入口函数 |
| `joinstate` | JOINABLE/DETACHED | 由属性决定 |
| `schedpolicy` | 继承或指定 | 调度策略 |
| `schedparam` | 继承或指定 | 调度参数 |
| `sigmask` | 父线程当前掩码 | 初始信号掩码 |
| `stackblock` | mmap 起始 | 栈块首地址 |
| `stackblock_size` | 总大小 | 含 guard |
| `guardsize` | guard 大小 | 保护页大小 |
| `cancelhandling` | 0 | 取消状态位图 |
| `lock` | 初始化 | 内部锁 |
| `robust_head` | 空列表 | 健壮互斥锁链表 |
| `setxid_futex` | -1 | setxid 同步 |

### 4.2 joinstate 枚举

```c
enum {
    THREAD_STATE_EXITED   = 0,  // 已退出
    THREAD_STATE_EXITING  = 1,  // 正在退出
    THREAD_STATE_JOINABLE = 2,  // 可 join
    THREAD_STATE_DETACHED = 3,  // 已分离
};
```

---

## 5. clone 系统调用 (create_thread)

### 5.1 调用方式

```c
// pthread_create.c:288-300
__clone_internal(&clone_args, &start_thread, pd);
```

### 5.2 clone 标志

```
CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM |
CLONE_SIGHAND | CLONE_THREAD | CLONE_SETTLS |
CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID
```

| 标志 | 说明 |
|------|------|
| CLONE_VM | 共享地址空间 |
| CLONE_FS | 共享文件系统信息 |
| CLONE_FILES | 共享文件描述符表 |
| CLONE_SIGHAND | 共享信号处理器 |
| CLONE_THREAD | 同一线程组 |
| CLONE_SETTLS | 设置 TLS |
| CLONE_PARENT_SETTID | 在父进程写入子 TID |
| CLONE_CHILD_CLEARTID | 子退出时清零 TID 并唤醒 futex |

### 5.3 stopped_start 机制

当需要调试器事件通知时:
1. 父线程先持有 `pd->lock`
2. clone 后子线程尝试获取 `pd->lock`（阻塞）
3. 父线程发送 `TD_CREATE` 事件
4. 父线程释放 `pd->lock`
5. 子线程获得锁继续执行

---

## 6. 子线程启动流程 (start_thread)

```
start_thread (pthread_create.c:338)
    │
    ├── 1. stopped_start 同步（如需）
    │       获取 pd->lock，检查 setup_failed
    │
    ├── 2. 初始化运行时环境
    │       ├── 解析器状态 (resolver)
    │       ├── locale 设置
    │       ├── rseq 注册
    │       ├── robust list 注册
    │       └── 栈命名 (prctl PR_SET_VMA)
    │
    ├── 3. 建立取消展开点 (setjmp)
    │
    ├── 4. 允许 setxid 操作
    │
    ├── 5. 调用用户函数
    │       result = start_routine(arg)
    │
    └── 6. 退出清理
            ├── TLS 析构函数
            ├── 线程特定数据 (TSD) 清理
            ├── libc freeres
            ├── 发送 death 事件
            ├── 设置 EXITING 状态
            ├── robust mutex 清理
            ├── madvise(MADV_DONTNEED) 回收栈页
            ├── 等待 setxid 完成
            ├── 必要时释放 TCB
            └── exit(0) — 线程终止
```

---

## 7. 线程属性 (pthread_attr_t)

### 7.1 可设置的属性

| 属性 | 函数 | 默认值 |
|------|------|--------|
| 分离状态 | `pthread_attr_setdetachstate` | JOINABLE |
| 栈大小 | `pthread_attr_setstacksize` | ~8MB |
| 栈地址 | `pthread_attr_setstack` | 自动分配 |
| Guard 大小 | `pthread_attr_setguardsize` | 1 页 (4KB) |
| 调度策略 | `pthread_attr_setschedpolicy` | 继承 |
| 调度参数 | `pthread_attr_setschedparam` | 继承 |
| 继承调度 | `pthread_attr_setinheritsched` | INHERIT |
| CPU 亲和性 | `pthread_attr_setaffinity_np` | 继承 |
| 作用域 | `pthread_attr_setscope` | SYSTEM |

### 7.2 属性生效时机

- **栈相关**: 在 `allocate_stack()` 中使用
- **调度相关**: clone 后通过 `sched_setscheduler` 设置
- **分离状态**: 写入 `pd->joinstate`
- **亲和性**: clone 后通过 `sched_setaffinity` 设置

---

## 8. 错误处理

| 错误码 | 原因 |
|--------|------|
| `EAGAIN` | 资源不足（栈分配失败，线程数上限） |
| `EINVAL` | 属性无效（如栈太小） |
| `EPERM` | 无权限设置调度参数 |

### 8.1 clone 后失败的处理

如果 clone 成功但后续属性设置失败:
1. 设置 `pd->setup_failed = 1`
2. 释放 `pd->lock`（通知子线程）
3. 子线程检测到 `setup_failed` 后自行退出
4. 父线程等待子线程 `joinstate == EXITED` 后回收栈

---

## 9. 性能优化

- **栈缓存**: 退出的线程栈通过 `madvise(MADV_DONTNEED)` 回收物理页后缓存复用
- **单线程优化**: 首个线程创建前避免不必要的同步开销
- **TLS 快速路径**: TCB 直接作为线程指针，无需间接访问
- **信号阻塞优化**: 创建期间阻塞所有信号，避免竞态

---

## 10. 源码位置速查

| 内容 | 文件:行号 |
|------|-----------|
| 主入口 | pthread_create.c:649-929 |
| late_init | pthread_create.c:73 |
| create_thread | pthread_create.c:234-334 |
| start_thread | pthread_create.c:338-627 |
| allocate_stack | allocatestack.c:356-659 |
| get_cached_stack | allocatestack.c:59-151 |
| struct pthread | descr.h:147-425 |
| joinstate 枚举 | descr.h:135-144 |
