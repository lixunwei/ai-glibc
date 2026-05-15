# pthread_create 完整调用链分析

> 基于 clangd LSP 语义分析获取的精确调用层次，涵盖线程创建从用户 API 到内核 clone3 的全路径。

---

## 一、调用层次总览

```
用户调用: pthread_create() / __pthread_create()    [pthread_create.c:930/933]
  └── __pthread_create_2_1()                        [pthread_create.c:649]
      ├── late_init()                               [pthread_create.c:73]
      ├── __pthread_getattr_default_np()            [pthread_getattr_default_np.c:34]
      ├── allocate_stack()                          [allocatestack.c:357]
      │   ├── __getpagesize()                       — 获取页面大小
      │   ├── __nptl_tls_static_size_for_stack()   — TLS 静态大小
      │   ├── get_cached_stack()                    [allocatestack.c:60]
      │   ├── allocate_thread_stack()               [allocatestack.c:158]
      │   ├── _dl_allocate_tls()                    [dl-tls.c:730]
      │   ├── setup_stack_prot()                    [allocatestack.c:192]
      │   ├── __madvise()                           — 栈保护页
      │   ├── __nptl_stack_list_add()               [nptl-stack.c:42]
      │   └── adjust_stack_prot()                   [allocatestack.c:241]
      ├── tls_setup_tcbhead()                       [tls-setup.h:20]
      ├── RSEQ_SELF()                               [rseq-internal.h:91]
      ├── collect_default_sched()                   [default-sched.h:27]
      ├── _IO_enable_locks()                        [genops.c:543]
      ├── internal_signal_block_all()               [internal-signals.h:79]
      ├── report_thread_creation()                  [pthread_create.c:632]
      ├── create_thread()                           [pthread_create.c:234]
      │   ├── __clone_internal()                    [clone-internal.c:95] → clone3 系统调用
      │   └── → start_thread()                      （新线程入口，见下方）
      ├── internal_signal_restore_set()             [internal-signals.h:87]
      ├── __nptl_create_event()                     [events.c:24] — GDB 通知
      └── 返回新线程 tid
```

---

## 二、新线程执行入口 start_thread()

`start_thread` 是 `clone3` 创建的新线程的执行入口（通过函数指针传递给内核）：

```
start_thread()                                      [pthread_create.c:338]
  ├── rseq_register_current_thread()                [rseq-internal.h:98]
  ├── __ctype_init()                                [ctype-info.c:36]
  ├── name_stack_maps()                             [allocatestack.c:674]
  ├── _setjmp()                                     — 设置 longjmp 点（用于 pthread_exit）
  ├── 用户函数执行: pd->start_routine(pd->arg)
  ├── internal_signal_restore_set()                 — 恢复信号
  ├── __nptl_deallocate_tsd()                       [nptl_deallocate_tsd.c:22] — TSD 析构
  ├── __libc_thread_freeres()                       [thread-freeres.c:30] — 线程 TLS 资源
  ├── __nptl_death_event()                          [events.c:30] — GDB 死亡通知
  ├── advise_stack_range()                          [allocatestack.c:329] — MADV_DONTNEED
  ├── __getrandom_vdso_release()                    [getrandom.c:302]
  ├── futex_wake()                                  — 唤醒 pthread_join 等待者
  ├── futex_wait_simple()                           — 等待分离线程完成
  └── __nptl_free_tcb()                             [nptl_free_tcb.c:24] — 释放 TCB
```

---

## 三、关键子调用详解

### 3.1 allocate_stack() — 栈分配

**源文件**: `nptl/allocatestack.c:357-658`

| 调用的函数 | 源文件 | 作用 |
|------------|--------|------|
| `get_cached_stack()` | allocatestack.c:60 | 从缓存复用已释放的线程栈 |
| `allocate_thread_stack()` | allocatestack.c:158 | mmap 新栈 + mprotect 守护页 |
| `_dl_allocate_tls()` | dl-tls.c:730 | 分配 TLS 块 |
| `setup_stack_prot()` | allocatestack.c:192 | 设置栈 mprotect |
| `__nptl_stack_list_add()` | nptl-stack.c:42 | 将栈加入全局链表 |
| `__madvise()` | — | MADV_DONTNEED 回收缓存栈物理页 |
| `_dl_deallocate_tls()` | dl-tls.c:745 | 失败时回收 TLS |
| `__munmap()` | — | 失败时释放 mmap |

### 3.2 create_thread() — 创建内核线程

**源文件**: `nptl/pthread_create.c:234-334`

| 调用的函数 | 源文件 | 作用 |
|------------|--------|------|
| `__clone_internal()` | clone-internal.c:95 | 封装 clone3 系统调用 |
| `start_thread` | （函数指针参数） | 新线程的入口 |

clone3 参数：
- `flags`: `CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID`
- `child_tidptr` = `&pd->tid`（用于 pthread_join 的 futex 等待）

### 3.3 __pthread_create_2_1() 的信号处理

```
信号处理流程：
  1. internal_signal_block_all()          — 阻塞所有信号（防止新线程未就绪时收信号）
  2. internal_sigdelset(SIGCANCEL)         — 但保留 SIGCANCEL（取消信号）
  3. internal_sigdelset(SIGSETXID)         — 但保留 SIGSETXID（setuid 广播信号）
  4. [clone3 创建新线程]
  5. internal_signal_restore_set()         — 父线程恢复原信号掩码
  6. 新线程在 start_thread 中自行设置正确的信号掩码
```

---

## 四、线程退出与资源清理

线程退出有三种路径：
1. **正常返回**: 用户函数 return → 继续 start_thread 清理流程
2. **pthread_exit()**: longjmp 到 start_thread 中 _setjmp 设置的点
3. **pthread_cancel()**: 通过 SIGCANCEL 信号触发 longjmp

清理顺序：
```
1. __nptl_deallocate_tsd()    — 调用所有 TSD 析构函数（最多迭代 PTHREAD_DESTRUCTOR_ITERATIONS 次）
2. __libc_thread_freeres()    — 释放 malloc arena 绑定、locale 等线程 TLS 资源
3. __nptl_death_event()       — 通知 GDB
4. advise_stack_range()       — MADV_DONTNEED 释放栈物理页
5. futex_wake(&pd->tid)       — 唤醒 pthread_join 等待者
6. [如果 detached] __nptl_free_tcb() — 释放 TCB 和栈内存
7. [如果 joinable] 等待 joiner 释放 → futex_wait_simple()
```

---

## 五、数据结构流转

```
pthread_create() 参数:
  ├── pthread_t *newthread  → 输出: 线程 ID (实际是 struct pthread* 的地址)
  ├── pthread_attr_t *attr  → 输入: 栈大小/地址、分离状态、调度策略
  ├── start_routine         → 输入: 用户函数指针
  └── arg                   → 输入: 用户参数

allocate_stack() 输出:
  ├── struct pthread *pd    — 线程描述符（位于栈顶/底部）
  ├── char **stackaddr      — 栈基地址
  └── size_t *stacksize     — 实际栈大小

struct pthread 关键字段（传递给新线程）:
  ├── pd->start_routine     — 用户函数
  ├── pd->arg               — 用户参数
  ├── pd->tid               — 内核分配的 tid（clone3 写入）
  ├── pd->cancelhandling    — 取消状态
  ├── pd->schedpolicy/param — 调度策略
  └── pd->specific[]        — TSD 数组
```

---

## 六、源码位置快速参考

| 文件:行 | 内容 |
|---------|------|
| `nptl/pthread_create.c:649-929` | `__pthread_create_2_1` 主函数 |
| `nptl/pthread_create.c:234-334` | `create_thread` |
| `nptl/pthread_create.c:338-627` | `start_thread` 新线程入口 |
| `nptl/pthread_create.c:73-97` | `late_init` 首次初始化 |
| `nptl/pthread_create.c:632-647` | `report_thread_creation` |
| `nptl/allocatestack.c:357-658` | `allocate_stack` |
| `nptl/allocatestack.c:60-151` | `get_cached_stack` |
| `nptl/allocatestack.c:158-239` | `allocate_thread_stack` |
| `nptl/allocatestack.c:192-237` | `setup_stack_prot` |
| `nptl/allocatestack.c:329-354` | `advise_stack_range` |
| `nptl/nptl-stack.c:42-86` | `__nptl_stack_list_add` |
| `nptl/nptl_deallocate_tsd.c:22-86` | TSD 析构函数调用 |
| `nptl/nptl_free_tcb.c:24-50` | TCB 释放 |
| `sysdeps/unix/sysv/linux/clone-internal.c:95-170` | `__clone_internal` |
| `sysdeps/unix/sysv/linux/internal-signals.h:79-96` | 信号阻塞/恢复 |
| `elf/dl-tls.c:730-741` | `_dl_allocate_tls` |
