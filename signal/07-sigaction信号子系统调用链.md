# sigaction / signal 信号子系统调用链

> 基于 glibc 2.43 源码，使用 clangd LSP 进行精确调用层次分析。

---

## 1. 概述

glibc 信号子系统涉及三大方面：

| 方面 | 核心函数 | 系统调用 |
|------|---------|---------|
| **信号注册** | sigaction, signal | rt_sigaction |
| **信号发送** | raise, kill, pthread_kill | tgkill, kill |
| **信号等待** | sigsuspend, sigwait, sigtimedwait | rt_sigsuspend, rt_sigtimedwait |
| **信号掩码** | sigprocmask, pthread_sigmask | rt_sigprocmask |
| **内部信号** | SIGCANCEL, SIGSETXID | tgkill (内部) |

---

## 2. sigaction — 信号注册调用链

### 2.1 用户接口层

```
sigaction(sig, act, oact)               [signal/sigaction.c:26]
│  weak_alias: __sigaction → sigaction
│
├── 1. 参数校验
│   ├── sig <= 0 || sig >= NSIG → EINVAL
│   └── is_internal_signal(sig) → EINVAL
│         检查是否为 SIGCANCEL(32) 或 SIGSETXID(33)
│         禁止用户修改内部信号         [sysdeps/unix/sysv/linux/internal-signals.h:51]
│
├── 2. abort 锁保护 (仅 SIGABRT)
│   └── sig == SIGABRT:
│       __abort_lock_wrlock(&set)       [stdlib/abort.c:60]
│         防止与 abort() 的信号重置竞争
│
├── 3. __libc_sigaction(sig, act, oact)
│      调用底层实现                     [sysdeps/unix/sysv/linux/libc_sigaction.c:42]
│
└── 4. sig == SIGABRT:
    __abort_lock_unlock(&set)           [stdlib/abort.c:67]
```

### 2.2 底层实现

```
__libc_sigaction(sig, act, oact)        [sysdeps/unix/sysv/linux/libc_sigaction.c:42]
│
├── [构造 kernel_sigaction]
│   if (act != NULL) {
│     kact.k_sa_handler = act->sa_handler
│     memcpy(&kact.sa_mask, &act->sa_mask, sizeof(sigset_t))
│     kact.sa_flags = act->sa_flags
│     SET_SA_RESTORER(&kact, act)       ← AArch64: 保留 sa_restorer
│   }
│
├── INLINE_SYSCALL_CALL(rt_sigaction, sig, &kact, &koact, __NSIG_BYTES)
│     ← 系统调用：注册信号处理器
│     内核将 handler、mask、flags 保存到 task_struct->sighand
│
└── [提取旧处理器]
    if (oact != NULL && result >= 0) {
      oact->sa_handler = koact.k_sa_handler
      memcpy(&oact->sa_mask, &koact.sa_mask, sizeof(sigset_t))
      oact->sa_flags = koact.sa_flags
      RESET_SA_RESTORER(oact, &koact)
    }
```

### 2.3 AArch64 SA_RESTORER

```
[sysdeps/unix/sysv/linux/aarch64/libc_sigaction.c]

#define SA_RESTORER  0x04000000

#define SET_SA_RESTORER(kact, act)
  if ((kact)->sa_flags & SA_RESTORER)
    (kact)->sa_restorer = (act)->sa_restorer

#define RESET_SA_RESTORER(act, kact)
  (act)->sa_restorer = (kact)->sa_restorer
```

**AArch64 注意**：
- AArch64 原生不需要 SA_RESTORER（内核通过 vDSO 提供 `__kernel_rt_sigreturn`）
- 保留 SA_RESTORER 仅为 AArch32 兼容层（运行 32 位进程）
- 纯 AArch64 程序的信号返回通过 vDSO 自动完成

### 2.4 signal() — 简化接口

```
signal(sig, handler)                    [signal/signal.c / sysv_signal.c]
└── sigaction(sig, &new_act, &old_act)
      new_act = {
        .sa_handler = handler,
        .sa_mask = 空集,
        .sa_flags = SA_RESTART           ← BSD 语义（不重置）
      }
      return old_act.sa_handler
```

---

## 3. sigprocmask / pthread_sigmask — 掩码操作

### 3.1 调用链

```
sigprocmask(how, set, oldset)           [sysdeps/unix/sysv/linux/sigprocmask.c:23]
└── __pthread_sigmask(how, set, oldset) [nptl/pthread_sigmask.c:24]
      │
      ├── [内部信号过滤]
      │   if (set != NULL) {
      │     local_copy = *set
      │     clear_internal_signals(&local_copy)
      │       ← 清除 SIGCANCEL(bit 32) 和 SIGSETXID(bit 33)
      │       ← 防止用户解除对内部信号的阻塞
      │     set = &local_copy
      │   }
      │
      └── INTERNAL_SYSCALL_CALL(rt_sigprocmask, how, set, oldset, __NSIG_BYTES)
            ← 系统调用：修改线程信号掩码
```

**关键设计**：用户永远无法通过 `sigprocmask` 或 `pthread_sigmask` 解除对内部信号的阻塞。
这确保 `SIGCANCEL`/`SIGSETXID` 只在 glibc 预期的时刻可达。

---

## 4. raise / pthread_kill — 信号发送

### 4.1 raise

```
raise(sig)                              [sysdeps/posix/raise.c:24]
├── __pthread_self()                    [nptl/pthread_self.c:22]
│     获取当前线程 ID
└── __pthread_kill(self, sig)           [nptl/pthread_kill.c:93]
```

### 4.2 pthread_kill

```
__pthread_kill(threadid, signo)         [nptl/pthread_kill.c:93]
├── is_internal_signal(signo)           [sysdeps/unix/sysv/linux/internal-signals.h:51]
│     signo == SIGCANCEL || SIGSETXID → EINVAL
│     禁止用户通过 pthread_kill 发送内部信号
└── __pthread_kill_internal(threadid, signo)
                                        [nptl/pthread_kill.c:84]
    └── __pthread_kill_implementation(th, signo, NO_RESULT_TID)
                                        [nptl/pthread_kill.c:28]
```

### 4.3 __pthread_kill_implementation

```
__pthread_kill_implementation(th, signo, tid_p)
                                        [nptl/pthread_kill.c:28]
├── internal_signal_block_all(&set)
│     阻塞所有信号（防止信号中断原子检查）
│                                       [sysdeps/unix/sysv/linux/internal-signals.h:79]
│
├── __libc_lock_lock(th->siglock)       [获取线程信号锁]
│
├── [检查线程是否仍然活动]
│   pid = __getpid()                    [include/unistd.h:119]
│   tid = th->tid (原子读取)
│   if (tid > 0 && pid == th->pid) {
│     if (signo > 0)
│       ret = INTERNAL_SYSCALL_CALL(tgkill, pid, tid, signo)
│                                       ← tgkill: 线程精确投递
│     if (tid_p != NULL) *tid_p = tid
│   } else {
│     ret = ESRCH                       ← 线程已退出
│   }
│
├── __libc_lock_unlock(th->siglock)
│
└── internal_signal_restore_set(&set)
      恢复信号掩码                     [sysdeps/unix/sysv/linux/internal-signals.h:87]
```

**关键安全措施**：
1. 阻塞所有信号防止中断
2. 持有 `siglock` 期间检查 `tid > 0`（线程可能正在退出）
3. 使用 `tgkill`（而非 `kill`）确保信号到达指定线程

---

## 5. 内部信号机制

### 5.1 SIGCANCEL — 线程取消

```
[安装：pthread_create.c 中 __libc_sigaction]
sigcancel_handler(sig, si, ctx)         [nptl/pthread_cancel.c:33]
├── __getpid()
│     验证信号来自自己的进程            [防止外部进程伪造]
├── cancel_enabled_and_canceled_and_async()
│     检查取消状态                      [nptl/descr.h:454]
├── cancellation_pc_check(ctx)
│     检查当前 PC 是否在可取消区间内   [sysdeps/nptl/cancellation-pc-check.h:43]
│     (syscall_cancel 汇编包围的代码区)
└── __syscall_do_cancel()               [nptl/cancellation.c:85]
      设置取消标志并触发栈展开
```

**SIGCANCEL 发送路径** (`pthread_cancel`):
```
pthread_cancel(th)                      [nptl/pthread_cancel.c:101]
├── 设置 cancelhandling CANCELED 位（原子 CAS）
└── __pthread_kill_internal(th, SIGCANCEL)
      → tgkill(pid, tid, 32)
```

### 5.2 SIGSETXID — 凭据广播

```
__nptl_setxid(cmdp)                     [nptl/nptl_setxid.c:175]
│  用于 setuid/setgid 等需要所有线程同步执行的操作
│
├── __libc_lock_lock(GL(dl_stack_cache_lock))
│     获取线程列表锁
│
├── [Phase 1: 标记所有线程]
│   for each thread in __stack_used + __stack_user:
│     setxid_mark_thread(th, cmdp)      [nptl/nptl_setxid.c:97]
│       ← 设置 setxid_futex 为 0（等待完成标记）
│
├── [Phase 2: 发送信号]
│   for each thread in __stack_used + __stack_user:
│     setxid_signal_thread(th, cmdp)    [nptl/nptl_setxid.c:154]
│       └── tgkill(pid, tid, SIGSETXID)
│
├── [Phase 3: 等待所有线程完成]
│   for each thread:
│     while (th->setxid_futex == 0)
│       futex_wait_simple(&th->setxid_futex, 0)
│                                       [sysdeps/nptl/futex-internal.h:154]
│
├── [Phase 4: 取消标记]
│   for each thread:
│     setxid_unmark_thread(th, cmdp)    [nptl/nptl_setxid.c:134]
│
└── __libc_lock_unlock(GL(dl_stack_cache_lock))
```

**SIGSETXID 处理器**：
```
__nptl_setxid_sighandler(sig, si, ctx)  [nptl/nptl_setxid.c:56]
├── __getpid()                          [验证来源]
├── INTERNAL_SYSCALL(cmdp->syscall_no, cmdp->id)
│     在本线程中执行凭据变更系统调用
│     (如 setuid, setgid, setgroups)
├── setxid_error(cmdp, error)           [记录错误]
│                                       [nptl/nptl_setxid.c:30]
├── th->setxid_futex = 1               [标记完成]
└── futex_wake(&th->setxid_futex, 1)   [唤醒等待者]
                                        [sysdeps/nptl/futex-internal.h:187]
```

---

## 6. 信号等待调用链

### 6.1 sigsuspend

```
sigsuspend(set)                         [sysdeps/unix/sysv/linux/sigsuspend.c:24]
└── SYSCALL_CANCEL(rt_sigsuspend, set, __NSIG_BYTES)
      ← 可取消点：挂起直到有未阻塞信号到达
      ← 使用 SYSCALL_CANCEL 宏，支持 pthread_cancel
```

### 6.2 sigwait

```
sigwait(set, sig)                       [sysdeps/unix/sysv/linux/sigwait.c:23]
└── __sigtimedwait(set, &info, NULL)    [include/signal.h:47]
      └── __sigtimedwait64(set, info, NULL)
                                        [sysdeps/unix/sysv/linux/sigtimedwait.c:22]
          └── SYSCALL_CANCEL(rt_sigtimedwait_time64, set, info, NULL, __NSIG_BYTES)
                ← 等待 set 中任意信号到达
```

### 6.3 sigtimedwait

```
sigtimedwait(set, info, timeout)        [sysdeps/unix/sysv/linux/sigtimedwait.c:70]
└── __sigtimedwait64(set, info, &ts64)
      └── SYSCALL_CANCEL(rt_sigtimedwait_time64, set, info, timeout, __NSIG_BYTES)
            ← 可取消点
            ← 支持 64 位时间戳（Y2038 安全）
```

---

## 7. 信号返回 — sigreturn 机制

### 7.1 AArch64 信号帧结构

信号处理器返回时，内核需要恢复被中断线程的上下文：

```
信号处理器栈帧（由内核在投递信号时构造）：

┌─────────────────────────────────┐  高地址
│ 128 bytes 保留区                 │  ← sp + 总大小
├─────────────────────────────────┤
│ __aux_context (可选，RT 扩展)    │
├─────────────────────────────────┤
│ struct rt_sigframe:              │
│   ├── siginfo_t info             │  ← 信号信息 (SA_SIGINFO 时)
│   ├── struct ucontext_t:         │
│   │   ├── uc_flags               │
│   │   ├── uc_link                │
│   │   ├── uc_stack (sigaltstack) │
│   │   ├── uc_sigmask             │
│   │   └── uc_mcontext:           │
│   │       ├── fault_address      │
│   │       ├── regs[31] (x0-x30) │
│   │       ├── sp                 │
│   │       ├── pc                 │
│   │       └── pstate             │
│   └── __reserved[]:             │  ← 扩展上下文区域
│       ├── fpsimd_context         │  (NEON/FP 寄存器)
│       ├── sve_context (可选)     │  (SVE 向量寄存器)
│       ├── za_context (可选)      │  (SME 矩阵)
│       ├── zt_context (可选)      │  (SME ZT0)
│       └── 终止标记 (size=0)      │
├─────────────────────────────────┤
│ 返回地址 → __kernel_rt_sigreturn │  ← 在 vDSO 中
└─────────────────────────────────┘  低地址 (sp)
```

### 7.2 sigreturn 路径

```
信号处理器执行完毕
  │
  ▼
__kernel_rt_sigreturn()                 [内核 vDSO 中]
  │  由栈帧中的返回地址自动调用
  │  等价于:
  │    mov x8, #__NR_rt_sigreturn (139)
  │    svc #0
  │
  ▼
[内核 rt_sigreturn 系统调用]
  ├── 从用户栈帧中恢复 uc_mcontext:
  │   ├── x0-x30 通用寄存器
  │   ├── sp, pc, pstate
  │   ├── FPSIMD/NEON 寄存器
  │   ├── SVE 向量寄存器 (若有)
  │   └── PSTATE 中的 DIT/SSBS 等位
  ├── 恢复信号掩码 (uc_sigmask)
  └── 跳转到保存的 pc
        → 回到被中断的代码位置继续执行
```

### 7.3 glibc 中的 sigreturn 辅助

```
[glibc 不直接实现 sigreturn — 它在 vDSO/内核中]

但 glibc 提供:
- getcontext/setcontext/makecontext  (用户态上下文切换)
- sigreturn stub (某些架构，AArch64 不需要)

sysdeps/unix/sysv/linux/sigreturn.c:    stub only (AArch64 不使用)
```

---

## 8. 信号安装时序

### 8.1 SIGCANCEL/SIGSETXID 安装

```
[在第一次 pthread_cancel 调用时懒安装]

pthread_cancel()                        [nptl/pthread_cancel.c:101]
  ├── static once: 
  │   struct sigaction sa = {
  │     .sa_sigaction = sigcancel_handler,
  │     .sa_flags = SA_SIGINFO | SA_RESTART,
  │   }
  │   __sigaddset(&sa.sa_mask, SIGCANCEL)  ← 阻塞自身
  │   __libc_sigaction(SIGCANCEL, &sa, NULL)
  │     ← 使用 __libc_sigaction 绕过 is_internal_signal 检查
  │
  └── 同时在 nptl 初始化时:
      __libc_sigaction(SIGSETXID, &sa_setxid, NULL)
        ← sa_setxid.sa_sigaction = __nptl_setxid_sighandler
```

### 8.2 pthread_create 中的掩码处理

```
pthread_create()
  └── pthread_create_2_1()              [nptl/pthread_create.c]
      ├── sigaddset(&pd->sigmask, SIGCANCEL)
      │     确保新线程阻塞 SIGCANCEL
      │     （只在取消请求时才临时解除阻塞）
      └── internal_sigdelset(&pd->sigmask, SIGCANCEL)  [line 821]
            实际上确保新线程在 start_thread 中
            正确继承信号掩码
```

---

## 9. 完整信号处理时序

```
用户调用 sigaction(SIGUSR1, &my_handler, NULL)
  │
  ▼
__sigaction()
  ├── is_internal_signal(SIGUSR1) → No (不是32/33)
  └── __libc_sigaction(SIGUSR1, ...)
      └── rt_sigaction syscall
          → 内核记录: task->sighand->action[SIGUSR1] = my_handler

... 稍后某线程收到 SIGUSR1 ...

[内核信号投递]
  ├── 保存当前上下文到用户栈 (rt_sigframe)
  ├── 设置栈帧返回地址 = vDSO __kernel_rt_sigreturn
  ├── 设置 PC = my_handler
  ├── 设置 x0 = signo, x1 = &siginfo, x2 = &ucontext (SA_SIGINFO)
  └── 返回用户态执行 my_handler

my_handler(signo, info, ctx) 执行
  │
  └── return (函数返回到栈帧中的返回地址)
      │
      ▼
__kernel_rt_sigreturn() [vDSO]
  └── svc #0 (rt_sigreturn)
      │
      ▼
[内核]
  ├── 从 rt_sigframe 恢复寄存器
  ├── 恢复信号掩码
  └── 恢复 PC → 回到被中断代码
```

---

## 10. kill / killpg / tgkill 信号发送

```
kill(pid, sig)
  └── INLINE_SYSCALL_CALL(kill, pid, sig)
        ← kill syscall: 发送到进程（任意线程接收）

killpg(pgrp, sig)
  └── kill(-pgrp, sig)
        ← 发送到整个进程组

pthread_kill(thread, sig)
  └── ... → tgkill(pid, tid, sig)
        ← tgkill syscall: 精确发送到指定线程

tkill(tid, sig)  [已废弃，被 tgkill 替代]
```

---

## 11. 关键设计总结

### 11.1 三层信号防护

```
┌─────────────────────────────────────────────┐
│          用户层保护 (__sigaction)             │
│  · is_internal_signal 拒绝修改 32/33        │
│  · abort_lock 保护 SIGABRT 一致性           │
├─────────────────────────────────────────────┤
│          掩码层保护 (__pthread_sigmask)       │
│  · clear_internal_signals 过滤内部信号      │
│  · 用户永远无法 unblock SIGCANCEL/SIGSETXID │
├─────────────────────────────────────────────┤
│          发送层保护 (__pthread_kill)          │
│  · 拒绝用户发送 SIGCANCEL/SIGSETXID         │
│  · tgkill + pid 验证 防止线程 TID 复用攻击  │
│  · siglock 保护 线程存活性检查原子性         │
└─────────────────────────────────────────────┘
```

### 11.2 可取消系统调用

信号等待函数使用 `SYSCALL_CANCEL` 宏：
```c
#define SYSCALL_CANCEL(name, args...)
  ({
    // 检查是否在取消点
    int oldtype = LIBC_CANCEL_ASYNC();
    long result = INTERNAL_SYSCALL_CALL(name, args);
    LIBC_CANCEL_RESET(oldtype);
    result;
  })
```

这确保 `sigsuspend`、`sigwait`、`sigtimedwait` 都是取消点。

### 11.3 与其他子系统的交互

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  取消子系统   │────→│  SIGCANCEL   │←────│ pthread_kill │
│pthread_cancel│     │  (sig 32)    │     │              │
└──────────────┘     └──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  凭据子系统   │────→│  SIGSETXID   │←────│ __nptl_setxid│
│ setuid/gid   │     │  (sig 33)    │     │ futex 广播   │
└──────────────┘     └──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  abort()     │────→│  SIGABRT     │←────│ __sigaction  │
│ abort_lock_wr│     │  SIG_DFL强制 │     │ abort_lock_wr│
└──────────────┘     └──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐
│  内核 vDSO   │────→│ rt_sigreturn │← 信号处理器返回
│ sigreturn桩  │     │ 恢复上下文   │
└──────────────┘     └──────────────┘
```

---

## 12. 涉及的源文件

| 源文件 | 内容 |
|--------|------|
| `signal/sigaction.c` | `__sigaction` 入口，内部信号过滤 + abort 锁 |
| `sysdeps/unix/sysv/linux/libc_sigaction.c` | `__libc_sigaction` 系统调用封装 |
| `sysdeps/unix/sysv/linux/aarch64/libc_sigaction.c` | AArch64 SA_RESTORER 定义 |
| `nptl/pthread_sigmask.c` | `__pthread_sigmask` 内部信号清除 |
| `sysdeps/unix/sysv/linux/sigprocmask.c` | `__sigprocmask` → pthread_sigmask |
| `sysdeps/posix/raise.c` | `raise` → pthread_kill(self) |
| `nptl/pthread_kill.c` | `__pthread_kill` 完整实现（tgkill） |
| `nptl/pthread_cancel.c` | `sigcancel_handler` + SIGCANCEL 安装 |
| `nptl/nptl_setxid.c` | `__nptl_setxid` + SIGSETXID 广播协议 |
| `sysdeps/unix/sysv/linux/sigsuspend.c` | `sigsuspend` SYSCALL_CANCEL |
| `sysdeps/unix/sysv/linux/sigwait.c` | `sigwait` → sigtimedwait |
| `sysdeps/unix/sysv/linux/sigtimedwait.c` | `sigtimedwait` 64位时间支持 |
| `sysdeps/unix/sysv/linux/internal-signals.h` | 内部信号定义与操作宏 |
| `sysdeps/unix/sysv/linux/kernel_sigaction.h` | kernel_sigaction 结构转换 |

---

> 本文档基于 clangd 调用层次分析生成，覆盖 glibc 2.43 信号子系统的注册、发送、等待、返回完整路径。
