---
title: Claude Code Agent Teams
tags:
  - agents
  - orchestration
  - claude-code
  - multi-agent
  - agent-teams
source: https://claudefa.st/blog/guide/agents/agent-teams
date: 2026-04-27
---

# Claude Code Agent Teams

> [!tip] 核心概念
> Agent Teams 是 Claude Code 的实验性功能，允许多个 Claude 会话并行协作。一个 session 担任 Team Lead，其余作为 Teammates，通过共享任务列表和 Mailbox 直接通信。

相关笔记：[[Agent-Fundamentals]] | [[Team-Orchestration]] | [[Sub-Agent-Best-Practices]]

---

## 启用方式

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

或写入 `settings.json` 永久生效：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

---

## 与 Subagents 的核心区别

| 特性 | Subagents | Agent Teams |
|------|-----------|-------------|
| **通信方式** | 只能向主 agent 汇报结果 | Teammates 之间可直接互发消息 |
| **协调方式** | 主 agent 统一管理 | 共享任务列表，自主协调 |
| **适用场景** | 专注任务，只关心结果 | 需要讨论、相互挑战、跨层协作 |
| **Token 成本** | 较低（结果汇总回主上下文） | 较高（每个 teammate 是独立实例） |
| **通信拓扑** | Hub-and-spoke（全经主 agent） | Mesh（任意 teammate 互通） |

> [!note] 关键判断
> **决策问题**：你的 workers 需要互相通信吗？需要 → Agent Teams；不需要 → Subagents。

---

## 架构组成

| 组件 | 作用 |
|------|------|
| **Team Lead** | 主 Claude Code session，创建团队、分配任务、综合结果 |
| **Teammates** | 独立 Claude Code 实例，各自拥有独立上下文窗口 |
| **Shared Task List** | 所有 agent 可见的中央任务队列，支持状态和依赖 |
| **Mailbox** | agent 间的消息系统 |

本地存储：
- Team config：`~/.claude/teams/{team-name}/config.json`
- Task list：`~/.claude/tasks/{team-name}/`

每个 teammate 启动时加载相同的项目上下文（CLAUDE.md、MCP servers、skills），但不继承 Lead 的对话历史。

---

## 使用步骤

### 1. 描述任务与团队结构

```
Create an agent team to review our authentication system. Spawn three teammates:
- Security reviewer: audit for vulnerabilities, check token handling
- Performance analyst: profile response times, identify bottlenecks
- Test coverage checker: verify edge cases, find untested paths
Have them share findings and coordinate through the task list.
```

### 2. 监控与操作

| 快捷键 | 操作 |
|--------|------|
| `Shift+Up/Down` | 选择 teammate |
| `Ctrl+T` | 查看任务列表 |
| `Enter` | 进入某个 session |
| `Escape` | 中断 |

### 3. 清理

```
Ask all teammates to shut down, then clean up the team.
```

> [!warning] 注意顺序
> 必须先让所有 teammates shut down，Lead 才能完成清理。

---

## 适用场景

**强烈推荐**：
- 研究 & 评审：多 teammate 分头调查不同方面，汇总后互相挑战
- 独立模块开发：各 teammate 拥有独立文件范围，无冲突
- Debug 竞争假说：并行测试不同理论，互相证伪
- 跨层协调：前端/后端/测试各自归属不同 teammate
- 架构辩论：多 teammate 争论不同立场，收敛到最优方案

**不适合**：
- 顺序任务、同文件编辑、强依赖工作 → 用单 session 或 subagents
- 独立并行但无需协调的变更 → 用 `/batch`

---

## 大规模协调的三个模式

1. **标准化 spawn prompt 模板**：为常见团队配置（review/implementation/research）准备可复用模板，定义角色、文件边界、成功标准。
2. **权限预设**：在 spawn 前预先批准常见操作，避免大量权限提示拖慢团队。
3. **CLAUDE.md 作为共享运行时上下文**：清晰的模块边界、验证命令、操作上下文，可大幅减少每个 teammate 的探索成本。

---

## 多 Agent 能力谱系

| 方式 | 通信 | 最适合 |
|------|------|--------|
| 单 session | N/A | 顺序、专注任务 |
| Subagents (Task tool) | 仅结果回传 | 并行专注工作 |
| Builder-Validator pairs | 结构化任务交接 | 带质量门控的实现 |
| **Agent Teams** | 全 mesh 直接消息 | 协作探索 |

推荐组合：用 Agent Teams 做协作探索阶段，切换到 [[Team-Orchestration]] 的 builder-validator 模式做需要质量门控的实现阶段。

---

## 延伸阅读

- [Advanced Controls](https://claudefa.st/blog/guide/agents/agent-teams-controls) — 显示模式、delegate 模式、plan 审批、hooks、token 成本管理
- [Use Cases & Prompt Templates](https://claudefa.st/blog/guide/agents/agent-teams-use-cases) — 10+ 场景及即用 prompt
- [Best Practices & Troubleshooting](https://claudefa.st/blog/guide/agents/agent-teams-best-practices) — 实战经验、当前限制、已知修复
- [End-to-End Workflow](https://claudefa.st/blog/guide/agents/agent-teams-workflow) — 从 brain dump 到生产的完整 7 步流程
