---
title: Claude Code Agent Teams Advanced Controls
tags:
  - agents
  - claude-code
  - agent-teams
  - configuration
source: https://claudefa.st/blog/guide/agents/agent-teams-controls
date: 2026-04-27
---

# Claude Code Agent Teams: Advanced Controls

> [!tip] 快速入门
> 启动团队后按 `Shift+Tab` 进入 delegate mode，Lead 立即停止写代码，专注协调。

前置阅读：[[Agent-Teams]]

---

## 显示模式

### In-Process Mode（默认）

所有 teammate 在主终端内运行，通过快捷键切换：

| 快捷键 | 操作 |
|--------|------|
| `Shift+Up/Down` | 选择 teammate |
| `Enter` | 查看 teammate 完整会话 |
| `Escape` | 中断 teammate 当前操作 |
| `Ctrl+T` | 切换任务列表 |
| `Shift+Tab` | 循环切换模式（含 delegate） |

### Split Pane Mode

每个 teammate 独占一个终端面板，同时可见所有输出。需要 tmux 或 iTerm2（macOS 推荐 `tmux -CC` + iTerm2）。

> [!warning] 兼容性限制
> Split pane **不支持** VS Code 集成终端、Windows Terminal、Ghostty。

`settings.json` 配置：

```json
{
  "teammateMode": "in-process"
}
```

可选值：`"auto"`（默认）、`"tmux"`、`"in-process"`。也可按会话覆盖：

```bash
claude --teammate-mode in-process
```

---

## Delegate Mode：让 Lead 专注协调

无 delegate mode 时，Lead 有时会自己去实现任务而非分配给 teammate。

**启用**：启动团队后按 `Shift+Tab`。

Delegate mode 下 Lead 只能使用协调类工具（spawn、消息、shutdown、task 管理），无法直接修改代码。

> [!note] 使用时机
> 4+ teammate 的会话中尤其重要；也可直接说 "Wait for your teammates to complete their tasks before proceeding"，但 delegate mode 是系统级约束，更可靠。

---

## Plan Approval：执行前审查

对复杂或高风险任务，让 teammate 先出计划，经 Lead 审批后再动手：

```
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

**工作流**：
1. Teammate 进入只读 plan mode
2. Teammate 提交计划给 Lead
3. Lead 审批或拒绝（可附反馈）
4. 拒绝则 teammate 修改后重提，通过后开始实现

通过初始 prompt 影响 Lead 的审批标准，例如："only approve plans that include test coverage"。

> [!important] 适用场景
> 修改共享基础设施、数据库 schema、或难以回滚的变更时，plan mode 的成本远低于事后修复。

---

## Hooks 质量门控

两个专为团队设计的 hook：

**`TeammateIdle`**：teammate 即将空闲时触发。退出码 2 → 发送反馈并保持 teammate 工作（如自动分配复审任务）。

**`TaskCompleted`**：任务被标记完成时触发。退出码 2 → 阻止完成并发送反馈（如强制测试通过才能关闭任务）。

详见 [Hooks Guide](https://claudefa.st/blog/tools/hooks/hooks-guide)。

---

## 直接与 Teammate 通信

- **In-process**：`Shift+Up/Down` 选中 teammate → 直接输入消息；`Enter` 查看完整会话；`Escape` 中断
- **Split-pane**：点击对应面板直接交互

适合需要给特定 teammate 补充上下文或纠偏时，比通过 Lead 路由更快。

---

## 任务分配与认领

- 任务三态：**pending** → **in progress** → **completed**
- Lead 可显式分配，也可让 teammate 自主认领
- 文件锁防止两个 teammate 同时抢同一任务
- 依赖关系自动解锁：前置任务完成后，被阻塞任务立即可认领

---

## 指定 Teammate 数量与模型

```
Create a team with 4 teammates to refactor these modules in parallel.
Use Sonnet for each teammate.
```

**混合模型策略**：Lead 用 Opus（强推理，任务拆解/审批），Teammates 用 Sonnet（聚焦实现，成本低）。

---

## Token 成本

粗估：3 个 teammate 运行 30 分钟 ≈ 单 session 顺序执行的 3-4 倍 token。Plan mode 阶段约 7 倍。

**成本主要来源**：
- 每个 teammate 独立加载项目上下文
- agent 间消息在发送方和接收方上下文中均计费
- Broadcast 成本 × teammate 数量

**控制成本**：
- Teammate 用 Sonnet，Lead 用 Opus
- 优先 direct message，避免 broadcast
- 团队规模以够用为准（3 人通常优于 6 人）

---

## 为 Agent Teams 优化 CLAUDE.md

每个 teammate 启动时用全新上下文加载 CLAUDE.md，Lead 对话历史不传递，因此 CLAUDE.md 质量直接影响团队效率。

### 规则 1：描述模块边界

```markdown
## Independent Modules

| Module  | Directory | Notes                    |
|---------|-----------|--------------------------|
| API     | api/      | Each file is independent |
| CLI     | src/      | Core logic               |
| Website | docs/js/  | Static content           |

**Shared files (coordinate before editing):**
- package.json
- tsconfig.json
```

Claude 读取此结构后自动为每个 teammate 分配明确的文件列表，消除冲突。

### 规则 2：保持项目上下文简短实用

```markdown
## Quick Context

- **Stack**: Node.js CLI + Static site + Vercel Serverless
- **Entry point**: src/index.js
- **Tests**: Jest (`npm test`)
- **Database**: Neon
```

不让 teammate 向 Lead 询问"这是什么框架"——每次询问都消耗双方 token。

### 规则 3：定义"完成"的标准

```markdown
## Verification

- `npm test`
- `npm run lint`
- `npm run build`
```

有了明确的验收标准，teammate 可自主验证并如实汇报，无需 Lead 介入。

---

## 延伸阅读

- [[Agent-Teams]] — Agent Teams 总览与基础概念
- [[Team-Orchestration]] — Builder-Validator 模式（DIY 方案）
- [Use Cases & Prompt Templates](https://claudefa.st/blog/guide/agents/agent-teams-use-cases)
- [Best Practices & Troubleshooting](https://claudefa.st/blog/guide/agents/agent-teams-best-practices)
- [End-to-End Workflow](https://claudefa.st/blog/guide/agents/agent-teams-workflow)
- [CLAUDE.md Mastery](https://claudefa.st/blog/guide/mechanics/claude-md-mastery)
