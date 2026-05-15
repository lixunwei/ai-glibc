# exit / _exit 进程退出与清理调用链

> 基于 glibc 2.43 源码，使用 clangd LSP 进行精确调用层次分析。

---

## 1. 概述

glibc 提供多种进程终止方式，清理程度从高到低：

| 函数 | 清理级别 | 行为 |
|------|---------|------|
| `exit(status)` | **完全清理** | TLS 析构 → atexit/on_exit 处理器 → stdio flush → _exit |
| `quick_exit(status)` | **部分清理** | at_quick_exit 处理器 → _exit（无 TLS 析构、无 stdio flush） |
| `abort()` | **异常终止** | raise(SIGABRT) → 若返回则 _exit |
| `_exit(status)` / `_Exit(status)` | **无清理** | 直接 exit_group 系统调用 |

---

## 2. exit() 完整调用链

### 2.1 入口

```
exit(status)                            [stdlib/exit.c:146]
└── __run_exit_handlers(status, &__exit_funcs, true, true)
                                        [stdlib/exit.c:43]
      参数含义：
        run_list_atexit = true   → 执行 _IO_cleanup
        run_dtors = true         → 执行 TLS 析构器
```

### 2.2 __run_exit_handlers — 核心清理引擎

```
__run_exit_handlers(status, listp, run_list_atexit, run_dtors)
                                        [stdlib/exit.c:43]
│
├── 1. __libc_lock_lock_recursive(__exit_lock)
│      获取递归锁（允许 atexit 处理器中再调 exit）
│
├── 2. __call_tls_dtors()               [stdlib/cxa_thread_atexit_impl.c:147]
│      (仅 run_dtors == true 时)
│      执行 C++ thread_local 变量析构器
│      ├── while (tls_dtor_list) {
│      │     func = PTR_DEMANGLE(cur->func)
│      │     func(cur->obj)              ← 调用析构函数
│      │     atomic_fetch_add_release(&cur->map->l_tls_dtor_count, -1)
│      │     free(cur)                   ← 释放注册节点
│      │   }
│      └── 与 _dl_close_worker 通过 l_tls_dtor_count 原子同步
│
├── 3. __libc_lock_lock(__exit_funcs_lock)
│
├── 4. [主循环：逆序执行所有注册函数]
│      while (true) {
│        cur = *listp                    ← 链表头
│        if (cur == NULL) {
│          __exit_funcs_done = true      ← 标记完成，阻止新注册
│          break
│        }
│        while (cur->idx > 0) {
│          f = &cur->fns[--cur->idx]
│          switch (f->flavor) {
│
│   ├── case ef_at:    (atexit 注册)
│   │     atfct = PTR_DEMANGLE(f->func.at)
│   │     __libc_lock_unlock(__exit_funcs_lock)
│   │     atfct()                        ← 调用 atexit 处理器
│   │     __libc_lock_lock(__exit_funcs_lock)
│   │
│   ├── case ef_on:    (on_exit 注册)
│   │     onfct = PTR_DEMANGLE(f->func.on.fn)
│   │     arg = f->func.on.arg
│   │     __libc_lock_unlock(__exit_funcs_lock)
│   │     onfct(status, arg)             ← 调用 on_exit 处理器
│   │     __libc_lock_lock(__exit_funcs_lock)
│   │
│   ├── case ef_cxa:   (__cxa_atexit 注册，C++ 静态析构)
│   │     f->flavor = ef_free            ← 防止 dlclose/exit 竞争双调用
│   │     cxafct = PTR_DEMANGLE(f->func.cxa.fn)
│   │     arg = f->func.cxa.arg
│   │     __libc_lock_unlock(__exit_funcs_lock)
│   │     cxafct(arg, status)            ← 调用 C++ 析构器
│   │     __libc_lock_lock(__exit_funcs_lock)
│   │
│   └── case ef_free/ef_us: break        ← 已释放或未使用
│
│          if (new_exitfn_called changed)
│            goto restart                ← 有新注册，重新扫描
│        }
│        *listp = cur->next
│        free(cur)                       ← 释放链表节点
│      }
│
├── 5. __libc_lock_unlock(__exit_funcs_lock)
│
├── 6. _IO_cleanup()                    [libio/genops.c:887]
│      (仅 run_list_atexit == true 时)
│      stdio 流清理
│
└── 7. _exit(status)                    [sysdeps/unix/sysv/linux/_exit.c:26]
       最终终止
```

### 2.3 函数指针安全：PTR_DEMANGLE

所有通过函数指针调用的处理器都经过 **指针加密/解密** 保护：

```
注册时：PTR_MANGLE(func)   → 存储加密后的指针
调用时：PTR_DEMANGLE(func) → 解密还原原始指针
```

这防止攻击者通过覆写 `__exit_funcs` 链表来劫持控制流。

---

## 3. _IO_cleanup — stdio 流清理

### 3.1 调用链

```
_IO_cleanup()                           [libio/genops.c:887]
├── _IO_flush_all()                     [libio/genops.c:709]
│     刷新所有打开的流
│     ├── __libc_cleanup_push_defer(flush_cleanup)
│     │     设置取消清理                [nptl/libc-cleanup.c:23]
│     ├── __libc_lock_lock(_IO_list_lock)
│     │     获取全局流链表锁
│     ├── [遍历 _IO_list_all 链表]
│     │   for (fp = _IO_list_all; fp != NULL; fp = fp->_chain) {
│     │     IO_validate_vtable(fp)      [libio/libioP.h:1035]
│     │       验证 vtable 合法性
│     │     if (有待写数据 && 非错误/无缓冲区)
│     │       _IO_OVERFLOW(fp, EOF)     ← 通过 vtable 调用实际 flush
│     │   }
│     ├── __libc_lock_unlock(_IO_list_lock)
│     └── __libc_cleanup_pop_restore(0)
│
└── _IO_unbuffer_all()                  [libio/genops.c:811]
      关闭所有流的缓冲区（防止 freeres 后访问已释放内存）
      ├── __libc_cleanup_push_defer(flush_cleanup)
      ├── __libc_lock_lock(_IO_list_lock)
      ├── [遍历 _IO_list_all 链表]
      │   for (fp = _IO_list_all; fp != NULL; fp = fp->_chain) {
      │     _IO_free_backup_area(fp)    [libio/genops.c:212]
      │     _IO_free_wbackup_area(fp)   [libio/wgenops.c:414]
      │     if (非不可关闭流)
      │       IO_validate_vtable(fp)
      │       _IO_SETBUF(fp, NULL, 0)   ← 通过 vtable 禁用缓冲
      │       _IO_wsetb(fp, NULL, NULL, 0)
      │   }
      ├── __libc_lock_unlock(_IO_list_lock)
      └── __libc_cleanup_pop_restore(0)
```

### 3.2 freeres 竞争问题

`_IO_cleanup` 和 `__libc_freeres`（valgrind 用于检测泄漏）之间存在调用顺序不确定性：
- 如果 `_IO_cleanup` 先执行：设置 `dealloc_buffers = true`，`_IO_unbuffer_all` 释放缓冲区
- 如果 `__libc_freeres` 先执行：通过 `_IO_unbuffer_all` 的 freeres 链表版本处理

---

## 4. quick_exit() 调用链

```
quick_exit(status)                      [stdlib/quick_exit.c:28]
└── __run_exit_handlers(status, &__quick_exit_funcs, false, false)
      参数：
        listp = &__quick_exit_funcs     ← 独立的处理器链表
        run_list_atexit = false          ← 不执行 _IO_cleanup
        run_dtors = false                ← 不执行 TLS 析构
```

**与 exit() 的区别**：
- 使用独立的 `__quick_exit_funcs` 链表（通过 `at_quick_exit` 注册）
- 不刷新 stdio 流
- 不调用 TLS 析构器
- C++11 语义：不运行对象析构函数

**历史兼容版本**（GLIBC_2_10）会执行 TLS 析构（`run_dtors = true`）。

---

## 5. abort() 调用链

```
abort()                                 [stdlib/abort.c:75]
│
├── 1. raise(SIGABRT)                   [sysdeps/posix/raise.c:24]
│      首次尝试：发送 SIGABRT 给自己
│      如果用户安装了 SIGABRT handler，handler 可以返回
│
├── 2. __abort_lock_wrlock()            [stdlib/abort.c:60]
│      获取写锁（与 posix_spawn 的读锁互斥）
│      防止 posix_spawn 在 abort 过程中操作信号
│
├── 3. __sigfillset(&sigs)
│      填充全信号集
│
├── 4. __libc_sigaction(SIGABRT, &act, NULL)
│      强制重置 SIGABRT 为 SIG_DFL     [sysdeps/unix/sysv/linux/libc_sigaction.c:42]
│
├── 5. internal_signal_unblock_signal(SIGABRT)
│      确保 SIGABRT 未被阻塞          [sysdeps/unix/sysv/linux/internal-signals.h:94]
│
├── 6. __pthread_raise_internal(SIGABRT)
│      再次发送 SIGABRT（此时必定终止） [nptl/pthread_kill.c:76]
│
└── 7. _exit(127)                       [sysdeps/unix/sysv/linux/_exit.c:26]
       最后手段：如果 raise 仍然返回，直接 _exit
       （理论上不可达，但作为防御性编程）
```

**abort_lock 的作用**：
- `posix_spawn` 的 `__spawnix` 在 clone 前获取读锁
- `abort` 获取写锁，确保不会与 posix_spawn 的信号操作交错
- 这防止了 SIGABRT handler 在 posix_spawn 子进程中被意外重置

---

## 6. _exit() — 最终终止

```
_exit(status)                           [sysdeps/unix/sysv/linux/_exit.c:26]
└── while (1) {
      INLINE_SYSCALL(exit_group, 1, status)
        ← 终止进程中所有线程
      ABORT_INSTRUCTION
        ← 不可达保护（如 brk #1 on AArch64）
    }
```

- `exit_group`（syscall nr 94）终止整个线程组
- 无限循环 + `ABORT_INSTRUCTION` 确保即使内核异常返回也不会继续执行
- `_Exit` 是 `_exit` 的弱别名，语义完全相同

---

## 7. atexit 注册机制

### 7.1 注册调用链

```
atexit(func)                            [stdlib/atexit.c]
└── __cxa_atexit(func, NULL, NULL)      [stdlib/cxa_atexit.c:66]
      └── __internal_atexit(func, arg, d, &__exit_funcs)
                                        [stdlib/cxa_atexit.c:34]

__cxa_atexit(func, arg, dso_handle)     [stdlib/cxa_atexit.c:66]
└── __internal_atexit(func, arg, dso_handle, &__exit_funcs)

at_quick_exit(func)                     [stdlib/at_quick_exit.c]
└── __cxa_at_quick_exit(func, NULL)
      └── __internal_atexit(func, NULL, d, &__quick_exit_funcs)
```

### 7.2 __internal_atexit

```
__internal_atexit(func, arg, d, listp)  [stdlib/cxa_atexit.c:34]
├── assert(!__exit_funcs_done)          [不允许在 exit 后注册]
├── __libc_lock_lock(__exit_funcs_lock)
├── __new_exitfn(listp)                 [stdlib/cxa_atexit.c:79]
│     ├── 检查当前块是否已满 (idx == fns 数组大小)
│     │   ├── 满：calloc 新块，插入链表头
│     │   └── 未满：直接使用
│     └── 返回空闲的 exit_function 槽位
├── PTR_MANGLE(func)                    [加密函数指针]
├── 设置 new->func.cxa = { .fn = func, .arg = arg, .dso_handle = d }
├── new->flavor = ef_cxa
├── __libc_lock_unlock(__exit_funcs_lock)
└── 增加 __new_exitfn_called 计数器
```

### 7.3 数据结构

```c
struct exit_function_list {
  struct exit_function_list *next;   /* 链表指针 */
  size_t idx;                        /* 当前已用槽位数 */
  struct exit_function fns[32];      /* 函数数组（每块32个） */
};

struct exit_function {
  long int flavor;                   /* ef_free/ef_us/ef_on/ef_at/ef_cxa */
  union {
    void (*at)(void);                /* atexit */
    struct { void (*fn)(int, void*); void *arg; } on;  /* on_exit */
    struct { void (*fn)(void*, int); void *arg; void *dso_handle; } cxa;
  } func;
};
```

**两个独立链表**：
- `__exit_funcs` — exit() 使用（atexit + __cxa_atexit 注册）
- `__quick_exit_funcs` — quick_exit() 使用（at_quick_exit 注册）

---

## 8. __cxa_finalize — DSO 卸载时的清理

当 `dlclose` 卸载一个共享库时，需要执行该 DSO 注册的析构器：

```
__cxa_finalize(dso_handle)              [stdlib/cxa_finalize.c:44]
├── __libc_lock_lock(__exit_funcs_lock)
├── [遍历 __exit_funcs 链表]
│     for each exit_function f:
│       if (f->flavor == ef_cxa && f->func.cxa.dso_handle == dso_handle) {
│         f->flavor = ef_free           ← 标记已处理
│         cxafct = PTR_DEMANGLE(f->func.cxa.fn)
│         __libc_lock_unlock(__exit_funcs_lock)
│         cxafct(arg, 0)                ← 调用 DSO 的析构器
│         __libc_lock_lock(__exit_funcs_lock)
│       }
├── __libc_lock_unlock(__exit_funcs_lock)
└── __unregister_atfork(dso_handle)     [posix/register-atfork.c:81]
      移除该 DSO 注册的 atfork 处理器
```

**dso_handle 机制**：
- 每个 DSO 编译时生成唯一的 `__dso_handle` 符号
- `__cxa_atexit` 注册时保存 `dso_handle`
- `dlclose` 时 `__cxa_finalize(__dso_handle)` 只执行该 DSO 的析构器
- `exit()` 时 `dso_handle` 被忽略，执行所有析构器

---

## 9. __call_tls_dtors — 线程局部析构

```
__call_tls_dtors()                      [stdlib/cxa_thread_atexit_impl.c:147]
└── while (tls_dtor_list != NULL) {
      cur = tls_dtor_list
      func = PTR_DEMANGLE(cur->func)
      tls_dtor_list = cur->next

      func(cur->obj)                    ← 调用 thread_local 析构器

      atomic_fetch_add_release(&cur->map->l_tls_dtor_count, -1)
        ← release 语义确保 link_map 在计数减到 0 之前可用
        ← 与 _dl_close_worker 的 acquire load 配对

      free(cur)
    }
```

**并发安全**：
- `tls_dtor_list` 是线程局部变量，无需锁
- `l_tls_dtor_count` 使用原子操作与 `_dl_close_worker` 同步
- 防止 DSO 在析构器还在执行时被卸载

---

## 10. 完整退出时序图

```
exit(status)
  │
  ▼
__run_exit_handlers()
  │
  ├── ① __call_tls_dtors()
  │     └── C++ thread_local 析构器（LIFO 逆序）
  │
  ├── ② atexit/on_exit/__cxa_atexit 处理器（LIFO 逆序）
  │     ├── ef_cxa: C++ 全局/静态对象析构器
  │     ├── ef_at:  atexit() 注册的函数
  │     └── ef_on:  on_exit() 注册的函数
  │
  ├── ③ _IO_cleanup()
  │     ├── _IO_flush_all()
  │     │     刷新所有 FILE* 写缓冲区
  │     │     （确保 printf 输出不丢失）
  │     └── _IO_unbuffer_all()
  │           禁用所有流的缓冲区
  │           释放备份缓冲区
  │
  └── ④ _exit(status)
        └── exit_group syscall
              终止所有线程，不返回
```

**对比其他退出路径**：

```
┌──────────────────────────────────────────────────────────┐
│        exit()        │  quick_exit()  │    abort()       │
├──────────────────────┼────────────────┼──────────────────┤
│ ✓ TLS 析构器         │ ✗              │ ✗               │
│ ✓ atexit 处理器      │ ✗              │ ✗               │
│ ✗                    │ ✓ at_quick_exit│ ✗               │
│ ✓ stdio flush        │ ✗              │ ✗               │
│ ✓ _exit()           │ ✓ _exit()      │ ✓ SIGABRT→_exit │
└──────────────────────┴────────────────┴──────────────────┘
```

---

## 11. 锁机制设计

### 11.1 三把锁的协作

| 锁 | 类型 | 保护内容 |
|----|------|---------|
| `__exit_lock` | 递归互斥锁 | 防止并发 exit()、允许处理器中递归调 exit |
| `__exit_funcs_lock` | 普通互斥锁 | 保护 exit_function_list 链表读写 |
| `abort_lock` | 读写锁 | 协调 abort 与 posix_spawn 的信号操作 |

### 11.2 处理器执行期间解锁

```
调用前：__libc_lock_unlock(__exit_funcs_lock)  ← 释放链表锁
调用：  func()                                  ← 处理器可能：
          - 注册新 atexit 处理器
          - 调用 exit()（递归）
          - 调用 dlclose()
调用后：__libc_lock_lock(__exit_funcs_lock)    ← 重新获取
```

这避免了处理器中注册新处理器时的死锁，但引入了 `restart` 机制来处理链表变化。

### 11.3 __exit_funcs_done 标志

```c
bool __exit_funcs_done = false;
```

在所有处理器执行完毕后设为 `true`，此后：
- `__internal_atexit` 中的 `assert(!__exit_funcs_done)` 触发
- 防止在清理已完成后再注册新处理器（未定义行为的防护）

---

## 12. 涉及的源文件

| 源文件 | 内容 |
|--------|------|
| `stdlib/exit.c` | `exit()`、`__run_exit_handlers()` |
| `stdlib/quick_exit.c` | `quick_exit()` |
| `stdlib/abort.c` | `abort()`、abort_lock |
| `stdlib/cxa_atexit.c` | `__cxa_atexit`、`__internal_atexit`、`__new_exitfn` |
| `stdlib/cxa_finalize.c` | `__cxa_finalize`（dlclose 清理） |
| `stdlib/cxa_thread_atexit_impl.c` | `__cxa_thread_atexit_impl`、`__call_tls_dtors` |
| `stdlib/exit.h` | `exit_function`/`exit_function_list` 结构体定义 |
| `libio/genops.c` | `_IO_cleanup`、`_IO_flush_all`、`_IO_unbuffer_all` |
| `sysdeps/unix/sysv/linux/_exit.c` | `_exit()` exit_group syscall |
| `malloc/set-freeres.c` | `__libc_freeres`（内存泄漏检测辅助） |
| `posix/register-atfork.c` | `__unregister_atfork`（dlclose 时清理） |

---

## 13. 与其他子系统的交互

```
┌──────────────┐  dlclose      ┌───────────────┐
│ 动态链接器    │──────────────→│ __cxa_finalize │
│ _dl_close    │               │  (DSO 析构)   │
└──────────────┘               └───────┬───────┘
                                       │
┌──────────────┐                       ▼
│  C++ 运行时  │──── __cxa_atexit ──→ ┌────────────────┐
│ 全局对象构造 │                       │ __exit_funcs   │
└──────────────┘                       │ (处理器链表)   │
                                       └────────┬───────┘
┌──────────────┐                                │
│  用户代码    │──── atexit/on_exit ────────────→│
└──────────────┘                                │
                                                ▼
                               ┌────────────────────────────┐
                               │    __run_exit_handlers()    │
                               ├────────────────────────────┤
                               │ ① TLS dtors                │
                               │ ② atexit/cxa handlers      │
                               │ ③ _IO_cleanup (stdio)      │
                               │ ④ _exit (syscall)          │
                               └────────────────────────────┘
                                                │
                            ┌───────────────────┼──────────┐
                            ▼                   ▼          ▼
                    ┌──────────────┐  ┌──────────┐  ┌──────────┐
                    │ stdio 子系统  │  │ malloc   │  │  内核    │
                    │ flush + free │  │  free    │  │exit_group│
                    └──────────────┘  └──────────┘  └──────────┘
```

---

> 本文档基于 clangd 调用层次分析生成，覆盖 glibc 2.43 中所有进程退出路径的完整实现。
