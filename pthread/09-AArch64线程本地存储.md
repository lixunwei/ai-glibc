# AArch64 线程本地存储 (TLS) 深度分析

## 概述

线程本地存储 (Thread-Local Storage, TLS) 允许每个线程拥有独立的全局变量副本。
在 AArch64 平台上，glibc 使用 **TLS_DTV_AT_TP** 模型：线程指针 (`tpidr_el0`)
指向 TCB (Thread Control Block)，TLS 数据块放在 TCB **之后**。

本文分析 glibc TLS 子系统的完整实现，包括数据结构、内存布局、分配管理、
访问模型和线程创建时的 TLS 初始化。

---

## 一、核心数据结构

### 1.1 TCB 头部 (tcbhead_t)

**源文件**: `sysdeps/aarch64/nptl/tls.h:43-47`

```c
typedef struct {
  dtv_t *dtv;      // 指向 DTV (动态线程向量)
  void *private;    // 保留字段
} tcbhead_t;
```

> AArch64 的 tcbhead_t 非常简洁，只有 16 字节。

### 1.2 DTV (Dynamic Thread Vector)

**源文件**: `sysdeps/generic/dl-dtv.h:22-36`

```c
struct dtv_pointer {
  void *val;       // 指向 TLS 数据块，或 TLS_DTV_UNALLOCATED
  void *to_free;   // malloc 返回的原始指针（用于 free）
};

typedef union dtv {
  size_t counter;              // dtv[0]: DTV 代数计数器
  struct dtv_pointer pointer;  // dtv[1..N]: 各模块的 TLS 指针
} dtv_t;

#define TLS_DTV_UNALLOCATED ((void *) -1l)
```

DTV 是一个数组，索引 0 特殊用于存储代数 (generation) 计数器，
索引 1~N 对应各 TLS 模块：

```
DTV 布局:
  dtv[-1].counter = DTV 数组长度
  dtv[0].counter  = 当前代数 (用于懒更新检测)
  dtv[1].pointer  = 模块 1 的 TLS 块地址
  dtv[2].pointer  = 模块 2 的 TLS 块地址
  ...
  dtv[N].pointer  = 模块 N 的 TLS 块地址
```

### 1.3 TLSDESC 相关结构

**源文件**: `sysdeps/aarch64/dl-tlsdesc.h:25-43`

```c
struct tlsdesc {
  ptrdiff_t (*entry)(struct tlsdesc *);  // 处理函数
  void *arg;                             // 参数
};

struct tlsdesc_dynamic_arg {
  tls_index tlsinfo;    // { ti_module, ti_offset }
  size_t gen_count;     // 创建时的 DTV 代数
};
```

### 1.4 struct pthread 中的 TLS 相关字段

**源文件**: `nptl/descr.h:148-183`

```c
struct pthread {
  union {
    // AArch64 使用 TLS_DTV_AT_TP，所以 header 不与 tcbhead_t 重叠
    struct {
      int multiple_threads;
      int gscope_flag;
    } header;
    void *__padding[24];   // 192 字节对齐填充
  };
  // ... 其他线程描述符字段 ...
};
```

---

## 二、AArch64 TLS 内存布局

### 2.1 关键宏定义

**源文件**: `sysdeps/aarch64/nptl/tls.h:37-56`

```c
#define TLS_DTV_AT_TP    1    // TLS 块在 TP 之后
#define TLS_TCB_AT_TP    0    // 不使用此模式

#define TLS_INIT_TCB_SIZE  sizeof(tcbhead_t)  // = 16
#define TLS_TCB_SIZE       sizeof(tcbhead_t)  // = 16
#define TLS_PRE_TCB_SIZE   sizeof(struct pthread)  // ≈ 2048+
```

### 2.2 内存布局图

```
低地址                                                    高地址
┌──────────────────┬─────────────┬────────┬────────┬─────────┬──────┐
│  struct pthread   │  tcbhead_t  │ TLS[1] │ TLS[2] │  ...    │ rseq │
│  (TLS_PRE_TCB)   │   (TCB)     │ 模块1  │ 模块2  │         │ 区域 │
│                  │  ┌───────┐  │ .tdata │ .tdata │         │      │
│  multiple_threads│  │ dtv ──┼──┼─→ DTV  │ .tbss  │         │      │
│  gscope_flag     │  │private│  │        │        │         │      │
│  ...             │  └───────┘  │        │        │         │      │
└──────────────────┴──────┬──────┴────────┴────────┴─────────┴──────┘
                          │
                    tpidr_el0 指向此处
                          │
                          ▼
              THREAD_SELF = tpidr_el0 - sizeof(struct pthread)
```

### 2.3 关键指针关系

```c
// 获取线程描述符（源文件: tls.h:85-86）
#define THREAD_SELF  ((struct pthread *)__builtin_thread_pointer() - 1)

// 获取 DTV（源文件: tls.h:81-82）
#define THREAD_DTV() (((tcbhead_t *)__builtin_thread_pointer())->dtv)

// 从 pthread 描述符得到 TCB 指针（传给 dl-tls 的参数）
// 源文件: nptl/descr.h:464-465
#define TLS_TPADJ(pd)  ((struct pthread *)((char *)(pd) + TLS_PRE_TCB_SIZE))

// 设置线程指针（源文件: tls.h:74-75）
#define TLS_INIT_TP(tcbp) \
  ({ __asm __volatile ("msr tpidr_el0, %0" : : "r" (tcbp)); true; })

// 安装 DTV（源文件: tls.h:60-61）
// 注意: dtvp 指向 dtv[-1]，所以 +1 跳过长度字段
#define INSTALL_DTV(tcbp, dtvp) \
  (((tcbhead_t *)(tcbp))->dtv = (dtvp) + 1)
```

---

## 三、静态 TLS 偏移计算

### 3.1 _dl_determine_tlsoffset（TLS_DTV_AT_TP 分支）

**源文件**: `elf/dl-tls.c:374-456`

```
_dl_determine_tlsoffset():
  offset = TLS_TCB_SIZE                     // 起始 = 16（紧接 TCB 之后）
  
  // 遍历所有启动时加载的模块
  for each link_map l with l_tls_blocksize > 0:
    offset = roundup(offset, l->l_tls_align)
    l->l_tls_offset = offset - firstbyte    // 记录此模块的偏移
    offset = offset + l->l_tls_blocksize - firstbyte
  
  // 追加 extra TLS（rseq 区域）
  offset = roundup(offset, extra_tls_align)
  _dl_extra_tls_set_offset(offset - TLS_TP_OFFSET)  // TLS_TP_OFFSET=0 on AArch64
  offset += extra_tls_size
  
  // 最终大小 = 已用 + 剩余空间（用于 dlopen 的 IE TLS）
  dl_tls_static_used = offset
  dl_tls_static_size = roundup(offset + dl_tls_static_surplus, TCB_ALIGNMENT)
```

### 3.2 静态 TLS 剩余空间 (surplus)

**源文件**: `elf/dl-tls.c:117-148`

```c
// 计算公式:
surplus = (nns - 1) * LIBC_IE_TLS    // 额外命名空间中 libc 的 IE TLS
        + nns * OTHER_IE_TLS          // 每个命名空间其他库的 IE TLS
        + opt_tls                     // 可选的优化空间 (默认 512)
        + LEGACY_TLS                  // 向后兼容余量

// 其中 LIBC_IE_TLS = 144, OTHER_IE_TLS = 144, DEFAULT_NNS = 4
```

剩余空间的用途：
- 允许 `dlopen` 加载带 IE TLS 的库（不需要调用 `__tls_get_addr`）
- TLSDESC 优化：将动态 TLS 转为静态 TLS

---

## 四、TLS 分配与初始化

### 4.1 分配存储空间

**源文件**: `elf/dl-tls.c:522-552`

```
_dl_allocate_tls_storage():
  size = _dl_tls_block_size_with_pre()
       = TLS_PRE_TCB_SIZE + dl_tls_static_size
  
  // 分配内存（含对齐余量）
  allocated = malloc(size + dl_tls_static_align + sizeof(void*))
  
  // 对齐后得到 TCB 指针
  result = _dl_tls_block_align(size, allocated)
  
  // 记录原始指针（用于后续 free）
  *tcb_to_pointer_to_free_location(result) = allocated
  
  // 分配 DTV
  result = allocate_dtv(result)
```

### 4.2 分配 DTV

**源文件**: `elf/dl-tls.c:465-493`

```
allocate_dtv(result):
  max_modid = dl_tls_max_dtv_idx
  dtv_length = max_modid + DTV_SURPLUS     // 预分配额外槽位
  dtv = calloc(dtv_length + 2, sizeof(dtv_t))  // +2: 长度 + 代数
  
  dtv[0].counter = dtv_length              // 存储长度
  INSTALL_DTV(result, dtv)                 // tcb->dtv = &dtv[1]
```

### 4.3 初始化 TLS 内容

**源文件**: `elf/dl-tls.c:622-726`

```
_dl_allocate_tls_init(result, main_thread):
  dtv = GET_DTV(result)
  
  // 检查 DTV 大小，必要时扩容
  if dtv[-1].counter < dl_tls_max_dtv_idx:
    dtv = _dl_resize_dtv(dtv, dl_tls_max_dtv_idx, result)
  
  // 遍历 slotinfo 列表
  for each module with TLS:
    // 默认标记为未分配
    dtv[map->l_tls_modid].pointer.val = TLS_DTV_UNALLOCATED
    
    if map->l_tls_offset == NO_TLS_OFFSET:
      continue  // 动态 TLS，保持 UNALLOCATED
    
    // 静态 TLS: 计算目标地址
    dest = (char *)result + map->l_tls_offset   // TLS_DTV_AT_TP
    
    // 设置 DTV 条目指向静态 TLS 块
    dtv[map->l_tls_modid].pointer.val = dest
    
    // 复制 .tdata 初始化数据 + 清零 .tbss
    memcpy(dest, map->l_tls_initimage, map->l_tls_initimage_size)
    memset(dest + initimage_size, 0, blocksize - initimage_size)
  
  // 设置 DTV 代数为当前最新
  dtv[0].counter = maxgen
```

### 4.4 组合接口

**源文件**: `elf/dl-tls.c:729-741`

```c
void *_dl_allocate_tls(void *mem) {
  // 如果 mem == NULL，先分配存储空间
  return _dl_allocate_tls_init(
    mem == NULL ? _dl_allocate_tls_storage() : allocate_dtv(mem),
    false  // 非主线程
  );
}
```

---

## 五、__tls_get_addr 慢路径

### 5.1 调用时机

当线程通过 GD/LD 模型或 TLSDESC 动态路径访问 TLS 变量时，
最终会调用 `__tls_get_addr`。

### 5.2 实现

**源文件**: `elf/dl-tls.c:1090-1142`

```
__tls_get_addr(tls_index *ti):
  dtv = THREAD_DTV()
  gen = atomic_load_relaxed(&dl_tls_generation)
  
  // ① 代数检查: DTV 是否过期？
  if dtv[0].counter != gen:
    if _dl_tls_allocate_active() && ti->ti_module < initial_modid_limit:
      // 重入保护: 在 TLS 分配过程中被再次调用
      // 初始模块的 TLS 不会变化，可以安全跳过更新
      pass
    else:
      // 需要更新 DTV
      gen = atomic_load_acquire(&dl_tls_generation)
      return update_get_addr(ti, gen)
  
  // ② 检查目标模块是否已分配
  p = dtv[ti->ti_module].pointer.val
  if p == TLS_DTV_UNALLOCATED:
    return tls_get_addr_tail(ti, dtv, NULL)  // 分配并初始化
  
  // ③ 快速返回
  return p + ti->ti_offset
```

### 5.3 DTV 更新机制

**源文件**: `elf/dl-tls.c:834-980`

```
_dl_update_slotinfo(req_modid, new_gen):
  dtv = THREAD_DTV()
  
  // 遍历所有 slotinfo 条目
  for each slotinfo entry with gen <= new_gen:
    if entry was updated (gen > old_gen):
      // 可能需要扩容 DTV
      if modid >= dtv[-1].counter:
        dtv = _dl_resize_dtv(dtv, modid, tcb)
      
      // 释放旧的动态 TLS 块
      free(dtv[modid].pointer.to_free)
      
      if entry.map != NULL:
        // 模块存在: 标记为待分配
        dtv[modid].pointer.val = TLS_DTV_UNALLOCATED
      else:
        // 模块已卸载: 清空
        dtv[modid].pointer.val = NULL
  
  // 更新线程的 DTV 代数
  dtv[0].counter = new_gen
```

### 5.4 代数计数器工作原理

```
时间线:
  t=0  启动完成  dl_tls_generation = 3 (3个初始TLS模块)
       所有线程 dtv[0].counter = 3
  
  t=1  dlopen("libfoo.so")  →  dl_tls_generation = 4
       新模块 modid=4
  
  t=2  线程 A 访问 libfoo 的 TLS 变量
       → __tls_get_addr({ti_module=4, ti_offset=...})
       → dtv[0].counter (3) != dl_tls_generation (4)
       → _dl_update_slotinfo(): 更新 DTV, dtv[0].counter = 4
       → dtv[4] = TLS_DTV_UNALLOCATED
       → tls_get_addr_tail(): 分配 TLS 块, dtv[4].val = 新地址
       → 返回 dtv[4].val + ti_offset
  
  t=3  线程 A 再次访问
       → dtv[0].counter (4) == dl_tls_generation (4) ✓
       → dtv[4].val != TLS_DTV_UNALLOCATED ✓
       → 直接返回（快速路径）
```

---

## 六、线程创建时的 TLS 设置

### 6.1 allocate_stack 中的 TLS

**源文件**: `nptl/allocatestack.c:356-658`

```
allocate_stack(attr, &pd, &stack, &stacksize):
  tls_static_size_for_stack = __nptl_tls_static_size_for_stack()
  //   = roundup(dl_tls_static_size, dl_tls_static_align)
  
  // 分配栈内存 (mmap)
  mem = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, ...)
  
  // 在栈顶端放置 struct pthread（TLS_DTV_AT_TP 分支）
  pd = (struct pthread *)((((uintptr_t)mem + size
                             - tls_static_size_for_stack)
                            & ~tls_static_align_m1)
                           - TLS_PRE_TCB_SIZE)
  
  // 分配 DTV 并初始化 TLS
  _dl_allocate_tls(TLS_TPADJ(pd))
  // TLS_TPADJ(pd) = (char *)pd + TLS_PRE_TCB_SIZE = TCB 指针
```

### 6.2 栈+TLS 内存布局

```
mmap 区域（单个连续映射）:
┌──────────┬───────────────────────────┬────────────────────────────┐
│ guard    │          栈空间            │       TLS 区域              │
│ pages    │   (向下增长 ↓)             │                            │
│          │                           │ pthread │ TCB │ TLS 块... │
│          │          ← SP             │         │←TP  │           │
└──────────┴───────────────────────────┴────────────────────────────┘
低地址                                                          高地址
```

### 6.3 缓存栈的 TLS 重用

**源文件**: `nptl/allocatestack.c:141-149`

当复用缓存栈时，需要重新初始化 TLS：

```c
// 清除旧 DTV 的动态 TLS 分配
dtv_t *dtv = GET_DTV(TLS_TPADJ(result));
for (size_t cnt = 0; cnt < dtv[-1].counter; ++cnt)
  free(dtv[1 + cnt].pointer.to_free);
memset(dtv, '\0', (dtv[-1].counter + 1) * sizeof(dtv_t));

// 重新初始化 TLS 内容（复制 .tdata，标记动态为 UNALLOCATED）
_dl_allocate_tls_init(TLS_TPADJ(result), false);
```

### 6.4 clone 时设置 tpidr_el0

**源文件**: `sysdeps/aarch64/nptl/tls.h:78`

```c
// 传给 clone 的线程指针值
#define TLS_DEFINE_INIT_TP(tp, pd) void *tp = (pd) + 1
```

`(pd) + 1` 即 `pd + sizeof(struct pthread)` = TCB 地址。
内核的 `clone` 系统调用会将此值写入新线程的 `tpidr_el0`。

---

## 七、静态链接的 TLS 初始化

### 7.1 __libc_setup_tls

**源文件**: `csu/libc-tls.c:100-165`

静态链接程序没有动态链接器，TLS 初始化由 `__libc_setup_tls` 完成：

```
__libc_setup_tls():
  // ① 扫描 PT_TLS 程序头
  for each phdr:
    if phdr->p_type == PT_TLS:
      记录 memsz, filesz, initimage, align
      设置 main_map 的 TLS 字段
      init_slotinfo(main_map)    // modid = 1
  
  // ② 计算偏移和大小
  _dl_tls_static_surplus_init(0)  // 无审计模块
  _dl_determine_tlsoffset()
  
  // ③ 分配 TCB + TLS 块
  size = _dl_tls_block_size_with_pre()
  allocated = _dl_early_allocate(size + dl_tls_static_align)
  tcbp = _dl_tls_block_align(size, allocated)
  
  // ④ 初始化 DTV
  _dl_static_dtv[0].counter = array_length - 2  // DTV 长度
  INSTALL_DTV(tcbp, _dl_static_dtv)
  
  // ⑤ 设置线程指针
  call_tls_init_tp(tcbp)    // → msr tpidr_el0, tcbp
  
  // ⑥ 初始化 TLS 内容
  _dl_allocate_tls_init(tcbp, true)
```

---

## 八、TLS 访问模型

### 8.1 四种模型对比

| 模型 | 全称 | 适用场景 | AArch64 实现 |
|------|------|----------|-------------|
| **LE** | Local Exec | 主程序内的 `_Thread_local` | 直接 `tpidr_el0 + 常量偏移` |
| **IE** | Initial Exec | 启动时加载的 .so 的 TLS | GOT 中存储偏移，`tpidr_el0 + GOT[n]` |
| **GD** | Global Dynamic | 任何 .so 的 TLS | 通过 `__tls_get_addr` |
| **TLSDESC** | TLS Descriptor | GD 的优化替代 | 描述符回调（见 AArch64 重定位文档） |

### 8.2 LE 模型（最快）

```asm
// _Thread_local int x;  （主程序中）
mrs   x0, tpidr_el0          // 读线程指针
ldr   w1, [x0, #offset_of_x] // 直接偏移访问
```

编译时就确定偏移，无需任何运行时开销。

### 8.3 IE 模型

```asm
// 编译器生成:
adrp  x0, :gottprel:var         // GOT 条目的页地址
ldr   x0, [x0, :gottprel_lo12:var]  // 加载 TP-相对偏移
mrs   x1, tpidr_el0
ldr   w2, [x1, x0]             // 访问 TLS 变量
```

需要一次 GOT 查找，但偏移在加载时已确定，无需函数调用。

### 8.4 TLSDESC 模型（AArch64 默认）

```asm
// 编译器生成:
adrp  x0, :tlsdesc:var
ldr   x1, [x0, :tlsdesc_lo12:var]    // td->entry
add   x0, x0, :tlsdesc_lo12:var      // &td
.tlsdesccall var
blr   x1                              // 调用 td->entry(td)
mrs   x2, tpidr_el0
add   x2, x2, x0                     // TLS 变量地址
```

三种运行时路径：
- **静态 TLS**: `_dl_tlsdesc_return` — 1 条 `ldr` + `ret` (见 `dl-tlsdesc.S:76-81`)
- **未定义弱符号**: `_dl_tlsdesc_undefweak` — 3 条指令 (见 `dl-tlsdesc.S:97-108`)
- **动态 TLS**: `_dl_tlsdesc_dynamic` — 快速路径 ~10 条，慢路径调用 `__tls_get_addr` (见 `dl-tlsdesc.S:143-239`)

---

## 九、DTV 扩容与释放

### 9.1 DTV 扩容

**源文件**: `elf/dl-tls.c:560-610`

```
_dl_resize_dtv(dtv, max_modid, tcb):
  newsize = max_modid + DTV_SURPLUS
  oldsize = dtv[-1].counter
  
  if dtv == dl_initial_dtv:
    // 初始 DTV（不能 realloc，因为可能不是 malloc 分配的）
    newp = malloc((2 + newsize) * sizeof(dtv_t))
    memcpy(newp, &dtv[-1], (2 + oldsize) * sizeof(dtv_t))
  else:
    newp = realloc(&dtv[-1], (2 + newsize) * sizeof(dtv_t))
  
  newp[0].counter = newsize            // 更新长度
  memset(newp + 2 + oldsize, 0, ...)   // 清零新增部分
  return &newp[1]                      // 返回 dtv[0] 位置
```

### 9.2 TLS 释放

**源文件**: `elf/dl-tls.c:744-772`

```
_dl_deallocate_tls(tcb, dealloc_tcb):
  dtv = GET_DTV(tcb)
  
  // 释放所有动态分配的 TLS 块
  for cnt = 0..dtv[-1].counter:
    free(dtv[1 + cnt].pointer.to_free)
  
  // 释放 DTV 本身（初始 DTV 除外）
  if dtv != dl_initial_dtv:
    free(dtv - 1)
  
  // 可选：释放 TCB 存储
  if dealloc_tcb:
    free(*tcb_to_pointer_to_free_location(tcb))
```

---

## 十、dlopen/dlclose 与 TLS 模块管理

### 10.1 模块 ID 分配

**源文件**: `elf/dl-tls.c:160-229`

```
_dl_assign_tls_modid(link_map *l):
  if dl_tls_dtv_gaps:
    // 有空隙：遍历 slotinfo 找到空闲槽
    找到 map == NULL 的槽 → result = 该索引
    dl_tls_dtv_gaps = false (如果没有更多空隙)
  else:
    // 无空隙：追加新槽
    result = ++dl_tls_max_dtv_idx
    // 必要时扩展 slotinfo 列表
  
  l->l_tls_modid = result
  // 递增全局代数
  dl_tls_generation++
  slotinfo[result].gen = dl_tls_generation
  slotinfo[result].map = l
```

### 10.2 dlclose 回收

**源文件**: `elf/dl-close.c:45-110`

```
remove_slotinfo(modid):
  slotinfo[modid].map = NULL
  dl_tls_generation++
  slotinfo[modid].gen = dl_tls_generation
  dl_tls_dtv_gaps = true
  
  // 如果是尾部条目，可以缩减 max_dtv_idx
  if modid == dl_tls_max_dtv_idx:
    从尾部向前扫描，跳过所有 map==NULL 的条目
    dl_tls_max_dtv_idx = 最后一个有效条目的索引
```

---

## 十一、汇编层偏移量定义

**源文件**: `sysdeps/aarch64/tlsdesc.sym`

```c
// 这些偏移量由 awk 从 .sym 文件自动生成 .h 供汇编使用
TLSDESC_ARG        = offsetof(struct tlsdesc, arg)           // = 8
TCBHEAD_DTV        = offsetof(tcbhead_t, dtv)                // = 0
DTV_COUNTER        = offsetof(dtv_t, counter)                // = 0
TLSDESC_GEN_COUNT  = offsetof(struct tlsdesc_dynamic_arg, gen_count)
TLSDESC_MODID      = offsetof(struct tlsdesc_dynamic_arg, tlsinfo.ti_module)
TLSDESC_MODOFF     = offsetof(struct tlsdesc_dynamic_arg, tlsinfo.ti_offset)
TLS_DTV_UNALLOCATED = (void *) -1
```

---

## 十二、完整 TLS 访问时序

```
用户代码访问 _Thread_local 变量 (TLSDESC 模型):
    │
    ▼
blr  td->entry ──→ _dl_tlsdesc_return? ──→ ldr x0,[x0,#8]; ret (1条指令)
    │                      │
    │                      ├─ _dl_tlsdesc_dynamic? ──→ 快速路径?
    │                      │        │                      │
    │                      │        │                 mrs tpidr_el0
    │                      │        │                 load DTV
    │                      │        │                 check gen_count
    │                      │        │                 load dtv[modid]
    │                      │        │                 check != UNALLOCATED
    │                      │        │                 return val+offset-TP  ←── 正常返回
    │                      │        │
    │                      │        └─ 慢路径: save regs → __tls_get_addr
    │                      │                                    │
    │                      │                    ┌───────────────┘
    │                      │                    ▼
    │                      │           dtv[0].counter == gen? ──→ dtv[mod].val?
    │                      │                 │ no                   │ UNALLOCATED
    │                      │                 ▼                      ▼
    │                      │        _dl_update_slotinfo()    allocate_and_init()
    │                      │        更新 DTV 到最新代数       malloc + memcpy .tdata
    │                      │                                     │
    │                      │                                     ▼
    │                      └────────────────────── 返回 TLS 变量地址
    ▼
mrs x2, tpidr_el0
add x2, x2, x0    // x2 = TLS 变量绝对地址
```

---

## 十三、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `sysdeps/aarch64/nptl/tls.h:37-38` | `TLS_DTV_AT_TP=1, TLS_TCB_AT_TP=0` |
| `sysdeps/aarch64/nptl/tls.h:43-47` | `tcbhead_t` 定义 |
| `sysdeps/aarch64/nptl/tls.h:50-56` | `TLS_TCB_SIZE, TLS_PRE_TCB_SIZE` |
| `sysdeps/aarch64/nptl/tls.h:60-61` | `INSTALL_DTV` 宏 |
| `sysdeps/aarch64/nptl/tls.h:74-75` | `TLS_INIT_TP` (msr tpidr_el0) |
| `sysdeps/aarch64/nptl/tls.h:78` | `TLS_DEFINE_INIT_TP` (clone 参数) |
| `sysdeps/aarch64/nptl/tls.h:81-82` | `THREAD_DTV()` |
| `sysdeps/aarch64/nptl/tls.h:85-86` | `THREAD_SELF` |
| `sysdeps/generic/dl-dtv.h:22-36` | `dtv_t`, `TLS_DTV_UNALLOCATED` |
| `sysdeps/generic/dl-tls.h:24-28` | `tls_index` 结构 |
| `sysdeps/generic/dl-tls.h:34-37` | `TLS_DTV_OFFSET=0, TLS_TP_OFFSET=0` |
| `sysdeps/aarch64/dl-tlsdesc.h:25-29` | `struct tlsdesc` |
| `sysdeps/aarch64/dl-tlsdesc.h:39-43` | `struct tlsdesc_dynamic_arg` |
| `sysdeps/aarch64/tlsdesc.sym` | 汇编偏移量定义 |
| `nptl/descr.h:148-183` | `struct pthread` 头部（TLS 相关） |
| `nptl/descr.h:464-465` | `TLS_TPADJ` 宏 |
| `elf/dl-tls.c:132-148` | `_dl_tls_static_surplus_init` |
| `elf/dl-tls.c:160-229` | `_dl_assign_tls_modid` |
| `elf/dl-tls.c:257-463` | `_dl_determine_tlsoffset`（含 AArch64 分支 374-456） |
| `elf/dl-tls.c:465-493` | `allocate_dtv` |
| `elf/dl-tls.c:522-552` | `_dl_allocate_tls_storage` |
| `elf/dl-tls.c:560-610` | `_dl_resize_dtv` |
| `elf/dl-tls.c:622-726` | `_dl_allocate_tls_init` |
| `elf/dl-tls.c:729-741` | `_dl_allocate_tls` |
| `elf/dl-tls.c:744-772` | `_dl_deallocate_tls` |
| `elf/dl-tls.c:834-981` | `_dl_update_slotinfo` |
| `elf/dl-tls.c:1090-1142` | `__tls_get_addr` |
| `elf/dl-tls.c:1148-1189` | `_dl_tls_get_addr_soft` |
| `csu/libc-tls.c:39-71` | 静态链接 TLS 全局变量 |
| `csu/libc-tls.c:100-165` | `__libc_setup_tls` |
| `nptl/allocatestack.c:356-658` | `allocate_stack` (TLS 分配路径) |
| `nptl/allocatestack.c:141-149` | 缓存栈 TLS 重初始化 |
| `elf/dl-close.c:45-111` | `remove_slotinfo` (dlclose TLS 回收) |
| `sysdeps/aarch64/dl-tlsdesc.S:76-81` | `_dl_tlsdesc_return` |
| `sysdeps/aarch64/dl-tlsdesc.S:97-108` | `_dl_tlsdesc_undefweak` |
| `sysdeps/aarch64/dl-tlsdesc.S:143-239` | `_dl_tlsdesc_dynamic` |
