# stdio/IO 子系统深度分析

> 基于 glibc 2.43.9000 libio 源码分析

---

## 文档索引

| 文档 | 内容 | 关键主题 |
|------|------|----------|
| [01-FILE结构与vtable机制.md](01-FILE结构与vtable机制.md) | 核心数据结构 | struct _IO_FILE、缓冲区指针、_IO_FILE_plus、vtable 跳转表、vtable 安全检查、标准流初始化 |
| [02-流生命周期与缓冲管理.md](02-流生命周期与缓冲管理.md) | 流的创建到销毁 | fopen 分配流程、缓冲模式（全缓冲/行缓冲/无缓冲）、setvbuf、fclose 清理、_IO_list_all 链表管理 |
| [03-读写路径与刷新机制.md](03-读写路径与刷新机制.md) | I/O 数据流 | fread/fwrite 数据路径、缓冲区填充与刷新、overflow/underflow、fflush 全局刷新、fseek 与缓冲交互、printf 格式化输出、线程安全锁 |

---

## libio 架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                     POSIX / ISO C API                            │
│  fopen  fclose  fread  fwrite  fprintf  fseek  ftell  fflush    │
├──────────────────────────────────────────────────────────────────┤
│                  _IO_FILE_plus + vtable                          │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐  │
│  │  struct _IO_FILE         │  │  struct _IO_jump_t (vtable) │  │
│  │  ├ _flags (模式标志)      │  │  ├ __overflow  (缓冲区满)    │  │
│  │  ├ _IO_read_ptr/end/base │  │  ├ __underflow (缓冲区空)    │  │
│  │  ├ _IO_write_ptr/end/base│  │  ├ __xsputn    (批量写)      │  │
│  │  ├ _IO_buf_base/end      │  │  ├ __xsgetn    (批量读)      │  │
│  │  ├ _chain (链表)          │  │  ├ __seekoff   (定位)        │  │
│  │  ├ _fileno (fd)           │  │  ├ __sync      (同步)        │  │
│  │  └ _lock (线程锁)        │  │  ├ __close     (关闭)        │  │
│  └──────────────────────────┘  │  └ __read/__write/__seek     │  │
│                                └─────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│              _IO_list_all 全局流链表                              │
│  stderr → stdout → stdin → fopen'd files → ...                  │
├──────────────────────────────────────────────────────────────────┤
│              系统调用层                                           │
│  read(fd) / write(fd) / lseek(fd) / close(fd) / mmap            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 关键源文件

| 文件 | 内容 |
|------|------|
| libio/bits/types/struct_FILE.h | struct _IO_FILE 定义 |
| libio/libioP.h | _IO_FILE_plus、_IO_jump_t、vtable 宏 |
| libio/fileops.c | 文件流操作（open/close/read/write/seek/sync） |
| libio/genops.c | 通用操作（链表管理、默认缓冲分配、flush_all） |
| libio/iofopen.c | fopen 实现 |
| libio/iofclose.c | fclose 实现 |
| libio/iofread.c | fread 实现 |
| libio/iofwrite.c | fwrite 实现 |
| libio/iofflush.c | fflush 实现 |
| libio/iosetvbuf.c | setvbuf 实现 |
| libio/stdfiles.c | stdin/stdout/stderr 定义 |
| libio/vtables.c | vtable 安全验证 |
| sysdeps/nptl/stdio-lock.h | FILE 递归锁实现 |
| stdio-common/vfprintf-internal.c | printf 格式化核心 |
