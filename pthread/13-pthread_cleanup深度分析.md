# pthread cleanup 机制深度分析

> 基于 glibc 2.43.9000 源码，深入分析线程清理处理器（cleanup handler）的注册、
> 链接、展开和执行全过程。

---

## 目录

1. [概述与设计目标](#1-概述与设计目标)
2. [核心数据结构](#2-核心数据结构)
3. [三套宏展开路径](#3-三套宏展开路径)
4. [注册与反注册实现](#4-注册与反注册实现)
5. [defer 变体：保存/恢复取消类型](#5-defer-变体保存恢复取消类型)
6. [start_thread 根帧与 setjmp 跳板](#6-start_thread-根帧与-setjmp-跳板)
7. [栈展开与 unwind_stop](#7-栈展开与-unwind_stop)
8. [C++ 异常交互](#8-c-异常交互)
9. [libc 内部清理：\_\_libc\_cleanup\_push\_defer](#9-libc-内部清理libc_cleanup_push_defer)
10. [combined 宏：pthread_once 的双重保护](#10-combined-宏pthread_once-的双重保护)
11. [条件变量等待中的 cleanup](#11-条件变量等待中的-cleanup)
12. [退出顺序：cleanup vs TSD vs TLS](#12-退出顺序cleanup-vs-tsd-vs-tls)
13. [Fork 安全性](#13-fork-安全性)
14. [AArch64 展开细节](#14-aarch64-展开细节)
15. [总结：cleanup 帧链全景图](#15-总结cleanup-帧链全景图)

---

## 1. 概述与设计目标

POSIX 要求 `pthread_cleanup_push` / `pthread_cleanup_pop` 提供"try-finally"语义：
当线程被取消或调用 `pthread_exit` 时，已注册的清理函数按 LIFO 顺序执行。

glibc 的实现需要同时满足：

| 需求 | 实现方式 |
|------|----------|
| C 语言无异常的环境 | `setjmp/longjmp` 跳板机制 |
| C++ 异常/RAII 环境 | `__attribute__((cleanup))` 或 C++ 析构函数 |
| 与 `_Unwind_ForcedUnwind` 协作 | 让 C++ 析构函数在取消时也能运行 |
| glibc 内部使用（如 stdio 锁） | 老式 `_pthread_cleanup_buffer` 链表 |

因此，glibc 维护了**两条并行的清理链**，在 `struct pthread` 上表现为两个字段。

---

## 2. 核心数据结构

### 2.1 struct pthread 中的清理字段

```c
/* nptl/descr.h:287-292 */
struct _pthread_cleanup_buffer *cleanup;       /* 老式 cleanup 链 */
struct pthread_unwind_buf *cleanup_jmp_buf;    /* 新式 unwind 帧链 */
```

- **`cleanup`**：传统 `_pthread_cleanup_buffer` 单向链表头，供老式
  `__pthread_cleanup_push/__pthread_cleanup_pop` 和 libc 内部使用。
- **`cleanup_jmp_buf`**：新式 `pthread_unwind_buf` 帧链头，供用户态
  `pthread_cleanup_push` 宏使用，内含 `setjmp` 保存的上下文。

### 2.2 老式清理缓冲区

```c
/* sysdeps/nptl/pthread.h:159-165 */
struct _pthread_cleanup_buffer
{
  void (*__routine) (void *);             /* 清理函数 */
  void *__arg;                            /* 参数 */
  int __canceltype;                       /* 保存的取消类型 */
  struct _pthread_cleanup_buffer *__prev; /* 链接前一个 */
};
```

这是最简单的链表节点。`__canceltype` 字段供 defer 变体保存原取消类型。

### 2.3 新式展开缓冲区

**用户可见的公开定义**：

```c
/* sysdeps/nptl/pthread.h:538-548 */
struct __cancel_jmp_buf_tag
{
  __jmp_buf __cancel_jmp_buf;   /* setjmp 保存的寄存器 */
  int __mask_was_saved;         /* 是否保存了信号掩码 */
};

typedef struct
{
  struct __cancel_jmp_buf_tag __cancel_jmp_buf[1];
  void *__pad[4];               /* ABI 占位，内部复用 */
} __pthread_unwind_buf_t __attribute__ ((__aligned__));
```

**内部真实布局**（`__pad[4]` 被重新解释）：

```c
/* nptl/descr.h:65-92 */
struct pthread_unwind_buf
{
  struct {
    __jmp_buf jmp_buf;
    int mask_was_saved;
  } cancel_jmp_buf[1];

  union {
    void *pad[4];               /* 与公开 ABI 对齐 */
    struct {
      struct pthread_unwind_buf *prev;     /* 前一个帧 */
      struct _pthread_cleanup_buffer *cleanup;  /* 旧链快照 */
      int canceltype;                      /* 保存的取消类型 */
    } data;
  } priv;
};
```

关键设计：`priv.data` 与 `pad[4]` 共用同一块内存。公开头文件只暴露 `pad`，
内部代码通过 `priv.data` 访问链接信息。

### 2.4 双链关系图

```
struct pthread
├── cleanup ──→ [_pthread_cleanup_buffer] → [prev] → ...  (老式链)
│
└── cleanup_jmp_buf ──→ [pthread_unwind_buf] ──→ [priv.data.prev] → ...
                         │                         │
                         ├── cancel_jmp_buf[0]     ├── priv.data.cleanup
                         │   (setjmp 上下文)        │   (老式链快照)
                         │                         └── priv.data.canceltype
                         └── (用于 longjmp 跳回)
```

`unwind_stop` 展开时同时处理两条链：新式帧通过 `_Unwind_ForcedUnwind` 自动
遍历，老式帧在每一步中检查 `cleanup` 并调用。

---

## 3. 三套宏展开路径

glibc 为 `pthread_cleanup_push/pop` 提供了三套完全不同的宏实现，
根据编译环境自动选择：

### 3.1 路径一：C++ 类 RAII（`__EXCEPTIONS && __cplusplus`）

```c
/* sysdeps/nptl/pthread.h:568-600 */
class __pthread_cleanup_class
{
  void (*__cancel_routine) (void *);
  void *__cancel_arg;
  int __do_it;
  int __cancel_type;

 public:
  __pthread_cleanup_class(void (*fct)(void *), void *arg)
    : __cancel_routine(fct), __cancel_arg(arg), __do_it(1) { }
  ~__pthread_cleanup_class()
    { if (__do_it) __cancel_routine(__cancel_arg); }
  void __setdoit(int newval) { __do_it = newval; }
  void __defer()
    { pthread_setcanceltype(PTHREAD_CANCEL_DEFERRED, &__cancel_type); }
  void __restore() const
    { pthread_setcanceltype(__cancel_type, 0); }
};

#define pthread_cleanup_push(routine, arg) \
  do { __pthread_cleanup_class __clframe(routine, arg)

#define pthread_cleanup_pop(execute) \
    __clframe.__setdoit(execute); } while (0)
```

**特点**：纯 RAII，析构函数自动调用清理函数。不使用 setjmp，
不涉及 `cleanup_jmp_buf` 链。依赖 C++ 异常展开机制。

### 3.2 路径二：GCC `__cleanup__` 属性（`__EXCEPTIONS && !__cplusplus`）

```c
/* sysdeps/nptl/pthread.h:638-649 */
#define pthread_cleanup_push(routine, arg) \
  do {                                                    \
    struct __pthread_cleanup_frame __clframe               \
      __attribute__((__cleanup__(__pthread_cleanup_routine))) \
      = { .__cancel_routine = (routine), .__cancel_arg = (arg), \
          .__do_it = 1 };

#define pthread_cleanup_pop(execute) \
    __clframe.__do_it = (execute); } while (0)
```

`__pthread_cleanup_routine` 是一个简单的包装函数：

```c
/* nptl/cleanup_routine.c:22-26 */
void ___pthread_cleanup_routine(struct __pthread_cleanup_frame *f)
{
  if (f->__do_it)
    f->__cancel_routine(f->__cancel_arg);
}
```

**特点**：利用 GCC 的 `__attribute__((cleanup))` 实现自动调用，
当变量离开作用域时编译器自动插入调用。纯 C 代码，不需要 setjmp。

### 3.3 路径三：setjmp/longjmp 跳板（无异常支持时）

```c
/* sysdeps/nptl/pthread.h:681-708 */
#define pthread_cleanup_push(routine, arg) \
  do {                                                    \
    __pthread_unwind_buf_t __cancel_buf;                   \
    void (*__cancel_routine)(void *) = (routine);          \
    void *__cancel_arg = (arg);                            \
    int __not_first_call =                                 \
      __sigsetjmp_cancel(__cancel_buf.__cancel_jmp_buf, 0); \
    if (__glibc_unlikely(__not_first_call))                 \
      {                                                    \
        __cancel_routine(__cancel_arg);                     \
        __pthread_unwind_next(&__cancel_buf);               \
        /* NOTREACHED */                                   \
      }                                                    \
    __pthread_register_cancel(&__cancel_buf);               \
    do {

#define pthread_cleanup_pop(execute) \
      do { } while (0);                                    \
    } while (0);                                           \
    __pthread_unregister_cancel(&__cancel_buf);             \
    if (execute)                                            \
      __cancel_routine(__cancel_arg);                       \
  } while (0)
```

**执行流程**：

```
正常路径:
  sigsetjmp → 返回0 → register_cancel → 用户代码 → unregister → 可选执行

取消路径:
  _Unwind_ForcedUnwind → unwind_stop → longjmp 到 sigsetjmp
  → 返回非0 → 调用 __cancel_routine → __pthread_unwind_next → 继续展开
```

**特点**：最重量级，在栈上分配 `__pthread_unwind_buf_t`（包含完整 `jmp_buf`），
每次 push 都执行一次 `sigsetjmp`。这是没有异常支持时的后备方案。

### 3.4 三路径对比

| 特性 | C++ RAII | GCC cleanup 属性 | setjmp 跳板 |
|------|----------|------------------|-------------|
| 条件 | `__EXCEPTIONS && __cplusplus` | `__EXCEPTIONS && !__cplusplus` | 其他 |
| 栈开销 | 最小（4 字段） | 最小（4 字段） | 最大（含 jmp_buf） |
| C++ 析构 | ✓ 自动 | ✓ 编译器插入 | ✗ 需要 ForcedUnwind |
| setjmp | ✗ | ✗ | ✓ 每次 push 一次 |
| 链表注册 | ✗ 不注册 | ✗ 不注册 | ✓ cleanup_jmp_buf 链 |

---

## 4. 注册与反注册实现

### 4.1 \_\_\_pthread\_register\_cancel

```c
/* nptl/cleanup.c:22-35 */
void ___pthread_register_cancel(__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  struct pthread *self = THREAD_SELF;

  /* 保存旧的帧和清理链头 */
  ibuf->priv.data.prev = THREAD_GETMEM(self, cleanup_jmp_buf);
  ibuf->priv.data.cleanup = THREAD_GETMEM(self, cleanup);

  /* 将新帧设为当前头 */
  THREAD_SETMEM(self, cleanup_jmp_buf, (struct pthread_unwind_buf *) buf);
}
```

**关键操作**：
1. 将当前 `cleanup_jmp_buf` 保存到新帧的 `priv.data.prev` → 形成单向链表
2. 快照当前老式 `cleanup` 链头到 `priv.data.cleanup` → 用于展开时区分新旧帧
3. 将 `self->cleanup_jmp_buf` 指向新帧 → 推入栈顶

### 4.2 \_\_\_pthread\_unregister\_cancel

```c
/* nptl/cleanup.c:46-53 */
void ___pthread_unregister_cancel(__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  THREAD_SETMEM(THREAD_SELF, cleanup_jmp_buf, ibuf->priv.data.prev);
}
```

极其简单：恢复上一帧的指针即可。帧本身在栈上自动回收。

### 4.3 老式 \_\_pthread\_cleanup\_push/\_\_pthread\_cleanup\_pop

供 glibc 内部使用（如 condvar wait）：

```c
/* nptl/cleanup_compat.c:23-33 */
void __pthread_cleanup_push(struct _pthread_cleanup_buffer *buffer,
                            void (*routine)(void *), void *arg)
{
  struct pthread *self = THREAD_SELF;
  buffer->__routine = routine;
  buffer->__arg = arg;
  buffer->__prev = THREAD_GETMEM(self, cleanup);
  THREAD_SETMEM(self, cleanup, buffer);
}

/* nptl/cleanup_compat.c:39-49 */
void __pthread_cleanup_pop(struct _pthread_cleanup_buffer *buffer, int execute)
{
  THREAD_SETMEM(THREAD_SELF, cleanup, buffer->__prev);
  if (execute)
    buffer->__routine(buffer->__arg);
}
```

操作 `cleanup` 链（不是 `cleanup_jmp_buf`），不涉及 setjmp。

---

## 5. defer 变体：保存/恢复取消类型

### 5.1 用户态宏

`pthread_cleanup_push_defer_np` / `pthread_cleanup_pop_restore_np` 是 GNU 扩展。
它们在注册 cleanup 的同时临时切换为延迟取消，pop 时恢复原取消类型。

C++ 路径使用 RAII 类的 `__defer()` / `__restore()` 方法
（`pthread.h:606-617`）。

setjmp 路径使用专门的注册/反注册函数（`pthread.h:716-744`）。

### 5.2 \_\_\_pthread\_register\_cancel\_defer

```c
/* nptl/cleanup_defer.c:22-52 */
void ___pthread_register_cancel_defer(__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  struct pthread *self = THREAD_SELF;

  /* 保存链和旧 cleanup */
  ibuf->priv.data.prev = THREAD_GETMEM(self, cleanup_jmp_buf);
  ibuf->priv.data.cleanup = THREAD_GETMEM(self, cleanup);

  /* 如果当前是异步取消，CAS 切换为延迟取消 */
  int cancelhandling = atomic_load_relaxed(&self->cancelhandling);
  if (__glibc_unlikely(cancelhandling & CANCELTYPE_BITMASK))
    {
      int newval;
      do {
        newval = cancelhandling & ~CANCELTYPE_BITMASK;
      } while (!atomic_compare_exchange_weak_acquire(
                  &self->cancelhandling, &cancelhandling, newval));
    }

  /* 保存原取消类型 */
  ibuf->priv.data.canceltype = (cancelhandling & CANCELTYPE_BITMASK
                                ? PTHREAD_CANCEL_ASYNCHRONOUS
                                : PTHREAD_CANCEL_DEFERRED);

  THREAD_SETMEM(self, cleanup_jmp_buf, (struct pthread_unwind_buf *) buf);
}
```

### 5.3 \_\_\_pthread\_unregister\_cancel\_restore

```c
/* nptl/cleanup_defer.c:61-87 */
void ___pthread_unregister_cancel_restore(__pthread_unwind_buf_t *buf)
{
  struct pthread *self = THREAD_SELF;
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;

  /* 恢复帧链 */
  THREAD_SETMEM(self, cleanup_jmp_buf, ibuf->priv.data.prev);

  /* 如果原来是延迟模式，无需恢复 */
  if (ibuf->priv.data.canceltype == PTHREAD_CANCEL_DEFERRED)
    return;

  /* 恢复异步取消模式 */
  int cancelhandling = atomic_load_relaxed(&self->cancelhandling);
  if ((cancelhandling & CANCELTYPE_BITMASK) == 0)
    {
      int newval;
      do {
        newval = cancelhandling | CANCELTYPE_BITMASK;
      } while (!atomic_compare_exchange_weak_acquire(
                  &self->cancelhandling, &cancelhandling, newval));

      /* 恢复异步后检查是否有挂起的取消 */
      if (cancel_enabled_and_canceled(cancelhandling))
        __do_cancel(PTHREAD_CANCELED);
    }
}
```

**重要**：在恢复异步取消后，如果发现取消已挂起，立即调用 `__do_cancel` 执行取消。
这防止了在 defer 保护区间内积压的取消请求被遗漏。

---

## 6. start_thread 根帧与 setjmp 跳板

每个新线程在 `start_thread` 中建立**根展开帧**，作为所有 cleanup 帧链的终止点：

```c
/* nptl/pthread_create.c:401-436 */
struct pthread_unwind_buf unwind_buf;

/* 保存根跳转上下文 */
not_first_call = setjmp((struct __jmp_buf_tag *) unwind_buf.cancel_jmp_buf);

/* 设置链终止：无前驱、无老式 cleanup */
unwind_buf.priv.data.prev = NULL;
unwind_buf.priv.data.cleanup = NULL;

if (__glibc_likely(!not_first_call))
  {
    /* 将根帧注册为 cleanup_jmp_buf 头 */
    THREAD_SETMEM(pd, cleanup_jmp_buf, &unwind_buf);

    /* 运行用户函数 */
    ret = pd->start_routine(pd->arg);
  }
```

**跳板模式工作原理**：

```
                    start_thread
                    ┌─────────────────────────┐
                    │ setjmp(unwind_buf)       │◄─── longjmp 跳回这里
                    │   ↓ (首次返回0)          │
                    │ cleanup_jmp_buf = &buf   │
                    │ user_function()          │
                    │   ↓ (被取消)             │
                    │                          │
  _Unwind_ForcedUnwind                         │
  → unwind_stop                                │
    → 遍历老式 cleanup 链                       │
    → longjmp 到 setjmp ─────────────────────→ │
                    │   ↓ (返回非0)            │
                    │ 继续线程退出流程          │
                    └─────────────────────────┘
```

注意 `priv.data.prev = NULL` 必须在 `setjmp` **之后**设置，因为 setjmp 可能
覆盖 `priv` 区域的部分内容（`jmp_buf` 比 `cancel_jmp_buf` 大，多余部分与
`priv` 重叠）。源码注释明确解释了这一点（`pthread_create.c:415-425`）。

---

## 7. 栈展开与 unwind_stop

### 7.1 \_\_pthread\_unwind 入口

```c
/* nptl/unwind.c:118-135 */
void __attribute__((noreturn))
__pthread_unwind(__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  struct pthread *self = THREAD_SELF;

  /* 设置异常对象，无类型信息（不可捕获） */
  THREAD_SETMEM(self, exc.exception_class, 0);
  THREAD_SETMEM(self, exc.exception_cleanup, &unwind_cleanup);

  _Unwind_ForcedUnwind(&self->exc, unwind_stop, ibuf);
  /* NOTREACHED */
  abort();
}
```

`_Unwind_ForcedUnwind` 来自 `libgcc_s.so`（或静态链接的 `libgcc_eh.a`）。
调用前，`pthread_cancel` 和 `pthread_exit` 会先通过 `__libc_unwind_link_get()`
确保 libgcc_s 已加载：

```c
/* nptl/pthread_exit.c:27-33 */
struct unwind_link *unwind_link = __libc_unwind_link_get();
if (unwind_link == NULL)
  __libc_fatal(UNWIND_SONAME
               " must be installed for pthread_exit to work\n");
```

### 7.2 unwind_stop 停止函数

`_Unwind_ForcedUnwind` 在展开每一帧时调用此回调：

```c
/* nptl/unwind.c:38-106 */
static _Unwind_Reason_Code
unwind_stop(int version, _Unwind_Action actions,
            _Unwind_Exception_Class exc_class,
            struct _Unwind_Exception *exc_obj,
            struct _Unwind_Context *context, void *stop_parameter)
{
  struct pthread_unwind_buf *buf = stop_parameter;
  struct pthread *self = THREAD_SELF;
  struct _pthread_cleanup_buffer *curp = THREAD_GETMEM(self, cleanup);
  int do_longjump = 0;

  /* 栈地址调整，处理栈在主栈之上的情况 */
  uintptr_t adj = (uintptr_t) self->stackblock + self->stackblock_size;

  /* 到达栈底或 CFA 已越过 jmp_buf 目标 → 应该跳转 */
  if ((actions & _UA_END_OF_STACK)
      || !_JMPBUF_CFA_UNWINDS_ADJ(buf->cancel_jmp_buf[0].jmp_buf,
                                   context, adj))
    do_longjump = 1;

  /* 处理老式 cleanup 链中需要执行的处理器 */
  if (__glibc_unlikely(curp != NULL))
    {
      struct _pthread_cleanup_buffer *oldp = buf->priv.data.cleanup;
      void *cfa = (void *)(_Unwind_Ptr) _Unwind_GetCFA(context);

      if (curp != oldp && (do_longjump || FRAME_LEFT(cfa, curp, adj)))
        {
          do {
            struct _pthread_cleanup_buffer *nextp = curp->__prev;
            curp->__routine(curp->__arg);    /* 执行清理函数 */
            curp = nextp;
          } while (curp != oldp
                   && (do_longjump || FRAME_LEFT(cfa, curp, adj)));

          THREAD_SETMEM(self, cleanup, curp);
        }
    }

  if (do_longjump)
    __libc_unwind_longjmp(
      (struct __jmp_buf_tag *) buf->cancel_jmp_buf, 1);

  return _URC_NO_REASON;
}
```

**执行逻辑详解**：

1. **判断是否到达目标帧**：`_UA_END_OF_STACK`（栈底）或 CFA 已经越过
   `cancel_jmp_buf` 保存的 SP 目标。
2. **执行老式 cleanup**：遍历 `cleanup` 链，对每个在当前展开范围内的
   `_pthread_cleanup_buffer` 调用其 `__routine`。比较用 `oldp`（注册时快照）
   作为终止标记。
3. **longjmp 跳转**：到达目标帧后，调用 `__libc_unwind_longjmp` 跳回
   `sigsetjmp` 调用点。跳回后 sigsetjmp 返回 1（非零），触发用户宏中的
   cleanup 调用和 `__pthread_unwind_next`。
4. **继续展开**：返回 `_URC_NO_REASON` 告诉 unwinder 继续下一帧。

### 7.3 \_\_\_pthread\_unwind\_next

```c
/* nptl/unwind.c:138-145 */
void __attribute__((noreturn))
___pthread_unwind_next(__pthread_unwind_buf_t *buf)
{
  struct pthread_unwind_buf *ibuf = (struct pthread_unwind_buf *) buf;
  __pthread_unwind((__pthread_unwind_buf_t *) ibuf->priv.data.prev);
}
```

在 `pthread_cleanup_push` 宏的 longjmp 回归路径中被调用。执行完当前帧的
cleanup 后，跳到前一个帧继续展开。

### 7.4 完整展开流程

```
pthread_cancel / pthread_exit
  │
  ├── __libc_unwind_link_get()  确保 libgcc_s 已加载
  │
  └── __do_cancel(PTHREAD_CANCELED / value)
        │
        ├── 设置 CANCELSTATE | EXITING 标志
        │
        └── __pthread_unwind(&current_buf)
              │
              ├── exc.exception_class = 0
              ├── exc.exception_cleanup = unwind_cleanup
              │
              └── _Unwind_ForcedUnwind(&exc, unwind_stop, ibuf)
                    │
                    │   ┌──── 对每一帧调用 ────────────┐
                    │   │                               │
                    │   ▼                               │
                    │  unwind_stop()                    │
                    │   ├── 执行老式 cleanup 链          │
                    │   ├── CFA 未到目标 → return NO_REASON
                    │   │                               │
                    │   └── CFA 到达目标                 │
                    │       └── longjmp → sigsetjmp 返回 │
                    │           ├── __cancel_routine()   │
                    │           └── __pthread_unwind_next│
                    │               └── __pthread_unwind(prev)
                    │                   └── ForcedUnwind ┘
                    │
                    └── 到达根帧 (prev == NULL)
                        └── longjmp → start_thread 的 setjmp
                            └── 线程退出流程
```

---

## 8. C++ 异常交互

### 8.1 ForcedUnwind 与 C++ personality

当 `_Unwind_ForcedUnwind` 展开经过 C++ 帧时，C++ personality 函数看到
`_UA_FORCE_UNWIND` 标志（`sysdeps/generic/unwind.h:95-99`），知道这不是正常异常，
必须执行析构函数但**不能捕获**。

这意味着：
- C++ `try-catch` 块**不能**捕获 pthread 取消
- C++ 局部对象的析构函数**会**被正确调用
- RAII 风格的资源管理在取消路径上正常工作

### 8.2 异常吞噬保护

```c
/* nptl/unwind.c:109-115 */
static void
unwind_cleanup(_Unwind_Reason_Code reason, struct _Unwind_Exception *exc)
{
  __libc_fatal("FATAL: exception not rethrown\n");
}
```

如果 C++ 代码中有 `catch(...)` 捕获了取消异常但没有重新抛出（`throw;`），
`unwind_cleanup` 被调用，直接 `abort` 进程。这是对误用 `catch(...)` 的强制保护。

### 8.3 C++ 中的正确处理模式

```cpp
try {
  // 可能被取消的操作
} catch (abi::__forced_unwind &) {
  // 必须重新抛出！
  throw;
} catch (...) {
  // 这里不会捕获到取消
}
```

---

## 9. libc 内部清理：\_\_libc\_cleanup\_push\_defer

### 9.1 宏定义

```c
/* sysdeps/nptl/libc-lock.h:162-185 */
#define __libc_cleanup_region_start(DOIT, FCT, ARG)             \
  {   bool _cleanup_start_doit;                                  \
  struct _pthread_cleanup_buffer _buffer;                        \
  void (*_cleanup_routine)(void *) = (FCT);                     \
  _buffer.__arg = (ARG);                                         \
  if (DOIT)                                                      \
    {                                                            \
      _cleanup_start_doit = true;                                \
      _buffer.__routine = _cleanup_routine;                      \
      __libc_cleanup_push_defer(&_buffer);                       \
    }                                                            \
  else                                                           \
      _cleanup_start_doit = false;

#define __libc_cleanup_region_end(DOIT)         \
  if (_cleanup_start_doit)                      \
    __libc_cleanup_pop_restore(&_buffer);        \
  if (DOIT)                                     \
    _cleanup_routine(_buffer.__arg);             \
  }
```

### 9.2 函数实现

```c
/* nptl/libc-cleanup.c:22-49 */
void __libc_cleanup_push_defer(struct _pthread_cleanup_buffer *buffer)
{
  struct pthread *self = THREAD_SELF;

  buffer->__prev = THREAD_GETMEM(self, cleanup);

  /* 禁用异步取消 */
  int cancelhandling = atomic_load_relaxed(&self->cancelhandling);
  if (__glibc_unlikely(cancelhandling & CANCELTYPE_BITMASK))
    {
      int newval;
      do {
        newval = cancelhandling & ~CANCELTYPE_BITMASK;
      } while (!atomic_compare_exchange_weak_acquire(
                  &self->cancelhandling, &cancelhandling, newval));
    }

  buffer->__canceltype = (cancelhandling & CANCELTYPE_BITMASK
                          ? PTHREAD_CANCEL_ASYNCHRONOUS
                          : PTHREAD_CANCEL_DEFERRED);

  THREAD_SETMEM(self, cleanup, buffer);
}
```

**与 `___pthread_register_cancel_defer` 的区别**：
- libc 版本操作**老式 `cleanup` 链**
- register_cancel_defer 操作**新式 `cleanup_jmp_buf` 链**
- 两者都保存取消类型并切换到延迟模式，但挂载点不同

### 9.3 恢复函数

```c
/* nptl/libc-cleanup.c:52-75 */
void __libc_cleanup_pop_restore(struct _pthread_cleanup_buffer *buffer)
{
  struct pthread *self = THREAD_SELF;
  THREAD_SETMEM(self, cleanup, buffer->__prev);

  int cancelhandling = atomic_load_relaxed(&self->cancelhandling);
  if (buffer->__canceltype != PTHREAD_CANCEL_DEFERRED
      && (cancelhandling & CANCELTYPE_BITMASK) == 0)
    {
      int newval;
      do {
        newval = cancelhandling | CANCELTYPE_BITMASK;
      } while (!atomic_compare_exchange_weak_acquire(
                  &self->cancelhandling, &cancelhandling, newval));

      if (cancel_enabled_and_canceled(cancelhandling))
        __do_cancel(PTHREAD_CANCELED);
    }
}
```

与 `___pthread_unregister_cancel_restore` 逻辑完全一致：恢复取消类型后
检查挂起的取消。

---

## 10. combined 宏：pthread_once 的双重保护

### 10.1 问题背景

某些场景（如 `pthread_once`、`qsort`）需要同时支持：
- C++ 异常从回调函数抛出（需要 `__attribute__((cleanup))`）
- 没有展开信息的 C 帧中的取消（需要老式 `__pthread_cleanup_push`）

### 10.2 combined 帧结构

```c
/* sysdeps/nptl/pthreadP.h:574-581 */
struct __pthread_cleanup_combined_frame
{
  void (*__cancel_routine)(void *);
  void *__cancel_arg;
  int __do_it;
  struct _pthread_cleanup_buffer __buffer;   /* 内嵌老式缓冲区 */
};
```

### 10.3 combined 宏

```c
/* sysdeps/nptl/pthreadP.h:614-630 */
#define pthread_cleanup_combined_push(routine, arg) \
  do {                                                    \
    void (*__cancel_routine)(void *) = (routine);          \
    struct __pthread_cleanup_combined_frame __clframe       \
      __attribute__((__cleanup__(                           \
        __pthread_cleanup_combined_routine)))               \
      = { .__cancel_routine = __cancel_routine,            \
          .__cancel_arg = (arg), .__do_it = 1 };           \
    __pthread_cleanup_push(&__clframe.__buffer,            \
                           __pthread_cleanup_combined_routine_voidptr, \
                           &__clframe);

#define pthread_cleanup_combined_pop(execute) \
    __pthread_cleanup_pop(&__clframe.__buffer, 0);        \
    __clframe.__do_it = 0;                                 \
    if (execute)                                           \
      __cancel_routine(__clframe.__cancel_arg);             \
  } while (0)
```

**双重保护机制**：
1. `__attribute__((cleanup))` → 编译器在作用域结束时调用
   `__pthread_cleanup_combined_routine`（处理 C++ 异常路径）
2. `__pthread_cleanup_push` → 注册到老式 `cleanup` 链（处理取消路径）

### 10.4 使用示例：pthread_once

```c
/* nptl/pthread_once.c:114-118 */
pthread_cleanup_combined_push(clear_once_control, once_control);
init_routine();
pthread_cleanup_combined_pop(0);
```

如果 `init_routine` 中发生取消，`clear_once_control` 被调用以重置
`once_control`，允许其他线程重试初始化。

---

## 11. 条件变量等待中的 cleanup

### 11.1 cleanup 注册

```c
/* nptl/pthread_cond_wait.c:413-424 */
struct _pthread_cleanup_buffer buffer;
struct _condvar_cleanup_buffer cbuffer;
cbuffer.wseq = wseq;
cbuffer.cond = cond;
cbuffer.mutex = mutex;
cbuffer.private = private;
__pthread_cleanup_push(&buffer, __condvar_cleanup_waiting, &cbuffer);

err = __futex_abstimed_wait_cancelable64(
    cond->__data.__g_signals + g, signals, clockid, abstime, private);

__pthread_cleanup_pop(&buffer, 0);
```

使用老式 `__pthread_cleanup_push`，注册到 `cleanup` 链。

### 11.2 cleanup 处理函数

```c
/* nptl/pthread_cond_wait.c:149-171 */
static void
__condvar_cleanup_waiting(void *arg)
{
  struct _condvar_cleanup_buffer *cbuffer = arg;
  pthread_cond_t *cond = cbuffer->cond;
  unsigned g = cbuffer->wseq & 1;

  /* 1. 取消等待注册 */
  __condvar_cancel_waiting(cond, cbuffer->wseq >> 1, g, cbuffer->private);

  /* 2. 保守唤醒，防止信号丢失 */
  futex_wake(cond->__data.__g_signals + g, 1, cbuffer->private);

  /* 3. 确认唤醒 */
  __condvar_confirm_wakeup(cond, cbuffer->private);

  /* 4. 重新获取互斥锁 */
  __pthread_mutex_cond_lock(cbuffer->mutex);
}
```

源码中的 FIXME 注释（`pthread_cond_wait.c:158-163`）指出：当前取消实现可能导致
取消后的等待者消费了一个 futex 唤醒，使同组的另一个等待者无法醒来。因此保守地
额外发送一次 `futex_wake`。

### 11.3 \_\_condvar\_cancel\_waiting

```c
/* nptl/pthread_cond_wait.c:77-144 */
static void
__condvar_cancel_waiting(pthread_cond_t *cond, uint64_t seq,
                         unsigned int g, int private)
```

这个函数处理三种情况：

| 条件 | 处理 |
|------|------|
| `g1_start > seq`（我们的组已关闭） | 我们实际消费了一个信号 → 需要替代信号 |
| 在 G2 中 | 减少 G2 有效大小；溢出时广播唤醒所有 |
| 在 G1 中且 `g_size == 0` | 消费了信号 → 需要替代信号 |
| 在 G1 中且 `g_size > 0` | 减少大小，相当于放入再消费信号 |

如果 `consumed_signal == true`，调用 `__pthread_cond_signal(cond)` 发送替代信号。

---

## 12. 退出顺序：cleanup vs TSD vs TLS

线程退出时的清理顺序定义在 `start_thread` 中：

```c
/* nptl/pthread_create.c:443-465 */
/* 1. 用户函数返回 (或通过 cancel/exit 跳回) */
ret = pd->start_routine(pd->arg);

/* 2. C++ thread_local 析构函数 */
call_function_static_weak(__call_tls_dtors);     /* :458-459 */

/* 3. POSIX TSD (pthread_key) 析构函数 */
__nptl_deallocate_tsd();                         /* :461-462 */

/* 4. libc 线程局部资源释放 */
__libc_thread_freeres();                         /* :464-465 */
```

**完整顺序**：

```
1. cleanup handlers     ← 在展开过程中执行（取消/exit 路径）
                          或由 pthread_cleanup_pop(1) 手动触发
2. C++ thread_local 析构 ← __call_tls_dtors
3. POSIX TSD 析构        ← __nptl_deallocate_tsd (最多 PTHREAD_DESTRUCTOR_ITERATIONS 轮)
4. libc 内部释放         ← __libc_thread_freeres
5. 栈/TCB 回收           ← 在 start_thread 尾部或 join 时
```

注意：cleanup handlers 在展开过程中执行，早于所有其他析构函数。TSD 析构函数
可能迭代多次（默认 4 次），因为一个析构函数可能设置另一个 TSD 键值。

---

## 13. Fork 安全性

### 13.1 cleanup 链在 fork 后的状态

`fork()` 创建的子进程只有调用线程存在。cleanup 链作为线程私有数据
（`struct pthread` 的字段），随内存复制到子进程。

**行为**：
- 子进程继承父线程的 `cleanup` 和 `cleanup_jmp_buf` 链
- 这些帧指向父进程栈上的地址，在子进程中仍然有效（COW 复制）
- 但子进程不应该执行这些 cleanup（它们属于父线程的控制流）

**实践规则**：
- `fork()` 后不应调用 `pthread_cleanup_pop`
- `fork()` 后应立即 `exec()` 或谨慎操作
- `atfork` handlers 是正确的 fork 清理机制

### 13.2 atfork 与 cleanup 的关系

`pthread_atfork` 注册的 handler 在 fork 时执行，不与 cleanup 链交互。
它们是两套独立的清理机制：
- **cleanup**：线程取消/退出时
- **atfork**：`fork()` 前后的准备/恢复

---

## 14. AArch64 展开细节

### 14.1 jmpbuf-unwind.h

```c
/* sysdeps/aarch64/jmpbuf-unwind.h:24-37 */
#define _JMPBUF_UNWINDS(jmpbuf, address, demangle) \
  ((void *)(address) < (void *) demangle(jmpbuf[JB_SP]))

#define _JMPBUF_CFA_UNWINDS_ADJ(jmpbuf, context, adj) \
  _JMPBUF_UNWINDS_ADJ(jmpbuf,                          \
    (void *)(uintptr_t) _Unwind_GetCFA(context), adj)

#define _JMPBUF_UNWINDS_ADJ(_jmpbuf, _address, _adj) \
  ((uintptr_t)(_address) - (_adj) < _jmpbuf_sp(_jmpbuf) - (_adj))

/* 使用正常的 longjmp */
#define __libc_unwind_longjmp(buf, val) __libc_longjmp(buf, val)
```

AArch64 上 `__libc_unwind_longjmp` 直接映射到 `__libc_longjmp`，
不需要额外的展开操作。

### 14.2 \_\_longjmp.S

```asm
/* sysdeps/aarch64/__longjmp.S:24-150 */
```

AArch64 的 longjmp 恢复寄存器（x19-x28、FP、LR、SP）并跳转到保存的 PC。
这是取消展开到达 `cancel_jmp_buf` 目标后的最终跳转。

### 14.3 DWARF 展开

AArch64 使用标准 DWARF `.eh_frame` / `.eh_frame_hdr` 展开信息。
`_Unwind_ForcedUnwind` 依赖 libgcc_s 中的 DWARF unwinder 遍历调用帧。
每帧的 CFA (Canonical Frame Address) 用于与 `cancel_jmp_buf` 中保存的 SP 比较，
判断是否到达目标帧。

---

## 15. 总结：cleanup 帧链全景图

### 15.1 完整架构图

```
┌─────────────────────────────────────────────────────────┐
│                    用户层 API                            │
├─────────────────────────────────────────────────────────┤
│ pthread_cleanup_push/pop                                │
│   ├── C++ RAII 类      (路径一，最轻量)                  │
│   ├── GCC cleanup 属性 (路径二，纯 C)                    │
│   └── setjmp 跳板      (路径三，最重，后备)              │
│                                                         │
│ pthread_cleanup_push_defer_np/pop_restore_np            │
│   └── 同上三路径 + 保存/恢复取消类型                      │
├─────────────────────────────────────────────────────────┤
│                    glibc 内部 API                        │
├─────────────────────────────────────────────────────────┤
│ __pthread_cleanup_push/pop (老式链，condvar 等内部使用)   │
│ __libc_cleanup_push_defer/pop_restore (libc 临界区)     │
│ pthread_cleanup_combined_push/pop (双重保护，once/qsort) │
├─────────────────────────────────────────────────────────┤
│                    注册/展开层                            │
├─────────────────────────────────────────────────────────┤
│ ___pthread_register_cancel          → cleanup_jmp_buf 链│
│ ___pthread_register_cancel_defer    → 同上 + 切延迟     │
│ __pthread_cleanup_push              → cleanup 链         │
│ __libc_cleanup_push_defer           → cleanup 链 + 延迟  │
├─────────────────────────────────────────────────────────┤
│                    展开引擎                               │
├─────────────────────────────────────────────────────────┤
│ __pthread_unwind                                        │
│   └── _Unwind_ForcedUnwind (libgcc_s)                   │
│         └── unwind_stop (每帧回调)                       │
│               ├── 执行老式 cleanup 链                    │
│               └── longjmp → sigsetjmp 跳板               │
│                     ├── 执行用户 cleanup                  │
│                     └── __pthread_unwind_next(prev)       │
└─────────────────────────────────────────────────────────┘
```

### 15.2 两条链的协作

```
struct pthread
│
├── cleanup_jmp_buf ──→ [帧3] ──→ [帧2] ──→ [帧1 (根帧)] ──→ NULL
│                        │          │          │
│                        │          │          └── priv.data.cleanup: NULL
│                        │          └── priv.data.cleanup ──→ oldC
│                        └── priv.data.cleanup ──→ curC
│
└── cleanup ──→ [bufD] ──→ [bufC] ──→ [bufB] ──→ [bufA] ──→ NULL
                 │                     │
                 └── (当前帧3的快照)     └── (帧2注册时的快照: oldC)

展开帧3时：unwind_stop 遍历 cleanup 链从 bufD 到 curC（不含），
  执行 bufD 的 __routine。
展开帧2时：遍历 cleanup 链从 curC 到 oldC（不含），执行其间的 handler。
展开帧1时：遍历 cleanup 链从 oldC 到 NULL，执行剩余 handler。
```

### 15.3 设计精妙之处

1. **ABI 兼容**：`__pad[4]` 与 `priv.data` 共用 union，公开 ABI 不暴露内部结构
2. **栈分配**：所有帧都在栈上，无需堆分配，自动回收
3. **双链快照**：新式帧快照老式链头，展开时精确知道每个帧对应哪些老式 handler
4. **渐进展开**：longjmp 跳回 → 执行 cleanup → unwind_next → 再次
   ForcedUnwind → 再跳回，逐帧处理
5. **三路径适配**：根据编译器能力自动选择最优实现，从 RAII 到 setjmp 渐进退化

---

## 源码文件速查

| 文件 | 行号 | 内容 |
|------|------|------|
| `sysdeps/nptl/pthread.h` | 159-165 | `_pthread_cleanup_buffer` 结构 |
| `sysdeps/nptl/pthread.h` | 538-548 | `__pthread_unwind_buf_t` 公开定义 |
| `sysdeps/nptl/pthread.h` | 551-553 | `__cleanup_fct_attribute` |
| `sysdeps/nptl/pthread.h` | 568-583 | C++ `__pthread_cleanup_class` |
| `sysdeps/nptl/pthread.h` | 592-600 | C++ RAII 路径宏 |
| `sysdeps/nptl/pthread.h` | 606-617 | C++ defer 变体宏 |
| `sysdeps/nptl/pthread.h` | 624-629 | `__pthread_cleanup_routine` 内联 |
| `sysdeps/nptl/pthread.h` | 638-649 | GCC cleanup 属性路径宏 |
| `sysdeps/nptl/pthread.h` | 655-670 | GCC cleanup defer 变体宏 |
| `sysdeps/nptl/pthread.h` | 681-708 | setjmp 跳板路径宏 |
| `sysdeps/nptl/pthread.h` | 716-744 | setjmp defer 变体宏 |
| `nptl/descr.h` | 65-92 | `pthread_unwind_buf` 内部定义 |
| `nptl/descr.h` | 287-292 | `cleanup` / `cleanup_jmp_buf` 字段 |
| `nptl/cleanup.c` | 22-35 | `___pthread_register_cancel` |
| `nptl/cleanup.c` | 46-53 | `___pthread_unregister_cancel` |
| `nptl/cleanup_defer.c` | 22-52 | `___pthread_register_cancel_defer` |
| `nptl/cleanup_defer.c` | 61-87 | `___pthread_unregister_cancel_restore` |
| `nptl/cleanup_compat.c` | 23-33 | `__pthread_cleanup_push`（老式） |
| `nptl/cleanup_compat.c` | 39-49 | `__pthread_cleanup_pop`（老式） |
| `nptl/cleanup_routine.c` | 22-26 | `___pthread_cleanup_routine` |
| `nptl/libc-cleanup.c` | 22-49 | `__libc_cleanup_push_defer` |
| `nptl/libc-cleanup.c` | 52-75 | `__libc_cleanup_pop_restore` |
| `nptl/unwind.c` | 38-106 | `unwind_stop` 停止函数 |
| `nptl/unwind.c` | 109-115 | `unwind_cleanup`（异常吞噬保护） |
| `nptl/unwind.c` | 118-135 | `__pthread_unwind` |
| `nptl/unwind.c` | 138-145 | `___pthread_unwind_next` |
| `nptl/pthread_create.c` | 401-436 | start_thread 根展开帧 |
| `nptl/pthread_create.c` | 458-465 | 退出顺序（TLS→TSD→freeres） |
| `nptl/pthread_exit.c` | 24-37 | `__pthread_exit` |
| `nptl/pthread_cond_wait.c` | 77-144 | `__condvar_cancel_waiting` |
| `nptl/pthread_cond_wait.c` | 149-171 | `__condvar_cleanup_waiting` |
| `nptl/pthread_cond_wait.c` | 413-424 | condvar 等待中的 cleanup 注册 |
| `nptl/pthread_once.c` | 114-118 | combined 宏使用示例 |
| `sysdeps/nptl/pthreadP.h` | 554-571 | 内部 cleanup_push/pop 宏重定义 |
| `sysdeps/nptl/pthreadP.h` | 574-630 | combined 帧结构与宏 |
| `sysdeps/nptl/libc-lock.h` | 155-185 | libc cleanup 宏 |
| `sysdeps/aarch64/jmpbuf-unwind.h` | 24-37 | AArch64 展开宏 |
