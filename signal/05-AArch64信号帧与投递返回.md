# AArch64 信号帧与信号投递/返回机制

## 概述

当 Linux 内核向进程投递信号时，需要在用户栈上构建一个**信号帧 (signal frame)**，
保存被中断的寄存器上下文，然后跳转到用户注册的信号处理函数。处理函数返回后，
通过**信号返回跳板**执行 `rt_sigreturn` 系统调用，内核据此恢复原始上下文。

本文分析 AArch64 平台上 glibc 对这一完整流程的支持：
- 信号帧数据结构 (`kernel_rt_sigframe`, `ucontext_t`, `mcontext_t`)
- `sigaction` 注册路径与 SA_RESTORER 机制
- 信号返回跳板: AArch64 依赖内核 vDSO，x86_64 依赖 glibc `__restore_rt`
- 栈回溯/异常展开对信号帧的识别
- getcontext/setcontext/makecontext 上下文操作

---

## 一、信号帧数据结构

### 1.1 kernel_rt_sigframe

**源文件**: `sysdeps/unix/sysv/linux/aarch64/kernel_rt_sigframe.h:21-25`

```c
struct kernel_rt_sigframe
{
  siginfo_t info;       // 信号信息 (si_signo, si_code, si_value 等)
  ucontext_t uc;        // 被中断时的完整处理器上下文
};
```

内核在投递信号时，将此结构压入用户栈顶。

### 1.2 ucontext_t — 用户上下文

**源文件**: `sysdeps/unix/sysv/linux/aarch64/sys/ucontext.h:67-74`

```c
typedef struct ucontext_t {
    unsigned long   uc_flags;       // 标志位
    struct ucontext_t *uc_link;     // 后继上下文（用于 makecontext）
    stack_t         uc_stack;       // 备用信号栈信息 (ss_sp, ss_size, ss_flags)
    sigset_t        uc_sigmask;     // 信号处理时的掩码
    mcontext_t      uc_mcontext;    // 机器上下文（寄存器 + 扩展状态）
} ucontext_t;
```

### 1.3 mcontext_t — 机器上下文

**源文件**: `sysdeps/unix/sysv/linux/aarch64/sys/ucontext.h:52-64`

```c
typedef struct {
    unsigned long long fault_address;       // 故障地址
    unsigned long long regs[31];            // 通用寄存器 x0-x30
    unsigned long long sp;                  // 栈指针
    unsigned long long pc;                  // 程序计数器
    unsigned long long pstate;              // 处理器状态寄存器 (NZCV 等)
    unsigned char __reserved[4096]          // 扩展块 (FP/SIMD, SVE, GCS 等)
        __attribute__((__aligned__(16)));
} mcontext_t;
```

### 1.4 扩展块 (__reserved) 布局

`__reserved` 是一个 4096 字节的区域，存储可变长度的扩展状态。
每个扩展块以 `struct _aarch64_ctx` 头部开始：

```
__reserved[4096]:
  ┌───────────────────────────────────┐
  │ struct _aarch64_ctx {             │
  │   __u32 magic;   // 魔术数        │
  │   __u32 size;    // 块总大小       │
  │ }                                 │
  ├───────────────────────────────────┤
  │ struct fpsimd_context {           │  ← FPSIMD_MAGIC (0x46508001)
  │   head: {magic, size}             │
  │   fpsr, fpcr                      │
  │   vregs[32]  (128位 SIMD 寄存器)  │
  │ }                                 │
  ├───────────────────────────────────┤
  │ struct gcs_context (可选) {       │  ← GCS_MAGIC (0x47435300)
  │   head: {magic, size}             │
  │   gcspr (GCS 栈指针)              │
  │   ...                             │
  │ }                                 │
  ├───────────────────────────────────┤
  │ terminator: {magic=0, size=0}     │  ← 终止标记
  └───────────────────────────────────┘
```

**偏移量定义**: `sysdeps/unix/sysv/linux/aarch64/ucontext_i.sym:37-55`

```
oX0       = mcontext(regs)          // 通用寄存器数组起始
oSP       = mcontext(sp)            // 栈指针偏移
oPC       = mcontext(pc)            // 程序计数器偏移
oPSTATE   = mcontext(pstate)        // 状态寄存器偏移
oEXTENSION = mcontext(__reserved)   // 扩展块起始偏移
oV0       = fpsimd_context(vregs)   // SIMD 寄存器数组偏移
oFPSR     = fpsimd_context(fpsr)    // FPSR 偏移
oFPCR     = fpsimd_context(fpcr)    // FPCR 偏移
```

---

## 二、信号帧在用户栈上的布局

```
信号投递时的用户栈:

  高地址
  ┌──────────────────────────────────────┐
  │  被中断程序的栈帧                      │
  ├──────────────────────────────────────┤
  │  padding (16 字节对齐)                │
  ├──────────────────────────────────────┤ ← 信号帧起始
  │  siginfo_t info                      │  ~128 字节
  │    si_signo, si_errno, si_code       │
  │    si_value (union sigval)           │
  ├──────────────────────────────────────┤
  │  ucontext_t uc                       │
  │    uc_flags                          │
  │    uc_link                           │
  │    uc_stack (stack_t)                │
  │    uc_sigmask (sigset_t)             │
  │    uc_mcontext:                      │
  │      fault_address                   │
  │      regs[31] (x0-x30)              │
  │      sp, pc, pstate                  │
  │      __reserved[4096]:               │
  │        fpsimd_context                │
  │        [gcs_context]                 │
  │        terminator {0,0}              │
  ├──────────────────────────────────────┤ ← SP 在信号处理函数中
  │                                      │
  低地址

  信号处理函数的参数:
    x0 = signum
    x1 = &info   (SA_SIGINFO 时)
    x2 = &uc     (SA_SIGINFO 时)
    LR = __kernel_rt_sigreturn (vDSO 中的返回地址)
```

---

## 三、sigaction 注册与 SA_RESTORER

### 3.1 AArch64 的 libc_sigaction

**源文件**: `sysdeps/unix/sysv/linux/aarch64/libc_sigaction.c:18-30`

```c
/* Required for AArch32 compatibility. */
#define SA_RESTORER  0x04000000

#define SET_SA_RESTORER(kact, act)           \
  ({                                         \
    if ((kact)->sa_flags & SA_RESTORER)       \
      (kact)->sa_restorer = (act)->sa_restorer; \
  })

#define RESET_SA_RESTORER(act, kact)         \
  (act)->sa_restorer = (kact)->sa_restorer;

#include <sysdeps/unix/sysv/linux/libc_sigaction.c>
```

**关键差异**: AArch64 的 `SET_SA_RESTORER` **不会主动设置** SA_RESTORER 标志位！
它只在用户已经显式设置了 SA_RESTORER 的情况下才传递 `sa_restorer`。
这仅用于 AArch32 兼容程序。

### 3.2 通用 Linux libc_sigaction

**源文件**: `sysdeps/unix/sysv/linux/libc_sigaction.c:41-72`

```c
int __libc_sigaction(int sig, const struct sigaction *act,
                     struct sigaction *oact)
{
  struct kernel_sigaction kact, koact;
  
  if (act) {
    kact.k_sa_handler = act->sa_handler;       // 复制处理函数
    memcpy(&kact.sa_mask, &act->sa_mask, ...); // 复制信号掩码
    kact.sa_flags = (unsigned int) act->sa_flags;
    SET_SA_RESTORER(&kact, act);               // 架构相关的跳板安装
  }
  
  result = INLINE_SYSCALL_CALL(rt_sigaction, sig,
    act ? &kact : NULL, oact ? &koact : NULL, __NSIG_BYTES);
  
  if (oact && result >= 0) {
    oact->sa_handler = koact.k_sa_handler;
    memcpy(&oact->sa_mask, &koact.sa_mask, ...);
    oact->sa_flags = koact.sa_flags;
    RESET_SA_RESTORER(oact, &koact);
  }
  return result;
}
```

### 3.3 kernel_sigaction 结构

**源文件**: `sysdeps/unix/sysv/linux/kernel_sigaction.h:1-28`

```c
struct kernel_sigaction {
  __sighandler_t k_sa_handler;  // 信号处理函数
  unsigned long sa_flags;       // 标志位
#ifdef HAS_SA_RESTORER
  void (*sa_restorer)(void);    // 信号返回跳板（仅 SA_RESTORER 架构有）
#endif
  sigset_t sa_mask;             // 阻塞信号集
};
```

---

## 四、信号返回跳板: AArch64 vs x86_64

### 4.1 AArch64: 内核/vDSO 提供跳板

在 AArch64 上，glibc **不提供** `__restore_rt`。内核在投递信号时，将返回地址
（LR 寄存器）设置为 vDSO 中的 `__kernel_rt_sigreturn`：

```asm
// 内核 vDSO 中的 __kernel_rt_sigreturn (AArch64)
// 这是固定的两条指令:
__kernel_rt_sigreturn:
    movz  x8, #0x8b          // __NR_rt_sigreturn = 139 = 0x8b
    svc   #0                 // 执行系统调用
```

当信号处理函数返回 (`ret`) 时，由于 LR 指向此跳板，处理器跳转到这里，
执行 `rt_sigreturn` 系统调用，内核从信号帧恢复原始上下文。

### 4.2 x86_64: glibc 提供 __restore_rt

**源文件**: `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:19-134`

```c
// x86_64 定义 SA_RESTORER 并强制设置
extern void restore_rt(void) asm("__restore_rt") attribute_hidden;

#define SET_SA_RESTORER(kact, act)           \
  (kact)->sa_flags |= SA_RESTORER;          \  // 强制加上标志
  (kact)->sa_restorer = &restore_rt            // 安装 glibc 跳板

// 跳板代码 (x86_64 libc_sigaction.c:134):
// RESTORE(restore_rt, __NR_rt_sigreturn)
// 展开为:
__restore_rt:
    movq $__NR_rt_sigreturn, %rax
    syscall
```

x86_64 的跳板还附带了精心构建的 DWARF unwind 信息 (`.eh_frame`)，
使 GDB 和 libunwind 能正确回溯穿越信号帧。

### 4.3 对比总结

| 特性 | AArch64 | x86_64 |
|------|---------|--------|
| 信号返回跳板提供者 | 内核 vDSO (`__kernel_rt_sigreturn`) | glibc (`__restore_rt`) |
| SA_RESTORER 标志 | 不主动设置，仅 AArch32 兼容 | 强制设置 |
| `__restore_rt` 存在? | 否 | 是 |
| 跳板中的 unwind 信息 | 由内核/vDSO 提供 | glibc 手工编写 .eh_frame |
| kernel_sigaction 中 sa_restorer | 仅兼容路径 | 必须存在 |
| 系统调用号 | `__NR_rt_sigreturn = 139 (0x8b)` | `__NR_rt_sigreturn = 15` |

---

## 五、信号帧栈回溯与异常展开

### 5.1 SFrame 展开器

**源文件**: `sysdeps/unix/sysv/linux/aarch64/uw-sigframe.h:37-66`

```c
static _Unwind_Reason_Code
aarch64_decode_signal_frame(frame *frame) {
    unsigned int *pc = (unsigned int *) frame->pc;
    
    // PC 必须 4 字节对齐
    if ((frame->pc & 3) != 0)
        return _URC_END_OF_STACK;
    
    // 识别 __kernel_rt_sigreturn 的指令模式:
    //   0xd2801168  movz x8, #0x8b
    //   0xd4000001  svc  0x0
    if (pc[0] != MOVZ_X8_8B || pc[1] != SVC_0)
        return _URC_END_OF_STACK;      // 不是信号帧
    
    // 从信号帧提取上下文
    struct kernel_rt_sigframe *rt_ =
        (struct kernel_rt_sigframe *) frame->sp;
    mcontext_t *mt = &rt_->uc.uc_mcontext;
    
    // 恢复 PC、SP、FP
    frame->pc = (_Unwind_Ptr) mt->pc;
    frame->sp = (_Unwind_Ptr) mt->sp;
    frame->fp = (_Unwind_Ptr) mt->regs[30];  // x30 = FP_REGNUM
    return _URC_NO_REASON;
}
```

常量定义 (同文件 27-33 行):
```c
#ifdef __AARCH64EL__               // 小端
#define MOVZ_X8_8B  0xd2801168
#define SVC_0       0xd4000001
#else                               // 大端
#define MOVZ_X8_8B  0x681180d2
#define SVC_0       0x010000d4
#endif
```

### 5.2 sigcontext_get_pc

**源文件**: `sysdeps/unix/sysv/linux/aarch64/sigcontextinfo.h:25-29`

```c
static inline uintptr_t
sigcontext_get_pc(const ucontext_t *ctx) {
    return ctx->uc_mcontext.pc;
}
```

### 5.3 栈回溯穿越信号帧的完整流程

```
正常调用帧回溯:
  frame N: FP → frame N-1: FP → ...

遇到信号帧时:
  frame N: FP → (信号帧边界)
    ↓
  展开器检测到 PC 指向 __kernel_rt_sigreturn
    ↓
  从 SP 定位 kernel_rt_sigframe
    ↓
  从 rt_sigframe.uc.uc_mcontext 读取:
    - pc → 被中断时的 PC
    - sp → 被中断时的 SP
    - regs[30] → 被中断时的 FP
    ↓
  继续从被中断位置回溯
```

---

## 六、getcontext / setcontext / makecontext

这组函数实现用户态上下文切换，使用与信号帧相同的 `ucontext_t` 结构。

### 6.1 getcontext — 保存当前上下文

**源文件**: `sysdeps/unix/sysv/linux/aarch64/getcontext.S:32-123`

```
__getcontext(ucp):
  // ① 保存通用寄存器 (仅 callee-saved: x18-x30)
  str  xzr, [x0, oX0]                // x0 = 0（恢复后返回 0）
  stp  x18, x19, [x0, oX0 + 18*8]
  stp  x20, x21, [x0, oX0 + 20*8]
  ...
  str  x30, [x0, oX0 + 30*8]         // LR
  
  // ② 保存 PC = LR (恢复后从调用点继续)
  str  x30, [x0, oPC]
  
  // ③ 保存 SP
  mov  x2, sp
  str  x2, [x0, oSP]
  
  // ④ 清零 pstate
  str  xzr, [x0, oPSTATE]
  
  // ⑤ 保存 FP/SIMD 状态 (扩展块)
  写入 FPSIMD_MAGIC 头部
  stp  q8, q9, ...                    // 保存 callee-saved SIMD: q8-q15
  mrs  x4, fpsr / fpcr
  
  // ⑥ 保存 GCS 状态（如果支持）
  CHKFEAT → MRS_GCSPR → 写入 GCS_MAGIC 块
  
  // ⑦ 写入终止标记 {magic=0, size=0}
  
  // ⑧ 获取信号掩码
  rt_sigprocmask(SIG_BLOCK, NULL, &ucp->uc_sigmask, _NSIG/8)
  
  return 0
```

### 6.2 setcontext — 恢复上下文

**源文件**: `sysdeps/unix/sysv/linux/aarch64/setcontext.S:36-163`

```
__setcontext(ucp):
  // ① 恢复信号掩码
  rt_sigprocmask(SIG_SETMASK, &ucp->uc_sigmask, NULL, _NSIG/8)
  
  // ② 清除 SME ZA 状态
  CALL_LIBC_ARM_ZA_DISABLE
  
  // ③ 恢复通用寄存器 (x18-x30, SP)
  ldp  x18, x19, [x0, oX0 + 18*8]
  ...
  ldr  x30, [x0, oX0 + 30*8]
  ldr  x2, [x0, oSP]
  mov  sp, x2
  
  // ④ 恢复 FP/SIMD 状态
  检查 FPSIMD_MAGIC → ldp q8-q15 → msr fpsr/fpcr
  
  // ⑤ 恢复 GCS 状态（如果有）
  检查 GCS_MAGIC → 扫描 GCS 栈 → GCSSS1/GCSSS2 切换 → GCSPOPM 弹出
  
  // ⑥ 跳转到保存的 PC
  ldr  x16, [x0, oPC]
  ldp  x0, x1, [x0, oX0]             // 恢复参数寄存器
  br   x16                            // 跳转（不是 blr!）
```

### 6.3 makecontext — 构造新上下文

**源文件**: `sysdeps/unix/sysv/linux/aarch64/makecontext.c:61-105`

```c
void __makecontext(ucontext_t *ucp, void (*func)(void), int argc, ...) {
    uint64_t *sp = (uint64_t *)
        ((uintptr_t) ucp->uc_stack.ss_sp + ucp->uc_stack.ss_size);
    
    // 为超过 8 个的参数分配栈空间
    sp -= argc < 8 ? 0 : argc - 8;
    sp = (uint64_t *)(((uintptr_t)sp) & -16L);  // 16 字节对齐
    
    // 设置寄存器
    ucp->uc_mcontext.regs[19] = (uintptr_t) ucp->uc_link;  // 后继上下文
    ucp->uc_mcontext.regs[20] = (uintptr_t) func;           // 目标函数
    ucp->uc_mcontext.sp = (uintptr_t) sp;
    ucp->uc_mcontext.pc = (uintptr_t) __startcontext;       // 入口点
    ucp->uc_mcontext.regs[29] = 0;  // FP = 0 (栈底标记)
    ucp->uc_mcontext.regs[30] = 0;  // LR = 0
    
    // 设置参数: x0-x7 → 直接到寄存器, 其余 → 栈
    va_start(ap, argc);
    for (i = 0; i < argc; i++)
        if (i < 8)
            ucp->uc_mcontext.regs[i] = va_arg(ap, uint64_t);
        else
            sp[i - 8] = va_arg(ap, uint64_t);
    va_end(ap);
}
```

### 6.4 __startcontext — 上下文启动函数

**源文件**: `sysdeps/unix/sysv/linux/aarch64/setcontext.S:167-173`

```asm
__startcontext:
    blr   x20              // 调用目标函数 (x20 = func)
    mov   x0, x19          // x19 = uc_link (后继上下文)
    cbnz  x0, __setcontext // 有后继? → 切换到后继上下文
    b     exit             // 无后继 → exit()
```

---

## 七、信号投递与返回完整时序图

```
应用程序正在执行
     │
     │← 内核投递信号
     ▼
┌─────────────────────────────────────────────────┐
│  内核: do_signal()                               │
│  ① 在用户栈分配 kernel_rt_sigframe               │
│  ② 保存当前寄存器到 uc_mcontext                  │
│     regs[0-30], sp, pc, pstate                   │
│  ③ 保存 FP/SIMD/SVE/GCS 到 __reserved           │
│  ④ 设置信号处理函数参数:                          │
│     x0 = signum                                  │
│     x1 = &siginfo  (SA_SIGINFO)                  │
│     x2 = &ucontext (SA_SIGINFO)                  │
│  ⑤ 设置返回地址:                                  │
│     LR = __kernel_rt_sigreturn (vDSO)            │
│  ⑥ PC = sa_handler (用户注册的处理函数)           │
│  ⑦ 应用信号掩码 (sa_mask + 当前信号)              │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
        用户信号处理函数执行 (sa_handler)
                      │
                      │ ret (返回到 LR)
                      ▼
┌─────────────────────────────────────────────────┐
│  __kernel_rt_sigreturn (vDSO):                   │
│    movz  x8, #0x8b     // __NR_rt_sigreturn      │
│    svc   #0            // 陷入内核                │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  内核: sys_rt_sigreturn()                        │
│  ① 从用户栈读取 kernel_rt_sigframe               │
│  ② 从 uc_mcontext 恢复寄存器                     │
│  ③ 恢复 FP/SIMD/SVE/GCS 状态                    │
│  ④ 恢复信号掩码                                   │
│  ⑤ 返回到被中断的 PC 继续执行                     │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
        应用程序从被中断处继续执行
```

---

## 八、sigaltstack 与信号栈选择

当用户通过 `sigaltstack` 注册了备用信号栈，且 `sigaction` 设置了 `SA_ONSTACK`，
内核会将信号帧放在备用栈上：

```
正常情况:
  信号帧放在被中断线程的用户栈顶

SA_ONSTACK + sigaltstack:
  信号帧放在 ss_sp ~ ss_sp+ss_size 的备用栈上
  
  ┌───────────────────────────┐ ← ss_sp + ss_size
  │  kernel_rt_sigframe       │
  │    siginfo_t              │
  │    ucontext_t             │
  ├───────────────────────────┤ ← 信号处理函数的 SP
  │  信号处理函数的栈帧        │
  │  ...                      │
  └───────────────────────────┘ ← ss_sp
```

---

## 九、AArch64 特有: GCS (Guarded Control Stack)

GCS 是 ARMv9 的控制流完整性特性，为信号帧引入了额外复杂度：

1. **内核投递信号时**: 在 GCS 栈上压入 cap token，保存 GCSPR 到 `__reserved`
2. **信号返回时**: 内核验证 cap token，恢复 GCSPR
3. **getcontext/setcontext**: 需要保存/恢复 GCS 状态

**getcontext 中的 GCS 处理** (`getcontext.S:88-100`):
```asm
    CHKFEAT_X16              // 检查 GCS 是否可用
    tbnz x16, 0, gcs_done   // 不可用则跳过
    写入 GCS_MAGIC 头部
    MRS_GCSPR(x4)           // 读取当前 GCS 指针
    add x4, x4, 8           // 调整到 getcontext 返回后的状态
    str x4, [x2, #oGCSPR]   // 保存
```

**setcontext 中的 GCS 恢复** (`setcontext.S:116-151`):
```
    读取目标 GCSPR
    扫描 GCS 栈找到 cap token
    GCSSS1/GCSSS2 切换到目标 GCS
    GCSPOPM 弹出多余的条目
```

---

## 十、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `sysdeps/unix/sysv/linux/aarch64/kernel_rt_sigframe.h:21-25` | `kernel_rt_sigframe` 结构定义 |
| `sysdeps/unix/sysv/linux/aarch64/sys/ucontext.h:52-64` | `mcontext_t` (regs/sp/pc/pstate/__reserved) |
| `sysdeps/unix/sysv/linux/aarch64/sys/ucontext.h:67-74` | `ucontext_t` (flags/link/stack/sigmask/mcontext) |
| `sysdeps/unix/sysv/linux/aarch64/ucontext_i.sym:37-55` | 偏移量常量 (oX0/oSP/oPC/oPSTATE/oEXTENSION) |
| `sysdeps/unix/sysv/linux/aarch64/libc_sigaction.c:18-30` | AArch64 SA_RESTORER (仅 AArch32 兼容) |
| `sysdeps/unix/sysv/linux/libc_sigaction.c:41-72` | 通用 __libc_sigaction (struct 转换 + syscall) |
| `sysdeps/unix/sysv/linux/kernel_sigaction.h:1-28` | kernel_sigaction 结构（条件 sa_restorer） |
| `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c:19-134` | x86_64 __restore_rt 跳板 + DWARF unwind |
| `sysdeps/unix/sysv/linux/aarch64/uw-sigframe.h:27-66` | 信号帧展开器 (识别 movz+svc 指令模式) |
| `sysdeps/unix/sysv/linux/aarch64/sigcontextinfo.h:25-29` | `sigcontext_get_pc` (读取 mcontext.pc) |
| `sysdeps/unix/sysv/linux/aarch64/getcontext.S:32-123` | getcontext (保存 GPR+SIMD+GCS+信号掩码) |
| `sysdeps/unix/sysv/linux/aarch64/setcontext.S:36-163` | setcontext (恢复掩码+GPR+SIMD+GCS→br x16) |
| `sysdeps/unix/sysv/linux/aarch64/setcontext.S:167-173` | __startcontext (makecontext 启动函数) |
| `sysdeps/unix/sysv/linux/aarch64/makecontext.c:61-105` | makecontext (构造上下文: regs/sp/pc+参数) |
| `sysdeps/unix/sysv/linux/aarch64/arch-syscall.h` | `__NR_rt_sigreturn = 139` |
