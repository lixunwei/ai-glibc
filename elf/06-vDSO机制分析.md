# vDSO 机制深度分析

## 概述

vDSO (virtual Dynamic Shared Object) 是 Linux 内核提供的一种加速系统调用的机制。
内核将一小段共享库代码映射到每个进程的地址空间，包含一些频繁调用的系统调用的
用户态实现（如 `clock_gettime`），使这些调用无需陷入内核即可完成。

本文以 AArch64 平台为主线，分析 glibc 如何发现、解析和使用 vDSO。

---

## 一、vDSO 工作原理

### 1.1 内核侧

```
进程创建时 (execve/fork):
  内核将 vDSO 映射到进程地址空间的随机位置 (ASLR)
  
  vDSO 本质是一个完整的 ELF 共享库:
  ┌──────────────────────────────────────────┐
  │ ELF Header                               │
  │ Program Headers (PT_LOAD, PT_DYNAMIC)    │
  │ .dynsym (符号表)                          │
  │ .dynstr (字符串表)                        │
  │ .gnu.hash (哈希表)                        │
  │ .text (函数代码):                         │
  │   __kernel_clock_gettime                  │
  │   __kernel_gettimeofday                  │
  │   __kernel_clock_getres                  │
  │   __kernel_getrandom                     │
  │   __kernel_rt_sigreturn (信号返回)        │
  └──────────────────────────────────────────┘
  
  内核同时映射 vvar 页（只读共享数据页）:
  ┌──────────────────────────────────────────┐
  │ vvar 数据页                               │
  │   - 当前时间戳 (wall clock)               │
  │   - 单调时钟数据                           │
  │   - 时钟源信息 (clocksource)              │
  │   - 时间命名空间偏移                       │
  │   - 序列计数器 (seqcount)                 │
  └──────────────────────────────────────────┘
  
  内核通过辅助向量 (auxiliary vector) 将 vDSO 地址传给进程:
  auxv[AT_SYSINFO_EHDR] = vDSO 的 ELF 头地址
```

### 1.2 性能对比

```
传统系统调用:
  用户态 → svc #0 → 内核态 → 处理 → 返回用户态
  开销: ~500-1000 CPU 周期 (含上下文切换、TLB flush 等)

vDSO 调用:
  用户态 → vDSO 函数 (读取 vvar 数据页) → 返回
  开销: ~10-50 CPU 周期 (纯用户态内存读取)
```

---

## 二、AArch64 vDSO 符号名

**源文件**: `sysdeps/unix/sysv/linux/aarch64/sysdep.h:168-175`

```c
#define VDSO_NAME  "LINUX_2.6.39"
#define VDSO_HASH  123718537

#define HAVE_CLOCK_GETRES64_VSYSCALL   "__kernel_clock_getres"
#define HAVE_CLOCK_GETTIME64_VSYSCALL  "__kernel_clock_gettime"
#define HAVE_GETTIMEOFDAY_VSYSCALL     "__kernel_gettimeofday"
#define HAVE_GETRANDOM_VSYSCALL        "__kernel_getrandom"
```

> **注意**: AArch64 使用 `__kernel_*` 前缀（而非 x86_64 的 `__vdso_*`），
> 且版本标签为 `LINUX_2.6.39`。

---

## 三、glibc 发现 vDSO 的完整流程

### 3.1 第一步: 解析辅助向量

**源文件**: `sysdeps/unix/sysv/linux/dl-parse_auxv.h:31-61`

```c
void _dl_parse_auxv(ElfW(auxv_t) *av, dl_parse_auxv_t auxv_values) {
  // 遍历内核传入的辅助向量
  for (; av->a_type != AT_NULL; av++)
    if (av->a_type <= AT_MINSIGSTKSZ)
      auxv_values[av->a_type] = av->a_un.a_val;
  
  // ... 设置各种 GLRO 变量 ...
  
  // 关键: 记录 vDSO 的 ELF 头地址
  GLRO(dl_sysinfo_dso) = (void *) auxv_values[AT_SYSINFO_EHDR];   // 行 57
  if (GLRO(dl_sysinfo_dso) != NULL)
    GLRO(dl_sysinfo) = auxv_values[AT_SYSINFO];                    // 行 59-60
}
```

辅助向量常量定义于 `elf/elf.h:1255-1256`:
```c
#define AT_SYSINFO       32    // 系统调用入口（x86 特有）
#define AT_SYSINFO_EHDR  33    // vDSO ELF 头虚拟地址
```

### 3.2 第二步: 构建 vDSO link_map

**源文件**: `elf/setup-vdso.h:19-114`

在 `elf/rtld.c:1733` 中调用 `setup_vdso(main_map, &first_preload)`：

```
setup_vdso(main_map, first_preload):
  if dl_sysinfo_dso == NULL:
    return   // 无 vDSO (容器或旧内核)
  
  // ① 创建新的 link_map 对象
  l = _dl_new_object("", "", lt_library, NULL, __RTLD_VDSO, LM_ID_BASE)
  
  // ② 解析 vDSO 的 ELF 程序头
  l->l_phdr = dl_sysinfo_dso + dl_sysinfo_dso->e_phoff
  l->l_phnum = dl_sysinfo_dso->e_phnum
  
  for each program header:
    if PT_DYNAMIC:
      l->l_ld = ph->p_vaddr      // .dynamic 段
    if PT_LOAD:
      l->l_addr = ...             // 计算加载基址
    assert(ph->p_type != PT_TLS)  // vDSO 不能有 TLS 段！
  
  // ③ 设置 ELF 数据结构
  elf_get_dynamic_info(l, false, false)   // 解析 .dynamic
  _dl_setup_hash(l)                       // 构建哈希表
  l->l_relocated = 1                      // 已预重定位
  l->l_used = 1                           // 标记为使用中
  
  // ④ 初始化符号查找作用域
  l->l_local_scope[0]->r_nlist = 1
  l->l_local_scope[0]->r_list = &l->l_real
  
  // ⑤ 加入全局对象链表
  _dl_add_to_namespace_list(l, LM_ID_BASE)
  
  // ⑥ 记录为系统 vDSO 映射
  GLRO(dl_sysinfo_map) = l
```

### 3.3 第三步: 查找 vDSO 符号

**源文件**: `sysdeps/unix/sysv/linux/dl-vdso.h:37-56`

```c
static inline void *
dl_vdso_vsym(const char *name) {
  struct link_map *map = GLRO(dl_sysinfo_map);
  if (map == NULL)
    return NULL;
  
  // 创建弱引用符号（未找到不报错）
  ElfW(Sym) wsym = { 0 };
  wsym.st_info = ELFW(ST_INFO(STB_WEAK, STT_NOTYPE));
  
  // 版本匹配
  const struct r_found_version rfv = { VDSO_NAME, VDSO_HASH, 1, NULL };
  // AArch64: VDSO_NAME = "LINUX_2.6.39", VDSO_HASH = 123718537
  
  // 在 vDSO 的符号表中查找
  const ElfW(Sym) *ref = &wsym;
  lookup_t result = GLRO(dl_lookup_symbol_x)(
    name, map, &ref, map->l_local_scope, &rfv, 0, 0, NULL);
  
  return ref != NULL ? DL_SYMBOL_ADDRESS(result, ref) : NULL;
}
```

### 3.4 第四步: 注册 vDSO 函数指针

**源文件**: `sysdeps/unix/sysv/linux/dl-vdso-setup.h:23-56`

在 `elf/rtld.c:1736` 中调用 `setup_vdso_pointers()`：

```c
setup_vdso_pointers():
  // 按条件编译查找各 vDSO 函数
  GLRO(dl_vdso_clock_gettime64) = dl_vdso_vsym("__kernel_clock_gettime");
  GLRO(dl_vdso_clock_getres_time64) = dl_vdso_vsym("__kernel_clock_getres");
  GLRO(dl_vdso_gettimeofday) = dl_vdso_vsym("__kernel_gettimeofday");
  GLRO(dl_vdso_getrandom) = dl_vdso_vsym("__kernel_getrandom");
  // ... 其他函数（按架构条件编译）
```

---

## 四、全局状态与 GLRO 变量

vDSO 相关的全局状态存储在 `GLRO()` 宏访问的全局变量中：

| 变量 | 类型 | 含义 |
|------|------|------|
| `dl_sysinfo_dso` | `ElfW(Ehdr) *` | vDSO 的 ELF 头地址 (来自 AT_SYSINFO_EHDR) |
| `dl_sysinfo_map` | `struct link_map *` | vDSO 的 link_map 对象 |
| `dl_sysinfo` | `ElfW(Addr)` | 快速系统调用入口 (x86 用, AArch64 不用) |
| `dl_vdso_clock_gettime` | 函数指针 | vDSO 中 clock_gettime (32 位) |
| `dl_vdso_clock_gettime64` | 函数指针 | vDSO 中 clock_gettime (64 位) |
| `dl_vdso_gettimeofday` | 函数指针 | vDSO 中 gettimeofday |
| `dl_vdso_time` | 函数指针 | vDSO 中 time (部分架构) |
| `dl_vdso_getcpu` | 函数指针 | vDSO 中 getcpu (部分架构) |
| `dl_vdso_clock_getres_time64` | 函数指针 | vDSO 中 clock_getres |
| `dl_vdso_getrandom` | 函数指针 | vDSO 中 getrandom (较新内核) |

---

## 五、clock_gettime 的 vDSO 加速

### 5.1 调用链

**源文件**: `sysdeps/unix/sysv/linux/clock_gettime.c:28-86`

```
__clock_gettime64(clock_id, tp):
  // ① 尝试 64 位 vDSO
  vdso_time64 = GLRO(dl_vdso_clock_gettime64)
  if vdso_time64 != NULL:
    r = vdso_time64(clock_id, tp)       // 用户态直接调用！
    if r == 0: return 0
    return -r
  
  // ② 尝试 32 位 vDSO (兼容旧内核)
  vdso_time = GLRO(dl_vdso_clock_gettime)
  if vdso_time != NULL:
    r = vdso_time(clock_id, &tp32)
    if r == 0 && tp32.tv_sec >= 0:
      *tp = convert(tp32)
      return 0
  
  // ③ 最终回退: 真正的系统调用
  r = syscall(clock_gettime64, clock_id, tp)
  if r == 0: return 0
  if r == -ENOSYS:
    r = syscall(clock_gettime, clock_id, &tp32)  // 32位回退
```

### 5.2 vDSO 内部实现（内核侧，概念说明）

```c
// 内核 vDSO 中的 __kernel_clock_gettime 伪代码:
int __kernel_clock_gettime(clockid_t id, struct timespec *tp) {
  struct vdso_data *vd = __arch_get_vdso_data();  // 读取 vvar 页
  
  if (id == CLOCK_REALTIME || id == CLOCK_MONOTONIC) {
    do {
      seq = read_seqcount_begin(&vd->seq);        // 序列锁
      
      // 从共享数据页读取粗粒度时间
      sec = vd->basetime[id].sec;
      nsec = vd->basetime[id].nsec;
      
      // 读取硬件计数器获取精确偏移
      cycles = __arch_get_hw_counter();             // AArch64: mrs cntvct_el0
      nsec += (cycles - vd->cycle_last) * vd->mult;
      nsec >>= vd->shift;
      
    } while (read_seqcount_retry(&vd->seq, seq));  // 重试如果被更新
    
    // 归一化
    while (nsec >= NSEC_PER_SEC) { nsec -= NSEC_PER_SEC; sec++; }
    tp->tv_sec = sec;
    tp->tv_nsec = nsec;
    return 0;
  }
  
  return -ENOSYS;  // 不支持的时钟源，回退到系统调用
}
```

---

## 六、gettimeofday 的 IFUNC 加速

### 6.1 共享库版本 (SHARED)

**源文件**: `sysdeps/unix/sysv/linux/gettimeofday.c:31-44`

```c
// 定义系统调用回退
static int __gettimeofday_syscall(struct timeval *tv, void *tz) {
  if (__glibc_unlikely(tz != NULL))
    memset(tz, 0, sizeof *tz);
  return INLINE_SYSCALL_CALL(gettimeofday, tv, tz);
}

// IFUNC resolver: 运行时选择最佳实现
libc_ifunc(__gettimeofday,
  GLRO(dl_vdso_gettimeofday) != NULL
    ? VDSO_IFUNC_RET(GLRO(dl_vdso_gettimeofday))  // 直接跳转 vDSO
    : __gettimeofday_syscall)                        // 回退到 syscall
```

这种方式使 `gettimeofday` 的 PLT 条目直接指向 vDSO 函数，
省去了每次调用时的函数指针检查。

### 6.2 静态链接版本

```c
int __gettimeofday(struct timeval *tv, void *tz) {
  if (__glibc_unlikely(tz != NULL))
    memset(tz, 0, sizeof *tz);
  return INLINE_VSYSCALL(gettimeofday, 2, tv, tz);
}
```

---

## 七、INLINE_VSYSCALL 回退机制

**源文件**: `sysdeps/unix/sysv/linux/sysdep-vdso.h:29-66`

```c
// 带回退的 vDSO 调用宏
#define INLINE_VSYSCALL(name, nr, args...) ({
  long sc_ret;
  
  __typeof(GLRO(dl_vdso_##name)) vdsop = GLRO(dl_vdso_##name);
  if (vdsop != NULL) {
    sc_ret = vdsop(args);                       // 先尝试 vDSO
    if (!INTERNAL_SYSCALL_ERROR_P(sc_ret))
      goto out;                                  // 成功
    if (INTERNAL_SYSCALL_ERRNO(sc_ret) != ENOSYS)
      goto iserr;                                // 真正的错误
  }
  
  sc_ret = INTERNAL_SYSCALL_CALL(name, args);   // 回退到 syscall
  if (INTERNAL_SYSCALL_ERROR_P(sc_ret)) {
    iserr:
    __set_errno(INTERNAL_SYSCALL_ERRNO(sc_ret));
    sc_ret = -1L;
  }
  out:
  sc_ret;
})

// 不设置 errno 的版本
#define INTERNAL_VSYSCALL(name, nr, args...) ({
  long sc_ret = -ENOSYS;
  __typeof(GLRO(dl_vdso_##name)) vdsop = GLRO(dl_vdso_##name);
  if (vdsop != NULL)
    sc_ret = vdsop(args);
  if (sc_ret == -ENOSYS)
    sc_ret = INTERNAL_SYSCALL_CALL(name, args);
  sc_ret;
})
```

---

## 八、信号返回 (rt_sigreturn)

### 8.1 AArch64 上的信号返回

在 AArch64 上，内核 vDSO 提供 `__kernel_rt_sigreturn` 作为信号处理函数的
返回跳板。当信号处理函数执行 `return` 时，会跳转到此函数执行 `svc #0` 调用
`rt_sigreturn` 系统调用。

### 8.2 glibc 的 unwind 支持

**源文件**: `sysdeps/unix/sysv/linux/aarch64/uw-sigframe.h:47-54`

glibc 的异常展开器通过识别信号帧中的指令模式来判断是否在信号处理上下文中：

```c
// 检测内核信号跳板的指令模式
if (pc[0] != MOVZ_X8_8B || pc[1] != SVC_0)
  return _URC_END_OF_STACK;
// 如果匹配，解析信号帧中的寄存器上下文
```

---

## 九、AArch64 上可用的 vDSO 函数

| vDSO 函数名 | glibc GLRO 变量 | 用途 |
|-------------|-----------------|------|
| `__kernel_clock_gettime` | `dl_vdso_clock_gettime64` | 获取时间戳 (64 位) |
| `__kernel_clock_getres` | `dl_vdso_clock_getres_time64` | 获取时钟分辨率 |
| `__kernel_gettimeofday` | `dl_vdso_gettimeofday` | 获取当前时间 |
| `__kernel_getrandom` | `dl_vdso_getrandom` | 获取随机数 (6.11+) |
| `__kernel_rt_sigreturn` | — (内核自动设置) | 信号返回跳板 |

> **注意**: 不同架构的 vDSO 函数集不同。x86_64 还有 `__vdso_time`、
> `__vdso_getcpu`；AArch64 没有这两个。

---

## 十、完整调用时序图

```
程序启动 (execve)
    │
    ├─ 内核: 映射 vDSO + vvar 到进程地址空间
    ├─ 内核: 设置 auxv[AT_SYSINFO_EHDR] = vDSO 地址
    │
    ▼
_dl_parse_auxv (dl-parse_auxv.h:31-61)
    │ GLRO(dl_sysinfo_dso) = auxv[AT_SYSINFO_EHDR]
    ▼
setup_vdso (setup-vdso.h:19-114)        ← rtld.c:1733
    │ 解析 vDSO ELF 头
    │ 构建 link_map
    │ GLRO(dl_sysinfo_map) = l
    ▼
setup_vdso_pointers (dl-vdso-setup.h:23-56)   ← rtld.c:1736
    │ dl_vdso_vsym("__kernel_clock_gettime") → GLRO(dl_vdso_clock_gettime64)
    │ dl_vdso_vsym("__kernel_clock_getres")  → GLRO(dl_vdso_clock_getres_time64)
    │ dl_vdso_vsym("__kernel_gettimeofday")  → GLRO(dl_vdso_gettimeofday)
    │ dl_vdso_vsym("__kernel_getrandom")     → GLRO(dl_vdso_getrandom)
    ▼
应用调用 clock_gettime(CLOCK_REALTIME, &ts)
    │
    ▼
__clock_gettime64 (clock_gettime.c:28-86)
    │
    ├─ vDSO 可用? ──→ vdso_time64(clock_id, tp)   // 用户态调用, ~10 周期
    │                     │
    │                     ├─ 成功 → return 0
    │                     └─ -ENOSYS → 继续
    │
    └─ syscall(clock_gettime64, ...)               // 内核调用, ~500 周期
```

---

## 十一、与 x86_64 的对比

| 特性 | x86_64 | AArch64 |
|------|--------|---------|
| vDSO 名称 | `linux-vdso.so.1` | `linux-vdso.so.1` |
| 版本标签 | `LINUX_2.6` | `LINUX_2.6.39` |
| 符号前缀 | `__vdso_*` | `__kernel_*` |
| clock_gettime | `__vdso_clock_gettime` | `__kernel_clock_gettime` |
| time() | `__vdso_time` ✓ | 无 (通过 clock_gettime) |
| getcpu() | `__vdso_getcpu` ✓ | 无 |
| getrandom() | `__vdso_getrandom` ✓ (6.11+) | `__kernel_getrandom` ✓ (6.11+) |
| 硬件计数器 | `rdtsc` | `mrs cntvct_el0` |
| AT_SYSINFO | 用于 vsyscall 入口 | 不使用 |
| IFUNC 优化 | gettimeofday, time | gettimeofday |

---

## 十二、源文件速查

| 文件:行 | 内容 |
|---------|------|
| `elf/elf.h:1255-1256` | `AT_SYSINFO=32, AT_SYSINFO_EHDR=33` |
| `sysdeps/unix/sysv/linux/dl-parse_auxv.h:31-61` | `_dl_parse_auxv` (辅助向量解析) |
| `sysdeps/unix/sysv/linux/dl-parse_auxv.h:57` | `GLRO(dl_sysinfo_dso) = auxv[AT_SYSINFO_EHDR]` |
| `elf/setup-vdso.h:19-114` | `setup_vdso` (构建 vDSO link_map) |
| `elf/setup-vdso.h:32-33` | `_dl_new_object` 创建 vDSO 对象 |
| `elf/setup-vdso.h:67-68` | `elf_get_dynamic_info` + `_dl_setup_hash` |
| `elf/setup-vdso.h:107` | `GLRO(dl_sysinfo_map) = l` |
| `elf/rtld.c:1733` | `setup_vdso(main_map, &first_preload)` |
| `elf/rtld.c:1736` | `setup_vdso_pointers()` |
| `sysdeps/unix/sysv/linux/dl-vdso.h:37-56` | `dl_vdso_vsym` (vDSO 符号查找) |
| `sysdeps/unix/sysv/linux/dl-vdso.h:48` | 版本匹配: `VDSO_NAME, VDSO_HASH` |
| `sysdeps/unix/sysv/linux/dl-vdso-setup.h:23-56` | `setup_vdso_pointers` (注册函数指针) |
| `sysdeps/unix/sysv/linux/aarch64/sysdep.h:168-175` | AArch64 vDSO 符号名定义 |
| `sysdeps/unix/sysv/linux/clock_gettime.c:28-86` | `__clock_gettime64` (vDSO→syscall 回退) |
| `sysdeps/unix/sysv/linux/clock_gettime.c:37-46` | 64 位 vDSO 快速路径 |
| `sysdeps/unix/sysv/linux/clock_gettime.c:68` | syscall 回退路径 |
| `sysdeps/unix/sysv/linux/gettimeofday.c:31-44` | IFUNC resolver (vDSO 直跳) |
| `sysdeps/unix/sysv/linux/sysdep-vdso.h:29-54` | `INLINE_VSYSCALL` 宏 |
| `sysdeps/unix/sysv/linux/sysdep-vdso.h:56-66` | `INTERNAL_VSYSCALL` 宏 |
| `sysdeps/unix/sysv/linux/aarch64/uw-sigframe.h:47-54` | 信号帧 unwind 识别 |
| `elf/dl-support.c:225-259` | `_dl_aux_init` (非 rtld 的辅助向量处理) |
| `elf/dl-support.c:275` | 非 rtld 路径的 `setup_vdso_pointers()` |
