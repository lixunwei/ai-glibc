# ldconfig 与 ld.so.cache 管理机制

> 基于 glibc 2.43.9000 源码分析  
> 涵盖 ldconfig 工作流程、ld.so.cache 生成/更新/查找、数据格式、glibc-hwcaps 支持

---

## 目录

1. [概述](#1-概述)
2. [ldconfig 主流程](#2-ldconfig-主流程)
3. [配置文件解析](#3-配置文件解析)
4. [目录扫描与库发现](#4-目录扫描与库发现)
5. [ELF 文件处理与 soname 提取](#5-elf-文件处理与-soname-提取)
6. [ld.so.cache 数据格式](#6-ldsocache-数据格式)
7. [缓存生成——save_cache](#7-缓存生成save_cache)
8. [运行时缓存查找——ld.so 侧](#8-运行时缓存查找ldso-侧)
9. [glibc-hwcaps 支持](#9-glibc-hwcaps-支持)
10. [辅助缓存](#10-辅助缓存)
11. [缓存管理操作总结](#11-缓存管理操作总结)
12. [源文件速查表](#12-源文件速查表)

---

## 1. 概述

### 1.1 角色分工

```
┌─────────────────────────────────────────────────────────────────┐
│                    ld.so.cache 生态系统                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ldconfig (管理工具)              ld.so (运行时消费者)            │
│  ├─ 扫描目录                     ├─ mmap 缓存文件                │
│  ├─ 读取 ELF 提取 soname         ├─ 二分搜索库名                 │
│  ├─ 创建/更新 soname 符号链接    ├─ 版本感知比较                 │
│  ├─ 排序并写入 ld.so.cache       └─ hwcap 优先级选择             │
│  └─ 原子替换缓存文件                                            │
│                                                                 │
│  输入:                            输出:                          │
│  ├─ /etc/ld.so.conf              └─ /etc/ld.so.cache            │
│  ├─ 命令行目录参数                                              │
│  └─ 系统默认目录 (SLIBDIR/LIBDIR)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 为什么需要 ld.so.cache

- **性能**：避免 dlopen/程序启动时对数十个目录逐一 stat + open
- **预排序**：缓存中条目已按库名排序，支持 O(log n) 二分搜索
- **版本感知**：数字部分按数值排序（`libfoo.so.10` > `libfoo.so.9`）
- **HWCAP 优化**：优先使用针对当前 CPU 优化的库版本

---

## 2. ldconfig 主流程

### 2.1 main() 完整流程

`elf/ldconfig.c:1030-1175`：

```
main(argc, argv):
  │
  ├─ 1. 设置 locale (LC_ALL="", LC_COLLATE="C") + textdomain
  │     ldconfig.c:1033-1041
  │
  ├─ 2. argp_parse() 解析命令行参数
  │     ldconfig.c:1043-1045
  │
  ├─ 3. 处理额外目录参数
  │     ldconfig.c:1049-1059
  │
  ├─ 4. chroot 处理 (-r ROOT)
  │     ldconfig.c:1061-1080
  │
  ├─ 5. 设置默认路径
  │     cache_file = /etc/ld.so.cache   ldconfig.c:1083-1087
  │     config_file = /etc/ld.so.conf   ldconfig.c:1089-1090
  │
  ├─ 6. 分支处理:
  │     ├─ -p: print_cache() 并退出     ldconfig.c:1092-1106
  │     ├─ -l: manual_link() 并退出     ldconfig.c:1131-1139
  │     └─ 正常模式: 继续
  │
  ├─ 7. init_cache()                    ldconfig.c:1143-1144
  │     初始化缓存条目链表为空
  │
  ├─ 8. 解析配置文件 + 添加系统目录
  │     ldconfig_parse_config()          ldconfig.c:1148
  │     add_system_dir(SLIBDIR)          ldconfig.c:1151
  │     add_system_dir(LIBDIR)           ldconfig.c:1152-1153
  │
  ├─ 9. 加载辅助缓存
  │     load_aux_cache()                 ldconfig.c:1160-1163
  │
  ├─ 10. 扫描所有目录
  │      search_dirs()                   ldconfig.c:1165
  │
  └─ 11. 保存缓存
         save_cache(cache_file)          ldconfig.c:1169
         save_aux_cache(aux_cache_file)  ldconfig.c:1170-1171
```

### 2.2 命令行选项

| 选项 | 变量 | 说明 |
|------|------|------|
| `-p` | `opt_print_cache` | 打印缓存内容 |
| `-v` | `opt_verbose` | 详细输出 |
| `-N` | `opt_build_cache=0` | 不重建缓存 |
| `-X` | `opt_link=0` | 不更新符号链接 |
| `-r ROOT` | `opt_chroot` | 使用 ROOT 作为根目录 |
| `-C CACHE` | `cache_file` | 指定缓存文件路径 |
| `-f CONF` | `config_file` | 指定配置文件路径 |
| `-n` | `opt_only_cline` | 仅处理命令行目录 |
| `-l` | `opt_manual_link` | 手动链接模式 |
| `-c FORMAT` | `opt_format` | 缓存格式：new/old/compat |
| `-i` | `opt_ignore_aux_cache` | 忽略辅助缓存 |

定义位置：`ldconfig.c:122-136`（argp_option 数组）。

---

## 3. 配置文件解析

### 3.1 /etc/ld.so.conf 格式

```
# 注释行
/usr/local/lib
/opt/myapp/lib

# 包含其他配置
include /etc/ld.so.conf.d/*.conf
```

### 3.2 解析流程

`elf/ldconfig-parse.c:61-143`（`ldconfig_parse_config_1`）：

```
ldconfig_parse_config_1(config_file, chroot, callback):
  │
  ├─ 打开配置文件（chroot 感知）
  │   ldconfig-parse.c:71-94
  │
  ├─ 逐行读取 (getline):
  │   ldconfig-parse.c:103-107
  │   │
  │   ├─ 去除注释 (#) 和行首空白
  │   │   ldconfig-parse.c:110-123
  │   │
  │   ├─ 跳过空行
  │   │   ldconfig-parse.c:123-125
  │   │
  │   ├─ "include <glob>" 处理:
  │   │   → parse_conf_include()
  │   │   ldconfig-parse.c:127-134
  │   │
  │   └─ 普通路径行:
  │       → callback(line, file, lineno)
  │       ldconfig-parse.c:135-136
  │
  └─ 关闭文件
```

### 3.3 include 指令处理

`elf/ldconfig-parse.c:147-204`（`parse_conf_include`）：

```
parse_conf_include(pattern, ...):
  │
  ├─ 安全检查: chroot 模式下 include 路径必须是绝对路径
  │   ldconfig-parse.c:152-155
  │
  ├─ 解析相对路径（相对于当前配置文件目录）
  │   ldconfig-parse.c:157-163
  │
  ├─ glob64() 展开通配符
  │   ldconfig-parse.c:165-177
  │
  └─ 递归调用 ldconfig_parse_config_1() 处理每个匹配文件
      ldconfig-parse.c:178-184
```

### 3.4 目录注册

每条路径通过 `add_dir_1()` 注册（`ldconfig.c:366-421`）：

```
add_dir_1(path):
  │
  ├─ 去除尾部空白和 '/'
  ├─ chroot 下规范化路径
  ├─ stat() 获取 inode/dev
  ├─ add_single_dir() 去重（按 dev/ino）
  │   ldconfig.c:261-297
  └─ add_glibc_hwcaps_subdirectories()
      扫描 <path>/glibc-hwcaps/* 子目录
      ldconfig.c:301-363
```

---

## 4. 目录扫描与库发现

### 4.1 search_dir() 算法

`elf/ldconfig.c:700-1008`，这是 ldconfig 的核心扫描函数：

```
search_dir(entry):
  │
  ├─ 1. opendir(entry->path)
  │
  ├─ 2. 遍历 readdir64():
  │     │
  │     ├─ 过滤:
  │     │   ├─ 仅接受 DT_REG / DT_LNK / DT_DIR / DT_UNKNOWN
  │     │   ├─ _dl_is_dso() 检查文件名是否像共享库
  │     │   │   (包含 ".so" 且不是 ".o" 等)
  │     │   └─ skip_dso_based_on_name() 排除临时文件
  │     │       (.#prelink# / RPM 分号文件 / .dpkg-new/tmp)
  │     │       ldconfig.c:676-698
  │     │
  │     ├─ 符号链接处理:
  │     │   ├─ 解析目标，stat 获取真实信息
  │     │   ├─ 如果目标不存在 → 删除过期链接
  │     │   └─ 更新 lstat_buf 为目标的 dev/ino/size/ctime
  │     │
  │     ├─ 获取元数据:
  │     │   ├─ 优先查辅助缓存 (search_aux_cache)
  │     │   │   ldconfig.c:844
  │     │   └─ 否则 process_file() 读取 ELF
  │     │       ldconfig.c:846-847
  │     │
  │     ├─ soname 决策:
  │     │   如果 ELF 无 DT_SONAME → 使用文件名作为 soname
  │     │   ldconfig.c:857-858
  │     │
  │     ├─ 符号链接 vs 普通文件判定:
  │     │   链接目标 != soname 且不是 .so 开发链接 → 视为普通文件
  │     │   ldconfig.c:860-897
  │     │
  │     └─ 去重（按 soname）:
  │         已有同名条目时，优先选择:
  │         1. 普通文件 > 符号链接
  │         2. 更新的文件（_dl_cache_libcmp 比较文件名）
  │         ldconfig.c:915-956
  │
  ├─ 3. 扫描完成后:
  │     ├─ 普通目录: 为每个库创建 soname 符号链接
  │     │   create_links(dir, path, filename, soname)
  │     │   ldconfig.c:975-980
  │     │
  │     ├─ glibc-hwcaps 子目录: 不创建链接
  │     │   ldconfig.c:983-993
  │     │
  │     └─ 加入缓存:
  │         add_to_cache(path, filename, soname, flags, isa_level, hwcaps)
  │         ldconfig.c:991-993
  │
  └─ 4. 释放资源
```

### 4.2 符号链接管理

`create_links()` 函数（`ldconfig.c:461-544`）：

```
create_links(real_path, path, libname, soname):
  │
  ├─ 拼接完整路径:
  │   full_libname = path/libname     (如 /usr/lib/libfoo.so.1.2)
  │   full_soname  = path/soname      (如 /usr/lib/libfoo.so.1)
  │
  ├─ 检查现有链接:
  │   ├─ soname 已存在且指向正确文件 → 不操作
  │   ├─ soname 存在但不是符号链接 → 跳过（报警告）
  │   └─ soname 不存在或指向错误 → 需要更新
  │
  └─ 更新链接:
      unlink(soname)
      symlink(libname, soname)
```

示例：

```
/usr/lib/
  libfoo.so.1.2.3    (实际文件)
  libfoo.so.1     → libfoo.so.1.2.3   (ldconfig 创建的 soname 链接)
  libfoo.so       → libfoo.so.1       (开发包安装的链接，ldconfig 不管理)
```

---

## 5. ELF 文件处理与 soname 提取

### 5.1 入口分层

```
search_dir()
  → process_file()              elf/readlib.c:58-170
      → process_elf_file()      elf/readelflib.c:41-250
```

`process_file()` 处理多种格式（`readlib.c:58-170`）：
- 检查 ELF 魔数 → 调用 `process_elf_file()`
- 检查链接器脚本（如 `GROUP ( ... )`）→ 识别后跳过（抑制错误输出）
- a.out 格式 → 合成 soname 并返回

### 5.2 ELF 处理流程

`elf/readelflib.c:41-250`：

```
process_elf_file(file_name, lib, &flag, &isa_level, &soname, ...):
  │
  ├─ 1. 验证 ELF class（32/64位匹配）
  │     readelflib.c:56-70
  │
  ├─ 2. 验证类型必须为 ET_DYN（共享库）
  │     readelflib.c:72-77
  │
  ├─ 3. 遍历程序头表:
  │     ├─ PT_DYNAMIC → 记录 .dynamic 段偏移和大小
  │     │   readelflib.c:97-105
  │     ├─ PT_INTERP → 记录解释器段
  │     │   readelflib.c:107-110
  │     └─ PT_GNU_PROPERTY → 解析 GNU 属性笔记
  │         → read_gnu_property() 提取 ISA 级别
  │         readelflib.c:112-182
  │
  ├─ 4. 从 .dynamic 段定位字符串表:
  │     扫描 DT_STRTAB 得到 strtab 偏移
  │     通过 PT_LOAD 段的 p_vaddr/p_offset 映射到文件偏移
  │     readelflib.c:190-234
  │
  └─ 5. 提取 DT_SONAME:
        扫描 .dynamic 中的 DT_SONAME 条目
        *soname = strdup(strtab + d_val)
        readelflib.c:236-246
```

### 5.3 ISA 级别检测

通过 `PT_GNU_PROPERTY` 段中的 GNU 属性笔记提取 ISA 级别信息
（如 x86-64-v2/v3/v4），存储在 `isa_level` 中，用于缓存条目的
hwcap 字段高 32 位。

---

## 6. ld.so.cache 数据格式

### 6.1 文件整体结构

```
ld.so.cache 文件布局：
┌────────────────────────────────────────────────┐
│  [可选] 旧格式 (struct cache_file)             │
│  ├─ magic: "ld.so-1.7.0\0"      (11+1 字节)  │
│  ├─ nlibs: 条目数               (uint32_t)     │
│  └─ libs[nlibs]: file_entry 数组               │
│     每条 12 字节: flags + key偏移 + value偏移   │
├────────────────────────────────────────────────┤
│  [对齐填充]                                     │
├────────────────────────────────────────────────┤
│  新格式 (struct cache_file_new)                 │
│  ├─ magic: "glibc-ld.so.cache"  (17 字节)     │
│  ├─ version: "1.1"              (3 字节)       │
│  ├─ nlibs: 条目数               (uint32_t)     │
│  ├─ len_strings: 字符串表大小   (uint32_t)     │
│  ├─ flags: 字节序标志           (uint8_t)      │
│  ├─ extension_offset            (uint32_t)     │
│  └─ libs[nlibs]: file_entry_new 数组           │
│     每条 24 字节: flags+key+value+osver+hwcap  │
├────────────────────────────────────────────────┤
│  字符串表                                       │
│  (库名和路径的连续存储)                          │
├────────────────────────────────────────────────┤
│  [对齐到 4 字节边界]                            │
├────────────────────────────────────────────────┤
│  扩展区 (cache_extension)                       │
│  ├─ magic: 0xd34d5e2c                          │
│  ├─ count: 扩展段数量                           │
│  └─ sections[]: cache_extension_section 数组    │
│     ├─ tag: 扩展类型                            │
│     ├─ flags + offset + size                    │
│     └─ glibc-hwcaps 子目录名列表                │
└────────────────────────────────────────────────┘
```

### 6.2 新格式条目结构

`sysdeps/generic/dl-cache.h:85-100`：

```c
struct file_entry_new {
  union {
    struct file_entry entry;       // 兼容旧格式
    struct {
      int32_t  flags;              // 库类型标志 (FLAG_ELF_LIBC6 等)
      uint32_t key;                // 库名在字符串表中的偏移
      uint32_t value;              // 文件路径在字符串表中的偏移
    };
  };
  uint32_t osversion_unused;       // 已废弃的 OS 版本字段
  uint64_t hwcap;                  // HWCAP 或 glibc-hwcaps 索引
};
```

### 6.3 hwcap 字段编码

`sysdeps/generic/dl-cache.h:102-127`：

```
hwcap 字段 (64位):
┌────────────────────────────────────────────────────────────────┐
│  bit 62: DL_CACHE_HWCAP_EXTENSION                             │
│    0 → 低 32 位为传统 HWCAP 位掩码（已废弃，glibc 2.37 移除） │
│    1 → 低 32 位为 glibc-hwcaps 扩展区的索引                   │
│                                                                │
│  bit [63:32]: ISA 级别信息（如 x86-64-v2 = 2）                │
│  bit [31:0]:  hwcap 索引或传统位掩码                           │
└────────────────────────────────────────────────────────────────┘
```

### 6.4 缓存文件头

新格式头（`dl-cache.h:157-182`）：

```c
struct cache_file_new {
  char     magic[17];      // "glibc-ld.so.cache"
  char     version[3];     // "1.1"
  uint32_t nlibs;          // 条目数
  uint32_t len_strings;    // 字符串表大小
  uint8_t  flags;          // 字节序标志
  uint8_t  padding[3];
  uint32_t extension_offset; // 扩展区偏移
  uint32_t unused[3];
  struct file_entry_new libs[0]; // 条目数组
};
// sizeof == 48
```

### 6.5 三种格式模式

| 格式 | ldconfig 选项 | 说明 |
|------|-------------|------|
| `new` (默认) | `-c new` | 仅新格式，推荐 |
| `old` | `-c old` | 仅旧格式（兼容古老系统） |
| `compat` | `-c compat` | 旧+新混合（旧在前，新在后） |

ld.so 读取时自动检测格式（`dl-cache.c:397-449`）。

---

## 7. 缓存生成——save_cache

### 7.1 条目收集

`add_to_cache()`（`cache.c:753-800`）在 `search_dir()` 扫描时被调用：

```c
void add_to_cache(path, filename, soname, flags, isa_level, hwcaps)
{
  // 1. 拼接完整路径 path/filename → 存入字符串表
  // 2. soname 存入字符串表
  // 3. 创建 cache_entry 结构
  // 4. 按排序位置插入链表（维持有序）
  //    使用 compare() 比较
}
```

### 7.2 排序规则

`compare()` 函数（`cache.c:411-439`）定义缓存条目的排序方式：

```
排序优先级:
1. 库名（soname）: 使用 _dl_cache_libcmp() 逆序
   → 数字部分按数值比较，确保版本号正确排序
2. flags: 数值大的在前（高标志优先）
3. glibc-hwcaps 条目在非 hwcaps 条目之前
   → 确保 search_cache 优先匹配 hwcaps 版本
4. hwcaps 子目录名: 字典序
```

**关键**: 排序使用**逆序**调用 `_dl_cache_libcmp`（交换 e1、e2），
因为二分搜索算法要求条目按库名降序排列。

### 7.3 save_cache() 完整流程

`cache.c:529-750`：

```
save_cache(cache_name):
  │
  ├─ 1. 分配 glibc-hwcaps 索引号
  │     assign_glibc_hwcaps_indices()
  │     cache.c:535
  │
  ├─ 2. 统计条目数
  │     cache.c:539-549
  │
  ├─ 3. 字符串表最终化
  │     stringtable_finalize()
  │     cache.c:551-552
  │
  ├─ 4. 构建旧格式头部和数组（如果需要）
  │     填充 struct cache_file + file_entry[]
  │     cache.c:554-577
  │
  ├─ 5. 构建新格式头部（如果需要）
  │     填充 struct cache_file_new 头 + 条目数组
  │     cache.c:579-599
  │
  ├─ 6. 填充条目 + 设置扩展偏移
  │     遍历链表，填充旧/新格式条目、hwcap 字段
  │     计算 extension_offset
  │     cache.c:617-673
  │
  ├─ 6. 写入临时文件:
  │     temp_name = cache_name + "~"
  │     open(temp_name, O_CREAT|O_WRONLY|O_TRUNC)
  │     cache.c:678-683
  │     │
  │     ├─ 写旧格式段     cache.c:689-694
  │     ├─ 写对齐填充     cache.c:697-704
  │     ├─ 写新格式段     cache.c:705-707
  │     ├─ 写字符串表     cache.c:710-712
  │     └─ 写扩展区       cache.c:714-721
  │
  ├─ 7. 设置权限
  │     chmod(temp_name, 0644)
  │     cache.c:724-727
  │
  ├─ 8. fsync + close 确保数据落盘
  │     cache.c:730-731
  │
  └─ 9. 原子替换
        rename(temp_name, cache_name)
        cache.c:734-736
```

### 7.4 原子替换机制

```
  写入:   /etc/ld.so.cache~  (临时文件)
     │
     ├─ write() 所有数据
     ├─ chmod() 设置可读
     ├─ fsync() 确保落盘
     └─ close()
     │
     ▼
  替换:   rename("/etc/ld.so.cache~", "/etc/ld.so.cache")
```

`rename()` 在同一文件系统上是**原子操作**。这意味着：
- ld.so 在读取时不会看到半写的缓存
- 如果 ldconfig 中途崩溃，旧缓存仍然完好
- 正在读取旧缓存的 ld.so 进程不受影响（mmap 引用旧 inode）

---

## 8. 运行时缓存查找——ld.so 侧

### 8.1 入口函数

`_dl_load_cache_lookup()`（`dl-cache.c:384-499`）在 dlopen 搜索步骤 5 中被调用：

```
_dl_load_cache_lookup(name):
  │
  ├─ 1. 首次调用时加载缓存:
  │     _dl_sysdep_read_whole_file(LD_SO_CACHE, &cachesize, PROT_READ)
  │     → 实质为 open + mmap 整个文件
  │     dl-cache.c:391-396
  │
  ├─ 2. 格式检测:
  │     ├─ 纯新格式: magic == "glibc-ld.so.cache1.1"
  │     ├─ 旧格式: magic == "ld.so-1.7.0"
  │     └─ 混合格式: 旧格式头后接新格式
  │     dl-cache.c:397-449
  │
  ├─ 3. 调用 search_cache():
  │     传入字符串表、条目数组、条目大小、库名
  │     dl-cache.c:460-483
  │
  └─ 4. 返回路径:
        复制到 alloca + strdup（避免 malloc 重入问题）
        dl-cache.c:490-498
```

### 8.2 二分搜索算法

`search_cache()`（`dl-cache.c:194-336`）：

```
search_cache(string_table, nlibs, entry_size, name):
  │
  ├─ 1. 标准二分搜索:
  │     left=0, right=nlibs-1
  │     while (left <= right):
  │       middle = (left + right) / 2
  │       cmpres = _dl_cache_libcmp(name, string_table[key])
  │       if cmpres < 0: left = middle + 1
  │       elif cmpres > 0: right = middle - 1
  │       else: 找到匹配 → 跳出
  │     dl-cache.c:206-222
  │
  ├─ 2. 向前扫描所有同名条目:
  │     找到匹配后，向前移动到第一个同名条目
  │     dl-cache.c:226-236
  │
  ├─ 3. 遍历所有同名条目，选择最佳匹配:
  │     │
  │     ├─ 验证字符串表偏移合法性
  │     ├─ 检查 flags 匹配 (_DL_CACHE_DEFAULT_ID)
  │     ├─ glibc-hwcaps 条目优先:
  │     │   ├─ 检查 ISA 级别兼容性
  │     │   ├─ 计算优先级 (glibc_hwcaps_priority)
  │     │   └─ 选择最高优先级的条目
  │     │   dl-cache.c:270-310
  │     │
  │     └─ 普通条目 (hwcap==0):
  │         如果已有 hwcaps 匹配则停止
  │         否则使用此普通条目
  │         dl-cache.c:315-332
  │
  └─ 4. 返回最佳匹配的路径字符串
```

### 8.3 版本感知比较

`_dl_cache_libcmp()`（`dl-cache.c:338-374`）：

```c
int _dl_cache_libcmp(const char *p1, const char *p2)
{
  // 逐字符比较，但遇到数字时按数值比较
  while (*p1 != '\0') {
    if (isdigit(*p1) && isdigit(*p2)) {
      // 解析完整数字，按数值比较
      val1 = atoi(p1); val2 = atoi(p2);
      if (val1 != val2) return val1 - val2;
    } else if (isdigit(*p1)) return 1;
      else if (isdigit(*p2)) return -1;
      else if (*p1 != *p2) return *p1 - *p2;
    ...
  }
}
```

效果：
```
libfoo.so.1   < libfoo.so.2      (数值: 1 < 2)
libfoo.so.9   < libfoo.so.10     (数值: 9 < 10，字典序会得 "9" > "10")
libfoo.so.1.2 < libfoo.so.1.10   (逐段数值比较)
```

### 8.4 缓存卸载

`_dl_unload_cache()`（`dl-cache.c:506-518`）：

```c
void _dl_unload_cache(void)
{
  if (cache != NULL && cache != (void *) -1)
    {
      __munmap(cache, cachesize);
      cache = NULL;
    }
  // 同时重置 glibc_hwcaps_priorities 数组
  glibc_hwcaps_priorities_length = 0;
}
```

仅在不支持 `MAP_COPY` 的系统上可用。通常 ld.so 在程序启动完成后
调用此函数释放缓存映射。

---

## 9. glibc-hwcaps 支持

### 9.1 目录结构

```
/usr/lib/
  ├─ libfoo.so.1          (通用版本)
  └─ glibc-hwcaps/
      ├─ x86-64-v3/
      │   └─ libfoo.so.1  (AVX2 优化版本)
      └─ x86-64-v4/
          └─ libfoo.so.1  (AVX-512 优化版本)
```

### 9.2 ldconfig 处理

1. `add_glibc_hwcaps_subdirectories()` 扫描 `glibc-hwcaps/` 目录
   （`ldconfig.c:301-363`）
2. 每个子目录创建独立的 `dir_entry`，关联 `glibc_hwcaps_subdirectory` 结构
3. 扫描时，hwcaps 子目录中的库**不创建符号链接**
4. 缓存条目的 `hwcap` 字段设置为 `DL_CACHE_HWCAP_EXTENSION | section_index`

### 9.3 ld.so 运行时选择

`dl-cache.c:83-169`：

```
glibc_hwcaps_priorities_init():
  │
  ├─ 从缓存扩展区加载 glibc-hwcaps 子目录索引数组
  │   cache_extension_load()
  │   dl-cache.c:86-88
  │
  ├─ 与预计算的 _dl_hwcaps_priorities 数组合并:
  │   ├─ 双指针遍历（缓存数组 vs 运行时能力数组）
  │   ├─ 匹配 → 赋予运行时优先级值
  │   ├─ 不匹配 → 优先级 = 0（不可用）
  │   dl-cache.c:107-149
  │
  └─ glibc_hwcaps_priorities_length = length

glibc_hwcaps_priority(index):
  ├─ 首次调用触发 init
  └─ 返回 priorities[index]（0=不可用，越小越优先）
  dl-cache.c:155-169

search_cache() 中的 hwcap 选择:
  ├─ glibc_hwcaps_priority(hwcap_index) 返回优先级
  ├─ 0 → 不可用（CPU 不支持）
  ├─ 1,2,3... → 优先级（值越小越优先）
  └─ 选择优先级最高（值最小）的条目
```

---

## 10. 辅助缓存

### 10.1 作用

辅助缓存（`/var/cache/ldconfig/aux-cache`）保存每个库文件的元数据，
避免重新读取 ELF 文件：

```c
struct aux_cache_entry_id {   // cache.c:805-810
  uint64_t ino;               // inode 号
  uint64_t ctime;             // 修改时间
  uint64_t size;              // 文件大小
  uint64_t dev;               // 设备号
};
```

### 10.2 流程

```
1. ldconfig 启动 → load_aux_cache()
2. search_dir() 中:
   ├─ search_aux_cache(&stat) → 命中: 直接使用 flag/soname
   └─ 未命中: process_file() 读 ELF → add_to_aux_cache()
3. ldconfig 退出 → save_aux_cache()
```

当文件的 `ino + ctime + size + dev` 完全匹配时，辅助缓存命中，
跳过昂贵的 ELF 解析操作。这大幅加速了重复运行 ldconfig 的速度。

---

## 11. 缓存管理操作总结

### 11.1 生成

```bash
# 完整重建缓存（读配置 + 扫描 + 写缓存 + 更新符号链接）
sudo ldconfig

# 仅处理指定目录
sudo ldconfig /usr/local/lib /opt/myapp/lib

# 不更新符号链接，仅重建缓存
sudo ldconfig -N
```

### 11.2 更新

```bash
# 安装新库后更新缓存
sudo ldconfig

# 添加新目录到配置
echo "/opt/newlib" | sudo tee /etc/ld.so.conf.d/newlib.conf
sudo ldconfig
```

ldconfig **没有增量更新**机制——每次都是**全量重建**：
1. 重新解析所有配置文件
2. 重新扫描所有目录
3. 重新生成完整缓存
4. 原子替换旧缓存

辅助缓存（aux-cache）通过跳过 ELF 解析来加速此过程。

### 11.3 查看

```bash
# 打印缓存内容
ldconfig -p

# 输出格式:
# 1282 libs found in cache `/etc/ld.so.cache'
#     libz.so.1 (libc6,x86-64) => /lib/x86_64-linux-gnu/libz.so.1
#     libm.so.6 (libc6,x86-64) => /lib/x86_64-linux-gnu/libm.so.6
```

### 11.4 删除

```bash
# 删除缓存文件（ld.so 将回退到逐目录搜索）
sudo rm /etc/ld.so.cache

# 删除辅助缓存
sudo rm /var/cache/ldconfig/aux-cache

# 重建
sudo ldconfig
```

删除缓存后 ld.so 的行为：
- dlopen 搜索步骤 5（缓存查找）被跳过
- 直接回退到步骤 6（默认系统目录逐一 open）
- 性能退化但功能不受影响

### 11.5 数据流总图

```
/etc/ld.so.conf ──────┐
  include *.conf       │
  /usr/local/lib       │
  /opt/app/lib         │     ldconfig
                       │     ┌──────────────────────────────────┐
命令行目录 ────────────┤     │                                  │
                       ├────→│  1. 解析配置                     │
系统目录               │     │  2. 扫描目录 (readdir64)         │
  SLIBDIR (/lib)  ─────┤     │  3. 读取 ELF (process_elf_file)  │
  LIBDIR  (/usr/lib)───┘     │  4. 提取 soname                  │
                             │  5. 创建/更新符号链接             │
辅助缓存 ←──────────────────→│  6. 排序条目                     │
/var/cache/ldconfig/         │  7. 写入临时文件                  │
  aux-cache                  │  8. fsync + rename 原子替换       │
                             └─────────────┬────────────────────┘
                                           │
                                           ▼
                              /etc/ld.so.cache
                                           │
                                           │  ld.so (运行时)
                              ┌────────────┴────────────────────┐
                              │  1. mmap 缓存文件                │
                              │  2. 检测格式 (旧/新/混合)        │
                              │  3. 二分搜索 search_cache()      │
                              │  4. _dl_cache_libcmp 版本比较    │
                              │  5. glibc-hwcaps 优先级选择      │
                              │  6. 返回最佳匹配路径             │
                              └──────────────────────────────────┘
```

---

## 12. 源文件速查表

| 源文件 | 行号 | 内容 |
|--------|------|------|
| `elf/ldconfig.c` | 59-72 | `struct dir_entry` 目录条目结构 |
| `elf/ldconfig.c` | 122-136 | 命令行选项定义 |
| `elf/ldconfig.c` | 154-205 | `parse_opt` 参数解析 |
| `elf/ldconfig.c` | 261-297 | `add_single_dir` 目录去重（dev/ino） |
| `elf/ldconfig.c` | 301-363 | `add_glibc_hwcaps_subdirectories` |
| `elf/ldconfig.c` | 366-421 | `add_dir_1` 注册目录 |
| `elf/ldconfig.c` | 461-544 | `create_links` 创建 soname 符号链接 |
| `elf/ldconfig.c` | 663-672 | `struct dlib_entry` 库信息结构 |
| `elf/ldconfig.c` | 676-698 | `skip_dso_based_on_name` 临时文件过滤 |
| `elf/ldconfig.c` | 700-1008 | `search_dir` 目录扫描核心 |
| `elf/ldconfig.c` | 1010-1027 | `search_dirs` 遍历所有目录 |
| `elf/ldconfig.c` | 1030-1175 | `main` 函数 |
| `elf/ldconfig-parse.c` | 61-143 | `ldconfig_parse_config_1` 配置解析 |
| `elf/ldconfig-parse.c` | 147-204 | `parse_conf_include` include 处理 |
| `elf/readlib.c` | 58-170 | `process_file` 文件格式分派 |
| `elf/readelflib.c` | 41-250 | `process_elf_file` ELF 解析与 soname 提取 |
| `elf/cache.c` | 292-402 | `print_cache` 打印缓存（-p 模式） |
| `elf/cache.c` | 404-409 | `init_cache` 初始化 |
| `elf/cache.c` | 411-439 | `compare` 排序比较函数 |
| `elf/cache.c` | 454-514 | `write_extensions` 写扩展区 |
| `elf/cache.c` | 529-750 | `save_cache` 生成缓存文件 |
| `elf/cache.c` | 675-736 | 临时文件 + rename 原子替换 |
| `elf/cache.c` | 753-800 | `add_to_cache` 添加条目 |
| `elf/dl-cache.c` | 83-150 | `glibc_hwcaps_priorities_init` |
| `elf/dl-cache.c` | 155-169 | `glibc_hwcaps_priority` 优先级查询 |
| `elf/dl-cache.c` | 194-336 | `search_cache` 二分搜索 |
| `elf/dl-cache.c` | 338-374 | `_dl_cache_libcmp` 版本感知比较 |
| `elf/dl-cache.c` | 384-499 | `_dl_load_cache_lookup` 运行时查找入口 |
| `elf/dl-cache.c` | 506-518 | `_dl_unload_cache` 卸载缓存映射 |
| `sysdeps/generic/dl-cache.h` | 67-78 | `file_entry` / `cache_file` 旧格式 |
| `sysdeps/generic/dl-cache.h` | 85-100 | `file_entry_new` 新格式条目 |
| `sysdeps/generic/dl-cache.h` | 102-127 | hwcap 字段编码与解析函数 |
| `sysdeps/generic/dl-cache.h` | 157-182 | `cache_file_new` 新格式头 |
| `sysdeps/generic/dl-cache.h` | 200-256 | 缓存扩展区结构定义 |
