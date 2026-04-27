---
title: Claude Code Agent Orchestration Patterns
tags:
  - agents
  - claude-code
  - orchestration
  - patterns
source: https://claudefa.st/blog/guide/agents/agent-patterns
date: 2026-04-27
---

# Claude Code Agent Patterns：六种编排策略

> [!tip] 核心原则
> 模式选错会导致 token 浪费、merge conflict 或输出不一致。选对模式，agent 能以最少监督处理复杂工作。

相关笔记：[[Agent-Fundamentals]] | [[Custom-Agents]] | [[Team-Orchestration]]

---

## 1. Orchestrator 模式

**核心**：中央 AI 线程协调各专家 agent，自己不写代码，只负责计划、委派、审查和路由。

**适用场景**：跨前端、后端、数据库的多域功能，需要有人看到全局。

```
You are the orchestrator. For this feature request, create a plan that:
1. Breaks the work into domain-specific tasks
2. Identifies dependencies between tasks
3. Dispatches each task to a sub-agent with explicit file scope
4. Reviews outputs before marking complete

Do NOT write implementation code yourself. Coordinate only.
```

主 chat session 成为 orchestrator，通过 Task tool dispatch 专家，审查返回结果后决定下一步。

**不适用**：单 agent 能一次完成的简单任务。编排开销超过收益。

---

## 2. Fan-Out / Fan-In 模式

**核心**：并行 dispatch 多个 agent，合并结果为单一输出。

**适用场景**：调研任务、多文件分析，agent 在独立输入上工作。各模块代码审查、在做决策前从代码库不同部分收集信息。

```
Complete these 4 tasks using parallel sub-agents:

1. Read src/api/ and list all endpoints missing input validation
2. Read src/auth/ and identify any hardcoded secrets or weak patterns
3. Read src/db/ and check for missing indexes on frequently queried columns
4. Read src/utils/ and flag any functions with no error handling

After all agents report back, synthesize findings into a prioritized action list.
```

Fan-out 阶段快速（agent 无共享状态）。Fan-in 阶段产生价值：orchestrator 发现单个 agent 无法看到的跨域关联。

**不适用**：agent 需要修改同一文件时。并行写入重叠文件 = merge conflict。

---

## 3. Validation Chain 模式

**核心**：builder agent 创建代码，独立的 validator agent 检查代码，两者角色永不重叠。

**适用场景**：生产代码变更、安全敏感工作、输出错误代价高的任何场景。

```
TaskCreate(
  subject="Build payment webhook handler",
  description="Create Stripe webhook handler in src/api/webhooks/stripe.ts.
  Handle checkout.session.completed, payment_intent.failed events.
  Verify webhook signatures. Include error handling."
)

TaskCreate(
  subject="Validate payment webhook handler",
  description="Read src/api/webhooks/stripe.ts. Verify:
  - Webhook signature verification exists
  - Both event types handled with proper responses
  - Error handling covers malformed payloads
  - No hardcoded secrets
  Report issues only. Do NOT modify any files."
)

TaskUpdate(taskId="2", addBlockedBy=["1"])
```

`addBlockedBy` 确保 validator 等待 builder 完成。Validator 以全新视角开始，没有 builder 的假设和盲点。详见 [[Team-Orchestration]]。

**不适用**：速度优先于正确性的快速原型。验证步骤使每个任务的计算成本翻倍。

---

## 4. Specialist Routing 模式

**核心**：根据任务类型将任务路由到领域专家 agent。Frontend 任务给前端专家，数据库迁移给数据库专家。

**适用场景**：有多个域的大型项目；各域有既定规范；通用 agent 指令产生不一致输出。

CLAUDE.md 中定义路由表：

```markdown
## Agent Routing Table

| Task Domain | Route To            | File Scope           |
|-------------|---------------------|----------------------|
| React/UI    | frontend-specialist | src/components/      |
| API routes  | backend-engineer    | src/api/, src/lib/   |
| Database    | database-specialist | src/db/, migrations/ |
| Security    | security-auditor    | Any (read-only)      |
| Tests       | quality-engineer    | tests/, **tests**/   |
```

`.claude/agents/` 中的专家定义自动继承 CLAUDE.md，无需额外配置编码规范。新增域 = 新增一个 agent 定义 + 路由表一行。

**不适用**：小项目，单个 agent 足以了解所有内容。

---

## 5. Progressive Refinement 模式

**核心**：先粗稿，再通过多轮迭代提升质量，每轮专注一个质量维度。

**适用场景**：内容生成、复杂架构设计、任何"一次性搞定"概率低的任务。

```
Phase 1 - Draft: "Generate the initial API schema for a task management
system. Include all entities, relationships, and basic validation rules."

Phase 2 - Security review: "Review this schema. Add authentication
requirements, permission checks, and input sanitization rules.
Don't change the core structure."

Phase 3 - Performance review: "Review the schema for performance.
Add indexes, identify N+1 query risks, suggest denormalization
where read performance matters."

Phase 4 - Final validation: "Verify the schema is consistent.
Check that all referenced entities exist, foreign keys are valid,
and naming conventions are uniform."
```

草稿 agent 优先完整性；安全 agent 添加约束；性能 agent 做优化；验证 agent 检查一致性。无单一 agent 同时处理所有关注点。

**不适用**：必须一次完成的任务，或第一遍已经足够好的简单任务。

---

## 6. Watchdog 模式

**核心**：后台 agent 持续监控特定条件，触发时告警或行动，不阻塞主工作流。

**适用场景**：长时间 session 中的漂移监控；重构期间的回归检查；在做其他事的同时监视构建失败。

```
Background task: Monitor the test suite while I refactor the auth module.
Every time I complete a change, run the test suite for src/auth/.
If any test fails, immediately create a task with:
- Which test failed
- The assertion error
- Which file I likely broke based on the test name
```

用 `Ctrl+B` 后台运行监控 agent。发现问题时自动在任务列表创建条目。

配合 [[Agent-Fundamentals]] 中的 async workflows 使用效果最佳。

**不适用**：短 session，监控开销超过监控价值。

---

## 模式组合

真实项目通常不单独使用某一模式：

```
1. Orchestrator  读取需求、创建计划
2. Specialist Routing  将任务分发给域专家
3. Fan-Out  并行运行独立域任务
4. Validation Chain  验证每个专家的输出
5. Progressive Refinement  打磨集成结果
6. Watchdog  全程监控测试套件
```

**关键技能**：识别当前任务适合哪种模式，从最简单的开始，只在更简单的方案失败时增加复杂度。

---

## 延伸阅读

- [[Agent-Fundamentals]] — Sub-agent 基础
- [[Custom-Agents]] — 自定义 agent 定义与 slash commands
- [[Team-Orchestration]] — Builder-Validator 完整工作流
- [[Agent-Teams]] — Native Agent Teams 协作
- [Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) — 并行 vs 顺序路由决策
