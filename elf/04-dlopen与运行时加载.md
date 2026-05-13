# dlopen 与运行时动态加载

## 一、API 概览

| 函数 | 源文件 | 说明 |
|------|--------|------|
| `dlopen` | `dlfcn/dlopen.c:75-103` | 加载共享库 |
| `dlsym` | `dlfcn/dlsym.c:35-69` | 查找符号 |
| `dlclose` | `dlfcn/dlclose.c:23-32` | 卸载共享库 |
| `dladdr` | `dlfcn/dladdr.c:23-31` | 地址反查符号/库（核心逻辑 `elf/dl-addr.c:117-138`） |
| `dlerror` | `dlfcn/dlerror.c:31-104` | 获取错误信息 |
| `dlinfo` | `dlfcn/dlinfo.c:37-114` | 查询库信息 |
| `dlmopen` | `dlfcn/dlmopen.c:45-108` | 命名空间加载 |
| `dl_iterate_phdr` | `elf/dl-iteratephdr.c:31-84` | 遍历所有已加载对象 |

---

## 二、dlopen

### 调用路径

```
dlopen(filename, flags)
  → ___dlopen()                    [dlfcn/dlopen.c:75-103]
    → _dlerror_run(dlopen_doit)    [错误捕获包装]
      → dlopen_doit()              [dlfcn/dlopen.c:46-60]
        → _dl_open(filename, mode, caller, nsid, argc, argv, env)
                                   [elf/dl-open.c:812]
```

### _dl_open 核心流程

**源文件**: `elf/dl-open.c:513-748`

```
dl_open_worker_begin(filename, mode):
  │
  ├─ 1. 查找/映射目标文件
  │     _dl_map_new_object(filename, ...)
  │     如果 RTLD_NOLOAD: 仅查找已加载的，不实际加载
  │     elf/dl-open.c:532-544
  │
  ├─ 2. 更新引用计数
  │     new->l_direct_opencount++
  │     如果已加载: 直接返回 handle
  │     elf/dl-open.c:550-600
  │
  ├─ 3. 加载依赖
  │     _dl_map_object_deps(new, ...)  // BFS 递归
  │     elf/dl-open.c:603-610
  │
  ├─ 4. 版本检查
  │     _dl_open_check(new)
  │     elf/dl-open.c:612-628
  │
  ├─ 5. 重定位
  │     reloc_mode = RTLD_NOW ? eager : lazy
  │     _dl_relocate_object(dep, scope, reloc_mode)
  │     elf/dl-open.c:634-683
  │
  ├─ 6. 作用域/TLS/NODELETE 最终确定
  │     RTLD_GLOBAL: 加入全局搜索作用域
  │     RTLD_NODELETE: 标记不可卸载
  │     elf/dl-open.c:685-713
  │
  └─ 7. 运行构造函数
        _dl_init(new, ...)
        elf/dl-open.c:717-747
```

### dlopen 标志

**源文件**: `dlfcn/dlopen.c:51-53`，`bits/dlfcn.h:23-41`

| 标志 | 值 | 说明 |
|------|-----|------|
| `RTLD_LAZY` | 0x01 | 延迟绑定（PLT 首次调用时解析） |
| `RTLD_NOW` | 0x02 | 立即绑定所有符号 |
| `RTLD_GLOBAL` | 0x100 | 符号加入全局搜索范围 |
| `RTLD_LOCAL` | 0x00 | 符号仅在此 handle 范围内可见 |
| `RTLD_NODELETE` | 0x1000 | dlclose 不卸载 |
| `RTLD_NOLOAD` | 0x04 | 不加载，仅查询是否已加载 |
| `RTLD_DEEPBIND` | 0x08 | 库内符号查找优先于全局 |

### 线程安全

```
dlopen 使用两级锁:
  GL(dl_load_lock)       — 序列化 dlopen/dlclose 操作
  GL(dl_load_write_lock) — 保护 link_map 链表修改
  GL(dl_load_tls_lock)   — TLS 模块分配
```

---

## 三、dlsym

### 调用路径

**源文件**: `dlfcn/dlsym.c:35-69`，`elf/dl-sym.c:88-196`

```
dlsym(handle, symbol)
  → ___dlsym()
    → _dlerror_run(dlsym_doit)
      → _dl_sym(handle, name, caller_addr)
```

### 搜索语义

**源文件**: `elf/dl-sym.c:95-170`

```
_dl_sym(handle, name, who):
  caller_map = _dl_sym_find_caller_link_map(who)
  
  switch handle:
    case RTLD_DEFAULT:
      // 搜索全局作用域（所有 RTLD_GLOBAL 库 + 可执行文件）
      scope = 全局作用域
      result = _dl_lookup_symbol_x(name, caller_map, &ref, scope, ...)
    
    case RTLD_NEXT:
      // 从调用者之后的库开始搜索
      找到 caller_map 在全局链表中的位置
      scope = caller_map 之后的所有 map
      result = _dl_lookup_symbol_x(name, ..., scope, ...)
    
    default:
      // handle 是具体的 link_map*
      scope = handle->l_local_scope
      result = _dl_lookup_symbol_x(name, ..., scope, ...)
  
  // IFUNC 处理
  if result.sym->type == STT_GNU_IFUNC:
    return elf_ifunc_invoke(result.value)
  
  // TLS 处理
  if result.sym 是 TLS 符号:
    return tls_address(result)
  
  return result.value
```

### RTLD_DEFAULT vs RTLD_NEXT

```
加载顺序: [exec] → [preload] → [libA] → [libB] → [libc]

从 libA 中调用:
  dlsym(RTLD_DEFAULT, "foo"):
    搜索: exec → preload → libA → libB → libc (全部)
  
  dlsym(RTLD_NEXT, "foo"):
    搜索: libB → libc (libA 之后)
    用途: 包装函数调用原始实现
```

### 调用者识别

**源文件**: `elf/dl-sym.c:88-98`

```
// 通过返回地址确定调用者
who = __builtin_return_address(0)
caller_map = _dl_sym_find_caller_link_map(who)
// 遍历 link_map 找到包含 who 地址的映射
```

---

## 四、dlclose

### 调用路径

**源文件**: `elf/dl-close.c:114-758`（`_dl_close_worker`），`762-801`（`_dl_close`）

```
dlclose(handle)
  → _dl_close(handle)
```

### _dl_close 流程

```
_dl_close(map):
  │
  ├─ 1. 递减引用计数
  │     map->l_direct_opencount--
  │     if count > 0: return (仍有引用)
  │     elf/dl-close.c:114-136
  │
  ├─ 2. 检查是否可卸载
  │     if RTLD_NODELETE: return
  │     if 有 TLS 析构器: 标记 NODELETE, return
  │     elf/dl-close.c:181-189
  │
  ├─ 3. 确定卸载集合（该库 + 无其他引用的依赖）
  │     BFS 标记所有可卸载的 map
  │
  ├─ 4. 运行析构函数
  │     _dl_call_fini(map)  // DT_FINI_ARRAY + DT_FINI
  │     elf/dl-close.c:249-270
  │
  ├─ 5. 从搜索作用域中移除
  │     更新所有引用此 map 的 scope
  │
  ├─ 6. 清理 unique 符号表项
  │     elf/dl-close.c:606-629
  │
  ├─ 7. munmap 所有映射段
  │     DL_UNMAP(map)
  │     elf/dl-close.c:631-710
  │
  └─ 8. 释放 link_map 结构
        free(map)
        elf/dl-close.c:742-746
```

### 引用计数规则

```
dlopen("libA.so")  → libA.opencount = 1
dlopen("libA.so")  → libA.opencount = 2  (同一对象)
dlclose(handleA)   → libA.opencount = 1  (不卸载)
dlclose(handleA)   → libA.opencount = 0  (执行卸载)
```

### 不可卸载的情况

| 条件 | 原因 |
|------|------|
| `RTLD_NODELETE` | 显式标记 |
| `STB_GNU_UNIQUE` 符号 | 全局唯一不可重复创建 |
| 仍有 TLS 析构器 | 其他线程可能持有 TLS 引用 |
| 其他库依赖它 | 引用计数 > 0 |

---

## 五、dladdr — 地址反查

**源文件**: `elf/dl-addr.c:117-138`（内部函数 `_dl_addr`，公共入口 `dlfcn/dladdr.c:23-31`）

```
dladdr(addr, info):
  lock(dl_load_lock)
  
  // 遍历所有 link_map 找到包含 addr 的映射
  for each map in link_map_list:
    if map->l_addr <= addr < map->l_addr + map->l_map_end:
      info->dli_fname = map->l_name
      info->dli_fbase = map->l_addr
      
      // 在该 map 的 symtab 中找最近的符号
      best_sym = 二分/遍历 symtab 找 st_value <= addr 的最大者
      info->dli_sname = strtab + best_sym->st_name
      info->dli_saddr = map->l_addr + best_sym->st_value
      
      unlock; return 1
  
  unlock; return 0
```

用途: 堆栈回溯、地址→函数名映射

---

## 六、dl_iterate_phdr

**源文件**: `elf/dl-iteratephdr.c:31-84`

```c
int dl_iterate_phdr(callback, data):
  lock(dl_load_write_lock)
  
  for each map in namespace[ns]:
    struct dl_phdr_info info = {
      .dlpi_addr = map->l_addr,
      .dlpi_name = map->l_name,
      .dlpi_phdr = map->l_phdr,
      .dlpi_phnum = map->l_phnum,
      .dlpi_adds = GL(dl_load_adds),
      .dlpi_subs = GL(dl_load_subs),
      .dlpi_tls_modid = map->l_tls_modid,
      .dlpi_tls_data = TLS block pointer,
    };
    
    ret = callback(&info, sizeof(info), data);
    if (ret != 0) break;
  
  unlock; return ret;
```

**用途**:
- `libgcc_s`: 查找 `.eh_frame` 段进行异常处理
- `libunwind`: 栈回溯
- 安全工具: 枚举所有加载的库

---

## 七、dlmopen — 命名空间隔离

**源文件**: `dlfcn/dlmopen.c:45-108`

```
dlmopen(nsid, filename, flags):
  if nsid == LM_ID_NEWLM:
    创建新的独立命名空间
  elif nsid == LM_ID_BASE:
    使用默认命名空间（等同 dlopen）
  
  // 新命名空间中的库有独立的:
  // - link_map 链表
  // - 搜索作用域
  // - libc 副本（独立的 malloc/stdio/...状态）
```

### 命名空间用途

| 场景 | 说明 |
|------|------|
| 插件隔离 | 不同插件的全局变量互不干扰 |
| 多版本共存 | 同一库的不同版本同时加载 |
| 安全沙箱 | 限制库的符号可见性 |

### 限制

- `DL_NNS` 定义最大命名空间数（通常 16）
- `RTLD_GLOBAL` 不能跨命名空间
- 每个命名空间有独立的 libc 实例（内存开销）

---

## 八、错误处理

### dlerror

**源文件**: `dlfcn/dlerror.c:31-197`

```
dlerror():
  // 线程本地错误字符串
  result = TLS_error_string
  TLS_error_string = NULL  // 一次性消费
  return result

内部错误设置:
  _dlerror_run(func, args):
    try:
      func(args)
      error = NULL
    catch (dl_exception):
      error = 格式化错误消息
      存入 TLS
```

- 错误信息是线程本地的
- 每次 `dlerror()` 调用会清除错误（只能读一次）
- 内部使用 `_dl_catch_error` / `_dl_catch_exception` 实现

---

## 九、完整 dlopen 时序图

```
应用程序            dlopen              ld.so 内部
   │                  │                     │
   │ dlopen("x.so")  │                     │
   │─────────────────→│                     │
   │                  │ _dl_open()          │
   │                  │─────────────────────→│
   │                  │                     │ 1. 查找文件路径
   │                  │                     │ 2. open() + 读 ELF 头
   │                  │                     │ 3. mmap PT_LOAD 段
   │                  │                     │ 4. 创建 link_map
   │                  │                     │ 5. 解析 .dynamic
   │                  │                     │ 6. 递归加载 DT_NEEDED
   │                  │                     │ 7. 重定位
   │                  │                     │ 8. mprotect RELRO
   │                  │                     │ 9. 调用 DT_INIT_ARRAY
   │                  │ return handle       │
   │                  │←─────────────────────│
   │ handle           │                     │
   │←─────────────────│                     │
   │                  │                     │
   │ dlsym(h,"func") │                     │
   │─────────────────→│ _dl_sym()           │
   │                  │─────────────────────→│
   │                  │                     │ GNU hash 查找
   │                  │ return addr         │
   │                  │←─────────────────────│
   │ func_ptr         │                     │
   │←─────────────────│                     │
```

---

## 十、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `dlfcn/dlopen.c:46-103` | dlopen 入口 + 标志验证 |
| `elf/dl-open.c:513-748` | `dl_open_worker_begin` 核心 |
| `elf/dl-open.c:812` | `_dl_open` 入口 |
| `dlfcn/dlsym.c:35-69` | dlsym 入口 |
| `elf/dl-sym.c:88-196` | `_dl_sym` 查找逻辑 |
| `elf/dl-close.c:114-758` | `_dl_close_worker` 卸载 |
| `elf/dl-close.c:249-270` | 析构函数调用 |
| `elf/dl-close.c:631-710` | munmap + 清理 |
| `elf/dl-addr.c:117-138` | `_dl_addr` 地址反查 |
| `elf/dl-iteratephdr.c:31-84` | `dl_iterate_phdr` |
| `dlfcn/dlmopen.c:45-108` | dlmopen 命名空间 |
| `dlfcn/dlerror.c:31-197` | 错误管理 |
| `dlfcn/dlinfo.c:37-114` | dlinfo 查询 |
