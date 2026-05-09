# AArch64 硬件扩展支持

## 概述

glibc 对 AArch64 平台的现代硬件扩展提供了全面的支持，涵盖性能优化和安全加固两大方面。
本文分析六个关键扩展在 glibc 中的实现：

| 分类 | 扩展 | 说明 |
|------|------|------|
| 性能 | **SVE** (Scalable Vector Extension) | 可伸缩向量扩展，优化内存/数学运算 |
| 性能 | **SME** (Scalable Matrix Extension) | 可伸缩矩阵扩展，ZA 寄存器管理 |
| 性能 | **MOPS** (Memory Operations) | 硬件原生内存拷贝/填充 |
| 安全 | **MTE** (Memory Tagging Extension) | 内存标签，检测堆越界/use-after-free |
| 安全 | **BTI** (Branch Target Identification) | 分支目标标识，防 JOP/COP 攻击 |
| 安全 | **PAC** (Pointer Authentication) | 指针认证，防 ROP 攻击 |
| 安全 | **GCS** (Guarded Control Stack) | 守护控制栈，硬件影子栈 |

---

## 一、特性检测: HWCAP 与 cpu_features

### 1.1 HWCAP 定义

**源文件**: `sysdeps/unix/sysv/linux/aarch64/bits/hwcap.h`

内核通过 `auxv[AT_HWCAP]` / `AT_HWCAP2` / `AT_HWCAP3` 向用户态传递 CPU 能力位图：

| 宏 | 位 | 扩展 |
|----|-----|------|
| `HWCAP_SVE` | bit 22 (HWCAP) | SVE |
| `HWCAP_PACA` | bit 30 (HWCAP) | PAC (地址密钥) |
| `HWCAP_PACG` | bit 31 (HWCAP) | PAC (通用密钥) |
| `HWCAP_GCS` | bit 32 (HWCAP) | GCS |
| `HWCAP2_BTI` | bit 17 (HWCAP2) | BTI |
| `HWCAP2_MTE` | bit 18 (HWCAP2) | MTE |
| `HWCAP2_SME` | bit 23 (HWCAP2) | SME |
| `HWCAP2_MOPS` | bit 43 (HWCAP2) | MOPS |
| `HWCAP2_SVE2` | bit 1 (HWCAP2) | SVE2 |
| `HWCAP2_SME2` | bit 37 (HWCAP2) | SME2 |

### 1.2 cpu_features 结构

**源文件**: `sysdeps/aarch64/cpu-features.h:62-72`

```c
struct cpu_features {
    uint64_t midr_el1;    // 处理器型号 (Main ID Register)
    unsigned zva_size;    // DC ZVA 块大小 (字节)
    bool bti;             // BTI 可用
    uint8_t mte_state;    // MTE 状态 (tunable 值)
    bool sve;             // SVE 可用
    bool unused;
    bool mops;            // MOPS 可用
};
```

### 1.3 init_cpu_features — 特性初始化

**源文件**: `sysdeps/unix/sysv/linux/aarch64/cpu-features.c:63-135`

在动态链接器初始化时执行，完成以下检测：

```
init_cpu_features():
  ① 读取 MIDR_EL1 (处理器型号识别)
     - 支持 tunable glibc.cpu.name 覆盖
     - 识别 kunpeng920/950, a64fx, oryon1

  ② ZVA (Data Cache Zero by VA) 检测
     - mrs dczid_el0 → zva_size = 4 << (dczid & 0xf)

  ③ BTI 检测
     - HWCAP2 & HWCAP2_BTI → bti = true
     - 读取 tunable glibc.cpu.aarch64_bti

  ④ MTE 配置
     - HWCAP2 & HWCAP2_MTE + tunable glibc.mem.tagging
     - mte_state bit 0: 异步模式
     - mte_state bit 1: 同步模式
     - mte_state bit 2: 系统自选模式
     - prctl(PR_SET_TAGGED_ADDR_CTRL, ...) 启用 MTE

  ⑤ SVE 检测
     - HWCAP & HWCAP_SVE → sve = true

  ⑥ MOPS 检测
     - HWCAP2 & HWCAP2_MOPS → mops = true

  ⑦ GCS 配置
     - HWCAP & HWCAP_GCS + tunable glibc.cpu.aarch64_gcs
```

---

## 二、SVE — 可伸缩向量扩展

### 2.1 IFUNC 多版本调度

SVE 的主要价值在于为关键内存函数提供优化实现。glibc 使用 IFUNC 机制
在运行时根据 CPU 能力选择最优实现。

**源文件**: `sysdeps/aarch64/multiarch/memcpy.c:39-61`

```c
select_memcpy_ifunc():
  if (mops)          → __memcpy_mops      // ARMv9.3 硬件原生
  if (sve):
    if IS_KUNPENG950 → __memcpy_kunpeng950 // 鲲鹏 950 优化
    if IS_A64FX      → __memcpy_a64fx      // 富士通 A64FX 优化
    else             → __memcpy_sve        // 通用 SVE
  if IS_ORYON1       → __memcpy_oryon1     // 高通 Oryon 优化
  else               → __memcpy_generic    // ASIMD 通用
```

### 2.2 SVE 优化实现列表

**源文件**: `sysdeps/aarch64/multiarch/ifunc-impl-list.c:36-63`

| 函数 | SVE 变体 | 选择条件 |
|------|----------|---------|
| `memcpy` | `__memcpy_sve` | `sve == true` |
| `memcpy` | `__memcpy_a64fx` | `sve && IS_A64FX` |
| `memcpy` | `__memcpy_kunpeng950` | `sve && IS_KUNPENG950` |
| `memmove` | `__memmove_sve` | `sve == true` |
| `memmove` | `__memmove_a64fx` | `sve && IS_A64FX` |
| `memset` | `__memset_sve_zva64` | `sve && zva_size == 64` |
| `memset` | `__memset_a64fx` | `sve && zva_size == 256` |
| `strlen` | `__strlen_asimd` | `!mte` (MTE 启用时回退) |

### 2.3 SVE memcpy 实现

**源文件**: `sysdeps/aarch64/multiarch/memcpy_sve.S:1-199`

使用 SVE 的可变长度向量寄存器进行内存拷贝，核心优势：
- **向量长度无关 (VLA)**: 一份代码适应 128-2048 位向量长度
- **谓词寄存器**: `WHILELO` 指令自动处理尾部不足一个向量的情况
- **大块传输**: 展开的 SVE ld1/st1 循环

### 2.4 SVE memset 实现

**源文件**: `sysdeps/aarch64/multiarch/memset_sve_zva64.S:1-120`

针对 ZVA 大小为 64 字节的 CPU 优化：
- 小块: 直接 STR/STP
- 中块: SVE st1 向量化
- 大块: `DC ZVA` (清零) 对齐后批量操作

### 2.5 libmvec SVE 数学库

**源文件**: `sysdeps/aarch64/fpu/` 目录

glibc 为 AArch64 提供大量 SVE 优化的向量数学函数：
- `float-sve-funcs` / `double-sve-funcs` 列表
- 例如: `cosf_sve.c`, `sinf_sve.c`, `expf_sve.c`, `logf_sve.c` 等

---

## 三、SME — 可伸缩矩阵扩展

### 3.1 核心挑战

SME 引入了两个新的处理器状态：
- **PSTATE.SM** (Streaming SVE mode): SVE 向量长度可能变化
- **PSTATE.ZA**: ZA 矩阵寄存器是否活跃

问题：glibc 内部函数可能在用户的 SME 流式模式中被调用，但 glibc 的
SIMD/SVE 代码可能不兼容流式模式。因此 glibc 必须在关键点清除 ZA 状态。

### 3.2 __libc_arm_za_disable — 核心清理函数

**源文件**: `sysdeps/aarch64/__arm_za_disable.S:33-105`

```
__libc_arm_za_disable():
  ① 检查 HWCAP2_SME (bit 23)
     → 不支持则直接返回

  ② 读取 tpidr2_el0 (ZA 保存描述符)
     → 为 0 则 ZA 未活跃，返回

  ③ 检查保留字节 (兼容性)
     → 非零则调用 __libc_fatal (不支持的 SME 扩展)

  ④ 保存 ZA 内容到用户提供的缓冲区
     - 读取 za_save_buffer 指针和 num_za_save_slices
     - 循环: str za[w15, 0..15], [x16, ...]
     - 每轮处理 16 个切片

  ⑤ 清除 ZA 状态
     - msr tpidr2_el0, xzr   // 清零描述符
     - smstop za              // 关闭 ZA
```

### 3.3 SME 状态清理调用点

| 调用点 | 文件 | 原因 |
|--------|------|------|
| `setjmp` | `sysdeps/aarch64/setjmp.S:41` | 保存点前清除 ZA |
| `longjmp` | `sysdeps/aarch64/__longjmp.S:28-31` | 跳转前清除 ZA |
| `setcontext` | `sysdeps/unix/sysv/linux/aarch64/setcontext.S:51-52` | 上下文切换前清除 ZA |

### 3.4 信号处理与 SME

当 SME 活跃时内核投递信号，内核会自动保存 ZA/SM 状态到信号帧的
`__reserved` 扩展块中，并在 `rt_sigreturn` 时恢复。glibc 信号处理器
不需要额外操作，但 glibc 的 SME 测试验证了这一行为：

**测试文件**: `sysdeps/aarch64/tst-sme-signal.c` — 验证信号处理不破坏 ZA 内容

---

## 四、MTE — 内存标签扩展

### 4.1 原理

MTE 为每个 16 字节内存粒度附加 4 位标签 (0-15)。指针的高 4 位也携带标签。
当指针标签与内存标签不匹配时，CPU 触发故障或异步报告。

### 4.2 glibc MTE 接口

**源文件**: `sysdeps/aarch64/libc-mtag.h:25-65`

```c
#define __MTAG_GRANULE_SIZE 16      // MTE 粒度: 16 字节
#define __MTAG_SBRK_UNTAGGED 1      // sbrk 内存不带标签
#define __MTAG_MMAP_FLAGS PROT_MTE  // mmap 需要 PROT_MTE 标志

// LDG: 从内存读取标签
__libc_mtag_address_get_tag(p):
  ldg x0, [x0]     // 将 p 指向地址的标签加载到 x0 的高位

// GMI + IRG: 生成与当前标签不同的新随机标签
__libc_mtag_new_tag(p):
  gmi x1, x0, xzr  // 获取当前标签的排除掩码
  irg x0, x0, x1    // 生成不同的新标签
```

### 4.3 批量标签操作

| 函数 | 文件 | 作用 |
|------|------|------|
| `__libc_mtag_tag_region` | `sysdeps/aarch64/__mtag_tag_region.S` | STG/ST2G + DC GVA 批量设置标签 |
| `__libc_mtag_tag_zero_region` | `sysdeps/aarch64/__mtag_tag_zero_region.S` | STZG/STZ2G + DC GZVA 清零+标签 |

### 4.4 malloc MTE 集成

**源文件**: `malloc/malloc.c:382-440`, `malloc/arena.c`

```
malloc/free 中的 MTE 使用:

  malloc(size):
    ① 对齐到 __MTAG_GRANULE_SIZE (16 字节)
    ② _int_malloc → 获得 chunk
    ③ tag_new_usable(ptr):
       - __libc_mtag_new_tag(ptr)   // 生成新标签
       - __libc_mtag_tag_region()   // 标记整个可用区域
    ④ 返回带标签的指针

  free(ptr):
    ① tag_at(ptr) → ldg 获取实际标签
    ② mem2chunk(ptr) → 定位 chunk 头（需要正确标签）
    ③ tag_region(chunk2mem(p), ...) → 重置标签（防 use-after-free）

  堆创建:
    mmap(..., PROT_MTE) → 分配带 MTE 保护的内存区域
```

### 4.5 MTE tunable

**tunable**: `glibc.mem.tagging`

| 值 | 模式 | 说明 |
|----|------|------|
| 0 | 关闭 | 不使用 MTE |
| 1 | 异步 | `PR_MTE_TCF_ASYNC` — 延迟报告，性能影响小 |
| 2 | 同步 | `PR_MTE_TCF_SYNC` — 立即触发 SIGSEGV，精确定位 |
| 4 | 系统自选 | `PR_MTE_TCF_SYNC + ASYNC` — 内核选择最优模式 |

编译时需要 `--enable-memory-tagging` 配置选项。

标签 0 被排除 (`MTE_ALLOWED_TAGS = 0xfffe`)，保留给堆元数据结构使用。

---

## 五、BTI — 分支目标标识

### 5.1 原理

BTI 在每个合法的间接分支目标处放置 `BTI` 指令。CPU 执行间接分支后，
如果目标不是 `BTI` 指令，则触发异常。这防止了 JOP (Jump-Oriented Programming)
和 COP (Call-Oriented Programming) 攻击。

### 5.2 ELF 标记

**源文件**: `sysdeps/aarch64/dl-prop.h:44-67`

```c
_dl_process_gnu_property(l, fd, type, datasz, data):
  if type == GNU_PROPERTY_AARCH64_FEATURE_1_AND:
    feature_1 = *(unsigned int *) data
    if feature_1 & GNU_PROPERTY_AARCH64_FEATURE_1_BTI:
      if cpu_features.bti:
        _dl_bti_protect(l, fd)    // 启用 BTI 保护
```

编译器在 ELF 文件的 `.note.gnu.property` 段中写入 BTI 标记。
动态链接器在加载每个共享库时读取此标记。

### 5.3 运行时 BTI 保护

**源文件**: `sysdeps/aarch64/dl-bti.c:31-63`

```c
_dl_bti_protect(map, fd):
  map->l_mach.bti = true
  for each PT_LOAD with PF_X:
    prot = PROT_EXEC | PROT_BTI | (PF_R?) | (PF_W?)
    mmap(start, len, prot, MAP_FIXED|MAP_COPY, fd, off)
    // 或 mprotect(start, len, prot) 对内核映射的二进制
```

BTI 通过 `mprotect(..., PROT_BTI)` 对可执行段启用。

### 5.4 BTI 一致性检查

**源文件**: `sysdeps/aarch64/dl-bti.c:101-133`

```c
_dl_bti_check(l, program):
  if !cpu_features.bti: return   // CPU 不支持则跳过
  
  if l->l_mach.bti_fail:
    bti_failed(l, program)       // mmap 失败 → 致命错误
  
  for each dependency dep:
    if dep->l_mach.bti_fail:
      bti_failed(dep, program)
    if !dep->l_mach.bti:
      if enforce_bti:  bti_failed  // 强制模式: abort
      else:            bti_warning // 宽松模式: 警告
```

### 5.5 BTI tunable

**tunable**: `glibc.cpu.aarch64_bti`

| 值 | 模式 | 说明 |
|----|------|------|
| 0 | 宽松 (`PERMISSIVE`) | 仅警告未标记 BTI 的库 |
| 1 | 强制 (`ENFORCED`) | 未标记 BTI 的库导致加载失败 |

### 5.6 glibc 自身的 BTI 支持

| 文件 | BTI 指令位置 |
|------|-------------|
| `sysdeps/aarch64/start.S` | `bti c` — 进程入口点 |
| `sysdeps/aarch64/dl-trampoline.S:37-43` | `bti c` — PLT 解析器入口 |
| `sysdeps/aarch64/dl-tlsdesc.S:76-106` | `bti c` — TLS 描述符入口 |

---

## 六、PAC — 指针认证

### 6.1 原理

PAC 使用密钥对指针进行签名。函数入口 (`PACIASP`) 对返回地址签名，
函数出口 (`AUTIASP`) 验证签名。篡改返回地址会导致认证失败和异常。

### 6.2 glibc 中的 PAC 使用

| 文件 | 用途 |
|------|------|
| `sysdeps/aarch64/crti.S:72-96` | `_init`/`_fini` 中 `paciasp` 签名 |
| `sysdeps/aarch64/crtn.S:42-50` | `_init`/`_fini` 返回前 `autiasp` 验证 |
| `sysdeps/aarch64/dl-trampoline.S:129-133` | PLT 解析器 `paciasp`/`autiasp` |

### 6.3 setjmp/longjmp 与指针保护

**源文件**: `sysdeps/aarch64/setjmp.S:50-55`

```asm
// setjmp 保存 LR 和 SP 时使用 PTR_MANGLE 加密:
#ifdef PTR_MANGLE
  PTR_MANGLE (x4, x30, x3)     // 加密 LR (x30)
  stp x29, x4, [x0, #JB_X29]   // 存储加密的 LR
#else
  stp x29, x30, [x0, #JB_X29]  // 直接存储 LR
#endif
```

`PTR_MANGLE` / `PTR_DEMANGLE` 使用线程局部的随机密钥进行 XOR 加密，
与 PAC 硬件签名形成互补保护。

---

## 七、GCS — 守护控制栈

### 7.1 原理

GCS 是 ARMv9.4 的硬件影子栈特性。每次 `BL` (函数调用) 时，硬件自动将
返回地址压入 GCS 栈；每次 `RET` 时，硬件自动从 GCS 栈弹出并验证。
伪造返回地址的 ROP 攻击将导致硬件异常。

### 7.2 GCS 策略

**源文件**: `sysdeps/aarch64/dl-gcs.c:21-32`

```c
#define GCS_POLICY_DISABLED  0  // 关闭 GCS
#define GCS_POLICY_ENFORCED  1  // 强制: 未标记 GCS 的库 → abort
#define GCS_POLICY_OPTIONAL  2  // 可选: 所有依赖标记则启用
#define GCS_POLICY_OVERRIDE  3  // 覆盖: 忽略标记，强制启用
```

**tunable**: `glibc.cpu.aarch64_gcs`

### 7.3 GCS 兼容性检查

**源文件**: `sysdeps/aarch64/dl-gcs.c:124-143`

```c
_dl_gcs_check(l, program, dlopen_mode):
  switch (policy):
    case DISABLED / OVERRIDE: return
    case ENFORCED:  check_gcs_depends(l, program, true, ...)
    case OPTIONAL:  check_gcs_depends(l, program, false, ...)
    default:        unsupported() → abort
```

对于 `OPTIONAL` 策略，如果任一依赖库缺少 GCS 标记，则整个进程禁用 GCS。

### 7.4 GCS 栈分配

**源文件**: `sysdeps/aarch64/__alloc_gcs.c:38-70`

```c
__alloc_gcs(stack_size, gcs):
  size = (stack_size / 2 + 160) & -8   // GCS 大小 ≈ 主栈一半
  base = map_shadow_stack(NULL, size,
    SHADOW_STACK_SET_MARKER | SHADOW_STACK_SET_TOKEN)
  
  验证 GCS cap token (末尾第二个 64 位值)
  返回 GCS 栈顶指针 (gcsp + 1)
```

### 7.5 GCS 特性检测

**源文件**: `sysdeps/aarch64/aarch64-gcs.h:35-41`

```c
static inline bool has_gcs(void) {
    register unsigned long x16 asm ("x16") = 1;
    asm ("hint 40" /* chkfeat x16 */: "+r" (x16));
    return x16 == 0;   // 如果 GCS 可用则返回 0
}
```

### 7.6 GCS 与 setjmp/longjmp

**源文件**: `sysdeps/aarch64/setjmp.S:65-72`

```asm
// setjmp: 保存 GCS 指针
  mov  x16, 1
  CHKFEAT_X16              // 检查 GCS 是否可用
  tbnz x16, 0, gcs_done   // 不可用则跳过
  MRS_GCSPR(x2)            // 读取当前 GCS 栈指针
  add  x2, x2, 8           // setjmp 返回后的 GCS 状态
  str  x2, [x0, #JB_GCSPR] // 保存到 jmp_buf
```

longjmp 恢复时需要扫描 GCS 栈找到对应的 cap token 并切换。

### 7.7 GCS 与上下文操作

在 `getcontext.S` 和 `setcontext.S` 中同样需要保存/恢复 GCS 状态：
- **getcontext** (`getcontext.S:87-100`): 保存 GCSPR 到 `__reserved` 扩展块
- **setcontext** (`setcontext.S:115-150`): 恢复 GCS — 扫描 cap token → GCSSS1/GCSSS2 切换
- **makecontext** (`makecontext.c:88-95`): 为新上下文分配独立的 GCS 栈

---

## 八、MOPS — 硬件内存操作

### 8.1 概述

MOPS (Memory Copy and Memory Set instructions) 是 ARMv9.3 引入的硬件原生
内存拷贝/填充指令。与软件实现相比，MOPS 由 CPU 微码实现，不需要循环展开、
预取等软件优化技巧。

### 8.2 IFUNC 选择

**源文件**: `sysdeps/aarch64/multiarch/memcpy.c:44-45`

```c
if (mops)
    return __memcpy_mops;   // 优先级最高
```

MOPS 优先级最高——如果 CPU 支持，始终使用硬件指令。

### 8.3 MOPS 实现

glibc 为 memcpy/memmove/memset 各提供 MOPS 变体：
- `__memcpy_mops`
- `__memmove_mops`
- `__memset_mops`

使用 `CPYP/CPYM/CPYE` (拷贝) 和 `SETP/SETM/SETE` (填充) 指令三段式执行。

---

## 九、IFUNC 选择优先级总结

以 memcpy 为例，完整的选择优先级链：

```
select_memcpy_ifunc():
  ┌──────────────────────────────────────────┐
  │ 1. mops?         → __memcpy_mops        │  ARMv9.3 硬件原生
  │ 2. sve + KP950?  → __memcpy_kunpeng950  │  鲲鹏 950 优化
  │ 3. sve + A64FX?  → __memcpy_a64fx       │  A64FX 优化
  │ 4. sve?          → __memcpy_sve         │  通用 SVE
  │ 5. Oryon1?       → __memcpy_oryon1      │  高通 Oryon 优化
  │ 6. 默认          → __memcpy_generic     │  ASIMD 通用
  └──────────────────────────────────────────┘
```

memset 的选择还受 `zva_size` 影响 (64/256 字节不同的优化路径)。

---

## 十、各特性交互与注意事项

### 10.1 MTE 与 strlen

`__strlen_asimd` 使用的优化手段（读取超出字符串末尾的内存）在 MTE 启用时
可能触发标签不匹配故障。因此 MTE 启用时回退到 `__strlen_generic`：

```c
IFUNC_IMPL_ADD(array, i, strlen, !mte, __strlen_asimd)  // 仅无 MTE 时
IFUNC_IMPL_ADD(array, i, strlen, 1, __strlen_generic)    // 始终可用
```

### 10.2 SVE 与 getcontext/setcontext

glibc 的 `getcontext`/`setcontext` **不保存 SVE 状态**。
这是设计决定：SVE 寄存器是 caller-saved，上下文切换点等同于函数调用边界。
内核在信号帧中会完整保存 SVE 状态（通过 `SVE_MAGIC` 扩展块）。

### 10.3 SME 与 setjmp/setcontext

`setjmp` 和 `setcontext` 都会调用 `__libc_arm_za_disable` 清除 ZA 状态。
这是因为 C 语言的非本地跳转语义无法保证 ZA 寄存器的一致性。
如果用户使用 SME 并需要保留 ZA，必须在 setjmp 前手动保存。

---

## 十一、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `sysdeps/unix/sysv/linux/aarch64/bits/hwcap.h:25-126` | HWCAP/HWCAP2/HWCAP3 定义 |
| `sysdeps/aarch64/cpu-features.h:62-72` | `struct cpu_features` |
| `sysdeps/unix/sysv/linux/aarch64/cpu-features.c:63-135` | `init_cpu_features` |
| `sysdeps/aarch64/multiarch/memcpy.c:39-61` | memcpy IFUNC 选择器 |
| `sysdeps/aarch64/multiarch/ifunc-impl-list.c:36-63` | 所有 IFUNC 变体列表 |
| `sysdeps/aarch64/multiarch/memcpy_sve.S` | SVE memcpy/memmove 实现 |
| `sysdeps/aarch64/multiarch/memset_sve_zva64.S` | SVE memset (ZVA=64) |
| `sysdeps/aarch64/__arm_za_disable.S:33-105` | SME ZA 清理函数 |
| `sysdeps/aarch64/setjmp.S:39-72` | setjmp (含 SME 清理 + GCS 保存) |
| `sysdeps/aarch64/__longjmp.S:28-31` | longjmp SME ZA 清理 |
| `sysdeps/aarch64/libc-mtag.h:25-65` | MTE 接口 (LDG/IRG/GMI) |
| `sysdeps/aarch64/__mtag_tag_region.S:42-109` | MTE 批量标签 (STG/ST2G) |
| `sysdeps/aarch64/__mtag_tag_zero_region.S:42-109` | MTE 清零+标签 (STZG/STZ2G) |
| `malloc/malloc.c:382-440` | malloc MTE 集成函数 |
| `sysdeps/aarch64/dl-bti.c:31-133` | BTI 保护与检查 |
| `sysdeps/aarch64/dl-prop.h:44-67` | GNU property 处理 (BTI+GCS) |
| `sysdeps/aarch64/dl-gcs.c:21-156` | GCS 策略检查与启用 |
| `sysdeps/aarch64/__alloc_gcs.c:38-70` | GCS 栈分配 (map_shadow_stack) |
| `sysdeps/aarch64/aarch64-gcs.h:26-41` | `has_gcs()` / `gcs_record` |
| `sysdeps/aarch64/crti.S:72-96` | PAC (paciasp) — 初始化函数 |
| `sysdeps/aarch64/dl-trampoline.S:37-43,129-133` | BTI `bti c` + PAC 签名 |
