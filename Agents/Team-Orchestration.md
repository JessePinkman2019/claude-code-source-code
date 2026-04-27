---
title: Claude Code Team Orchestration: Builder-Validator Agent Patterns
tags:
  - agents
  - orchestration
  - claude-code
  - multi-agent
source: https://claudefa.st/blog/guide/agents/team-orchestration
date: 2026-04-27
---

# Claude Code Team Orchestration: Builder-Validator Agent Patterns

> [!tip] 核心模式
> 用 builder + validator 角色对，通过 TaskUpdate `addBlockedBy` 建立依赖链，让 agent 互相检查工作。

相关笔记：[[Agent-Fundamentals]] | [[Sub-Agent-Best-Practices]]

---

## 为什么要用 Pairs 而不是单个 Agent

单个 agent 无法客观审查自己的输出——它和创造 bug 时有相同的盲点。将 builder 与独立的 validator 配对，因为 validator 从全新上下文出发，能捕获 builder 遗漏的问题。

---

## Builder-Validator 模式

**Builder prompt**（限定为创建）：
```
You are a builder agent. Your job:
1. Read the task description carefully
2. Implement the solution in the specified files
3. Run any relevant tests
4. Mark your task complete

Rules:
- Only modify files listed in your task
- Do not modify test files
- If you hit a blocker, document it and mark complete
```

**Validator prompt**（限定为验证）：
```
You are a validator agent. Your job:
1. Read all files the builder created or modified
2. Check against the acceptance criteria
3. Run the test suite
4. Report findings as a new task if issues exist

Rules:
- Do NOT modify any source files
- Do NOT create new implementation code
- You may only create or update task entries to report issues
- Use Read and Bash (for tests) only - never Edit or Write
```

> [!important] 关键约束
> Validator 不能写代码，只能通过创建新 task 来暴露问题，可用 `disallowedTools` 在工具层面强制执行。

---

## 用 addBlockedBy 建立依赖链

```
// Phase 1：并行 builders
TaskCreate(subject="Build user API routes", ...)
TaskCreate(subject="Build user database schema", ...)

// Phase 2：validator 等待各自的 builder
TaskCreate(subject="Validate API routes", ...)
TaskCreate(subject="Validate database schema", ...)

TaskUpdate(taskId="3", addBlockedBy=["1"])
TaskUpdate(taskId="4", addBlockedBy=["2"])
```

跨切面验证（需要所有 builder 完成后才开始）：
```
TaskCreate(subject="Integration validation", ...)
TaskUpdate(taskId="5", addBlockedBy=["1", "2"])
```

---

## 快速上手示例

```
TaskCreate(subject="Build auth middleware", description="Create JWT validation middleware in src/middleware/auth.ts. Export verifyToken and requireAuth functions.")
TaskCreate(subject="Validate auth middleware", description="Read src/middleware/auth.ts. Verify: exports exist, error handling covers expired/malformed tokens, no hardcoded secrets. Report issues only. Do NOT modify files.")
TaskUpdate(taskId="2", addBlockedBy=["1"])
```

---

## Meta-Prompt：从需求自动生成团队计划

在 CLAUDE.md 中添加：

```
## Team Plan Generation

When I say "team plan: [feature]", generate a task structure:

For each component:
1. TaskCreate a builder task with specific files and acceptance criteria
2. TaskCreate a validator task scoped to read-only verification
3. TaskUpdate to chain validator behind its builder

After all component pairs, add one integration validator blocked by ALL builders.

Format each task description with:
- **Files**: exact paths to create or read
- **Criteria**: measurable acceptance conditions
- **Constraints**: what this agent must NOT do
```

使用时只需说："team plan: add Stripe webhook handler"，Claude 自动生成完整依赖图。

---

## 失败验证的恢复循环

1. Validator 创建描述问题的 fix task
2. Fix task 分配给 builder agent
3. 新的 validator task 链在 fix 之后

```
TaskCreate(subject="Fix: add error handling to user API", description="GET /users/:id on invalid ID format returns 500. Add input validation, return 400 for malformed IDs.")
TaskCreate(subject="Re-validate user API error handling", ...)
TaskUpdate(taskId="7", addBlockedBy=["6"])
```

每次循环缩小问题范围，直至收敛。

---

## 扩展阅读

- [Agent Fundamentals](https://claudefa.st/blog/guide/agents/agent-fundamentals)
- [Task Distribution](https://claudefa.st/blog/guide/agents/task-distribution)
- [Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices)
- [Hooks Guide](https://claudefa.st/blog/tools/hooks/hooks-guide)
- [Self-Validating Agents](https://claudefa.st/blog/tools/hooks/self-validating-agents)
