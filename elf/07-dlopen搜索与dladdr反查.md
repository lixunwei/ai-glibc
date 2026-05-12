# dlopen/dladdr/dl_iterate_phdr 深度分析

> 基于 glibc 2.43.9000 动态链接器 API 源码分析  
> 重点：dlopen 库搜索逻辑、dladdr 地址反查算法、dl_iterate_phdr 遍历机制

---

## 目录

1. [dlopen 库搜索逻辑](#1-dlopen-库搜索逻辑)
2. [搜索路径解析与 DST 替换](#2-搜索路径解析与-dst-替换)
3. [ld.so.cache 缓存查找](#3-ldsocache-缓存查找)
4. [已加载检测——名称匹配与 dev/ino 去重](#4-已加载检测名称匹配与-devino-去重)
5. [_dl_map_object_from_fd——从 fd 到 link_map](#5-_dl_map_object_from_fd从-fd-到-link_map)
6. [dladdr——地址反查符号](#6-dladdr地址反查符号)
7. [dl_iterate_phdr——遍历所有已加载对象](#7-dl_iterate_phdr遍历所有已加载对象)
8. [dlinfo——查询库元信息](#8-dlinfo查询库元信息)
9. [link_map 组织与命名空间](#9-link_map-组织与命名空间)
10. [线程安全与锁分析](#10-线程安全与锁分析)
11. [源文件速查表](#11-源文件速查表)

---

## 1. dlopen 库搜索逻辑

### 1.1 入口函数拆分

```
dlopen(filename, flags)
  → ___dlopen()                              dlfcn/dlopen.c:75-103
    → _dlerror_run(dlopen_doit)              错误捕获包装
      → dlopen_doit()                        dlfcn/dlopen.c:46-60
        → _dl_open(filename, mode, ...)      elf/dl-open.c:812
          → _dl_open_worker_begin()          elf/dl-open.c:513-747
            → _dl_map_object()               elf/dl-load.c:2191-2199
              → _dl_lookup_map()             名称匹配已加载
              → _dl_map_new_object()         实际搜索+加载
```

`_dl_map_object` 是搜索的入口点，它首先调用 `_dl_lookup_map` 检查是否已加载，
如果未找到则调用 `_dl_map_new_object` 启动文件搜索流程。

**源码**（`dl-load.c:2191-2199`）：

```c
struct link_map *
_dl_map_object (struct link_map *loader, const char *name,
                int type, int trace_mode, int mode, Lmid_t nsid)
{
  struct link_map *l = _dl_lookup_map (nsid, name);
  if (l != NULL)
    return l;
  return _dl_map_new_object (loader, name, type, trace_mode, mode, nsid);
}
```

### 1.2 核心搜索顺序

当库名不含 `/`（即非绝对路径/相对路径）时，`_dl_map_new_object` 按以下**严格顺序**
搜索（`dl-load.c:1969-2110`）：

```
┌──────────────────────────────────────────────────────────────────┐
│              dlopen("libfoo.so", RTLD_LAZY) 搜索顺序             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. DT_RPATH（仅当 loader 无 DT_RUNPATH 时）                     │
│     └─ 从 loader 沿 l_loader 链向上遍历每个对象的 DT_RPATH       │
│        dl-load.c:1980-2002                                       │
│                                                                  │
│  2. 可执行文件的 DT_RPATH（如果步骤1未搜索到且可执行文件有 RPATH） │
│     dl-load.c:2004-2013                                          │
│                                                                  │
│  3. LD_LIBRARY_PATH 环境变量                                      │
│     dl-load.c:2031-2036                                          │
│                                                                  │
│  4. loader 的 DT_RUNPATH                                          │
│     dl-load.c:2038-2044                                          │
│                                                                  │
│  5. /etc/ld.so.cache（ldconfig 缓存）                             │
│     dl-load.c:2046-2101                                          │
│                                                                  │
│  6. 默认系统目录（/lib, /usr/lib 等）                              │
│     dl-load.c:2104-2110                                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**关键规则**：

| 规则 | 代码位置 | 说明 |
|------|---------|------|
| RUNPATH 禁用 RPATH | `dl-load.c:1982` | `loader->l_info[DT_RUNPATH] == NULL` 时才搜 RPATH |
| DF_1_NODEFLIB | `dl-load.c:2068` | 禁止使用默认目录的缓存条目 |
| 安全模式过滤 | `dl-load.c:2048-2049` | `__RTLD_SECURE` 模式下跳过 ld.so.cache |
| LD_AUDIT dlopen | `dl-load.c:2015-2028` | 审计模块也搜索可执行文件的 DT_RUNPATH |

### 1.3 DT_RPATH vs DT_RUNPATH

```
                DT_RPATH                    DT_RUNPATH
搜索优先级    最高（步骤1/2）              中等（步骤4）
互斥关系      有 RUNPATH 时被禁用          与 RPATH 互斥
搜索范围      loader 链上所有对象          仅 loader 自身
ELF 标准      已废弃（但仍广泛使用）       推荐使用
链接器选项    -rpath                       -rpath + --enable-new-dtags
```

### 1.4 open_path——目录逐一尝试

每一步的实际文件查找由 `open_path()` 完成（`dl-load.c:1737-1894`）：

```
open_path(name, namelen, mode, search_path_struct, ...):
  for each dir in search_path_struct.dirs:
    │
    ├─ 拼接路径: buf = dir->dirname + capstr + name
    │
    ├─ open_verify(buf, ...)
    │    └─ open() + 验证 ELF 头 (魔数、机器类型、class)
    │
    ├─ 缓存目录状态:
    │    unknown → existing/nonexisting（下次跳过不存在的目录）
    │
    └─ 安全模式检查: __RTLD_SECURE 时验证 SUID 位
```

目录状态缓存是一个重要优化：一旦某目录被标记为 `nonexisting`，后续搜索不再 stat。

---

## 2. 搜索路径解析与 DST 替换

### 2.1 LD_LIBRARY_PATH 解析

`_dl_init_paths()` 在 ld.so 启动时解析 LD_LIBRARY_PATH（`dl-load.c:800-831`）：

```c
// 按 ':' 和 ';' 分割
for (const char *cp = llp_tmp; *cp != '\0'; ++cp)
  if (*cp == ':' || *cp == ';')
    ++nllp;

// 分配路径元素数组
__rtld_env_path_list.dirs = malloc((nllp + 1) * sizeof(...));

// 填充路径: 展开 DST、去重、规范化
fillin_rpath(llp_tmp, __rtld_env_path_list.dirs, ":;", ...);
```

若 LD_LIBRARY_PATH 为空或解析后无有效目录，`dirs` 设为 `(void *) -1` 表示无效。

### 2.2 DST（动态字符串令牌）替换

路径中可包含三种动态令牌（`dl-load.c:227-401`）：

| 令牌 | 含义 | 示例 |
|------|------|------|
| `$ORIGIN` | 包含 loader 的目录 | `$ORIGIN/../lib` → `/opt/app/../lib` |
| `$PLATFORM` | 平台标识字符串 | `$PLATFORM` → `aarch64` |
| `$LIB` | 库目录名 | `$LIB` → `lib` 或 `lib64` |

**处理流程**：

```
_dl_dst_count(input)          计算令牌数量   dl-load.c:227-255
  │
  ▼
DL_DST_REQUIRED(l, input)     计算替换后长度
  │
  ▼
_dl_dst_substitute(l, input)  逐字符替换     dl-load.c:265-366
```

安全限制：`$ORIGIN` 在 SUID/SGID 程序中被限制在受信目录（`dl-load.c:288-317`）。

### 2.3 fillin_rpath——路径列表规范化

`fillin_rpath()` 对每条路径执行（`dl-load.c:443-549`）：

```
1. strsep 按分隔符拆分
2. expand_dynamic_string_token() 展开 $ORIGIN 等
3. 去除尾部多余 '/'，确保以 '/' 结尾
4. 在 GL(dl_all_dirs) 全局链表中去重
5. 创建 r_search_path_elem 结构加入结果数组
```

### 2.4 默认系统目录

编译时定义（`dl-load.c:109-116`）：

```c
#include "trusted-dirs.h"
static const char system_dirs[] = SYSTEM_DIRS;
static const size_t system_dirs_len[] = { SYSTEM_DIRS_LEN };
```

通常包含 `/lib`、`/usr/lib`（以及架构变体如 `/lib/aarch64-linux-gnu`）。
`SYSTEM_DIRS` 是编译时由 `trusted-dirs.h` 生成的常量字符串。

---

## 3. ld.so.cache 缓存查找

### 3.1 缓存文件格式

缓存位于 `/etc/ld.so.cache`，由 `ldconfig` 生成（`sysdeps/generic/dl-cache.h:37-182`）：

```
ld.so.cache 文件结构：
┌──────────────────────────────────────────┐
│  旧格式头 (struct cache_file)             │  可选
│  ├─ magic: "ld.so-1.7.0"                │
│  ├─ nlibs: 条目数                         │
│  └─ libs[nlibs]: file_entry 数组          │
├──────────────────────────────────────────┤
│  对齐填充                                 │
├──────────────────────────────────────────┤
│  新格式头 (struct cache_file_new)         │  主要使用
│  ├─ magic: "glibc-ld.so.cache1.1"        │
│  ├─ nlibs: 条目数                         │
│  ├─ flags_generation: 生成标志            │
│  └─ libs[nlibs]: file_entry_new 数组     │
├──────────────────────────────────────────┤
│  字符串表（库名和路径）                    │
└──────────────────────────────────────────┘
```

每个 `file_entry_new` 包含：

```c
struct file_entry_new {      // dl-cache.h:86-100
  int32_t flags;             // 库类型标志（ELF/libc5/架构）
  uint32_t key;              // 库名在字符串表中的偏移
  uint32_t value;            // 文件路径在字符串表中的偏移
  uint32_t osversion_unused;
  uint64_t hwcap;            // HWCAP 或 glibc-hwcaps 子目录索引
};
```

### 3.2 查找算法——二分搜索

`_dl_load_cache_lookup()` 是缓存查找入口（`dl-cache.c:384-499`）：

```
_dl_load_cache_lookup(name):
  │
  ├─ 首次调用时 mmap 整个 /etc/ld.so.cache
  │    dl-cache.c:391-458
  │
  ├─ 检测格式：纯新格式 / 旧格式 / 旧+新混合格式
  │
  └─ search_cache()           dl-cache.c:194-336
       │
       ├─ 二分搜索匹配库名
       │    使用 _dl_cache_libcmp() 比较
       │    dl-cache.c:206-332
       │
       ├─ 找到后向前扫描所有同名条目
       │    dl-cache.c:226-236
       │
       ├─ 按优先级选择最佳匹配：
       │    1. named hwcap 条目（glibc-hwcaps 子目录）
       │    2. 普通条目（hwcap == 0）
       │    dl-cache.c:270-310
       │
       └─ 返回路径（从字符串表复制）
            dl-cache.c:490-498
```

### 3.3 _dl_cache_libcmp——版本感知比较

缓存中的库名排序使用特殊比较函数（`dl-cache.c:338-374`），
它**将数字部分作为数值比较**而非字典序：

```
libfoo.so.1   vs  libfoo.so.2   → 按数值: 1 < 2
libfoo.so.10  vs  libfoo.so.9   → 按数值: 10 > 9  (字典序会得 "10" < "9")
```

这确保了版本号的正确排序。

### 3.4 HWCAP 优先级

缓存条目包含 HWCAP 信息，`search_cache` 优先选择与当前 CPU 匹配的
glibc-hwcaps 条目（如 `x86-64-v3`），然后才回退到通用条目。

---

## 4. 已加载检测——名称匹配与 dev/ino 去重

glibc 使用**两级去重**防止同一库被加载两次：

### 4.1 第一级：名称匹配（_dl_lookup_map）

在文件搜索之前进行（`dl-load.c:1896-1926`）：

```c
struct link_map *
_dl_lookup_map (Lmid_t nsid, const char *name)
{
  for (struct link_map *l = GL(dl_ns)[nsid]._ns_loaded; l; l = l->l_next)
    {
      if (l->l_faked | l->l_removed)
        continue;
      if (!_dl_name_match_p (name, l))           // dl-misc.c:65-80
        {
          // 名称不匹配，尝试 soname 匹配
          if (l->l_soname_added || l_soname(l) == NULL
              || strcmp(name, l_soname(l)) != 0)
            continue;
          // soname 匹配成功，缓存名称
          add_name_to_object(l, l_soname(l));
          l->l_soname_added = 1;
        }
      return l;  // 找到已加载的对象
    }
  return NULL;
}
```

`_dl_name_match_p` 检查 `l_name` 和 `l_libname` 链表中的所有别名。

### 4.2 第二级：设备/inode 去重

文件打开后，在 `_dl_map_object_from_fd` 中进行（`dl-load.c:999-1013`）：

```c
// 获取文件的 st_dev + st_ino
_dl_get_file_id(fd, &id);

// 扫描已加载对象的 l_file_id
for (l = GL(dl_ns)[nsid]._ns_loaded; l != NULL; l = l->l_next)
  if (!l->l_removed && _dl_file_id_match_p(&l->l_file_id, &id))
    {
      // 同一物理文件——关闭 fd，缓存别名，返回已有 link_map
      __close_nocancel(fd);
      add_name_to_object(l, name);
      return l;
    }
```

文件 ID 比较（`sysdeps/posix/dl-fileid.h:22-50`）：

```c
static inline bool
_dl_file_id_match_p (const struct r_file_id *a, const struct r_file_id *b)
{
  return a->dev == b->dev && a->ino == b->ino;
}
```

这确保即使通过不同路径（如 `/lib/libfoo.so` 和 `/usr/lib/libfoo.so` 是符号链接指向
同一文件）也不会重复加载。

---

## 5. _dl_map_object_from_fd——从 fd 到 link_map

文件找到并通过 dev/ino 去重后，进入 ELF 加载核心（`dl-load.c:938-1315`）：

```
_dl_map_object_from_fd(name, fd, fbp, realname, loader, ...):
  │
  ├─ 1. 获取 st_dev/st_ino → r_file_id
  │     dl-load.c:960-997
  │
  ├─ 2. dev/ino 去重检查（见上节）
  │     dl-load.c:999-1013
  │
  ├─ 3. 命名空间检查（非 base 命名空间不能加载 ld.so）
  │     dl-load.c:1016-1022
  │
  ├─ 4. 创建 link_map 结构
  │     _dl_new_object(realname, name, ...)
  │     dl-load.c:1065-1070
  │
  ├─ 5. 解析 ELF 头和程序头表
  │     验证: ELF magic、e_type (ET_DYN)、e_machine
  │     遍历 PT_LOAD 段计算映射范围
  │     dl-load.c:1072-1160
  │
  ├─ 6. mmap 所有 PT_LOAD 段
  │     第一段: mmap(NULL, ...)   → 基址
  │     后续段: mmap(fixed, ...)  → 相对偏移
  │     dl-load.c:1162-1218
  │
  ├─ 7. 处理 .dynamic 段
  │     解析 DT_* 标记，填充 l_info[] 数组
  │     dl-load.c:1220-1235
  │
  ├─ 8. 安全检查
  │     拒绝 ET_EXEC (dlopen 只能加载 ET_DYN)
  │     拒绝 DF_1_PIE 和 DF_1_NOOPEN
  │     dl-load.c:1236-1291
  │
  └─ 9. 设置 l_file_id、关闭 fd、处理 GNU_PROPERTY
        dl-load.c:1293-1315
```

---

## 6. dladdr——地址反查符号

### 6.1 调用路径

```
dladdr(address, info)
  → _dl_addr(address, info, NULL, NULL)       elf/dl-addr.c:117-138

dladdr1(address, info, extra, flags)
  → __dladdr1(address, info, extra, flags)    dlfcn/dladdr1.c:23-41
    → _dl_addr(address, info, mapp, symbolp)
```

### 6.2 定位包含对象

`_dl_addr` 首先调用 `_dl_find_dso_for_object` 查找包含地址的 link_map
（`elf/dl-open.c:212-227`）：

```c
_dl_find_dso_for_object (const ElfW(Addr) addr)
{
  for (Lmid_t ns = 0; ns < GL(dl_nns); ++ns)
    for (l = GL(dl_ns)[ns]._ns_loaded; l != NULL; l = l->l_next)
      if (addr >= l->l_map_start && addr < l->l_map_end
          && (l->l_contiguous
              || _dl_addr_inside_object(l, addr)))
        return l;
  return NULL;
}
```

两级检查：
1. **快速范围检查**：`l_map_start <= addr < l_map_end`
2. **精确段检查**：`_dl_addr_inside_object` 逐段验证（仅在非连续映射时）

### 6.3 _dl_addr_inside_object——逐段验证

`elf/dl-addr-obj.c:63-74`：

```c
int _dl_addr_inside_object (struct link_map *l, const ElfW(Addr) addr)
{
  int n = l->l_phnum;
  const ElfW(Addr) reladdr = addr - l->l_addr;

  while (--n >= 0)
    if (l->l_phdr[n].p_type == PT_LOAD
        && reladdr - l->l_phdr[n].p_vaddr < l->l_phdr[n].p_memsz)
      return 1;
  return 0;
}
```

使用无符号减法技巧：`reladdr - p_vaddr < p_memsz` 同时检查下界和上界。

### 6.4 最近符号查找——线性扫描

`determine_info` 在找到的 link_map 中查找最近的符号（`dl-addr.c:24-114`）：

```
determine_info(addr, match, info, ...):
  │
  ├─ 填充基本信息:
  │    dli_fname = match->l_name   (主程序特判: 使用 _dl_argv[0])
  │    dli_fbase = match->l_map_start
  │
  ├─ 如果有 GNU hash 表:
  │    遍历所有桶 → 遍历链表中每个符号
  │    过滤: SHN_UNDEF(值为0)、SHN_ABS、STT_TLS、名称越界
  │    DL_ADDR_SYM_MATCH 比较: st_value 最接近且不超过 addr
  │    dl-addr.c:45-72
  │
  ├─ 否则如果有 SysV hash 表:
  │    线性扫描整个 DT_SYMTAB
  │    额外过滤: 仅 STB_GLOBAL/STB_WEAK、非本地可见性
  │    dl-addr.c:74-90
  │
  └─ 设置结果:
       dli_sname = strtab + matchsym->st_name
       dli_saddr = DL_SYMBOL_ADDRESS(matchl, matchsym)
```

**注意：使用线性扫描，非二分搜索**。因为符号表未按地址排序，
且 GNU hash 仅按哈希值组织。对于大型库，dladdr 的性能为 O(n)，n 为导出符号数。

### 6.5 dladdr vs dladdr1

| API | 返回内容 | 额外信息 |
|-----|---------|---------|
| `dladdr(addr, info)` | `Dl_info` | 无 |
| `dladdr1(addr, info, extra, RTLD_DL_SYMENT)` | `Dl_info` + `const ElfW(Sym)*` | 符号表项 |
| `dladdr1(addr, info, extra, RTLD_DL_LINKMAP)` | `Dl_info` + `struct link_map*` | link_map 指针 |

---

## 7. dl_iterate_phdr——遍历所有已加载对象

### 7.1 实现

完整实现位于 `elf/dl-iteratephdr.c:30-87`：

```c
int __dl_iterate_phdr (callback, data)
{
  __rtld_lock_lock_recursive(GL(dl_load_write_lock));
  __libc_cleanup_push(cancel_handler, NULL);   // 取消安全

  // 确定调用者的命名空间
  Lmid_t ns = 0;
  const void *caller = RETURN_ADDRESS(0);
  for (cnt = GL(dl_nns) - 1; cnt > 0; --cnt)
    for (l = GL(dl_ns)[cnt]._ns_loaded; l; l = l->l_next)
      if (caller >= l->l_map_start && caller < l->l_map_end
          && (l->l_contiguous || _dl_addr_inside_object(l, caller)))
        ns = cnt;

  // 遍历该命名空间的所有对象
  for (l = GL(dl_ns)[ns]._ns_loaded; l != NULL; l = l->l_next)
    {
      info = { l->l_addr, l->l_name, l->l_phdr, l->l_phnum,
               GL(dl_load_adds),
               GL(dl_load_adds) - nloaded,
               l->l_tls_modid,
               (modid ? dl_tls_get_addr_soft(l) : NULL) };
      ret = callback(&info, sizeof(info), data);
      if (ret) break;
    }

  __libc_cleanup_pop(0);
  __rtld_lock_unlock_recursive(GL(dl_load_write_lock));
  return ret;
}
```

### 7.2 struct dl_phdr_info 字段

定义位于 `elf/link.h:155-180`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dlpi_addr` | `ElfW(Addr)` | 对象加载基址（l_addr） |
| `dlpi_name` | `const char *` | 对象路径名 |
| `dlpi_phdr` | `const ElfW(Phdr) *` | 程序头表指针 |
| `dlpi_phnum` | `ElfW(Half)` | 程序头表条目数 |
| `dlpi_adds` | `unsigned long long` | 全局累计加载次数 |
| `dlpi_subs` | `unsigned long long` | 推导的卸载次数 = adds - nloaded |
| `dlpi_tls_modid` | `size_t` | TLS 模块 ID（0 = 无 TLS） |
| `dlpi_tls_data` | `void *` | 当前线程的 TLS 数据块指针 |

### 7.3 adds/subs 计数器

```
dlpi_adds = GL(dl_load_adds)       // 全局加载计数器（只增）
dlpi_subs = GL(dl_load_adds) - nloaded  // 推导：总加载 - 当前加载数
```

用途：异常处理库（libgcc_s）通过比较两次回调的 adds 值判断 `.eh_frame` 是否需要重建。

### 7.4 回调安全性

**回调中不能安全调用 dlopen/dlclose**。原因：
- 迭代全程持有 `dl_load_write_lock`（递归锁）
- 虽然递归锁允许重入，但修改 link_map 链表会破坏迭代状态
- 回调设有取消清理处理器（`cancel_handler`），防止线程取消时死锁

### 7.5 主要使用场景

| 使用者 | 目的 |
|--------|------|
| libgcc_s | 查找 `.eh_frame` 段进行 C++ 异常处理 |
| libunwind | 栈回溯 |
| 安全工具 | 枚举所有加载的共享库 |
| 调试器 | 获取加载信息 |
| sanitizer | 检查内存映射 |

---

## 8. dlinfo——查询库元信息

### 8.1 实现

`dlfcn/dlinfo.c:37-96`，通过 `_dlerror_run` 包装错误处理：

```c
static void dlinfo_doit (void *argsblock)
{
  struct link_map *l = args->handle;

  switch (args->request) {
    case RTLD_DI_LMID:       *(Lmid_t *)arg = l->l_ns;           break;
    case RTLD_DI_LINKMAP:    *(struct link_map **)arg = l;        break;
    case RTLD_DI_SERINFO:    _dl_rtld_di_serinfo(l, arg, false);  break;
    case RTLD_DI_SERINFOSIZE:_dl_rtld_di_serinfo(l, arg, true);   break;
    case RTLD_DI_ORIGIN:     strcpy(arg, l->l_origin);            break;
    case RTLD_DI_ORIGIN_PATH:*(const char **)arg = l->l_origin;   break;
    case RTLD_DI_TLS_MODID:  *(size_t *)arg = l->l_tls_modid;    break;
    case RTLD_DI_TLS_DATA:   *(void **)arg = dl_tls_get_addr_soft(l); break;
    case RTLD_DI_PHDR:       *(Phdr **)arg = l->l_phdr;
                             result = l->l_phnum;                 break;
    default: error("unsupported dlinfo request");
  }
}
```

### 8.2 请求类型汇总

| 请求 | 值 | 返回内容 | 定义位置 |
|------|-----|---------|---------|
| `RTLD_DI_LMID` | 1 | 命名空间 ID | `dlfcn/dlfcn.h:137` |
| `RTLD_DI_LINKMAP` | 2 | link_map 指针 | `dlfcn/dlfcn.h:139` |
| `RTLD_DI_SERINFO` | 4 | 搜索路径列表 | `dlfcn/dlfcn.h:143` |
| `RTLD_DI_SERINFOSIZE` | 5 | 搜索路径列表大小 | `dlfcn/dlfcn.h:145` |
| `RTLD_DI_ORIGIN` | 6 | $ORIGIN 目录 | `dlfcn/dlfcn.h:147` |
| `RTLD_DI_ORIGIN_PATH` | 12 | $ORIGIN 路径指针 | `dlfcn/dlfcn.h:175` |
| `RTLD_DI_TLS_MODID` | 9 | TLS 模块 ID | `dlfcn/dlfcn.h:163` |
| `RTLD_DI_TLS_DATA` | 10 | 当前线程 TLS 块 | `dlfcn/dlfcn.h:167` |
| `RTLD_DI_PHDR` | 11 | 程序头表 | `dlfcn/dlfcn.h:171` |

`RTLD_DI_SERINFO` / `RTLD_DI_SERINFOSIZE` 提供了一种编程方式获取
dlopen 搜索特定库时将使用的完整目录列表（RPATH、LD_LIBRARY_PATH、
RUNPATH、cache、default 的合并结果）。

---

## 9. link_map 组织与命名空间

### 9.1 命名空间结构

```
GL(dl_ns)[DL_NNS]  ← 命名空间数组，最多 DL_NNS 个（通常 16）
  │
  ├─ [0] (LM_ID_BASE) ── 默认命名空间
  │    ├─ _ns_loaded → exec → libc.so → libm.so → ... (链表)
  │    ├─ _ns_nloaded = 已加载对象数
  │    └─ _ns_main_searchlist = 全局搜索作用域
  │
  ├─ [1] ── dlmopen 创建的命名空间
  │    ├─ _ns_loaded → libfoo.so → libc.so(副本) → ...
  │    └─ ...
  │
  └─ [N-1] ── ...

GL(dl_nns) = 当前使用的命名空间数量
```

每个命名空间由 `struct link_namespaces` 描述（`sysdeps/generic/ldsodefs.h:320-358`）。

### 9.2 link_map 链表

对象通过 `l_next` 指针形成**单向链表**，按加载顺序排列：

```
namespace[0]._ns_loaded
  → [exec]  (l_type = lt_executable)
  → [vdso]  (l_type = lt_library)
  → [ld.so] (l_type = lt_library)
  → [libc]  (l_type = lt_library)
  → [libm]  (l_type = lt_library)
  → [dlopen'd libs] (l_type = lt_loaded)
  → NULL
```

### 9.3 API 如何使用命名空间

| API | 命名空间行为 |
|-----|-------------|
| `dlopen` | 在调用者所在命名空间加载 |
| `dlmopen(LM_ID_NEWLM, ...)` | 创建新命名空间加载 |
| `dlmopen(LM_ID_BASE, ...)` | 在默认命名空间加载 |
| `dladdr` | 搜索**所有**命名空间 |
| `dl_iterate_phdr` | 仅遍历调用者所在命名空间 |
| `dlinfo(RTLD_DI_LMID)` | 返回对象所在命名空间 ID |

---

## 10. 线程安全与锁分析

### 10.1 三把锁

```
┌─────────────────────────────────────────────────────┐
│  GL(dl_load_lock)         — 主互斥锁（递归）         │
│  序列化 dlopen/dlclose/dladdr 操作                   │
├─────────────────────────────────────────────────────┤
│  GL(dl_load_write_lock)   — 写锁（递归）             │
│  保护 link_map 链表修改                               │
│  dl_iterate_phdr 持有此锁全程                         │
├─────────────────────────────────────────────────────┤
│  GL(dl_load_tls_lock)     — TLS 锁                   │
│  保护 TLS 模块 ID 分配和 DTV 扩展                     │
└─────────────────────────────────────────────────────┘
```

### 10.2 各 API 的锁使用

| API | 锁 | 持有范围 |
|-----|-----|---------|
| `dlopen` | `dl_load_lock` + `dl_load_write_lock` | 整个加载过程 |
| `dlclose` | `dl_load_lock` + `dl_load_write_lock` | 整个卸载过程 |
| `dlsym` | `dl_load_lock` | 符号查找期间 |
| `dladdr` | `dl_load_lock` | 地址查找期间 |
| `dl_iterate_phdr` | `dl_load_write_lock` | 整个遍历过程 |
| `dlinfo` | `dl_load_lock` (通过 `_dlerror_run`) | 信息查询期间 |

---

## 11. 源文件速查表

| 源文件 | 行号 | 内容 |
|--------|------|------|
| `elf/dl-load.c` | 109-116 | 默认系统目录 `system_dirs` 定义 |
| `elf/dl-load.c` | 227-255 | `_dl_dst_count` DST 令牌计数 |
| `elf/dl-load.c` | 265-366 | `_dl_dst_substitute` DST 替换 |
| `elf/dl-load.c` | 374-401 | `expand_dynamic_string_token` 展开入口 |
| `elf/dl-load.c` | 443-549 | `fillin_rpath` 路径列表规范化 |
| `elf/dl-load.c` | 800-831 | `_dl_init_paths` 中 LD_LIBRARY_PATH 解析 |
| `elf/dl-load.c` | 938-1315 | `_dl_map_object_from_fd` 从 fd 到 link_map |
| `elf/dl-load.c` | 999-1013 | dev/ino 去重检查 |
| `elf/dl-load.c` | 1737-1894 | `open_path` 目录逐一尝试 |
| `elf/dl-load.c` | 1896-1926 | `_dl_lookup_map` 名称匹配已加载 |
| `elf/dl-load.c` | 1969-2110 | 搜索顺序：RPATH→LD_LIBRARY_PATH→RUNPATH→cache→default |
| `elf/dl-load.c` | 2191-2199 | `_dl_map_object` 入口 |
| `elf/dl-cache.c` | 194-336 | `search_cache` 二分搜索缓存 |
| `elf/dl-cache.c` | 338-374 | `_dl_cache_libcmp` 版本感知比较 |
| `elf/dl-cache.c` | 384-499 | `_dl_load_cache_lookup` 缓存查找入口 |
| `sysdeps/generic/dl-cache.h` | 86-100 | `file_entry_new` 缓存条目结构 |
| `sysdeps/generic/dl-cache.h` | 157-182 | `cache_file_new` 缓存文件头 |
| `sysdeps/posix/dl-fileid.h` | 22-50 | `_dl_file_id_match_p` dev/ino 比较 |
| `elf/dl-misc.c` | 65-80 | `_dl_name_match_p` 名称/别名匹配 |
| `elf/dl-addr.c` | 24-114 | `determine_info` 最近符号查找 |
| `elf/dl-addr.c` | 117-138 | `_dl_addr` 入口 |
| `elf/dl-addr-obj.c` | 63-74 | `_dl_addr_inside_object` 逐段地址验证 |
| `elf/dl-open.c` | 212-228 | `_dl_find_dso_for_object` 地址→link_map |
| `elf/dl-iteratephdr.c` | 30-87 | `__dl_iterate_phdr` 完整实现 |
| `dlfcn/dladdr1.c` | 23-41 | `__dladdr1` 扩展接口 |
| `dlfcn/dlinfo.c` | 37-96 | `dlinfo_doit` 请求分发 |
| `dlfcn/dlfcn.h` | 137-175 | `RTLD_DI_*` 常量定义 |
