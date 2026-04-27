---
title: Claude Code Agent Teams Use Cases with Copy-Paste Prompts
tags:
  - agents
  - claude-code
  - agent-teams
  - prompts
source: https://claudefa.st/blog/guide/agents/agent-teams-use-cases
date: 2026-04-27
---

# Claude Code Agent Teams: Use Cases with Copy-Paste Prompts

> [!tip] 快速入门
> 最通用的起点是并行代码审查——三个 reviewer，三个视角，适用任何代码库。

前置阅读：[[Agent-Teams]] | [[Agent-Teams-Controls]]

---

## 开发类场景

### 1. 并行代码审查

```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings. Use delegate mode so the
lead synthesizes a final review without doing its own analysis.
```

**为什么有效**：将审查维度拆分为独立域，三个视角同时进行。开启 delegate mode 让 Lead 专注综合，不做自己的审查。预计 token 消耗约为单 session 的 2-3 倍。

---

### 2. 竞争假说 Debug

```
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk
to each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```

**为什么有效**：辩论结构对抗"锚定偏差"——多个独立调查员主动证伪，存活的理论更可能是真正根因。Teammates 之间可直接分享发现，无需经 Lead 中转。

---

### 3. 全栈功能实现

```
Create an agent team to implement the user notifications system.
Spawn four teammates:
- Backend: Create the notification service, database schema, and API endpoints
- Frontend: Build the notification bell component, dropdown, and read/unread states
- Tests: Write integration tests for the full notification flow
- Docs: Update the API documentation and add usage examples

Assign each teammate clear file boundaries. Backend owns src/api/notifications/
and src/db/migrations/. Frontend owns src/components/notifications/.
Tests own tests/notifications/. No file overlap.
```

> [!important] 关键细节
> **目录级文件所有权**是实现类 prompt 中最重要的一条，直接防止 merge conflict。

---

### 4. 架构决策记录

```
Create an agent team to evaluate database options for our new analytics feature.
Spawn three teammates, each advocating for a different approach:
- Teammate 1: Argue for PostgreSQL with materialized views
- Teammate 2: Argue for ClickHouse as a dedicated analytics store
- Teammate 3: Argue for keeping everything in the existing MongoDB

Have them challenge each other's arguments. Focus on: query performance
at 10M+ rows, operational complexity, migration effort, and cost.
The lead should synthesize a decision document with the strongest arguments
from each side.
```

**为什么有效**：对抗结构强制真正评估备选方案，而非单 agent 过早选定后自我合理化。

---

### 5. 性能瓶颈分析

```
Create an agent team to identify performance bottlenecks in the application.
Spawn three teammates:
- One profiling API response times across all endpoints
- One analyzing database query performance and indexing
- One reviewing frontend bundle size and rendering performance

Have them share findings when they discover something that affects
another teammate's domain (e.g., slow API caused by missing DB index).
```

**跨域通信**是 Agent Teams 优于 Subagents 的关键：Database analyst 发现的 missing index 可直接通知 API teammate，无需经 Lead 路由。

---

### 6. 大批量分类处理

```
Create an agent team to classify our product catalog. We have 500 items
that need categorization, tagging, and description updates.
Spawn 4 teammates, each handling a segment:
- Teammate 1: Items 1-125
- Teammate 2: Items 126-250
- Teammate 3: Items 251-375
- Teammate 4: Items 376-500

Use the classification schema in docs/taxonomy.md. Have teammates
flag edge cases for the lead to review.
```

按数据边界拆分的任务可线性扩展，4 个 teammate 速度约为单 session 的 4 倍。

---

## 非开发类场景

### 7. 营销调研冲刺

```
Create an agent team to research the launch strategy for [product].
Spawn three teammates:
- Competitor analyst: study competitor ad copy, positioning, and pricing
- Voice of customer researcher: mine reviews, Reddit threads, and forums
  for pain points and language customers actually use
- Positioning stress-tester: take findings from both teammates and
  pressure-test our current positioning against what they discover

Have them share findings and challenge each other. The lead synthesizes
a strategy document with positioning recommendations.
```

---

### 8. 落地页构建（含对抗审查）

```
Create an agent team to build the landing page for [offer].
Spawn three teammates:
- Copywriter: develop messaging, headlines, and body copy
- CRO specialist: design conversion structure, CTA placement, and flow
- Skeptical buyer: review everything as a resistant prospect, flag
  weak claims, missing proof, and friction points

Require plan approval before any implementation.
```

Plan approval 在此处尤为重要——落地页文案重写代价高，方案阶段早发现弱 value proposition 远比全页重写便宜。

---

### 9. 广告创意探索

```
Spawn 4 teammates to explore different hook angles for [product].
Each teammate develops one direction with headline variations,
supporting copy, and a rationale for why the angle works.
Have them debate which direction is strongest.
Update findings doc with consensus and runner-up options.
```

---

### 10. 内容生产流水线

```
Create a team for this week's content calendar.
Spawn three teammates:
- Researcher: identify search intent gaps and competitive opportunities
- Writer: draft content based on research findings
- Quality reviewer: run each piece through clarity, proof, and SEO checks

Chain tasks so the researcher finishes before the writer starts,
and the reviewer checks each piece before marking it complete.
```

通过任务依赖链确保执行顺序，详见 [[Team-Orchestration]]。

---

## 渐进式入门路径

| 周 | 场景 | 目标 |
|----|------|------|
| 第 1 周 | 并行代码审查（PR review） | 了解任务列表与通信机制 |
| 第 2 周 | 竞争假说 Debug | 观察 agent 间实时协作 |
| 第 3 周 | 功能实现（带文件边界） | 掌握并行构建与协调全流程 |

---

## 写出更好 Team Prompt 的技巧

- **明确角色**："one on security, one on performance" 优于 "reviewers"
- **定义文件边界**：目录级所有权，实现类任务必须
- **包含成功标准**："Report findings" 或 "update the decision doc"
- **delegate mode**：纯协调场景下使用
- **对风险任务要求 plan approval**：创意和实现类任务尤其重要
- **让 teammate 争论**：辩论模式优于共识模式
- **团队规模 3-5**：超过 5 人协调开销往往超过并行收益

---

## 延伸阅读

- [[Agent-Teams]] — 总览与基础概念
- [[Agent-Teams-Controls]] — 控制与配置
- [[Agent-Teams-Best-Practices]] — 最佳实践与 Troubleshooting
- [[Agent-Teams-Workflow]] — 端到端生产构建工作流
