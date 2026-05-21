---
name: code-tracer
description: Use when wanting to trace a function call chain, message processing flow, or feature implementation path through source code. Use when someone asks "how does X work", "trace the flow of Y", "where is Z handled", or needs to understand the complete processing path of a function, command, or feature.
---

# Code Tracer

## Overview

Given a function name, message/command ID, or feature description, produce a layered call chain showing the complete processing flow with exact source locations.

**Core principle:** Start from code-reader's SKILL.md for navigation, then read source code only for the specific path — not the entire module.

## Input Types

| Input | Example | First action |
|-------|---------|-------------|
| **Function name** | `zx_hub_play_tone` | Search SKILL.md for file:line, then read that function |
| **Message/Command ID** | `APP_CMD_START_REALTIME_MEDIA` | Search SKILL.md for handler, then trace dispatch chain |
| **Feature description** | "预览出流" | Search SKILL.md for related entry points, pick the most likely one |

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

## Output Format

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
