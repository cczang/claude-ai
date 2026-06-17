---
name: coredump-analyzer
description: Use when analyzing coredump/core files from embedded ARM aarch64 devices, when user provides core files with names like core_PID_ThreadName_Timestamp, program images like app.img (squashfs), or ELF binaries. Triggers include "分析coredump", "分析core文件", "崩溃分析", "crash analysis", "core dump", "SIGSEGV", "SIGABRT", "段错误".
---

# Coredump Analyzer

## Overview

分析嵌入式 ARM aarch64 设备的 coredump 文件，从 core 文件 + 程序镜像中还原崩溃现场，输出结构化分析报告。

**核心原则：** 先对齐版本（core、ELF、so 三者一致），再还原现场（GDB 加载），最后归因（事实→证据→判断）。版本不一致的分析结论不可信。

## When to Use

- 用户提供了 core 文件（如 `core_2201_DiskMgrThread_1780631922`）
- 用户提供了程序镜像 `app.img`（squashfs）或直接的 ELF 二进制
- 用户提到 "coredump"、"core dump"、"崩溃分析"、"crash"、"SIGSEGV"、"SIGABRT"
- 用户提供了设备日志压缩包中包含 core 文件

**不适用：**
- 纯日志分析（无 core 文件）→ 使用 eufy-log-analyzer
- 源码级 bug 定位（已知崩溃点，需要代码审查）→ 使用 code-tracer

## Quick Reference

### 文件命名规则

| 文件 | 格式 | 示例 |
|------|------|------|
| Core 文件 | `core_{PID}_{ThreadName}_{Timestamp}` | `core_2201_DiskMgrThread_1780631922` |
| 程序镜像 | `app.img` (squashfs) | 包含 `/bin/baize_eufy` 等 |
| 构建信息 | `build.info` | 版本、CI 编号、构建类型 |
| 版本文件 | `main_version` | 如 `4.2.2.2` |

### Core 文件名解析

```
core_2201_DiskMgrThread_1780631922
      │         │            │
      PID    线程名      Unix时间戳
```

**注意：** 线程名可能包含下划线（如 `db_helper_main`、`diskmgr_eufy`）。解析规则：第一个 `_` 后的数字是 PID，最后一个 `_` 后的纯数字是时间戳，中间部分是线程名。

```bash
# 解析 core 文件名
CORE_BASENAME=$(basename "$CORE_FILE")
PID=$(echo "$CORE_BASENAME" | sed 's/^core_\([0-9]*\)_.*/\1/')
THREAD=$(echo "$CORE_BASENAME" | sed 's/^core_[0-9]*_\(.*\)_[0-9]*$/\1/')
TIMESTAMP=$(echo "$CORE_BASENAME" | sed 's/^.*_\([0-9]*\)$/\1/')
echo "PID=$PID Thread=$THREAD Time=$(date -d @$TIMESTAMP)"
```

### 关键路径

| 用途 | 路径 |
|------|------|
| 交叉 GDB | `/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-gdb` |
| addr2line | `/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-addr2line` |
| readelf | `/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-readelf` |
| objdump | `/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-objdump` |
| Toolchain sysroot | `/home/zang/compile/release/cross-toolchain/aarch64-buildroot-linux-gnu/sysroot` |
| 设备 rootfs 中主程序 | `{rootfs}/bin/baize_eufy` |
| 设备 rootfs 中 so 库 | `{rootfs}/lib/` |

## 分析流程

### Step 0: 准备工作目录

```bash
WORK_DIR="/tmp/opencode/coredump_analysis"
mkdir -p "$WORK_DIR"
```

### Step 1: 识别输入文件

确认用户提供了什么：

1. **Core 文件** — 必须。用 `file` 确认是 `ELF 64-bit LSB core file, ARM aarch64`
2. **程序文件** — 必须。两种形式：
   - `app.img`（squashfs）→ 需要解包
   - 直接 ELF 二进制 → 直接使用
3. **build.info** — 可选但强烈建议。用于版本校验
4. **main_version** — 可选。固件版本号

```bash
file "$CORE_FILE"
file "$APP_IMG"
```

### Step 2: 解包程序镜像（如果是 squashfs）

```bash
ROOTFS="$WORK_DIR/rootfs"
unsquashfs -d "$ROOTFS" "$APP_IMG"
```

解包后确认主程序位置：

```bash
# 从 core 文件中提取进程名
PROC_NAME=$(file "$CORE_FILE" | grep -oP "from '\K[^']+")
# 在 rootfs 中查找
find "$ROOTFS" -name "$PROC_NAME" -type f
```

**典型路径：** `{rootfs}/bin/baize_eufy`

### Step 3: 版本一致性校验（关键！）

**这一步不能跳过。版本不一致的分析结论不可信。**

```bash
# 1. 读 build.info
cat "$BUILD_INFO"

# 2. 检查 ELF 的 Build ID
readelf -n "$ROOTFS/bin/$PROC_NAME" 2>/dev/null | grep "Build ID"

# 3. 检查 core 中记录的进程信息
readelf -n "$CORE_FILE" 2>/dev/null | grep -A5 "CORE"

# 4. 检查 rootfs 中的 .build.info（如果有）
cat "$ROOTFS/.build.info" 2>/dev/null
```

如果 Build ID 不匹配或构建时间差异大，**必须在报告中标注置信度降级**。

### Step 4: GDB 批量分析

创建 GDB 命令脚本：

```bash
cat > "$WORK_DIR/gdb_cmds.txt" << 'GDBEOF'
set pagination off
set confirm off
set print pretty on
set print elements 0
set print frame-arguments all

# === 设置符号搜索路径 ===
# ROOTFS_PATH 和 SYSROOT_PATH 需要替换为实际路径
set sysroot ROOTFS_PATH
set solib-absolute-prefix ROOTFS_PATH
set solib-search-path ROOTFS_PATH/lib:ROOTFS_PATH/usr/lib:SYSROOT_PATH/lib:SYSROOT_PATH/usr/lib

# === 基本信息 ===
echo \n=== CRASH SIGNAL ===\n
info program

echo \n=== SHARED LIBRARIES ===\n
info sharedlibrary

echo \n=== ALL THREADS ===\n
info threads

# === 崩溃线程详情 ===
echo \n=== CRASH THREAD BACKTRACE ===\n
bt
bt full

echo \n=== CRASH FRAME DETAILS ===\n
frame 0
info args
info locals
info registers

echo \n=== CRASH POINT DISASSEMBLY ===\n
x/20i $pc-40
x/8i $pc

echo \n=== STACK MEMORY ===\n
x/32gx $sp

echo \n=== KEY REGISTERS (AArch64) ===\n
printf "PC  = 0x%lx\n", $pc
printf "LR  = 0x%lx\n", $x30
printf "SP  = 0x%lx\n", $sp
printf "FP  = 0x%lx\n", $x29
printf "x0  = 0x%lx\n", $x0
printf "x1  = 0x%lx\n", $x1
printf "x2  = 0x%lx\n", $x2
printf "x3  = 0x%lx\n", $x3

# === 所有线程调用栈 ===
echo \n=== ALL THREADS BACKTRACE ===\n
thread apply all bt

echo \n=== ALL THREADS BACKTRACE FULL ===\n
thread apply all bt full

quit
GDBEOF
```

替换路径并执行：

```bash
sed -i "s|ROOTFS_PATH|$ROOTFS|g" "$WORK_DIR/gdb_cmds.txt"
sed -i "s|SYSROOT_PATH|/home/zang/compile/release/cross-toolchain/aarch64-buildroot-linux-gnu/sysroot|g" "$WORK_DIR/gdb_cmds.txt"

GDB="/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-gdb"
"$GDB" -batch -x "$WORK_DIR/gdb_cmds.txt" \
  "$ROOTFS/bin/$PROC_NAME" \
  "$CORE_FILE" \
  > "$WORK_DIR/gdb_output.txt" 2>&1
```

### Step 5: 解析 GDB 输出

从 `gdb_output.txt` 中提取关键信息，**原始 GDB 输出必须完整保留到报告中**：

1. **崩溃信号** — `SIGSEGV` / `SIGABRT` / `SIGBUS` / `SIGILL`
2. **崩溃线程** — 线程 ID 和名称
3. **崩溃调用栈** — `bt` 和 `bt full` 的**原始输出**（包含地址、库路径、参数），不得简化
4. **崩溃点** — `info registers` 和反汇编的**原始输出**
5. **寄存器状态** — 特别是 `x0-x3`（函数参数）、`PC`、`LR`
6. **全线程状态** — 是否有其他线程异常

**关键原则：报告中的崩溃堆栈部分必须是 GDB 的原始输出，不做格式美化。原始堆栈是分析的第一手证据，是判断根因的最主要依据。**

### Step 6: 地址解析（如果 backtrace 中有 `??`）

对于没有符号的地址，使用 addr2line：

```bash
ADDR2LINE="/home/zang/compile/release/cross-toolchain/bin/aarch64-buildroot-linux-gnu-addr2line"

# 对于主程序中的地址（注意 PIE 需要减去基址）
"$ADDR2LINE" -f -C -e "$ROOTFS/bin/$PROC_NAME" 0xADDRESS

# 对于 so 库中的地址（需要用偏移地址，不是虚拟地址）
"$ADDR2LINE" -f -C -e "$ROOTFS/lib/libxxx.so" 0xOFFSET
```

**PIE 地址换算：**
```
ELF偏移 = 运行时虚拟地址 - 模块装载基址
```
装载基址从 GDB 的 `info sharedlibrary` 或 `info proc mappings` 获取。

## 报告模板

**报告文件名：** 以 core 文件的 PID 和线程名作为前缀，保存在 core 文件同目录下：

```
core_{PID}_{ThreadName}_analysis.md
```

示例：分析 `core_2201_DiskMgrThread_1780631922` → 报告文件名为 `core_2201_DiskMgrThread_analysis.md`

分析完成后，输出以下格式的报告：

```markdown
# Core Dump 崩溃分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| Core 文件 | `{filename}` |
| 进程名 | `{proc_name}` |
| 崩溃 PID | {pid} |
| 崩溃线程 | {thread_info} |
| 崩溃信号 | {signal} |
| 架构 | ARM aarch64 |
| 版本 | {version} |
| 构建信息 | {build_info_summary} |
| 崩溃时间 | {timestamp_parsed} |
| 版本一致性 | {matched/mismatched/unknown} |

## 崩溃堆栈（GDB 原始输出）

**必须保留 GDB `bt` 命令的原始输出，包括地址、库路径、参数信息。这是最主要的判断依据，不得简化或重新格式化。**

### 崩溃线程 bt

{直接粘贴 GDB `bt` 的原始输出，不做任何修改}

### 崩溃线程寄存器

{直接粘贴 `info registers` 的原始输出}

## 根因分析

### 崩溃机制
{signal 含义 + 崩溃点描述}

### 根因判断
- **高置信**: {primary_cause}
- **中置信备选**: {alternative_cause}（如果有）

### 证据链
1. {evidence_1}
2. {evidence_2}
...

## 其他线程状态概览

| 线程类型 | 数量 | 状态 |
|----------|------|------|
| ... | ... | ... |

## 修复建议

### 立即修复（P0）
{fix}

### 改进建议（P1）
{improvements}
```

## Common Mistakes

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 直接用 `app.img` 喂 GDB | GDB 报错或分析完全错误 | 先 `unsquashfs` 解包，找到真正的 ELF |
| 不设 `sysroot`/`solib-search-path` | backtrace 全是 `??` | 同时设置 rootfs 和 toolchain sysroot |
| 不校验版本一致性 | 分析结论不可信 | 比对 build.info、Build ID、构建时间 |
| PIE 地址直接送 addr2line | 行号完全错误 | 先减去装载基址得到 ELF 偏移 |
| 只看崩溃线程 | 遗漏真正的根因线程 | `thread apply all bt` 看全部线程 |
| 把 SIGABRT 当 SIGSEGV 分析 | 方向错误 | SIGABRT 通常是 assert/abort，查断言条件 |
| 忽略 stripped 二进制的影响 | 过度解读不完整的符号 | 标注符号完整度，降级结论置信度 |
| 美化/简化 GDB 原始堆栈输出 | 丢失地址、库路径、参数等关键判断信息 | 报告中必须保留 `bt`/`bt full` 的原始输出 |

## 批量分析

如果目录中有多个 core 文件：

```bash
for core in /path/to/core_*; do
  PID=$(echo "$core" | sed 's/.*core_\([0-9]*\)_.*/\1/')
  THREAD=$(echo "$core" | sed 's/.*core_[0-9]*_\(.*\)_[0-9]*/\1/')
  TIMESTAMP=$(echo "$core" | sed 's/.*_\([0-9]*\)$/\1/')
  echo "=== Core: PID=$PID Thread=$THREAD Time=$(date -d @$TIMESTAMP) ==="
  # 对每个 core 执行 Step 4-6
done
```

## 信号速查

| 信号 | 含义 | 常见原因 |
|------|------|----------|
| SIGSEGV (11) | 段错误 | 空指针、野指针、越界访问 |
| SIGABRT (6) | 主动终止 | assert 失败、double-free、堆损坏 |
| SIGBUS (7) | 总线错误 | 未对齐访问、映射失效 |
| SIGILL (4) | 非法指令 | ABI 不匹配、栈破坏跳转到数据区 |
| SIGFPE (8) | 浮点异常 | 除零、整数溢出 |

## AArch64 寄存器速查

| 寄存器 | 用途 |
|--------|------|
| x0-x7 | 函数参数 / 返回值 (x0) |
| x8 | 间接结果寄存器 |
| x9-x15 | 临时寄存器（caller-saved） |
| x16-x17 | IP0/IP1（PLT/veneer） |
| x18 | 平台寄存器（TLS） |
| x19-x28 | callee-saved |
| x29 (FP) | 帧指针 |
| x30 (LR) | 链接寄存器（返回地址） |
| SP | 栈指针 |
| PC | 程序计数器 |
