# _GNU_SOURCE 宏分析

> 基于 glibc 2.43.9000 源码分析  
> 分析 `_GNU_SOURCE` 特性测试宏的机制、影响范围和开启的功能

---

## 目录

1. [机制概述](#1-机制概述)
2. [features.h 处理流程](#2-featuresh-处理流程)
3. [开启的功能分类汇总](#3-开启的功能分类汇总)
4. [各头文件详细影响](#4-各头文件详细影响)
5. [与其他特性宏的关系](#5-与其他特性宏的关系)
6. [使用建议与注意事项](#6-使用建议与注意事项)

---

## 1. 机制概述

### 1.1 什么是 _GNU_SOURCE

`_GNU_SOURCE` 是 glibc 提供的**最大化特性测试宏**。用户在包含任何系统头文件之前
定义此宏，即可一次性开启 glibc 提供的全部扩展功能：

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
// 现在可以使用 GNU 扩展函数：mempcpy、rawmemchr、pipe2 等
```

### 1.2 内部实现原理

```
用户定义 _GNU_SOURCE
         │
         ▼
include/features.h:218-245  ← 首次包含任何系统头文件时触发
         │
         ├─ 开启所有 ISO C 标准:  _ISOC95/99/11/23/2Y_SOURCE
         ├─ 开启完整 POSIX:      _POSIX_SOURCE + _POSIX_C_SOURCE=202405L
         ├─ 开启 X/Open:         _XOPEN_SOURCE=800
         ├─ 开启大文件:           _LARGEFILE64_SOURCE
         ├─ 开启默认扩展:         _DEFAULT_SOURCE
         ├─ 开启 AT 接口:        _ATFILE_SOURCE
         └─ 开启动态栈大小:       _DYNAMIC_STACK_SIZE_SOURCE
         │
         ▼
include/features.h:445-446
         │
         └─ 定义 __USE_GNU=1    ← 这是头文件中实际使用的条件守卫
```

核心内部宏 `__USE_GNU` 在 98 个头文件中共出现 **227 个条件守卫块**，
控制着数百个 GNU 扩展函数、常量和类型的可见性。

### 1.3 关键源码位置

| 文件 | 行号 | 内容 |
|------|------|------|
| `include/features.h` | 218-245 | `_GNU_SOURCE` → 开启所有子特性宏 |
| `include/features.h` | 445-446 | `_GNU_SOURCE` → `__USE_GNU=1` |
| `include/features.h` | 90-111 | 所有 `__USE_*` 内部宏的说明 |

---

## 2. features.h 处理流程

### 2.1 宏展开链

```c
// include/features.h:218-245
#ifdef _GNU_SOURCE
# define _ISOC95_SOURCE   1
# define _ISOC99_SOURCE   1
# define _ISOC11_SOURCE   1
# define _ISOC23_SOURCE   1
# define _ISOC2Y_SOURCE   1
# define _POSIX_SOURCE    1
# define _POSIX_C_SOURCE  202405L    // POSIX.1-2024
# define _XOPEN_SOURCE    800        // XPG8
# define _XOPEN_SOURCE_EXTENDED 1
# define _LARGEFILE64_SOURCE    1
# define _DEFAULT_SOURCE  1
# define _ATFILE_SOURCE   1
# define _DYNAMIC_STACK_SIZE_SOURCE 1
#endif
```

### 2.2 _GNU_SOURCE 间接开启的内部宏

通过上述子特性宏的级联效应，最终开启的 `__USE_*` 内部宏：

| 内部宏 | 控制内容 | 来源 |
|--------|---------|------|
| `__USE_GNU` | GNU 独有扩展 | `_GNU_SOURCE` 直接 |
| `__USE_ISOC99` | C99 函数 | `_ISOC99_SOURCE` |
| `__USE_ISOC11` | C11 函数 | `_ISOC11_SOURCE` |
| `__USE_POSIX` | POSIX.1-1990 | `_POSIX_C_SOURCE >= 1` |
| `__USE_POSIX2` | POSIX.2 | `_POSIX_C_SOURCE >= 2` |
| `__USE_POSIX199309` | POSIX.1b 实时扩展 | `_POSIX_C_SOURCE >= 199309L` |
| `__USE_POSIX199506` | POSIX.1c 线程 | `_POSIX_C_SOURCE >= 199506L` |
| `__USE_XOPEN` | X/Open 基础 | `_XOPEN_SOURCE` |
| `__USE_XOPEN_EXTENDED` | X/Open 扩展 | `_XOPEN_SOURCE_EXTENDED` |
| `__USE_UNIX98` | Single UNIX V2 | `_XOPEN_SOURCE >= 500` |
| `__USE_XOPEN2K` | XPG6 | `_XOPEN_SOURCE >= 600` |
| `__USE_XOPEN2K8` | XPG7 | `_XOPEN_SOURCE >= 700` |
| `__USE_XOPEN2K24` | XPG8 | `_XOPEN_SOURCE >= 800` |
| `__USE_LARGEFILE64` | 64位文件接口 | `_LARGEFILE64_SOURCE` |
| `__USE_MISC` | BSD/SysV 混合扩展 | `_DEFAULT_SOURCE` |
| `__USE_ATFILE` | `*at()` 函数族 | `_ATFILE_SOURCE` |
| `__USE_DYNAMIC_STACK_SIZE` | 动态栈大小常量 | `_DYNAMIC_STACK_SIZE_SOURCE` |
| `__GLIBC_USE_ISOC23` | C23 特性 | `_ISOC23_SOURCE` |
| `__GLIBC_USE_ISOC2Y` | C2Y 特性 | `_ISOC2Y_SOURCE` |

**总结**：`_GNU_SOURCE` = 一切可用特性的超集。

---

## 3. 开启的功能分类汇总

以下列出 `__USE_GNU`（非其他子宏）**独有守卫**的功能，即只有定义 `_GNU_SOURCE` 才能使用的功能。

### 3.1 进程/线程管理

| 函数/宏 | 头文件 | 说明 |
|---------|--------|------|
| `clone()` | `<sched.h>` | Linux 进程/线程创建底层接口 |
| `unshare()` | `<sched.h>` | 分离命名空间 |
| `setns()` | `<sched.h>` | 加入指定命名空间 |
| `SCHED_BATCH/IDLE/ISO` | `<sched.h>` | 额外调度策略常量 |
| `CPU_SET/CLR/ISSET/ZERO` | `<sched.h>` | CPU 亲和性宏集 |
| `gettid()` | `<unistd.h>` | 获取内核线程 ID |
| `_Fork()` | `<unistd.h>` | async-signal-safe 的 fork |
| `execvpe()` | `<unistd.h>` | 带环境变量的 PATH 搜索执行 |
| `pipe2()` | `<unistd.h>` | 带标志的管道创建 |
| `dup3()` | `<unistd.h>` | 带标志的文件描述符复制 |
| `getresuid/getresgid()` | `<unistd.h>` | 获取 real/effective/saved UID/GID |
| `syncfs()` | `<unistd.h>` | 同步文件系统 |
| `euidaccess()` | `<unistd.h>` | 用 effective UID 检查访问权限 |
| `get_current_dir_name()` | `<unistd.h>` | malloc 分配的当前目录名 |
| `environ` | `<unistd.h>` | 全局环境变量数组 |

### 3.2 pthread 扩展

| 函数/宏 | 说明 |
|---------|------|
| `pthread_tryjoin_np()` | 非阻塞 join |
| `pthread_timedjoin_np()` | 带超时 join |
| `pthread_getname_np/setname_np()` | 获取/设置线程名 |
| `pthread_attr_setaffinity_np()` | 设置线程 CPU 亲和性 |
| `pthread_attr_getaffinity_np()` | 获取线程 CPU 亲和性 |
| `pthread_setaffinity_np()` | 运行时设置亲和性 |
| `pthread_getaffinity_np()` | 运行时获取亲和性 |
| `pthread_yield()` | 让出 CPU（已废弃，等价 sched_yield） |
| `pthread_mutex_clocklock()` | 指定时钟源的互斥锁等待 |
| `pthread_rwlock_clockrdlock()` | 指定时钟源的读写锁 |
| `pthread_cond_clockwait()` | 指定时钟源的条件变量等待 |
| `pthread_gettid_np()` | 获取线程的内核 TID |
| `PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP` | 递归互斥锁静态初始化器 |
| `PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP` | 错误检查互斥锁初始化器 |
| `PTHREAD_MUTEX_FAST_NP` | 互斥锁类型（兼容名） |

### 3.3 文件 I/O 扩展

| 函数/宏 | 头文件 | 说明 |
|---------|--------|------|
| `O_DIRECT` | `<fcntl.h>` | 直接磁盘 I/O |
| `O_NOATIME` | `<fcntl.h>` | 不更新访问时间 |
| `O_PATH` | `<fcntl.h>` | 仅解析路径名 |
| `O_TMPFILE` | `<fcntl.h>` | 原子创建匿名文件 |
| `F_OFD_GETLK/SETLK/SETLKW` | `<fcntl.h>` | Open File Description 锁 |
| `F_SETLEASE/GETLEASE` | `<fcntl.h>` | 文件租约（lease） |
| `F_NOTIFY` | `<fcntl.h>` | 目录变更通知 |
| `F_SETPIPE_SZ/GETPIPE_SZ` | `<fcntl.h>` | 管道大小控制 |
| `F_ADD_SEALS/GET_SEALS` | `<fcntl.h>` | 文件密封（memfd） |
| `F_DUPFD_QUERY` | `<fcntl.h>` | 文件描述符比较 |
| `SEEK_DATA/SEEK_HOLE` | `<stdio.h>` | 稀疏文件空洞定位 |
| `RENAME_NOREPLACE/EXCHANGE/WHITEOUT` | `<stdio.h>` | renameat2 标志 |
| `fcloseall()` | `<stdio.h>` | 关闭所有流 |
| `fgets_unlocked()` | `<stdio.h>` | 无锁版 fgets |
| `fputs_unlocked()` | `<stdio.h>` | 无锁版 fputs |
| `obstack_printf/vprintf()` | `<stdio.h>` | obstack 格式化输出 |
| `scandirat()` | `<dirent.h>` | 相对目录扫描 |
| `versionsort()` | `<dirent.h>` | 版本感知排序 |
| `DN_ACCESS/MODIFY/CREATE/...` | `<fcntl.h>` | 目录通知类型常量 |
| `LOCK_MAND/READ/WRITE/RW` | `<fcntl.h>` | 强制锁（已废弃） |

### 3.4 内存管理扩展

| 函数/宏 | 头文件 | 说明 |
|---------|--------|------|
| `mremap()` | `<sys/mman.h>` | 重映射虚拟内存区域 |
| `MREMAP_MAYMOVE/FIXED/DONTUNMAP` | `<sys/mman.h>` | mremap 标志 |
| `mmap 扩展标志` | `<bits/mman-shared.h>` | Linux 特有 mmap 标志 |

### 3.5 字符串处理扩展

| 函数 | 头文件 | 说明 |
|------|--------|------|
| `rawmemchr()` | `<string.h>` | 无长度限制的 memchr（必须找到） |
| `mempcpy()` | `<string.h>` | 返回目标尾部的 memcpy |
| `strchrnul()` | `<string.h>` | 找不到返回 NUL 位置 |
| `memmem()` | `<string.h>` | 内存块中搜索子串 |
| `strverscmp()` | `<string.h>` | 版本号字符串比较 |
| `strerrordesc_np()` | `<string.h>` | 错误号→描述字符串（不使用 locale） |
| `strerrorname_np()` | `<string.h>` | 错误号→名称字符串（如 "ENOENT"） |
| `sigabbrev_np()` | `<string.h>` | 信号号→缩写（如 "HUP"） |
| `sigdescr_np()` | `<string.h>` | 信号号→描述字符串 |

### 3.6 动态链接扩展

| 函数/类型 | 头文件 | 说明 |
|-----------|--------|------|
| `dlmopen()` | `<dlfcn.h>` | 在指定命名空间中加载库 |
| `dladdr1()` | `<dlfcn.h>` | dladdr 扩展（返回符号/link_map） |
| `dlinfo()` | `<dlfcn.h>` | 查询已加载库的元信息 |
| `Lmid_t` | `<dlfcn.h>` | 命名空间 ID 类型 |
| `RTLD_DI_*` 常量 | `<dlfcn.h>` | dlinfo 请求类型 |
| `_dl_find_object()` | `<dlfcn.h>` | 快速查找地址所在对象 |
| `dl_iterate_phdr()` | `<link.h>` | 遍历所有已加载对象 |

### 3.7 网络/套接字扩展

| 函数/类型 | 头文件 | 说明 |
|-----------|--------|------|
| `accept4()` | `<sys/socket.h>` | 带标志的 accept |
| `sendmmsg()` | `<sys/socket.h>` | 批量发送消息 |
| `recvmmsg()` | `<sys/socket.h>` | 批量接收消息 |
| `struct mmsghdr` | `<sys/socket.h>` | 批量消息头结构 |
| `IN6_IS_ADDR_*` 扩展宏 | `<netinet/in.h>` | IPv6 地址判断宏 |

### 3.8 信号处理扩展

| 函数 | 头文件 | 说明 |
|------|--------|------|
| `sysv_signal()` | `<signal.h>` | System V 风格信号处理 |
| `sigisemptyset()` | `<signal.h>` | 判断信号集是否为空 |
| `sigandset()` | `<signal.h>` | 信号集与运算 |
| `sigorset()` | `<signal.h>` | 信号集或运算 |

### 3.9 时间与定时器

| 函数 | 头文件 | 说明 |
|------|--------|------|
| `strptime_l()` | `<time.h>` | 带 locale 的时间解析 |
| `getdate_r()` | `<time.h>` | getdate 的线程安全版本 |
| `timespec_get/getres` 扩展 | `<time.h>` | 高精度时间获取 |
| `TIMER_ABSTIME` | `<time.h>` | 绝对时间标志 |

### 3.10 标准库扩展

| 函数/类型 | 头文件 | 说明 |
|-----------|--------|------|
| `secure_getenv()` | `<stdlib.h>` | SUID 安全的环境变量获取 |
| `canonicalize_file_name()` | `<stdlib.h>` | malloc 版 realpath |
| `mkostemp/mkostemps()` | `<stdlib.h>` | 带标志的临时文件创建 |
| `qsort_r()` | `<stdlib.h>` | 带额外参数的排序 |
| `ptsname_r()` | `<stdlib.h>` | 线程安全的 PTY 名获取 |
| `__compar_d_fn_t` | `<stdlib.h>` | 带数据的比较函数类型 |

### 3.11 数学函数扩展

| 内容 | 头文件 | 说明 |
|------|--------|------|
| `M_Ef/M_PIf/...` | `<math.h>` | float 精度数学常量 |
| `M_El/M_PIl/...` | `<math.h>` | long double 精度数学常量 |
| `feenableexcept/fedisableexcept()` | `<fenv.h>` | 浮点异常陷阱控制 |
| `fegetexcept()` | `<fenv.h>` | 获取当前异常陷阱状态 |

### 3.12 宽字符扩展

| 函数 | 头文件 | 说明 |
|------|--------|------|
| `wcschrnul()` | `<wchar.h>` | 宽字符版 strchrnul |
| `wmempcpy()` | `<wchar.h>` | 宽字符版 mempcpy |
| `wcstoq/wcstouq()` | `<wchar.h>` | 宽字符→long long 转换 |
| `wcstol_l/wcstod_l()` 系列 | `<wchar.h>` | 带 locale 的宽字符转换 |

### 3.13 其他

| 函数/宏 | 头文件 | 说明 |
|---------|--------|------|
| `assert_perror()` | `<assert.h>` | 基于 errno 的断言 |
| `POLLRDNORM/POLLWRNORM` 等 | `<poll.h>` | 扩展 poll 事件 |
| `ppoll()` | `<poll.h>` | 带信号掩码的 poll |
| `nl_langinfo_l()` | `<langinfo.h>` | 带 locale 的语言信息查询 |
| `NL_LOCALE_NAME()` 等 | `<langinfo.h>` | 扩展 locale 信息常量 |
| `putgrent()` | `<grp.h>` | 写入组数据库条目 |
| `putpwent()` | `<pwd.h>` | 写入密码数据库条目 |

---

## 4. 各头文件详细影响

### 4.1 影响文件统计

```
__USE_GNU 守卫出现的头文件数量:      98 个
__USE_GNU 条件守卫块总数:            227 个

按类别分布:
  fcntl 相关 (bits/fcntl-linux.h):   ~15 个守卫块
  locale/langinfo.h:                  ~25 个守卫块（各种 NL_* 常量）
  stdio.h:                            ~8 个守卫块
  pthread.h:                          ~8 个守卫块
  wchar.h:                            ~8 个守卫块
  unistd.h:                           ~10 个守卫块
  其他:                               ~150+ 个守卫块
```

### 4.2 `__USE_GNU` 独有 vs 通过子宏间接开启

`_GNU_SOURCE` 开启的内容分为两类：

| 类别 | 说明 | 示例 |
|------|------|------|
| **GNU 独有** (`__USE_GNU`) | 只有 `_GNU_SOURCE` 才开启 | `pipe2()`, `mremap()`, `clone()` |
| **间接开启** (子宏) | 通过 `_POSIX_C_SOURCE` 等已可得到 | `pread()`, `strtok_r()`, `sigaction()` |

大多数 POSIX 和 XPG 函数无需 `_GNU_SOURCE`，仅定义相应的标准宏即可。
`__USE_GNU` 守卫的是真正的 GNU/Linux 非标准扩展。

---

## 5. 与其他特性宏的关系

### 5.1 特性宏层次

```
严格 C 标准        ←  __STRICT_ANSI__ (gcc -std=c11)
   │
   ▼
ISO C 扩展         ←  _ISOC99_SOURCE / _ISOC11_SOURCE / _ISOC23_SOURCE
   │
   ▼
POSIX 基础         ←  _POSIX_C_SOURCE=200809L
   │
   ▼
X/Open 扩展        ←  _XOPEN_SOURCE=700
   │
   ▼
默认扩展 (BSD/SysV) ← _DEFAULT_SOURCE
   │
   ▼
GNU 全部扩展        ←  _GNU_SOURCE         ⬅ 包含以上全部 + __USE_GNU
```

### 5.2 常用组合对比

| 宏定义 | 可见性 | 典型用途 |
|--------|--------|---------|
| 无（默认，非 strict） | `__USE_MISC` + 基本 POSIX | 一般应用 |
| `-std=c11` | 仅 ISO C11 | 最大可移植性 |
| `_POSIX_C_SOURCE=200809L` | POSIX.1-2008 | POSIX 合规应用 |
| `_XOPEN_SOURCE=700` | XPG7 全部 | X/Open 合规应用 |
| `_DEFAULT_SOURCE` | POSIX + BSD/SysV 扩展 | 传统 Unix 应用 |
| `_GNU_SOURCE` | **一切** | Linux 专用应用 |

### 5.3 _GNU_SOURCE 与 _DEFAULT_SOURCE 的区别

`_DEFAULT_SOURCE` 开启 `__USE_MISC`（BSD/SysV 函数如 `bzero`、`usleep` 等），
但**不**开启 `__USE_GNU`。因此以下函数仅在 `_GNU_SOURCE` 下可用：

- `pipe2()`, `dup3()`, `accept4()` — 带 flags 的系统调用
- `clone()`, `unshare()`, `setns()` — Linux 命名空间
- `mremap()` — 内存重映射
- `sendmmsg()`/`recvmmsg()` — 批量消息
- 全部 `pthread_*_np()` 非标准扩展

---

## 6. 使用建议与注意事项

### 6.1 何时使用

✅ **推荐使用 `_GNU_SOURCE` 的场景**：
- Linux 专用应用程序
- 需要 `clone()`、`pipe2()`、`mremap()` 等 Linux 特有 API
- 需要 pthread 非标准扩展（线程命名、CPU 亲和性）
- 需要高性能字符串函数（`rawmemchr`、`mempcpy`）
- 系统级工具开发

❌ **不推荐使用的场景**：
- 需要跨 Unix 可移植的代码（应使用 `_POSIX_C_SOURCE`）
- 嵌入式系统（musl libc 仅部分支持 GNU 扩展）
- 库代码（可能与用户的特性宏选择冲突）

### 6.2 正确使用方式

```c
// 方式1：源码顶部（推荐）
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>

// 方式2：编译器命令行
// gcc -D_GNU_SOURCE -o prog prog.c

// 方式3：构建系统
// CMakeLists.txt: add_definitions(-D_GNU_SOURCE)
// Makefile: CFLAGS += -D_GNU_SOURCE
```

### 6.3 注意事项

| 问题 | 说明 |
|------|------|
| **必须在所有头文件之前定义** | 否则行为未定义（headers.h 只处理一次） |
| **影响全局** | 一旦定义，该翻译单元中所有头文件都受影响 |
| **可能遮蔽标准行为** | 部分 GNU 扩展改变了标准函数的声明（如 `strerror_r` 返回值） |
| **非 POSIX 可移植** | 代码将依赖 glibc，无法直接移植到 musl/macOS/FreeBSD |

### 6.4 strerror_r 兼容性陷阱

这是 `_GNU_SOURCE` 最著名的兼容性问题：

```c
// POSIX 版本 (无 _GNU_SOURCE):
int strerror_r(int errnum, char *buf, size_t buflen);
// 返回 0 表示成功

// GNU 版本 (有 _GNU_SOURCE):
char *strerror_r(int errnum, char *buf, size_t buflen);
// 返回字符串指针（可能不在 buf 中！）
```

两个版本的返回类型不同，混用会导致编译错误或运行时 bug。

### 6.5 废弃的 scanf 扩展

`_GNU_SOURCE` + C89 模式下，`%a` 在 scanf 中表示"分配缓冲区"（GNU 扩展）；
C99 模式下 `%a` 表示浮点数格式。POSIX 的替代方案是使用 `%m` 修饰符：

```c
// GNU 扩展 (C89 + _GNU_SOURCE): %as 自动分配
char *buf;
scanf("%as", &buf);

// POSIX 标准 (推荐): %ms 自动分配
char *buf;
scanf("%ms", &buf);
```

---

## 源文件速查表

| 源文件 | 行号 | 内容 |
|--------|------|------|
| `include/features.h` | 55-68 | 用户可定义的特性测试宏列表 |
| `include/features.h` | 90-111 | `__USE_*` 内部宏说明 |
| `include/features.h` | 132-163 | 清除所有 `__USE_*` 宏 |
| `include/features.h` | 217-245 | `_GNU_SOURCE` → 开启所有子特性 |
| `include/features.h` | 445-446 | `_GNU_SOURCE` → `__USE_GNU=1` |
| `include/features.h` | 496-500 | GNU 废弃 scanf 扩展条件 |
| `bits/libc-header-start.h` | 38-106 | ISO C 特性宏的每文件处理 |
