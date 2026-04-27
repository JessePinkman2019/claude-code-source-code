---
tags:
  - agents
  - claude-code
  - async
  - sub-agents
  - parallel
source: https://claudefa.st/blog/guide/agents/async-workflows
date: 2026-04-21
---

# Async Workflows

> Run Claude Code sub-agents in the background while you keep working. True parallel AI development that eliminates blocking and boosts throughput.

## 核心问题

Claude Code 派生 sub-agent 时，整个会话会阻塞——你必须等待 sub-agent 完成才能继续。

**快速上手**：sub-agent 启动后按 `Ctrl+B` 移至后台：

```
You: Research authentication best practices for our Next.js app
Claude: I'll spawn a sub-agent to research this...
[Sub-agent starts]
You: [Press Ctrl+B]
You: While that runs, let's work on the database schema...
```

会话继续，sub-agent 独立工作，完成后自动浮现结果。

## 后台 Agent 工作流

1. **发起请求**：让 Claude 处理适合 sub-agent 的任务
2. **Claude 派生**：主 agent 创建 sub-agent
3. **移至后台**：按 `Ctrl+B`
4. **继续工作**：与主 agent 继续聊其他任务
5. **自动恢复**：sub-agent 完成后，主 agent 自动接收结果

随时用 `/tasks` 查看运行中的 agent 状态、token 用量和进度。

## 适合后台执行的场景

> [!tip] 适合后台
> - 需要网络搜索的研究任务
> - 大型代码库分析
> - 文档生成
> - 安全审计与漏洞扫描
> - 性能分析报告

> [!warning] 保持前台
> - 需要你立即输入的任务
> - 需要审查的文件修改
> - 与当前工作有顺序依赖的任务

## 后台 Bash 命令

同样的模式适用于长时间运行的 shell 命令（`npm install`、`docker build`、`ffmpeg` 等）：

```
Claude: Running npm install...
[Command starts]
You: [Press Ctrl+B]
You: While that installs, can you review the API routes?
```

用 `/tasks` 监控，与 agent 一致。

## `--agent` Flag：调试 Sub-Agent

新 CLI flag 让你以任意 sub-agent 身份运行 Claude Code：

```bash
claude --agent plan
```

以 planning agent 的配置启动 Claude Code，可测试其行为、验证指令和工具访问权限。

**使用场景：**

- 上线前调试自定义 sub-agent
- 测试 agent 指令和工具访问
- 手动运行专家 agent 处理一次性任务
- 了解内置 agent 能力

适用于内置 agent（`plan`、`explore` 等）和 `.claude/agents/` 中的自定义 agent。

## 同期发布的其他功能

| 功能 | 说明 |
| --- | --- |
| **Instant compaction** | `/compact` 立即执行，加载摘要到新上下文 |
| **Session memory** | 每个会话维护结构化摘要：状态、已完成工作、讨论点、工作日志 |
| **Stats dashboard** | `/stats` 查看用量模式、模型偏好、token 计数；`Ctrl+S` 复制分享 |
| **Session naming** | `/rename` 命名会话，`claude --resume session-name` 按名恢复 |
| **MCP quick toggle** | 无需编辑配置文件即可启用/禁用 MCP server |
| **Slack integration** | 在 Slack 频道中 @Claude 直接委派任务 |

## 常见问题

**Agent 无法移至后台**：在 agent 活跃运行时按 `Ctrl+B`，而非完成后。

**找不到 agent**：用 `/tasks` 查看所有后台进程及其 ID。

**Agent 完成但没有结果**：`AgentOutputTool` 会自动浮现结果；若遗漏，检查 `/tasks` 中已完成 agent 的输出。

## 相关笔记

- [[Agent Fundamentals]]
- [[Sub-Agent Design Patterns]]
- [[Agent Patterns]]
