# malloc 分配路径

## 概述

`malloc(size)` 的完整调用路径：

```
__libc_malloc(size)
  ├─ checked_request2size(size) → nb
  ├─ tcache_get(nb)            → 命中则直接返回
  ├─ arena_get(thread_arena)   → 加锁 arena
  └─ _int_malloc(arena, nb)
       ├─ smallbin 精确匹配
       ├─ unsorted bin 扫描
       ├─ largebin best-fit
       ├─ binmap 跳跃扫描
       ├─ top chunk 切割
       └─ sysmalloc 扩展
```

---

## 一、入口：__libc_malloc

**源文件**: `malloc/malloc.c:3261-3286`（tcache 快速路径）+ `malloc/malloc.c:3224-3258`（`__libc_malloc2` 慢路径）

> **注意**: 当前版本将 `__libc_malloc` 拆分为两层：
> - `__libc_malloc`（3261-3286）：仅做 tcache 查找，命中即返回
> - `__libc_malloc2`（3224-3258）：arena_get + _int_malloc + 重试逻辑
> 
> 下面的伪代码合并了两者以便理解整体流程。

```
__libc_malloc(bytes):
  nb = checked_request2size(bytes)    // 请求→chunk大小
  
  // 快速路径: tcache
  if nb < mp_.tcache_max_bytes:
    tc_idx = csize2tidx(nb)
    if tcache->entries[tc_idx] != NULL:
      return tcache_get(tc_idx)       // 无锁 O(1)
  
  // 大 tcache bin (1024~1216)
  tc_idx = large_csize2tidx(nb)
  if tcache->entries[tc_idx] != NULL:
    return tcache_get(tc_idx)
  
  // 慢路径: 进入 arena
  arena = arena_get(thread_arena)
  victim = _int_malloc(arena, nb)
  return chunk2mem(victim)
```

### 大小转换

**源文件**: `malloc/malloc.c:1245-1280`

```c
// request2size: 用户请求 → 实际 chunk 大小
#define request2size(req) \
    (((req) + CHUNK_HDR_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
    // 至少 MINSIZE(32)

// checked_request2size: 含溢出检查 + 标签对齐
// malloc/malloc.c:1257-1280
```

---

## 二、_int_malloc 核心算法

**源文件**: `malloc/malloc.c:3757-4250`

### 步骤 1: Smallbin 精确匹配

**源文件**: `malloc/malloc.c:3811-3860`

```
if nb < MIN_LARGE_SIZE:
  idx = smallbin_index(nb)
  bin = bin_at(arena, idx)
  victim = last(bin)           // FIFO: 取最旧的
  if victim != bin:            // bin 非空
    unlink(victim)
    set_inuse_bit_at_offset(victim)
    // tcache 机会性填充
    return victim
```

**tcache 填充**: 在 smallbin 中找到 victim 后，还会将同 bin 中其他 chunk 填入 tcache（最多填满）。

### 步骤 2: Unsorted Bin 扫描

**源文件**: `malloc/malloc.c:3876-4015`

```
while (victim = unsorted_chunks(arena)->bk) != unsorted_chunks(arena):
  size = chunksize(victim)
  
  // 精确匹配快捷路径
  if size == nb:
    unlink_from_unsorted(victim)
    return victim
  
  // last_remainder 切割（仅小请求）
  if victim == last_remainder && size > nb + MINSIZE:
    remainder = split(victim, nb)
    last_remainder = remainder
    unsorted_bin.insert(remainder)
    return victim
  
  // 放入对应 small/large bin
  if size < MIN_LARGE_SIZE:
    smallbin[idx].insert_head(victim)
  else:
    largebin[idx].sorted_insert(victim)
  
  // tcache 机会性填充
  if tcache_nb 已匹配且 tcache 未满:
    tcache_put(victim)
    continue
```

**关键设计**: unsorted bin 扫描同时完成**三件事**：
1. 寻找精确匹配
2. 将 chunk 分类放入正确 bin
3. 顺便填充 tcache

### 步骤 3: Large Bin Best-fit

**源文件**: `malloc/malloc.c:4032-4090`

```
idx = largebin_index(nb)
bin = bin_at(arena, idx)

if (victim = first(bin)) != bin:
  // 检查最大块是否够用
  if chunksize(last(bin)) >= nb:
    // 沿 bk_nextsize 找最小够用的
    victim = traverse_nextsize(bin, nb)
    
    if remainder_size >= MINSIZE:
      remainder = split(victim, nb)
      unsorted_bin.insert(remainder)
    
    return victim
```

- Large bin 按大小**降序**排列
- `fd_nextsize`/`bk_nextsize` 链接不同大小的代表 chunk（跳表思想）
- Best-fit: 找最小的 ≥ nb 的 chunk

### 步骤 4: Binmap 跳跃扫描

**源文件**: `malloc/malloc.c:4105-4199`

```
// 当前 large bin 为空或不够大，扫描更大的 bin
bit = binmap_bit(idx)
map = arena->binmap[block]

// 用位运算快速跳过空 bin
bit = next_set_bit(map, bit)
if found:
  bin = bin_at(arena, bit_to_idx)
  victim = last(bin)     // 取最大的（一定够用）
  split_and_return(victim, nb)
```

### 步骤 5: Top Chunk 切割

**源文件**: `malloc/malloc.c:4202-4248`

```
victim = arena->top
size = chunksize(victim)

if size >= nb + MINSIZE:
  remainder = chunk_at_offset(victim, nb)
  arena->top = remainder
  set_head(victim, nb | PREV_INUSE)
  set_head(remainder, remainder_size | PREV_INUSE)
  return victim
else:
  // top 不够
  return sysmalloc(nb, arena)
```

### 步骤 6: sysmalloc 扩展

```
sysmalloc(nb, arena):
  if nb >= mmap_threshold && n_mmaps < n_mmaps_max:
    直接 mmap 独立映射
  else if arena == main_arena:
    sbrk(nb + top_pad) 扩展堆
  else:
    grow_heap() 或 new_heap() 扩展二级 arena
  
  设置新 top chunk
  重新尝试 _int_malloc
```

---

## 三、分配路径流程图

```
┌──────────────────────────────────────────────────┐
│              __libc_malloc(size)                   │
├──────────────────────────────────────────────────┤
│                                                  │
│  ① checked_request2size → nb                     │
│                                                  │
│  ② Tcache 查找 ─────── 命中 → return             │
│     │                                            │
│     │ miss                                       │
│     ▼                                            │
│  ③ arena_get + lock                              │
│     │                                            │
│     ▼                                            │
│  ④ _int_malloc(arena, nb)                        │
│     │                                            │
│     ├─ smallbin[idx] ─── 非空 → unlink+return    │
│     │                                            │
│     ├─ unsorted bin scan ─ exact match → return  │
│     │   (同时分类+填tcache)                       │
│     │                                            │
│     ├─ largebin best-fit ─ 找到 → split+return   │
│     │                                            │
│     ├─ binmap scan ─── 更大bin → split+return    │
│     │                                            │
│     ├─ top chunk ──── 够大 → split+return        │
│     │                                            │
│     └─ sysmalloc ──── 扩展 → 重试                │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## 四、Chunk 切割规则

当找到的 chunk 大于请求时：

```
if remainder_size >= MINSIZE(32):
  切割: 返回前部 nb，剩余部分插入 unsorted bin
else:
  不切割: 整个 chunk 返回（避免产生过小碎片）
```

- `MINSIZE = 32`（64 位）确保剩余块能存放完整的 malloc_chunk 头
- 剩余块总是进入 **unsorted bin**（不直接放入 small/large bin）

---

## 五、Tcache 机会性填充

在 smallbin 和 unsorted bin 扫描中，malloc 会**顺便**填充 tcache：

```
// smallbin 取 chunk 后 (malloc.c:3828-3855)
while same bin still has chunks && tcache not full:
  tcache_put(next_chunk)

// unsorted bin 扫描中 (malloc.c:3940-3949)  
if chunk_size == nb && tcache not full:
  tcache_put(chunk)  // 不直接返回，先填 tcache
  continue scanning
// 填满后才返回第一个匹配
```

这确保下次相同大小的 malloc 能直接命中 tcache。

---

## 六、性能特征

| 路径 | 复杂度 | 是否加锁 | 系统调用 |
|------|--------|---------|---------|
| tcache hit | O(1) | 否 | 无 |
| smallbin | O(1) | arena mutex | 无 |
| unsorted scan | O(n) 最坏 | arena mutex | 无 |
| largebin | O(log n) 实际 | arena mutex | 无 |
| top chunk | O(1) | arena mutex | 无 |
| sysmalloc | O(1) | arena mutex | sbrk/mmap |
| mmap 直接 | O(1) | 否 | mmap |

---

## 七、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `malloc/malloc.c:1245-1280` | request2size / checked_request2size |
| `malloc/malloc.c:3261-3286` | `__libc_malloc` 入口（tcache 快速路径） |
| `malloc/malloc.c:3224-3258` | `__libc_malloc2`（arena + _int_malloc 慢路径） |
| `malloc/malloc.c:3757-3791` | `_int_malloc` 入口 + 大小检查 |
| `malloc/malloc.c:3811-3860` | Smallbin 路径 |
| `malloc/malloc.c:3876-4025` | Unsorted bin 扫描 + 分类入 bin |
| `malloc/malloc.c:4032-4091` | Large bin best-fit |
| `malloc/malloc.c:4105-4200` | Binmap 扫描 |
| `malloc/malloc.c:4202-4248` | Top chunk 切割 |
| `malloc/malloc.c:2229-2258` | sysmalloc_mmap 路径 |
