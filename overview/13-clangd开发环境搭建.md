# glibc AArch64 交叉编译与 clangd 开发环境搭建手册

本文档记录如何为 glibc 2.43.9000 源码搭建 AArch64 交叉编译环境，并生成
`compile_commands.json` 供 clangd 使用，实现完整的代码导航、补全与诊断功能。

---

## 一、前置依赖

| 组件 | 最低版本 | 安装命令 |
|------|----------|----------|
| GCC 交叉编译器 | **12.1+** | `sudo apt install gcc-12-aarch64-linux-gnu g++-12-aarch64-linux-gnu` |
| Linux 头文件 | — | `sudo apt install linux-libc-dev-arm64-cross` |
| AArch64 系统头文件 | — | 通常随交叉编译器安装 (`/usr/aarch64-linux-gnu/include`) |
| bear | 3.0+ | `sudo apt install bear` |
| clangd | 14+ (推荐 18+) | 见下方安装说明 |
| 构建工具 | — | `sudo apt install make gawk bison texinfo python3` |

### 安装 clangd（推荐最新版）

```bash
# 方法1: 使用官方预编译包
mkdir -p ~/.local/bin
curl -L https://github.com/clangd/clangd/releases/latest/download/clangd-linux-x86_64.zip -o /tmp/clangd.zip
unzip /tmp/clangd.zip -d /tmp/clangd
cp /tmp/clangd/clangd_*/bin/clangd ~/.local/bin/
chmod +x ~/.local/bin/clangd

# 方法2: apt 安装
sudo apt install clangd-18  # Ubuntu 24.04+
```

### 验证版本

```bash
aarch64-linux-gnu-gcc-12 --version
# aarch64-linux-gnu-gcc-12 (Ubuntu 12.3.0-...) 12.3.0

clangd --version
# clangd version 22.1.1 (或 ≥14)
```

> **注意**: glibc 2.43 要求 GCC ≥ 12.1 或 Clang ≥ 18，以及 GNU ld ≥ 2.39。
> 如果系统 binutils 版本不足（如 2.38），可用 ld 版本包装脚本绕过（见下文）。

---

## 二、配置与编译

### 2.1 创建构建目录

glibc **必须**在源码树外的独立目录中构建：

```bash
cd /path/to/glibc-source
mkdir -p build-aarch64
cd build-aarch64
```

### 2.2 处理 binutils 版本不足的问题

如果系统 `aarch64-linux-gnu-ld` 版本 < 2.39，创建包装脚本：

```bash
cat > ldwrap.sh << 'EOF'
#!/bin/bash
if [ "$1" = "--version" ]; then
  echo "GNU ld (GNU Binutils) 2.41"
  exit 0
fi
exec /usr/bin/aarch64-linux-gnu-ld "$@"
EOF
chmod +x ldwrap.sh
```

> **说明**: 此技巧仅绕过版本检测。实际链接可能因缺少新特性而出现个别 warning，
> 但对生成 `compile_commands.json` 不影响（编译阶段不需要链接器）。

### 2.3 运行 configure

```bash
../configure --host=aarch64-linux-gnu --prefix=/usr \
  --disable-werror \
  --enable-shared \
  --with-headers=/usr/aarch64-linux-gnu/include \
  CC=aarch64-linux-gnu-gcc-12 \
  CXX=aarch64-linux-gnu-g++-12 \
  LD=$(pwd)/ldwrap.sh    # 仅在 ld 版本不足时需要
```

成功输出：
```
config.status: creating config.make
config.status: creating Makefile
config.status: creating config.h
config.status: executing default commands
```

### 2.4 使用 bear 生成 compile_commands.json

```bash
bear -- make -j$(nproc)
```

编译完成后会在 `build-aarch64/` 下生成 `compile_commands.json`。

**典型结果**:
- 编译时间: 5-10 分钟（8核）
- 编译条目数: ~6,456 个 C 源文件
- JSON 文件大小: ~14MB

### 2.5 创建符号链接

将编译数据库链接到源码根目录（某些工具会在此查找）：

```bash
cd /path/to/glibc-source
ln -sf build-aarch64/compile_commands.json compile_commands.json
```

---

## 三、clangd 配置

### 3.1 `.clangd` 文件（源码根目录）

```yaml
# clangd configuration for glibc (AArch64 cross-compilation)
# See: https://clangd.llvm.org/config

CompileFlags:
  CompilationDatabase: build-aarch64
  Remove:
    - -fno-allow-store-data-races
    - -fmerge-all-constants
    - -frounding-math
    - -ftrapping-math
    - -fstack-protector*
    - -fexceptions
    - -moutline-atomics
  Add:
    - --target=aarch64-linux-gnu
    - -Wno-gnu-include-next

Diagnostics:
  Suppress:
    - builtin_definition_p_unevaluated
    - pp_including_mainfile_in_preamble
  ClangTidy:
    Remove:
      - modernize-*
      - readability-*
      - bugprone-reserved-identifier
      - cert-dcl37-c
      - cert-dcl51-cpp

InlayHints:
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes

Hover:
  ShowAKA: Yes
```

**配置说明**:

| 配置项 | 作用 |
|--------|------|
| `CompilationDatabase` | 告诉 clangd 在哪里找 `compile_commands.json` |
| `Remove` | 移除 clangd 不认识的 GCC 专有编译选项 |
| `Add: --target=aarch64-linux-gnu` | 确保 clangd 使用 AArch64 目标的内建类型和宏 |
| `Suppress` | 抑制 glibc 内部宏产生的误报诊断 |
| `ClangTidy Remove` | 排除对 C 系统库不适用的 lint 规则 |

### 3.2 为什么需要 Remove 这些 flags？

| Flag | 问题 |
|------|------|
| `-fno-allow-store-data-races` | GCC 专有，clang 不支持 |
| `-fmerge-all-constants` | clang 不支持此特定变体 |
| `-frounding-math` / `-ftrapping-math` | clang 行为不同，会产生大量 warning |
| `-moutline-atomics` | AArch64 GCC 专有选项 |

---

## 四、VSCode 配置

### 4.1 `.vscode/settings.json`

```json
{
    "git.ignoreLimitWarning": true,

    // 禁用微软 C/C++ IntelliSense（使用 clangd 代替）
    "C_Cpp.intelliSenseEngine": "disabled",

    // clangd 配置
    "clangd.path": "/home/nio/.local/bin/clangd",
    "clangd.arguments": [
        "--background-index",
        "--clang-tidy",
        "--completion-style=detailed",
        "--header-insertion=iwyu",
        "--pch-storage=memory",
        "--all-scopes-completion",
        "--function-arg-placeholders",
        "-j=8",
        "--compile-commands-dir=${workspaceFolder}/build-aarch64"
    ],

    // 文件类型关联
    "files.associations": {
        "*.h": "c",
        "*.S": "asm",
        "*.sym": "c"
    },

    // 隐藏构建产物
    "files.exclude": {
        "build-aarch64/**/*.o": true,
        "build-aarch64/**/*.os": true,
        "build-aarch64/**/*.so": true,
        ".ai-search": true
    },

    // 搜索排除
    "search.exclude": {
        "build-aarch64": true,
        ".ai-search": true
    }
}
```

### 4.2 关键参数说明

| 参数 | 说明 |
|------|------|
| `--background-index` | 后台构建索引，首次打开需 3-5 分钟 |
| `--clang-tidy` | 启用静态检查 |
| `--completion-style=detailed` | 补全时显示详细类型信息 |
| `--header-insertion=iwyu` | 自动建议缺少的 #include |
| `--pch-storage=memory` | PCH 存内存，加速解析 |
| `-j=8` | 8 线程并行索引 |

### 4.3 推荐安装的 VSCode 扩展

| 扩展 | ID | 说明 |
|------|----|------|
| clangd | `llvm-vs-code-extensions.vscode-clangd` | 必装 |
| AArch64 ASM | `dan-c-underwood.arm` | ARM 汇编语法高亮 |
| Hex Editor | `ms-vscode.hexeditor` | 查看 ELF 二进制 |

> **重要**: 如果同时安装了微软 C/C++ 扩展，必须在 settings 中禁用其 IntelliSense
> (`"C_Cpp.intelliSenseEngine": "disabled"`)，否则两个 LSP 会冲突。

---

## 五、使用验证

### 5.1 首次索引

打开 VSCode 后，clangd 会开始后台索引。查看进度：
- 状态栏会显示 `clangd: indexing...`
- 索引文件存储在 `.cache/clangd/index/` 下

### 5.2 功能验证

| 功能 | 测试方法 |
|------|----------|
| 跳转定义 | 在 `malloc/malloc.c` 中 Ctrl+Click `_int_malloc` |
| 查找引用 | 右键 `__pthread_create_2_1` → Find All References |
| 代码补全 | 输入 `atomic_` 查看候选列表 |
| Hover 信息 | 悬停 `INTERNAL_SIZE_T` 查看展开后的类型 |
| 调用层次 | 右键函数 → Show Call Hierarchy |
| 符号搜索 | `Ctrl+T` 输入 `allocate_stack` |

### 5.3 已知限制

1. **汇编文件** (`.S`): clangd 对汇编支持有限，不能跳转到汇编中的符号
2. **条件编译**: 只能分析 AArch64 路径的代码，x86 特定分支会灰显
3. **内部头文件**: glibc 使用 `#include_next` 链式包含，偶尔会有误报
4. **首次索引慢**: 6000+ 文件首次索引需要 3-5 分钟

---

## 六、故障排除

### 问题: clangd 报 "compilation database not found"

```bash
# 确认文件存在
ls build-aarch64/compile_commands.json
# 确认 .clangd 中 CompilationDatabase 路径正确
cat .clangd | grep CompilationDatabase
```

### 问题: 大量 "unknown argument" 错误

在 `.clangd` 的 `CompileFlags.Remove` 中添加报错的 flag。

### 问题: 找不到头文件

```bash
# 检查交叉编译器的系统头文件
ls /usr/aarch64-linux-gnu/include/linux/
# 确认 compile_commands.json 中包含正确的 -isystem 路径
python3 -c "
import json
with open('build-aarch64/compile_commands.json') as f:
    data = json.load(f)
args = data[0].get('arguments', [])
for i,a in enumerate(args):
    if a == '-isystem':
        print(args[i+1])
"
```

### 问题: 索引占用过多内存

减少 `-j` 参数值（如 `-j=4`），或关闭 `--pch-storage=memory`。

---

## 七、目录结构总览

```
glibc/
├── .clangd                          ← clangd 配置
├── .vscode/
│   └── settings.json                ← VSCode 配置
├── compile_commands.json            ← 符号链接 → build-aarch64/...
├── build-aarch64/
│   ├── compile_commands.json        ← bear 生成（14MB, 6456 条目）
│   ├── config.make
│   ├── config.h
│   ├── ldwrap.sh                    ← ld 版本包装（如需要）
│   └── ...（编译产物）
├── .ai-search/                      ← 源码索引（zoekt/ctags/cscope）
├── darren/                          ← 分析文档
└── (glibc 源码...)
```

---

## 八、快速复制命令

一键完整搭建（假设依赖已安装）：

```bash
cd /path/to/glibc-source

# 1. 创建构建目录
mkdir -p build-aarch64 && cd build-aarch64

# 2. ld 包装（如果 binutils < 2.39）
cat > ldwrap.sh << 'EOF'
#!/bin/bash
[ "$1" = "--version" ] && echo "GNU ld (GNU Binutils) 2.41" && exit 0
exec /usr/bin/aarch64-linux-gnu-ld "$@"
EOF
chmod +x ldwrap.sh

# 3. Configure
../configure --host=aarch64-linux-gnu --prefix=/usr \
  --disable-werror --enable-shared \
  --with-headers=/usr/aarch64-linux-gnu/include \
  CC=aarch64-linux-gnu-gcc-12 \
  CXX=aarch64-linux-gnu-g++-12 \
  LD=$(pwd)/ldwrap.sh

# 4. 编译并生成 compile_commands.json
bear -- make -j$(nproc)

# 5. 创建符号链接
cd .. && ln -sf build-aarch64/compile_commands.json .

# 6. 完成！用 VSCode 打开即可
code .
```
