# dl-tls 与动态链接器辅助模块分析

> 基于 glibc 2.43.9000 源码分析  
> 涵盖 TLS 管理（dl-tls）、link_map 创建（dl-object）、依赖排序（dl-sort-maps）、地址查找（dl-find_object）

---

## 目录

1. [概述](#1-概述)
2. [TLS 核心概念](#2-tls-核心概念)
3. [TLS 模块注册与偏移计算](#3-tls-模块注册与偏移计算)
4. [DTV 分配与初始化](#4-dtv-分配与初始化)
5. [__tls_get_addr 运行时解析](#5-__tls_get_addr-运行时解析)
6. [TLS 代际更新机制](#6-tls-代际更新机制)
7. [静态 TLS 初始化](#7-静态-tls-初始化)
8. [link_map 对象创建——dl-object](#8-link_map-对象创建dl-object)
9. [依赖排序——dl-sort-maps](#9-依赖排序dl-sort-maps)
10. [快速地址查找——dl-find_object](#10-快速地址查找dl-find_object)
11. [四模块协作关系](#11-四模块协作关系)
12. [源文件速查表](#12-源文件速查表)

---

## 1. 概述

### 1.1 模块职责

```
┌─────────────────────────────────────────────────────────────────┐
│                  动态链接器辅助模块                               │
├──────────────────┬──────────────────────────────────────────────┤
│  dl-tls.c        │ TLS 全生命周期管理:                          │
│  (1337 行)       │ 模块 ID 分配、静态偏移计算、DTV 分配/调整、  │
│                  │ __tls_get_addr 解析、代际同步、线程间更新     │
├──────────────────┼──────────────────────────────────────────────┤
│  dl-object.c     │ link_map 对象工厂:                           │
│  (263 行)        │ 分配初始化 link_map、设置名称/作用域/来源、  │
│                  │ 加入命名空间链表                              │
├──────────────────┼──────────────────────────────────────────────┤
│  dl-sort-maps.c  │ 依赖拓扑排序:                                │
│  (310 行)        │ 确定初始化/析构顺序、支持新旧两种算法、      │
│                  │ 处理循环依赖                                  │
├──────────────────┼──────────────────────────────────────────────┤
│  dl-find_object.c│ 快速地址→对象查找:                           │
│  (881 行)        │ 按地址排序的分段数组、O(log n) 二分搜索、    │
│                  │ 异步信号安全的 STM 并发控制                   │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## 2. TLS 核心概念

### 2.1 静态 TLS vs 动态 TLS

| 类别 | 静态 TLS | 动态 TLS |
|------|----------|----------|
| 分配时机 | 线程创建时一次性分配 | 首次访问时延迟分配 |
| 适用对象 | 程序启动时已加载的模块 | dlopen 动态加载的模块 |
| 访问模型 | Initial-Exec (IE) / Local-Exec (LE) | General-Dynamic (GD) / Local-Dynamic (LD) |
| 性能 | 直接偏移访问，无函数调用 | 需要 `__tls_get_addr()` |
| 偏移计算 | `_dl_determine_tlsoffset()` 编译时确定 | 运行时按需计算 |

### 2.2 线程内存布局

**x86_64 (TLS_TCB_AT_TP=1)**：TP 指向 TCB，静态 TLS 在 TP 之前

```
低地址 ←────────────────────────────────────→ 高地址

  ┌──────────┬──────────┬──────────┬─────────┐
  │ TLS 模块3 │ TLS 模块2 │ TLS 模块1 │  TCB    │
  │ (最晚加载) │          │ (最先加载) │ (pthread)│
  └──────────┴──────────┴──────────┴─────────┘
                                     ↑
                                  TP (%fs)
  访问: TP - offset
```

**AArch64 (TLS_DTV_AT_TP=1)**：TP 指向 TLS 块起始

```
低地址 ←────────────────────────────────────→ 高地址

  ┌─────────┬──────────┬──────────┬──────────┐
  │  TCB    │ TLS 模块1 │ TLS 模块2 │ TLS 模块3 │
  │ (pthread)│ (最先加载) │          │ (最晚加载) │
  └─────────┴──────────┴──────────┴──────────┘
  ↑
  TP (TPIDR_EL0)
  访问: TP + TLS_PRE_TCB_SIZE + offset
```

### 2.3 DTV（Dynamic Thread Vector）

`sysdeps/generic/dl-dtv.h:22-33`：

```c
struct dtv_pointer {
  void *val;       // 指向 TLS 块，或 TLS_DTV_UNALLOCATED (-1)
  void *to_free;   // 原始 malloc 指针（用于释放）
};

typedef union dtv {
  size_t counter;            // 用于 dtv[-1] 和 dtv[0]
  struct dtv_pointer pointer; // dtv[1..N] 模块 TLS 指针
} dtv_t;
```

```
DTV 布局:
  dtv[-1].counter = 已分配的 DTV 槽数（容量）
  dtv[0].counter  = 当前代际号（generation）
  dtv[1].pointer  = 模块 1 的 TLS 块
  dtv[2].pointer  = 模块 2 的 TLS 块
  ...
  dtv[N].pointer  = 模块 N 的 TLS 块
```

### 2.4 Slotinfo 链表

全局 TLS 模块注册表，定义在 `sysdeps/generic/ldsodefs.h`：

```c
struct dtv_slotinfo {
  size_t gen;              // 该模块注册时的代际号
  struct link_map *map;    // 模块的 link_map
};

struct dtv_slotinfo_list {
  size_t len;                          // 本段槽位数
  struct dtv_slotinfo_list *next;      // 下一段
  struct dtv_slotinfo slotinfo[];      // 柔性数组
};
```

---

## 3. TLS 模块注册与偏移计算

### 3.1 模块 ID 分配

`_dl_assign_tls_modid()`（`dl-tls.c:160-230`）：

```
_dl_assign_tls_modid(link_map *l):
  │
  ├─ 有空洞 (dl_tls_dtv_gaps=true):
  │   遍历 slotinfo 链表，找到 map==NULL 的空槽
  │   从 dl_tls_static_nelem+1 开始搜索
  │   dl-tls.c:165-212
  │
  ├─ 无空洞:
  │   result = dl_tls_max_dtv_idx + 1
  │   原子更新 dl_tls_max_dtv_idx
  │   dl-tls.c:213-221
  │
  └─ l->l_tls_modid = result
      dl-tls.c:223
```

模块 ID 从 1 开始（dtv[0] 用于代际计数器），dlclose 时会产生空洞。

### 3.2 静态 TLS 偏移计算

`_dl_determine_tlsoffset()`（`dl-tls.c:256-463`）：

```
_dl_determine_tlsoffset():
  │
  ├─ TLS_TCB_AT_TP 模式（x86_64）:
  │   从 offset=0 开始，向负方向增长
  │   每个模块: offset += blocksize + alignment_padding
  │   l->l_tls_offset = offset（正值，表示 TP-offset 距离）
  │   dl-tls.c:291-373
  │
  ├─ TLS_DTV_AT_TP 模式（AArch64）:
  │   从 TCB 之后开始，向正方向增长
  │   每个模块: offset = roundup(current, align) + firstbyte
  │   l->l_tls_offset = offset
  │   dl-tls.c:374-456
  │
  └─ 间隙复用优化:
      如果两个模块之间的对齐间隙足够大
      下一个小模块可以嵌入间隙中
      dl-tls.c:306-331 / dl-tls.c:389-414
```

间隙复用示意:

```
TCB_AT_TP 模式下:
  ┌──────┬────┬──────────┬─────────┐
  │mod 3 │gap │  mod 2   │  mod 1  │ TCB
  └──────┴────┴──────────┴─────────┘
             ↑
         如果 mod4 小到能放入 gap
         则 mod4 直接嵌入此处
```

---

## 4. DTV 分配与初始化

### 4.1 allocate_dtv()

`dl-tls.c:465-493`：

```c
static void *allocate_dtv(void *result)
{
  size_t max_modid = atomic_load_relaxed(&GL(dl_tls_max_dtv_idx));
  size_t dtv_length = max_modid + DTV_SURPLUS; // 预留余量
  dtv = calloc(dtv_length + 2, sizeof(dtv_t)); // +2 for dtv[-1] 和 dtv[0]
  dtv[0].counter = dtv_length;  // 容量
  // dtv[1].counter == 0 → 代际为 0，待初始化
  INSTALL_DTV(result, dtv);     // 安装到 TCB
}
```

### 4.2 _dl_allocate_tls_storage()

`dl-tls.c:522-552`——分配 TCB + 静态 TLS 存储：

```
_dl_allocate_tls_storage():
  │
  ├─ 计算总大小 = TLS 块 + 对齐 + 指针（用于 free）
  ├─ malloc() 分配
  ├─ _dl_tls_block_align() 对齐
  ├─ 记录原始指针（用于后续释放）
  │   dl-tls.c:542
  └─ allocate_dtv() 安装 DTV
      dl-tls.c:544
```

### 4.3 _dl_allocate_tls_init()

`dl-tls.c:622-727`——初始化静态 TLS 数据：

```
_dl_allocate_tls_init(result, main_thread):
  │
  ├─ 获取 dl_load_tls_lock
  │
  ├─ 如果 DTV 太小 → _dl_resize_dtv()
  │   dl-tls.c:638-645
  │
  ├─ 遍历 slotinfo 链表中所有模块:
  │   │
  │   ├─ 跳过空槽（map==NULL）
  │   │
  │   ├─ 设置 dtv[modid] = TLS_DTV_UNALLOCATED（动态 TLS）
  │   │   dl-tls.c:674-675
  │   │
  │   ├─ 有静态偏移的模块:
  │   │   dest = TP ± l_tls_offset（取决于架构）
  │   │   memcpy(dest, l_tls_initimage, initimage_size)
  │   │   memset(dest+initimage_size, 0, blocksize-initimage_size)
  │   │   dtv[modid].pointer.val = dest
  │   │   dl-tls.c:683-710
  │   │
  │   └─ 记录最大代际号
  │       dl-tls.c:672
  │
  ├─ 释放锁
  │
  └─ dtv[0].counter = maxgen（当前代际）
      dl-tls.c:723
```

### 4.4 _dl_allocate_tls() 总入口

`dl-tls.c:729-741`：

```c
void *_dl_allocate_tls(void *mem)
{
  // mem==NULL → 分配新 TCB + DTV
  // mem!=NULL → 仅分配 DTV（用户提供 TCB 内存）
  return _dl_allocate_tls_init(
    mem == NULL ? _dl_allocate_tls_storage() : allocate_dtv(mem),
    false);
}
```

### 4.5 _dl_resize_dtv()

`dl-tls.c:560-609`——扩容 DTV：

```
_dl_resize_dtv(dtv, max_modid, tcb):
  │
  ├─ newsize = max_modid + DTV_SURPLUS
  │
  ├─ 初始 DTV（不可 free）:
  │   malloc 新 DTV + memcpy 旧内容
  │   dl-tls.c:574-591
  │
  ├─ 非初始 DTV:
  │   realloc 扩容
  │   dl-tls.c:593-599
  │
  ├─ 更新容量: newp[0].counter = newsize
  │
  └─ memset 清零新增部分
      dl-tls.c:604-606
```

### 4.6 _dl_deallocate_tls()

`dl-tls.c:744-772`——线程退出时释放 TLS：

```
_dl_deallocate_tls(tcb, dealloc_tcb):
  │
  ├─ 遍历 dtv[1..max]
  │   释放动态分配的 TLS 块 (to_free 指针)
  │
  ├─ 释放 DTV 数组本身
  │
  └─ 如果 dealloc_tcb → 释放 TCB 存储
```

---

## 5. __tls_get_addr 运行时解析

### 5.1 核心流程

`__tls_get_addr()`（`dl-tls.c:1090-1142`）——TLS 访问的运行时入口：

```
__tls_get_addr(tls_index *ti):
  │  ti->ti_module = TLS 模块 ID
  │  ti->ti_offset = 模块内偏移
  │
  ├─ dtv = THREAD_DTV()
  │
  ├─ 检查代际:
  │   gen = GL(dl_tls_generation)  // relaxed load
  │   if (dtv[0].counter != gen):
  │     │
  │     ├─ 重入安全检查:
  │     │   如果正在 TLS 分配中，且是初始模块
  │     │   → 直接使用旧 DTV（初始模块不变）
  │     │   dl-tls.c:1102-1115
  │     │
  │     └─ 正常更新:
  │         gen = GL(dl_tls_generation)  // acquire load
  │         → update_get_addr(ti, gen)
  │         dl-tls.c:1118-1126
  │
  ├─ 检查是否已分配:
  │   p = dtv[ti->ti_module].pointer.val
  │   if (p == TLS_DTV_UNALLOCATED):
  │     → tls_get_addr_tail(ti, dtv, NULL)  // 延迟分配
  │     dl-tls.c:1132-1138
  │
  └─ 返回 tls_get_addr_adjust(p, ti)
      → 计算 (uintptr_t)p + ti_offset + TLS_DTV_OFFSET
      dl-tls.c:1141（实际调整在 dl-tls.c:985-994 的内联函数中）
```

### 5.2 延迟分配

当 `dtv[modid]` 为 `TLS_DTV_UNALLOCATED`（-1）时：

```
tls_get_addr_tail(ti, dtv, saved):
  │
  ├─ _dl_update_slotinfo() 同步代际
  ├─ allocate_dtv_entry() 分配模块 TLS 块
  │   → malloc(blocksize + alignment)
  │   → memcpy 初始化映像 + memset BSS
  │   dl-tls.c:779-809
  └─ 返回块内偏移地址
```

### 5.3 _dl_tls_get_addr_soft()

`dl-tls.c:1148-1190`——非分配式查询：

```
_dl_tls_get_addr_soft(link_map *l):
  │
  ├─ 无 TLS 段 → 返回 NULL
  ├─ DTV 代际不匹配:
  │   ├─ 模块 ID ≥ DTV 容量 → NULL
  │   └─ slotinfo 代际 > DTV 代际 → NULL
  ├─ dtv[modid] == TLS_DTV_UNALLOCATED → NULL
  └─ 返回 dtv[modid].pointer.val
```

不触发分配，不加锁——用于调试和安全检查场景。

---

## 6. TLS 代际更新机制

### 6.1 代际模型

```
全局代际计数器: GL(dl_tls_generation)
  dlopen 加载 TLS 模块 → generation++

每线程代际: dtv[0].counter
  初始 = 创建时的全局代际
  更新 = __tls_get_addr 触发时追赶全局代际

┌──────────────────────────────────────────────────────┐
│  时序:                                                │
│                                                       │
│  线程 A 创建 (gen=5)    dlopen 加载 mod7 (gen→6)     │
│  dtv[0].counter=5       slotinfo[7].gen=6            │
│       │                       │                       │
│       │   线程 A 访问 mod7    │                       │
│       ├── __tls_get_addr() ──→│                       │
│       │   dtv[0].counter!=6   │                       │
│       │   → update_get_addr() │                       │
│       │   → 扩容 DTV          │                       │
│       │   → 分配 mod7 TLS 块  │                       │
│       │   dtv[0].counter=6    │                       │
│       │   返回 TLS 地址       │                       │
│       │                       │                       │
└──────────────────────────────────────────────────────┘
```

### 6.2 _dl_add_to_slotinfo()

`dl-tls.c:1237-1302`——dlopen 时注册新 TLS 模块：

```
_dl_add_to_slotinfo(link_map *l, bool do_add):
  │
  ├─ 无 TLS 或已注册 → 返回 false
  │
  ├─ 在 slotinfo 链表中找到对应槽位
  │   dl-tls.c:1250-1262
  │
  ├─ 槽位不存在 → 分配新的 slotinfo_list 段
  │   长度 = TLS_SLOTINFO_SURPLUS
  │   atomic_store_release 链接到链表
  │   dl-tls.c:1264-1289
  │
  └─ 写入槽位:
      slotinfo[idx].map = l
      slotinfo[idx].gen = dl_tls_generation + 1  // 比当前代际大 1
      dl-tls.c:1295-1298
```

### 6.3 _dl_update_slotinfo()

`dl-tls.c:835-980`——线程追赶全局代际：

```
_dl_update_slotinfo(modid, new_gen):
  │
  ├─ 遍历 slotinfo 链表
  │   找到 gen > dtv[0].counter 的新模块
  │
  ├─ 对每个新模块:
  │   ├─ 如果 DTV 太小 → _dl_resize_dtv()
  │   ├─ 静态 TLS 模块 → 分配并初始化
  │   └─ 动态 TLS 模块 → 标记为 TLS_DTV_UNALLOCATED
  │
  └─ dtv[0].counter = new_gen
```

---

## 7. 静态 TLS 初始化

### 7.1 _dl_init_static_tls()

`dl-tls.c:1322-1337`——dlopen 时为所有已有线程初始化静态 TLS：

```
_dl_init_static_tls(link_map *map):
  │
  ├─ 获取 dl_stack_cache_lock
  │
  ├─ 遍历 dl_stack_used 链表（系统分配栈的线程）
  │   init_one_static_tls(curp, map)
  │
  ├─ 遍历 dl_stack_user 链表（用户分配栈的线程）
  │   init_one_static_tls(curp, map)
  │
  └─ 释放锁
```

`init_one_static_tls()`（`dl-tls.c:1305-1318`）：

```c
static void init_one_static_tls(struct pthread *curp, struct link_map *map)
{
#if TLS_TCB_AT_TP
  void *dest = (char *) curp - map->l_tls_offset;
#elif TLS_DTV_AT_TP
  void *dest = (char *) curp + map->l_tls_offset + TLS_PRE_TCB_SIZE;
#endif
  memcpy(dest, map->l_tls_initimage, map->l_tls_initimage_size);
  memset(dest + initimage_size, 0, blocksize - initimage_size);
}
```

---

## 8. link_map 对象创建——dl-object

### 8.1 _dl_new_object()

`dl-object.c:54-263`——link_map 工厂函数：

```
_dl_new_object(realname, libname, type, loader, mode, nsid):
  │
  ├─ 1. 一次性分配:
  │     sizeof(link_map) + audit_space
  │     + sizeof(link_map*)      // l_symbolic_searchlist
  │     + sizeof(libname_list)   // l_libname
  │     + strlen(libname)+1      // 库名字符串
  │     → calloc（所有字段初始为零）
  │     dl-object.c:92-96
  │
  ├─ 2. 基本字段:
  │     l_real = self                    :98
  │     l_symbolic_searchlist.r_list     :99-100
  │     l_libname = newname              :102-104
  │     newname->name = libname 副本     :104
  │     newname->dont_free = 1           :105
  │
  ├─ 3. l_name 设置:
  │     非空 realname → l_name = realname  :119-123
  │     空字符串 → 指向 newname->name      :124-125
  │
  ├─ 4. 类型与状态:
  │     l_type = type                    :127
  │     l_used = 1                       :128
  │     l_loader = loader                :131
  │     l_tls_offset = NO_TLS_OFFSET     :133
  │     l_ns = nsid                      :137
  │
  ├─ 5. 作用域设置:
  │     l_scope = l_scope_mem            :146-150
  │     全局作用域: ns[nsid]._ns_loaded  :155-158
  │     加载者作用域 + RTLD_DEEPBIND     :159-177
  │     l_local_scope[0] = &l_searchlist :179
  │
  └─ 6. 来源路径 (l_origin):
        从 realname 提取目录部分
        处理相对路径 → 拼接 cwd
        dl-object.c:181-259
```

### 8.2 _dl_add_to_namespace_list()

`dl-object.c:28-51`——将 link_map 加入命名空间链表：

```
_dl_add_to_namespace_list(new, nsid):
  │
  ├─ 获取 dl_load_write_lock
  │
  ├─ 遍历到链表尾部
  │   new->l_prev = tail
  │   tail->l_next = new
  │   dl-object.c:35-42
  │
  ├─ 或者设为链表头（首个对象）
  │   dl-object.c:44-45
  │
  ├─ ns._ns_nloaded++                   :46
  ├─ new->l_serial = dl_load_adds       :47
  ├─ dl_load_adds++                     :48
  │
  └─ 释放锁
```

`l_serial` 是全局递增序列号，用于追踪加载顺序。

### 8.3 内存布局

一次 calloc 分配的完整内存块：

```
┌──────────────────────────────┐
│  struct link_map             │ ← new
│  (核心字段、l_scope_mem 等)  │
├──────────────────────────────┤
│  auditstate[naudit]          │ ← 审计状态数组
├──────────────────────────────┤
│  struct link_map *           │ ← l_symbolic_searchlist.r_list
├──────────────────────────────┤
│  struct libname_list         │ ← l_libname (newname)
├──────────────────────────────┤
│  "libfoo.so\0"              │ ← newname->name
└──────────────────────────────┘
```

---

## 9. 依赖排序——dl-sort-maps

### 9.1 排序目的

- **初始化顺序**：库 A 依赖库 B → B 的 DT_INIT 必须先于 A 执行
- **析构顺序**：与初始化相反，A 的 DT_FINI 先于 B 执行
- 用于 `_dl_init()` 和 `_dl_fini()` 的调用顺序

### 9.2 算法选择

`_dl_sort_maps()`（`dl-sort-maps.c:295-310`）：

```c
void _dl_sort_maps(struct link_map **maps, unsigned int nmaps,
                   bool force_first, bool for_fini)
{
  if (GLRO(dl_dso_sort_algo) == dso_sort_algorithm_original)
    _dl_sort_maps_original(maps, nmaps, force_first, for_fini);
  else
    _dl_sort_maps_dfs(maps, nmaps, force_first, for_fini);
}
```

通过 tunable `glibc.rtld.dynamic_sort` 控制（`dl-sort-maps.c:287-293`）：
- `1`（默认）→ 原始算法（_original，glibc 2.35 前默认）
- `2` → DFS 拓扑排序算法

### 9.3 原始算法

`_dl_sort_maps_original()`（`dl-sort-maps.c:28-122`）：

```
原始算法（冒泡/插入风格）:
  │
  ├─ 如果 force_first → 跳过 maps[0]
  │
  ├─ 对每个 maps[i]:
  │   扫描 maps[i+1..n] 寻找依赖关系
  │   ├─ 检查 l_initfini[]（链接时依赖）
  │   └─ 检查 l_reldeps[]（重定位时依赖，仅 for_fini）
  │
  ├─ 如果 maps[k] 被 maps[i] 依赖:
  │   memmove 将 maps[k] 提前到 maps[i] 位置
  │   重新从 maps[i] 开始检查
  │
  └─ 循环依赖处理:
      seen[] 计数器检测重复访问
      如果某节点被访问 > nmaps 次 → 放弃排序该节点
      dl-sort-maps.c:90-108
```

**特点**：简单但 O(n²) 最坏情况，循环依赖处理较粗糙。

### 9.4 DFS 拓扑排序

`_dl_sort_maps_dfs()`（`dl-sort-maps.c:175-285`）：

```
DFS 算法:
  │
  ├─ 第一遍 DFS:
  │   dfs_traversal() 递归遍历依赖图
  │   后序遍历 → rpo[] 逆后序 = 拓扑序
  │   dl-sort-maps.c:210-229
  │
  ├─ 遍历顺序:
  │   ├─ l_initfini[] 正序（链接时依赖）
  │   │   dl-sort-maps.c:145-154
  │   └─ l_reldeps[] 逆序（重定位依赖，仅 for_fini）
  │       dl-sort-maps.c:156-169
  │
  ├─ 如果存在重定位依赖 → 第二遍 DFS:
  │   仅按 l_initfini 排序
  │   与第一遍结果合并，优先使用链接时顺序
  │   dl-sort-maps.c:232-263
  │
  └─ force_first 修复:
      如果排序后 maps[0] 不是原始第一个 map
      → memmove 将原始第一个移回 maps[0]
      dl-sort-maps.c:265-284
```

`dfs_traversal()` 递归函数（`dl-sort-maps.c:134-173`）：

```
dfs_traversal(rpo, map, do_reldeps):
  │
  ├─ 已访问或 faked → 返回
  ├─ map->l_visited = 1
  │
  ├─ 遍历 l_initfini[] 中的依赖
  │   递归 dfs_traversal(dep)
  │
  ├─ 遍历 l_reldeps[]（如果需要）
  │   逆序递归 dfs_traversal(dep)
  │
  └─ 后序记录: *rpo -= 1; **rpo = map
```

### 9.5 循环依赖

```
循环依赖示例: A → B → C → A

原始算法: 检测到重复访问后放弃重排
DFS 算法: 接受循环中的一条边被违反
         注释明确说明: "在循环中至少有一个违反是不可避免的"
```

---

## 10. 快速地址查找——dl-find_object

### 10.1 问题与动机

```
给定一个 PC 地址，快速确定它属于哪个已加载的共享库

传统方案: dladdr() → 线性扫描 link_map 链表 → O(n)
新方案: _dl_find_object() → 排序数组 + 二分搜索 → O(log n)
        + 异步信号安全 + 无锁并发
```

### 10.2 数据结构

`dl-find_object.h:34-47`：

```c
struct dl_find_object_internal {
  uintptr_t map_start;    // 映射起始地址
  uintptr_t map_end;      // 映射结束地址（dlclose 时设为 map_start）
  struct link_map *map;   // link_map 指针（dlclose 时设为 NULL）
  void *eh_frame;         // 异常处理帧信息
  void *sframe;           // SFrame 信息
};
```

`dl-find_object.c:108-122`：

```c
struct dlfo_mappings_segment {
  struct dlfo_mappings_segment *previous; // 上一段（更低地址）
  void *to_free;                         // 释放用指针
  size_t size;                           // 已用条目数
  size_t allocated;                      // 分配容量
  struct dl_find_object_internal objects[]; // 按地址排序的数组
};
```

### 10.3 三层查找结构

```
┌─────────────────────────────────────────────────────────────────┐
│  层 1: _dlfo_main                                               │
│  主可执行文件的单个条目，快速路径判断                              │
├─────────────────────────────────────────────────────────────────┤
│  层 2: _dlfo_nodelete_mappings[]                                │
│  启动时加载的不可卸载对象，排序数组，二分搜索                      │
├─────────────────────────────────────────────────────────────────┤
│  层 3: _dlfo_loaded_mappings[2]                                 │
│  dlopen 加载的对象，分段链表                                     │
│  使用 STM（软件事务内存）双缓冲实现并发安全                       │
│  _dlfo_loaded_mappings_version 的最低位选择活跃缓冲区             │
└─────────────────────────────────────────────────────────────────┘
```

### 10.4 _dl_find_object() 查找流程

`dl-find_object.c:358-465`：

```
_dl_find_object(pc, result):
  │
  ├─ 快速路径 1: 主可执行文件
  │   if (pc >= _dlfo_main.map_start && pc < _dlfo_main.map_end)
  │     → 直接返回
  │   dl-find_object.c:371-376
  │
  ├─ 快速路径 2: 启动时加载的对象
  │   _dlfo_lookup(pc, _dlfo_nodelete_mappings, size)
  │   → 二分搜索排序数组
  │   dl-find_object.c:378-393
  │
  └─ 慢速路径: dlopen 对象
      STM 事务循环:
      ├─ start_version = _dlfo_read_start_version()
      ├─ 遍历段链表:
      │   对每个段 → _dlfo_lookup(pc, seg->objects, seg->size)
      │   dl-find_object.c:417-458
      ├─ 找到 → 复制数据 → 验证版本未变 → 返回
      └─ 版本变化 → 重试
          dl-find_object.c:440-447
```

### 10.5 _dlfo_lookup() 二分搜索

`dl-find_object.c:315-356`：

```c
static struct dl_find_object_internal *
_dlfo_lookup(uintptr_t pc, struct dl_find_object_internal *first, size_t size)
{
  // 标准二分搜索 lower_bound(map_start <= pc)
  while (size > 0) {
    half = size >> 1;
    middle = first + half;
    if (middle->map_start < pc) {
      first = middle + 1;
      size -= half + 1;
    } else
      size = half;
  }

  // 精确匹配: pc == first->map_start
  if (pc == first->map_start && pc < first->map_end)
    return first;

  // 检查前一个映射: pc 可能在 (first-1) 范围内
  --first;
  if (pc < first->map_end)
    return first;  // pc >= first->map_start 由搜索保证

  return NULL;
}
```

### 10.6 更新操作

**初始化**（`_dl_find_object_init`，`dl-find_object.c:576-621`）：

```
_dl_find_object_init():
  ├─ 设置 _dlfo_main（主可执行文件）
  ├─ _dlfo_process_initial() 收集所有初始对象
  ├─ 分配 nodelete 数组和 loaded 段
  └─ 排序两个数组
```

**dlopen 更新**（`_dl_find_object_update`，`dl-find_object.c:807-831`）：

```
_dl_find_object_update(new_maps):
  ├─ 收集未处理的 link_map 到数组
  ├─ 按 map_start 排序
  └─ 调用 _dl_find_object_update_1() (dl-find_object.c:669-805)
      ├─ 合并到事务性段链表
      └─ 切换活跃版本（原子递增 version）
```

**dlclose 标记**（`_dl_find_object_dlclose`，`dl-find_object.c:834-864`）：

```
_dl_find_object_dlclose(map):
  ├─ map_end = map_start  // 零长度，标记为已删除
  └─ map = NULL
```

### 10.7 与 dladdr 的关系

| 特性 | `dladdr()` | `_dl_find_object()` |
|------|-----------|-------------------|
| API 类型 | 公开 POSIX 扩展 | glibc 内部 |
| 算法 | 线性扫描 link_map | 二分搜索排序数组 |
| 并发安全 | 需要 dl_load_lock | 异步信号安全（STM） |
| 返回内容 | 文件名 + 符号名 + 基址 | 映射范围 + eh_frame |
| 主要用途 | 调试、堆栈回溯 | 异常处理（unwinder） |

`_dl_find_object()` 不替代 `dladdr()`——两者返回的信息不同，
服务于不同用途。

---

## 11. 四模块协作关系

```
dlopen("libfoo.so") 完整流程中各模块的参与:

_dl_open()
  │
  ├─ _dl_new_object()                    ← dl-object.c
  │   创建 link_map，设置名称/作用域
  │   加入命名空间链表
  │
  ├─ _dl_map_object_from_fd()
  │   映射 ELF 段，解析程序头
  │   如果有 PT_TLS:
  │     ├─ _dl_assign_tls_modid()        ← dl-tls.c
  │     │   分配 TLS 模块 ID
  │     └─ 记录 l_tls_blocksize / l_tls_align / l_tls_initimage
  │
  ├─ _dl_map_object_deps()
  │   构建依赖图
  │
  ├─ _dl_sort_maps()                     ← dl-sort-maps.c
  │   拓扑排序确定初始化顺序
  │
  ├─ _dl_relocate_object()
  │   重定位
  │
  ├─ _dl_add_to_slotinfo()               ← dl-tls.c
  │   注册到全局 slotinfo，递增代际
  │
  ├─ _dl_init_static_tls()               ← dl-tls.c (如果需要)
  │   为所有已有线程初始化静态 TLS
  │
  ├─ _dl_find_object_update()            ← dl-find_object.c
  │   更新地址查找数据结构
  │
  └─ _dl_init()
      按排序顺序调用 DT_INIT / DT_INIT_ARRAY

线程访问 libfoo 的 TLS 变量:
  __tls_get_addr()                       ← dl-tls.c
    检测代际 → 更新 DTV → 延迟分配 → 返回地址

异常处理 unwinder 查找 eh_frame:
  _dl_find_object(pc)                    ← dl-find_object.c
    三层二分搜索 → 返回 eh_frame 信息
```

---

## 12. 源文件速查表

| 源文件 | 行号 | 内容 |
|--------|------|------|
| `elf/dl-tls.c` | 160-230 | `_dl_assign_tls_modid` 分配模块 ID |
| `elf/dl-tls.c` | 256-463 | `_dl_determine_tlsoffset` 静态偏移计算 |
| `elf/dl-tls.c` | 465-493 | `allocate_dtv` DTV 分配 |
| `elf/dl-tls.c` | 498-503 | `_dl_get_tls_static_info` 静态 TLS 大小 |
| `elf/dl-tls.c` | 522-552 | `_dl_allocate_tls_storage` TCB+TLS 分配 |
| `elf/dl-tls.c` | 560-609 | `_dl_resize_dtv` DTV 扩容 |
| `elf/dl-tls.c` | 622-727 | `_dl_allocate_tls_init` TLS 初始化 |
| `elf/dl-tls.c` | 729-741 | `_dl_allocate_tls` 总入口 |
| `elf/dl-tls.c` | 744-772 | `_dl_deallocate_tls` 释放 TLS |
| `elf/dl-tls.c` | 779-809 | `allocate_dtv_entry` 延迟分配单个模块 |
| `elf/dl-tls.c` | 835-980 | `_dl_update_slotinfo` 代际同步 |
| `elf/dl-tls.c` | 1090-1142 | `__tls_get_addr` 运行时 TLS 访问 |
| `elf/dl-tls.c` | 1148-1190 | `_dl_tls_get_addr_soft` 非分配查询 |
| `elf/dl-tls.c` | 1194-1214 | `_dl_tls_initial_modid_limit_setup` |
| `elf/dl-tls.c` | 1237-1302 | `_dl_add_to_slotinfo` 注册 TLS 模块 |
| `elf/dl-tls.c` | 1305-1318 | `init_one_static_tls` 单线程静态 TLS 初始化 |
| `elf/dl-tls.c` | 1322-1337 | `_dl_init_static_tls` 全线程静态 TLS 初始化 |
| `sysdeps/generic/dl-dtv.h` | 22-33 | `dtv_t` / `dtv_pointer` 定义 |
| `sysdeps/x86_64/nptl/tls.h` | 42-70 | x86_64 `tcbhead_t` 定义 |
| `sysdeps/x86_64/nptl/tls.h` | 112-115 | TLS_TCB_AT_TP 定义 |
| `sysdeps/aarch64/nptl/tls.h` | 43-47 | AArch64 `tcbhead_t` 定义 |
| `sysdeps/aarch64/nptl/tls.h` | 74-75 | AArch64 `TLS_INIT_TP` |
| `elf/dl-object.c` | 28-51 | `_dl_add_to_namespace_list` |
| `elf/dl-object.c` | 54-263 | `_dl_new_object` link_map 创建 |
| `elf/dl-sort-maps.c` | 28-122 | `_dl_sort_maps_original` 原始排序 |
| `elf/dl-sort-maps.c` | 134-173 | `dfs_traversal` DFS 遍历 |
| `elf/dl-sort-maps.c` | 175-285 | `_dl_sort_maps_dfs` DFS 拓扑排序 |
| `elf/dl-sort-maps.c` | 287-293 | `_dl_sort_maps_init` 算法选择 |
| `elf/dl-sort-maps.c` | 295-310 | `_dl_sort_maps` 排序入口 |
| `elf/dl-find_object.c` | 108-122 | `dlfo_mappings_segment` 段结构 |
| `elf/dl-find_object.c` | 315-356 | `_dlfo_lookup` 二分搜索 |
| `elf/dl-find_object.c` | 358-465 | `_dl_find_object` 查找入口 |
| `elf/dl-find_object.c` | 576-621 | `_dl_find_object_init` 初始化 |
| `elf/dl-find_object.c` | 623-646 | `_dl_find_object_link_map_sort` 排序 |
| `elf/dl-find_object.c` | 807-831 | `_dl_find_object_update` dlopen 更新 |
| `elf/dl-find_object.c` | 834-864 | `_dl_find_object_dlclose` dlclose 标记 |
| `elf/dl-find_object.h` | 34-47 | `dl_find_object_internal` 结构 |
