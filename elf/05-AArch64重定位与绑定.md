# AArch64 平台重定位与绑定机制

## 概述

本文分析 glibc 动态链接器在 AArch64 (ARM64) 平台上的重定位处理、延迟绑定流程和
TLS 描述符机制。AArch64 相对 x86_64 有以下特殊之处：

- PLT 使用 `ip0`/`ip1` (x16/x17) 作为链接器保留寄存器
- 支持 **BTI** (Branch Target Identification) 和 **PAC** (Pointer Authentication) 安全特性
- TLS 使用 **TLSDESC** 描述符方案（而非传统 __tls_get_addr 直接调用）
- 支持 **Variant PCS** (非标准调用约定符号) 的特殊处理

---

## 一、核心文件布局

| 文件 | 行数 | 职责 |
|------|------|------|
| `sysdeps/aarch64/dl-machine.h` | ~375 | 架构特定重定位引擎、运行时设置、延迟绑定 |
| `sysdeps/aarch64/dl-trampoline.S` | ~330 | PLT resolver 汇编（_dl_runtime_resolve/_profile） |
| `sysdeps/aarch64/dl-tlsdesc.S` | ~240 | TLS 描述符处理函数 |
| `sysdeps/aarch64/dl-tlsdesc.h` | ~58 | TLS 描述符数据结构定义 |
| `sysdeps/aarch64/dl-bti.c` | ~135 | BTI 保护启用与检查 |
| `sysdeps/aarch64/dl-irel.h` | — | IRELATIVE (IFUNC) 支持 |
| `sysdeps/aarch64/ldsodefs.h` | ~47 | 架构 link_map 扩展字段 |
| `sysdeps/aarch64/bits/link.h` | ~48 | La_aarch64_regs/retval (审计接口) |

---

## 二、GOT 与 PLT 布局

### GOT (Global Offset Table) 结构

```
GOT[0] = _DYNAMIC 地址           # 本 DSO 的 .dynamic 段地址
GOT[1] = struct link_map *l      # 本 DSO 的 link_map 指针
GOT[2] = _dl_runtime_resolve     # PLT resolver 入口

GOT[3] = 第一个 GLOB_DAT/JUMP_SLOT 数据
GOT[4] = ...
```

### PLT 条目格式（AArch64 标准）

```asm
PLT[0] (PLT header):
  stp  x16, x30, [sp, #-16]!   # 保存 ip0 和 lr
  adrp x16, GOT+16             # 加载 &GOT[2] 的页地址
  ldr  x17, [x16, :lo12:GOT+16]  # x17 = GOT[2] = resolver
  add  x16, x16, :lo12:GOT+16    # x16 = &GOT[2]
  br   x17                       # 跳转到 resolver

PLT[n] (每个函数的 PLT stub):
  adrp x16, GOT+offset          # &GOT[n] 页地址
  ldr  x17, [x16, :lo12:GOT+n]  # x17 = GOT[n] (初始指向 PLT header)
  add  x16, x16, :lo12:GOT+n    # x16 = &GOT[n]
  br   x17                       # 跳转目标
```

> **注意**: 启用 BTI 时，每个 PLT 条目前会有 `bti c` 指令（兼容间接调用目标标记）。

---

## 三、运行时设置

**源文件**: `sysdeps/aarch64/dl-machine.h:63-108`

```c
elf_machine_runtime_setup(link_map *l, scope[], lazy, profile):
  if l->l_info[DT_JMPREL] && lazy:
    got = (Addr *) l->l_info[DT_PLTGOT]
    
    // 保存原 GOT[1] 到 l->l_mach.plt（用于 lazy_rel 后备）
    if got[1]:
      l->l_mach.plt = got[1] + l->l_addr
    
    got[1] = (Addr) l                     // link_map 指针
    
    if profile:
      got[2] = &_dl_runtime_profile       // 审计/性能分析
    else:
      got[2] = &_dl_runtime_resolve       // 正常 resolver
```

### 关键常量

```c
#define ELF_MACHINE_NAME "aarch64"
#define ELF_MACHINE_JMP_SLOT  R_AARCH64_JUMP_SLOT  // = 1026
#define RTLD_START  asm (".globl _dl_start");       // dl-start.S 中定义
```

---

## 四、延迟绑定流程

### 4.1 PLT 调用触发

```
应用调用 foo():
  → PLT[foo] stub
  → ldr x17, [GOT_foo]   // 首次: GOT_foo 指向 PLT header
  → br x17
  → PLT header:
      stp x16, x30, [sp, #-16]!   // 压入 &GOT[n] 和返回地址
      → br GOT[2]                   // 跳到 _dl_runtime_resolve
```

### 4.2 _dl_runtime_resolve

**源文件**: `sysdeps/aarch64/dl-trampoline.S:36-123`

```asm
_dl_runtime_resolve:
  bti   c                          // BTI: 合法间接调用目标
  
  // ① 保存参数寄存器（防止 resolver 破坏调用现场）
  stp   x8, x9, [sp, #-(80+128)]! // 80字节通用寄存器 + 128字节 NEON
  stp   x6, x7, [sp, #16]
  stp   x4, x5, [sp, #32]
  stp   x2, x3, [sp, #48]
  stp   x0, x1, [sp, #64]
  stp   q0, q1, [sp, #80]         // 8 个 NEON/FP 参数寄存器
  stp   q2, q3, [sp, #112]
  stp   q4, q5, [sp, #144]
  stp   q6, q7, [sp, #176]
  
  // ② 计算参数
  ldr   x0, [ip0, -8]             // x0 = GOT[1] = link_map*
  ldr   x1, [sp, 80+128]          // x1 = &GOT[n]（由 PLT 压栈）
  // 计算 reloc_arg = .rela.plt 中的字节偏移 = (n-3)*24
  sub   x1, x1, ip0               // &GOT[n] - &GOT[2] (字节差)
  add   x1, x1, x1, lsl #1       // ×3
  lsl   x1, x1, #3               // ×8
  sub   x1, x1, #(RELA_SIZE<<3)  // - 24*8 (跳过 GOT[2] 对应偏移)
  lsr   x1, x1, #3               // ÷8 → 最终字节偏移
  
  // ③ 调用 C 函数
  bl    _dl_fixup                  // 返回值 = 目标函数地址
  mov   ip0, x0                    // 保存到 ip0
  
  // ④ 恢复所有参数寄存器
  ldp   q0, q1, [sp, #80]
  ldp   q2, q3, [sp, #112]
  ldp   q4, q5, [sp, #144]
  ldp   q6, q7, [sp, #176]
  ldp   x0, x1, [sp, #64]
  ldp   x2, x3, [sp, #48]
  ldp   x4, x5, [sp, #32]
  ldp   x6, x7, [sp, #16]
  ldp   x8, x9, [sp], #(80+128)
  ldp   ip1, lr, [sp], #16        // 恢复返回地址
  
  // ⑤ 跳转到已解析的目标
  br    ip0
```

### 4.3 寄存器保存说明

| 寄存器 | 保存原因 |
|--------|---------|
| x0-x9 | 函数参数 (x0-x7) + 间接结果 (x8) + 临时 (x9) |
| q0-q7 | NEON/FP 参数寄存器 |
| lr (x30) | 原始返回地址 |
| ip0 (x16) | PLT stub 中使用的临时寄存器 |

总保存空间 = 80 (通用) + 128 (NEON) + 16 (PLT 压栈) = **224 字节**

---

## 五、重定位引擎

**源文件**: `sysdeps/aarch64/dl-machine.h:169-292`

### elf_machine_rela() 处理的重定位类型

```
elf_machine_rela(map, scope[], reloc, sym, version, reloc_addr, skip_ifunc):
  r_type = R_TYPE(reloc->r_info)
  
  // 快速路径: RELATIVE（数量最多，优先处理）
  if r_type == R_AARCH64_RELATIVE:
    *reloc_addr = map->l_addr + reloc->r_addend
    return
  
  if r_type == R_AARCH64_NONE:
    return
  
  // 慢路径: 需要符号解析
  sym_map = RESOLVE_MAP(...)
  value = SYMBOL_ADDRESS(sym_map, sym)
  
  // IFUNC 处理
  if sym->st_info == STT_GNU_IFUNC && !skip_ifunc:
    value = elf_ifunc_invoke(value)   // 调用 resolver 获取实际地址
  
  switch (r_type):
    GLOB_DAT / JUMP_SLOT:  *reloc_addr = value + addend
    ABS32 / ABS64:         *reloc_addr = value + addend
    COPY:                  memcpy(reloc_addr, value, size)
    TLSDESC:               → 设置 TLS 描述符（见第六节）
    TLS_DTPMOD:            *reloc_addr = sym_map->l_tls_modid
    TLS_DTPREL:            *reloc_addr = sym->st_value + addend
    TLS_TPREL:             *reloc_addr = sym->st_value + addend + tls_offset
    IRELATIVE:             value = map->l_addr + addend; *reloc_addr = ifunc(value)
```

### 动态重定位类型一览

| 类型 | 值 | 语义 | 公式 |
|------|-----|------|------|
| `R_AARCH64_NONE` | 0 | 无操作 | — |
| `R_AARCH64_ABS64` | 257 | 绝对 64 位 | S + A |
| `R_AARCH64_ABS32` | 258 | 绝对 32 位 | S + A |
| `R_AARCH64_COPY` | 1024 | 数据复制 | memcpy |
| `R_AARCH64_GLOB_DAT` | 1025 | GOT 数据条目 | S + A |
| `R_AARCH64_JUMP_SLOT` | 1026 | PLT 函数条目 | S + A |
| `R_AARCH64_RELATIVE` | 1027 | 基址偏移调整 | B + A |
| `R_AARCH64_TLS_DTPMOD` | 1028 | TLS 模块 ID | module_id |
| `R_AARCH64_TLS_DTPREL` | 1029 | TLS 模块内偏移 | S + A |
| `R_AARCH64_TLS_TPREL` | 1030 | TLS 线程指针偏移 | S + A + tls_offset |
| `R_AARCH64_TLSDESC` | 1031 | TLS 描述符 | {entry, arg} |
| `R_AARCH64_IRELATIVE` | 1032 | IFUNC 间接 | ifunc(B + A) |

> S = 符号地址, A = addend, B = 基址 (l_addr)

---

## 六、TLS 描述符 (TLSDESC) 机制

AArch64 使用 TLS 描述符方案代替传统 GD/LD 模型的直接 `__tls_get_addr` 调用，
以获得更好的性能（静态 TLS 时零系统调用）。

### 6.1 数据结构

**源文件**: `sysdeps/aarch64/dl-tlsdesc.h:24-43`

```c
// GOT 中的 TLS 描述符（2 个指针大小）
struct tlsdesc {
  ptrdiff_t (*entry)(struct tlsdesc *);   // 处理函数指针
  void *arg;                              // 参数（含义取决于 entry）
};

// 动态 TLS 的参数结构
struct tlsdesc_dynamic_arg {
  tls_index tlsinfo;    // { ti_module, ti_offset }
  size_t gen_count;     // DTV 代数（用于快速路径验证）
};
```

### 6.2 三种描述符处理函数

| 函数 | 场景 | 性能 |
|------|------|------|
| `_dl_tlsdesc_return` | 静态 TLS（编译时已知偏移） | **1 条 ldr + ret** |
| `_dl_tlsdesc_undefweak` | 未定义弱符号 | 3 条指令 |
| `_dl_tlsdesc_dynamic` | 动态 TLS（运行时加载的 .so） | 快速路径 ~10 条 / 慢路径调用 __tls_get_addr |

### 6.3 描述符设置（链接器填充）

**源文件**: `sysdeps/aarch64/dl-machine.h:227-256`

```c
case R_AARCH64_TLSDESC:
  if sym == NULL:
    // 未定义弱符号
    td->arg = addend
    td->entry = _dl_tlsdesc_undefweak
  else if TRY_STATIC_TLS(map, sym_map):
    // 可以使用静态 TLS（主程序或 LD_PRELOAD 的库）
    td->arg = sym->st_value + sym_map->l_tls_offset + addend
    td->entry = _dl_tlsdesc_return     // 最快路径！
  else:
    // 必须用动态 TLS (dlopen 加载的库)
    td->arg = _dl_make_tlsdesc_dynamic(sym_map, sym->st_value + addend)
    td->entry = _dl_tlsdesc_dynamic
```

### 6.4 _dl_tlsdesc_return（静态 TLS 快速路径）

**源文件**: `sysdeps/aarch64/dl-tlsdesc.S:76-81`

```asm
_dl_tlsdesc_return:
  bti   c
  ldr   x0, [x0, #8]    // x0 = td->arg = TP 偏移
  ret                     // 返回给调用者
```

调用者随后执行 `mrs xN, tpidr_el0; add xN, xN, x0` 得到 TLS 变量地址。

### 6.5 _dl_tlsdesc_dynamic（动态 TLS）

**源文件**: `sysdeps/aarch64/dl-tlsdesc.S:143-239`

```
_dl_tlsdesc_dynamic(td):
  paciasp                              // PAC 签名返回地址
  
  // 快速路径:
  tp = mrs tpidr_el0                   // 读取线程指针
  arg = td->arg                        // tlsdesc_dynamic_arg*
  dtv = *(dtv_t**)(tp + DTV_OFFSET)    // 当前线程的 DTV
  
  if arg->gen_count <= dtv[0].counter:   // 代数有效
    val = dtv[arg->ti_module].pointer
    if val != TLS_DTV_UNALLOCATED:       // 已分配
      return val + arg->ti_offset - tp   // 快速返回！
  
  // 慢速路径: 保存所有寄存器，调用 __tls_get_addr
  save x5-x18, q0-q31
  result = __tls_get_addr(&arg->tlsinfo)
  return result - tp                     // 转为相对 TP 的偏移
  restore all regs
  autiasp                                // PAC 验证
```

### 6.6 访问 TLS 变量的典型代码序列

```asm
// 编译器生成的 TLS 变量访问（使用 TLSDESC）
adrp    x0, :tlsdesc:var          // 加载描述符 GOT 页地址
ldr     x1, [x0, :tlsdesc_lo12:var]  // x1 = td->entry
add     x0, x0, :tlsdesc_lo12:var    // x0 = &td
.tlsdesccall var                   // 给链接器的 relaxation hint
blr     x1                         // 调用 td->entry(td)
mrs     x2, tpidr_el0             // x2 = 线程指针
add     x2, x2, x0               // x2 = TLS 变量地址
```

---

## 七、Variant PCS 处理

AArch64 允许函数使用非标准调用约定（如 SVE 向量函数），标记为 `STO_AARCH64_VARIANT_PCS`。
这些函数**不能使用延迟绑定**（因为 resolver 只保存标准 ABI 寄存器）。

**源文件**: `sysdeps/aarch64/dl-machine.h:316-337`

```c
elf_machine_lazy_rel(...):
  if r_type == R_AARCH64_JUMP_SLOT:
    if map->l_info[DT_AARCH64(VARIANT_PCS)] != NULL:
      // 检查此符号是否标记了 VARIANT_PCS
      sym = &symtab[R_SYM(reloc->r_info)]
      if sym->st_other & STO_AARCH64_VARIANT_PCS:
        // 立即解析（eagerly resolve），不使用延迟绑定
        elf_machine_rela(map, scope, reloc, sym, version, reloc_addr, ...)
        return
    
    // 普通符号: 设置延迟绑定
    if map->l_mach.plt == 0:
      *reloc_addr += l_addr         // 相对重定位
    else:
      *reloc_addr = map->l_mach.plt // 指向 PLT header
```

---

## 八、BTI (Branch Target Identification) 保护

### 8.1 概述

BTI 是 ARMv8.5-A 引入的控制流完整性特性。启用后，间接跳转/调用的目标地址
必须是 `bti` 指令，否则触发异常。

### 8.2 实现

**源文件**: `sysdeps/aarch64/dl-bti.c:31-63`

```c
_dl_bti_protect(link_map *map, int fd):
  map->l_mach.bti = true
  
  // 对所有可执行的 PT_LOAD 段启用 PROT_BTI
  for each phdr where p_type == PT_LOAD && (p_flags & PF_X):
    prot = PROT_EXEC | PROT_BTI | (PF_R? PROT_READ:0) | (PF_W? PROT_WRITE:0)
    
    if fd == -1:
      mprotect(start, len, prot)        // 内核映射的可执行文件
    else:
      mmap(start, len, prot, MAP_FIXED|MAP_COPY, fd, off)  // 动态库
```

### 8.3 BTI 在 PLT/resolver 中的体现

所有入口点都以 `bti c` 开头：

```asm
_dl_runtime_resolve:
  bti   c                 // ← 合法的间接调用目标

_dl_tlsdesc_return:
  bti   c                 // ← 合法的间接调用目标

_dl_tlsdesc_undefweak:
  bti   c                 // ← 合法的间接调用目标
```

### 8.4 相关 ELF 标记

| 动态标签 | 含义 |
|---------|------|
| `DT_AARCH64_BTI_PLT` | DSO 的 PLT 使用 BTI 标记 |
| `DT_AARCH64_PAC_PLT` | DSO 的 PLT 使用 PAC 保护 |
| `DT_AARCH64_VARIANT_PCS` | DSO 包含 Variant PCS 符号 |

---

## 九、PAC (Pointer Authentication) 在链接器中的使用

PAC 用于保护返回地址防止 ROP 攻击。在 glibc 动态链接器中的体现：

```asm
_dl_runtime_profile:
  paciasp                  // 签名 LR（入口）
  ...
  autiasp                  // 验证 LR（出口）

_dl_tlsdesc_dynamic:
  paciasp                  // 签名 LR
  ...
  autiasp                  // 验证 LR
```

> **注意**: `_dl_runtime_resolve` 本身不使用 PAC（因为 LR 由 PLT stub 压栈传入，
> 不在 x30 中），但 `_dl_runtime_profile` 和 `_dl_tlsdesc_dynamic` 使用。

---

## 十、与 x86_64 的对比

| 特性 | x86_64 | AArch64 |
|------|--------|---------|
| PLT 跳转方式 | `jmp *GOT[n](%rip)` | `ldr x17, [x16]; br x17` |
| Resolver 寄存器 | rdi, rsi 传参 | x0=link_map, x1=reloc_offset |
| 保存的参数寄存器 | rdi,rsi,rdx,rcx,r8,r9 + xmm0-7 | x0-x9 + q0-q7 |
| TLS 方案 | GD/LD/IE/LE (调用 __tls_get_addr) | TLSDESC (描述符回调) |
| 控制流保护 | CET (ENDBR64) | BTI (`bti c/j`) |
| 返回地址保护 | Shadow Stack | PAC (`paciasp`/`autiasp`) |
| SA_RESTORER | glibc 设置 __restore_rt | 内核自动处理 |
| IFUNC 支持 | R_X86_64_IRELATIVE | R_AARCH64_IRELATIVE |
| RELATIVE 编码 | r_addend | r_addend |
| Variant PCS | 无 | `STO_AARCH64_VARIANT_PCS` → 禁用延迟绑定 |

---

## 十一、延迟绑定完整时序图

```
应用代码          PLT stub         _dl_runtime_resolve      _dl_fixup
    │                │                     │                    │
    │ call foo ──→   │                     │                    │
    │                │ ldr x17,[GOT_foo]   │                    │
    │                │ br x17 ─────────→   │                    │
    │                │                     │ bti c              │
    │                │                     │ save x0-x9,q0-q7  │
    │                │                     │ x0 = link_map      │
    │                │                     │ x1 = reloc_offset  │
    │                │                     │ bl _dl_fixup ──→   │
    │                │                     │                    │ 查找符号
    │                │                     │                    │ 解析地址
    │                │                     │                    │ GOT[foo] = addr
    │                │                     │     ←── return addr│
    │                │                     │ restore regs       │
    │                │                     │ br ip0 ──→ foo()  │
    │                │                     │                    │
    │ (后续调用)     │                     │                    │
    │ call foo ──→   │                     │                    │
    │                │ ldr x17,[GOT_foo]   │                    │
    │                │ br x17 ──→ foo() 直接│                    │
```

---

## 十二、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `sysdeps/aarch64/dl-machine.h:22` | `ELF_MACHINE_NAME "aarch64"` |
| `sysdeps/aarch64/dl-machine.h:63-108` | `elf_machine_runtime_setup`（GOT 初始化） |
| `sysdeps/aarch64/dl-machine.h:111` | `RTLD_START` 定义 |
| `sysdeps/aarch64/dl-machine.h:113-119` | `elf_machine_type_class`（类型分类宏） |
| `sysdeps/aarch64/dl-machine.h:121` | `ELF_MACHINE_JMP_SLOT = R_AARCH64_JUMP_SLOT` |
| `sysdeps/aarch64/dl-machine.h:140-148` | `elf_machine_fixup_plt`（写入 GOT） |
| `sysdeps/aarch64/dl-machine.h:162-163` | `ARCH_LA_PLTENTER/PLTEXIT` |
| `sysdeps/aarch64/dl-machine.h:169-292` | `elf_machine_rela`（主重定位引擎） |
| `sysdeps/aarch64/dl-machine.h:177-178` | R_AARCH64_RELATIVE 处理 |
| `sysdeps/aarch64/dl-machine.h:198-201` | GLOB_DAT/JUMP_SLOT 处理 |
| `sysdeps/aarch64/dl-machine.h:208-225` | COPY 重定位 |
| `sysdeps/aarch64/dl-machine.h:227-256` | TLSDESC 描述符设置 |
| `sysdeps/aarch64/dl-machine.h:258-277` | TLS_DTPMOD/DTPREL/TPREL |
| `sysdeps/aarch64/dl-machine.h:279-284` | IRELATIVE (IFUNC) |
| `sysdeps/aarch64/dl-machine.h:304-372` | `elf_machine_lazy_rel`（延迟绑定） |
| `sysdeps/aarch64/dl-machine.h:316-337` | Variant PCS 特殊处理 |
| `sysdeps/aarch64/dl-trampoline.S:36-123` | `_dl_runtime_resolve` |
| `sysdeps/aarch64/dl-trampoline.S:129-327` | `_dl_runtime_profile` |
| `sysdeps/aarch64/dl-tlsdesc.h:25-29` | `struct tlsdesc` |
| `sysdeps/aarch64/dl-tlsdesc.h:39-43` | `struct tlsdesc_dynamic_arg` |
| `sysdeps/aarch64/dl-tlsdesc.S:76-81` | `_dl_tlsdesc_return`（静态 TLS） |
| `sysdeps/aarch64/dl-tlsdesc.S:97-108` | `_dl_tlsdesc_undefweak` |
| `sysdeps/aarch64/dl-tlsdesc.S:143-239` | `_dl_tlsdesc_dynamic`（动态 TLS） |
| `sysdeps/aarch64/dl-bti.c:31-63` | `_dl_bti_protect`（启用 PROT_BTI） |
| `elf/elf.h:2911-3034` | R_AARCH64_* 重定位常量 |
| `elf/elf.h:3040-3042` | DT_AARCH64_BTI_PLT/PAC_PLT/VARIANT_PCS |
