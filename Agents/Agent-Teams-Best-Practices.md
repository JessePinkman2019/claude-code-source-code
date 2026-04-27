---
title: Claude Code Agent Teams Best Practices and Troubleshooting
tags:
  - agents
  - claude-code
  - agent-teams
  - troubleshooting
source: https://claudefa.st/blog/guide/agents/agent-teams-best-practices
date: 2026-04-27
---

# Claude Code Agent Teams: Best Practices & Troubleshooting

> [!tip] 快速修复
> 两个改动消除最常见失败：开启 delegate mode（Shift+Tab）+ 在 spawn prompt 中给每个 teammate 明确的文件边界。

前置阅读：[[Agent-Teams]] | [[Agent-Teams-Controls]]

---

## 核心最佳实践

### 给 Teammate 足够的上下文

Teammate 启动时自动加载项目上下文（CLAUDE.md、MCP servers、skills），但**不继承** Lead 的对话历史。spawn prompt 中需包含具体文件、验收标准和约束条件。

**弱 prompt**：`review the auth module`

**强 prompt**：
```
Review the authentication module at src/auth/ for security vulnerabilities.
Focus on token handling, session management, and input validation.
The app uses JWT tokens stored in httpOnly cookies.
Report any issues with severity ratings.
```

---

### 任务粒度：5-6 个任务/Teammate

- 太小 → 协调开销超过收益
- 太大 → 无中间检查点，出错代价高
- 目标：每个任务产出清晰可交付物（一个函数、一个测试文件、一份报告）

---

### 避免文件冲突

> [!warning] 最重要规则
> 两个 teammate 编辑同一文件 = 覆盖。目录级所有权是实现类任务的硬性要求。

无法避免共享文件时，在 CLAUDE.md 中标注 "coordinate before editing"，由 Lead 通过任务排序管理访问。

---

### 默认开启 Delegate Mode

`Shift+Tab` 启动后立即切换。Lead 不能动代码，专注协调。4+ teammate 的会话效果尤为显著。

---

### 从调研和审查起步

研究类任务（PR review、bug 调查、模块审计）：
- 文件边界清晰，无写入冲突
- 失败代价低（最多损失 token，不影响代码）
- 适合熟悉团队协作机制

熟悉后再转向实现类任务。

---

### 定期监控与干预

`Ctrl+T` 查看进度，及时重定向偏离方向的 teammate。Agent Teams 是**监督式工作流**，Lead 协调，你做战略决策：何时重定向、何时 spawn 替换者、何时关闭卡住的 teammate。

---

### 保持团队小规模

**实际最佳点：3-5 个 teammate。**

超过 5 人：
- Lead 上下文填充更快
- 每次 broadcast 成本 × teammate 数
- 协调噪音超过并行收益

任务确实需要更多并行时：拆分为多个阶段（3 人一波 → 清理 → 再 3 人一波）。

---

## Plan Mode 行为特殊说明

> [!important] 两个非直觉行为
> 1. **Plan mode 每轮都评估**：teammate 在 plan mode 下整个生命周期保持只读，不是只在第一轮
> 2. **模式在生命周期内固定**：无法将已 spawn 的 teammate 从 plan mode 切出

需要"先计划后实现"时，使用 [[Agent-Teams-Controls]] 中的 **plan approval** 功能，而非直接用 plan mode。

---

## 自报告模式

CLAUDE.md 中有明确验收标准时，teammate 会自主验证并精确汇报，无需 Lead 介入：

> "Removed 27 console.log across 3 files. Kept all 12 console.error and 2 console.warn in component-page.js. Verified zero console.log remaining in my assigned files."

清晰的规则 → 清晰的报告。详见 [[Agent-Teams-Controls]] CLAUDE.md 优化部分。

---

## Troubleshooting 速查表

| 问题 | 解决方案 |
|------|----------|
| Teammate 未出现 | `Shift+Down` 循环查找；确认任务复杂度值得建团队；split-pane 模式检查 tmux/iTerm2 配置 |
| 权限提示过多 | spawn 前在 permission settings 预批准常用操作，所有 teammate 继承 Lead 权限 |
| Teammate 遇错停止 | `Shift+Up/Down` 选中 teammate 直接补充指令；或 spawn 替换者继续工作 |
| Lead 提前关闭 | 直接告诉 Lead："Wait for your teammates to complete their tasks before proceeding" |
| 孤立的 tmux session | `tmux ls` 列出 → `tmux kill-session -t <name>` 清理 |
| Teammate 互相覆盖文件 | spawn prompt 中定义明确文件边界，使用目录级所有权 |
| 任务状态卡住 | `Ctrl+T` 手动检查，提示 teammate 更新状态 |
| Bedrock/Vertex/Foundry 失败 | 升级到 v2.1.45+（修复模型标识符和 API provider 环境变量） |
| 切换 agent teams 设置崩溃 | 升级到 v2.1.34+ |
| tmux teammate 无法收发消息 | 升级到 v2.1.33+ |

---

## 当前已知限制

| 限制 | 说明 |
|------|------|
| 无法恢复 session | `/resume` 或 `/rewind` 后 in-process teammate 不会恢复，Lead 需重新 spawn |
| 任务状态可能滞后 | Teammate 有时忘记标记完成，手动检查卡住的依赖 |
| 关闭较慢 | Teammate 完成当前请求/工具调用后才关闭 |
| 每 session 只有一个团队 | 启动新团队前需清理当前团队 |
| 不支持嵌套团队 | Teammate 不能 spawn 自己的团队 |
| Lead 固定 | 无法提升 teammate 或转让 Lead 身份 |
| 权限在 spawn 时固定 | spawn 后可更改单个 teammate 模式，但无法在 spawn 时指定 |
| Split pane 依赖 tmux/iTerm2 | 不支持 VS Code 集成终端、Windows Terminal、Ghostty |

---

## 版本修复记录

**v2.1.33**：
- 新增 `TeammateIdle` 和 `TaskCompleted` hooks
- 新增 `Task(agent_type)` spawn 限制
- 新增 `memory` 持久字段（user/project/local scope）
- 修复 tmux teammate 消息收发
- 修复 team 上下文中的 plan mode 警告

**v2.1.34**：修复 agent teams 设置切换时的崩溃

**v2.1.41**：修复 Bedrock/Vertex/Foundry 的错误模型标识符

**v2.1.45**：修复 tmux session 的 API provider 环境变量传递；修复 subagent 调用的 skill 在 compaction 后错误出现在主 session

---

## 延伸阅读

- [[Agent-Teams]] — 总览与基础概念
- [[Agent-Teams-Controls]] — 控制与配置
- [[Agent-Teams-Use-Cases]] — 10+ 场景 prompt 模板
- [[Agent-Teams-Workflow]] — 端到端生产构建工作流
