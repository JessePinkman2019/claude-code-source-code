---
tags:
  - agents
  - claude-code
  - sub-agents
  - orchestration
  - parallel
source: https://claudefa.st/blog/guide/agents/task-distribution
date: 2026-04-27
---

# Task Distribution

> Parallelize complex projects with Claude Code task distribution. Coordination strategies, failure modes, and proven patterns.

## 核心问题

复杂项目在 Claude Code 中容易被单线程执行卡住：Claude 一次只做一个任务，但很多工作本可以分给多个独立 sub-agent 并行完成。

**快速上手**：在 CLAUDE.md 中加入功能实现分发模式：

```markdown
# Feature Implementation Pattern
When implementing features, use 7-parallel-Task distribution:
1. Component: Create main component file
2. Styles: Create component CSS/styling
3. Tests: Create test files
4. Types: Create TypeScript definitions
5. Hooks: Create custom hooks/utilities
6. Integration: Update routing and imports
7. Config: Update docs and package.json
```

这样在请求复杂功能时，Claude 会倾向于派生多个 Task agents 并行执行，而不是串行排队。

---

## Task Agent Orchestration

Claude Code 的 Task tool 是并行执行的机制：

- 每次调用 Task tool 都会派生一个独立 sub-agent
- sub-agent 拥有自己的 context window
- 主 agent 负责协调、收集结果、合并输出
- sub-agent 可减少主会话中的交互等待、上下文切换和串行瓶颈

默认情况下，Claude 通常会用 Read / Grep / Glob 等工具在主线程完成读取、搜索和抓取；如果没有明确的委派规则，它很少主动大规模并行派生 agent。

因此，CLAUDE.md 中的分发规则会改变默认行为。

---

## 多线程心智模型

把 Claude 当成线程协调器：

- **Boundary Definition**：每个 agent 负责清晰的文件类型或操作范围
- **Conflict Avoidance**：避免多个 agent 写同一资源
- **Context Optimization**：委派时去掉无关上下文
- **Logical Grouping**：小任务合并，避免过度拆碎

关键不是“越多 agent 越好”，而是要像设计并发程序一样设计任务边界。

---

## 并行分发策略

### 7-Agent Feature Pattern

适合典型前端 / 全栈功能开发：

```markdown
## Parallel Feature Implementation Workflow

When implementing features, spawn 7 parallel Task agents:

1. Component: Create main component file
2. Styles: Create component styles/CSS
3. Tests: Create test files
4. Types: Create type definitions
5. Hooks: Create custom hooks/utilities
6. Integration: Update routing, imports, exports
7. Remaining: Update package.json, docs, config files

### Context Optimization Rules

- Strip comments when reading code files for analysis
- Each Task handles ONLY specified files or file types
- Task 7 combines small config/doc updates to avoid over-fragmentation
```

作用：消除 feature implementation 中的串行瓶颈。

成功信号：你会看到 Claude 在同一轮里多次调用 Task tool，多个 agent 同时运行。

### Role-Based Delegation

适合代码审查和分析任务：

```markdown
Analyze this codebase using parallel Task agents with these roles:
- Senior engineer: Architecture and performance
- Security expert: Vulnerability assessment
- QA tester: Edge cases and validation
- Frontend specialist: UI/UX optimization
- DevOps engineer: Deployment considerations
```

不同角色会自然关注不同问题，能产生单 agent 难以覆盖的分析宽度。

### Domain-Specific Distribution

适合后端或全栈系统：

```markdown
Implement user authentication system using parallel Task agents:
1. Database schema and migrations
2. Auth middleware and JWT handling
3. User model and validation
4. API routes and controllers
5. Integration tests
6. Documentation updates
```

关键：每个 domain 要有清晰文件边界。

---

## Agent 协调优化

### Token 成本 vs 性能

更多 Task agents 不一定更好。

每个 Task 都需要：

- 初始化上下文
- 理解任务
- 读取必要文件
- 返回结果

如果一个功能只改 4 个文件，却派生 12 个 agent，会把大量 token 花在重复上下文加载上。

### Context Preservation

主 agent 会决定把哪些上下文传给 sub-agent。委派指令应保证：

- sub-agent 拿到 domain-specific 信息
- 不携带无关项目背景
- 不继承过长 CLAUDE.md 式百科上下文

这与 [[Effective Context Engineering for AI Agents]] 的“最小高信噪比上下文”一致。

### Conflict Resolution

并行工作的第一原则：不要让两个 agent 写同一个文件。

推荐边界：

- 文件级所有权
- 目录级所有权
- feature-level 所有权

避免：

- function-level 拆分
- 两个 agent 同时改 barrel file / index.ts
- 多个 agent 同时改 package.json 或路由表

### Feedback Integration

Task agents 的结果会回到主 agent。分发前要考虑：

- 哪些结果需要合并
- 哪些 agent 有依赖关系
- 哪些输出是后续验证阶段输入

---

## 高级分发模式

### Validation Chains

最常见质量模式：先并行实现，再串行验证。

```markdown
# Implementation phase (parallel Task agents)
Tasks 1-5: Core feature development

# Validation phase (sequential, after implementation)
Task 6: Integration testing
Task 7: Security review
Task 8: Performance verification
```

为什么验证要串行？

因为验证 agent 必须看到所有实现完成后的最终状态。如果实现还在并行写文件，验证会产生：

- false positives
- missed issues
- 基于中间态的错误判断

### Research Coordination

研究任务最适合并行，因为通常只读、无写冲突。

```markdown
Research user dashboard implementations using parallel Tasks:
1. Technical: React dashboard libraries and patterns
2. Design: Modern dashboard UI/UX examples
3. Performance: Optimization strategies for data-heavy UIs
4. Accessibility: WCAG compliance for dashboard interfaces
```

每个 research agent 返回结构化总结，主 agent 再综合成统一建议。

### Cross-Domain Projects

全栈功能可并行，但必须先建立共享契约。

推荐流程：

1. 先串行生成 API schema / TypeScript interface
2. 再并行派发：
   - backend agent owns `src/api/`
   - frontend agent owns `src/components/`
   - infra agent owns `infra/`
3. 最后串行验证整合

关键规则：

> 共享接口是依赖，必须先于消费它的并行任务完成。

---

## 常见错误

### Over-Fragmentation

过度拆分会把 token 浪费在 context setup 上。

坏例子：12 个 agent 处理只涉及 4 个文件的功能。

修复：合并相关微任务，例如让一个 agent 同时处理 types / interfaces / validation schemas。

### Under-Specification

委派过于模糊会导致 agent 猜 scope。

坏指令：

```markdown
handle the frontend
```

好指令：

```markdown
Create `src/components/Dashboard.tsx` that exports a `Dashboard` component accepting `DashboardProps` with a `data: TimeSeriesPoint[]` prop.
```

有效委派应包含：

- 精确文件
- 创建 / 修改范围
- 预期函数签名
- 输出格式
- 不要动哪些资源

### Resource Conflicts

两个 agent 写同一 `index.ts`，最后一个写入者会覆盖前一个导出。

这类错误可能 build 仍通过，但后续使用 feature 时才暴露。

修复：给 agent 分配文件所有权，而不是函数所有权。

### Context Duplication

CLAUDE.md 太长时，多个 sub-agent 会重复加载大量无关上下文。

如果 CLAUDE.md 400 行，派生 7 个 agent，就可能复制 7 份上下文。

修复：

- CLAUDE.md 只放操作规则
- 项目百科放到按需文档 / Skills
- 让 agent 自己读取必要文件

---

## 典型失败案例

用户 settings 功能被拆成 5 个 agents：

1. database migration
2. API route
3. React form component
4. tests
5. TypeScript types

问题：types agent 和 API agent 都需要 `UserSettings` 的结构，但两者并行运行，没有共享契约。

结果：

- types agent 定义 flat `preferences`
- API agent 期待 nested `preferences.theme / notifications`
- React form 又假设第三种 shape
- 所有 agent 单独完成，但最终 build 出现 14 个 type errors

正确流程：

1. types agent 先串行生成共享接口
2. API / frontend / tests 再并行消费该接口

教训：

> Shared interfaces are dependencies. Dependencies must run before tasks that consume them.

---

## 实践建议

从 7-agent feature pattern 开始，但要遵守：

- 并行只用于独立文件 / 独立领域
- 共享接口先串行
- 实现后再验证
- 小任务合并，避免 over-fragmentation
- research 是最安全的并行入口
- 监控 task completion velocity，持续优化分发策略

---

## 与其他笔记的关系

- [[Agent Fundamentals]]：介绍 Task tool / sub-agents 的基本能力
- [[Sub-Agent Best Practices]]：判断 parallel / sequential / background 的路由规则
- [[Sub-Agent Design]]：设计 specialist sub-agents 的方法
- [[Async Workflows]]：将 sub-agent 放到后台继续工作

## 关键 takeaways

- Task distribution 的本质是并发工程：边界、依赖、资源所有权比 agent 数量更重要。
- 并行前先识别共享契约；共享接口必须先串行生成。
- 研究任务天然适合并行，写文件任务必须严格分配所有权。
- Validation chain 应该在实现完成后串行执行，避免验证中间态。
- CLAUDE.md 中的分发规则会改变 Claude Code 默认的串行倾向。
