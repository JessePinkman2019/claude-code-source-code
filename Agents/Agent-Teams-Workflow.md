---
title: Claude Code Agent Teams End-to-End Workflow
tags:
  - agents
  - claude-code
  - agent-teams
  - workflow
source: https://claudefa.st/blog/guide/agents/agent-teams-workflow
date: 2026-04-27
---

# Claude Code Agent Teams: End-to-End Workflow

> [!tip] 核心洞察
> Agent Teams 的可靠性来自两个阶段：**规划阶段**（消除假设、定义契约）和**执行阶段**（按波次 spawn agent，注入契约，并行构建）。

前置阅读：[[Agent-Teams]] | [[Agent-Teams-Controls]]

---

## 为什么需要工作流，而不只是功能

没有流程的 Agent Teams 等同于给五个承包商钥匙却没有蓝图——各自构建，互不兼容。

**两类核心失败**：
1. **假设漂移**：每个 agent 独立决定数据结构、命名、错误格式。后端返回 `{ notif_type: "comment" }`，前端期望 `{ type: "COMMENT" }`——各自通过测试，集成失败。
2. **缺失验证**：所有 agent 报告成功，但无人做端到端测试，错误到手动测试才暴露。

---

## 完整七步流程

### 阶段一：规划（第 1-3 步）

#### 第 1 步：Brain Dump

不要过度思考初始输入，直接用自然语言写出所有想法，包括混乱、矛盾、不完整的部分：

```
I need a payment integration for my chat app. Users should purchase
tokens and consume one token per conversation turn. I want to use
ChargeB as the payment processor. Need to touch the database for
token balances, the backend for webhook handling and token deduction,
and the frontend for a billing page. Users should get 10 free tokens
on signup. If the AI call fails, auto-refund the token.
```

Brain dump 也是范围检查——如果你连几段话都说不清楚，说明功能太大，先拆分。

---

#### 第 2 步：调研与 Q&A（最容易被跳过、最重要的步骤）

> [!important] 目标
> **在写代码之前减少假设**。不是 2-3 个问题，至少 10 个。

```
Read the codebase. Understand our current database schema, API patterns,
auth system, and frontend component structure. Then ask me at least 10
clarifying questions about the payment integration before we plan anything.
Focus on things that could go wrong if you assumed incorrectly.
```

Claude 会问：hosted checkout 还是 embedded in-app？token 套餐和价格？零 token 时 UI 行为？……每个问题代表一个实现分叉点，10 分钟 Q&A 省去数小时返工。

---

#### 第 3 步：结构化计划

计划必须包含：**团队成员（角色+agent type）**、**任务（含依赖链 `Depends On` 字段）**、**文件所有权边界**、**验收标准**、**验证命令**。

```markdown
## Team Orchestration

### Team Members
- builder-database (backend-engineer): Database schema and migrations
- builder-api (backend-engineer): API endpoints and webhook handling
- builder-frontend (frontend-specialist): Billing page and token display
- validator (quality-engineer): Validate against acceptance criteria

## Step by Step Tasks

### 1. Setup Database
- Task ID: setup-database
- Depends On: none
- Assigned To: builder-database

### 2. Build API
- Task ID: build-api
- Depends On: setup-database
- Parallel: true (with build-frontend)

### 3. Build Frontend
- Task ID: build-frontend
- Depends On: setup-database
- Parallel: true (with build-api)

### 4. Final Validation
- Task ID: validate-all
- Depends On: setup-database, build-api, build-frontend
- Assigned To: validator
```

`Depends On` 字段不只是文档，而是执行阶段用于推导波次顺序和提取契约的依据。

---

### 阶段二：执行（第 4-7 步）

#### 第 4 步：全新上下文启动

**开一个新 session，只带计划文件**。规划对话已消耗大量上下文，充满探索性问答和被否定的想法——这些是噪音。计划是精华，丢掉规划对话不丢失任何信息，还能复用计划做重试。

---

#### 第 5 步：契约链分析

在 spawn 任何 agent 之前，Lead 分析依赖图，推导**契约链**：

```
依赖图 → 波次划分：
  Wave 1: [builder-database]   → 产出 schema 契约
  Wave 2: [builder-api] + [builder-frontend] 并行
          → 双方消费 schema 契约
          → builder-api 产出 API 契约
  Wave 3: [validator]          → 验证全部

关键原则：任何 agent 在拿到所依赖的契约之前不启动。
```

Database agent 完成后，将**实际** schema 定义（表结构、列类型、TypeScript 类型）发给 Lead，Lead 将这些真实内容直接粘贴进后续 agent 的 spawn prompt——不是"去看 database agent 做了什么"的引用，而是实际内容。

---

#### 第 6 步：波次执行

**Teammate spawn prompt 结构**：

```
You are builder-api, the backend specialist on this team.

YOUR TASKS:
- Implement ChargeB webhook handler at /api/webhooks/chargeb
- Build token deduction middleware
- Handle auto-refund on AI call failure

FILES YOU OWN (only modify these):
- src/api/webhooks/chargeb/*
- src/middleware/tokens.*

UPSTREAM CONTRACTS (your inputs):
[粘贴 database agent 实际产出的 schema 定义和 TypeScript 类型]

CONTRACTS YOU MUST PRODUCE (your outputs):
- API endpoint signatures (routes, request/response shapes)
- Webhook payload format
- Token deduction function signature

COORDINATION:
- Message the lead when your contract is ready
- Message builder-frontend if you change any API response shape
- Mark your task complete in the shared task list when done
```

> [!note] Brownfield 代码库
> 多 agent 同时修改已有代码库时，CLAUDE.md 必须记录命名规范、错误处理、文件结构等约定，否则一个 agent 用 camelCase，另一个用 snake_case。

---

#### 第 7 步：验证循环

**要求证据，不要确认**：

```
For each integration point, show me:
1. The exact API response from hitting POST /api/webhooks/chargeb
2. The database state after a token purchase
3. The chat UI behavior at zero tokens
4. The full test suite output

If any of these cannot be demonstrated, the task is not complete.
```

发现问题时的修复循环：
1. Lead 定位不匹配点（如 webhook handler 返回错误状态码）
2. Lead 消息责任 teammate 或 spawn 定向修复 agent
3. 应用修复
4. Lead 重新运行受影响的验证检查

有完善契约链时，通常 1-2 轮即可收敛。

---

## 全流程速查表

| 步骤 | 阶段 | 动作 | 为什么重要 |
|------|------|------|------------|
| 1. Brain dump | 规划 | 自然语言写需求 | 无需过早结构化，捕获意图 |
| 2. 调研 & Q&A | 规划 | Claude 分析代码库，问 10+ 问题 | 在规划前消除假设 |
| 3. 结构化计划 | 规划 | 定义团队、任务、依赖、验收标准 | 为每个 agent 给出清晰非重叠范围 |
| 4. 全新上下文 | 执行 | 新 session 只带计划文件 | 最大化上下文，丢弃规划噪音 |
| 5. 契约链分析 | 执行 | 从依赖图推导波次顺序和接口 | 防止并行构建的集成失败 |
| 6. 波次执行 | 执行 | 注入契约后按波次 spawn agent | 快速并行构建，保证兼容性 |
| 7. 验证 | 执行 | 按验收标准端到端测试 | 捕获单元测试无法发现的接缝错误 |

**时间分配参考**：规划阶段 15-30 分钟（占 30%），执行阶段基本自动运行（占 70%）。30% 的规划投入节省 70% 的集成返工。

---

## 延伸阅读

- [[Agent-Teams]] — 总览与基础概念
- [[Agent-Teams-Controls]] — delegate mode、plan approval、hooks
- [[Agent-Teams-Use-Cases]] — 场景 prompt 模板
- [[Agent-Teams-Best-Practices]] — Troubleshooting 与限制
- [[Team-Orchestration]] — Builder-Validator 模式（结构化修复循环）
