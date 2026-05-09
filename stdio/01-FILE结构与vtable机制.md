# FILE 结构与 vtable 机制

## 1. struct _IO_FILE 定义

`struct _IO_FILE` 是 C 标准库 `FILE` 类型的底层实现，定义于
`libio/bits/types/struct_FILE.h:51-86`：

```c
struct _IO_FILE
{
  int _flags;            // 高 16 位 = _IO_MAGIC (0xFBAD)，低位 = 状态标志

  // ---- 读缓冲区指针 ----
  char *_IO_read_ptr;    // 当前读位置（下次 getc 读的位置）
  char *_IO_read_end;    // 读区域结束（已读入数据的末尾）
  char *_IO_read_base;   // 读区域起始（含 putback 区域）

  // ---- 写缓冲区指针 ----
  char *_IO_write_base;  // 写区域起始（待刷新数据的开头）
  char *_IO_write_ptr;   // 当前写位置（下次 putc 写的位置）
  char *_IO_write_end;   // 写区域结束

  // ---- 底层缓冲区 ----
  char *_IO_buf_base;    // 缓冲区起始地址
  char *_IO_buf_end;     // 缓冲区结束地址

  // ---- 备份/回退区域 ----
  char *_IO_save_base;   // 非当前 get 区域的起始
  char *_IO_backup_base; // 备份区域的第一个有效字符
  char *_IO_save_end;    // 非当前 get 区域的结束

  struct _IO_marker *_markers;  // 标记链表

  struct _IO_FILE *_chain;      // 全局流链表（_IO_list_all）

  int _fileno;                  // 底层文件描述符
  int _flags2:24;               // 扩展标志
  char _short_backupbuf[1];     // malloc 失败时的备用回退缓冲

  __off_t _old_offset;          // 兼容偏移量

  unsigned short _cur_column;   // 当前列号（用于 TAB 处理）
  signed char _vtable_offset;   // vtable 偏移（旧 ABI 兼容）
  char _shortbuf[1];            // 无缓冲模式下的 1 字节缓冲区

  _IO_lock_t *_lock;            // 线程安全锁指针
};
```

### 1.1 缓冲区指针关系图

```
缓冲区（_IO_buf_base ~ _IO_buf_end）
┌──────────────────────────────────────────────────────┐
│                                                      │
│  读模式 (GET mode):                                   │
│  _IO_read_base        _IO_read_ptr    _IO_read_end   │
│       ▼                    ▼               ▼         │
│  ┌────┬────────────────────┬───────────────┬─────┐   │
│  │已读│    可读数据         │  未填充       │     │   │
│  └────┴────────────────────┴───────────────┴─────┘   │
│                                                      │
│  写模式 (PUT mode):                                   │
│  _IO_write_base      _IO_write_ptr   _IO_write_end   │
│       ▼                   ▼               ▼          │
│  ┌────────────────────────┬───────────────┬──────┐   │
│  │   待刷新数据            │  可用空间     │      │   │
│  └────────────────────────┴───────────────┴──────┘   │
│                                                      │
│  _IO_buf_base                           _IO_buf_end  │
│       ▼                                      ▼       │
│  ┌───────────────────────────────────────────────┐   │
│  │          底层缓冲区存储                        │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

**关键不变量（`fileops.c:49-99` 中有详细文档）：**
- 流同时只处于 PUT 模式、GET 模式或 PUTBACK 模式之一
- PUT 模式时：`_IO_read_ptr == _IO_read_end == _IO_read_base`
- GET 模式时：未刷新的写数据 = `_IO_write_base` 到 `_IO_write_ptr`
- 行缓冲时：`_IO_write_end == _IO_write_ptr`（强制每次 putc 调用 overflow）
- 无缓冲时：使用 `_shortbuf[1]` 作为 1 字节缓冲区

### 1.2 常用标志位

| 标志 | 含义 |
|------|------|
| `_IO_NO_READS` | 不允许读操作 |
| `_IO_NO_WRITES` | 不允许写操作 |
| `_IO_EOF_SEEN` | 已到达文件末尾 |
| `_IO_ERR_SEEN` | 发生 I/O 错误 |
| `_IO_UNBUFFERED` | 无缓冲模式 |
| `_IO_LINE_BUF` | 行缓冲模式 |
| `_IO_CURRENTLY_PUTTING` | 当前处于写模式 |
| `_IO_IS_FILEBUF` | 是文件缓冲区（非字符串流等） |
| `_IO_LINKED` | 已加入 `_IO_list_all` 链表 |
| `_IO_IN_BACKUP` | 当前在备份（putback）区域 |
| `_IO_USER_LOCK` | 用户管理锁（禁用自动锁） |

---

## 2. _IO_FILE_plus 与 vtable

### 2.1 _IO_FILE_plus 结构

`_IO_FILE_plus` 在 `FILE` 基础上附加了 vtable 指针
（`libio/libioP.h:326-330`）：

```c
struct _IO_FILE_plus
{
  FILE file;                          // 内嵌 FILE 结构
  const struct _IO_jump_t *vtable;    // 操作跳转表
};
```

所有通过 `fopen` 创建的流实际上都是 `_IO_FILE_plus`，
`vtable` 指向具体的操作实现（通常是 `_IO_file_jumps`）。

### 2.2 vtable 跳转表：struct _IO_jump_t

跳转表定义于 `libio/libioP.h:295-319`，包含 18 个函数指针：

```c
struct _IO_jump_t
{
    size_t __dummy;           // 保留
    size_t __dummy2;          // 保留
    _IO_finish_t    __finish;      // 流销毁（释放资源）
    _IO_overflow_t  __overflow;    // 缓冲区满时的处理
    _IO_underflow_t __underflow;   // 缓冲区空时填充（不消费字符）
    _IO_underflow_t __uflow;       // 缓冲区空时填充（消费字符）
    _IO_pbackfail_t __pbackfail;   // putback 失败处理
    _IO_xsputn_t    __xsputn;     // 批量写入（fwrite 核心）
    _IO_xsgetn_t    __xsgetn;     // 批量读取（fread 核心）
    _IO_seekoff_t   __seekoff;    // 偏移定位（fseek）
    _IO_seekpos_t   __seekpos;    // 绝对定位
    _IO_setbuf_t    __setbuf;     // 设置缓冲区（setvbuf）
    _IO_sync_t      __sync;       // 同步（fflush）
    _IO_doallocate_t __doallocate; // 分配缓冲区
    _IO_read_t      __read;       // 底层读（→ read 系统调用）
    _IO_write_t     __write;      // 底层写（→ write 系统调用）
    _IO_seek_t      __seek;       // 底层定位（→ lseek 系统调用）
    _IO_close_t     __close;      // 底层关闭（→ close 系统调用）
    _IO_stat_t      __stat;       // 底层 stat
    _IO_showmanyc_t __showmanyc;  // C++ 兼容
    _IO_imbue_t     __imbue;      // C++ 兼容
};
```

### 2.3 vtable 调用宏

通过宏间接调用 vtable 中的函数（`libio/libioP.h:118-128`）：

```c
// 无参调用
#define JUMP0(FUNC, THIS)       (_IO_JUMPS_FUNC(THIS)->FUNC) (THIS)
// 单参调用
#define JUMP1(FUNC, THIS, X1)   (_IO_JUMPS_FUNC(THIS)->FUNC) (THIS, X1)
// 双参调用
#define JUMP2(FUNC, THIS, X1, X2) ...
// 三参调用
#define JUMP3(FUNC, THIS, X1, X2, X3) ...
```

常用的高级宏（内部使用）：

| 宏 | 展开为 | 用途 |
|----|--------|------|
| `_IO_OVERFLOW(fp, ch)` | `JUMP1(__overflow, fp, ch)` | 写缓冲区满 |
| `_IO_UNDERFLOW(fp)` | `JUMP0(__underflow, fp)` | 读缓冲区空 |
| `_IO_SYNC(fp)` | `JUMP0(__sync, fp)` | fflush |
| `_IO_SEEKOFF(fp,off,dir,mode)` | `JUMP3(__seekoff,...)` | fseek |
| `_IO_SYSREAD(fp,buf,n)` | `JUMP2(__read,fp,buf,n)` | 系统 read |
| `_IO_SYSWRITE(fp,buf,n)` | `JUMP2(__write,fp,buf,n)` | 系统 write |
| `_IO_SYSCLOSE(fp)` | `JUMP0(__close, fp)` | 系统 close |

### 2.4 文件流的默认 vtable

文件流使用 `_IO_file_jumps` 跳转表，其中各操作映射到 `fileops.c` 中的函数：

| vtable 槽位 | 函数 | 源文件位置 |
|-------------|------|-----------|
| `__overflow` | `_IO_new_file_overflow` | `fileops.c:781-838` |
| `__underflow` | `_IO_new_file_underflow` | `fileops.c:511-588` |
| `__xsputn` | `_IO_new_file_xsputn` | `fileops.c:1274-1345` |
| `__xsgetn` | `_IO_file_xsgetn` | `fileops.c:1348-1439` |
| `__seekoff` | `_IO_new_file_seekoff` | `fileops.c:968-1125` |
| `__sync` | `_IO_new_file_sync` | `fileops.c:841-871` |
| `__doallocate` | `_IO_file_doallocate` | `filedoalloc.c:67-96` |
| `__close` | `_IO_file_close` | `fileops.c:1240-1245` |
| `__read` | `_IO_file_read` | `fileops.c:1206-1211` |
| `__write` | `_IO_new_file_write` | `fileops.c:1249-1271` |
| `__seek` | `_IO_file_seek` | `fileops.c:1215-1218` |
| `__finish` | `_IO_new_file_finish` | `fileops.c:212-221` |
| `__setbuf` | `_IO_new_file_setbuf` | `fileops.c:431-441` |

---

## 3. vtable 安全验证

### 3.1 验证机制

glibc 对 vtable 进行安全检查，防止攻击者通过伪造 vtable 劫持控制流。
验证逻辑在 `libio/libioP.h:108-122`：

```c
#define _IO_JUMPS_FUNC(THIS)                                    \
  (IO_validate_vtable                                           \
   (*(struct _IO_jump_t **) ((void *) &_IO_JUMPS_FILE_plus(THIS) \
                             + (THIS)->_vtable_offset)))
```

`IO_validate_vtable()` 检查 vtable 指针是否落在合法的 `__io_vtables`
区域内。如果不在范围内，则调用 `_IO_vtable_check()`。

### 3.2 `_IO_vtable_check` 实现

```c
// libio/vtables.c:503-535
void _IO_vtable_check (void)
{
#ifdef SHARED
  // 检查是否已接受外部 vtable（C++ 兼容）
  void (*flag) (void) = atomic_load_relaxed (&IO_accept_foreign_vtables);
  PTR_DEMANGLE (flag);
  if (flag == &_IO_vtable_check)
    return;

  // 非默认命名空间的库始终接受外部 vtable
  Dl_info di;
  struct link_map *l;
  if (!rtld_active ()
      || (_dl_addr (_IO_vtable_check, &di, &l, NULL) != 0
          && l->l_ns != LM_ID_BASE))
    return;
#else
  // 静态 dlopen 场景：禁用检查
  if (__dlopen != NULL)
    return;
#endif
  // 不合法的 vtable → 进程终止
  __libc_fatal ("Fatal error: glibc detected an invalid stdio handle\n");
}
```

### 3.3 启动时 vtable 检测

glibc 启动时检查标准流的 vtable 是否被 C++ 库替换
（`libio/vtables.c:542-550`）：

```c
// 构造函数，在 main() 之前运行
static void check_stdfiles_vtables (void)
{
  if (_IO_2_1_stdin_.vtable != &_IO_file_jumps
      || _IO_2_1_stdout_.vtable != &_IO_file_jumps
      || _IO_2_1_stderr_.vtable != &_IO_file_jumps)
    IO_set_accept_foreign_vtables (&_IO_vtable_check);
}
```

---

## 4. 标准流 stdin/stdout/stderr

### 4.1 定义

标准流以静态 `_IO_FILE_plus` 对象形式定义（`libio/stdfiles.c:36-56`）：

```c
// 宏展开定义，包含锁、宽字符数据、vtable
#define DEF_STDFILE(NAME, FD, CHAIN, FLAGS) \
  static _IO_lock_t _IO_stdfile_##FD##_lock = _IO_lock_initializer; \
  static struct _IO_wide_data _IO_wide_data_##FD \
    = { ._wide_vtable = &_IO_wfile_jumps }; \
  struct _IO_FILE_plus NAME \
    = {FILEBUF_LITERAL(CHAIN, FLAGS, FD, &_IO_wide_data_##FD), \
       &_IO_file_jumps};

DEF_STDFILE(_IO_2_1_stdin_,  0, 0,                   _IO_NO_WRITES);
DEF_STDFILE(_IO_2_1_stdout_, 1, &_IO_2_1_stdin_,     _IO_NO_READS);
DEF_STDFILE(_IO_2_1_stderr_, 2, &_IO_2_1_stdout_,
            _IO_NO_READS + _IO_UNBUFFERED);
```

### 4.2 默认缓冲模式

| 流 | fd | 标志 | 缓冲模式 |
|----|-----|------|----------|
| stdin | 0 | `_IO_NO_WRITES` | 全缓冲（tty 时行缓冲） |
| stdout | 1 | `_IO_NO_READS` | 全缓冲（tty 时行缓冲） |
| stderr | 2 | `_IO_NO_READS + _IO_UNBUFFERED` | 无缓冲 |

**注意：** 当 stdout 连接到终端时，`_IO_file_doallocate()` 会检测到 `isatty(fd)`
并自动设置 `_IO_LINE_BUF` 标志，使其变为行缓冲。

### 4.3 _IO_list_all 链表

所有打开的流组成全局单向链表（`libio/stdfiles.c:56`）：

```c
struct _IO_FILE_plus *_IO_list_all = &_IO_2_1_stderr_;
```

链表通过 `_chain` 字段连接：

```
_IO_list_all → stderr → stdout → stdin → fopen'd_file1 → fopen'd_file2 → NULL
```

`_IO_stdfiles_init()` 构造函数（`stdfiles.c:61-70`）在启动时完成
`_prevchain` 双向链接的初始化。

**链表操作（`libio/genops.c:65-134`）：**
- `_IO_link_in(fp)`：将新流插入链表头部
- `_IO_un_link(fp)`：从链表中移除流

这两个操作在多线程环境下通过 `list_all_lock` 全局锁保护。

---

## 5. 源码位置速查表

| 功能 | 源文件 | 行号 |
|------|--------|------|
| struct _IO_FILE 定义 | `libio/bits/types/struct_FILE.h` | 51-86 |
| _IO_FILE_plus 定义 | `libio/libioP.h` | 326-330 |
| struct _IO_jump_t 定义 | `libio/libioP.h` | 295-319 |
| vtable 调用宏 (JUMP0~3) | `libio/libioP.h` | 125-128 |
| _IO_JUMPS_FUNC / IO_validate_vtable | `libio/libioP.h` | 108-122 |
| _IO_vtable_check 实现 | `libio/vtables.c` | 503-535 |
| check_stdfiles_vtables | `libio/vtables.c` | 542-550 |
| DEF_STDFILE 宏 | `libio/stdfiles.c` | 36-50 |
| stdin/stdout/stderr 定义 | `libio/stdfiles.c` | 52-54 |
| _IO_list_all 定义 | `libio/stdfiles.c` | 56 |
| _IO_stdfiles_init 构造函数 | `libio/stdfiles.c` | 61-70 |
| _IO_un_link | `libio/genops.c` | 65-104 |
| _IO_link_in | `libio/genops.c` | 107-134 |
| 缓冲区模式文档注释 | `libio/fileops.c` | 49-99 |
| _IO_new_file_init_internal | `libio/fileops.c` | 105-116 |
