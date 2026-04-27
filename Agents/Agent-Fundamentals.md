---
tags:
  - agents
  - claude-code
  - sub-agents
  - slash-commands
source: https://claudefa.st/blog/guide/agents/agent-fundamentals
date: 2026-04-21
---

# Agent Fundamentals

> Master agent engineering in Claude Code. Design and deploy autonomous agents that handle complex multi-step tasks independently and reliably.

## 核心思路

**问题**：复杂项目需要多种视角——安全审查、性能分析、文档编写。频繁切换心智模型耗时又分散注意力。

**快速上手**：立即派生一个 sub-agent 处理并行任务：

```
Use a sub-agent to review the authentication module for security vulnerabilities while I continue working on the API endpoints.
```

Claude Code 启动隔离的 sub-agent 独立工作，完成后将结果汇报回主会话。

## 五种 Agent 方式对比

| 方式 | 最适合 | 持久性 |
| --- | --- | --- |
| **Task Tool（Sub-agents）** | 并行执行、隔离工作 | 仅限当前会话 |
| **`.claude/agents/` 定义** | 持久化专家 sub-agent | 永久 |
| **Custom Slash Commands** | 可复用工作流、团队共享 | 永久 |
| **CLAUDE.md Personas** | 项目级行为规则 | 永久 |
| **Perspective Prompting** | 快速切换上下文 | 单次请求 |

## Sub-Agents：内置并行执行

Task tool 在会话内派生迷你 Claude Code 实例。每个 sub-agent 拥有独立的上下文窗口，独立工作后将结果返回给协调者。

**并行启动多个 sub-agent：**

```
Complete these tasks using 3 sub-agents in parallel:
1. Review src/auth/ for security vulnerabilities
2. Analyze src/api/ for performance bottlenecks
3. Check src/utils/ for code duplication
```

**Sub-agents 的优势：**

- 隔离上下文，避免任务间污染
- 并行执行加速多文件分析
- sub-agent 失败不会崩溃主会话
- 后台执行（`Ctrl+B`）让你可以继续工作，结果自动浮现

**控制 sub-agent 模型**：设置 `CLAUDE_CODE_SUBAGENT_MODEL` 环境变量来选择模型，可用于成本优化：

```bash
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-5-20250929"
```

## `.claude/agents/` 持久化 Agent 定义

在 `agents/` 目录中以 YAML frontmatter 的 Markdown 文件定义自定义 sub-agent。与 slash commands 不同——slash commands 是手动调用的提示词，agent 定义是让协调者在需要时自动派生的持久 sub-agent。

**两种作用域：**

- **项目 agents**（`.claude/agents/`）——特定于仓库，可通过 git 与团队共享
- **用户 agents**（`~/.claude/agents/`）——跨项目可用，个人专属

### 用权限规则限制 Sub-Agent 访问

在 `settings.json` 的 `deny` 数组中添加 `Task(AgentName)` 规则：

```json
{
  "permissions": {
    "deny": ["Task(Explore)"]
  }
}
```

内置 agent 名称：`Explore`、`Plan`、`Verify`。也可在启动时禁用：

```bash
claude --disallowedTools "Task(Plan)" "Task(Verify)"
```

## Custom Slash Commands：可复用专家

在 `.claude/commands/` 中添加 Markdown 文件创建永久可复用命令：

```markdown
<!-- .claude/commands/security-review.md -->
description: "Security-focused code review"
allowed-tools: ["Read", "Grep", "Bash"]

Review the specified code for security vulnerabilities:
1. Check for SQL injection risks
2. Identify authentication bypasses
3. Flag hardcoded secrets
4. Report findings with severity ratings

Target: $ARGUMENTS
```

运行 `/project:security-review src/auth/` 即可调用专家。

**命令位置：**

- `.claude/commands/`——项目专属，可通过 git 共享
- `~/.claude/commands/`——个人专属，随处可用

## CLAUDE.md：持久化 Agent 行为

`CLAUDE.md` 为项目中每次交互塑造 Claude 的行为，创造无需显式调用的 agent 级一致性。

## Perspective Prompting：快速切换上下文

一次性分析时，让 Claude 采用特定视角：

```
Analyze this authentication flow from three perspectives:
1. Security engineer: What vulnerabilities exist?
2. Performance engineer: What bottlenecks could occur at scale?
3. Junior developer: What parts are confusing or poorly documented?
```

无需任何配置，立即获得专业分析。

## 如何选择

- **用 sub-agents**：需要并行执行或多任务隔离上下文时
- **用 `.claude/agents/`**：需要持久化、有名称的专家 agent，协调者可自动调用时
- **用 slash commands**：跨会话重复同一工作流或团队共享时
- **用 CLAUDE.md**：希望所有交互自动应用一致行为时
- **用 perspective prompting**：需要快速一次性专业分析时

> [!tip] 成熟设置通常组合使用
> agent 定义负责专家角色、slash commands 负责可重复工作流、CLAUDE.md 负责项目标准、sub-agents 负责并行执行。

## 相关笔记

- [[Sub-Agent Design Patterns]]
- [[Custom Agents]]
- [[Agent Patterns]]
