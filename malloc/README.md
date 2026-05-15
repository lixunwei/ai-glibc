# malloc 内存分配器深度分析

> 基于 glibc 2.43.9000 ptmalloc2 源码分析

---

## 文档索引

| 文档 | 内容 | 关键主题 |
|------|------|----------|
| [01-数据结构与架构.md](01-数据结构与架构.md) | 核心架构 | malloc_chunk、malloc_state、heap_info、bin 体系 |
| [02-malloc分配路径.md](02-malloc分配路径.md) | 分配算法 | tcache→smallbin→unsorted→largebin→top→sysmalloc |
| [03-free释放路径.md](03-free释放路径.md) | 释放算法 | tcache插入、合并、smallbin/unsorted bin、内存归还 |
| [04-Arena与Tcache.md](04-Arena与Tcache.md) | 多线程扩展 | Arena 创建/绑定、tcache 无锁快速路径 |
| [05-调优与调试.md](05-调优与调试.md) | 配置与诊断 | mallopt、tunables、realloc/calloc/memalign |
| [06-malloc_free调用链.md](06-malloc_free调用链.md) | malloc/free 完整调用链 | clangd 精确分析：tcache快速路径→arena锁→_int_malloc→sysmalloc→mmap/sbrk；三阶段free架构（2.43新设计）；完整数据流时序 |

---

## ptmalloc2 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                      应用程序                                     │
│              malloc / free / realloc / calloc                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Tcache（Per-Thread Cache）                   │    │
│  │  76 个 bin，每 bin 最多 16 个 chunk，无锁 LIFO          │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │ miss                                 │
│  ┌────────────────────────▼────────────────────────────────┐    │
│  │              Arena（malloc_state）                        │    │
│  │  ┌──────────┐ ┌────────────┐ ┌───────────┐ ┌────────┐  │    │
│  │  │ SmallBins│ │Unsorted Bin│ │ LargeBins │ │  Top   │  │    │
│  │  │  (64个)  │ │  (bin[1])  │ │  (63个)   │ │ Chunk  │  │    │
│  │  └──────────┘ └────────────┘ └───────────┘ └────────┘  │    │
│  │              arena->mutex 保护                            │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │ top 不够                             │
│  ┌────────────────────────▼────────────────────────────────┐    │
│  │              sysmalloc                                    │    │
│  │  main_arena: sbrk()    secondary: mmap(HEAP_MAX_SIZE)   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              大块直接 mmap                                │    │
│  │  size >= mmap_threshold → mmap() → 独立映射              │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│              Linux Kernel                                         │
│  brk / mmap / munmap / mremap / madvise(MADV_DONTNEED)          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 设计原则

| 原则 | 实现方式 |
|------|----------|
| 多线程可扩展 | Per-thread tcache（无锁）+ 多 arena（减少锁竞争） |
| 空间效率 | 使用中的 chunk 仅需 2×SIZE_SZ 头部 |
| 时间局部性 | tcache LIFO；小chunk释放后直接入smallbin，大chunk进unsorted bin |
| 碎片控制 | 前后合并；best-fit large bin；top chunk 回收 |
| 安全防护 | tcache_key 防双重释放；size/pointer 校验 |

---

## 源文件布局

| 文件 | 行数 | 职责 |
|------|------|------|
| `malloc/malloc.c` | ~5500 | 核心：chunk/bin/arena/tcache、malloc/free/realloc |
| `malloc/arena.c` | ~900 | Arena 创建/销毁、heap 管理、线程绑定 |
| `malloc/malloc.h` | ~160 | 公开 API、mallopt 常量 |
| `malloc/malloc-internal.h` | ~50 | 内部接口声明 |
| `malloc/malloc-debug.c` | ~200 | MALLOC_CHECK_ 调试模式 |
| `malloc/malloc-check.c` | ~240 | 调试检查实现 |
| `malloc/hooks.c` | — | 兼容性 hook（已废弃） |

---

## 关键常量

| 常量 | 值 | 定义位置 | 说明 |
|------|-----|---------|------|
| `MALLOC_ALIGNMENT` | 16（64位） | `malloc.c` | 最小对齐 |
| `MIN_CHUNK_SIZE` | 32（64位） | `malloc.c:1231` | 最小 chunk 大小 |
| `NBINS` | 128 | `malloc.c:1533` | bin 总数 |
| `NSMALLBINS` | 64 | `malloc.c:1534` | smallbin 数量 |
| `TCACHE_MAX_BINS` | 76 | `malloc.c:2888` | tcache bin 数量 |
| `TCACHE_FILL_COUNT` | 16 | `malloc.c` | 默认每 bin 容量 |
| `HEAP_MAX_SIZE` | 64MB（64位） | `arena.c` | 单个 heap 最大尺寸 |
