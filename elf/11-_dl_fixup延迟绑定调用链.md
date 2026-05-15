# _dl_fixup 延迟绑定调用链分析

> 基于 clangd LSP 语义分析 (glibc 2.43.9000, AArch64)

## 概述

当程序首次调用一个外部共享库函数时，PLT (Procedure Linkage Table) 存根代码跳转到动态链接器的 `_dl_fixup()` 完成符号解析和重定位。本文档展示从 PLT 触发到函数地址写入 GOT 的完整路径。

---

## 1. 触发机制

### 1.1 PLT 调桩流程 (AArch64)

```
用户代码调用 printf():
  → PLT 存根 (.plt 段):
      adrp  x16, GOT_PAGE
      ldr   x17, [x16, GOT_OFFSET]  ; 首次: 指向 PLT[0]
      br    x17
  → PLT[0] (公共入口):
      push  {ip0, lr}               ; 保存调用者信息
      ldr   x16, [GOT+8]           ; _dl_runtime_resolve 地址
      br    x16                     ; 跳转到 resolver
  → _dl_runtime_resolve (汇编):
      保存所有寄存器
      调用 _dl_fixup(link_map, reloc_index)
      恢复寄存器
      跳转到解析后的地址
```

### 1.2 调用上下文

`_dl_fixup` 由汇编 trampoline `_dl_runtime_resolve` 调用（因此 clangd 显示 incoming = 0，因为调用者是汇编代码）。

---

## 2. `_dl_fixup()` 调用链 [elf/dl-runtime.c:41]

```
_dl_fixup(struct link_map *l, ElfW(Word) reloc_arg)
│
├── dl_relocate_ld()                [ldsodefs.h:76]
│   call sites: line 48, 49, 51, 54, 74
│   — 获取重定位相关的 link_map 字段 (l_info[])
│
├── reloc_offset()                  [dl-runtime.h:21]
│   call site: line 55
│   — 计算重定位条目偏移 (reloc_arg × sizeof(ElfW(Rela)))
│
├── __assert_single_arg()           [assert.h:114]
│   call site: line 63
│   — 断言: ELF_RTYPE_CLASS_PLT 匹配
│
├── _dl_lookup_symbol_x()           [dl-lookup.c:762] ★核心符号查找
│   call site: line 95
│   — 在所有已加载 SO 的符号表中搜索
│
├── elf_machine_plt_value()         [dl-machine.h:152]
│   call site: line 121
│   — 计算最终函数地址 (st_value + l_addr)
│
├── elf_ifunc_invoke()              [dl-irel.h:48]
│   call site: line 125
│   — 如果是 IFUNC: 调用解析器选择最优实现
│
└── elf_machine_fixup_plt()         [dl-machine.h:141]
    call site: line 162
    — ★将解析后的地址写入 GOT 条目
```

---

## 3. 核心子函数详解

### 3.1 `_dl_lookup_symbol_x()` [elf/dl-lookup.c:762]

符号查找的核心，遍历 scope 列表搜索符号：

```
_dl_lookup_symbol_x(name, undef_map, ref, symbol_scope, ...)
├── _dl_new_hash()                  [dl-new-hash.h:67]
│   — 计算 GNU hash (djb2 变体)
│
├── do_lookup_x()                   [dl-lookup.c:339] ★搜索引擎
│   ├── dl_relocate_ld()            [ldsodefs.h:76] — 获取 SO 符号表
│   ├── _dl_elf_hash()              [dl-hash.h:28] — 旧式 SYSV hash
│   ├── check_match()              [dl-lookup.c:59] — 验证符号匹配
│   ├── dl_symbol_visibility_binds_local_p()  [ldsodefs.h:149]
│   ├── _dl_check_protected_symbol()  [dl-protected.h:23]
│   └── do_lookup_unique()          [dl-lookup.c:208] — STB_GNU_UNIQUE 处理
│
├── add_dependency()                [dl-lookup.c:524]
│   — 记录 SO 间依赖关系 (DT_NEEDED 级别)
│
├── _dl_lookup_symbol_x()           [递归调用自身]
│   — scope 切换后重新查找 (RTLD_NEXT 等场景)
│
├── _dl_exception_create_format()   [dl-exception.c:105]
├── _dl_signal_cexception()         [ldsodefs.h:856]
├── _dl_exception_free()            [dl-exception.c:249]
│   — 符号未找到时的异常处理
│
└── _dl_debug_printf[_c]()          [dl-printf.c:251/263]
    — LD_DEBUG 调试输出
```

### 3.2 `do_lookup_x()` [elf/dl-lookup.c:339]

实际的符号搜索循环：

```
搜索策略:
for (scope 中的每个 link_map):
  1. 检查 GNU hash table (DT_GNU_HASH)
     → bucket[hash % nbuckets]
     → chain[] 线性探测
  2. 若无 GNU hash，回退 SYSV hash (DT_HASH)
     → _dl_elf_hash(name)
  3. check_match(): 验证名称 + 版本匹配
  4. 处理 STB_GNU_UNIQUE (全局唯一符号)
  5. 处理 protected visibility
```

### 3.3 `elf_machine_fixup_plt()` [sysdeps/aarch64/dl-machine.h:141]

AArch64 架构特定的 GOT 写入：

```c
// 将解析后的地址写入 GOT
*reloc_addr = value;
// 此后 PLT 存根直接跳转到目标函数，不再经过 resolver
```

### 3.4 `elf_ifunc_invoke()` [sysdeps/aarch64/dl-irel.h:48]

GNU IFUNC 机制：运行时选择最优实现：

```c
// 例如: memcpy 根据 CPU 特性选择 NEON/SVE 版本
typedef ElfW(Addr) (*ifunc_resolver_t)(uint64_t hwcap);
return resolver(hwcap);
```

---

## 4. 完整执行流

### 4.1 首次调用外部函数

```
用户调用 printf("hello")
  → PLT[printf] → GOT[printf] → PLT[0]
  → _dl_runtime_resolve (保存所有寄存器)
  → _dl_fixup(libc_link_map, printf_reloc_idx)
    ├── 从 .rela.plt 获取重定位条目
    │   sym_idx = ELF64_R_SYM(rela->r_info)
    │   sym_name = strtab + symtab[sym_idx].st_name → "printf"
    ├── _dl_lookup_symbol_x("printf", ...)
    │   → do_lookup_x()
    │     → GNU hash: hash("printf") = 0x...
    │     → 遍历 libc.so 的 .gnu.hash
    │     → 找到 symtab 条目: st_value = 0x... (偏移)
    │   → result.s->st_value + result.m->l_addr
    │   → value = libc_base + printf_offset
    ├── elf_machine_fixup_plt(*GOT_entry, value)
    │   — GOT[printf] = 实际 printf 地址
    └── 返回 value
  → _dl_runtime_resolve (恢复寄存器, 跳转 value)
  → printf("hello") 执行

第二次调用 printf():
  → PLT[printf] → GOT[printf] → 直接跳转到 printf (无 resolver)
```

### 4.2 IFUNC 解析流程

```
用户调用 memcpy()
  → _dl_fixup()
    → 发现 reloc type = R_AARCH64_IRELATIVE
    → _dl_lookup_symbol_x("memcpy") → 找到 resolver 函数
    → elf_ifunc_invoke(resolver)
      → resolver(HWCAP_ATOMICS | HWCAP_SVE | ...)
      → 返回 __memcpy_sve 或 __memcpy_generic
    → elf_machine_fixup_plt(GOT, 选定实现地址)
```

---

## 5. 性能考量

### 5.1 GNU Hash vs SYSV Hash

| 特性 | GNU Hash | SYSV Hash |
|------|----------|-----------|
| 时间复杂度 | O(1) 平均 | O(n/buckets) |
| Bloom filter | ✓ 快速排除 | ✗ |
| 缓存友好 | ✓ (线性内存) | △ |
| 现代 SO | 默认使用 | 兼容保留 |

### 5.2 绑定模式

| 模式 | 触发时机 | 性能特征 |
|------|----------|----------|
| Lazy (默认) | 首次调用 | 启动快，首次调用慢 |
| NOW (LD_BIND_NOW) | 加载时全部解析 | 启动慢，运行时无开销 |
| `-z relro` + NOW | 加载时 + GOT 只读 | 安全 + 性能确定性 |

---

## 6. 安全机制

### 6.1 RELRO (Relocation Read-Only)

```
Full RELRO 流程:
  ld.so 加载时:
    1. 解析所有 PLT 重定位 (NOW 语义)
    2. mprotect(.got, PROT_READ) → GOT 变为只读
    3. 攻击者无法覆盖 GOT 条目
```

### 6.2 符号版本控制

`check_match()` 中验证:
- 符号名称匹配
- 版本标签匹配 (e.g., `GLIBC_2.34`)
- 防止链接到错误版本的符号

---

## 7. 源码位置索引

| 函数 | 文件 | 行号 |
|------|------|------|
| `_dl_fixup` | elf/dl-runtime.c | 41 |
| `_dl_lookup_symbol_x` | elf/dl-lookup.c | 762 |
| `do_lookup_x` | elf/dl-lookup.c | 339 |
| `do_lookup_unique` | elf/dl-lookup.c | 208 |
| `check_match` | elf/dl-lookup.c | 59 |
| `add_dependency` | elf/dl-lookup.c | 524 |
| `_dl_new_hash` | sysdeps/generic/dl-new-hash.h | 67 |
| `_dl_elf_hash` | sysdeps/generic/dl-hash.h | 28 |
| `reloc_offset` | elf/dl-runtime.h | 21 |
| `elf_machine_fixup_plt` | sysdeps/aarch64/dl-machine.h | 141 |
| `elf_machine_plt_value` | sysdeps/aarch64/dl-machine.h | 152 |
| `elf_ifunc_invoke` | sysdeps/aarch64/dl-irel.h | 48 |
| `dl_relocate_ld` | sysdeps/generic/ldsodefs.h | 76 |
| `_dl_runtime_resolve` | sysdeps/aarch64/dl-trampoline.S | (汇编) |
