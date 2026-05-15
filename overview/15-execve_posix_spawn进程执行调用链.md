# execve / posix_spawn 进程执行调用链

> 基于 glibc 2.43 源码，使用 clangd LSP 进行精确调用层次分析。

---

## 1. 概述

glibc 提供两大进程执行接口族：

| 接口族 | 特点 | 典型使用场景 |
|--------|------|-------------|
| **exec 族** | 替换当前进程映像，不返回 | fork()+exec() 经典模型 |
| **posix_spawn** | 一步完成创建+执行，使用 CLONE_VM+CLONE_VFORK | 高性能进程创建，替代 fork+exec |
| **system()** | 通过 /bin/sh 执行命令字符串 | 简单命令执行 |

---

## 2. exec 函数族调用关系

### 2.1 总体架构

```
execl(path, arg, ...)   ──→ __execve(path, argv, environ)
execle(path, arg, ..., envp) → __execve(path, argv, envp)
execv(path, argv)       ──→ __execve(path, argv, environ)
execlp(file, arg, ...)  ──→ execvp(file, argv)
execvp(file, argv)      ──→ __execvpe(file, argv, environ)
execvpe(file, argv, envp) → __execvpe(file, argv, envp)
fexecve(fd, argv, envp) ──→ execveat/execve 通过 /proc/self/fd/N

           ┌─────────────────────────────┐
           │     __execvpe_common()      │← PATH 搜索逻辑
           │  posix/execvpe.c:71         │
           └──────────┬──────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  __execve()  │← 最终系统调用入口
              │  (syscall)   │
              └──────────────┘
```

### 2.2 __execve — 系统调用封装

```
__execve()                              [syscalls.list 自动生成]
  └── SYS_execve (syscall nr 221 on AArch64)
```

`__execve` 由 `sysdeps/unix/sysv/linux/syscalls.list` 自动生成：
```
execve    -    execve    i:spp    __execve    execve
```

它直接发起 `execve` 系统调用，将进程映像完全替换。

### 2.3 __execvpe_common — PATH 搜索

```
__execvpe_common()                      [posix/execvpe.c:71]
├── getenv("PATH")                      [stdlib/getenv.c:24]
│     获取 PATH 环境变量
├── strchr(file, '/')                   [检查是否含路径分隔符]
│     含 '/' 则直接 execve，不搜索 PATH
├── __strnlen()                         [include/string.h:12]
├── __libc_alloca_cutoff()              [nptl/alloca_cutoff.c:26]
│     判断是否使用栈分配（避免大 PATH 栈溢出）
├── __strchrnul()                       [include/string.h:39]
│     逐段解析 PATH 中的目录
├── mempcpy() / memcpy()                [构造完整路径]
├── __execve(fullpath, argv, envp)      [include/unistd.h:111]
│     尝试执行完整路径
└── maybe_script_execute()              [posix/execvpe.c:39]
      ENOEXEC 时尝试作为 shell 脚本执行
      构造 /bin/sh -c argv[0] 参数列表
```

**PATH 搜索算法**：
1. 如果 `file` 包含 `/`，直接调用 `__execve`
2. 否则获取 `PATH` 环境变量（为空则用 `/bin:/usr/bin`）
3. 逐个 PATH 目录拼接文件名，尝试 `__execve`
4. 如果返回 `ENOEXEC`（ELF 格式错误），尝试作为 shell 脚本执行
5. 记录遇到的 `EACCES`/`ENOENT` 等错误，最后报告最相关的错误

### 2.4 fexecve — 通过 fd 执行

```
fexecve(fd, argv, envp)                 [sysdeps/unix/sysv/linux/fexecve.c:34]
├── INLINE_SYSCALL(execveat, fd, "", argv, envp, AT_EMPTY_PATH)
│     首先尝试 execveat 系统调用（Linux 3.19+）
├── __fd_to_filename(fd)                [misc/fd_to_filename.c:26]
│     构造 /proc/self/fd/N 路径
├── __execve("/proc/self/fd/N", argv, envp)
│     回退：通过 /proc 路径执行
└── __stat64(fd)                        [include/sys/stat.h:33]
      确认 fd 有效（区分 ENOENT vs EBADF）
```

---

## 3. posix_spawn 调用链

### 3.1 总体流程

```
posix_spawn(pid, path, actions, attr, argv, envp)
posix_spawnp(pid, file, actions, attr, argv, envp)
       │
       ▼
__posix_spawn()  [posix/spawn.c:25]
__posix_spawnp() [posix/spawnp.c:25]
       │
       ▼
__spawni()       [sysdeps/unix/sysv/linux/spawni.c:488]
       │
       ▼
__spawnix()      [sysdeps/unix/sysv/linux/spawni.c:313]
       │
       ├─── clone3 / clone (创建子进程)
       │        │
       │        ▼
       │    __spawni_child() (子进程空间)
       │        │
       │        ├── 信号处理重置
       │        ├── 进程属性设置
       │        ├── 文件操作执行
       │        ├── 信号掩码恢复
       │        └── execve (替换进程映像)
       │
       └─── waitid (等待子进程 exec 或失败)
```

### 3.2 __spawni — 分发入口

```
__spawni()                              [sysdeps/unix/sysv/linux/spawni.c:488]
└── __spawnix(pid, file, acts, attrp, argv, envp, xflags, exec_func)
      exec_func 选择逻辑：
        SPAWN_XFLAGS_USE_PATH ? __execvpex : __execve
```

- `posix_spawn` 传入 `__execve`（不搜索 PATH）
- `posix_spawnp` 传入 `__execvpex`（搜索 PATH，但不兼容脚本执行）

### 3.3 __spawnix — 核心实现

```
__spawnix()                             [sysdeps/unix/sysv/linux/spawni.c:313]
├── __clone_pidfd_supported()           [sysdeps/unix/sysv/linux/clone-pidfd-support.c:35]
│     检查内核是否支持 CLONE_PIDFD + P_PIDFD waitid
├── 参数计数（限制 INT_MAX - 1）
├── __mmap(NULL, stack_size, PROT_*, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK)
│     分配子进程栈（最少 32KB + argv 大小）
│                                       [sysdeps/unix/sysv/linux/mmap64.c:66]
├── __pthread_setcancelstate(PTHREAD_CANCEL_DISABLE)
│     禁用异步取消                      [nptl/pthread_setcancelstate.c:24]
├── __abort_lock_rdlock(&args.oldmask)
│     获取 abort 锁（防止与 abort 竞争） [stdlib/abort.c:53]
├── [设置 clone_args 结构体]
│     flags: CLONE_VM | CLONE_VFORK | CLONE_CLEAR_SIGHAND
│            [+ CLONE_INTO_CGROUP] [+ CLONE_PIDFD]
│     exit_signal: SIGCHLD
│     stack: mmap 分配的栈
│     cgroup: attrp->__cgroup (可选)
│
├── __clone3(&clone_args, sizeof, __spawni_child, &args)
│     首选 clone3 系统调用              [include/clone_internal.h:19]
│     ↓ 失败回退 (ENOSYS/EINVAL)
├── __clone_internal_fallback(&clone_args, __spawni_child, &args)
│     回退到 clone 系统调用             [sysdeps/unix/sysv/linux/clone-internal.c:47]
│     └── __clone(fn, stack, flags, arg, ...)
│                                       [include/sched.h:25]
│
├── [子进程 exec 失败处理]
│   └── __waitid(P_PIDFD/P_PID, pid, NULL, WEXITED)
│         等待失败的子进程退出          [sysdeps/unix/sysv/linux/waitid.c:25]
│       └── __close_nocancel_nostatus(args.pidfd)
│             关闭 pidfd（如果使用）
│
├── __munmap(stack, stack_size)          [include/sys/mman.h:12]
│     释放子进程栈
├── __abort_lock_unlock(&args.oldmask)  [stdlib/abort.c:67]
└── __pthread_setcancelstate(state, NULL)
      恢复取消状态
```

**关键设计决策**：

| 标志 | 作用 |
|------|------|
| `CLONE_VM` | 父子共享内存空间，避免 COW 页表复制开销 |
| `CLONE_VFORK` | 父进程挂起直到子进程 exec/exit，保证共享内存安全 |
| `CLONE_CLEAR_SIGHAND` | 内核自动清除所有信号处理器（Linux 5.5+） |
| `CLONE_INTO_CGROUP` | 直接将子进程放入指定 cgroup（Linux 5.7+） |
| `CLONE_PIDFD` | 返回进程文件描述符，用于无竞争等待 |

### 3.4 __spawni_child — 子进程执行

```
__spawni_child(arguments)               [sysdeps/unix/sysv/linux/spawni.c:103]
│
├── 1. 信号处理重置
│   ├── __sigprocmask(SIG_BLOCK, NULL, &hset)
│   │     获取当前信号掩码              [sysdeps/unix/sysv/linux/sigprocmask.c:23]
│   ├── [遍历所有信号 1..._NSIG]
│   │   ├── POSIX_SPAWN_SETSIGDEF 中的信号 → SIG_DFL
│   │   ├── 非 clone3 时有处理器的信号 → SIG_DFL
│   │   └── 内部信号 (SIGCANCEL/SIGSETXID) → SIG_IGN
│   └── __libc_sigaction(sig, &sa, NULL)
│         逐个重置                      [sysdeps/unix/sysv/linux/libc_sigaction.c:42]
│
├── 2. 进程属性设置 (POSIX_SPAWN_SET* flags)
│   ├── __sched_setparam(0, &attr->__sp)
│   │     设置调度参数                  [include/sched.h:6]
│   ├── __sched_setscheduler(0, policy, &sp)
│   │     设置调度策略                  [include/sched.h:11]
│   ├── __setsid()
│   │     创建新会话                    [include/unistd.h:122]
│   ├── __setpgid(0, pgrp)
│   │     设置进程组                    [include/unistd.h:133]
│   └── local_seteuid(__getuid()) + local_setegid(__getgid())
│         重置有效 UID/GID（POSIX_SPAWN_RESETIDS）
│
├── 3. 文件操作执行 (file_actions)
│   ├── spawn_do_close:
│   │   └── __close_nocancel(fd)        [sysdeps/unix/sysv/linux/close_nocancel.c:24]
│   ├── spawn_do_open:
│   │   ├── __close_nocancel(fd)        [先关闭目标 fd]
│   │   ├── __open_nocancel(path, oflag|O_LARGEFILE, mode)
│   │   │                               [sysdeps/unix/sysv/linux/open64_nocancel.c:46]
│   │   └── __dup2(new_fd, target_fd)   [sysdeps/unix/sysv/linux/dup2.c:26]
│   ├── spawn_do_dup2:
│   │   ├── fd == newfd: __fcntl(fd, F_SETFD, flags & ~FD_CLOEXEC)
│   │   │                               [sysdeps/unix/sysv/linux/fcntl64.c:63]
│   │   └── fd != newfd: __dup2(fd, newfd)
│   ├── spawn_do_chdir:
│   │   └── __chdir(path)              [include/unistd.h:85]
│   ├── spawn_do_fchdir:
│   │   └── __fchdir(fd)               [include/unistd.h:86]
│   ├── spawn_do_closefrom:
│   │   ├── close_range(lowfd, ~0U, 0) [直接 syscall，Linux 5.9+]
│   │   └── __closefrom_fallback(lowfd) [回退：遍历 /proc/self/fd]
│   │                                   [sysdeps/unix/sysv/linux/closefrom_fallback.c:31]
│   └── spawn_do_tcsetpgrp:
│       ├── __getpgid(0)               [获取当前进程组]
│       └── __tcsetpgrp(fd, pgrp)      [设置终端前台进程组]
│                                       [sysdeps/unix/bsd/tcsetpgrp.c:25]
│
├── 4. 信号掩码恢复
│   ├── POSIX_SPAWN_SETSIGMASK:
│   │   └── __sigprocmask(SIG_SETMASK, &attr->__ss, NULL)
│   └── 否则:
│       └── internal_sigprocmask(SIG_SETMASK, &args->oldmask, NULL)
│
├── 5. 执行程序
│   └── args->exec(file, argv, envp)
│         → __execve() 或 __execvpex()
│
└── 6. exec 失败处理
    ├── maybe_script_execute(args)      [sysdeps/unix/sysv/linux/spawni.c:77]
    │     ENOEXEC 时构造 /bin/sh 参数重试
    └── _exit(SPAWN_ERROR)              [sysdeps/unix/sysv/linux/_exit.c:26]
          设置 args->err = errno，通知父进程
```

### 3.5 __execvpex — PATH 搜索变体

```
__execvpex()                            [posix/execvpe.c:201]
└── __execvpe_common(file, argv, envp, __execve)
      使用 __execve 作为底层执行函数
      不执行脚本兼容（ENOEXEC 不重试）
      脚本执行留给 maybe_script_execute 处理
```

与 `__execvpe` 的区别：`__execvpex` 在遇到 `ENOEXEC` 时不会自行处理脚本执行，
而是返回错误让调用者（`__spawni_child`）的 `maybe_script_execute` 处理。

---

## 4. system() 调用链

### 4.1 __libc_system — 入口

```
__libc_system(line)                     [sysdeps/posix/system.c:205]
├── line == NULL: return 1              [检查 shell 可用性]
└── do_system(line)                     [sysdeps/posix/system.c:101]
```

### 4.2 do_system — 核心实现

```
do_system(line)                         [sysdeps/posix/system.c:101]
├── __sigemptyset(&reset)
├── __sigaddset(&reset, SIGINT/SIGQUIT) [准备阻塞信号]
├── __lll_lock_wait[_private]()         [获取 handler_lock]
│     保护信号处理器安装               [nptl/lowlevellock.c:25/40]
├── __sigaction(SIGINT, &sa_ign, &old_int)
│     忽略 SIGINT                       [signal/sigaction.c:26]
├── __sigaction(SIGQUIT, &sa_ign, &old_quit)
│     忽略 SIGQUIT
├── __lll_lock_wake[_private]()         [释放 handler_lock]
│                                       [nptl/lowlevellock.c:55/62]
├── __sigprocmask(SIG_BLOCK, &block, &omask)
│     阻塞 SIGCHLD                     [sysdeps/unix/sysv/linux/sigprocmask.c:23]
│
├── [设置 posix_spawn 属性]
│   ├── __posix_spawnattr_init(&spawn_attr)
│   ├── __posix_spawnattr_setsigmask(&spawn_attr, &omask)
│   ├── __posix_spawnattr_setsigdefault(&spawn_attr, &reset)
│   ├── __posix_spawnattr_setflags(&spawn_attr, SETSIGMASK|SETSIGDEF)
│   └── __posix_spawn(&pid, "/bin/sh", NULL, &spawn_attr,
│                     {"/bin/sh", "-c", line, NULL}, environ)
│                                       [posix/spawn.c:25]
│
├── __posix_spawnattr_destroy(&spawn_attr)
│
├── __libc_cleanup_push_defer(cancel_handler, &new_sa)
│     设置取消清理处理器               [nptl/libc-cleanup.c:23]
├── __waitpid(pid, &status, 0)          [posix/waitpid.c:36]
│     等待子进程结束
├── __libc_cleanup_pop_restore(0)       [nptl/libc-cleanup.c:53]
│
├── [恢复信号]
│   ├── __lll_lock_wait[_private]()     [获取 handler_lock]
│   ├── __sigaction(SIGINT, &old_int, NULL)
│   ├── __sigaction(SIGQUIT, &old_quit, NULL)
│   ├── __lll_lock_wake[_private]()     [释放 handler_lock]
│   └── __sigprocmask(SIG_SETMASK, &omask, NULL)
│
└── return status
```

**system() 的安全措施**：
1. **忽略 SIGINT/SIGQUIT** — 防止父进程被子进程的终端信号误杀
2. **阻塞 SIGCHLD** — 防止用户的 SIGCHLD handler 干扰 waitpid
3. **handler_lock** — 多线程并发调用 system() 时保护信号处理器恢复的原子性
4. **取消清理处理器** — 如果线程被取消，恢复信号处理和等待子进程

---

## 5. _exit — 进程终止

```
_exit(status)                           [sysdeps/unix/sysv/linux/_exit.c:26]
└── while(1) {
      INLINE_SYSCALL(exit_group, 1, status)  ← 终止所有线程
      ABORT_INSTRUCTION                       ← 不可达保护
    }
```

`exit_group` 终止进程中的所有线程，不执行任何清理（atexit、stdio flush 等）。

---

## 6. 关键设计分析

### 6.1 posix_spawn vs fork+exec 性能对比

| 维度 | fork + exec | posix_spawn |
|------|-------------|-------------|
| 地址空间 | COW 复制页表（大进程昂贵） | CLONE_VM 共享（零拷贝） |
| 信号处理 | 需手动重置 | clone3 CLONE_CLEAR_SIGHAND 自动清除 |
| 文件操作 | 用户代码手动处理 | 声明式 file_actions |
| 同步机制 | 无（需自建 pipe） | CLONE_VFORK 自动挂起 |
| 栈空间 | 继承父进程栈 | mmap 分配独立栈（32KB+） |
| 取消安全 | 需手动处理 | 自动禁用/恢复取消 |
| 错误报告 | 需 pipe 传递 errno | 共享内存 args.err |
| cgroup | 需额外 syscall | CLONE_INTO_CGROUP 一步到位 |

### 6.2 clone3 回退策略

```
                     clone3 (Linux 5.3+)
                         │
                    ENOSYS/EINVAL?
                         │
                    ┌────┴────┐
                    │  Yes    │  No → 成功
                    ▼         │
         __clone_internal_fallback
                    │
                    ▼
               __clone()  (传统接口)
                    │
                    ▼
              SYS_clone  (raw syscall)
```

回退时会丢失部分功能：
- `CLONE_CLEAR_SIGHAND` — 需要子进程手动遍历重置
- `CLONE_INTO_CGROUP` — 不支持回退，返回 `ENOTSUP`
- `CLONE_PIDFD` — 通过 `parent_tid` 模拟

### 6.3 exec 族统一调用图

```
┌─────────────────────────────────────────────────────────────┐
│                     用户可见接口                              │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│  execl   │  execle  │  execv   │ execlp   │    execvp       │
│  execl.c │ execle.c │ execv.c  │ execlp.c │   execvp.c      │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬────────────┘
     │          │          │          │          │
     │ va→argv  │ va→argv  │          │ va→argv  │
     ▼          ▼          ▼          ▼          ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│      __execve()      │  │        __execvpe()           │
│  (直接 path 执行)    │  │   (PATH 搜索 + 执行)        │
│  syscalls.list 生成  │  │   posix/execvpe.c:193        │
└──────────┬───────────┘  └──────────┬───────────────────┘
           │                         │
           │                         ▼
           │               __execvpe_common()
           │               ├── PATH 解析
           │               ├── 路径拼接
           │               ├── __execve() ←─┐
           │               └── maybe_script_execute()
           │                         │
           ▼                         ▼
┌────────────────────────────────────────────┐
│          SYS_execve (内核)                  │
│  替换进程映像：text/data/bss/heap/stack    │
│  保留：PID, 打开的 fd (非 CLOEXEC),       │
│        信号掩码, 进程组, 会话               │
└────────────────────────────────────────────┘
```

### 6.4 file_actions 操作类型

| 操作 | 系统调用 | 用途 |
|------|---------|------|
| `spawn_do_close` | `close()` | 关闭不需要的 fd |
| `spawn_do_open` | `open()` + `dup2()` | 重定向 stdin/stdout/stderr |
| `spawn_do_dup2` | `dup2()` / `fcntl(F_SETFD)` | fd 复制或清除 CLOEXEC |
| `spawn_do_chdir` | `chdir()` | 更改工作目录 |
| `spawn_do_fchdir` | `fchdir()` | 通过 fd 更改工作目录 |
| `spawn_do_closefrom` | `close_range()` / 遍历 | 批量关闭高位 fd |
| `spawn_do_tcsetpgrp` | `tcsetpgrp()` | 设置终端前台进程组 |

### 6.5 错误传播机制

由于 `posix_spawn` 使用 `CLONE_VM`（共享内存），子进程和父进程直接通过
共享的 `args.err` 变量通信：

```
父进程                              子进程
  │                                   │
  ├── args.err = 0                    │
  ├── clone3(CLONE_VM|CLONE_VFORK)────┤
  │   [挂起]                          ├── 设置属性
  │                                   ├── 执行 file_actions
  │                                   ├── execve()
  │                                   │   ├── 成功: 不返回
  │                                   │   └── 失败:
  │                                   │       args.err = errno
  │                                   └── _exit(SPAWN_ERROR)
  ├── [恢复: CLONE_VFORK 完成]        
  ├── 检查 args.err                   
  │   ├── > 0: 失败, waitid() 回收
  │   └── == 0: 成功, *pid = new_pid
  └── return ec
```

---

## 7. 涉及的源文件

| 源文件 | 内容 |
|--------|------|
| `posix/spawn.c` | `posix_spawn` 入口 |
| `posix/spawnp.c` | `posix_spawnp` 入口 |
| `sysdeps/unix/sysv/linux/spawni.c` | `__spawni`/`__spawnix`/`__spawni_child` 核心实现 |
| `posix/execve.c` | `__execve` stub（被 syscalls.list 覆盖） |
| `posix/execvpe.c` | `__execvpe`/`__execvpex`/`__execvpe_common` PATH 搜索 |
| `posix/execl.c` | `execl` va_list → argv 转换 |
| `posix/execlp.c` | `execlp` va_list + PATH |
| `posix/execle.c` | `execle` va_list + 显式 envp |
| `posix/execv.c` | `execv` 薄封装 |
| `posix/execvp.c` | `execvp` → `__execvpe` |
| `sysdeps/unix/sysv/linux/fexecve.c` | `fexecve` execveat/proc 实现 |
| `sysdeps/posix/system.c` | `system()` 完整实现 |
| `sysdeps/unix/sysv/linux/_exit.c` | `_exit` exit_group syscall |
| `sysdeps/unix/sysv/linux/clone-internal.c` | clone3 回退逻辑 |
| `sysdeps/unix/sysv/linux/clone-pidfd-support.c` | pidfd 支持检测 |

---

## 8. 与其他子系统的交互

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐
│  信号子系统  │────→│ posix_spawn  │←────│  线程子系统     │
│ sigaction    │     │              │     │ cancel_disable  │
│ sigprocmask  │     │              │     │ lowlevellock    │
└─────────────┘     └──────┬───────┘     └────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
      ┌──────────┐  ┌──────────┐  ┌──────────┐
      │ clone3   │  │  mmap    │  │  execve  │
      │ (进程)   │  │  (栈)    │  │  (执行)  │
      └──────────┘  └──────────┘  └──────────┘
              └────────────┼────────────┘
                           ▼
                     ┌──────────┐
                     │  内核    │
                     └──────────┘
```

---

> 本文档基于 clangd 调用层次分析生成，覆盖 glibc 2.43 中 exec 族和 posix_spawn 的完整实现路径。
