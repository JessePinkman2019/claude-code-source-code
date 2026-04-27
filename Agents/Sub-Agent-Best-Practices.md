---
tags:
  - agents
  - claude-code
  - sub-agents
  - orchestration
  - best-practices
source: https://claudefa.st/blog/guide/agents/sub-agent-best-practices
date: 2026-04-21
---

# Sub-Agent Best Practices

> Configure Claude Code to auto-choose parallel, sequential, or background execution patterns for sub-agents based on task dependencies.

## 核心问题

主会话（"central AI"）不断派生 sub-agent，但不知道何时该并行、何时该串行、何时该后台。没有明确的路由规则，它只能猜——而且经常猜错。

**快速上手**：在 CLAUDE.md 中添加路由决策框架：

```
## Sub-Agent Routing Rules

**Parallel dispatch** (ALL conditions must be met):
- 3+ unrelated tasks or independent domains
- No shared state between tasks
- Clear file boundaries with no overlap

**Sequential dispatch** (ANY condition triggers):
- Tasks have dependencies (B needs output from A)
- Shared files or state (merge conflict risk)
- Unclear scope (need to understand before proceeding)

**Background dispatch**:
- Research or analysis tasks (not file modifications)
- Results aren't blocking your current work
```

## 三种执行模式

| 模式 | 使用时机 | 选错的风险 |
| --- | --- | --- |
| **Parallel** | 独立领域、无文件重叠 | 合并冲突、状态不一致 |
| **Sequential** | 存在依赖、共享资源 | 独立工作被串行化，浪费时间 |
| **Background** | 研究任务，不阻塞当前工作 | 结果未检查则丢失 |

> [!warning] 默认行为
> 没有明确规则时，central AI 默认保守的串行执行——安全但缓慢。

## Parallel：基于领域拆分

在 CLAUDE.md 中配置领域并行路由：

```
## Domain Parallel Patterns

When implementing features across domains, spawn parallel agents:
- **Frontend agent**: React components, UI state, forms
- **Backend agent**: API routes, server actions, business logic
- **Database agent**: Schema, migrations, queries

Each agent owns their domain. No file overlap.
```

**关键规则**：并行只在 agent 操作不同文件时有效。Central AI 必须理解领域边界才能正确路由。

## Sequential：依赖链

当前一步的输出是下一步的输入时，必须串行：

| 依赖链 | 原因 |
| --- | --- |
| Schema → API → Frontend | 数据结构必须先于接口存在 |
| Research → Planning → Implementation | 先理解再执行 |
| Implementation → Testing → Security | 先构建，再验证，再审计 |

在 CLAUDE.md 中定义链条，让 central AI 识别依赖模式，避免错误并行化。

## Background：非阻塞研究

配置自动后台执行：

```
## Background Execution Rules

Run in background automatically:
- Web research and documentation lookups
- Codebase exploration and analysis
- Security audits and performance profiling
- Any task where results aren't immediately needed
```

随时用 `/tasks` 查看后台进度，详见 [[Async-Workflows]]。

## 配置 Sub-Agent 行为

### 选择 Sub-Agent 模型

```bash
# 用更轻量的模型运行 sub-agent 节省 token
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-5-20250929"
```

常见模式：主会话用 Opus 处理复杂推理，sub-agent 用 Sonnet 处理聚焦任务——大幅降低成本而不牺牲质量。

### 在 `.claude/agents/` 中定义持久化 Agent

以 Markdown + YAML frontmatter 定义专家 agent，协调者可自动调用。定义在此的 agent 会自动继承项目的 CLAUDE.md 上下文。

- **项目 agents**（`.claude/agents/`）——仓库专属，可 git 共享
- **用户 agents**（`~/.claude/agents/`）——跨项目可用

### 用权限规则限制 Agent

```json
{
  "permissions": {
    "deny": ["Task(Explore)", "Task(Plan)"]
  }
}
```

或启动时禁用：

```bash
claude --disallowedTools "Task(Explore)"
```

适用于成本敏感环境或防止在生产代码库中自主探索。

## 调用质量问题

> [!danger] 最常见的失败原因
> 大多数 sub-agent 失败不是执行失败，而是**调用失败**——指令模糊、上下文不足、交付物不清晰。

**差的调用**：`"Fix authentication"`

**好的调用**：`"Fix OAuth redirect loop where successful login redirects to /login instead of /dashboard. Reference the auth middleware in src/lib/auth.ts."`

Sub-agent 拥有临时上下文窗口，无法回头追问。每次 dispatch 必须提供：
- 完整上下文
- 明确指令
- 相关文件引用
- 清晰的成功标准

## 常见编排错误

- **过度并行**：为简单功能启动 10 个并行 agent 浪费 token，产生协调开销。把相关微任务合并。
- **不足并行**：4 个独立分析串行运行，本可并行。寻找领域独立性。
- **模糊调用**：发送 "implement the feature" 而非具体范围、文件引用和预期输出。

## 相关笔记

- [[Agent Fundamentals]]
- [[Async-Workflows]]
- [[Sub-Agent Design Patterns]]
- [[Agent Patterns]]
