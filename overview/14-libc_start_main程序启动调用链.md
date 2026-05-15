# __libc_start_main 程序启动调用链分析

> 基于 clangd LSP 语义分析 (glibc 2.43.9000, AArch64)

## 概述

从内核完成 `execve()` 后，用户空间代码的第一个 C 函数入口到用户 `main()` 执行的完整路径。本文档覆盖静态链接程序的启动流程（动态链接程序的 ld.so 启动阶段见 `elf/10-ld.so启动调用链.md`）。

---

## 1. 启动全景

```
内核 execve()
  → _start (汇编入口, sysdeps/aarch64/start.S)
    → __libc_start_main()          [csu/libc-start.c:386]
      → __libc_start_main_impl()   [csu/libc-start.c:234]
        → [7个初始化阶段]
        → __libc_start_call_main() [nptl/libc_start_call_main.h:24]
          → main(argc, argv, envp)
          → exit(retval)
```

---

## 2. `__libc_start_main_impl()` 调用链 [csu/libc-start.c:234]

### 完整初始化序列（按执行顺序）:

```
__libc_start_main_impl(main, argc, argv, init, fini, rtld_fini, stack_end)
│
│ ===== 阶段 1: 基础环境 =====
│
├── _dl_aux_init()                  [elf/dl-support.c:226]
│   — 解析 auxv (辅助向量): AT_PHDR, AT_PAGESZ, AT_HWCAP 等
│
├── __tunables_init()               [elf/dl-tunables.c:294]
│   — 解析 GLIBC_TUNABLES 环境变量 (替代旧式 MALLOC_TRIM_THRESHOLD_ 等)
│
│ ===== 阶段 2: 静态 PIE 重定位 =====
│
├── _dl_relocate_static_pie()       [csu/static-reloc.c:23]
│   — 静态 PIE 可执行文件的自重定位
│
│ ===== 阶段 3: 安全初始化 =====
│
├── apply_irel()                    [csu/libc-start.c:76]
│   — 处理 IRELATIVE 重定位 (IFUNC 解析)
│
├── aarch64_libc_setup_tls()        [aarch64/libc-start.h:41]
│   — AArch64: 设置 TLS (Thread-Local Storage) + TPIDR_EL0
│
├── _dl_setup_stack_chk_guard()     [dl-osinfo.h:26]
│   — 初始化 stack canary (__stack_chk_guard)
│   — 从 AT_RANDOM 获取随机值
│
├── __pthread_initialize_minimal()  [ldsodefs.h:1214]
│   — 最小化 pthread 初始化 (设置主线程 TCB)
│
├── _dl_setup_pointer_guard()       [dl-osinfo.h:49]
│   — 初始化指针保护值 (用于 PTR_MANGLE/PTR_DEMANGLE)
│
│ ===== 阶段 4: C 运行时初始化 =====
│
├── __cxa_atexit(call_fini, ...)    [stdlib/cxa_atexit.c:66]
│   — 注册 fini 清理函数
│
├── __libc_init_first()             [csu/init-first.c:38]
│   — libc 内部第一次初始化
│
├── __libc_early_init()             [elf/libc_early_init.c:33]
│   ├── __ctype_init()              [ctype/ctype-info.c:36]
│   │   — 初始化 ctype 表 (LC_CTYPE)
│   ├── __pthread_early_init()      [nptl/pthread_early_init.h:28]
│   │   — 早期 pthread 配置 (stack size 等)
│   └── __getrandom_early_init()    [getrandom.c:237]
│       — 初始化 getrandom vDSO 加速
│
├── __libc_check_standard_fds()     [csu/check_fds.c:87]
│   — 确保 fd 0/1/2 已打开 (安全: 防止特权程序被劫持)
│
│ ===== 阶段 5: 用户初始化代码 =====
│
├── call_init(argc, argv, envp)     [csu/libc-start.c:173]
│   └── _init()                     [csu/libc-start.c:166]
│       — 执行 .init 段代码 + .init_array 中的构造函数
│
├── _dl_debug_initialize()          [elf/dl-debug.c:119]
│   — 初始化调试接口 (r_debug 结构, GDB 断点)
│
├── __cxa_atexit(rtld_fini, ...)    [stdlib/cxa_atexit.c:66]
│   — 注册 rtld 清理函数 (动态链接器的 _dl_fini)
│
│ ===== 阶段 6: 调用 main =====
│
└── __libc_start_call_main(main, argc, argv)  [nptl/libc_start_call_main.h:24]
    ├── _setjmp()                   [setjmp.h:45]
    │   — 设置 unwind 点 (用于 pthread_exit 从 main 返回)
    ├── main(argc, argv, envp)      ★ 用户代码入口
    │
    │ ===== 阶段 7: 退出 =====
    │
    ├── internal_signal_block_all() [internal-signals.h:79]
    │   — 阻塞所有信号 (退出期间不被打断)
    ├── __nptl_deallocate_tsd()     [nptl_deallocate_tsd.c:22]
    │   — 释放主线程 TSD (Thread-Specific Data)
    ├── futex_wake()                [futex-internal.h:187]
    │   — 唤醒等待主线程退出的 joiners
    └── exit(retval)                [stdlib/exit.c:146]
        └── __run_exit_handlers()   [stdlib/exit.c:43]
            ├── atexit 回调 (LIFO 顺序)
            ├── call_fini()         [csu/libc-start.c:194]
            │   └── _fini()         — .fini 段 + .fini_array 析构函数
            ├── free()              — 清理 atexit 链表内存
            └── _exit()             [_exit.c:26]
                → exit_group 系统调用 → 进程终止
```

---

## 3. 关键阶段详解

### 3.1 辅助向量解析: `_dl_aux_init()` [elf/dl-support.c:226]

内核通过栈传递的辅助信息：

| auxv 类型 | 用途 |
|-----------|------|
| AT_PHDR | 程序头表地址 |
| AT_PHNUM | 程序头数量 |
| AT_PAGESZ | 系统页大小 (4096/16384/65536) |
| AT_HWCAP/AT_HWCAP2 | 硬件能力位 (NEON, SVE, MTE...) |
| AT_RANDOM | 16字节随机数 (用于 canary + pointer guard) |
| AT_SYSINFO_EHDR | vDSO 映射地址 |
| AT_CLKTCK | 时钟频率 |

### 3.2 安全初始化

```
Stack Canary 初始化:
  AT_RANDOM → 取前 8 字节
  → __stack_chk_guard = random_value
  → 每个函数 prologue: str guard, [sp, #offset]
  → 每个函数 epilogue: 比较 → __stack_chk_fail()

Pointer Guard (PTR_MANGLE):
  AT_RANDOM → 取后 8 字节
  → __pointer_chk_guard = random_value
  → 加密: rotated_ptr = (ptr ^ guard) <<< 某位数
  → 用于保护 longjmp buffer、atexit 函数指针
```

### 3.3 `__libc_start_call_main` 中的 setjmp [nptl/libc_start_call_main.h:24]

NPTL 版本使用 `_setjmp` 设置 unwind 点：

```c
// 简化逻辑:
if (_setjmp(self->canceltype_buf) == 0) {
    result = main(argc, argv, __environ);
} else {
    // pthread_exit() 从 main 中被调用时到达这里
    result = 0;
}
```

这确保 `pthread_exit()` 在 main 线程中也能正确清理。

### 3.4 退出处理: `__run_exit_handlers()` [stdlib/exit.c:43]

```
退出回调执行顺序 (LIFO):
  1. 最后注册的 atexit/on_exit 回调
  2. ...
  3. 最先注册的回调
  4. call_fini() → .fini_array (逆序)
  5. stdio flush (__libc_atexit 中注册)
  6. _exit() → 内核
```

---

## 4. 动态链接 vs 静态链接差异

| 特性 | 动态链接 | 静态链接 |
|------|----------|----------|
| 入口 | ld.so `_dl_start` → ... → `_start` | 直接 `_start` |
| TLS 初始化 | ld.so 中完成 | `aarch64_libc_setup_tls()` |
| IFUNC 解析 | ld.so 重定位阶段 | `apply_irel()` |
| 构造函数 | `_dl_init()` + `call_init()` | 仅 `call_init()` |
| PIE 重定位 | ld.so 负责 | `_dl_relocate_static_pie()` |
| tunables | ld.so `__tunables_init` (更早) | `__libc_start_main_impl` 中 |

---

## 5. AArch64 特定行为

### 5.1 `_start` 汇编 [sysdeps/aarch64/start.S]

```asm
_start:
    mov     x29, #0         // 清除 frame pointer (栈回溯终止)
    mov     x30, #0         // 清除 return address
    mov     x0, sp          // argc 在栈顶
    // ... 设置参数 ...
    bl      __libc_start_main
    bl      abort           // 不应返回
```

### 5.2 TLS 布局 (AArch64)

```
TPIDR_EL0 → ┌──────────────────┐
             │ TCB (pthread)    │ ← Thread Control Block
             ├──────────────────┤
             │ DTV pointer      │ ← Dynamic Thread Vector
             ├──────────────────┤
             │ stack canary     │
             │ pointer guard    │
             ├──────────────────┤
             │ TLS Block 1      │ ← 主模块 TLS (.tdata/.tbss)
             │ TLS Block 2      │ ← libpthread TLS
             │ ...              │
             └──────────────────┘
```

---

## 6. 源码位置索引

| 函数 | 文件 | 行号 |
|------|------|------|
| `_start` | sysdeps/aarch64/start.S | — |
| `__libc_start_main` | csu/libc-start.c | 386 |
| `__libc_start_main_impl` | csu/libc-start.c | 234 |
| `__libc_start_call_main` | sysdeps/nptl/libc_start_call_main.h | 24 |
| `_dl_aux_init` | elf/dl-support.c | 226 |
| `__tunables_init` | elf/dl-tunables.c | 294 |
| `_dl_relocate_static_pie` | csu/static-reloc.c | 23 |
| `apply_irel` | csu/libc-start.c | 76 |
| `aarch64_libc_setup_tls` | sysdeps/unix/sysv/linux/aarch64/libc-start.h | 41 |
| `_dl_setup_stack_chk_guard` | sysdeps/unix/sysv/linux/dl-osinfo.h | 26 |
| `_dl_setup_pointer_guard` | sysdeps/unix/sysv/linux/dl-osinfo.h | 49 |
| `__libc_early_init` | elf/libc_early_init.c | 33 |
| `__ctype_init` | ctype/ctype-info.c | 36 |
| `__pthread_early_init` | sysdeps/nptl/pthread_early_init.h | 28 |
| `__getrandom_early_init` | sysdeps/unix/sysv/linux/getrandom.c | 237 |
| `__libc_check_standard_fds` | csu/check_fds.c | 87 |
| `call_init` | csu/libc-start.c | 173 |
| `call_fini` | csu/libc-start.c | 194 |
| `_dl_debug_initialize` | elf/dl-debug.c | 119 |
| `__run_exit_handlers` | stdlib/exit.c | 43 |
| `_exit` | sysdeps/unix/sysv/linux/_exit.c | 26 |
| `__nptl_deallocate_tsd` | nptl/nptl_deallocate_tsd.c | 22 |
| `exit` | stdlib/exit.c | 146 |
