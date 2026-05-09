# pthread_cancel 深度分析：glibc 实现复杂度与 Bionic 取舍

## 1. 概述

POSIX `pthread_cancel()` 允许一个线程请求取消另一个线程的执行。这看似简单的功能，
在 glibc 中涉及**信号机制、汇编级 PC 检测、栈展开（unwinding）、原子状态机、
69 个系统调用包装器、9 个 nocancel 变体**等大量基础设施。

Android Bionic 选择**完全不实现** `pthread_cancel`，称其"不太可能被实现"。
本文深入分析 glibc 的实现复杂度，并与 Bionic 的决策进行对比。

---

## 2. glibc 取消状态机

### 2.1 cancelhandling 标志位

每个线程的取消状态由 `struct pthread.cancelhandling`（一个 32 位整数）
管理（`nptl/descr.h:294-316`）：

```c
int cancelhandling;   // 原子操作管理的位域

#define CANCELSTATE_BIT     0   // 置 1 = 取消已禁用
#define CANCELTYPE_BIT      1   // 置 1 = 异步取消模式
#define CANCELING_BIT       2   // 置 1 = 正在发起取消
#define CANCELED_BIT        3   // 置 1 = 已被标记为取消
#define EXITING_BIT         4   // 置 1 = 正在退出（不再接受取消）
#define TERMINATED_BIT      5   // 置 1 = 已终止、TCB 已释放
#define SETXID_BIT          6   // 置 1 = 需要执行 setxid 操作
```

### 2.2 状态判断辅助函数

`nptl/descr.h:427-459` 提供了原子状态判断函数：

```c
// 取消是否启用
static inline bool cancel_enabled (int value) {
    return (value & CANCELSTATE_BITMASK) == 0;
}

// 取消已启用且已被标记为取消（延迟模式检查）
static inline bool cancel_enabled_and_canceled (int value) {
    return (value & (CANCELSTATE | CANCELED | EXITING | TERMINATED))
           == CANCELED_BITMASK;  // 只有 CANCELED 位被设置
}

// 取消已启用且已被取消且异步模式（异步模式检查）
static inline bool cancel_enabled_and_canceled_and_async (int value) {
    return (value & (CANCELSTATE | CANCELTYPE | CANCELED | EXITING | TERMINATED))
           == (CANCELTYPE_BITMASK | CANCELED_BITMASK);
}
```

### 2.3 完整状态转换图

```
                    pthread_cancel(tid)
                          │
                          ▼
              ┌──── CANCELED 位置 1 ────┐
              │                         │
              ▼                         ▼
     取消已启用?                   取消已禁用?
       │    │                         │
       │    │                     记录 CANCELED
       │    │                     等待 re-enable
       ▼    ▼                         │
     异步?  延迟?                      │
       │      │                       │
       ▼      ▼                       ▼
   发送      等待下一个          setcancelstate(ENABLE)
  SIGCANCEL  取消点               检测到 CANCELED
       │      │                       │
       ▼      ▼                       ▼
   sigcancel  系统调用返回        ┌────┴────┐
   _handler   检查 CANCELED      │  异步?   │
       │      │                  ▼         ▼
       │      ▼              __do_cancel   等待取消点
       │  __syscall_do_cancel    │
       │      │                  │
       └──────┴──────────────────┘
                    │
                    ▼
              __do_cancel()
              ├→ 设置 CANCELSTATE | EXITING
              ├→ 清除 CANCELTYPE
              └→ __pthread_unwind()
                   └→ _Unwind_ForcedUnwind()
                        ├→ 执行 cleanup handlers
                        └→ 线程退出
```

---

## 3. pthread_cancel() 实现

### 3.1 核心流程

`__pthread_cancel()`（`nptl/pthread_cancel.c:58-155`）：

```c
int __pthread_cancel (pthread_t th)
{
  volatile struct pthread *pd = (volatile struct pthread *) th;

  // 1. 线程已退出 → 直接返回
  if (pd->joinstate == THREAD_STATE_EXITED)
    return 0;

  // 2. 首次调用时懒安装 SIGCANCEL handler
  static int init_sigcancel = 0;
  if (atomic_load_relaxed (&init_sigcancel) == 0) {
    struct sigaction sa;
    sa.sa_sigaction = sigcancel_handler;
    sa.sa_flags = SA_SIGINFO | SA_RESTART;  // SA_RESTART 避免虚假 EINTR
    __libc_sigaction (SIGCANCEL, &sa, NULL);
    atomic_store_relaxed (&init_sigcancel, 1);
  }

  // 3. 确保 libgcc_s.so 已加载（_Unwind_ForcedUnwind 需要）
  struct unwind_link *unwind_link = __libc_unwind_link_get ();
  if (unwind_link == NULL)
    __libc_fatal ("libgcc_s.so must be installed for pthread_cancel to work\n");

  // 4. 原子设置 CANCELED 位
  int oldval = atomic_load_relaxed (&pd->cancelhandling);
  do {
    newval = oldval | CANCELED_BITMASK;
    if (oldval == newval) break;  // 已经被取消

    if (cancel_enabled (newval)) {
      // 取消已启用：原子更新后发送信号
      atomic_compare_exchange (&pd->cancelhandling, &oldval, newval);

      if (pd == THREAD_SELF) {
        // 自我取消 + 异步模式 → 直接取消
        if (cancel_async_enabled (newval))
          __do_cancel (PTHREAD_CANCELED);
      } else {
        // 跨线程取消 → 发送 SIGCANCEL
        __pthread_kill_internal (th, SIGCANCEL);
      }
      break;
    }
  } while (!atomic_compare_exchange (...));
}
```

**复杂度分析：**
- 需要处理懒初始化信号处理器（线程安全的 init 标志）
- 需要确保 libgcc_s.so 可用（运行时动态链接依赖）
- 需要区分自我取消 vs 跨线程取消
- 所有状态更新通过 CAS 循环实现无锁原子操作

### 3.2 SIGCANCEL 信号处理器

```c
// nptl/pthread_cancel.c:32-56
static void sigcancel_handler (int sig, siginfo_t *si, void *ctx)
{
  // 安全检查：必须是内核发的 SIGCANCEL
  if (sig != SIGCANCEL || si->si_pid != __getpid()
      || si->si_code != SI_TKILL)
    return;

  struct pthread *self = THREAD_SELF;
  int oldval = atomic_load_relaxed (&self->cancelhandling);

  // 条件 1：异步取消模式且已被标记取消
  // 条件 2：被中断的 PC 在可取消系统调用桥中
  if (cancel_enabled_and_canceled_and_async (oldval)
      || cancellation_pc_check (ctx))
    __syscall_do_cancel ();
}
```

---

## 4. 可取消系统调用桥

### 4.1 机制概述

glibc 最精巧也最复杂的部分是**系统调用级取消点**。核心思想是：
在系统调用前后设置全局标签（`__syscall_cancel_arch_start` 和
`__syscall_cancel_arch_end`），SIGCANCEL 信号处理器通过检查被中断的
**程序计数器（PC）** 是否在这两个标签之间来决定是否取消。

### 4.2 AArch64 实现

```asm
;; sysdeps/unix/sysv/linux/aarch64/syscall_cancel.S

ENTRY (__syscall_cancel_arch)

    .globl __syscall_cancel_arch_start
__syscall_cancel_arch_start:
    ;; 检查是否已被标记取消
    ldr     w0, [x0]            ;; 读 *cancelhandling
    tbnz    w0, TCB_CANCELED_BIT, 1f  ;; 若已取消，跳转到取消

    ;; 执行系统调用
    mov     x8, x1              ;; syscall number
    mov     x0, x2              ;; arg1-6
    mov     x1, x3
    mov     x2, x4
    mov     x3, x5
    mov     x4, x6
    mov     x5, x7
    svc     #0                  ;; 系统调用

    .globl __syscall_cancel_arch_end
__syscall_cancel_arch_end:
    ret                         ;; 正常返回

1:  b   __syscall_do_cancel     ;; 取消路径
END (__syscall_cancel_arch)
```

### 4.3 PC 检测的关键约束

```
                 __syscall_cancel_arch_start
                           │
                    ┌──────┼──────┐
                    │  检查取消    │
                    │  设置参数    │  ← SIGCANCEL 在此范围内中断
                    │  svc #0     │     → 可以安全取消
                    └──────┼──────┘
                           │
                 __syscall_cancel_arch_end
                           │
                    ┌──────┼──────┐
                    │  返回值处理  │  ← SIGCANCEL 在此范围外中断
                    │  ...        │     → 系统调用已有副作用，不取消
                    └─────────────┘
```

**设计精髓：** 如果内核在有副作用的系统调用（如 partial read）被信号中断后
设置 PC 到 `svc` 指令之后（`__syscall_cancel_arch_end`），那么信号处理器
检测到 PC 不在范围内，不会取消线程，让调用者正确处理 partial 结果。

### 4.4 C 语言层包装

`nptl/cancellation.c:24-64` 提供 C 语言入口：

```c
long int __internal_syscall_cancel (..., __syscall_arg_t nr)
{
  struct pthread *pd = THREAD_SELF;
  int ch = atomic_load_relaxed (&pd->cancelhandling);

  // 快速路径：单线程 / 取消禁用 / 正在退出 → 直接调用
  if (SINGLE_THREAD_P || !cancel_enabled (ch) || cancel_exiting (ch))
    return INTERNAL_SYSCALL_NCS_CALL (nr, ...);

  // 慢路径：通过汇编桥执行
  result = __syscall_cancel_arch (&pd->cancelhandling, nr, ...);

  // 系统调用被中断且已被标记取消 → 取消线程
  ch = atomic_load_relaxed (&pd->cancelhandling);
  if (result == -EINTR && cancel_enabled_and_canceled (ch))
    __syscall_do_cancel ();

  return result;
}
```

### 4.5 影响范围

glibc 中 **69 个系统调用包装器** 使用 `SYSCALL_CANCEL` 宏，
这些都是 POSIX 定义的取消点：

| 类别 | 系统调用 |
|------|----------|
| 文件 I/O | open, close, read, write, pread64, pwrite64, readv, writev, preadv, pwritev, creat, openat, openat2 |
| 网络 | accept, connect, recv, send, recvfrom, sendto, recvmsg, sendmsg, recvmmsg, sendmmsg |
| 同步 | fsync, fdatasync, msync, sync_file_range |
| 等待 | select, pselect, poll, ppoll, epoll_wait, epoll_pwait, pause, sigsuspend, sigtimedwait, wait4, waitid |
| 管道/传输 | splice, tee, vmsplice, copy_file_range |
| 其他 | clock_nanosleep, tcdrain, getrandom, mq_timedreceive, mq_timedsend, msgrcv, msgsnd, fallocate |

---

## 5. 栈展开与清理处理器

### 5.1 __do_cancel：取消入口

`sysdeps/nptl/pthreadP.h:247-273`：

```c
static inline void __attribute__((noreturn, always_inline))
__do_cancel (void *result)
{
  struct pthread *self = THREAD_SELF;
  self->result = result;  // 设置线程返回值为 PTHREAD_CANCELED

  // 原子设置 CANCELSTATE | EXITING，清除 CANCELTYPE
  // 防止在执行 cleanup handler 时再次被取消
  int oldval = atomic_load_relaxed (&self->cancelhandling);
  do {
    newval = oldval | CANCELSTATE_BITMASK | EXITING_BITMASK;
    newval = newval & ~CANCELTYPE_BITMASK;
  } while (!atomic_compare_exchange (...));

  // 启动栈展开
  __pthread_unwind (THREAD_GETMEM (self, cleanup_jmp_buf));
}
```

### 5.2 __pthread_unwind：强制展开

`nptl/unwind.c:118-135`：

```c
void __attribute__((noreturn))
__pthread_unwind (__pthread_unwind_buf_t *buf)
{
  struct pthread *self = THREAD_SELF;
  THREAD_SETMEM (self, exc.exception_class, 0);
  THREAD_SETMEM (self, exc.exception_cleanup, &unwind_cleanup);

  // 使用 GCC 的 _Unwind_ForcedUnwind 进行强制栈展开
  _Unwind_ForcedUnwind (&self->exc, unwind_stop, ibuf);

  abort ();  // 不应到达
}
```

**关键依赖：** `_Unwind_ForcedUnwind` 来自 **libgcc_s.so**。这意味着：
1. pthread_cancel 在运行时依赖 libgcc_s.so（不是编译时）
2. 需要 DWARF .eh_frame 展开信息
3. 需要 C++ 异常处理基础设施

### 5.3 cleanup handler 注册

`pthread_cleanup_push/pop` 内部使用
`___pthread_register_cancel_defer()`（`nptl/cleanup_defer.c:22-52`）：

```c
void ___pthread_register_cancel_defer (__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  struct pthread *self = THREAD_SELF;

  // 保存旧的 cleanup 栈帧
  ibuf->priv.data.prev = THREAD_GETMEM (self, cleanup_jmp_buf);
  ibuf->priv.data.cleanup = THREAD_GETMEM (self, cleanup);

  // 如果当前是异步取消模式，临时切换到延迟模式
  int cancelhandling = atomic_load_relaxed (&self->cancelhandling);
  if (cancelhandling & CANCELTYPE_BITMASK) {
    // CAS 循环清除 CANCELTYPE 位
    do { newval = cancelhandling & ~CANCELTYPE_BITMASK; }
    while (!atomic_compare_exchange (...));
  }

  // 保存原始取消类型
  ibuf->priv.data.canceltype = ...;

  // 压入新的 cleanup 栈帧
  THREAD_SETMEM (self, cleanup_jmp_buf, ibuf);
}
```

**对应的 `___pthread_unregister_cancel_restore()`**
（`nptl/cleanup_defer.c:61-87`）在弹出时恢复原始取消类型。
如果恢复为异步模式且已被标记取消，立即触发 `__do_cancel()`。

---

## 6. NOCANCEL 变体：内部安全

### 6.1 问题

glibc 自身在执行内部操作时（如 `fclose` 期间的 `close(fd)`），
**不能**在中间被取消，否则会导致资源泄漏或数据结构不一致。
因此需要"不可取消"的系统调用版本。

### 6.2 9 个 NOCANCEL 变体

`sysdeps/unix/sysv/linux/not-cancel.h:32-72` 声明了全部变体：

| nocancel 函数 | 源文件 |
|---------------|--------|
| `__open_nocancel` | `open_nocancel.c` |
| `__open64_nocancel` | `open64_nocancel.c` |
| `__openat_nocancel` | `openat_nocancel.c` |
| `__openat64_nocancel` | `openat64_nocancel.c` |
| `__read_nocancel` | `read_nocancel.c` |
| `__pread64_nocancel` | `pread64_nocancel.c` |
| `__write_nocancel` | `write_nocancel.c` |
| `__close_nocancel` | `close_nocancel.c` |
| `__fcntl64_nocancel` | `fcntl_nocancel.c` |

另有辅助函数：
- `__close_nocancel_nostatus()` — 关闭 fd 但不设置 errno
- `__writev_nocancel_nostatus()` — 内联的不可取消 writev

### 6.3 使用场景

```
stdio:  _IO_file_read()  → __read_nocancel (if FLAGS2_NOTCANCEL)
        _IO_file_close() → __close_nocancel (总是)
ld.so:  所有文件操作     → __open_nocancel / __read_nocancel / __close_nocancel
malloc: 不涉及系统调用取消点
nptl:   allocatestack.c  → mmap 不是取消点，无需处理
```

### 6.4 取消安全的复杂性传播

取消功能对整个 libc 产生**传染效应**：

```
pthread_cancel 存在
  → 每个系统调用包装器需要两个版本（cancel / nocancel）
  → 每个库内部函数需要决定用哪个版本
  → stdio 需要 _IO_FLAGS2_NOTCANCEL 标志
  → 所有锁操作需要考虑取消时的清理
  → ld.so 必须全部使用 nocancel 版本
  → 信号掩码操作必须防止阻塞 SIGCANCEL
```

---

## 7. 与其他子系统的交互

### 7.1 stdio 锁与取消

`_IO_acquire_lock` 使用 GCC `cleanup` 属性自动释放
（`sysdeps/nptl/stdio-lock.h:98-107`）：

```c
#define _IO_acquire_lock(_fp)                                    \
  do {                                                           \
    FILE *_IO_acquire_lock_file                                  \
        __attribute__((cleanup (_IO_acquire_lock_fct))) = (_fp); \
    _IO_flockfile (_IO_acquire_lock_file);
```

这确保即使线程被取消，cleanup 函数也会释放 FILE 锁。
但问题是：取消发生在 `_IO_sputn` 内部时，FILE 缓冲区可能处于
不一致状态（写了一半的数据）。

### 7.2 pthread_sigmask 防护

`nptl/pthread_sigmask.c:23-37`：

```c
int __pthread_sigmask (int how, const sigset_t *newmask, sigset_t *oldmask)
{
  // 强制从掩码中移除 SIGCANCEL 和 SIGSETXID
  if (newmask != NULL
      && (__sigismember (newmask, SIGCANCEL)
          || __sigismember (newmask, SIGSETXID)))
  {
    local_newmask = *newmask;
    clear_internal_signals (&local_newmask);  // 移除内部信号
    newmask = &local_newmask;
  }
  ...
}
```

用户永远无法阻塞 SIGCANCEL，这是取消机制正确工作的前提。

### 7.3 SA_RESTART 与取消

`pthread_cancel` 安装 SIGCANCEL handler 时使用 `SA_RESTART`
（`pthread_cancel.c:79`）。这是为了避免在**取消未启用**时，
SIGCANCEL 信号导致正在阻塞的系统调用返回虚假的 EINTR。

但某些系统调用（如 `connect`, `sigsuspend`, `select`）即使设置了
SA_RESTART 也不会自动重启——这正是 glibc 选择在
`__internal_syscall_cancel` 返回后再检查 EINTR + CANCELED 的原因
（`nptl/cancellation.c:60-61`）。

---

## 8. Bionic 的决策：不实现 pthread_cancel

### 8.1 Bionic 的现状

- `pthread_cancel()` — **不提供**（头文件中被 `#if !__BIONIC__` 排除）
- `pthread_setcancelstate()` — **不提供**
- `pthread_setcanceltype()` — **不提供**
- `pthread_testcancel()` — **不提供**
- Bionic 定义了 `_POSIX_THREADS` 但明确标注缺失 cancel 相关函数

### 8.2 官方理由

Bionic 文档 `docs/status.md` 中的声明（大意）：

> "Cancellation is unlikely to ever be implemented. It's hard to implement
> correctly, expensive in terms of code size and runtime cost, and
> difficult to use correctly in applications."

三个核心论点：
1. **实现困难** — 需要信号、汇编、栈展开等大量基础设施
2. **代价高昂** — 运行时开销（每个系统调用多一次原子检查 + 分支）
3. **使用困难** — 应用程序难以正确使用取消（资源泄漏风险）

### 8.3 替代方案

Bionic/Android 推荐的线程终止方式：

| 场景 | 替代方案 |
|------|----------|
| 线程取消 | `pthread_exit()` + 应用层协议（设置退出标志） |
| 线程存活检查 | `pthread_gettid_np()` + `kill(tid, 0)` |
| 阻塞操作中断 | 使用 `poll/epoll` + eventfd/pipe 的自唤醒模式 |
| 超时等待 | `sem_timedwait()` / `pthread_cond_timedwait()` |

---

## 9. 对比分析

### 9.1 实现复杂度对比

| 维度 | glibc | Bionic |
|------|-------|--------|
| 取消相关源文件 | ~15 个 C 文件 + 每个架构 1 个 .S 文件 | 0 |
| 系统调用包装器 | 每个取消点 2 个版本（69 cancel + 9 nocancel = 78） | 无双版本开销 |
| 信号占用 | SIGCANCEL (signal 32) 被保留 | 无 |
| 运行时依赖 | libgcc_s.so（_Unwind_ForcedUnwind） | 无 |
| 每次系统调用开销 | 1 次原子读 + 1 次分支（快速路径） | 0 |
| 全局状态 | init_sigcancel 标志、sigaction 安装 | 无 |
| 信号掩码干预 | pthread_sigmask 必须过滤 SIGCANCEL | 无 |

### 9.2 glibc pthread_cancel 的优点

1. **POSIX 完整兼容** — 满足 POSIX.1 标准对线程取消的全部要求
2. **优雅的线程终止** — cleanup handler 确保资源被正确释放
3. **可中断阻塞操作** — 即使线程阻塞在 `read()`/`accept()` 等系统调用中，
   也能被取消
4. **延迟取消语义** — 只在安全的取消点响应取消，减少数据损坏风险
5. **有副作用的系统调用保护** — PC 检测机制确保 partial read/write
   的结果不会丢失
6. **C++ 异常集成** — 通过 `_Unwind_ForcedUnwind` 与 C++ 析构函数
   和 try/catch 自然集成

### 9.3 glibc pthread_cancel 的缺点

1. **实现极其复杂**
   - 需要汇编级别的 PC 标签检测（每个架构独立实现）
   - 依赖 GCC 的 `_Unwind_ForcedUnwind`（非标准）
   - 状态机有 7 个标志位和复杂的原子操作
   - 信号处理器必须小心验证信号来源

2. **性能开销**
   - 每个取消点系统调用多一次 `atomic_load_relaxed` + 条件分支
   - 单线程进程也受影响（有 `SINGLE_THREAD_P` 快速路径，但仍有函数调用开销）
   - 需要维护 69 个系统调用的 SYSCALL_CANCEL 包装

3. **传染性复杂度**
   - libc 内部代码必须区分"可取消"和"不可取消"操作
   - 9 个 nocancel 系统调用变体增加代码维护负担
   - stdio、ld.so、malloc 等子系统都需要感知取消机制
   - `_IO_FLAGS2_NOTCANCEL` 在 stdio 中的传播

4. **正确使用困难**
   - 应用程序员必须在每个取消点设置正确的 cleanup handler
   - 遗漏 cleanup handler 导致资源泄漏（锁、文件描述符、内存）
   - 异步取消几乎无法安全使用（只有纯计算代码才安全）
   - cleanup handler 栈管理容易出错

5. **信号资源占用**
   - SIGCANCEL (signal 32) 被永久保留，减少可用实时信号数
   - `pthread_sigmask` 被修改，用户无法感知内部信号

6. **调试困难**
   - 取消点发生在系统调用内部，调试器难以断点
   - 栈展开路径涉及信号帧、C 栈和 DWARF unwind 信息
   - 取消状态机的原子操作增加竞态条件的排查难度

### 9.4 Bionic 不实现的权衡

**获得的好处：**
- 系统调用路径更简洁，零额外开销
- 无需维护 nocancel 变体和双版本包装器
- 不占用实时信号
- 不依赖 libgcc_s.so
- 代码更简单，更易维护和审计

**付出的代价：**
- 不符合 POSIX 完整标准（但 Android 不追求完整 POSIX）
- 依赖 `pthread_cancel` 的第三方库无法直接移植
- 无法优雅中断阻塞在系统调用中的线程
- 应用需要自行实现协作式取消模式

---

## 10. 总结

```
┌────────────────────────────────────────────────────────────────┐
│               pthread_cancel 实现成本金字塔                     │
│                                                                │
│                        ╱╲                                      │
│                       ╱  ╲    应用层：正确使用 cleanup handler  │
│                      ╱    ╲                                    │
│                     ╱──────╲                                   │
│                    ╱        ╲   libc：69 个取消点 + 9 nocancel │
│                   ╱          ╲                                 │
│                  ╱────────────╲                                │
│                 ╱              ╲  NPTL：信号 + 状态机 + 展开   │
│                ╱                ╲                              │
│               ╱──────────────────╲                             │
│              ╱                    ╲  arch：汇编 PC 标签桥      │
│             ╱                      ╲                           │
│            ╱────────────────────────╲                          │
│           ╱  GCC/libgcc_s：_Unwind   ╲                        │
│          ╱────────────────────────────╲                        │
│         ╱     内核：SIGCANCEL 信号     ╲                       │
│        ╱──────────────────────────────────╲                    │
└────────────────────────────────────────────────────────────────┘
```

glibc 的 `pthread_cancel` 是一个**工程杰作**——它在用户态实现了
对任意线程的安全取消，同时保护了有副作用的系统调用的完整性。
但它也是一个**复杂度黑洞**——从内核信号到汇编标签到栈展开到 C++ 异常，
纵跨整个软件栈。

Bionic 选择不实现它并非技术能力不足，而是**成本收益分析**的结果：
在移动设备上，这个功能的使用频率极低，但实现和维护成本极高。
Android 的 Java/Kotlin 线程模型有自己的中断机制（`Thread.interrupt()`），
native 层使用协作式设计模式完全可以替代取消语义。

---

## 11. 源码位置速查表

| 功能 | 源文件 | 行号 |
|------|--------|------|
| cancelhandling 标志位定义 | `nptl/descr.h` | 294-316 |
| 取消状态辅助函数 | `nptl/descr.h` | 427-459 |
| pthread_cancel 实现 | `nptl/pthread_cancel.c` | 58-155 |
| sigcancel_handler | `nptl/pthread_cancel.c` | 32-56 |
| pthread_setcancelstate | `nptl/pthread_setcancelstate.c` | 23-58 |
| pthread_setcanceltype | `nptl/pthread_setcanceltype.c` | 23-61 |
| __internal_syscall_cancel | `nptl/cancellation.c` | 24-64 |
| __syscall_do_cancel | `nptl/cancellation.c` | 84-110 |
| __syscall_cancel_arch (C 参考) | `sysdeps/unix/sysv/linux/syscall_cancel.c` | 51-73 |
| __syscall_cancel_arch (AArch64) | `sysdeps/unix/sysv/linux/aarch64/syscall_cancel.S` | 全文 |
| __do_cancel | `sysdeps/nptl/pthreadP.h` | 247-273 |
| __pthread_unwind | `nptl/unwind.c` | 118-135 |
| ___pthread_unwind_next | `nptl/unwind.c` | 138-145 |
| cleanup_defer 注册 | `nptl/cleanup_defer.c` | 22-52 |
| cleanup_defer 恢复 | `nptl/cleanup_defer.c` | 61-87 |
| NOCANCEL 声明 | `sysdeps/unix/sysv/linux/not-cancel.h` | 32-72 |
| pthread_sigmask 过滤 | `nptl/pthread_sigmask.c` | 23-37 |
| _IO_acquire_lock（取消安全） | `sysdeps/nptl/stdio-lock.h` | 98-107 |
