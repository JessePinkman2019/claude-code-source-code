---
tags:
  - claude-code
  - opus-4-7
  - best-practices
  - model-behavior
source: https://claudefa.st/blog/guide/development/opus-4-7-best-practices
date: 2026-04-16
related:
  - "[[agentic-engineering-best-practices]]"
---

# Claude Opus 4.7 Best Practices for Claude Code

#claude-code #opus-4-7 #best-practices #model-behavior

> 原文：https://claudefa.st/blog/guide/development/opus-4-7-best-practices  
> 发布时间：2026-04-16  
> 作者：ClaudeFast（参考 Anthropic 官方 best-practices 指南）

---

## 核心变化：字面主义（Literalism）

Opus 4.7 最大的转变不是原始智能提升，而是**更严格的字面解释**。模型会精确执行你的指令——这对模糊 prompt 是惩罚，对详细计划是奖励。

Boris Cherny（Anthropic）原话：*"花了几天才学会怎么跟它有效配合。"*

在 4.6 上靠模型自行补全上下文的 prompt，在 4.7 上会产生更窄、更字面化的结果。

> 字面主义让 [[agentic-engineering-best-practices]] 中的五个实践价值成倍放大——PRD 和显式计划变得更加关键。

---

## 四个关键行为变化

### 1. 更严格的指令遵循
#instruction-following

- 字面解释指令，不再自行补全隐含需求
- Notion 发现它是第一个能通过其"隐式需求测试"的模型
- 副作用：依赖模型"脑补"的 prompt 表现下降

### 2. 更保守的子 Agent 生成
#multi-agent

- 默认偏向在单次响应中完成工作，而非展开并行
- **如果需要并行化，必须显式说明**

### 3. 自适应 Thinking（思考预算）
#thinking #reasoning

- 固定思考预算已废弃，模型根据上下文自行决定推理深度
- 可通过 prompt 影响：
  - 更多推理："think carefully before responding"
  - 快速响应："prioritize responding quickly"

### 4. 新的 `xhigh` Effort 默认值
#effort-levels

- Effort 层级：`low` → `medium` → `high` → `xhigh` → `max`
- Claude Code 默认为 `xhigh`，介于 `high` 和 `max` 之间
- 获得大部分推理深度，同时不付出 `max` 的全部成本

---

## 核心思维转变

> 把 Claude 当作你在委派任务的能干工程师，而不是一起结对编程的伙伴。

**Front-load（前置）所有信息**：
- 意图（Intent）
- 约束条件（Constraints）
- 验收标准（Acceptance Criteria）
- 相关文件路径（File Paths）

每次用户 turn 都会增加推理开销，所以**把问题批量放在一个 turn 里**。

---

## 计划的杠杆效应

#planning

4.7 最大的杠杆点是**范围清晰的计划**。

- 包含 12 条验收标准的计划 → 产出 12 个被检查的条目
- 意图模糊的计划 → 产出模糊的实现

好的计划文件应包含：
- 意图与范围边界
- 相关文件及行号
- 每个任务的验收标准
- 专项 Agent 分配
- 完成前的验证步骤

---

## Effort 级别使用策略

#effort-levels #token-optimization

| 阶段 | Effort | 原因 |
|------|--------|------|
| 规划 | `xhigh` | 计划质量会复利到每个执行步骤 |
| 执行 | `high` | 专项 Agent 基于清晰计划工作 |
| 验证 | `xhigh` | 在合并前捕捉偏差 |
| 探索 / 文档 | `medium` | 低风险，成本敏感 |
| 深度评估 | `max` | 正确性关键工作，值得付出成本 |

注意：新 tokenizer 使相同输入可能产生 1.0–1.35x 更多 token（与 4.6 相比），高 effort 下 session 消耗更快。

---

## 子 Agent 使用原则

#multi-agent #parallelization

Anthropic 原文：
> "不要为可以在单次响应中直接完成的工作生成子 Agent。在同一 turn 中，当需要在多个条目上展开时，生成多个子 Agent。"

**正向 prompt 优于否定 prompt**（4.7 的特性）：
- 好："spawn a specialist for each of: frontend, backend, database"
- 差："don't try to do this in one response"

---

## 长时任务（Auto Mode）推荐配置

#auto-mode #long-horizon

- 使用 auto mode + `xhigh` effort
- 不要在中途打断——Cognition（Devin）报告 4.7"能连续工作数小时，持续攻克难题"，但只有在不中断的情况下才会展现此行为

---

## 针对 4.7 的 5 条实用调整

1. **把隐式上下文转为显式**：如果 prompt 在 4.6 上能生效是因为模型推断"你显然也要测试"，那在 4.7 上要把测试明确写进验收标准

2. **用正向指令替代否定指令**：负面指令（"不要做 X"）在 4.7 上产生不可靠结果，改用正面示例

3. **显式声明并行需求**：想要多 agent 就说清楚展开模式，4.7 默认单次响应

4. **批量问题进单个 turn**：三个相关问题放一个 turn，比三次顺序 turn 更高效

5. **前置文件路径**：传具体路径（`apps/web/src/app/(home)/page.tsx`）而非"主页文件"，可节省两次工具调用

---

## 总结

4.7 奖励投入计划的操作者。模型不需要比 4.6 更少的指导，它需要**范围更清晰的指导**。

> 写下你希望模型能推断出来的计划，然后让模型去执行它。

**配套实践**：[[agentic-engineering-best-practices]]
