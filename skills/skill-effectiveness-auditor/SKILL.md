---
name: skill-effectiveness-auditor
description: Use when evaluating whether a skill's actual execution matches its design intent, when a skill seems to produce poor results despite being triggered, when wanting to systematically improve skill quality based on real usage data, or when needing to audit skill effectiveness across sessions. Triggers include "评估skill效果", "skill执行质量", "这个skill好像不太对", "skill效果审计", "audit skill effectiveness", "skill工作不符合预期", "检查skill是否按设计工作".
---

# Skill Effectiveness Auditor

## Overview

从 session 历史中提取 skill 的实际输入/输出，与 SKILL.md 的设计对比，评估执行是否符合设计意图，生成修复方案，灰度验证后可回滚。

**核心原则：Skill 被触发不等于 Skill 被正确执行。** `skill-trigger-auditor` 解决"该用没用"的问题，本 skill 解决"用了但没用好"的问题。

**与现有 skills 的关系：**

| Skill | 解决的问题 | 本 skill 的差异 |
|-------|-----------|----------------|
| `skill-trigger-auditor` | Skill 该触发没触发 | 本 skill 审计"触发了但执行不对" |
| `skill-creator` | 创建和迭代 skill（前瞻性） | 本 skill 基于历史数据回溯性评估 |
| `eval-harness` | 通用 eval 框架 | 本 skill 专注 skill 执行效果 |

## 首次使用 — 配置审计工作目录

检查 `.opencode/skills/skill-effectiveness-auditor/config.json` 是否存在。

**不存在时，询问用户：**
```
首次使用 skill-effectiveness-auditor。我需要一个目录来保存审计报告和 skill 备份。
请指定路径（例如 /home/zang/src/AnkOutput/skill-audits/）：
```

写入配置：
```json
{
  "audit_dir": "<用户指定的路径>"
}
```

## 审计流程

```
Phase 1: 历史数据提取 → 收集 skill 的实际输入/输出
Phase 2: 合规性评估   → 对比设计 vs 实际，量化偏差
Phase 3: 修复方案     → 针对偏差生成具体修改
Phase 4: 灰度验证     → 备份 → 应用修复 → 测试验证
Phase 5: 回滚/确认    → 验证通过则保留，失败则回滚
```

---

### Phase 1: 历史数据提取

**目标：** 从 session 历史中提取目标 skill 被调用的每个实例，记录其输入上下文和输出结果。

#### 1.1 确定审计目标

向用户确认：
- **目标 skill**：要审计哪个 skill？（如 `bug-investigator`、`symptom-to-hypothesis`）
- **审计范围**：最近 N 个 session？指定日期范围？全部？

#### 1.2 收集 Skill 设计规格

读取目标 skill 的 SKILL.md，提取：
- **设计意图**：Overview 中的核心原则
- **预期流程**：各 Step/Phase 的顺序和内容
- **预期输出**：应产出的交付物（报告、文件、索引条目等）
- **必须执行的步骤**：标记为"必须"的强制步骤
- **禁止的行为**：Common Mistakes 或明确禁止的操作

将以上提取为 **设计检查清单**，格式：

```markdown
## 设计检查清单：<skill-name>

### 必须执行的步骤
- [ ] Step X: [描述]
- [ ] Step Y: [描述]

### 预期输出
- [ ] 产出物 A: [描述]
- [ ] 产出物 B: [描述]

### 禁止行为
- [ ] 不得 [行为描述]
```

#### 1.3 从 Session 历史提取使用实例

```
1. session_list(limit=20) 获取最近 session 列表
2. session_search(query="<skill-name>") 定位包含目标 skill 名称的 session
   ⚠️ session_search 不支持引号和特殊字符，只用 skill 名称纯文本搜索
3. 对每个命中 session：session_read(session_id=..., limit=50) 获取对话
   ⚠️ session_read 返回的工具调用会被压缩为 [tool: xxx]，无法看到具体参数和输出
   因此需要从 AI 的文本响应中推断步骤执行情况（见下方推断规则）
4. 从对话中提取每次 skill 调用的：
   - 触发时的用户消息（输入上下文）
   - AI 在 skill 指导下的完整响应（执行过程）
   - 最终交付给用户的结果（输出）
   - 用户对结果的反馈（如有）
```

**步骤执行推断规则（应对 session API 限制）：**

由于 `session_read` 不返回工具调用的详细参数，需从 AI 的文本描述推断步骤是否执行：

| 推断信号 | 对应步骤 | 置信度 |
|---------|---------|--------|
| AI 文本中提到 "Step X" 或步骤名称 | 该步骤被执行 | 高 |
| 出现对应工具标记（如 `[tool: bash]` 后跟日志分析描述） | 相关步骤被执行 | 中 |
| 最终输出包含某产出物（如分析报告、简报） | 对应的输出步骤被执行 | 高 |
| AI 明确说"跳过 Step X" | 该步骤未执行 | 确定 |
| 整个对话中无任何该步骤的痕迹 | 该步骤可能未执行 | 中（标记为"不确定"） |

**用户体验信号（额外检查维度）：**

| 信号 | 含义 | 标记 |
|------|------|------|
| 用户连续发送"继续"≥3 次 | AI 执行中断（可能是 skill 问题，也可能是平台/模型中断） | ⚠️ 需区分根因 |
| 用户追问"为什么没有 X" | 预期产出缺失 | ⚠️ 产出可能不完整 |
| 用户正面反馈或无追问 | 执行可能正确 | ✅ 用户满意（推断） |
| 用户重新描述同一问题 | AI 的首次回答不对题 | ❌ 理解偏差 |

**区分"继续"信号的根因：**
- AI 响应为空或只有 `[tool: xxx]` 后无文本 → **平台/模型中断**（不计为 skill 缺陷）
- AI 有完整文本响应但内容偏题或遗漏步骤 → **skill 执行偏差**（计入合规性评估）
- 无法判断 → 标记 `⚠️ 外部因素待排除`，不计入合规率分母

**当推断置信度为"中"或"不确定"时：**
- 在检查结果中标注 `⚠️ 推断`，不标为确定性的 ✅ 或 ❌
- 样本量 < 3 时，在报告中声明"数据不足，结论待验证"

**session_search 限制补充：**
- 纯英文关键词可正常搜索
- **中文关键词搜索会报错** — 改用英文关键词或 skill 名称搜索
- 不支持引号、括号等特殊字符

**提取结果格式（每个实例一条）：**

```json
{
  "session_id": "ses_xxx",
  "timestamp": "2026-06-15",
  "trigger_message": "用户原始消息摘要",
  "input_context": {
    "user_request": "完整用户请求",
    "available_resources": ["日志文件", "源码"],
    "device_model": "T9000"
  },
  "execution_trace": {
    "steps_executed": ["Step 0", "Step 1", "Step 2"],
    "steps_skipped": ["Step 0.5"],
    "sub_skills_loaded": ["eufy-log-analyzer"],
    "sub_skills_not_loaded": ["symptom-to-hypothesis"]
  },
  "output": {
    "deliverables_produced": ["分析报告"],
    "deliverables_missing": ["调查简报"],
    "user_feedback": "正面/负面/无"
  }
}
```

保存到 `<audit_dir>/<skill-name>/instances.jsonl`。

---

### Phase 2: 合规性评估

**目标：** 将每个使用实例与设计检查清单逐项对比，量化执行偏差。

#### 2.1 逐实例检查

对 Phase 1 提取的每个实例，逐项检查设计清单：

| 检查项 | 判定标准 | 判定 |
|--------|---------|------|
| 步骤完整性 | 所有"必须"步骤是否都执行了？ | ✅ 全执行 / ⚠️ 部分跳过 / ❌ 关键步骤缺失 |
| 步骤顺序 | 执行顺序是否符合设计？ | ✅ 正确 / ⚠️ 顺序调整但结果合理 / ❌ 乱序导致问题 |
| 输出完整性 | 所有预期产出物是否都生成了？ | ✅ 全部 / ⚠️ 部分缺失 / ❌ 核心产出缺失 |
| 禁止行为 | 是否触犯了 Common Mistakes 中的禁止项？ | ✅ 无违反 / ❌ 存在违反 |
| Sub-skill 调用 | 该调用的 sub-skill 是否正确调用？ | ✅ 正确 / ⚠️ 遗漏但影响不大 / ❌ 遗漏导致结果错误 |

#### 2.2 汇总评估报告

```markdown
# Skill 效果审计报告：<skill-name>

**审计范围：** <日期范围/session 数量>
**使用实例数：** N
**审计时间：** YYYY-MM-DD

## 总体评分

| 维度 | 得分 | 说明 |
|------|------|------|
| 步骤完整性 | X/N 通过 | M 个实例跳过了必要步骤 |
| 输出完整性 | X/N 通过 | M 个实例缺少核心产出 |
| 禁止行为合规 | X/N 通过 | M 个实例违反禁止规则 |
| Sub-skill 调用 | X/N 通过 | M 个实例遗漏 sub-skill |

**综合合规率：** X% (通过实例数 / 总实例数)

## 偏差模式分析

### 高频偏差（出现 ≥2 次的问题）

| 偏差 | 出现次数 | 涉及实例 | 根因分析 |
|------|---------|---------|---------|
| [描述] | N | ses_xxx, ses_yyy | [为什么 AI 会这样做] |

### 根因分类

| 根因类别 | 含义 | 示例 |
|---------|------|------|
| **指令模糊** | Skill 文档中该步骤的描述不够明确 | "加载 sub-skill" 未说明用什么工具 |
| **指令冲突** | 两条指令互相矛盾 | "必须执行 Step X" 但 "有匹配时跳到 Step Y" |
| **上下文衰减** | 对话过长导致 skill 指令被遗忘 | 在第 30 轮对话后不再遵循 Step 4 |
| **条件判断错误** | AI 对条件分支的判断与设计意图不符 | 把"低置信度匹配"当成"高置信度"处理 |
| **懒惰跳过** | AI 合理化跳过了某个步骤 | "这次情况简单，不需要 Step X" |
```

保存到 `<audit_dir>/<skill-name>/audit-report-YYYY-MM-DD.md`。

---

### Phase 3: 修复方案生成

**目标：** 针对 Phase 2 发现的偏差模式，生成具体的 SKILL.md 修改方案。

#### 3.1 按根因生成修复

| 根因 | 修复策略 | 修改位置 |
|------|---------|---------|
| 指令模糊 | 增加具体操作说明、示例 | 对应 Step 的描述 |
| 指令冲突 | 消除矛盾，明确优先级 | 涉及的多个 Step |
| 上下文衰减 | 在关键位置增加强调标记（⚠️ 必须） | 容易被跳过的步骤 |
| 条件判断错误 | 增加判断标准的量化指标 | 条件分支处 |
| 懒惰跳过 | 在 Common Mistakes 中增加对应条目 | Common Mistakes 表 |

#### 3.2 生成 diff 形式的修复方案

对每个修复，输出：

```markdown
### 修复 #1: [偏差描述]

**根因：** [指令模糊/冲突/衰减/判断错误/懒惰跳过]
**出现频率：** N/M 个实例
**修复位置：** SKILL.md 第 X-Y 行

**当前内容：**
> [原文]

**修改为：**
> [修改后内容]

**修复理由：**
[为什么这样改能解决问题]
```

#### 3.3 用户确认

将所有修复方案展示给用户，逐条确认：
- ✅ 接受
- ❌ 拒绝
- ✏️ 修改后接受

**只有用户确认的修复才会进入 Phase 4。**

---

### Phase 4: 灰度验证

**目标：** 在安全的环境中验证修复是否真正解决了问题，且不引入新问题。

#### 4.1 备份原始 Skill

```bash
AUDIT_DIR="<从 config.json 读取>"
SKILL_NAME="<目标 skill>"
BACKUP_DIR="$AUDIT_DIR/$SKILL_NAME/backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# 完整备份 skill 目录
cp -r "<skill目录>/" "$BACKUP_DIR/"
echo "备份完成：$BACKUP_DIR"
```

**备份后记录回滚信息：**
```json
{
  "skill_name": "<skill-name>",
  "skill_path": "<skill原始路径>",
  "backup_path": "<备份路径>",
  "backup_time": "YYYY-MM-DD HH:MM:SS",
  "fixes_applied": ["修复#1", "修复#2"],
  "status": "pending_verification"
}
```
保存到 `<audit_dir>/<skill-name>/rollback.json`。

#### 4.2 应用修复

将用户确认的修复逐条应用到 SKILL.md。每条修复后校验文件格式（YAML frontmatter 完整、Markdown 无断裂）。

#### 4.3 设计验证场景

从 Phase 1 的使用实例中选取：
- **失败实例**：之前偏差最严重的 2-3 个实例的用户输入
- **成功实例**：之前执行正确的 1-2 个实例的用户输入（回归测试）

#### 4.4 执行验证

验证方式按上下文容量选择：

**方式 A：当前 session 模拟（适合修复 ≤ 2 项、场景简单时）**
1. 使用 `skill` 工具加载修复后的 skill
2. 以历史用户消息为输入，执行 skill 的流程
3. 对照设计检查清单判定

**方式 B：新 session 验证（适合修复多、场景复杂、或当前上下文已长时）**
1. 告知用户："修复已应用，建议在新对话中用以下测试输入验证"
2. 提供具体的验证输入和预期行为
3. 用户验证后回来报告结果

**判定标准（两种方式通用）：**
- 之前跳过的步骤现在是否执行了？
- 之前缺失的产出现在是否生成了？
- 之前正确的部分是否仍然正确？（回归）

#### 4.5 验证结果判定

```markdown
## 灰度验证结果

| 场景 | 类型 | 修复前 | 修复后 | 判定 |
|------|------|--------|--------|------|
| [实例摘要] | 失败修复 | ❌ 跳过 Step X | ✅ 执行了 Step X | 改善 ✅ |
| [实例摘要] | 失败修复 | ❌ 缺少产出 | ✅ 产出完整 | 改善 ✅ |
| [实例摘要] | 回归测试 | ✅ 正常 | ✅ 正常 | 无回归 ✅ |
```

**判定标准：**
- 全部"改善"且无"回归" → **通过**，进入确认
- 部分"改善"且无"回归" → **部分通过**，向用户报告未改善项
- 存在"回归" → **失败**，建议回滚

---

### Phase 5: 回滚/确认

#### 验证通过时

```
灰度验证通过。修复已应用到：<skill 路径>

修复内容摘要：
1. [修复#1 简述]
2. [修复#2 简述]

备份保留在：<备份路径>
如需回滚，使用命令：skill("skill-effectiveness-auditor") → 选择"回滚"
```

更新 `rollback.json` 的 `status` 为 `verified`。

#### 验证失败时

```bash
# 自动回滚
cp -r "$BACKUP_DIR/"* "<skill原始路径>/"
echo "已回滚到修复前版本"
```

更新 `rollback.json` 的 `status` 为 `rolled_back`。

向用户报告：
```
灰度验证未通过，已自动回滚到修复前版本。

失败原因：
- [具体回归/未改善的项目]

建议：
- 重新分析 Phase 2 的偏差根因
- 调整修复策略后重试
```

#### 手动回滚（任何时候）

用户可以随时触发回滚：

1. 读取 `<audit_dir>/<skill-name>/rollback.json`
2. 检查 `status` — 只有 `verified` 或 `pending_verification` 状态可回滚
3. 执行回滚：
```bash
cp -r "<backup_path>/"* "<skill_path>/"
```
4. 更新 `status` 为 `rolled_back`

---

## Quick Reference

```
Phase 1: session_search → 提取使用实例 → instances.jsonl
Phase 2: 设计检查清单 × 实例 → 偏差评估 → audit-report.md
Phase 3: 偏差根因 → 修复方案 → 用户确认
Phase 4: 备份 → 应用修复 → 验证场景 → 判定
Phase 5: 通过→保留 / 失败→回滚
```

| 想要... | 执行到... |
|--------|----------|
| 只看看 skill 执行得怎么样 | Phase 1 + 2（只审计不修复） |
| 审计 + 修复 | Phase 1-4 |
| 回滚之前的修复 | 直接 Phase 5 手动回滚 |

## Common Mistakes

| 错误 | 正确做法 |
|------|----------|
| 只看最终输出不看执行过程 | 必须检查 execution_trace（步骤是否完整、顺序是否正确） |
| 把"用户没投诉"等同于"执行正确" | 用户可能不知道 skill 跳过了某些步骤 |
| 修复前不备份 | Phase 4.1 备份是**强制**步骤，不可跳过 |
| 一次改太多 | 每次修复聚焦于 1-2 个高频偏差，避免引入新问题 |
| 灰度验证只测失败场景 | 必须同时测回归场景，确保原来正确的部分没被破坏 |
| 跳过用户确认直接修复 | Phase 3.3 用户确认是**强制**步骤 |
| 回滚后不更新 rollback.json | 状态不一致会导致后续操作混乱 |
| 只审计一次就下结论 | 样本量少于 3 个实例时，结论不可靠，需积累更多数据 |
| 假设 session_read 能看到工具调用细节 | 实际只能看到 `[tool: xxx]` 标记，需从 AI 文本推断步骤执行 |
| session_search 用中文或复杂查询 | 中文查询会报错，只用英文关键词或 skill 名称纯文本搜索 |
| 只看步骤完整性不看用户体验 | "继续"连发 ≥3 次 = 执行中断信号，必须记录 |
| 在上下文已满的 session 中做灰度验证 | 修复多或场景复杂时，建议用新 session 验证 |
