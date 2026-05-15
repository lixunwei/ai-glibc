# 信号子系统深度分析

## 概述

信号（Signal）是 Unix/Linux 进程间通信和异常处理的基本机制。glibc 的信号子系统
在内核提供的原始系统调用之上增加了：
- 用户态数据结构转换（`struct sigaction` ↔ `struct kernel_sigaction`）
- 信号返回跳板（`__restore_rt`）的自动安装
- NPTL 线程安全（保护内部信号 SIGCANCEL/SIGSETXID）
- 信号集的位域操作优化

## 架构图

```
┌───────────────────────────────────────────────────────────┐
│                    应用程序                                 │
├───────────────────────────────────────────────────────────┤
│  signal()  sigaction()  sigprocmask()  sigwait()  kill()  │
├───────────────────────────────────────────────────────────┤
│                glibc 信号层                                 │
│  ┌──────────────────────────────────────────────────┐     │
│  │  signal/sigaction.c     — 入参验证 + abort 锁     │     │
│  │  libc_sigaction.c       — struct 转换 + syscall   │     │
│  │  x86_64/libc_sigaction.c — SA_RESTORER 跳板安装   │     │
│  │  nptl/pthread_sigmask.c — 内部信号过滤            │     │
│  │  sigsetops.h            — 位域操作                │     │
│  └──────────────────────────────────────────────────┘     │
├───────────────────────────────────────────────────────────┤
│  rt_sigaction  rt_sigprocmask  rt_sigsuspend  tgkill ...  │
├───────────────────────────────────────────────────────────┤
│                    Linux 内核                               │
│  ┌──────────────────────────────────────────────────┐     │
│  │  信号投递 → 用户栈构建信号帧 → 跳转 sa_handler   │     │
│  │  sa_handler 返回 → __restore_rt → rt_sigreturn   │     │
│  └──────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────┘
```

## NPTL 内部保留信号

| 信号 | 值 | 用途 |
|------|-----|------|
| `SIGCANCEL` | `__SIGRTMIN` (32) | 线程取消 + POSIX 定时器 |
| `SIGSETXID` | `__SIGRTMIN + 1` (33) | 全线程 setuid/setgid 同步 |

用户不可操作这两个信号：`sigaction()`/`pthread_sigmask()` 自动过滤。

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-信号基础与数据结构.md](01-信号基础与数据结构.md) | sigset_t、struct sigaction、文件布局、syscall 对照 |
| [02-sigaction与信号注册.md](02-sigaction与信号注册.md) | sigaction 完整调用链、signal()、SA_RESTORER 跳板 |
| [03-信号掩码与等待.md](03-信号掩码与等待.md) | sigprocmask、pthread_sigmask、sigsuspend、sigwait |
| [04-高级特性与线程交互.md](04-高级特性与线程交互.md) | sigaltstack、实时信号、pthread_kill、SIGSETXID、abort |
| [05-AArch64信号帧与投递返回.md](05-AArch64信号帧与投递返回.md) | 信号帧布局、SA_RESTORER、vDSO 跳板、栈回溯、getcontext/setcontext |
| [06-SIGCANCEL与SIGSETXID内部信号.md](06-SIGCANCEL与SIGSETXID内部信号.md) | 信号定义与保留、用户屏蔽、cancelhandling 状态机、取消流程、凭据广播协议 |
| [07-sigaction信号子系统调用链.md](07-sigaction信号子系统调用链.md) | **调用链**：sigaction 注册→raise/pthread_kill 投递→sigreturn 返回、SIGCANCEL/SIGSETXID 完整路径 |

## 源文件布局

```
signal/                  — 通用 API 入口 + 公共头文件
├── sigaction.c          — __sigaction 验证层
├── sigprocmask.c        — 通用 stub（Linux 下被覆盖）
├── signal.c             — signal() 通用 stub
├── sigwait.c            — sigwait() 循环包装
├── sigempty.c           — sigemptyset 公共入口
├── sigfillset.c         — sigfillset 公共入口
├── sigaddset.c          — sigaddset 公共入口
├── sigdelset.c          — sigdelset 公共入口
├── sigismem.c           — sigismember 公共入口
├── allocrtsig.c         — 实时信号分配管理
└── sigvec.c             — BSD sigvec 兼容

sysdeps/unix/sysv/linux/
├── libc_sigaction.c     — __libc_sigaction (rt_sigaction syscall)
├── sigprocmask.c        — __sigprocmask → pthread_sigmask
├── sigsuspend.c         — rt_sigsuspend syscall
├── sigpending.c         — rt_sigpending syscall
├── sigtimedwait.c       — rt_sigtimedwait syscall
├── sigsetops.h          — 位域操作 inline 实现
├── internal-signals.h   — SIGCANCEL/SIGSETXID 定义
├── internal-sigset.h    — 内部信号集操作
├── kernel_sigaction.h   — struct kernel_sigaction
├── bits/sigaction.h     — struct sigaction 布局
└── x86_64/
    └── libc_sigaction.c — SA_RESTORER + __restore_rt 跳板

nptl/
├── pthread_sigmask.c    — 线程安全的信号掩码
├── pthread_kill.c       — pthread_kill (tgkill)
├── pthread_sigqueue.c   — pthread_sigqueue
├── pthread_cancel.c     — SIGCANCEL 处理器安装
└── nptl_setxid.c        — SIGSETXID 处理器
```
