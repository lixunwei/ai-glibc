# sigaction 与信号注册

## 一、sigaction() 完整调用链

```
sigaction(sig, act, oact)                [用户调用]
  → __sigaction(sig, act, oact)          [signal/sigaction.c:25-47]
    │  验证: sig > 0 && sig < NSIG && !is_internal_signal(sig)
    │  SIGABRT 特殊处理: 加写锁防止 abort() 竞争
    │
    → __libc_sigaction(sig, act, oact)   [sysdeps/unix/sysv/linux/libc_sigaction.c:41-72]
      │  struct sigaction → struct kernel_sigaction 转换
      │  SET_SA_RESTORER: 安装信号返回跳板
      │
      → INLINE_SYSCALL_CALL(rt_sigaction, sig, &kact, &koact, __NSIG_BYTES)
                                          [进入内核]
```

---

## 二、__sigaction — 验证层

**源文件**: `signal/sigaction.c:25-47`

```c
int __sigaction(int sig, const struct sigaction *act, struct sigaction *oact)
{
    // 1. 参数验证
    if (sig <= 0 || sig >= NSIG || is_internal_signal(sig)) {
        __set_errno(EINVAL);
        return -1;
    }

    // 2. SIGABRT 特殊保护
    //    防止 abort() 正在执行时修改 SIGABRT 处理器
    internal_sigset_t set;
    if (sig == SIGABRT)
        __abort_lock_wrlock(&set);   // 阻塞所有信号 + 加写锁

    // 3. 调用实际的系统调用包装
    int r = __libc_sigaction(sig, act, oact);

    if (sig == SIGABRT)
        __abort_lock_unlock(&set);   // 释放锁 + 恢复信号

    return r;
}
weak_alias(__sigaction, sigaction)
```

**关键设计点**:
- 拒绝内部信号（SIGCANCEL=32, SIGSETXID=33）
- SIGABRT 的 abort 锁确保 `abort()` 期间处理器不被修改

---

## 三、__libc_sigaction — 系统调用包装

**源文件**: `sysdeps/unix/sysv/linux/libc_sigaction.c:41-72`

```c
int __libc_sigaction(int sig, const struct sigaction *act, struct sigaction *oact)
{
    struct kernel_sigaction kact, koact;

    // 1. 用户结构 → 内核结构
    if (act) {
        kact.k_sa_handler = act->sa_handler;
        memcpy(&kact.sa_mask, &act->sa_mask, sizeof(sigset_t));
        kact.sa_flags = (unsigned int)act->sa_flags;
        SET_SA_RESTORER(&kact, act);  // 架构相关: 安装跳板
    }

    // 2. 系统调用
    result = INLINE_SYSCALL_CALL(rt_sigaction, sig,
                                 act ? &kact : NULL,
                                 oact ? &koact : NULL,
                                 __NSIG_BYTES);

    // 3. 内核结构 → 用户结构
    if (oact && result >= 0) {
        oact->sa_handler = koact.k_sa_handler;
        memcpy(&oact->sa_mask, &koact.sa_mask, sizeof(sigset_t));
        oact->sa_flags = koact.sa_flags;
        RESET_SA_RESTORER(oact, &koact);
    }
    return result;
}
```

---

## 四、SA_RESTORER 与信号返回跳板

### 问题

内核投递信号时，将信号帧压入用户栈后跳转到 `sa_handler`。当处理器函数返回时，
需要调用 `rt_sigreturn` 系统调用来恢复进程上下文。谁来执行这个 syscall？

### 解决方案：glibc 安装 restorer 跳板

**源文件**: `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:19-27`

```c
#define SA_RESTORER 0x04000000

extern void restore_rt(void) asm("__restore_rt") attribute_hidden;

#define SET_SA_RESTORER(kact, act)        \
    (kact)->sa_flags |= SA_RESTORER;     \
    (kact)->sa_restorer = &restore_rt
```

每次 `sigaction()` 调用都会：
1. 设置 `SA_RESTORER` 标志
2. 将 `sa_restorer` 指向 `__restore_rt` 函数

### __restore_rt 实现

**源文件**: `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:70-134`

```asm
__restore_rt:
    movq  $__NR_rt_sigreturn, %rax    # syscall 号 = 15
    syscall                            # 进入内核，恢复完整上下文
```

这段代码只有 2 条指令，但附带了完整的 DWARF unwind 信息（`.eh_frame`），
使得 GDB 和 libunwind 能正确回溯穿过信号帧的调用栈。

### 信号投递与返回时序

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 进程执行中 → 信号到达                                     │
│                                                             │
│ 2. 内核中断进程，在用户栈上构建信号帧:                        │
│    ┌─────────────────────────┐                              │
│    │ ucontext_t (保存寄存器)  │ ← RSP                       │
│    │ siginfo_t (信号信息)     │                              │
│    │ 返回地址 = sa_restorer   │                              │
│    └─────────────────────────┘                              │
│                                                             │
│ 3. 内核设置 RIP = sa_handler，跳转到用户态                   │
│                                                             │
│ 4. 用户信号处理器执行                                        │
│                                                             │
│ 5. 处理器返回 → ret 指令弹出栈顶 → 跳到 __restore_rt        │
│                                                             │
│ 6. __restore_rt 执行 rt_sigreturn syscall                   │
│                                                             │
│ 7. 内核从 ucontext_t 恢复所有寄存器，进程继续原来的执行       │
└─────────────────────────────────────────────────────────────┘
```

### GDB 注意事项

源码注释（第 35-50 行）强调：
- `__restore_rt` 的函数名和指令序列不可修改
- GDB 通过识别 `__restore_rt` 名称来标识信号帧
- unwind 信息将信号帧中的 `ucontext_t` 映射为 CFA

---

## 五、signal() — 简单信号 API

### BSD 语义（glibc 默认）

```c
// signal/signal.h:84-87
// glibc 默认使用 BSD 语义:
//   SA_RESTART: 被中断的 syscall 自动重启
//   不设置 SA_RESETHAND: 处理器持久有效
```

### 等价 sigaction 调用

```c
signal(sig, handler)
≡
struct sigaction act = {
    .sa_handler = handler,
    .sa_flags = SA_RESTART,   // BSD: 自动重启
    .sa_mask = <空>
};
sigaction(sig, &act, &old);
return old.sa_handler;
```

### SysV 语义

```c
// sysv_signal(sig, handler) 等价于:
struct sigaction act = {
    .sa_handler = handler,
    .sa_flags = SA_RESETHAND | SA_NODEFER,  // 一次性 + 不阻塞
    .sa_mask = <空>
};
```

SysV 语义的问题：
- `SA_RESETHAND`: 处理器执行后恢复为 SIG_DFL（不安全）
- `SA_NODEFER`: 处理器执行期间不阻塞自身信号（可重入风险）

---

## 六、sigvec — BSD 兼容

**源文件**: `signal/sigvec.c:61-114`

```c
// BSD sigvec → sigaction 标志转换
SV_INTERRUPT → 不设 SA_RESTART（中断不重启）
SV_RESETHAND → SA_RESETHAND
SV_ONSTACK   → SA_ONSTACK
```

---

## 七、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `signal/sigaction.c:25-47` | `__sigaction` 验证 + abort 锁 |
| `sysdeps/unix/sysv/linux/libc_sigaction.c:41-72` | `__libc_sigaction` syscall 包装 |
| `sysdeps/unix/sysv/linux/kernel_sigaction.h:8-19` | `struct kernel_sigaction` |
| `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:19-27` | SET_SA_RESTORER 宏 |
| `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:70-134` | `__restore_rt` 跳板 + unwind |
| `signal/signal.h:84-99` | signal() 语义声明 |
| `signal/sigvec.c:61-114` | BSD sigvec 兼容 |
