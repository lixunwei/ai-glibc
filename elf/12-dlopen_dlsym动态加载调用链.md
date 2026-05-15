# dlopen/dlsym/dlclose 动态加载调用链分析

> 基于 clangd LSP 语义分析 (glibc 2.43.9000, AArch64)

## 概述

`dlopen`/`dlsym`/`dlclose` 是用户空间动态加载共享库的标准接口。本文档分析从用户 API 到 ELF 文件映射、符号查找、重定位执行的完整调用路径。

---

## 1. dlopen 调用链

### 1.1 用户入口: `___dlopen()` → `dlopen_doit()` [dlfcn/dlopen.c:47-76]

```
用户调用 dlopen("libfoo.so", RTLD_LAZY)
│
├── dlopen_doit()                   [dlfcn/dlopen.c:47]
│   ├── 验证 mode 标志合法性
│   ├── __dcgettext()               [intl/dcgettext.c:35] — 错误消息国际化
│   ├── _dl_signal_error()          [elf/dl-catch.c:112] — 非法 mode 报错
│   └── _dl_open()                  [elf/dl-open.c:812] — ★核心打开逻辑
│
└── 通过 _dlerror_run() 执行，异常被捕获转为 dlerror() 字符串
```

### 1.2 核心打开: `_dl_open()` [elf/dl-open.c:812]

顶层协调函数，管理加载锁和错误恢复：

```
_dl_open(file, mode, caller_dlopen, nsid, argc, argv, env)
├── __pthread_mutex_lock()          [libc-lockP.h:196] — 获取全局加载锁
├── _dl_signal_error()              [dl-catch.c:112] — 参数校验
├── _dl_find_object()               [dl-find_object.c:23] — 检查调用者所属对象
├── strchr(file, '/')               — 判断是否含路径
├── _dl_lookup_map()                [dl-load.c:1897] — 按名搜索已加载列表
├── is_already_fully_open()         [dl-open.c:500] — 已打开则增加引用计数
│
├── _dl_catch_exception()           [dl-catch.c:206] — 异常保护
│   └── dl_open_worker()            [dl-open.c:751] — ★加载执行体
│
├── [失败回滚]
│   ├── _dl_close_worker()          [dl-close.c:114] — 清理部分加载
│   └── _dl_signal_exception()      [dl-catch.c:94] — 传播错误
│
├── _dl_debug_update()              [dl-debug.c:93] — 更新 r_debug
├── _dl_unload_cache()              [dl-cache.c:507] — 释放 ldconfig 缓存
└── __pthread_mutex_unlock()        — 释放加载锁
```

### 1.3 加载工作者: `dl_open_worker()` [elf/dl-open.c:751]

两阶段加载：先映射所有对象，再初始化：

```
dl_open_worker(args)
├── __pthread_mutex_lock()          — TLS 锁
├── _dl_catch_exception()
│   └── dl_open_worker_begin()      [dl-open.c:513] — ★阶段1: 映射+重定位
├── __pthread_mutex_unlock()
│
├── _dl_debug_change_state()        [dl-debug.c:103] — RT_ADD 状态
├── _dl_debug_update()              [dl-debug.c:93] — 通知 GDB
│
├── _dl_catch_exception()
│   └── call_dl_init()              [dl-open.c:489] — ★阶段2: 构造函数
│       └── _dl_init()              [dl-init.c:80] — 执行 .init + .init_array
│
├── [失败处理]
│   └── _dl_signal_exception()      [dl-catch.c:94]
│
├── add_to_global_update()          [dl-open.c:172] — 加入全局 scope
└── _dl_debug_printf()              — LD_DEBUG 输出
```

### 1.4 映射与重定位: `dl_open_worker_begin()` [elf/dl-open.c:513]

核心加载阶段，包含文件查找、ELF 映射、依赖解析、重定位：

```
dl_open_worker_begin(args)
│
│ ===== 阶段 A: 文件查找与映射 =====
│
├── _dl_debug_initialize()          [dl-debug.c:119] — 调试支持
├── _dl_map_new_object()            [dl-load.c:1931] — ★ELF 文件映射
│   ├── [搜索顺序] — 见 §3.1
│   ├── cache_rpath()               [dl-load.c:652] — DT_RPATH/DT_RUNPATH
│   ├── _dl_load_cache_lookup()     [dl-cache.c:385] — /etc/ld.so.cache
│   ├── open_path()                 [dl-load.c:1737] — 逐目录搜索
│   ├── open_verify()               [dl-load.c:1511] — 验证 ELF magic/class
│   ├── _dl_map_object_from_fd()    [dl-load.c:945] — ★内存映射
│   │   ├── _dl_get_file_id()       — fstat 获取 dev+ino
│   │   ├── _dl_file_id_match_p()   — 去重检查
│   │   ├── _dl_new_object()        [dl-object.c:57] — 创建 link_map
│   │   ├── _dl_map_segments()      [dl-map-segments.h:76] — mmap LOAD 段
│   │   ├── elf_get_dynamic_info()  [get-dynamic-info.h:29] — 解析 .dynamic
│   │   ├── _dl_setup_hash()        [dl-setup_hash.c:24] — 初始化 hash 表
│   │   ├── _dl_assign_tls_modid()  [dl-tls.c:161] — 分配 TLS 模块 ID
│   │   └── elf_machine_runtime_setup() — 设置 PLT 跳板
│   ├── _dl_new_object()            [dl-object.c:57] — 创建 link_map
│   └── _dl_add_to_namespace_list() [dl-object.c:30] — 加入命名空间
│
│ ===== 阶段 B: 依赖递归加载 =====
│
├── add_to_global_resize()          [dl-open.c:91] — 扩展全局 scope
├── add_to_global_update()          [dl-open.c:172] — 注册新对象
├── _dl_map_object_deps()           [dl-deps.c:140] — ★递归加载依赖
│   ├── [BFS 遍历 DT_NEEDED]
│   ├── openaux() / preload()       — 递归调用 _dl_map_new_object
│   ├── _dl_dst_substitute()        [dl-load.c:266] — $ORIGIN 等替换
│   ├── _dl_sort_maps()             [dl-sort-maps.c:296] — 拓扑排序
│   └── scratch_buffer_*            — 临时缓冲区管理
│
├── _dl_check_map_versions()        [dl-version.c:154] — 版本需求验证
├── _dl_open_check()                [dl-prop.h:38] — 架构属性检查
│
│ ===== 阶段 C: 重定位 =====
│
├── resize_scopes() / update_scopes()  — scope 更新
├── resize_tls_slotinfo() / update_tls_slotinfo()  — TLS 槽位更新
│
├── _dl_open_relocate_one_object()  [dl-open.c:448] — 对每个新对象执行
│   └── _dl_relocate_object()       [dl-reloc.c:320]
│       ├── _dl_relocate_object_no_relro()  [dl-reloc.c:184]
│       │   ├── elf_machine_runtime_setup() [dl-machine.h:64] — PLT GOT 初始化
│       │   ├── ELF_DYNAMIC_RELOCATE()      — 执行所有重定位
│       │   │   → 对每个 R_AARCH64_* 重定位:
│       │   │     → _dl_lookup_symbol_x() → do_lookup_x()
│       │   │     → 写入重定位地址
│       │   ├── __mprotect()        — 段权限恢复
│       │   └── calloc()            — 审计接口内存
│       └── _dl_protect_relro()     [dl-reloc.c:330]
│           — mprotect(.got, PROT_READ) (RELRO 保护)
│
│ ===== 阶段 D: 最终注册 =====
│
├── activate_nodelete()             [dl-open.c:421] — 标记 NODELETE
├── _dl_find_object_update()        [dl-find_object.c:808] — 更新快速查找
└── _dl_show_scope()                [dl-open.c:958] — LD_DEBUG=scopes
```

---

## 2. dlsym 调用链

### 2.1 用户入口: `___dlsym()` [dlfcn/dlsym.c:63]

```
用户调用 dlsym(handle, "function_name")
│
└── dlsym_implementation()          [dlfcn/dlsym.c:44]
    ├── __pthread_mutex_lock()      — 获取 dl 锁
    ├── _dlerror_run(dlsym_doit)    [dlerror.c:112] — 异常捕获框架
    │   └── dlsym_doit()            [dlfcn/dlsym.c:36]
    │       └── _dl_sym()           [elf/dl-sym.c:193]
    │           └── do_sym()        [elf/dl-sym.c:85] — ★核心符号查找
    └── __pthread_mutex_unlock()    — 释放锁
```

### 2.2 核心符号查找: `do_sym()` [elf/dl-sym.c:85]

```
do_sym(handle, name, who, ...)
├── _dl_sym_find_caller_link_map()  [dl-sym-post.h:22]
│   — 根据返回地址确定调用者所属 SO
│
├── [根据 handle 类型分支]
│   ├── RTLD_DEFAULT:
│   │   → _dl_lookup_symbol_x(name, caller_map, global_scope, ...)
│   │     — 搜索全局 scope (所有 RTLD_GLOBAL 的 SO)
│   │
│   ├── RTLD_NEXT:
│   │   → _dl_lookup_symbol_x(name, caller_map, caller_scope, NEXT, ...)
│   │     — 从调用者之后的 SO 开始搜索
│   │
│   └── 具体 handle:
│       → call_dl_lookup()          [dl-sym.c:76]
│         → _dl_catch_exception()
│           → _dl_lookup_symbol_x(name, handle_map, local_scope, ...)
│             — 仅在该 SO 及其依赖中搜索
│
├── _dl_signal_error()              — 符号未找到
│
└── _dl_sym_post()                  [dl-sym-post.h:37]
    — IFUNC 解析 + 审计回调 + 返回最终地址
```

### 2.3 符号查找引擎 (共享)

`_dl_lookup_symbol_x()` 和 `do_lookup_x()` 的详细分析见 `elf/11-_dl_fixup延迟绑定调用链.md` §3.1-3.2。

---

## 3. dlclose 调用链

### 3.1 用户入口: `_dl_close()` [elf/dl-close.c:762]

```
用户调用 dlclose(handle)
│
└── _dl_close(handle)               [dl-close.c:762]
    ├── __pthread_mutex_lock()      — 获取加载锁
    ├── _dl_close_worker()          [dl-close.c:114] — ★核心卸载逻辑
    ├── _dl_signal_error()          — 错误处理
    └── __pthread_mutex_unlock()    — 释放锁
```

### 3.2 卸载工作者: `_dl_close_worker()` [elf/dl-close.c:114]

```
_dl_close_worker(map, force)
│
│ ===== 阶段 1: 确定卸载集合 =====
│
├── 引用计数递减
├── _dl_sort_maps()                 [dl-sort-maps.c:296] — 拓扑排序(逆序)
├── [标记可卸载对象]
│   — 引用计数=0 且非 NODELETE
│
│ ===== 阶段 2: 执行析构函数 =====
│
├── _dl_call_fini()                 [dl-call_fini.c:23]
│   — 调用 .fini_array (逆序) + .fini
│
│ ===== 阶段 3: Scope 清理 =====
│
├── _dl_scope_free()                [dl-scope.c:25] — 释放 scope 数组
├── __thread_gscope_wait()          [dl-thread_gscope_wait.c:26]
│   — 等待所有线程退出临界区 (安全删除 scope)
│
│ ===== 阶段 4: TLS 清理 =====
│
├── remove_slotinfo()               [dl-close.c:45] — 释放 TLS 模块槽
│
│ ===== 阶段 5: 内存映射释放 =====
│
├── _dl_debug_change_state()        [dl-debug.c:103] — RT_DELETE 通知 GDB
├── _dl_debug_update()              [dl-debug.c:93]
├── _dl_unmap()                     [tlsdesc.c:31] — munmap 所有段
├── _dl_find_object_dlclose()       [dl-find_object.c:835] — 更新快速查找
│
│ ===== 阶段 6: 元数据释放 =====
│
├── free() × 多次                   — 释放 link_map 及附属结构
│   — l_name, l_origin, l_rpath, DT_NEEDED 字符串等
│
└── _dl_debug_change_state()        — RT_CONSISTENT 恢复
```

---

## 4. 共享库搜索顺序

### 4.1 `_dl_map_new_object` 中的搜索路径

```
dlopen("libfoo.so", RTLD_LAZY) 的搜索顺序:

1. DT_RPATH (调用者的 RPATH) — 已废弃，优先级最高
   → cache_rpath(caller_map->l_info[DT_RPATH])

2. LD_LIBRARY_PATH 环境变量
   → open_path(env_path)

3. DT_RUNPATH (调用者的 RUNPATH) — 推荐方式
   → cache_rpath(caller_map->l_info[DT_RUNPATH])

4. /etc/ld.so.cache (ldconfig 缓存)
   → _dl_load_cache_lookup("libfoo.so")

5. 系统默认路径
   → open_path("/lib:/usr/lib" 或架构特定路径)
   → open_path("/lib/aarch64-linux-gnu:/usr/lib/aarch64-linux-gnu")

注: 如果 file 包含 '/', 则跳过所有搜索，直接 open()
```

### 4.2 `open_verify()` 验证步骤

```
1. read() ELF header (前 64 字节)
2. 验证 ELF magic: \x7fELF
3. 验证 EI_CLASS (64-bit for AArch64)
4. 验证 EI_DATA (little-endian)
5. 验证 e_machine (EM_AARCH64 = 183)
6. 验证 e_type (ET_DYN)
7. 检查 PT_NOTE 中的 ABI 标签
```

---

## 5. 重定位执行细节

### 5.1 `_dl_relocate_object_no_relro` [elf/dl-reloc.c:184]

```
_dl_relocate_object_no_relro(l, scope, lazy, consider_profiling)
├── elf_machine_runtime_setup()     [dl-machine.h:64]
│   — 设置 GOT[1]=link_map, GOT[2]=_dl_runtime_resolve
│   — (仅 RTLD_LAZY 时设置 PLT 跳板)
│
├── [如果有 .text 段需要写入]
│   └── __mprotect(textrelro, PROT_READ|PROT_WRITE)
│
├── ELF_DYNAMIC_RELOCATE(l, lazy)   — 宏展开为:
│   ├── elf_dynamic_do_Rela()       — 处理 .rela.dyn
│   │   → 对每个重定位条目:
│   │     switch (ELF64_R_TYPE(r_info)):
│   │       R_AARCH64_GLOB_DAT:    *reloc = sym_addr
│   │       R_AARCH64_RELATIVE:    *reloc = l_addr + addend
│   │       R_AARCH64_ABS64:       *reloc = sym_addr + addend
│   │       R_AARCH64_COPY:        memcpy(reloc, sym_addr, size)
│   │       R_AARCH64_TLS_DTPMOD:  *reloc = module_id
│   │       R_AARCH64_IRELATIVE:   *reloc = ifunc_resolver()
│   │       ... (更多类型)
│   │     符号解析: _dl_lookup_symbol_x() → do_lookup_x()
│   │
│   └── elf_dynamic_do_Rela(.rela.plt)  — 处理 PLT 重定位
│       → RTLD_LAZY: 设置为 PLT 默认跳转地址
│       → RTLD_NOW: 立即解析每个 PLT 条目
│
└── [恢复 .text 段权限]
    └── __mprotect(textrelro, PROT_READ|PROT_EXEC)
```

---

## 6. 线程安全机制

### 6.1 锁层次

```
加载锁 (__rtld_lock_recursive):
  └── _dl_open() / _dl_close() 级别
      — 全局加载互斥锁，防止并发 dlopen/dlclose

TLS 锁 (dl_load_tls_lock):
  └── dl_open_worker() 中
      — 保护 TLS 模块 ID 分配

线程安全 scope 更新:
  └── __thread_gscope_wait()
      — 等待所有线程退出 DL 临界区后才修改 scope
      — 类似 RCU 的延迟释放机制
```

### 6.2 异常安全

```
dlopen 错误恢复:
  _dl_catch_exception() 包裹 dl_open_worker_begin
  → 失败时 _dl_close_worker() 清理已映射对象
  → 保证不泄漏 mmap/link_map/TLS 资源
```

---

## 7. dlopen 与 ld.so 启动的关系

| 阶段 | ld.so 启动时 | dlopen 时 |
|------|-------------|-----------|
| 文件查找 | `_dl_map_new_object` | 相同函数 |
| ELF 映射 | `_dl_map_object_from_fd` | 相同函数 |
| 依赖解析 | `_dl_map_object_deps` | 相同函数 |
| 重定位 | dl_main 中批量执行 | `_dl_open_relocate_one_object` |
| 初始化 | `_dl_init` | `call_dl_init → _dl_init` |
| scope 管理 | 直接设置 | `resize_scopes / update_scopes` |
| GDB 通知 | `_dl_debug_state` | `_dl_debug_change_state` |

**关键区别：** dlopen 需要处理已运行程序中的并发问题（线程安全 scope 更新），而 ld.so 启动时是单线程环境。

---

## 8. 源码位置索引

| 函数 | 文件 | 行号 |
|------|------|------|
| `___dlopen` | dlfcn/dlopen.c | 76 |
| `dlopen_doit` | dlfcn/dlopen.c | 47 |
| `_dl_open` | elf/dl-open.c | 812 |
| `dl_open_worker` | elf/dl-open.c | 751 |
| `dl_open_worker_begin` | elf/dl-open.c | 513 |
| `_dl_open_relocate_one_object` | elf/dl-open.c | 448 |
| `call_dl_init` | elf/dl-open.c | 489 |
| `add_to_global_resize` | elf/dl-open.c | 91 |
| `add_to_global_update` | elf/dl-open.c | 172 |
| `_dl_map_new_object` | elf/dl-load.c | 1931 |
| `_dl_map_object_from_fd` | elf/dl-load.c | 945 |
| `open_path` | elf/dl-load.c | 1737 |
| `open_verify` | elf/dl-load.c | 1511 |
| `_dl_map_object_deps` | elf/dl-deps.c | 140 |
| `_dl_relocate_object` | elf/dl-reloc.c | 320 |
| `_dl_relocate_object_no_relro` | elf/dl-reloc.c | 184 |
| `_dl_protect_relro` | elf/dl-reloc.c | 330 |
| `___dlsym` | dlfcn/dlsym.c | 63 |
| `dlsym_implementation` | dlfcn/dlsym.c | 44 |
| `dlsym_doit` | dlfcn/dlsym.c | 36 |
| `_dl_sym` | elf/dl-sym.c | 193 |
| `do_sym` | elf/dl-sym.c | 85 |
| `_dl_close` | elf/dl-close.c | 762 |
| `_dl_close_worker` | elf/dl-close.c | 114 |
| `_dl_call_fini` | elf/dl-call_fini.c | 23 |
| `_dl_sort_maps` | elf/dl-sort-maps.c | 296 |
| `_dl_check_map_versions` | elf/dl-version.c | 154 |
| `_dl_init` | elf/dl-init.c | 80 |
| `_dl_load_cache_lookup` | elf/dl-cache.c | 385 |
