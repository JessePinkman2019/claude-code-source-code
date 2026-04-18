---
title: "Multi-Agent 协作机制（基于 v2.1.88 源码）"
aliases:
  - "learning-multi-agent"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "`/simplify` 与 `/batch`：内置多 Agent 工作流命令"
  - "Ultraplan：Claude Code 云端规划机制深度解析"
  - "自主 Agent 循环：睡觉时也在发布功能"
---

# Multi-Agent 协作机制（基于 v2.1.88 源码）

> 所有分析均对应具体源码文件和行号。

---

## 1. 整体架构：一次 Agent 调用发生了什么

```
用户 prompt
    ↓
主线程 query loop（main REPL）
    ↓
模型决定调用 AgentTool
    ↓
AgentTool.call()                    ← src/tools/AgentTool/AgentTool.tsx:239
    ├── 选择 AgentDefinition
    ├── 构建 system prompt
    ├── 组装工具池（workerTools）
    ├── 同步 or 异步？
    │     ├── 同步：阻塞主线程，等待结果
    │     └── 异步：fire-and-forget，立即返回 async_launched
    └── runAgent()                  ← src/tools/AgentTool/runAgent.ts:248
            ↓
        query() loop（子 agent 独立运行）
            ↓
        结果通过 yield 返回给父 agent
```

---

## 2. Agent 的三种运行模式

### 2.1 同步（Foreground）

`AgentTool.tsx:765`：主线程阻塞，等待子 agent 完成后才继续。

- 适用：需要子 agent 结果才能继续的场景
- 子 agent 共享父 agent 的 `abortController`（`runAgent.ts:527`）
- 子 agent 可以弹出权限确认对话框（`runAgent.ts:440-445`）

### 2.2 异步（Background）

`AgentTool.tsx:686`：注册后立即返回 `{ status: 'async_launched', agentId, outputFile }`，子 agent 在后台独立运行。

触发条件（`AgentTool.tsx:567`）：
```ts
const shouldRunAsync = (
  run_in_background === true ||
  selectedAgent.background === true ||
  isCoordinator ||
  forceAsync ||
  assistantForceAsync ||
  proactiveModule?.isProactiveActive()
) && !isBackgroundTasksDisabled
```

- 子 agent 有**独立的** `AbortController`（`runAgent.ts:524-528`），不受父 agent ESC 影响
- 异步 agent **不能**弹出权限对话框（`shouldAvoidPermissionPrompts: true`，`runAgent.ts:446-450`）
- 完成后通过 `<task-notification>` 通知父 agent

### 2.3 Fork（实验性）

`forkSubagent.ts:32-39`：由 feature gate `FORK_SUBAGENT` 控制。

- 省略 `subagent_type` 时触发
- 子 agent **继承父 agent 的完整对话上下文**（`AgentTool.tsx:512`）
- 使用父 agent 的 system prompt 字节（`AgentTool.tsx:496-511`），保证 prompt cache 命中
- 所有 fork 强制异步（`AgentTool.tsx:557`）
- 禁止递归 fork（`AgentTool.tsx:332-334`）

---

## 3. Agent 定义：frontmatter 字段

自定义 agent 放在 `.claude/agents/` 目录下，用 `.md` 文件定义。

`loadAgentsDir.ts:73-98` 定义了完整的 schema：

```markdown
---
description: 这个 agent 的用途说明（whenToUse 字段）
tools:
  - Read
  - Bash
  - Glob
disallowedTools:
  - Write
model: sonnet          # 或 opus / haiku / inherit
effort: medium         # low / medium / high / max / 整数
permissionMode: plan   # default / plan / acceptEdits / bypassPermissions / bubble
maxTurns: 50
skills:
  - my-skill
mcpServers:
  - slack              # 引用已有 MCP server
  - my-server:         # 内联定义
      command: npx
      args: [my-mcp-server]
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: echo "before bash"
background: false      # true = 始终异步运行
isolation: worktree    # 在独立 git worktree 中运行
memory: project        # 持久化记忆：user / project / local
---

你的 agent system prompt 写在这里。
```

### Agent 优先级（同名时后者覆盖前者）

`loadAgentsDir.ts:203-220`：

```
built-in < plugin < userSettings < projectSettings < flagSettings < policySettings（最高）
```

---

## 4. 工具池：子 agent 能用哪些工具

### 4.1 所有 agent 都禁用的工具

`src/constants/tools.ts:36-46`（`ALL_AGENT_DISALLOWED_TOOLS`）：

```
TaskOutputTool, ExitPlanModeTool, EnterPlanModeTool,
AgentTool（防止递归，ant 内部用户除外）,
AskUserQuestionTool
```

### 4.2 异步 agent 只能用的工具白名单

`src/constants/tools.ts:55-71`（`ASYNC_AGENT_ALLOWED_TOOLS`）：

```
Read, WebSearch, TodoWrite, Grep, WebFetch, Glob,
Bash/Shell, Edit, Write, NotebookEdit, Skill,
EnterWorktree, ExitWorktree
```

**注意**：异步 agent 不能用 `AgentTool`（不能再嵌套 spawn），不能用 `AskUserQuestion`（无法弹 UI）。

### 4.3 in-process teammate 额外允许的工具

`src/constants/tools.ts:77-88`（`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`）：

```
TaskCreate, TaskGet, TaskList, TaskUpdate, SendMessage,
CronCreate, CronDelete, CronList（需要 AGENT_TRIGGERS feature）
```

### 4.4 Coordinator 模式只允许的工具

`src/constants/tools.ts:107-112`（`COORDINATOR_MODE_ALLOWED_TOOLS`）：

```
AgentTool, TaskStopTool, SendMessage, SyntheticOutputTool
```

---

## 5. 上下文隔离：子 agent 看到什么

### 5.1 system prompt

- 普通子 agent：调用 `selectedAgent.getSystemPrompt()`，独立构建（`AgentTool.tsx:518`）
- Fork 子 agent：继承父 agent 的 system prompt 字节（`AgentTool.tsx:496-511`）

### 5.2 CLAUDE.md

`runAgent.ts:390-398`：

```ts
const shouldOmitClaudeMd =
  agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)
```

**Explore 和 Plan agent 默认不注入 CLAUDE.md**（`exploreAgent.ts:81`、`planAgent.ts:90`），节省 token。

### 5.3 git status

`runAgent.ts:403-410`：Explore 和 Plan agent 也不注入 git status（同样节省 token）。

### 5.4 对话历史

- 普通子 agent：从空白开始，只有 `promptMessages`（`runAgent.ts:373`）
- Fork 子 agent：继承父 agent 的完整对话历史（`forkContextMessages`，`runAgent.ts:370-373`）

### 5.5 文件读取缓存（readFileState）

`runAgent.ts:375-378`：

- Fork 子 agent：**克隆**父 agent 的文件缓存（`cloneFileStateCache`）
- 普通子 agent：全新的空缓存

---

## 6. 权限模式（permissionMode）

`runAgent.ts:415-435`：子 agent 可以有自己的权限模式，但父 agent 的 `bypassPermissions` 和 `acceptEdits` 模式**不会被覆盖**。

| 模式 | 含义 |
|------|------|
| `default` | 每次操作都询问用户 |
| `plan` | 只能读，不能写，需要 ExitPlanMode 才能执行 |
| `acceptEdits` | 自动接受文件编辑，其他操作询问 |
| `bypassPermissions` | 跳过所有权限检查 |
| `bubble` | 权限提示冒泡到父 agent 的终端 |

---

## 7. 并行执行

`prompt.ts:271`（注入给模型的指令）：

> "If the user specifies that they want you to run agents 'in parallel', you MUST send a single message with multiple Agent tool use content blocks."

**机制**：在同一个 assistant message 里包含多个 `tool_use` 块，Claude Code 会并发执行它们。

---

## 8. Agent 间通信：SendMessage

`SendMessageTool.ts` 实现了 agent 间的消息传递。

### 8.1 给后台 agent 发消息

`AgentTool.tsx:700-712`：异步 agent 启动时，如果指定了 `name`，会注册到 `agentNameRegistry`：

```ts
rootSetAppState(prev => {
  const next = new Map(prev.agentNameRegistry)
  next.set(name, asAgentId(asyncAgentId))
  return { ...prev, agentNameRegistry: next }
})
```

之后可以用 `SendMessage({ to: 'agent-name', message: '...' })` 向它发消息。

### 8.2 消息路由

`SendMessageTool.ts` 支持多种收件人格式：
- `teammate-name`：发给指定 teammate
- `*`：广播给所有 teammate
- `uds:<socket-path>`：本地 Unix socket（UDS_INBOX feature）
- `bridge:<session-id>`：Remote Control peer

---

## 9. Worktree 隔离

`AgentTool.tsx:590-593`：

```ts
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}
```

- 在 `.claude/worktrees/agent-<id>/` 创建独立的 git worktree
- agent 完成后，如果没有改动则**自动删除** worktree（`AgentTool.tsx:666-670`）
- 有改动则保留，并在结果中返回 `worktreePath` 和 `worktreeBranch`

---

## 10. 内置 Agent 一览

| Agent | 模型 | 工具 | omitClaudeMd | 用途 |
|-------|------|------|--------------|------|
| `general-purpose` | 默认子 agent 模型 | `*` | 否 | 通用任务 |
| `Explore` | haiku（外部用户）/ inherit（ant） | 只读工具 | **是** | 代码探索 |
| `Plan` | inherit | 只读工具 | **是** | 实现规划 |
| `claude-code-guide` | — | Glob/Grep/Read/WebFetch/WebSearch | 否 | Claude Code 使用指南 |
| `verification` | — | — | 否 | 验证/审查 |

---

## 11. 自定义 Agent 的加载路径

`loadAgentsDir.ts:193-221`（`getActiveAgentsFromList`）：

```
~/.claude/agents/*.md          → userSettings
./CLAUDE.md 中的 agents 配置   → projectSettings
./.claude/agents/*.md          → projectSettings
/etc/claude-code/agents/*.md   → policySettings（最高优先级）
```

同名 agent 后者覆盖前者，policySettings 最终胜出。

---

## 12. 关键源码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| AgentTool 入口 | `src/tools/AgentTool/AgentTool.tsx` | 239 |
| 同步/异步判断 | `src/tools/AgentTool/AgentTool.tsx` | 567 |
| 异步 agent 注册 | `src/tools/AgentTool/AgentTool.tsx` | 688 |
| name→agentId 注册 | `src/tools/AgentTool/AgentTool.tsx` | 700-712 |
| runAgent 主函数 | `src/tools/AgentTool/runAgent.ts` | 248 |
| CLAUDE.md 是否注入 | `src/tools/AgentTool/runAgent.ts` | 390-398 |
| 权限模式覆盖逻辑 | `src/tools/AgentTool/runAgent.ts` | 415-435 |
| 工具池过滤 | `src/tools/AgentTool/agentToolUtils.ts` | 70-116 |
| 禁用工具常量 | `src/constants/tools.ts` | 36-112 |
| Fork 机制 | `src/tools/AgentTool/forkSubagent.ts` | 32-39 |
| Fork 消息构建 | `src/tools/AgentTool/forkSubagent.ts` | 107-160 |
| Agent 定义 schema | `src/tools/AgentTool/loadAgentsDir.ts` | 73-98 |
| Agent 优先级 | `src/tools/AgentTool/loadAgentsDir.ts` | 193-221 |
| SendMessage 工具 | `src/tools/SendMessageTool/SendMessageTool.ts` | 全文 |
| Agent prompt 模板 | `src/tools/AgentTool/prompt.ts` | 66-287 |

## Related

- [[`/simplify` 与 `/batch`：内置多 Agent 工作流命令]]
- [[Ultraplan：Claude Code 云端规划机制深度解析]]
- [[自主 Agent 循环：睡觉时也在发布功能]]
