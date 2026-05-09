# SIGCANCEL 与 SIGSETXID 内部信号机制

## 概述

glibc NPTL 保留了两个实时信号供内部使用，用户程序不可见也不可操作：

| 信号 | 值 | 定义 | 用途 |
|------|-----|------|------|
| **SIGCANCEL** | `__SIGRTMIN` (32) | 异步线程取消 + POSIX 定时器 |
| **SIGSETXID** | `__SIGRTMIN + 1` (33) | 全线程凭据同步 (setuid/setgid) |

- SIGCANCEL: 当 `pthread_cancel()` 被调用时，向目标线程发送此信号触发取消
- SIGSETXID: 当进程调用 `setuid()` 等时，广播到所有线程使其同步变更凭据

本文分析这两个信号的完整实现，包括信号定义、过滤保护、处理器安装、
取消状态机、凭据广播协议。

---

## 一、信号定义与保留

### 1.1 信号号分配

**源文件**: `sysdeps/unix/sysv/linux/internal-signals.h:30-46`

```c
#define SIGCANCEL    __SIGRTMIN        // 信号 32: 异步取消
#define SIGTIMER     SIGCANCEL         // 复用: POSIX 定时器
#define SIGSETXID    (__SIGRTMIN + 1)  // 信号 33: 凭据变更
#define RESERVED_SIGRT  2             // 保留 2 个实时信号
```

### 1.2 用户可见的实时信号范围

**源文件**: `signal/allocrtsig.c:26-39`

```c
static int current_rtmin = __SIGRTMIN + RESERVED_SIGRT;  // 34
static int current_rtmax = __SIGRTMAX;                   // 64

int __libc_current_sigrtmin(void) { return current_rtmin; }
int __libc_current_sigrtmax(void) { return current_rtmax; }
```

用户看到的 `SIGRTMIN` 实际是 34，而非内核的 32。信号 32-33 被隐藏。

---

## 二、内部信号过滤保护

glibc 在多个层面阻止用户操作 SIGCANCEL 和 SIGSETXID：

### 2.1 is_internal_signal() 检查

**源文件**: `sysdeps/unix/sysv/linux/internal-signals.h:50-54`

```c
static inline bool
is_internal_signal(int sig) {
    return (sig == SIGCANCEL) || (sig == SIGSETXID);
}
```

### 2.2 各 API 的过滤点

| API | 文件:行 | 过滤方式 |
|-----|---------|---------|
| `sigaction()` | `signal/sigaction.c:28-31` | `is_internal_signal(sig)` → `EINVAL` |
| `sigaddset()` | `signal/sigaddset.c:27-32` | `is_internal_signal(sig)` → `EINVAL` |
| `sigdelset()` | `signal/sigdelset.c:27-32` | `is_internal_signal(sig)` → `EINVAL` |
| `sigfillset()` | `signal/sigfillset.c:33-35` | `clear_internal_signals(set)` 清除这两位 |
| `pthread_sigmask()` | `nptl/pthread_sigmask.c:28-36` | 清除 set 中的 SIGCANCEL/SIGSETXID |
| `pthread_kill()` | `nptl/pthread_kill.c` | `is_internal_signal(signo)` → `EINVAL` |
| `pthread_sigqueue()` | `nptl/pthread_sigqueue.c:41-44` | 拒绝 SIGCANCEL/SIGTIMER/SIGSETXID |

### 2.3 clear_internal_signals()

**源文件**: `sysdeps/unix/sysv/linux/internal-signals.h:57-62`

```c
static inline void
clear_internal_signals(sigset_t *set) {
    __sigdelset(set, SIGCANCEL);
    __sigdelset(set, SIGSETXID);
}
```

### 2.4 绕过过滤的内部 API

glibc 内部需要直接操作这些信号时，使用不经过过滤的内部函数：

```c
// 直接调用 rt_sigprocmask 系统调用，绕过 pthread_sigmask 过滤
static inline int
internal_sigprocmask(int how, const internal_sigset_t *set,
                     internal_sigset_t *oldset) {
    return INTERNAL_SYSCALL_CALL(rt_sigprocmask, how, set, oldset,
                                 __NSIG_BYTES);
}

// 阻塞所有信号（含内部信号）
static inline void
internal_signal_block_all(internal_sigset_t *oset) {
    INTERNAL_SYSCALL_CALL(rt_sigprocmask, SIG_BLOCK, &sigall_set, oset, ...);
}
```

---

## 三、cancelhandling 位域状态机

### 3.1 位定义

**源文件**: `nptl/descr.h:294-316`

```c
int cancelhandling;  // struct pthread 中的取消状态字段

// 位定义:
#define CANCELSTATE_BIT    0   // 置位 = 取消被禁用
#define CANCELSTATE_BITMASK  (1 << 0)   // 0x01

#define CANCELTYPE_BIT     1   // 置位 = 异步取消模式
#define CANCELTYPE_BITMASK   (1 << 1)   // 0x02

#define CANCELING_BIT      2   // 置位 = 正在取消
#define CANCELING_BITMASK    (1 << 2)   // 0x04

#define CANCELED_BIT       3   // 置位 = 已被标记取消
#define CANCELED_BITMASK     (1 << 3)   // 0x08

#define EXITING_BIT        4   // 置位 = 正在退出
#define EXITING_BITMASK      (1 << 4)   // 0x10

#define TERMINATED_BIT     5   // 置位 = 已终止，TCB 已释放
#define TERMINATED_BITMASK   (1 << 5)   // 0x20

#define SETXID_BIT         6   // 置位 = 需要执行 XID 变更
#define SETXID_BITMASK       (1 << 6)   // 0x40
```

### 3.2 状态判断辅助函数

**源文件**: `nptl/descr.h:427-459`

```c
// 取消是否启用? (CANCELSTATE_BIT == 0 表示启用)
cancel_enabled(value)
  → (value & CANCELSTATE_BITMASK) == 0

// 异步取消是否启用?
cancel_async_enabled(value)
  → (value & CANCELTYPE_BITMASK) != 0

// 是否正在退出?
cancel_exiting(value)
  → (value & EXITING_BITMASK) != 0

// 取消已启用且已被标记?
cancel_enabled_and_canceled(value)
  → 仅 CANCELED_BIT 置位，CANCELSTATE/EXITING/TERMINATED 均未置位

// 异步取消已启用、已被标记且取消未禁用?
cancel_enabled_and_canceled_and_async(value)
  → CANCELTYPE + CANCELED 置位，CANCELSTATE/EXITING/TERMINATED 均未置位
```

### 3.3 状态转换图

```
  初始状态: cancelhandling = 0
  (取消启用, 延迟模式, 未取消)

  pthread_setcancelstate(DISABLE):
    cancelhandling |= CANCELSTATE_BITMASK    (bit 0 = 1)

  pthread_setcanceltype(ASYNC):
    cancelhandling |= CANCELTYPE_BITMASK     (bit 1 = 1)

  pthread_cancel() 标记目标:
    cancelhandling |= CANCELED_BITMASK       (bit 3 = 1)

  取消处理开始:
    cancelhandling |= CANCELSTATE_BITMASK | EXITING_BITMASK
    cancelhandling &= ~CANCELTYPE_BITMASK
    → 禁用进一步取消，标记正在退出

  线程终止:
    cancelhandling |= TERMINATED_BITMASK     (bit 5 = 1)

  setxid 标记:
    cancelhandling |= SETXID_BITMASK         (bit 6 = 1)
    → 处理完后清除
```

---

## 四、SIGCANCEL — 线程取消机制

### 4.1 处理器安装

SIGCANCEL 处理器在首次调用 `pthread_cancel()` 时**惰性安装**：

**源文件**: `nptl/pthread_cancel.c:70-83`

```c
// 静态标志避免重复安装
static int init_sigcancel = 0;
if (atomic_load_relaxed(&init_sigcancel) == 0) {
    struct sigaction sa;
    sa.sa_sigaction = sigcancel_handler;
    sa.sa_flags = SA_SIGINFO | SA_RESTART;  // SA_RESTART: 避免误触 EINTR
    __sigemptyset(&sa.sa_mask);
    __libc_sigaction(SIGCANCEL, &sa, NULL);  // 直接调用内部版本
    atomic_store_relaxed(&init_sigcancel, 1);
}
```

### 4.2 SIGCANCEL 处理器

**源文件**: `nptl/pthread_cancel.c:32-56`

```c
static void
sigcancel_handler(int sig, siginfo_t *si, void *ctx) {
    // ① 安全验证: 必须来自本进程的 tgkill
    if (sig != SIGCANCEL
        || si->si_pid != __getpid()
        || si->si_code != SI_TKILL)
        return;

    // ② 检查是否应该执行取消
    struct pthread *self = THREAD_SELF;
    int oldval = atomic_load_relaxed(&self->cancelhandling);
    
    if (cancel_enabled_and_canceled_and_async(oldval)    // 异步模式 + 已标记
        || cancellation_pc_check(ctx))                    // 或在可取消系统调用中
        __syscall_do_cancel();                            // 执行取消（不返回）
}
```

### 4.3 pthread_cancel() 完整流程

**源文件**: `nptl/pthread_cancel.c:58-154`

```
__pthread_cancel(th):
  // ① 检查线程是否已退出
  if pd->joinstate == THREAD_STATE_EXITED:
    return 0
  
  // ② 惰性安装 SIGCANCEL 处理器（首次调用时）
  安装 sigcancel_handler (SA_SIGINFO | SA_RESTART)
  
  // ③ 加载 libgcc_s（forced unwind 需要）
  __libc_unwind_link_get()  // 失败则 fatal
  
  // ④ 原子设置 CANCELED_BITMASK
  do:
    newval = oldval | CANCELED_BITMASK
    if oldval == newval: break   // 已经标记过
    
    if cancel_enabled(newval):   // 取消已启用
      CAS(&pd->cancelhandling, oldval, newval)
      
      if pd == THREAD_SELF:      // 取消自己
        if cancel_async_enabled: __do_cancel(PTHREAD_CANCELED)
      else:                       // 取消其他线程
        __pthread_kill_internal(th, SIGCANCEL)  // 发送信号
      break
      
  while CAS 失败重试
  
  // ⑤ 标记为多线程（确保取消点生效）
  THREAD_SETMEM(self, header.multiple_threads, 1)
```

### 4.4 取消点与可取消系统调用

#### 延迟取消 (默认模式)

**源文件**: `nptl/cancellation.c:24-64`

glibc 中的取消点通过 `__syscall_cancel_arch` 实现。这是一段带有特殊标记的
汇编代码，SIGCANCEL 处理器通过 `cancellation_pc_check(ctx)` 检查被中断的
PC 是否落在此代码范围内：

```
__internal_syscall_cancel(nr, a1..a6):
  // 如果取消被禁用或正在退出，直接执行 syscall
  ch = atomic_load(&self->cancelhandling)
  if SINGLE_THREAD_P || !cancel_enabled(ch) || cancel_exiting(ch):
    return INTERNAL_SYSCALL_NCS(nr, ...)
  
  // 调用带标记的汇编入口
  result = __syscall_cancel_arch(&self->cancelhandling, nr, ...)
  
  // 如果被 EINTR 中断且已标记取消
  if result == -EINTR && cancel_enabled_and_canceled(ch):
    __syscall_do_cancel()
  
  return result
```

#### __syscall_do_cancel

**源文件**: `nptl/cancellation.c:84-110`

```c
_Noreturn void __syscall_do_cancel(void) {
    struct pthread *self = THREAD_SELF;
    
    // 禁用取消（防止清理处理器中递归取消）
    int oldval = atomic_load_relaxed(&self->cancelhandling);
    while (1) {
        int newval = oldval | CANCELSTATE_BITMASK;
        if (oldval == newval) break;
        if (atomic_compare_exchange_weak_acquire(&self->cancelhandling,
                                                  &oldval, newval))
            break;
    }
    
    __do_cancel(PTHREAD_CANCELED);  // 不返回
}
```

#### __do_cancel

**源文件**: `sysdeps/nptl/pthreadP.h:249-273`

```c
static inline void __attribute__((noreturn, always_inline))
__do_cancel(void *result) {
    struct pthread *self = THREAD_SELF;
    self->result = result;  // 设置线程返回值为 PTHREAD_CANCELED
    
    // 原子设置: 禁用取消 + 标记退出 + 清除异步模式
    int newval = oldval | CANCELSTATE_BITMASK | EXITING_BITMASK;
    newval = newval & ~CANCELTYPE_BITMASK;
    CAS(&self->cancelhandling, oldval, newval);
    
    // 触发 forced unwind（执行清理处理器链）
    __pthread_unwind(self->cleanup_jmp_buf);
}
```

### 4.5 取消执行完整调用链

```
pthread_cancel(target_thread)
  │
  ├─ target == self && async?
  │    → __do_cancel(PTHREAD_CANCELED) [直接取消]
  │
  └─ target != self
       → tgkill(pid, tid, SIGCANCEL)
            │
            ▼ (目标线程收到信号)
       sigcancel_handler()
         ├─ 异步模式且已标记?
         │    → __syscall_do_cancel()
         │         → __do_cancel()
         │              → __pthread_unwind()
         │                   → _Unwind_ForcedUnwind()
         │                        → 遍历清理处理器栈
         │                        → __pthread_exit()
         │
         └─ 延迟模式且 PC 在可取消 syscall 中?
              → __syscall_do_cancel()  [同上]
```

---

## 五、SIGSETXID — 凭据同步机制

### 5.1 问题背景

Linux 内核中，`setuid()` 等系统调用只影响调用线程的凭据（per-thread credentials），
但 POSIX 要求凭据变更对整个进程生效。NPTL 通过 SIGSETXID 信号广播解决此问题。

### 5.2 xid_command 结构

**源文件**: `nptl/descr.h:97-109`

```c
struct xid_command {
    int syscall_no;              // 系统调用号 (如 __NR_setuid)
    unsigned long int id[3];     // 参数 (如 uid, gid)
    volatile int cntr;           // 待完成计数
    volatile int error;          // -1: 未调用, 0: 成功, >0: 错误码
};
```

### 5.3 处理器安装

**源文件**: `nptl/pthread_create.c:72-93`

SIGSETXID 处理器在 `late_init()` 中安装（首次创建线程时）：

```c
static void late_init(void) {
    struct sigaction sa;
    __sigemptyset(&sa.sa_mask);
    
    sa.sa_sigaction = __nptl_setxid_sighandler;
    sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;
    // SA_ONSTACK: 信号可能发给使用自定义栈的线程
    __libc_sigaction(SIGSETXID, &sa, NULL);
    
    // 解除阻塞内部信号
    __sigaddset(&sa.sa_mask, SIGCANCEL);
    __sigaddset(&sa.sa_mask, SIGSETXID);
    INTERNAL_SYSCALL_CALL(rt_sigprocmask, SIG_UNBLOCK, &sa.sa_mask, ...);
}
```

### 5.4 SIGSETXID 处理器

**源文件**: `nptl/nptl_setxid.c:55-93`

```c
void __nptl_setxid_sighandler(int sig, siginfo_t *si, void *ctx) {
    // ① 安全验证
    if (sig != SIGSETXID || si->si_pid != __getpid()
        || si->si_code != SI_TKILL)
        return;
    
    // ② 执行实际的系统调用
    result = INTERNAL_SYSCALL_NCS(xidcmd->syscall_no, 3,
                                  xidcmd->id[0], xidcmd->id[1], xidcmd->id[2]);
    setxid_error(xidcmd, error);  // 记录结果（不一致则 abort）
    
    // ③ 清除 SETXID 标志
    do {
        flags = self->cancelhandling;
        newval = flags & ~SETXID_BITMASK;
    } while (CMPXCHG(self->cancelhandling, newval, flags) != flags);
    
    // ④ 唤醒等待的发送方
    self->setxid_futex = 1;
    futex_wake(&self->setxid_futex, 1, FUTEX_PRIVATE);
    
    // ⑤ 递减计数器，最后一个完成者唤醒主线程
    if (atomic_fetch_add_relaxed(&xidcmd->cntr, -1) == 1)
        futex_wake(&xidcmd->cntr, 1, FUTEX_PRIVATE);
}
```

### 5.5 __nptl_setxid() 广播协议

**源文件**: `nptl/nptl_setxid.c:173-278`

```
__nptl_setxid(cmdp):
  // ① 加锁（保护线程列表遍历）
  lll_lock(dl_stack_cache_lock)
  
  xidcmd = cmdp       // 全局变量，处理器通过它获取命令
  cmdp->cntr = 0      // 待完成计数
  cmdp->error = -1    // 未调用标记
  
  // ② 标记所有线程（设置 SETXID_BITMASK）
  for each thread t in dl_stack_used + dl_stack_user:
    if t == self: continue
    setxid_mark_thread(cmdp, t):
      等待线程完成 clone（setxid_futex != -1）
      t->setxid_futex = 0
      原子设置 t->cancelhandling |= SETXID_BITMASK
      (若线程正在退出则跳过)
  
  // ③ 向所有标记线程发送 SIGSETXID
  do:
    signalled = 0
    for each thread t:
      if t->cancelhandling & SETXID_BITMASK:
        tgkill(pid, t->tid, SIGSETXID)
        cntr++
        signalled++
    
    // 等待所有线程完成
    while cmdp->cntr != 0:
      futex_wait(&cmdp->cntr, cur)
    
  while signalled != 0
  // 重复直到没有新线程需要通知
  
  // ④ 清理标记
  for each thread t:
    setxid_unmark_thread(cmdp, t)
    t->cancelhandling &= ~SETXID_BITMASK
    t->setxid_futex = 1
    futex_wake(...)
  
  // ⑤ 当前线程自己执行系统调用（必须最后做，保留发信号的权限）
  result = INTERNAL_SYSCALL_NCS(cmdp->syscall_no, ...)
  setxid_error(cmdp, error)
  
  // ⑥ 解锁
  lll_unlock(dl_stack_cache_lock)
  return result
```

### 5.6 setuid 等包装宏

**源文件**: `sysdeps/nptl/setxid.h:22-42`

```c
#define INLINE_SETXID_SYSCALL(name, nr, args...) ({
    int __result;
    if (!SINGLE_THREAD_P) {                      // 多线程
        struct xid_command __cmd;
        __cmd.syscall_no = __NR_##name;
        __SETXID_##nr(__cmd, args);
        __result = __nptl_setxid(&__cmd);        // 广播到所有线程
    } else {                                      // 单线程
        __result = INLINE_SYSCALL(name, nr, args); // 直接调用
    }
    __result;
})
```

使用此宏的函数包括：

| 函数 | 源文件 |
|------|--------|
| `setuid` | `sysdeps/unix/sysv/linux/setuid.c` |
| `seteuid` | `sysdeps/unix/sysv/linux/seteuid.c` |
| `setreuid` | `sysdeps/unix/sysv/linux/setreuid.c` |
| `setresuid` | `sysdeps/unix/sysv/linux/setresuid.c` |
| `setgid` | `sysdeps/unix/sysv/linux/setgid.c` |
| `setgroups` | `sysdeps/unix/sysv/linux/setgroups.c` |

---

## 六、SIGSETXID 广播时序图

```
主线程调用 setuid(1000)
    │
    ▼
INLINE_SETXID_SYSCALL(__NR_setuid, 1, 1000)
    │
    ▼
__nptl_setxid(&cmd)
    │
    ├─ ① 加锁 dl_stack_cache_lock
    ├─ ② 遍历 dl_stack_used + dl_stack_user
    │     标记每个线程: cancelhandling |= SETXID_BITMASK
    │
    ├─ ③ 循环发送 SIGSETXID:
    │     tgkill(pid, tid_1, SIGSETXID)  ──→ 线程1 信号处理器
    │     tgkill(pid, tid_2, SIGSETXID)  ──→ 线程2 信号处理器
    │     tgkill(pid, tid_3, SIGSETXID)  ──→ 线程3 信号处理器
    │                                          │
    │     futex_wait(&cmd.cntr, ...)            │
    │     ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ┘
    │           各线程完成后递减 cntr + futex_wake
    │
    ├─ ④ 清除所有线程的 SETXID_BITMASK
    │
    ├─ ⑤ 主线程自己执行 setuid(1000)
    │
    └─ ⑥ 解锁，返回结果
    
各线程的信号处理器执行:
  __nptl_setxid_sighandler():
    syscall(__NR_setuid, 1000)        // 本线程执行 setuid
    cancelhandling &= ~SETXID_BITMASK  // 清除标记
    setxid_futex = 1                   // 唤醒等待者
    cntr--                             // 递减计数
    if cntr == 0: futex_wake(&cntr)    // 最后一个唤醒主线程
```

---

## 七、清理处理器 (Cleanup Handlers)

### 7.1 pthread_unwind_buf 结构

**源文件**: `nptl/descr.h:65-92`

```c
struct pthread_unwind_buf {
    struct {
        __jmp_buf jmp_buf;     // setjmp 保存点
        int mask_was_saved;    // 是否保存了信号掩码
    } cancel_jmp_buf[1];

    union {
        void *pad[4];          // 公共版本占位
        struct {
            struct pthread_unwind_buf *prev;      // 前一个清理缓冲区
            struct _pthread_cleanup_buffer *cleanup; // 旧式清理处理器
            int canceltype;    // push 时的取消类型
        } data;
    } priv;
};
```

### 7.2 取消展开流程

**源文件**: `nptl/unwind.c:118-135`

```c
void __pthread_unwind(__pthread_unwind_buf_t *buf) {
    struct pthread *self = THREAD_SELF;
    
    // 初始化异常对象
    self->exc.exception_class = 0;
    self->exc.exception_cleanup = &unwind_cleanup;
    
    // 触发 forced unwind（由 libgcc_s 提供）
    _Unwind_ForcedUnwind(&self->exc, unwind_stop, ibuf);
    
    abort();  // 不应到达
}
```

`_Unwind_ForcedUnwind` 会遍历调用栈，在每一帧调用 `unwind_stop` 回调，
该回调执行注册的清理处理器，直到到达 `cancel_jmp_buf` 保存点。

---

## 八、一致性检查与错误处理

### 8.1 setxid_error — 凭据一致性

**源文件**: `nptl/nptl_setxid.c:29-47`

```c
static void setxid_error(struct xid_command *cmdp, int error) {
    do {
        int olderror = cmdp->error;
        if (olderror == error)
            break;
        if (olderror != -1) {
            // 不同线程返回不同结果 → 凭据不一致 → abort!
            volatile int xid_err = error;
            abort();
        }
    } while (atomic_compare_and_exchange_bool_acq(&cmdp->error, error, -1));
}
```

如果一个线程 setuid 成功但另一个失败（例如 capability 不同），则进程 **abort**。
这保证了 POSIX 要求的进程范围凭据一致性。

---

## 九、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `sysdeps/unix/sysv/linux/internal-signals.h:30-46` | SIGCANCEL/SIGSETXID 定义 |
| `sysdeps/unix/sysv/linux/internal-signals.h:50-54` | `is_internal_signal()` |
| `sysdeps/unix/sysv/linux/internal-signals.h:57-62` | `clear_internal_signals()` |
| `sysdeps/unix/sysv/linux/internal-signals.h:69-101` | `internal_sigprocmask` 等内部 API |
| `signal/allocrtsig.c:26-39` | 用户可见 SIGRTMIN = __SIGRTMIN + 2 |
| `signal/sigaction.c:28-31` | sigaction 拒绝内部信号 |
| `nptl/pthread_sigmask.c:28-36` | pthread_sigmask 过滤内部信号 |
| `nptl/descr.h:65-92` | `pthread_unwind_buf` (清理处理器结构) |
| `nptl/descr.h:97-109` | `struct xid_command` (XID 命令) |
| `nptl/descr.h:294-316` | `cancelhandling` 位定义 (7 个位) |
| `nptl/descr.h:427-459` | 取消状态判断辅助函数 |
| `nptl/pthread_cancel.c:32-56` | `sigcancel_handler` |
| `nptl/pthread_cancel.c:58-154` | `__pthread_cancel` 完整实现 |
| `nptl/pthread_cancel.c:70-83` | SIGCANCEL 处理器惰性安装 |
| `nptl/cancellation.c:24-64` | `__internal_syscall_cancel` (取消点) |
| `nptl/cancellation.c:84-110` | `__syscall_do_cancel` (执行取消) |
| `sysdeps/nptl/pthreadP.h:249-273` | `__do_cancel` (设置结果 + forced unwind) |
| `nptl/unwind.c:118-135` | `__pthread_unwind` (触发展开) |
| `nptl/pthread_create.c:72-93` | `late_init` (安装 SIGSETXID 处理器) |
| `nptl/nptl_setxid.c:29-47` | `setxid_error` (一致性检查) |
| `nptl/nptl_setxid.c:55-93` | `__nptl_setxid_sighandler` |
| `nptl/nptl_setxid.c:96-130` | `setxid_mark_thread` (标记线程) |
| `nptl/nptl_setxid.c:153-171` | `setxid_signal_thread` (发送 SIGSETXID) |
| `nptl/nptl_setxid.c:173-278` | `__nptl_setxid` (广播协议) |
| `sysdeps/nptl/setxid.h:22-42` | `INLINE_SETXID_SYSCALL` 宏 |
