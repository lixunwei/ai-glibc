# free 释放路径

## 概述

`free(ptr)` 的完整调用路径：

```
__libc_free(ptr)
  ├─ NULL/对齐检查
  ├─ tcache_put() → 命中则直接返回
  └─ _int_free_chunk(arena, chunk)
       ├─ mmap chunk? → munmap_chunk()
       └─ _int_free_merge_chunk(arena, p, size)
            ├─ 后向合并（与前块合并）
            └─ _int_free_create_chunk()
                 ├─ 前向合并（与后块合并）
                 ├─ 大块 → unsorted bin
                 ├─ 小块 → 直接入 smallbin
                 └─ 或合并入 top chunk
            └─ _int_free_maybe_trim()
```

---

## 一、入口：__libc_free

**源文件**: `malloc/malloc.c:3297-3355`

```
__libc_free(mem):
  if mem == NULL: return
  
  p = mem2chunk(mem)
  
  // 指针对齐检查
  if misaligned(p): malloc_printerr("free(): invalid pointer")
  
  // tcache 快速路径
  size = chunksize(p)
  tc_idx = csize2tidx(size)
  if tc_idx < TCACHE_MAX_BINS:
    // 双重释放检测
    e = (tcache_entry*)mem
    if e->key == tcache_key:
      tcache_double_free_verify(e, tc_idx)  // 扫描链表确认
    
    if tcache->num_slots[tc_idx] > 0:
      tcache_put(p, tc_idx)    // 入 tcache，return
      return
  
  // 慢路径
  _int_free_chunk(arena, p, size)
```

### tcache_put 操作

```
tcache_put(chunk, tc_idx):
  e = chunk2mem(chunk)
  e->next = tcache->entries[tc_idx]
  e->key = tcache_key              // 设置 cookie
  tcache->entries[tc_idx] = e
  tcache->num_slots[tc_idx]--      // 递减剩余槽位
```

---

## 二、_int_free_chunk 分发

**源文件**: `malloc/malloc.c:4260-4309`

```
_int_free_chunk(arena, p, size):
  if chunk_is_mmapped(p):
    // mmap 分配的，直接 munmap
    munmap_chunk(p)
    // 动态调整 mmap 阈值
    if size > mp_.mmap_threshold && !mp_.no_dyn_threshold:
      mp_.mmap_threshold = size
      mp_.trim_threshold = 2 * size
    return
  
  // 普通 arena chunk
  _int_free_merge_chunk(arena, p, size)
```

---

## 三、合并算法

**源文件**: `malloc/malloc.c:4315-4460`

当前版本将 free 的合并逻辑重构为三个独立函数：
- `_int_free_merge_chunk`（4315-4356）：安全检查 + 后向合并 + 调用下面两个函数
- `_int_free_create_chunk`（4364-4432）：前向合并 + 放入 bin（或合并入 top）
- `_int_free_maybe_trim`（4436-4460）：根据阈值决定是否收缩

### 安全检查

```
// malloc/malloc.c:4318-4338（在 _int_free_merge_chunk 中）
if p == arena->top: abort("double free or corruption (top)")
if next chunk 超出 arena 范围: abort("invalid next size")
if !prev_inuse(next): abort("double free or corruption (!prev)")
if next->size 不合理: abort("invalid next size (normal)")
```

### 后向合并（与物理前一块合并）

**源文件**: `malloc/malloc.c:4342-4351`

```
if !prev_inuse(p):                    // 前块是空闲的
  prevsize = prev_size(p)
  p = chunk_at_offset(p, -prevsize)   // 移到前块起始
  // 验证 prev_size 一致性
  if chunksize(p) != prevsize: abort("corrupted size vs. prev_size")
  unlink_chunk(arena, p)              // 从 bin 中摘除前块
  size += prevsize                    // 合并大小
```

### 前向合并（与物理后一块合并）+ 放入 Bin

**源文件**: `malloc/malloc.c:4364-4432`（`_int_free_create_chunk` 函数）

```
nextchunk = chunk_at_offset(p, size)
nextsize = chunksize(nextchunk)

if nextchunk == arena->top:
  // 情况A: 后面是 top → 合并入 top
  goto merge_with_top

if !inuse(nextchunk):                   // 后块是空闲的
  unlink_chunk(arena, nextchunk)        // 从 bin 中摘除后块
  size += nextsize                      // 合并大小

// 放入对应 bin（注意：不是全部进 unsorted bin！）
if size >= MIN_LARGE_SIZE:
  // 大块 → unsorted bin（等待 malloc 时再分类）
  unsorted_bin.insert(p)
  p->fd_nextsize = NULL
  p->bk_nextsize = NULL
else:
  // 小块 → 直接放入对应 smallbin（避免污染 unsorted bin）
  idx = smallbin_index(size)
  smallbin[idx].insert(p)
  mark_bin(arena, idx)

set_head(p, size | PREV_INUSE)
set_foot(p, size)
```

**重要设计变化**: 当前版本的 glibc 不再将所有合并后的 chunk 都放入 unsorted bin，
而是将小 chunk 直接放入 smallbin，仅大 chunk 进入 unsorted bin。

---

## 四、合并流程图

```
free(ptr) → chunk p, size s
            │
            ▼
┌─── prev_inuse(p) == 0? ───┐
│ YES: 后向合并               │ NO: 跳过
│   p = prev_chunk            │
│   s += prev_size            │
│   unlink(prev from bin)     │
└─────────────┬───────────────┘
              ▼
┌─── next == top? ───────────┐
│ YES: 合并入 top             │ NO: 继续
│   top = p                   │
│   top.size = s + top.size   │
│   可能 trim                 │
│   return                    │
└─────────────┬───────────────┘
              ▼
┌─── next chunk is free? ────┐
│ YES: 前向合并               │ NO: 跳过
│   s += next.size            │
│   unlink(next from bin)     │
└─────────────┬───────────────┘
              ▼
┌─── in_smallbin_range(s)? ──┐
│ YES: 直接入 smallbin        │ NO: 入 unsorted bin
│   idx = smallbin_index(s)   │   unsorted_bin.insert(p)
│   smallbin[idx].insert(p)   │   fd_nextsize = NULL
│   mark_bin(arena, idx)      │   bk_nextsize = NULL
└─────────────┬───────────────┘
              ▼
     设置 next_chunk->prev_size = s
     清除 next_chunk 的 PREV_INUSE
```

---

## 五、内存归还操作系统

### systrim（Main Arena）

**源文件**: `malloc/malloc.c:2721-2782`

```
systrim(pad, arena):
  top_size = chunksize(arena->top)
  extra = top_size - pad - MINSIZE
  if extra < (long)mp_.pagesize: return 0
  
  extra = ALIGN_DOWN(extra, mp_.pagesize)
  // 调用 sbrk(-extra) 收缩堆
  if __brk(current_brk - extra) 成功:
    arena->system_mem -= extra
    set_head(top, new_size | PREV_INUSE)
```

- 当 top chunk 大于 `trim_threshold`（默认 128KB）时触发
- 通过 `sbrk()` 负增长归还页面

### heap_trim（Secondary Arena）

**源文件**: `malloc/arena.c:520-592`

```
heap_trim(heap, pad):
  // 如果 top chunk 占据了整个 heap（除 heap_info）
  if top == heap + sizeof(heap_info):
    prev_heap = heap->prev
    unmap 整个 heap
    prev_heap 的 top 成为新 top
  else:
    // 收缩当前 heap
    shrink_heap(heap, extra)
```

### shrink_heap

**源文件**: `malloc/arena.c:487-515`

```
shrink_heap(heap, diff):
  new_size = heap->size - diff
  if 支持 MADV_DONTNEED:
    madvise(addr, diff, MADV_DONTNEED)   // 释放物理页
  else:
    mprotect(addr, diff, PROT_NONE)       // 取消映射权限
  heap->size = new_size
```

### munmap_chunk（大块释放）

**源文件**: `malloc/malloc.c:2785-2810`

```
munmap_chunk(p):
  size = chunksize(p)
  munmap(chunk - offset, size + offset)   // offset 用于对齐补偿
  mp_.n_mmaps--
  mp_.mmapped_mem -= size
```

### __malloc_trim（手动触发）

**源文件**: `malloc/malloc.c:4718-4734`

```
__malloc_trim(pad):
  遍历所有 arena:
    lock(arena)
    // 对每个 arena 尝试 systrim 或 heap_trim
    // 对可 madvise 的区域调用 MADV_DONTNEED
    unlock(arena)
```

---

## 六、安全检查汇总

| 检查位置 | 检查内容 | 错误信息 |
|----------|---------|---------|
| `__libc_free:3316` | 指针对齐 | "free(): invalid pointer" |
| `__libc_free:3326` | tcache 双重释放 | "free(): double free detected in tcache" |
| `__libc_free:3349` | size 溢出 | "free(): invalid size" |
| `_int_free_merge_chunk:4324` | chunk == top | "double free or corruption (top)" |
| `_int_free_merge_chunk:4327` | next 超范围 | "double free or corruption (out)" |
| `_int_free_merge_chunk:4332` | !prev_inuse(next) | "double free or corruption (!prev)" |
| `_int_free_merge_chunk:4336` | next size 不合理 | "free(): invalid next size (normal)" |
| `_int_free_merge_chunk:4348` | prev_size 不一致 | "corrupted size vs. prev_size" |

---

## 七、Mmap 阈值动态调整

**源文件**: `malloc/malloc.c:4294-4304`

```
// 释放 mmap chunk 时
if !mp_.no_dyn_threshold:
  if size > mp_.mmap_threshold:
    mp_.mmap_threshold = size          // 提高 mmap 阈值
    mp_.trim_threshold = 2 * size      // trim 阈值跟随调整
```

- 如果应用频繁分配大块，阈值会自动升高
- 避免反复在 mmap/brk 之间切换（减少碎片）
- `mallopt(M_MMAP_THRESHOLD, ...)` 禁用动态调整

---

## 八、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `malloc/malloc.c:3297-3355` | `__libc_free` 入口 |
| `malloc/malloc.c:3322-3345` | tcache 双重释放检测 |
| `malloc/malloc.c:4260-4309` | `_int_free_chunk` 分发 |
| `malloc/malloc.c:4315-4356` | `_int_free_merge_chunk`（安全检查+后向合并） |
| `malloc/malloc.c:4318-4338` | 安全检查（top/范围/prev_inuse/nextsize） |
| `malloc/malloc.c:4342-4351` | 后向合并 |
| `malloc/malloc.c:4364-4432` | `_int_free_create_chunk`（前向合并 + 放入 bin） |
| `malloc/malloc.c:4382-4396` | 大块 → unsorted bin |
| `malloc/malloc.c:4397-4409` | 小块 → 直接入 smallbin |
| `malloc/malloc.c:4421-4431` | Top chunk 合并 |
| `malloc/malloc.c:4436-4460` | `_int_free_maybe_trim` |
| `malloc/malloc.c:2721-2782` | systrim (sbrk 收缩) |
| `malloc/malloc.c:2785-2810` | munmap_chunk |
| `malloc/arena.c:487-515` | shrink_heap (madvise) |
| `malloc/arena.c:520-592` | heap_trim |
| `malloc/malloc.c:4718-4734` | __malloc_trim |
