# open/read/write 文件 I/O 调用链

> 基于 glibc 2.43.9000，分析 POSIX 文件 I/O API 的实现架构  
> 分析工具：源码阅读 + 宏展开追踪

---

## 概述

glibc 文件 I/O 层是 Linux 系统调用的**直接封装**，核心设计与 socket 层一致：

| 特性 | 实现方式 |
|------|----------|
| 取消点 | `SYSCALL_CANCEL` 包装所有可能阻塞的 I/O 操作 |
| 非取消版本 | `__xxx_nocancel` → `INLINE_SYSCALL_CALL`（供 glibc 内部使用） |
| LFS 透明 | 64位系统 `open` ≡ `open64`（`__OFF_T_MATCHES_OFF64_T`） |
| openat 统一 | `open()` 内部实现为 `openat(AT_FDCWD, ...)`（无独立 open syscall） |

---

## 1. 系统调用分类

### 1.1 取消点 API（SYSCALL_CANCEL）

| API | syscall | 源文件 | 取消条件 |
|-----|---------|--------|----------|
| `open()` / `open64()` | `openat` | open.c / open64.c | 等待设备就绪、NFS 挂载 |
| `openat()` / `openat64()` | `openat` | openat.c / openat64.c | 同上 |
| `read()` | `read` | read.c | 等待数据 |
| `write()` | `write` | write.c | 等待缓冲区空间 |
| `close()` | `close` | close.c | 等待写回完成 |
| `pread64()` | `pread64` | pread64.c | 同 read |
| `pwrite64()` | `pwrite64` | pwrite64.c | 同 write |
| `readv()` | `readv` | readv.c | 同 read |
| `writev()` | `writev` | writev.c | 同 write |
| `fcntl(F_SETLKW)` | `fcntl64` | fcntl64.c | 等待文件锁 |

### 1.2 非取消点 API（INLINE_SYSCALL_CALL）

| API | syscall | 源文件 |
|-----|---------|--------|
| `lseek()` / `lseek64()` | `lseek` / `_llseek` | lseek64.c |
| `fstat()` / `fstat64()` | `fstat` / `fstat64` | fstat64.c |
| `dup2()` | `dup2` / `dup3` | dup2.c |
| `fcntl(其他cmd)` | `fcntl64` | fcntl64.c |

### 1.3 内部非取消版本（供 glibc 自身使用）

| 函数 | syscall | 调用场景 |
|------|---------|----------|
| `__open_nocancel()` | `openat` (INLINE) | ld.so 加载、locale 读取 |
| `__read_nocancel()` | `read` (INLINE) | nscd、/proc 读取 |
| `__write_nocancel()` | `write` (INLINE) | 错误输出 |
| `__close_nocancel()` | `close` (INLINE) | 信号处理器中关闭 fd |
| `__close_nocancel_nostatus()` | `close` (INLINE) | 忽略返回值的关闭 |

---

## 2. 核心调用链

### 2.1 open() — 打开文件（取消点）

```
open(file, oflag, ...)               // 用户调用
│
├── [64位系统] __OFF_T_MATCHES_OFF64_T → 直接是 open64
│
└── __libc_open64(file, oflag, ...)  [sysdeps/unix/sysv/linux/open64.c:29]
    │
    ├── [O_CREAT | O_TMPFILE] → va_arg 提取 mode 参数
    │
    ├── oflag |= O_LARGEFILE         // 64位文件强制标志
    │
    └── SYSCALL_CANCEL(openat, AT_FDCWD, file, oflag, mode)
        │
        └── __syscall_cancel(AT_FDCWD, file, oflag|O_LARGEFILE, mode, 0, 0, __NR_openat)
            │
            ├── [取消检查] cancelhandling & CANCELED → __syscall_do_cancel()
            ├── [cancelpc 区间开始]
            ├── syscall(__NR_openat, AT_FDCWD, file, oflag, mode)
            └── [cancelpc 区间结束]
```

**关键设计**：
- Linux 现代内核不再有独立的 `open` syscall，统一使用 `openat`
- `AT_FDCWD` (-100) 表示相对于当前工作目录
- 64位系统上 `open` / `open64` / `__open` / `__open64` 全部指向同一实现

### 2.2 openat() — 相对路径打开

```
openat(dirfd, file, oflag, ...)
│
└── __libc_openat64(dirfd, file, oflag, ...)  [sysdeps/unix/sysv/linux/openat64.c:28]
    │
    └── SYSCALL_CANCEL(openat, dirfd, file, oflag | O_LARGEFILE, mode)
```

### 2.3 read() — 读取数据（取消点）

```
read(fd, buf, nbytes)
│
└── __libc_read(fd, buf, nbytes)  [sysdeps/unix/sysv/linux/read.c:24]
    │
    └── SYSCALL_CANCEL(read, fd, buf, nbytes)
        └── __syscall_cancel(fd, buf, nbytes, 0, 0, 0, __NR_read)
```

### 2.4 write() — 写入数据（取消点）

```
write(fd, buf, nbytes)
│
└── __libc_write(fd, buf, nbytes)  [sysdeps/unix/sysv/linux/write.c:24]
    │
    └── SYSCALL_CANCEL(write, fd, buf, nbytes)
        └── __syscall_cancel(fd, buf, nbytes, 0, 0, 0, __NR_write)
```

### 2.5 close() — 关闭文件（取消点）

```
close(fd)
│
└── __close(fd)  [sysdeps/unix/sysv/linux/close.c:25]
    │
    └── SYSCALL_CANCEL(close, fd)
        └── __syscall_cancel(fd, 0, 0, 0, 0, 0, __NR_close)
```

**注意**：`close` 是取消点是因为 POSIX 规定，但实际上 Linux 的 `close` 几乎不阻塞（NFS 除外）。

### 2.6 pread64() / pwrite64() — 定位读写（取消点）

```
pread64(fd, buf, count, offset)
│
└── __libc_pread64(fd, buf, count, offset)  [sysdeps/unix/sysv/linux/pread64.c:25]
    │
    └── SYSCALL_CANCEL(pread64, fd, buf, count, SYSCALL_LL64_PRW(offset))
        // SYSCALL_LL64_PRW: 将 64位 offset 适配为 syscall 参数格式
        // 64位系统: 直接传递
        // 32位系统: 拆分为两个 32位参数

pwrite64(fd, buf, count, offset)
│
└── __libc_pwrite64(fd, buf, count, offset)  [sysdeps/unix/sysv/linux/pwrite64.c:25]
    │
    └── SYSCALL_CANCEL(pwrite64, fd, buf, count, SYSCALL_LL64_PRW(offset))
```

### 2.7 readv() / writev() — 聚散 I/O（取消点）

```
readv(fd, iov, iovcnt)
│
└── __readv(fd, iov, iovcnt)  [sysdeps/unix/sysv/linux/readv.c]
    └── SYSCALL_CANCEL(readv, fd, iov, iovcnt)

writev(fd, iov, iovcnt)
│
└── __writev(fd, iov, iovcnt)  [sysdeps/unix/sysv/linux/writev.c]
    └── SYSCALL_CANCEL(writev, fd, iov, iovcnt)
```

### 2.8 lseek64() — 文件定位（非取消点）

```
lseek(fd, offset, whence)
│
└── __lseek64(fd, offset, whence)  [sysdeps/unix/sysv/linux/lseek64.c]
    │
    ├── [64位内核 __NR_lseek]
    │   └── INLINE_SYSCALL_CALL(lseek, fd, offset, whence)
    │
    └── [32位内核 需要 _llseek]
        └── INLINE_SYSCALL_CALL(_llseek, fd, offset>>32, offset&0xFFFFFFFF,
                                &result, whence)
```

### 2.9 fcntl() — 文件控制（条件取消点）

```
fcntl(fd, cmd, ...)
│
└── __libc_fcntl64(fd, cmd, arg)  [sysdeps/unix/sysv/linux/fcntl64.c:49]
    │
    ├── cmd = FCNTL_ADJUST_CMD(cmd)       // 平台特定调整
    │
    ├── [cmd == F_SETLKW | F_SETLKW64 | F_OFD_SETLKW]
    │   └── SYSCALL_CANCEL(fcntl64, fd, cmd, arg)   // 等待锁 → 取消点
    │
    └── [其他 cmd]
        └── __fcntl64_nocancel_adjusted(fd, cmd, arg)
            └── INLINE_SYSCALL_CALL(fcntl64, fd, cmd, arg)  // 非取消点
```

**只有等待文件锁的 cmd 是取消点**，其他如 `F_GETFL`/`F_SETFL`/`F_DUPFD` 不阻塞。

---

## 3. 非取消版本（__xxx_nocancel）

### 3.1 设计用途

glibc 内部操作（如读取 `/proc`、打开 locale 文件）不应被线程取消中断，因此使用不检查取消标志的版本：

```
__open_nocancel(file, oflag, ...)  [sysdeps/unix/sysv/linux/open_nocancel.c:30]
│
└── INLINE_SYSCALL_CALL(openat, AT_FDCWD, file, oflag, mode)
    └── syscall(__NR_openat, ...)    // 直接 syscall，不检查 cancelhandling

__read_nocancel(fd, buf, nbytes)   [sysdeps/unix/sysv/linux/read_nocancel.c:24]
│
└── INLINE_SYSCALL_CALL(read, fd, buf, nbytes)

__write_nocancel(fd, buf, nbytes)  [sysdeps/unix/sysv/linux/write_nocancel.c:24]
│
└── INLINE_SYSCALL_CALL(write, fd, buf, nbytes)

__close_nocancel(fd)               [sysdeps/unix/sysv/linux/close_nocancel.c:24]
│
└── INLINE_SYSCALL_CALL(close, fd)
```

### 3.2 使用场景

| 调用者 | 使用的 nocancel 函数 | 原因 |
|--------|---------------------|------|
| ld.so (动态链接器) | `__open_nocancel` | rtld 中无取消支持 |
| nscd (名称缓存) | `__read_nocancel` | 守护进程不应被取消 |
| locale 加载 | `__open_nocancel` + `__read_nocancel` | 初始化阶段无取消 |
| `/proc` 读取 | `__open_nocancel` + `__read_nocancel` | 内部操作 |
| abort() 输出 | `__write_nocancel` | 不可中断的错误报告 |
| 信号处理器中 | `__close_nocancel` | 信号处理器中不应有取消点 |
| `__close_nocancel_nostatus` | 忽略返回值 | 清理路径，失败无影响 |

---

## 4. LFS（大文件支持）透明化

### 4.1 64位系统

在 64 位系统上（`__OFF_T_MATCHES_OFF64_T` 已定义）：

```c
// open64.c 中：
strong_alias (__libc_open64, __libc_open)
strong_alias (__libc_open64, __open)
weak_alias (__libc_open64, open)
```

即 `open` ≡ `open64` ≡ `__open` ≡ `__libc_open`，只有一份实现。

### 4.2 32位系统

在 32 位系统上需要两个版本：
- `open()` → `SYSCALL_CANCEL(openat, AT_FDCWD, file, oflag, mode)` — 无 `O_LARGEFILE`
- `open64()` → `SYSCALL_CANCEL(openat, AT_FDCWD, file, oflag | O_LARGEFILE, mode)`

`O_LARGEFILE` 告诉内核允许文件大小 > 2GB。

### 4.3 pread/pwrite 的 64位参数适配

```c
// SYSCALL_LL64_PRW(offset) 宏：
// 64位系统：直接传递 offset
// 32位系统：将 off64_t 拆分为 (hi32, lo32) 两个参数
//           某些架构还需要对齐到偶数寄存器
```

---

## 5. 符号别名层次

### 5.1 open 族别名关系（64位系统）

```
用户可见符号        →  内部实现
─────────────────────────────────
open               →  __libc_open64
open64             →  __libc_open64
__open             →  __libc_open64
__open64           →  __libc_open64
__libc_open        →  __libc_open64
```

### 5.2 read/write 别名关系

```
read               →  __libc_read      (static_weak_alias)
__read             →  __libc_read      (strong_alias)

write              →  __libc_write     (static_weak_alias)
__write            →  __libc_write     (strong_alias)
```

### 5.3 别名设计目的

| 符号前缀 | 用途 |
|----------|------|
| 无前缀 `open` | 用户可见 API |
| `__open` | glibc 内部使用（libc_hidden_weak） |
| `__libc_open` | 内部 libc 强引用 |
| `__open_nocancel` | 内部无取消版本 |

---

## 6. 与 stdio 的关系

```
用户层:  fopen → fread → fwrite → fclose
         │         │        │        │
stdio层: _IO_new_file_fopen  _IO_new_file_close_it
         │         │        │        │
         │         ▼        ▼        │
         │   _IO_new_file_underflow  │
         │   _IO_new_file_overflow   │
         │         │        │        │
 I/O层:  ▼         ▼        ▼        ▼
        __open    __read   __write  __close
         │         │        │        │
         ▼         ▼        ▼        ▼
       openat     read     write    close
      (syscall)  (syscall) (syscall)(syscall)
```

stdio 是文件 I/O 之上的**用户空间缓冲层**：
- `fread` → 缓冲区为空时调用 `__read` 填充
- `fwrite` → 缓冲区满时调用 `__write` 刷新
- `fopen` → 调用 `__open` 获取 fd
- `fclose` → 刷新缓冲 + 调用 `__close`

---

## 7. 取消点 vs 非取消点决策

### 7.1 POSIX 规定的取消点

POSIX.1-2017 §2.9.5.2 明确列出以下为取消点：

```
open, openat, read, write, close, pread, pwrite,
readv, writev, fsync, fdatasync, fcntl(F_SETLKW*),
creat, ...
```

### 7.2 glibc 实现策略

```
┌───────────────────────────────────────────────────────────────┐
│           POSIX 规定为取消点的操作                              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  SYSCALL_CANCEL(name, args...)                                │
│  = __syscall_cancel(args..., __NR_name)                       │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 1. 检查 pd->cancelhandling                              │  │
│  │    - CANCELED + ASYNC_TYPE → 立即取消                    │  │
│  │ 2. 标记 cancelpc_start                                  │  │
│  │ 3. 执行 raw syscall                                     │  │
│  │ 4. 标记 cancelpc_end                                    │  │
│  │ 5. 正常返回结果                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  若在步骤 3 被 SIGCANCEL 中断:                                 │
│  → sigcancel_handler 检查 PC ∈ [start, end)                  │
│  → 触发 __syscall_do_cancel()                                │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│           非取消点操作 / glibc 内部使用                         │
├───────────────────────────────────────────────────────────────┤
│  INLINE_SYSCALL_CALL(name, args...)                           │
│  = direct syscall + errno 设置                                │
│  不检查 cancelhandling，不可被 SIGCANCEL 取消                  │
└───────────────────────────────────────────────────────────────┘
```

---

## 8. 源文件布局

```
io/                                  — 通用 stub（非 Linux 特定）
├── open.c                           — stub: __libc_open
├── open64.c                         — stub: __libc_open64
├── read.c                           — stub: __libc_read
├── write.c                          — stub: __libc_write
├── close.c                          — stub: __close
├── lseek.c / lseek64.c             — stub
├── dup.c / dup2.c / dup3.c         — stub
├── fcntl.c                          — stub
├── fstat.c / stat.c / lstat.c      — stub
└── openat.c / openat64.c           — stub

sysdeps/unix/sysv/linux/             — Linux 实际实现
├── open.c                           — SYSCALL_CANCEL(openat, AT_FDCWD, ...)
├── open64.c                         — + O_LARGEFILE
├── openat.c / openat64.c           — SYSCALL_CANCEL(openat, fd, ...)
├── read.c                           — SYSCALL_CANCEL(read, ...)
├── write.c                          — SYSCALL_CANCEL(write, ...)
├── close.c                          — SYSCALL_CANCEL(close, ...)
├── pread64.c / pwrite64.c          — SYSCALL_CANCEL + LL64_PRW
├── readv.c / writev.c              — SYSCALL_CANCEL(readv/writev, ...)
├── lseek64.c                        — INLINE_SYSCALL(lseek / _llseek)
├── fstat64.c / fstatat64.c         — INLINE_SYSCALL(fstat)
├── fcntl64.c                        — 条件取消(F_SETLKW) + nocancel
├── dup2.c / dup3.c                 — INLINE_SYSCALL
│
├── open_nocancel.c                  — __open_nocancel → INLINE_SYSCALL
├── read_nocancel.c                  — __read_nocancel → INLINE_SYSCALL
├── write_nocancel.c                 — __write_nocancel → INLINE_SYSCALL
├── close_nocancel.c                 — __close_nocancel → INLINE_SYSCALL
└── close_nocancel_nostatus.c        — 忽略返回值版本

sysdeps/unix/sysdep.h                — SYSCALL_CANCEL / INLINE_SYSCALL_CALL 宏定义
sysdeps/unix/sysv/linux/not-cancel.h — nocancel 函数声明
```

---

## 9. 设计总结

| 设计决策 | 原因 |
|----------|------|
| 所有 open 统一到 `openat` | Linux 移除了独立 `open` syscall number (AArch64 等) |
| O_LARGEFILE 自动添加 | 64位系统上 off_t == off64_t，无需用户关心 |
| close 作为取消点 | POSIX 规定，虽然实际很少阻塞 |
| nocancel 版本独立文件 | 编译单元隔离，避免影响普通版本的优化 |
| lseek 不是取消点 | 纯内核数据结构操作，不涉及 I/O 等待 |
| fcntl 条件取消 | 只有 F_SETLKW（等待锁）可能阻塞，其他不阻塞 |
| strong_alias + weak_alias | 支持 LD_PRELOAD 拦截 + 内部直接调用 |
| hidden_def | 避免 PLT 跳转，libc 内部调用走直接地址 |
