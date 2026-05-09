# pthread 线程库深度分析

> 基于 glibc 2.43.9000 NPTL (Native POSIX Threads Library) 源码分析

---

## 文档索引

| 文档 | 内容 | 关键主题 |
|------|------|----------|
| [01-pthread_create.md](01-pthread_create.md) | 线程创建 | 栈分配、TCB初始化、clone系统调用、子线程启动 |
| [02-pthread_mutex.md](02-pthread_mutex.md) | 互斥锁 | 四种类型、futex机制、健壮锁、优先级继承 |
| [03-pthread_cond.md](03-pthread_cond.md) | 条件变量 | 双组机制、wait/signal/broadcast算法、取消处理 |
| [04-pthread_cancel_signal.md](04-pthread_cancel_signal.md) | 取消与信号 | 状态机、延迟/异步取消、清理函数、unwind |
| [05-pthread_spinlock_rwlock.md](05-pthread_spinlock_rwlock.md) | 自旋锁与读写锁 | CAS自旋、读写偏好、写者交接、三futex设计 |
| [06-pthread_barrier.md](06-pthread_barrier.md) | 屏障同步 | 轮次模型、无锁设计、即时重用、溢出保护 |
| [07-pthread_tsd_once.md](07-pthread_tsd_once.md) | TSD与一次性初始化 | 键序列号、两级索引、析构函数、fork安全 |
| [08-pthread_sem_misc.md](08-pthread_sem_misc.md) | 信号量与杂项API | sem_wait/post、atfork、线程命名、CPU亲和性 |
| [09-AArch64线程本地存储.md](09-AArch64线程本地存储.md) | TLS 深度分析 | TCB/DTV布局、tpidr_el0、__tls_get_addr、TLSDESC、TLS分配 |
| [10-futex深入分析.md](10-futex深入分析.md) | Futex 深度分析 | 三态锁协议、操作语义、各原语使用模式、PI/Robust、超时处理 |
| [11-线程栈空间布局.md](11-线程栈空间布局.md) | 栈空间布局 | 初始栈argv/envp/auxv、mmap栈布局、TCB定位、guard page、栈金丝雀、栈缓存复用 |
| [12-pthread_cancel深度分析.md](12-pthread_cancel深度分析.md) | 取消机制深度分析 | cancelhandling 状态机、SIGCANCEL 信号、可取消系统调用桥（AArch64 汇编）、栈展开、NOCANCEL 变体、与 Bionic 对比 |
| [13-pthread_cleanup深度分析.md](13-pthread_cleanup深度分析.md) | 清理处理器深度分析 | 三套宏路径（C++ RAII/GCC cleanup/setjmp）、双链架构、注册/展开、defer 变体、combined 宏、condvar cleanup、C++ 交互 |

---

## NPTL 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    POSIX Threads API                          │
│  pthread_create/join/detach/exit/cancel/kill/sigmask         │
├─────────────────────────────────────────────────────────────┤
│                    同步原语                                   │
│  ┌────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌──────┐ │
│  │ Mutex  │ │ Condvar  │ │ Rwlock  │ │ Barrier │ │ Spin │ │
│  └───┬────┘ └────┬─────┘ └───┬─────┘ └───┬─────┘ └──┬───┘ │
├──────┼───────────┼───────────┼───────────┼──────────┼──────┤
│      │           │           │           │          │      │
│      ▼           ▼           ▼           ▼          ▼      │
│  ┌──────────────────────────────────────────┐   ┌───────┐  │
│  │         Futex 系统调用                    │   │  CAS  │  │
│  │  futex_wait / futex_wake                 │   │(纯用户)│  │
│  │  futex_lock_pi / futex_unlock_pi         │   └───────┘  │
│  └──────────────────────────────────────────┘              │
├─────────────────────────────────────────────────────────────┤
│              struct pthread (TCB)                             │
│  线程控制块: TLS、栈信息、取消状态、健壮锁列表等             │
├─────────────────────────────────────────────────────────────┤
│              Linux Kernel                                     │
│  clone / futex / tgkill / set_robust_list / rseq            │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心设计原则

1. **快速路径优先**: 无竞争时在用户态完成，不进入内核
2. **Futex 作为慢速路径**: 竞争时才使用 futex 系统调用阻塞
3. **TLS 集成**: struct pthread 即线程指针，零开销访问线程数据
4. **栈缓存复用**: 退出线程的栈通过 madvise 回收物理页后缓存
5. **信号安全**: 内部使用 SIGCANCEL/SIGSETXID，用户不可屏蔽

---

## 关键源文件

| 文件 | 内容 |
|------|------|
| nptl/descr.h | struct pthread 定义 |
| nptl/pthread_create.c | 线程创建 |
| nptl/allocatestack.c | 栈分配与缓存 |
| nptl/pthread_mutex_lock.c | 互斥锁加锁 |
| nptl/pthread_mutex_unlock.c | 互斥锁解锁 |
| nptl/pthread_cond_wait.c | 条件变量等待 |
| nptl/pthread_cond_common.c | 条件变量公共逻辑 |
| nptl/pthread_rwlock_common.c | 读写锁核心算法 |
| nptl/pthread_spin_lock.c | 自旋锁 |
| nptl/pthread_cancel.c | 取消机制 |
| nptl/cancellation.c | syscall 取消桥 |
| nptl/unwind.c | 取消展开 |
| sysdeps/nptl/lowlevellock.h | futex 封装 |
| sysdeps/nptl/bits/thread-shared-types.h | 同步原语内部结构 |
