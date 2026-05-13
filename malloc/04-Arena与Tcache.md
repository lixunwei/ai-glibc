# Arena 管理与 Tcache 机制

## 一、Arena 多线程架构

### 设计目标

传统 malloc 使用单一全局锁，高并发时成为瓶颈。ptmalloc2 通过**多 arena** 减少锁竞争：

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Thread 1│   │ Thread 2│   │ Thread 3│   │ Thread 4│
└────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
     │              │              │              │
     ▼              ▼              ▼              ▼
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Tcache 1│   │ Tcache 2│   │ Tcache 3│   │ Tcache 4│
│ (无锁)  │   │ (无锁)  │   │ (无锁)  │   │ (无锁)  │
└────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
     │              │              │              │
     ▼              ▼              ▼              ▼
┌──────────────┐   ┌──────────────┐
│  Arena 0     │   │  Arena 1     │
│ (main_arena) │   │ (secondary)  │
│   mutex      │   │   mutex      │
└──────────────┘   └──────────────┘
```

- 每线程有独立 tcache（完全无锁）
- 多个线程可能共享同一 arena（需加锁）
- arena 数量动态增长，受 CPU 核数限制

---

## 二、Arena 获取流程

### arena_get

**源文件**: `malloc/arena.c:119-137`

```
arena_get(thread_arena):
  ar_ptr = thread_arena         // TLS 变量
  if ar_ptr != NULL:
    __libc_lock_lock(ar_ptr->mutex)
    return ar_ptr
  else:
    return arena_get2(size, NULL)
```

### thread_arena TLS 变量

**源文件**: `malloc/arena.c:87-90`

```c
static __thread mstate thread_arena;
```

- 每个线程记住自己上次使用的 arena
- 线程退出时通过 `__malloc_arena_thread_freeres` 解绑

### arena_get2 — 创建或重用

**源文件**: `malloc/arena.c:802-849`

```
arena_get2(size, avoid):
  // 1. 尝试空闲列表
  lock(free_list_lock)
  if free_list 非空:
    result = 从 free_list 取一个
    thread_arena = result
    return result
  
  // 2. 检查 arena 数量限制
  if narenas <= narenas_limit - 1:
    result = _int_new_arena(size)
    return result
  
  // 3. 达到上限，重用已有 arena
  result = reused_arena(avoid)
  return result
```

### Arena 数量限制

**源文件**: `malloc/arena.c:802-826`

```
narenas_limit 计算规则:
  if mp_.arena_max != 0:
    narenas_limit = mp_.arena_max          // 用户显式指定
  else:
    当 narenas > mp_.arena_test(默认=8) 时:
      ncores = get_nprocs()
      narenas_limit = NARENAS_FROM_NCORES(ncores)
      // 64位: 8 * ncores
      // 32位: 2 * ncores
```

### 竞争重试

**源文件**: `malloc/arena.c:855-871`

```
arena_get_retry(arena, size):
  // 当前 arena 内存不足
  unlock(arena)
  if arena != main_arena:
    lock(main_arena)
    return main_arena        // 先试主 arena
  else:
    return arena_get2(size, arena)  // 创建新的或重用
```

---

## 三、Arena 创建

### _int_new_arena

**源文件**: `malloc/arena.c:615-680`

```
_int_new_arena(size):
  // 1. 分配新 heap
  h = new_heap(size + sizeof(heap_info) + sizeof(malloc_state))
  if h == NULL:
    h = new_heap(sizeof(heap_info) + sizeof(malloc_state))  // 最小 heap
  
  // 2. arena 紧跟 heap_info 之后
  a = (mstate)(h + 1)
  malloc_init_state(a)
  a->attached_threads = 1
  a->system_mem = a->max_system_mem = h->size
  
  // 3. 设置 top chunk
  top = (mchunkptr)(a + 1)  // arena 结构之后
  set_head(top, (h->size - overhead) | PREV_INUSE)
  a->top = top
  
  // 4. 加入全局 arena 链表
  lock(list_lock)
  a->next = main_arena.next
  main_arena.next = a
  unlock(list_lock)
  // narenas 在 arena_get2 中通过原子 CAS 递增 (arena.c:837-843)
  
  // 5. 绑定当前线程
  thread_arena = a
  lock(a->mutex)
  return a
```

### new_heap — 分配堆内存

**源文件**: `malloc/arena.c:343-453`（`alloc_new_heap` 343-437 + `new_heap` 440-453）

```
alloc_new_heap(size):
  // 先映射 max_size*2 区域再裁剪对齐（或 mmap HEAP_MAX_SIZE 直接对齐）
  p = mmap(NULL, HEAP_MAX_SIZE*2, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS)
  aligned = ALIGN_UP(p, HEAP_MAX_SIZE)
  // 裁掉多余部分
  munmap(p, aligned - p)
  munmap(aligned + HEAP_MAX_SIZE, ...)
  
  // 只开放需要的部分
  mprotect(aligned, size, PROT_READ|PROT_WRITE)
  
  h = (heap_info*)aligned
  h->size = size
  h->mprotect_size = size
  return h
```

- `HEAP_MAX_SIZE` = 64MB（64 位）/ 1MB（32 位）
- 初始只 mprotect 需要的页，后续通过 `grow_heap` 扩展

### grow_heap / shrink_heap

**源文件**: `malloc/arena.c:455-515`

```
grow_heap(heap, diff):
  new_size = heap->size + diff
  if new_size > HEAP_MAX_SIZE: return -1
  if new_size > heap->mprotect_size:
    mprotect(heap + mprotect_size, diff, PROT_READ|PROT_WRITE)
    heap->mprotect_size = new_size
  heap->size = new_size

shrink_heap(heap, diff):
  new_size = heap->size - diff
  // 优先尝试 MAP_FIXED 重映射为 PROT_NONE，否则 madvise
  madvise(heap + new_size, diff, MADV_DONTNEED)  // 释放物理页
  heap->size = new_size
```

---

## 四、Arena 线程解绑

**源文件**: `malloc/arena.c:875-897`

```
__malloc_arena_thread_freeres():
  // 线程退出时调用
  tcache_thread_shutdown()      // 清空 tcache
  a = thread_arena
  thread_arena = NULL
  
  lock(free_list_lock)
  a->attached_threads--
  if a->attached_threads == 0:
    a->next_free = free_list    // 放入空闲列表
    free_list = a
  unlock(free_list_lock)
```

---

## 五、Tcache 机制

### 数据结构

**源文件**: `malloc/malloc.c:2864-2907`

```c
typedef struct tcache_entry {
    struct tcache_entry *next;     // 单链表指针
    uintptr_t key;                // == tcache_key 时表示已在 tcache 中
} tcache_entry;

typedef struct tcache_perthread_struct {
    uint16_t num_slots[TCACHE_MAX_BINS];   // 每 bin 剩余可放入数
    tcache_entry *entries[TCACHE_MAX_BINS]; // 各 bin 链表头
} tcache_perthread_struct;
```

### Tcache 参数

| 参数 | 默认值 | 可调节 | 说明 |
|------|--------|--------|------|
| `TCACHE_MAX_BINS` | 76 | 否（编译期） | bin 数量 |
| `TCACHE_FILL_COUNT` | 16 | `glibc.malloc.tcache_count` | 每 bin 容量 |
| `mp_.tcache_max_bytes` | ~1216B | `glibc.malloc.tcache_max` | 最大 chunk 大小 |
| `tcache_key` | random | 否 | 双重释放检测 cookie |

### Tcache 大小覆盖（64 位）

```
bin[0]:  32 字节 chunk  (用户 16 字节)
bin[1]:  48 字节 chunk  (用户 32 字节)
bin[2]:  64 字节 chunk  (用户 48 字节)
...
bin[63]: 1024 字节 chunk (小 tcache bins)
bin[64]: 1040 字节
...
bin[75]: 1216 字节 (大 tcache bins)
```

### tcache_init

**源文件**: `malloc/malloc.c:3182-3210`

```
tcache_init(arena):
  // 临时禁用 tcache 避免递归
  tcache = &__tcache_dummy.disabled
  
  // 从 arena 分配 tcache 结构体本身
  victim = _int_malloc(arena, sizeof(tcache_perthread_struct))
  // 或用 __libc_malloc2 如果 arena 不可用
  
  tcache = (tcache_perthread_struct*)victim
  memset(tcache, 0, sizeof(*tcache))
  
  // 设置每个 bin 的初始容量
  for i in 0..TCACHE_MAX_BINS-1:
    tcache->num_slots[i] = mp_.tcache_count  // 默认 16
```

- tcache 自身也是通过 malloc 分配的（bootstrap 问题通过临时禁用解决）
- 首次 malloc 时延迟初始化

### tcache_put / tcache_get

**源文件**: `malloc/malloc.c:2982-3039`

```
tcache_put(chunk, tc_idx):
  e = chunk2mem(chunk)
  e->key = tcache_key             // 标记已入 tcache
  e->next = PROTECT_PTR(&e->next, tcache->entries[tc_idx])  // 指针保护
  tcache->entries[tc_idx] = e
  tcache->num_slots[tc_idx]--     // 递减可用槽位

tcache_get(tc_idx):
  e = tcache->entries[tc_idx]
  tcache->entries[tc_idx] = e->next
  e->key = 0                      // 清除标记
  tcache->num_slots[tc_idx]++     // 恢复槽位
  return e
```

- **完全无锁**: 只操作线程本地结构
- **LIFO** 顺序：最近释放的最先复用（缓存友好）

### 双重释放检测

**源文件**: `malloc/malloc.c:2931-2970, 3119-3145, 3322-3327`

```
tcache_key 初始化:
  getrandom(&tcache_key, sizeof(tcache_key))
  // 确保不为 0（避免误判）

free 时检测:
  e = chunk2mem(p)
  if e->key == tcache_key:
    // 可能是双重释放，遍历链表确认
    tcache_double_free_verify(e):
      // 遍历所有 TCACHE_MAX_BINS 检查 (malloc.c:3119-3145)
      for each bin in tcache->entries[0..TCACHE_MAX_BINS-1]:
        for each entry in bin:
        if entry == e:
          malloc_printerr("free(): double free detected in tcache")
```

- `key` 字段位于用户数据区（`fd` 之后的位置）
- 只有 key 恰好等于 tcache_key 时才触发链表扫描（概率极低误报）

---

## 六、Arena vs Tcache 对比

| 特性 | Tcache | Arena Bins |
|------|--------|-----------|
| 锁 | 无（线程私有） | arena mutex |
| 数据结构 | 单链表 LIFO | 双向循环链表 |
| 合并 | 不合并 | 自动合并相邻块 |
| 适用场景 | 高频小分配 | 大块 / tcache miss |
| 双重释放检测 | tcache_key cookie | prev_inuse 位检查 |
| 内存开销 | 每线程 ~5KB | 共享 |

---

## 七、主 Arena vs 二级 Arena

| 特性 | main_arena | Secondary Arena |
|------|-----------|----------------|
| 分配方式 | 静态全局变量 | mmap(HEAP_MAX_SIZE) |
| 堆扩展 | sbrk() | grow_heap() / new_heap() |
| 堆收缩 | systrim (sbrk负增长) | shrink_heap (madvise) |
| 堆上限 | 无硬性限制 | HEAP_MAX_SIZE per heap |
| chunk 标志 | NON_MAIN_ARENA = 0 | NON_MAIN_ARENA = 1 |
| 数量 | 恰好 1 个 | 0 ~ narenas_limit-1 个 |

---

## 八、Arena 环形链表

```
main_arena → arena_1 → arena_2 → ... → main_arena
     ↕              ↕            ↕
  free_list     free_list     free_list
```

- `main_arena.next` 链接所有 arena 形成环
- `free_list` 串联 attached_threads==0 的空闲 arena
- 新线程优先从 free_list 获取 arena

---

## 九、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `malloc/arena.c:67-79` | `heap_info` 结构 |
| `malloc/arena.c:87-90` | `thread_arena` TLS |
| `malloc/arena.c:91-100` | `free_list` / `free_list_lock` |
| `malloc/arena.c:119-137` | `arena_get` |
| `malloc/arena.c:343-453` | `new_heap` / `alloc_new_heap` |
| `malloc/arena.c:455-485` | `grow_heap` |
| `malloc/arena.c:487-515` | `shrink_heap` |
| `malloc/arena.c:520-592` | `heap_trim` |
| `malloc/arena.c:615-680` | `_int_new_arena` |
| `malloc/arena.c:802-849` | `arena_get2` + 限制逻辑 |
| `malloc/arena.c:855-871` | `arena_get_retry` |
| `malloc/arena.c:875-897` | `__malloc_arena_thread_freeres` |
| `malloc/malloc.c:1794-1805` | `main_arena` 定义 |
| `malloc/malloc.c:2864-2907` | Tcache 结构定义 |
| `malloc/malloc.c:2931-2970` | `tcache_key` 初始化 |
| `malloc/malloc.c:2982-3039` | `tcache_put` / `tcache_get` |
| `malloc/malloc.c:3119-3145` | 双重释放验证 |
| `malloc/malloc.c:3182-3210` | `tcache_init` |
