# 动态链接器 (ld.so) 深度分析

> 基于 glibc 2.43.9000 ELF 动态链接器源码分析

---

## 文档索引

| 文档 | 内容 | 关键主题 |
|------|------|----------|
| [01-启动与加载.md](01-启动与加载.md) | ld.so 自举与 ELF 加载 | 自重定位、_dl_main、PT_LOAD 映射、link_map |
| [02-符号解析.md](02-符号解析.md) | 符号表与查找算法 | GNU hash、bloom filter、版本化、IFUNC |
| [03-重定位与绑定.md](03-重定位与绑定.md) | GOT/PLT 与延迟绑定 | 重定位类型、_dl_fixup、RELRO、初始化顺序 |
| [04-dlopen与运行时加载.md](04-dlopen与运行时加载.md) | 运行时动态加载 API | dlopen/dlsym/dlclose、引用计数、命名空间 |
| [05-AArch64重定位与绑定.md](05-AArch64重定位与绑定.md) | ARM64 平台重定位机制 | PLT/GOT 格式、TLSDESC、BTI/PAC、Variant PCS |
| [06-vDSO机制分析.md](06-vDSO机制分析.md) | vDSO 虚拟共享库机制 | 内核映射、符号查找、clock_gettime 加速、IFUNC |
| [07-dlopen搜索与dladdr反查.md](07-dlopen搜索与dladdr反查.md) | dlopen/dladdr/dl_iterate_phdr 深度分析 | 库搜索顺序（RPATH→RUNPATH→cache）、DST 替换、ld.so.cache 二分搜索、dev/ino 去重、dladdr 线性扫描算法、dl_iterate_phdr 遍历机制、dlinfo 元信息查询 |
| [08-ldconfig与ld.so.cache管理.md](08-ldconfig与ld.so.cache管理.md) | ldconfig 工具与缓存管理机制 | ldconfig 主流程、/etc/ld.so.conf 解析、目录扫描与 ELF soname 提取、ld.so.cache 新旧格式、排序与原子写入、运行时二分搜索、glibc-hwcaps 优先级、辅助缓存 |

---

## 动态链接总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    程序启动                                       │
│  kernel execve → 映射 ELF + ld.so → 跳转 ld.so _start          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              ld.so 自举 (Bootstrap)                       │    │
│  │  _dl_start: 自重定位 → _dl_main: 加载所有依赖           │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                            │                                     │
│  ┌─────────────────────────▼───────────────────────────────┐    │
│  │              依赖加载                                     │    │
│  │  解析 DT_NEEDED → BFS 遍历 → mmap PT_LOAD 段            │    │
│  │  构建 link_map 链表 + 搜索作用域                         │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                            │                                     │
│  ┌─────────────────────────▼───────────────────────────────┐    │
│  │              符号解析与重定位                              │    │
│  │  R_*_RELATIVE (快速) → R_*_GLOB_DAT → R_*_JUMP_SLOT     │    │
│  │  GOT/PLT 填充 + RELRO mprotect                          │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                            │                                     │
│  ┌─────────────────────────▼───────────────────────────────┐    │
│  │              初始化                                        │    │
│  │  _dl_init: DT_PREINIT_ARRAY → DT_INIT → DT_INIT_ARRAY  │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│              跳转应用程序 main()                                 │
├─────────────────────────────────────────────────────────────────┤
│              运行时动态加载                                       │
│  dlopen → 加载+重定位+初始化  |  dlsym → 符号查找               │
│  dlclose → 析构+卸载          |  dladdr → 地址→符号反查         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心数据结构

| 结构 | 定义位置 | 作用 |
|------|---------|------|
| `struct link_map` | `include/link.h:95-268` | 描述一个已加载的 ELF 对象 |
| `Elf64_Sym` | `elf/elf.h:518-538` | 符号表条目 |
| `Elf64_Rela` | `elf/elf.h` | 重定位条目（带 addend） |
| `Elf64_Phdr` | `elf/elf.h` | 程序头表条目 |
| `r_scope_elem` | `include/link.h:258-268` | 搜索作用域 |

---

## 源文件布局

| 文件 | 行数 | 职责 |
|------|------|------|
| `elf/rtld.c` | ~2200 | 链接器主逻辑：自举 + _dl_main |
| `elf/dl-load.c` | ~2200 | ELF 文件加载 + mmap |
| `elf/dl-lookup.c` | ~900 | 符号查找算法 |
| `elf/dl-reloc.c` | ~350 | 重定位引擎 |
| `elf/dl-runtime.c` | ~170 | PLT 延迟绑定 (_dl_fixup) |
| `elf/dl-open.c` | ~850 | dlopen 实现 |
| `elf/dl-close.c` | ~750 | dlclose + 卸载 |
| `elf/dl-sym.c` | ~200 | dlsym 实现 |
| `elf/dl-deps.c` | ~300 | 依赖图 BFS |
| `elf/dl-init.c` | ~130 | 构造函数调用 |
| `elf/dl-fini.c` | ~120 | 析构函数调用 |
| `elf/dl-version.c` | ~280 | 符号版本化 |
| `sysdeps/x86_64/dl-machine.h` | ~470 | x86_64 架构特定：重定位 + PLT |
| `sysdeps/x86_64/dl-trampoline.h` | ~130 | PLT resolver 汇编 |

---

## 关键概念

| 概念 | 说明 |
|------|------|
| link_map | 每个加载的 .so / 可执行文件对应一个，形成链表 |
| 搜索作用域 (scope) | 符号查找时遍历的 link_map 有序集合 |
| GOT (Global Offset Table) | 存放全局数据/函数地址的表 |
| PLT (Procedure Linkage Table) | 函数调用的间接跳转表，支持延迟绑定 |
| RELRO | 重定位完成后将 GOT 设为只读（安全加固） |
| GNU hash | 基于 bloom filter 的 O(1) 符号查找 |
| IFUNC | 运行时根据 CPU 特性选择最优函数实现 |
| 符号版本化 | 同一符号可有多个版本共存（ABI 兼容） |
