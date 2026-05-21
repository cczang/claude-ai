---
name: code-tracer
description: Use when wanting to trace a function call chain, message processing flow, or feature implementation path through source code. Use when someone asks "how does X work", "trace the flow of Y", "where is Z handled", or needs to understand the complete processing path of a function, command, or feature. Also use when needing to find all log points, diagnostic traces, or debug outputs related to a specific feature — e.g. "what logs can help investigate X", "哪些log能协助调查Y", "how to debug Z".
---

# Code Tracer

## Overview

Given a function name, message/command ID, or feature description, produce a layered call chain showing the complete processing flow with exact source locations. Also supports extracting all diagnostic log points across the call chain for a given feature.

**Core principle:** Start from code-reader's SKILL.md for navigation, then read source code only for the specific path — not the entire module.

## Input Types

| Input | Example | First action |
|-------|---------|-------------|
| **Function name** | `zx_hub_play_tone` | Search SKILL.md for file:line, then read that function |
| **Message/Command ID** | `APP_CMD_START_REALTIME_MEDIA` | Search SKILL.md for handler, then trace dispatch chain |
| **Feature description** | "预览出流" | Search SKILL.md for related entry points, pick the most likely one |
| **Diagnostic query** | "VoIP有哪些log能调查" | First trace the feature flow, then extract all LOG/print statements along the path |

## The Two-Step Process

### Step 1: Locate via SKILL.md (fast, no source reading)

Before reading ANY source code, find the code-reader output:

1. **Auto-discover SKILL.md location:**
   - Look for `.code-reader-state.json` in the current working directory and parent directories
   - If found: read `output_dir` field → SKILL.md files are at `{output_dir}/{project}-fj-{module}/SKILL.md`
   - If not found: search common locations:
     - `./docs/*/SKILL.md`
     - `./.opencode/skills/*/SKILL.md`
     - `~/.claude/skills/*/SKILL.md`
   - If still not found: fall back to grep (Step 1c)

2. **Search SKILL.md for the target:**
   - Function name → grep for the function name in all discovered SKILL.md files
   - Command/Message ID → grep for the command constant
   - Feature description → grep for related keywords
   - Extract file:line from the match

3. **Fallback — no SKILL.md available:**
   - Use grep on source code directly: `grep -rn "function_name" {source-dir} --include="*.c" --include="*.h"`

**Gate:** You must have a starting file:line before proceeding to Step 2. Do NOT grep the entire codebase if SKILL.md can give you the answer.

### Step 2: Trace the call chain (targeted source reading)

Starting from the file:line found in Step 1:

1. Read ONLY that function (not the entire file)
2. For each function it calls, decide: is this part of the main flow or a side effect?
3. Follow the main flow deeper (read the next function)
4. Stop when you reach: a system call, an external API, or a hardware interface

**Do NOT read files that aren't on the call path.**

## When Input is a Diagnostic Query

Diagnostic queries (如"VoIP有哪些log能协助调查", "how to debug OTA failures") need a combined approach:

1. **First, trace the feature flow** using the standard Two-Step Process above
2. **Then, for each function on the call chain**, extract all LOG/print/dzlog statements
3. **Organize logs by phase** (not by file), following the call chain order
4. **Output format**: use the Diagnostic Log Map format (see below)

This is NOT a simple grep — you must first understand the call chain, then collect logs along that chain. Random grep for keywords will miss context and include irrelevant noise.

## Output Format

### Standard Call Chain

Present as a layered call chain:

```
## [Input Name] 处理流程

### 入口层
  [caller] → [function] (file:line)
  作用：[一句话]

### 业务层
  → [function] (file:line)
    作用：[一句话]
    关键判断：[if X then Y, else Z]
  → [function] (file:line)
    作用：[一句话]

### 底层
  → [function] (file:line)
    作用：[一句话]
    最终效果：[硬件操作/网络发送/数据库写入]
```

Rules:
- Every function must have file:line
- Every function must have一句话说明作用
- 关键分支判断只列影响主流程的（不列每个 error check）
- 如果有多条路径（如 P2P/WebRTC/WebSocket），先说明有几条路径，再分别展开主路径
- 侧链（日志、统计、通知）放在末尾"副作用"节，不混入主流程

### Diagnostic Log Map

When the input is a diagnostic query, append a log map after the call chain:

```
## [Feature] 诊断日志一览

### 阶段 1: [阶段名称]
| 日志关键字 | 文件:行 | 级别 | 含义 |
|-----------|---------|------|------|
| `关键字内容` | `file.c:123` | INFO/WARN/ERROR | 一句话说明什么情况会打印 |

### 阶段 2: [阶段名称]
...

### 排查清单
1. 先看 X 是否出现 → 确认 Y
2. 再看 Z 的值 → 判断 W
...

### 一键 grep 命令
grep -E "keyword1|keyword2|keyword3" /path/to/log
```

Rules for diagnostic output:
- 按调用链的阶段组织，不按文件组织
- 每个日志必须说明"什么条件下会打印"
- 提供排查清单：按时序列出检查步骤和判断逻辑
- 提供一键 grep 命令覆盖全链路关键日志

## When Input is a Feature Description

Feature description (如"预览出流") needs an extra step before Step 1:

1. Ask the user: "这个功能对应哪个模块？" — or if you know,直接说
2. Load that module's SKILL.md
3. In SKILL.md find the entry point function for this feature
4. Then proceed to Step 1 with that function name

## Anti-Patterns

**❌ Reading the entire module before tracing**
→ Fix: Only read functions on the call path

**❌ Listing every error check and side effect inline**
→ Fix: Main flow only. Side effects go to a separate section at the end.

**❌ Skipping SKILL.md and going straight to grep**
→ Fix: SKILL.md gives you the answer in seconds. grep is the fallback.

**❌ Output too long — every branch fully expanded**
→ Fix: Expand only the primary path. For secondary paths, list entry point + one-line description.

## Relationship to Other Skills

- **code-reader**: Generates the SKILL.md navigation maps this skill depends on
- **symptom-to-hypothesis**: Starts from a problem → generates hypotheses → this skill traces the hypothesis path
- **systematic-debugging**: After tracing, use systematic-debugging to investigate the root cause
