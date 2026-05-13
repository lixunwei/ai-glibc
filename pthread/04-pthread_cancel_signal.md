# pthread_cancel 与信号处理深度分析

## 概述

线程取消是 POSIX 定义的协作式线程终止机制。glibc 使用内部信号 `SIGCANCEL`
配合位图状态机实现延迟取消和异步取消。本文档同时覆盖 `pthread_kill` 和
`pthread_sigmask` 的多线程信号处理。

---

## 第一部分: 线程取消 (Cancellation)

---

## 1. 取消状态机

### 1.1 状态位图 (cancelhandling)

存储在 `struct pthread` 的 `cancelhandling` 字段 (descr.h:295-316):

```
位图: [... | SETXID | TERMINATED | EXITING | CANCELING | CANCELED | TYPE | STATE]

bit 0: CANCELSTATE_BITMASK   — 1=取消禁用, 0=取消使能
bit 1: CANCELTYPE_BITMASK    — 1=异步取消, 0=延迟取消
bit 2: CANCELED_BITMASK      — 1=已收到取消请求
bit 3: CANCELING_BITMASK     — 1=正在执行取消
bit 4: EXITING_BITMASK       — 1=正在退出
bit 5: TERMINATED_BITMASK    — 1=已终止
bit 6: SETXID_BITMASK        — setxid 同步标志
```

### 1.2 状态转换图

```
                 pthread_cancel()
                       │
                       ▼
┌─────────┐     ┌──────────┐     ┌───────────┐
│ ENABLED │────→│ CANCELED │────→│ CANCELING │────→ 线程终止
│ (默认)  │     │ (已请求) │     │ (执行中)  │
└─────────┘     └──────────┘     └───────────┘
                       │
              取消点触发 / 异步触发
```

### 1.3 判定宏

```c
cancel_enabled(ch)                    // 取消是否使能
cancel_async_enabled(ch)              // 是否异步模式
cancel_enabled_and_canceled(ch)       // 使能且已被取消
cancel_enabled_and_canceled_and_async(ch)  // 使能+已取消+异步
```

---

## 2. pthread_cancel 内部流程

```
pthread_cancel(target_thread)           // pthread_cancel.c:58-154
    │
    ├── 1. 检查目标线程是否已退出
    │       if (joinstate == EXITED) return 0
    │
    ├── 2. 首次调用: 安装 SIGCANCEL handler
    │       sigaction(SIGCANCEL, sigcancel_handler, SA_SIGINFO|SA_RESTART)
    │
    ├── 3. CAS 设置 CANCELED_BITMASK
    │       atomic_compare_exchange(&target->cancelhandling, old | CANCELED)
    │
    ├── 4. 判断是否自取消
    │       if (target == self && 异步模式) {
    │           __do_cancel(PTHREAD_CANCELED)  // 立即取消
    │       }
    │
    └── 5. 发送 SIGCANCEL 给目标线程
            __pthread_kill_internal(target, SIGCANCEL)
```

---

## 3. 延迟取消 (Deferred Cancellation)

### 3.1 取消点

POSIX 规定的取消点函数（部分）:
- I/O: `read`, `write`, `open`, `close`, `recv`, `send`, `accept`...
- 等待: `sleep`, `usleep`, `nanosleep`, `poll`, `select`, `sigwait`
- 同步: `pthread_cond_wait`, `pthread_join`, `sem_wait`
- 其他: `system`, `popen`, `dlopen`

### 3.2 取消点实现机制

glibc 使用 **syscall cancel bridge** 实现取消点:

```c
// nptl/cancellation.c:22-64
__internal_syscall_cancel(...)
    │
    ├── 进入 cancellable 区域（标记 PC 范围）
    │
    ├── 执行 syscall
    │
    └── 退出 cancellable 区域
```

### 3.3 SIGCANCEL handler 中的检查

```c
// pthread_cancel.c:31-56 (sigcancel_handler)
if (信号来自本进程 && SI_TKILL) {
    if (异步取消模式) {
        __syscall_do_cancel();
    }
    if (PC 在 cancellable syscall bridge 内) {
        __syscall_do_cancel();
    }
}
```

### 3.4 PC 范围判定

```c
// sysdeps/nptl/cancellation-pc-check.h:24-52
// 通过信号上下文中的 PC 判断是否在可取消的 syscall 执行中
if (cancel_pc >= __syscall_cancel_arch_start &&
    cancel_pc < __syscall_cancel_arch_end) {
    // 在取消点中，可以安全取消
}
```

---

## 4. 异步取消 (Asynchronous Cancellation)

### 4.1 启用方式

```c
pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, &oldtype);
```

### 4.2 触发时机

- 调用 `pthread_setcanceltype` 时: 若已被标记取消，立即触发
- 收到 `SIGCANCEL` 时: handler 检测到异步模式，立即取消
- 调用 `pthread_testcancel` 时

### 4.3 危险性

异步取消可能在**任意指令**处中断线程:
- 可能中断 malloc（导致堆损坏）
- 可能中断 I/O（导致资源泄露）
- **只在纯计算代码中安全使用**

---

## 5. 清理处理函数 (Cleanup Handlers)

### 5.1 使用方式

```c
pthread_cleanup_push(cleanup_func, arg);
// ... 可能被取消的代码 ...
pthread_cleanup_pop(execute);  // execute=1 则无条件执行
```

### 5.2 内部实现 (libc-cleanup.c:23-75)

**push**:
1. 保存当前清理链表头到 buffer 的 `__prev`
2. 若当前是异步取消模式，临时切换为延迟模式
3. 记录原始 canceltype

**pop**:
1. 恢复链表头
2. 恢复原始 canceltype
3. 若恢复后是异步模式且已被标记取消 → 立即触发取消

### 5.3 取消时的清理执行

```
__do_cancel(PTHREAD_CANCELED)
    │
    └── __pthread_unwind()
            │
            └── _Unwind_ForcedUnwind()
                    │
                    ├── unwind_stop() 逐帧回溯
                    │       执行每一层的 cleanup handler
                    │
                    └── 到达 setjmp 保存点
                            longjmp 回取消入口
                            线程退出
```

---

## 6. 取消展开机制 (Forced Unwind)

### 6.1 实现 (nptl/unwind.c:38-151)

1. `__pthread_unwind` 调用 `_Unwind_ForcedUnwind`
2. 运行时按调用栈逐帧回溯
3. 每帧检查是否有注册的清理函数
4. 执行所有清理函数（LIFO 顺序）
5. 到达 `start_thread` 中 setjmp 保存的展开点
6. 通过 longjmp 跳回，进入线程退出路径

### 6.2 与 C++ 异常的关系

- forced unwind 会穿过 C++ catch blocks
- C++ catch(...) 必须重新抛出 forced unwind 异常
- 否则行为未定义

---

## 7. pthread_testcancel

```c
// nptl/pthread_testcancel.c:23-29
void pthread_testcancel(void) {
    if (cancel_enabled_and_canceled(cancelhandling))
        __do_cancel(PTHREAD_CANCELED);
}
```

这是显式取消点 — 不涉及系统调用的代码可以用它主动检查取消请求。

---

## 第二部分: 信号处理

---

## 8. pthread_kill

### 8.1 实现 (nptl/pthread_kill.c:24-123)

```
pthread_kill(thread, signo)
    │
    ├── 1. 校验: 内部信号(SIGCANCEL/SIGSETXID)返回 EINVAL
    │
    ├── 2. 同线程情况:
    │       tgkill(getpid(), gettid(), signo)  // 同步送达
    │
    └── 3. 其他线程:
            ├── 阻塞所有信号
            ├── 加锁 pd->exit_lock（防止线程退出竞态）
            ├── 检查线程是否 exiting
            │   └── 是 → 返回 0（仅兼容符号 __pthread_kill_esrch 返回 ESRCH）
            ├── tgkill(pid, pd->tid, signo)
            └── 释放锁，恢复信号
```

### 8.2 竞态保护

为什么需要 `exit_lock`:
- 线程退出时 TID 被内核回收
- 新线程可能复用相同 TID
- 不加锁可能误发信号给错误线程
- 加锁期间阻塞所有信号防止死锁

---

## 9. pthread_sigmask

### 9.1 实现 (nptl/pthread_sigmask.c:24-46)

```c
int __pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset) {
    sigset_t local;
    if (set != NULL && __glibc_unlikely (__sigismember (set, SIGCANCEL)
                       || __sigismember (set, SIGSETXID))) {
        local = *set;
        clear_internal_signals(&local);  // 移除 SIGCANCEL, SIGSETXID
        set = &local;
    }
    return INTERNAL_SYSCALL_CALL (rt_sigprocmask, how, set, oldset, __NSIG_BYTES);
}
```

### 9.2 关键规则

- **SIGCANCEL 和 SIGSETXID 不可被用户屏蔽**
- 即使用户在集合中包含这些信号，glibc 会自动清除
- 这保证了取消和 setxid 机制始终可用

### 9.3 per-thread 语义

- `pthread_sigmask` / `sigprocmask` 操作的是**当前线程的信号掩码**
- Linux 内核为每个线程维护独立的信号掩码
- 信号分发规则:
  - 线程定向信号 (tgkill): 送给指定线程
  - 进程信号 (kill): 送给任意一个未屏蔽该信号的线程

---

## 10. 信号与取消的交互

### 10.1 SIGCANCEL 的特殊性

| 属性 | 值 |
|------|-----|
| 信号号 | `__SIGRTMIN` (通常 32) |
| handler | `sigcancel_handler` |
| 标志 | `SA_SIGINFO | SA_RESTART` |
| 来源检查 | 必须是 `SI_TKILL` 且来自本进程 |

### 10.2 系统调用中断

某些系统调用即使有 `SA_RESTART` 也会被 SIGCANCEL 中断:
- 这是设计使然: 使得阻塞在系统调用中的线程能被取消
- glibc 只在安全时发送 SIGCANCEL（目标处于取消使能状态）

### 10.3 sigwait 的多线程模式

推荐的多线程信号处理模式:

```c
// 主线程
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGINT);
sigaddset(&set, SIGTERM);
pthread_sigmask(SIG_BLOCK, &set, NULL);  // 所有线程继承此掩码

// 专用信号处理线程
void *signal_thread(void *arg) {
    sigset_t set = *(sigset_t *)arg;
    int sig;
    while (1) {
        sigwait(&set, &sig);  // 同步等待
        handle_signal(sig);
    }
}
```

### 10.4 sigwait 实现细节

```c
// sysdeps/unix/sysv/linux/sigwait.c:22-36
int sigwait(const sigset_t *set, int *sig) {
    do {
        ret = __sigtimedwait(set, &si, NULL);
    } while (ret == -1 && errno == EINTR);  // 重试，不暴露 EINTR
    *sig = si.si_signo;
    return 0;
}
```

- `sigwait` 循环重试 EINTR（应用不期望看到 EINTR）
- `sigtimedwait` 是取消点（通过 SYSCALL_CANCEL 包装）

---

## 11. 源码位置速查

| 内容 | 文件:行号 |
|------|-----------|
| cancelhandling 位图 | descr.h:295-316 |
| pthread_cancel 入口 | pthread_cancel.c:58-154 |
| sigcancel_handler | pthread_cancel.c:31-56 |
| pthread_setcancelstate | pthread_setcancelstate.c |
| pthread_setcanceltype | pthread_setcanceltype.c:24-59 |
| pthread_testcancel | pthread_testcancel.c:23-29 |
| syscall cancel bridge | cancellation.c:22-64 |
| PC 范围检查 | cancellation-pc-check.h:24-52 |
| __syscall_do_cancel | cancellation.c:84-110 |
| cleanup push/pop | libc-cleanup.c:23-75 |
| forced unwind | unwind.c:38-151 |
| pthread_kill | pthread_kill.c:24-123 |
| pthread_sigmask | pthread_sigmask.c:24-46 |
| sigwait | sigwait.c:22-36 |
| sigtimedwait | sigtimedwait.c:22-84 |
