# realloc / calloc / memalign 内存重分配调用链

> 基于 glibc 2.43.9000，使用 clangd LSP 进行精确调用层次分析  
> 分析工具：clangd 22.1.1 outgoing call hierarchy

---

## 概述

本文档分析 glibc ptmalloc2 中除 `malloc`/`free` 以外的三大分配器入口：

| API | 功能 | 核心复杂度 |
|-----|------|-----------|
| `realloc(ptr, size)` | 调整已分配块大小 | mmap→mremap 快速路径、in-place 扩展、arena 锁重试 |
| `calloc(n, size)` | 分配并清零 | 溢出检测、tcache 快速路径、sbrk 零页优化 |
| `memalign(align, size)` | 对齐分配 | 过量分配+切割、mmap 对齐、tcache 对齐搜索 |
| `reallocarray(ptr, n, size)` | 安全重分配（带溢出检测） | 薄封装 realloc |

---

## 1. realloc 调用链

### 1.1 入口：__libc_realloc

**位置**：`malloc/malloc.c:3358`

```
__libc_realloc(oldmem, bytes)
│
├── [NULL 指针] → __libc_malloc(bytes)                    // realloc(NULL) == malloc
├── [bytes == 0] → __libc_free(oldmem); return NULL       // realloc(p, 0) == free (可选)
│
├── musable(oldmem)                                       // 检查可用空间
│   └── 若 bytes ≤ usable 且差值 < 2*SIZE_T → 原地返回   // 小缩减不动
│
├── checked_request2size(bytes) → nb                      // 对齐+元数据
├── 安全检查：地址回绕 + 对齐
│
├── [mmap 块路径]
│   ├── mremap_chunk(oldp, nb)                            // 尝试 mremap 原地扩缩
│   │   ├── __mremap(block, old_total, new_total, MREMAP_MAYMOVE)
│   │   ├── madvise_thp()                                // THP 提升
│   │   └── mmap_set_chunk() → 返回新指针
│   ├── [mremap 失败 + 缩减] → 原地返回
│   └── [mremap 失败 + 扩展] → __libc_malloc + memcpy + munmap_chunk
│
├── [heap 块 — 单线程]
│   └── _int_realloc(ar_ptr, oldp, oldsize, nb)
│
└── [heap 块 — 多线程]
    ├── __libc_lock_lock(ar_ptr->mutex)
    ├── _int_realloc(ar_ptr, oldp, oldsize, nb)
    ├── __libc_lock_unlock(ar_ptr->mutex)
    └── [失败重试]
        ├── __libc_malloc(bytes)                          // 尝试其他 arena
        ├── memcpy(newp, oldmem, sz)
        └── _int_free_chunk(ar_ptr, oldp, ...)
```

**outgoing 调用（clangd 分析，21 个）**：

| 被调用函数 | 位置 | 调用行 | 作用 |
|-----------|------|--------|------|
| `__libc_malloc` | malloc.c:3261 | 3367,3437,3469 | NULL/mmap失败/重试 分配 |
| `__libc_free` | malloc.c:3297 | 3372 | realloc(p,0) 释放 |
| `musable` | malloc.c:4743 | 3389 | 查询可用空间大小 |
| `checked_request2size` | malloc.c:1258 | 3413 | 请求→内部大小 |
| `mremap_chunk` | malloc.c:2815 | 3420 | mmap 块原地 mremap |
| `tag_new_usable` | malloc.c:1393 | 3429 | MTE 标签 |
| `memcpy` | multiarch/memcpy.c | 3441,3473 | 数据拷贝 |
| `munmap_chunk` | malloc.c:2785 | 3442 | 释放旧 mmap 映射 |
| `arena_for_chunk` | arena.c:149 | 3446,3452,3463 | 定位 chunk 所属 arena |
| `_int_realloc` | malloc.c:4467 | 3450,3459 | 核心 heap 重分配 |
| `__lll_lock_wait[_private]` | lowlevellock.c | 3457 | arena 锁等待 |
| `__lll_lock_wake[_private]` | lowlevellock.c | 3461 | arena 锁唤醒 |
| `_int_free_chunk` | malloc.c:4260 | 3475 | 重试路径释放旧块 |
| `malloc_printerr` | malloc.c:5256 | 3406 | 安全检查失败终止 |

### 1.2 核心：_int_realloc

**位置**：`malloc/malloc.c:4467`

```
_int_realloc(av, oldp, oldsize, nb)
│
├── 安全检查：oldsize 合法性、next chunk 合法性
│
├── [oldsize >= nb]  → 原地缩减，split 余量
│
├── [扩展路径 1] next == top && 空间足够
│   ├── set_head_size(oldp, nb)
│   ├── av->top = chunk_at_offset(oldp, nb)       // top 后移
│   └── return tag_new_usable(chunk2mem(oldp))     // 原地返回
│
├── [扩展路径 2] next 为空闲 && 合并后足够
│   ├── unlink_chunk(av, next)                     // 从 bin 摘除 next
│   └── → 进入 split 逻辑
│
├── [扩展路径 3] 无法原地扩展
│   ├── _int_malloc(av, nb - MALLOC_ALIGN_MASK)    // 分配新块
│   ├── [新块恰好是 next] → 合并 (避免拷贝)
│   └── [新块在别处]
│       ├── memcpy(newmem, oldmem, sz)
│       ├── _int_free_chunk(av, oldp, ...)         // 释放旧块
│       └── return newmem
│
└── [split 余量]
    ├── remainder_size < MINSIZE → 不拆分，保留padding
    └── remainder_size >= MINSIZE
        ├── 设置 remainder header
        └── _int_free_chunk(av, remainder, ...)    // 余量归还
```

**outgoing 调用（clangd 分析，10 个）**：

| 被调用函数 | 调用行 | 作用 |
|-----------|--------|------|
| `_int_malloc` | 4530 | 无法原地时新分配 |
| `unlink_chunk` | 4524 | 合并 next 空闲块 |
| `memcpy` | 4551 | 数据拷贝到新位置 |
| `_int_free_chunk` | 4552,4580 | 释放旧块/余量 |
| `tag_new_usable` | 4514,4550,4584 | MTE 标签 |
| `tag_region` | 4549,4574 | 清除旧区域标签 |
| `malloc_printerr` | 4483,4494 | 安全校验 |

### 1.3 mremap_chunk（mmap 块的就地调整）

**位置**：`malloc/malloc.c:2815`

```
mremap_chunk(p, new_size)
│
├── 校验 chunk_is_mmapped(p)
├── 计算 block/total_size、new_size 对齐到页
├── [total_size == new_size] → 无需 remap，直接返回
├── __mremap(block, total_size, new_size, MREMAP_MAYMOVE)
│   └── 内核执行虚拟地址空间重映射
├── [total_size < thp_pagesize] → madvise_thp(cp, new_size)  // THP 提升
├── mmap_set_chunk() → 更新 chunk 头
└── 更新 mp_.mmapped_mem 统计
```

### 1.4 设计亮点

1. **musable 快速返回**：小缩减（差值 < 16 bytes）直接返回原指针，避免碎片化
2. **mremap 零拷贝**：mmap 块通过 `mremap(MREMAP_MAYMOVE)` 实现内核级重映射，无需用户态拷贝
3. **_int_realloc 三级扩展**：top 扩展 > next 合并 > malloc+copy+free
4. **巧妙优化**：`_int_malloc` 返回的新块恰好是 next chunk 时，跳过拷贝直接合并
5. **重试机制**：当前 arena 失败后尝试其他 arena（多线程低竞争）

---

## 2. calloc 调用链

### 2.1 入口：__libc_calloc（tcache 快速路径）

**位置**：`malloc/malloc.c:3706`

```
__libc_calloc(n, elem_size)
│
├── __builtin_mul_overflow(n, elem_size, &bytes)   // 溢出检测
│   └── [溢出] → errno=ENOMEM, return NULL
│
├── checked_request2size(bytes) → nb
│
├── [tcache 小块路径] nb < tcache_max_bytes
│   ├── csize2tidx(nb) → tc_idx
│   ├── [小 bin] tc_idx < TCACHE_SMALL_BINS
│   │   ├── tcache_get(tc_idx)                     // 从 tcache 取
│   │   ├── [mtag] → tag_new_zero_region()
│   │   └── clear_memory(mem, tidx2usize)          // 清零
│   └── [大 bin]
│       ├── large_csize2tidx(nb) → tc_idx
│       ├── tcache_get_large(tc_idx, nb)           // 大块 tcache 搜索
│       ├── [mtag] → tag_new_zero_region()
│       └── memset(mem, 0, size)                   // 大块用 memset
│
└── __libc_calloc2(bytes)                          // 慢速路径
```

### 2.2 慢速路径：__libc_calloc2

**位置**：`malloc/malloc.c:3612`

```
__libc_calloc2(sz)
│
├── [单线程] av = &main_arena
├── [多线程] arena_get(av, sz)
│   └── arena_get2() → 创建或切换 arena
│
├── [记录旧 top] 用于后续 sbrk 零页优化
│   ├── main_arena: oldtopsize 扩展到 sbrk_base+max_system_mem
│   └── non-main: heap_for_ptr → heap->mprotect_size
│
├── _int_malloc(av, sz)                            // 核心分配
│
├── [多线程失败重试]
│   ├── arena_get_retry(av, sz)
│   ├── _int_malloc(av, sz)
│   └── __libc_lock_unlock(av->mutex)
│
├── [分配失败] → return NULL
│
├── [mtag 启用] → tag_new_zero_region(mem, memsize)  // 标签+清零一体
│
├── [mmap 块] → 内核已清零
│   ├── [perturb_byte] → memset(mem, 0, sz)        // 仅调试模式
│   └── return mem                                  // 无需清零
│
├── [sbrk 新鲜页优化] MORECORE_CLEARS
│   └── 若 chunk 来自新 sbrk 扩展 → 只清零旧 top 部分
│
└── clear_memory(mem, clearsize)                    // 最终清零
    ├── [nclears ≤ 9] → 展开赋零（无 memset 开销）
    └── [nclears > 9] → memset(d, 0, clearsize)
```

**outgoing 调用（clangd 分析，15 个）**：

| 被调用函数 | 调用行 | 作用 |
|-----------|--------|------|
| `_int_malloc` | 3652,3663 | 核心分配 |
| `arena_get2` | 3623 | 获取/创建 arena |
| `arena_get_retry` | 3662 | 切换 arena 重试 |
| `heap_for_ptr` | 3640 | non-main arena 的 heap 查找 |
| `clear_memory` | 3702 | 小块展开清零 |
| `memset` | 3688 | 大块/调试清零 |
| `tag_new_zero_region` | 3680 | MTE 标签+清零 |
| `arena_for_chunk` | 3655 | 断言校验 |
| `__lll_lock_wait[_private]` | 3623 | arena 锁等待 |
| `__lll_lock_wake[_private]` | 3667 | arena 锁释放 |

### 2.3 clear_memory 清零优化

**位置**：`sysdeps/generic/calloc-clear-memory.h:20`

```c
static __always_inline void *
clear_memory (INTERNAL_SIZE_T *d, unsigned long clearsize)
{
  unsigned long nclears = clearsize / sizeof(INTERNAL_SIZE_T);

  if (nclears > 9)
    return memset(d, 0, clearsize);     // 大块走 memset (SIMD优化)

  if (nclears < 3)
    __builtin_unreachable();            // chunk 最小 3 words

  // 展开赋零：首 3 words + 尾 2 words + 中间 4 words
  *(d+0) = 0; *(d+1) = 0; *(d+2) = 0;
  *(d+nclears-2) = 0; *(d+nclears-1) = 0;
  if (nclears > 5) {
    *(d+3) = 0; *(d+4) = 0;
    *(d+nclears-4) = 0; *(d+nclears-3) = 0;
  }
  return d;
}
```

### 2.4 设计亮点

1. **tcache 快速路径**：calloc 2.43 新增 tcache 直取 + 清零，避免进入 arena 锁
2. **sbrk 零页优化**：内核 sbrk 扩展的内存已清零（`MORECORE_CLEARS`），只需清零旧 top 残留部分
3. **mmap 零页优化**：内核 mmap 保证返回零页，完全跳过清零
4. **展开赋零**：≤72 字节用编译器内联赋零，避免 memset 函数调用开销
5. **溢出安全**：使用 `__builtin_mul_overflow` 在乘法时检测溢出

---

## 3. memalign 调用链

### 3.1 入口：__libc_memalign

**位置**：`malloc/malloc.c:3484`

```
__libc_memalign(alignment, bytes)
│
├── [alignment 非 2 的幂] → 向上调整到 2 的幂
│   └── __bc64_inline()  // bit_ceil: 向上取 2^n
│
└── _mid_memalign(alignment, bytes)
```

### 3.2 中间层：_mid_memalign

**位置**：`malloc/malloc.c:3547`

```
_mid_memalign(alignment, bytes)
│
├── checked_request2size(bytes) → nb
│
├── [tcache 对齐搜索]
│   └── tcache_get_align(nb, alignment)
│       └── 遍历 large tcache bin 找对齐匹配块
│
├── [小对齐 ≤ MALLOC_ALIGNMENT]
│   └── __libc_malloc(bytes)               // 标准 malloc 已满足对齐
│
├── [大对齐 — 单线程]
│   └── _int_memalign(main_arena, alignment, bytes)
│
└── [大对齐 — 多线程]
    ├── arena_get2(av, nb + alignment + MINSIZE)
    ├── _int_memalign(av, alignment, bytes)
    ├── [失败] → arena_get_retry → _int_memalign 重试
    └── __libc_lock_unlock(av->mutex)
```

**outgoing 调用（clangd 分析，15 个）**：

| 被调用函数 | 调用行 | 作用 |
|-----------|--------|------|
| `tcache_get_align` | 3557 | tcache 对齐搜索 |
| `__libc_malloc` | 3554 | 小对齐直接 malloc |
| `_int_memalign` | 3564,3572,3577 | 核心对齐分配 |
| `checked_request2size` | 3557 | 大小计算 |
| `arena_get2` | 3570 | 获取 arena |
| `arena_get_retry` | 3576 | 重试另一 arena |
| `tag_new_usable` | 3559,3567,3585 | MTE 标签 |
| `arena_for_chunk` | 3566,3584 | 断言校验 |

### 3.3 核心：_int_memalign

**位置**：`malloc/malloc.c:4594`

```
_int_memalign(av, alignment, bytes)
│
├── checked_request2size(bytes) → nb
├── _int_malloc(av, nb + alignment + MINSIZE)     // 过量分配
│
├── [mmap 块] — 直接地址对齐
│   ├── newp = PTR_ALIGN_UP(m, alignment)
│   └── mmap_set_chunk(base, size, offset, is_hp)  // 调整 mmap 头
│
├── [heap 块 — 已对齐] → 直接使用
│
├── [heap 块 — 未对齐]
│   ├── newp = ALIGN_UP(m + MINSIZE, alignment)
│   ├── leadsize = newp - p                        // 前导填充
│   ├── set_head(newp, size | PREV_INUSE)
│   ├── set_head_size(p, leadsize)
│   └── _int_free_merge_chunk(av, p, leadsize)     // 释放前导填充
│
└── [释放尾部余量]
    ├── [size - nb >= MINSIZE]
    │   ├── _int_free_create_chunk(av, remainder, ...)  // 创建余量 chunk
    │   └── _int_free_maybe_trim(av, size)              // 可能触发 trim
    └── [余量不足 MINSIZE] → 保留在块内
```

**outgoing 调用（clangd 分析，10 个）**：

| 被调用函数 | 调用行 | 作用 |
|-----------|--------|------|
| `_int_malloc` | 4609 | 过量分配 |
| `_int_free_merge_chunk` | 4639 | 释放前导填充 |
| `_int_free_create_chunk` | 4649 | 创建尾部余量块 |
| `_int_free_maybe_trim` | 4651 | 尝试 trim |
| `mmap_set_chunk` | 4619 | mmap 块头调整 |
| `mmap_base` | 4619,4620 | mmap 基地址 |
| `mmap_size` | 4619 | mmap 映射大小 |
| `mmap_is_hp` | 4620 | 是否大页 |
| `checked_request2size` | 4603 | 大小计算 |

### 3.4 对齐分配算法图解

```
过量分配（nb + alignment + MINSIZE）：
┌──────────────────────────────────────────────────────────┐
│ chunk header │ . . . padding . . . │ aligned data │ tail │
└──────────────────────────────────────────────────────────┘
                 ↑ leadsize            ↑ 用户请求    ↑ remainder
                 (释放到 bin)           (返回给用户)  (释放到 bin)

mmap 块对齐（无需释放 padding）：
┌──────────────────────────────────────────────────────────┐
│ mmap header │ offset │ chunk header │ aligned user data  │
└──────────────────────────────────────────────────────────┘
  ↑ base               ↑ mmap_set_chunk 调整
```

---

## 4. reallocarray（安全重分配）

### 4.1 入口：__libc_reallocarray

**位置**：`malloc/reallocarray.c:24`

```
__libc_reallocarray(optr, nmemb, elem_size)
│
├── __builtin_mul_overflow(nmemb, elem_size, &bytes)
│   └── [溢出] → errno=ENOMEM, return NULL
│
└── realloc(optr, bytes)    // 直接调用 __libc_realloc
```

**设计目的**：防止 `realloc(p, n * size)` 中的整数溢出漏洞。

---

## 5. 相关辅助 API

### 5.1 valloc / pvalloc

```
__libc_valloc(bytes)
└── _mid_memalign(pagesize, bytes)      // alignment = 页大小

__libc_pvalloc(bytes)
├── bytes = ALIGN_UP(bytes, pagesize)   // 大小也对齐到页
└── _mid_memalign(pagesize, bytes)
```

### 5.2 posix_memalign

```
__posix_memalign(memptr, alignment, size)
├── 校验 alignment ≥ sizeof(void*) 且为 2 的幂
├── _mid_memalign(alignment, size)
└── *memptr = result
```

### 5.3 aligned_alloc (C11)

```
aligned_alloc(alignment, size)
├── 校验 size 为 alignment 的整数倍 (C11 要求)
└── __libc_memalign(alignment, size)
```

---

## 6. 调用关系总览

```
用户 API 层
┌─────────────────────────────────────────────────────────────────┐
│ realloc    calloc    memalign    reallocarray    aligned_alloc   │
│ valloc     pvalloc   posix_memalign                              │
└───────┬──────┬─────────┬───────────────┬────────────────────────┘
        │      │         │               │
  glibc 入口层 │         │               │
        │      │         │               │
        ▼      ▼         ▼               ▼
┌────────────┐ ┌─────────┐ ┌──────────────┐ ┌──────────────────┐
│__libc_     │ │__libc_  │ │__libc_       │ │__libc_           │
│realloc     │ │calloc   │ │memalign      │ │reallocarray      │
└────┬───────┘ └────┬────┘ └──────┬───────┘ └────────┬─────────┘
     │              │              │                   │
     │              │              ▼                   │
     │              │        _mid_memalign             │
     │              │         ├─tcache_get_align       │
     │              │         └─_int_memalign          │
     │              │              └─_int_malloc        │
     │              │                                  │
     ▼              ▼                                  ▼
┌──────────┐  ┌──────────┐                    ┌──────────────┐
│mremap_   │  │__libc_   │                    │__libc_realloc│
│chunk     │  │calloc2   │                    └──────────────┘
│(mmap块)  │  │          │
└────┬─────┘  └────┬─────┘
     │              │
     ▼              ▼
  __mremap      _int_malloc
  (syscall)     ├─arena_get
                └─clear_memory / memset
     │
     ▼
_int_realloc (heap块)
├─ [原地缩减] split remainder
├─ [向 top 扩展] 修改 top 指针
├─ [合并 next] unlink_chunk
└─ [malloc+copy+free] _int_malloc + memcpy + _int_free_chunk
```

---

## 7. 关键设计总结

### realloc 策略优先级

| 优先级 | 策略 | 条件 | 代价 |
|--------|------|------|------|
| 1 | 原地返回 | bytes ≤ usable, 差值小 | 零代价 |
| 2 | mremap | mmap 块 | 内核 VMA 操作，零拷贝 |
| 3 | 向 top 扩展 | next == top 且空间足 | 修改 top 指针 |
| 4 | 合并 next 空闲块 | next free 且合并后足够 | unlink + 可能 split |
| 5 | malloc+copy+free | 上述均失败 | 完整拷贝 |
| 6 | 其他 arena 重试 | 当前 arena 失败 | 跨 arena 分配+拷贝 |

### calloc 清零优化层次

| 层次 | 条件 | 清零方式 |
|------|------|----------|
| 1 | mmap 块 | 无需清零（内核保证） |
| 2 | sbrk 新鲜页 | 部分清零（仅旧 top 残留） |
| 3 | MTE 启用 | tag_new_zero_region 一体化 |
| 4 | ≤ 72 字节 | clear_memory 展开赋零 |
| 5 | > 72 字节 | memset (SIMD 向量化) |

### memalign 关键约束

- 过量分配 `nb + alignment + MINSIZE`：确保一定能找到对齐地址
- 前导填充 ≥ MINSIZE：确保能形成合法 chunk 释放
- mmap 块特殊处理：调整 offset 即可，无需释放填充

---

## 8. 源文件索引

| 文件 | 包含函数 |
|------|----------|
| `malloc/malloc.c` | __libc_realloc, __libc_calloc, __libc_calloc2, __libc_memalign, _mid_memalign, _int_realloc, _int_memalign, mremap_chunk, munmap_chunk, musable |
| `malloc/reallocarray.c` | __libc_reallocarray |
| `sysdeps/generic/calloc-clear-memory.h` | clear_memory |
| `sysdeps/unix/sysv/linux/mremap.c` | __mremap |
| `nptl/lowlevellock.c` | __lll_lock_wait, __lll_lock_wake |
| `malloc/arena.c` | arena_for_chunk, arena_get2, arena_get_retry, heap_for_ptr |
