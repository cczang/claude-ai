---
name: activity-tracker
description: Use at the START of every session to review recent activity, identify unfinished work, and discover user preferences. Reads session history, summarizes recent interactions, detects interrupted tasks, and maintains a user profile that improves AI interactions over time.
---

# Activity Tracker

## Overview

Two responsibilities:

1. **Session continuity** — summarize recent work so the user can pick up where they left off
2. **User profile** — discover and persist user preferences so AI behaves better over time

**Core principle:** The user should never have to repeat themselves — not about what they were doing, and not about how they like to work.

## Part 1: Session Continuity

### Step 1: Read Recent Sessions

Use the built-in session tools to load recent activity:

```
session_list(limit=5)
session_read(session_id=X)
```

Only read the last 1-2 sessions in detail. For older ones, read just the summary.

### Step 2: Analyze and Summarize

From the session messages, extract:

1. **What was being worked on** — which files, modules, features were discussed
2. **What was the last action** — did it complete successfully, or was it interrupted?
3. **Unfinished items** — any task that started but didn't reach a clear conclusion

### Step 3: Check State Files

After reading sessions, also check for tool-specific state files:

- `.opencode/activity.json` — operation-level activity log (if exists)
- `.code-reader-state.json` in output directories — code-reader execution state

These provide more precise status than session messages.

### Step 4: Present Session Summary

**If there are unfinished items:**

```
上次工作回顾：
• 你在 [项目/模块] 上进行了 [操作描述]
• ⚠️ [未完成的操作] — [具体状态和建议]

是否要继续？
```

**If everything was completed:**

```
上次工作回顾：
• [简要总结上次做了什么]

今天想做什么？
```

**If no previous sessions:**
Skip — say nothing, let user start fresh.

### Rules for Session Summary

- Maximum 3-5 bullet points, not a wall of text
- Focus on ACTIONABLE items — what needs to continue
- Use plain language, not technical IDs or JSON
- Don't repeat information the user already knows
- If the last session was >7 days ago, just say "上次操作是 X 天前" without details

---

## Part 2: User Profile

### What is User Profile

User Profile is a set of **natural language instructions** that tells AI how this user prefers to interact. It lives in `~/.claude/CLAUDE.md` under a `## User Profile` section, which means it is **automatically loaded into every future session's system prompt** without any manual action.

The profile is NOT a data record. Every line must pass one test: **"If AI reads this next session, will it behave differently?"** If not, don't write it.

### What belongs in User Profile

Only **stable, cross-task preferences** that AI cannot infer from the current conversation:

| Category | Example | Why it helps |
|---|---|---|
| Communication language | "默认用中文回复" | User may send mixed-language prompts |
| Answer structure | "先给结论，再展开分析" | Changes every response's shape |
| AI initiative level | "代码变更先提议，不要直接改" | Governs agent behavior |
| Domain background | "嵌入式 C 开发者，不需要解释基础概念" | Adjusts explanation depth |
| Standing constraints | "避免引入新依赖" | Changes recommendations |

### What does NOT belong in User Profile

- **Task-specific behavior**: "这次用并行 agent" — that's about this task, not the user
- **Tool usage patterns**: "常用 code-reader" — irrelevant to how AI should behave
- **System injected content**: "用户使用 analyze-mode" — that's prompt engineering, not preference
- **One-off requests**: "这次详细一点" — temporary, not stable
- **Speculation**: "可能会迁移到 Rust" — unconfirmed

### Step 5: Discover Preferences

While reading session history in Step 1-2, look for **preference signals** — moments where the user corrected the AI's behavior or explicitly stated a preference.

Preference signals include:

- **Explicit corrections**: "用中文回复", "别这么啰嗦", "先别改代码"
- **Repeated patterns**: user consistently rephrases AI output to be shorter (implies verbosity preference)
- **Explicit statements**: "我是做嵌入式的", "我们团队用 Conventional Commits"

Do NOT count:
- Tool choices (which skill/command was used) — these are task-driven
- Session length or timing — not a preference
- Anything the system injected rather than the user typed

### Step 6: Confirm and Persist

**Never silently write preferences. Always confirm with the user.**

If preference signals were found, present them **after** the session summary:

```
我注意到你可能有以下偏好，是否要记住？
1. 默认用中文回复
2. 回答先给结论再展开

确认后我会写入你的全局配置，后续所有 session 都会自动生效。
(输入 y 全部确认，或输入编号选择性确认，或 n 跳过)
```

**On confirmation:**

1. Read `~/.claude/CLAUDE.md`
2. If a `## User Profile` section exists, merge new preferences into it (don't duplicate)
3. If it doesn't exist, append one at the end of the file
4. Write the updated file

**Format of the User Profile section:**

```markdown
## User Profile

以下是 AI 在每次对话中应遵循的用户偏好（由 activity-tracker 维护，用户确认后写入）：

- 默认使用中文回复，除非用户明确要求英文
- 回答先给结论再展开，简洁优先
- 代码变更先检查提议，不要直接修改文件
- 用户是嵌入式 C 开发者，不需要解释基础概念

<!-- activity-tracker-profile-end -->
```

The `<!-- activity-tracker-profile-end -->` marker helps locate the section boundary for future updates.

### Step 7: Handle Existing Profile

When running Step 5-6, first check if `~/.claude/CLAUDE.md` already has a `## User Profile` section.

- If it exists, read current preferences before proposing new ones
- Do NOT propose preferences that are already recorded
- Only propose genuinely new discoveries
- If no new preferences are found, skip Step 6 silently

### User Commands for Profile Management

- **"查看我的偏好"** → Read and display the User Profile section from `~/.claude/CLAUDE.md`
- **"删除偏好 [内容]"** → Remove a specific preference line
- **"清空偏好"** → Remove the entire User Profile section
- **"添加偏好 [内容]"** → Directly add a preference without discovery (user knows what they want)

---

## Part 3: Operation Log

### Recording Operations (for skills to write)

When any skill completes a significant operation, it SHOULD append to `.opencode/activity.json`:

```json
{
  "id": "short-unique-id",
  "timestamp": "ISO-8601",
  "tool": "code-reader",
  "action": "generate",
  "target": "baize_eufy/alarm",
  "status": "completed | interrupted | failed",
  "summary": "One line describing what happened",
  "resume_hint": "What to do next (only if not completed)"
}
```

### File Schema

```json
{
  "version": 1,
  "max_entries": 50,
  "entries": []
}
```

### Write Pattern

```
1. Start of operation → write entry with status: "interrupted" (assume worst case)
2. End of operation (success) → update to status: "completed"
3. End of operation (failure) → update to status: "failed"
4. Session dies mid-operation → entry stays "interrupted" automatically
```

### Limits

- Max 50 entries. When adding #51, remove the oldest completed entry.
- Never auto-remove interrupted or failed entries.
- User can clear with: "清除操作记录"

---

## Anti-Patterns

**❌ Reading all session history at startup**
→ Only read the last 1-2 sessions. Older sessions are irrelevant.

**❌ Showing a wall of text on startup**
→ 3-5 bullet points max. Focus on what's actionable.

**❌ Asking the user to confirm every detail**
→ Present summary, offer to continue. Don't interrogate.

**❌ Silently writing user preferences**
→ Always confirm with the user before persisting any preference.

**❌ Recording tool usage as "user habits"**
→ Which tools were used is task-driven, not a user preference. Only record things that change how AI should respond.

**❌ Storing preferences in JSON data files**
→ The consumer is AI, not a program. Write natural language instructions into a file that gets loaded into the system prompt.

**❌ Each skill maintaining its own separate state file**
→ Use activity.json as the shared operation log. Skill-specific state (like .code-reader-state.json) is for internal detail only.
