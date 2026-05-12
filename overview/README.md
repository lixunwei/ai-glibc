# GNU C Library (glibc) 源码模块分析

> 版本: glibc 2.43.9000 (development)  
> 分析时间: 2026-05-07  
> 索引: .ai-search/ (zoekt + ctags + cscope)

## 概述

GNU C Library (glibc) 是 GNU/Linux 系统的标准 C 库，为所有 C/C++ 程序提供系统 API。
它实现了 POSIX.1、ISO C11/C23 标准，并提供大量 GNU 扩展功能。

glibc 包含约 **20,786 个源文件**，**133,907 个符号定义**，涵盖从内存分配到网络通信的
完整系统编程接口。

## 模块分类

| 分类 | 模块 | 说明文档 |
|------|------|----------|
| 核心运行时 | elf, malloc, nptl, csu | [01-核心运行时.md](01-核心运行时.md) |
| 标准库 | stdlib, string, math, stdio-common | [02-标准库.md](02-标准库.md) |
| I/O 子系统 | libio, io, stdio-common | [03-IO子系统.md](03-IO子系统.md) |
| 网络通信 | socket, inet, resolv, nss, sunrpc | [04-网络通信.md](04-网络通信.md) |
| 系统接口 | posix, signal, time, resource, sysvipc, misc | [05-系统接口.md](05-系统接口.md) |
| 国际化 | locale, iconv, iconvdata, intl, wcsmbs, ctype, wctype, catgets | [06-国际化.md](06-国际化.md) |
| 基础设施 | debug, sysdeps, gmon, setjmp, termios, argp, soft-fp | [07-基础设施.md](07-基础设施.md) |
| 辅助组件 | login, dlfcn, dirent, nscd, rt, hurd/htl/mach | [08-辅助组件.md](08-辅助组件.md) |

## 构建系统

glibc 使用 GNU Make 构建系统:
- **顶层 Makefile**: 驱动所有子目录的构建
- **Makeconfig**: 定义 `objdir`、`objpfx` 等路径变量
- **子目录 Makefile**: 每个模块独立的 Makefile 定义 `routines`、`headers`、`tests`
- **sysdeps 机制**: 通过目录搜索路径实现架构特定的代码覆盖

## 架构支持

glibc 支持以下 Linux 架构:
- x86_64, i386, aarch64, arm, riscv64/32
- mips/mips64, powerpc/powerpc64, s390x
- loongarch64, sparc/sparc64, sh, arc, csky, m68k, microblaze, or1k, hppa, alpha

## 子系统深度分析

以下核心子系统有独立的深入分析文档:

| 子系统 | 文档目录 | 内容 |
|--------|----------|------|
| **NPTL 线程库** | [../pthread/](../pthread/) | 线程创建、互斥锁、条件变量、屏障、TSD、信号量、TLS、Futex、栈空间布局、pthread_cancel/cleanup 深度分析 等 13 篇 |
| **ptmalloc2 内存分配器** | [../malloc/](../malloc/) | 数据结构、分配路径、释放路径、Arena/Tcache、调优调试 5 篇 |
| **ld.so 动态链接器** | [../elf/](../elf/) | 启动加载、符号解析、重定位绑定、dlopen、AArch64 重定位、vDSO 机制、dlopen 搜索/dladdr/dl_iterate_phdr 7 篇 |
| **信号子系统** | [../signal/](../signal/) | 数据结构、sigaction 调用链、信号掩码、线程交互、AArch64 信号帧、SIGCANCEL/SIGSETXID 内部信号 6 篇 |
| **stdio/IO 子系统** | [../stdio/](../stdio/) | FILE 结构与 vtable、流生命周期与缓冲、读写路径与刷新机制 3 篇 |

## 专题分析

| 文档 | 内容 |
|------|------|
| [10-AArch64硬件扩展支持.md](10-AArch64硬件扩展支持.md) | SVE/SME/MOPS/MTE/BTI/PAC/GCS 七大扩展的 glibc 支持实现 |
| [11-全局架构关系图.md](11-全局架构关系图.md) | 库文件构成、模块分层、启动时序、跨子系统交互、数据流图、AArch64 sysdeps 分层、HWCAP 传播、vDSO 集成 |
| [12-_GNU_SOURCE宏分析.md](12-_GNU_SOURCE宏分析.md) | 特性测试宏机制、features.h 处理流程、__USE_GNU 守卫的 227 个功能块分类汇总 |

## 辅助文档

| 文档 | 内容 |
|------|------|
| [../交叉索引.md](../交叉索引.md) | 按主题检索各概念在全部文档中的出现位置（13 个分类、数据结构/函数/源文件索引） |
| [../术语表.md](../术语表.md) | 核心术语中英对照与释义（8 个分类、130+ 术语） |

## 索引说明

本分析使用以下工具建立源码索引:
- **zoekt**: 全文三元组索引，支持正则搜索
- **ctags-universal**: 符号索引（函数、结构体、宏等）
- **cscope**: 调用图和交叉引用数据库

索引位于 `.ai-search/` 目录下。
