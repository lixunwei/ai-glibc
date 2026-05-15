# malloc/free 完整调用链分析

> 基于 clangd LSP 语义分析 (glibc 2.43.9000)

## 概述

glibc 2.43.9000 的内存分配器（ptmalloc2）从用户 `malloc()`/`free()` 入口到操作系统交互的完整调用路径。本文档基于 clangd 精确的 outgoing/incoming call hierarchy 生成。

---

## 1. malloc 调用链

### 1.1 顶层入口: `__libc_malloc()` [malloc/malloc.c:3261]

用户调用 `malloc(size)` 实际执行的是 `__libc_malloc()`。

```
__libc_malloc(size_t bytes)
├── checked_request2size()          [malloc.c:1258] — 大小对齐+溢出检查
├── tcache_get()                    [malloc.c:3036] — tcache 快速路径
├── tcache_get_large()              [malloc.c:3071] — 大块 tcache
├── large_csize2tidx()              [malloc.c:2973] — 计算 tcache 索引
├── tag_new_usable()                [malloc.c:1393] — MTE 标签设置
└── __libc_malloc2()                [malloc.c:3225] — 慢路径(需要锁)
```

### 1.2 慢路径: `__libc_malloc2()` [malloc/malloc.c:3225]

当 tcache 无法满足时，进入慢路径获取 arena 锁。

```
__libc_malloc2(size_t bytes)
├── __lll_lock_wait[_private]()     [lowlevellock.c:25/40] — futex 锁等待
├── __lll_lock_wake[_private]()     [lowlevellock.c:55/62] — futex 锁唤醒
├── arena_get2()                    [arena.c:802] — 获取/创建 arena
├── arena_get_retry()               [arena.c:856] — arena 获取重试
├── _int_malloc()                   [malloc.c:3757] — ★核心分配逻辑
├── arena_for_chunk()               [arena.c:149] — 查找 chunk 所属 arena
├── tag_new_usable()                [malloc.c:1393] — MTE 标签
└── tag_at()                        [malloc.c:437] — MTE 地址标签
```

### 1.3 核心分配: `_int_malloc()` [malloc/malloc.c:3757]

分配器的核心逻辑，按以下优先级搜索可用内存：

```
_int_malloc(mstate av, size_t bytes)
├── checked_request2size()          [malloc.c:1258] — 请求大小→chunk大小
├── tcache_inactive()               [malloc.c:2904] — tcache 是否已禁用
├── tcache_init()                   [malloc.c:3185] — 延迟初始化 tcache
├── tcache_put()                    [malloc.c:3029] — 回填 tcache
├── unlink_chunk()                  [malloc.c:1583] — 从 bin 链表摘除
├── alloc_perturb()                 [malloc.c:1873] — 调试填充
├── malloc_printerr()               [malloc.c:5256] — 一致性检查失败
└── sysmalloc()                     [malloc.c:2290] — ★向OS申请内存
```

**搜索顺序：**
1. fastbin → 精确匹配
2. smallbin → 精确匹配
3. unsorted bin → 遍历拆分
4. large bin → best-fit
5. 更大的 bin → 找到即切割
6. top chunk → 切割
7. sysmalloc → 向 OS 申请

### 1.4 系统内存申请: `sysmalloc()` [malloc/malloc.c:2290]

当所有 bin 和 top chunk 都无法满足时，向操作系统申请新内存。

```
sysmalloc(INTERNAL_SIZE_T nb, mstate av)
├── sysmalloc_mmap()                [malloc.c:2230] — 大块直接 mmap
│   ├── __mmap()                    [mmap64.c:66] — ★系统调用
│   ├── __set_vma_name()            [setvmaname.c:59] — prctl 设置 VMA 名
│   ├── madvise_thp()               [malloc.c:1893] — 透明大页建议
│   └── mmap_set_chunk()            [malloc.c:1436] — 设置 mmap chunk 元数据
├── sysmalloc_mmap_fallback()       [malloc.c:2267] — sbrk 失败时 mmap 兜底
├── heap_for_ptr()                  [arena.c:142] — 查找 heap 结构
├── grow_heap()                     [arena.c:459] — 扩展现有 heap
├── new_heap()                      [arena.c:440] — 分配全新 heap
│   ├── alloc_new_heap()            [arena.c:357] — 底层 mmap
│   │   ├── heap_max_size()         [arena.c:54]
│   │   └── heap_min_size()         [arena.c:47]
│   └── heap_max_size()             [arena.c:54]
├── __glibc_morecore()              [morecore.c:24] — sbrk 封装
├── madvise_thp()                   [malloc.c:1893] — THP madvise
├── __set_vma_name()                [setvmaname.c:59] — prctl 命名
├── _int_free_chunk()               [malloc.c:4260] — 释放旧 top
└── malloc_printerr()               [malloc.c:5256] — 错误处理
```

**内存获取策略：**
- 主 arena: `sbrk()`（通过 `__glibc_morecore`）
- 非主 arena: `mmap()` 创建 heap（通过 `new_heap`/`alloc_new_heap`）
- 大请求（≥ mmap_threshold）: 直接 `mmap()`

---

## 2. free 调用链

### 2.1 顶层入口: `__libc_free()` [malloc/malloc.c:3297]

```
__libc_free(void *mem)
├── tag_at()                        [malloc.c:437] — MTE 标签验证
├── tag_region()                    [malloc.c:417] — MTE 清除标签
├── tcache_inactive()               [malloc.c:2904] — tcache 是否有效
├── tcache_double_free_verify()     [malloc.c:3122] — 双重释放检测
├── tcache_put()                    [malloc.c:3029] — tcache 快速路径
├── tcache_put_large()              [malloc.c:3060] — 大块 tcache
├── tcache_free_init()              [malloc.c:3290] — tcache 延迟初始化
├── large_csize2tidx()              [malloc.c:2973] — 计算索引
├── arena_for_chunk()               [arena.c:149] — 查找所属 arena
├── _int_free_chunk()               [malloc.c:4260] — ★核心释放逻辑
└── malloc_printerr_tail()          [malloc.c:5271] — 错误处理
```

### 2.2 核心释放: `_int_free_chunk()` [malloc/malloc.c:4260]

三阶段释放架构（glibc 2.43 新设计）:

```
_int_free_chunk(mstate av, mchunkptr p, INTERNAL_SIZE_T size, int have_lock)
├── __lll_lock_wait[_private]()     [lowlevellock.c:25/40] — 获取 arena 锁
├── __lll_lock_wake[_private]()     [lowlevellock.c:55/62] — 释放 arena 锁
├── _int_free_merge_chunk()         [malloc.c:4316] — 阶段1: 合并相邻块
│   ├── free_perturb()              [malloc.c:1880] — 调试填充
│   ├── malloc_printerr()           [malloc.c:5256] — 一致性检查
│   ├── unlink_chunk()              [malloc.c:1583] — 摘除相邻空闲块
│   ├── _int_free_create_chunk()    [malloc.c:4365] — 阶段2: 创建空闲块
│   │   ├── unlink_chunk()          [malloc.c:1583] — 摘除旧块
│   │   └── malloc_printerr()       [malloc.c:5256]
│   └── _int_free_maybe_trim()      [malloc.c:4437] — 阶段3: 收缩堆
│       ├── systrim()               [malloc.c:2721] — 主 arena 收缩
│       │   └── __glibc_morecore()  [morecore.c:24] — 负值 sbrk
│       ├── heap_trim()             [arena.c:520] — 非主 arena 收缩
│       └── heap_for_ptr()          [arena.c:142]
└── munmap_chunk()                  [malloc.c:2785] — mmap 块直接释放
```

### 2.3 `_int_free_chunk` 的调用者

```
_int_free_chunk() ← 被以下函数调用:
├── __libc_free()                   [malloc.c:3297] — 用户 free()
├── __libc_realloc()                [malloc.c:3358] — realloc 缩小
├── _int_realloc()                  [malloc.c:4467] — 内部 realloc
├── free_check()                    [malloc-check.c:211] — 调试模式
├── sysmalloc()                     [malloc.c:2290] — 释放旧 top
└── tcache_thread_shutdown()        [malloc.c:3149] — 线程退出清理
```

---

## 3. 完整数据流

### 3.1 malloc 快速路径（无锁）

```
用户调用 malloc(64)
  → __libc_malloc()
    → checked_request2size(64) → nb = 80 (对齐后)
    → tcache_get(tcache, tc_idx=4) → 命中！
    → tag_new_usable(chunk) → 返回用户指针
  ← 返回 (耗时 ~10ns)
```

### 3.2 malloc 慢路径（需要锁 + 系统调用）

```
用户调用 malloc(256000)
  → __libc_malloc()
    → tcache: 无匹配 (size > tcache_max_bytes)
    → __libc_malloc2()
      → arena_get2() → 获取 arena 锁
      → _int_malloc(av, 256000)
        → fastbin: 无
        → smallbin: 无
        → unsorted bin: 无合适
        → large bin: 无合适
        → top chunk: 不够大
        → sysmalloc(nb=256016, av)
          → nb > mmap_threshold
          → sysmalloc_mmap(nb, pagesize)
            → __mmap(0, 260096, PROT_READ|PROT_WRITE, ...)
            → 返回 mmap 块
      → 释放 arena 锁
  ← 返回 (耗时 ~1μs)
```

### 3.3 free 路径

```
用户调用 free(ptr)
  → __libc_free()
    → chunk = mem2chunk(ptr)
    → size = chunksize(chunk)
    → tcache_double_free_verify() → OK
    → tcache_put(chunk, tc_idx) → tcache 未满时直接放入
  ← 返回 (耗时 ~5ns)

--- 或者 tcache 满时 ---

  → __libc_free()
    → arena_for_chunk(chunk) → av
    → _int_free_chunk(av, chunk, size, 0)
      → 获取 arena 锁
      → _int_free_merge_chunk(av, chunk, size)
        → 前向合并 (prev_inuse check)
        → 后向合并 (next chunk check)
        → _int_free_create_chunk(av, merged, total_size)
          → 放入 unsorted bin 或成为新 top
        → _int_free_maybe_trim(av, total_size)
          → if (size > FASTBIN_CONSOLIDATION_THRESHOLD)
            → systrim() 或 heap_trim()
      → 释放 arena 锁
  ← 返回
```

---

## 4. 关键设计要点

### 4.1 三阶段 free 架构 (glibc 2.43 新设计)

旧版 glibc 的 `_int_free` 是一个巨大函数，2.43 拆分为：

| 阶段 | 函数 | 职责 |
|------|------|------|
| 入口 | `_int_free_chunk` | 锁管理 + mmap 块快速释放 |
| 阶段1 | `_int_free_merge_chunk` | 合并相邻空闲块 |
| 阶段2 | `_int_free_create_chunk` | 将合并结果放入 bin/top |
| 阶段3 | `_int_free_maybe_trim` | 堆收缩归还 OS |

### 4.2 tcache 层

tcache (Thread-Local Cache) 是无锁快速路径：
- 每线程独立缓存
- 每个 size class 最多 7 个 chunk
- `tcache_get()` / `tcache_put()`: 无锁 LIFO 链表操作
- `tcache_double_free_verify()`: 扫描所有 bin 防止双重释放

### 4.3 Arena 锁策略

```
__libc_malloc2():
  锁获取: __lll_lock_wait_private() (futex)
  失败重试: arena_get2() → arena_get_retry()
  锁释放: __lll_lock_wake_private()
```

### 4.4 内存来源

| 场景 | 函数 | 系统调用 |
|------|------|----------|
| 主 arena 扩展 | `__glibc_morecore()` | `sbrk()` |
| 非主 arena 新 heap | `alloc_new_heap()` | `mmap()` |
| 大块分配 | `sysmalloc_mmap()` | `mmap()` |
| 主 arena 收缩 | `systrim()` | `sbrk(负值)` |
| 非主 arena 收缩 | `heap_trim()` | `munmap()` |
| mmap 块释放 | `munmap_chunk()` | `munmap()` |

---

## 5. 源码位置索引

| 函数 | 文件 | 行号 |
|------|------|------|
| `__libc_malloc` | malloc/malloc.c | 3261 |
| `__libc_malloc2` | malloc/malloc.c | 3225 |
| `__libc_free` | malloc/malloc.c | 3297 |
| `__libc_calloc` | malloc/malloc.c | 3706 |
| `__libc_calloc2` | malloc/malloc.c | 3612 |
| `__libc_realloc` | malloc/malloc.c | 3358 |
| `_int_malloc` | malloc/malloc.c | 3757 |
| `_int_free_chunk` | malloc/malloc.c | 4260 |
| `_int_free_merge_chunk` | malloc/malloc.c | 4316 |
| `_int_free_create_chunk` | malloc/malloc.c | 4365 |
| `_int_free_maybe_trim` | malloc/malloc.c | 4437 |
| `sysmalloc` | malloc/malloc.c | 2290 |
| `sysmalloc_mmap` | malloc/malloc.c | 2230 |
| `systrim` | malloc/malloc.c | 2721 |
| `munmap_chunk` | malloc/malloc.c | 2785 |
| `tcache_get` | malloc/malloc.c | 3036 |
| `tcache_put` | malloc/malloc.c | 3029 |
| `tcache_init` | malloc/malloc.c | 3185 |
| `arena_get2` | malloc/arena.c | 802 |
| `new_heap` | malloc/arena.c | 440 |
| `alloc_new_heap` | malloc/arena.c | 357 |
| `grow_heap` | malloc/arena.c | 459 |
| `heap_trim` | malloc/arena.c | 520 |
