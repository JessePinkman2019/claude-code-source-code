---
title: "Auto Mode：用 AI 分类器替代手动权限审批"
aliases:
  - "auto-mode"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Scaling Managed Agents: Decoupling the Brain from the Hands"
  - "Harness Design for Long-Running Application Development"
  - "Eval Awareness in Claude Opus 4.6's BrowseComp Performance"
---

# Auto Mode：用 AI 分类器替代手动权限审批

## 核心问题

深度重构时，Claude 每执行一步都弹权限确认——跑测试要确认、读文件要确认、写迁移要确认。三十个确认之后，你开始无脑点 approve，完全失去了审阅的意义。

另一个极端是 `--dangerously-skip-permissions`，跳过所有安全检查。在隔离容器里可以，在有 SSH 密钥、环境变量、git 凭证的本地机器上？不现实。

**Auto Mode**（2026 年 3 月 24 日发布）填补了这个空白：用一个后台 AI 分类器在每次工具调用执行前评估风险，安全操作自动放行，危险操作自动拦截，你不需要手动审批任何一步。

---

## 工作原理

Auto Mode 是介于 `default`（全手动审批）和 `bypassPermissions`（零检查）之间的权限模式。

分类器运行在 **Claude Sonnet 4.6** 上，无论主 session 用什么模型。它接收你的用户消息和工具调用作为输入，**Claude 自身的文字和工具结果被剥离**——这个设计是刻意的：工具结果不进入分类器，意味着文件或网页里嵌入的恶意内容无法直接操控分类器决策（prompt injection 防御）。

分类器也会接收你的 CLAUDE.md 内容，所以项目级别的指令会影响放行/拦截决定。与静态 permission rules 只做语法匹配不同，分类器读取散文描述，在上下文中推理每个操作。

### 每次操作的评估顺序

第一个匹配的步骤直接生效：

| 步骤 | 检查 | 结果 |
|------|------|------|
| 1 | 命中你的 allow/deny 规则 | 直接放行或拦截 |
| 2 | 只读操作或编辑工作目录内的文件 | 自动放行 |
| 3 | 其他所有操作 | 送分类器判断 |
| 4 | 分类器拦截 | Claude 尝试替代方案 |

你现有的 permission rules 仍然优先。`Bash(npm test)` 在 allow list 里，不经分类器直接执行；`Bash(rm -rf *)` 在 deny list 里，分类器都看不到就被拦了。

### 分类器检查三类风险

1. **范围越权**：Claude 是否在做超出你请求范围的事？
2. **不受信基础设施**：操作是否指向分类器不认识的系统？
3. **Prompt 注入**：操作是否像是被 Claude 读到的某个文件或网页里的恶意内容驱动的？

---

## 关键细节：宽泛 Allow 规则会被临时移除

进入 Auto Mode 时，Claude Code **会删掉**授予任意代码执行权的宽泛 allow 规则：

被删除：`Bash(*)`、`Bash(python*)`、`Bash(node*)`、任何 `Agent` allow 规则

保留：`Bash(npm test)`、`Bash(git status)` 等窄规则

原因：如果 `Bash(*)` 保留，每条 shell 命令都会绕过分类器直接执行，Auto Mode 形同虚设。退出 Auto Mode 后，被删的规则自动恢复。

---

## 默认拦截 vs 默认放行

### 默认拦截

| 类别 | 示例 |
|------|------|
| 远程代码执行 | `curl \| bash`、克隆仓库里的脚本 |
| 数据外泄 | 向外部端点发送敏感数据 |
| 生产操作 | 部署、数据库迁移、数据库操作 |
| 批量删除 | 云存储批量删除、对已有文件执行 `rm -rf` |
| 权限提升 | 授予 IAM 或仓库权限 |
| 基础设施变更 | 修改共享基础设施 |
| 危险 git 操作 | force push、直接推送到 `main` |

### 默认放行

| 类别 | 示例 |
|------|------|
| 本地文件操作 | 读写编辑工作目录内的文件 |
| 已声明依赖 | 安装 lock file 或 manifest 里已有的包 |
| 凭证使用 | 读取 `.env` 并向对应 API 发送凭证 |
| 只读网络 | HTTP GET、获取文档 |
| 分支操作 | 推送到当前分支或 Claude 创建的分支 |

---

## Sub-agent 的三重检查点

Auto Mode 在三个节点评估子 Agent：

**Spawn 时**：分类器评估委托任务描述，"删除所有匹配此模式的远程分支"在子 Agent 启动前就被拦截。

**执行中**：子 Agent 内部以与父 session 相同的规则运行，子 Agent frontmatter 里定义的 `permissionMode` 被忽略，每次工具调用独立经过分类器。

**返回时**：子 Agent 完成后，分类器审查其完整操作历史。spawn 时看似无害的子 Agent 可能中途被读到的内容污染，若返回检查标记出问题，安全警告会被前置到子 Agent 结果中供主 Agent 决策。

---

## 六种权限模式对比

|  | `default` | `acceptEdits` | `plan` | `auto` | `dontAsk` | `bypassPermissions` |
|--|-----------|--------------|--------|--------|-----------|---------------------|
| **权限提示** | 文件编辑和命令都提示 | 仅命令提示 | 同 default | 无（除非 fallback 触发） | 无（仅预授权操作） | 无 |
| **安全检查** | 你审每个操作 | 你审命令 | 你审命令 | 分类器审命令 | 仅你的规则 | 无 |
| **Token 消耗** | 标准 | 标准 | 标准 | 更高（分类器调用） | 标准 | 标准 |
| **最适合** | 敏感工作 | 代码迭代 | 代码库探索 | 长任务 | CI/CD 管道 | 隔离容器 |
| **风险等级** | 最低 | 低 | 低 | 中 | 取决于规则 | 最高 |

---

## 何时用，何时不用

**适合用 Auto Mode：**
- 长任务中不断的确认提示打断专注状态
- 信任工作大方向，但需要防止意外失误的安全网
- Agent 循环中无人可以逐步审批
- 需要比 `bypassPermissions` 更安全但不在容器里

**不适合用 Auto Mode：**
- 修改生产基础设施（分类器会拦截，并且应该拦截）
- 不熟悉的代码库，需要审阅每个操作
- 需要确定性、可审计的权限控制（用 `dontAsk` + 显式 allow 规则）
- 极度在意 token 消耗

---

## Fallback 机制

**连续拦截 3 次**或**单 session 累计拦截 20 次**，Auto Mode 自动暂停，恢复手动确认模式。

阈值不可配置。

- **CLI**：状态栏出现通知，批准一次被拦截的操作后计数器重置，可继续 Auto Mode
- **非交互模式**（`-p` flag）：session 直接终止

反复触发 fallback 通常意味着：任务本身需要分类器设计来阻止的操作，或分类器缺少你的受信基础设施上下文（让管理员在 managed settings 里配置 `autoMode.environment`）。

---

## 如何启用

**前置条件**：
- Claude Code **Team 计划**（Enterprise 和 API 支持陆续开放）
- **Claude Sonnet 4.6 或 Opus 4.6**（不支持 Haiku、claude-3 系列、Bedrock/Vertex 第三方提供商）
- **管理员先在** [Claude Code admin settings](https://claude.ai/admin-settings/claude-code) **启用**，用户才能打开

**CLI**：`Shift+Tab` 循环切换模式：`default` → `acceptEdits` → `plan` → `auto`

**直接启动 Auto Mode**：
```bash
claude --auto-mode
```

**非交互模式**：
```bash
claude -p "your task" --auto-mode
```

---

## 纵深防御：Auto Mode 只是一层

最强的安全姿态是多层组合：

**第 1 层：Permission rules**（`settings.json` allow/deny 规则）→ 对特定工具和命令的确定性控制，在分类器之前解析

**第 2 层：Auto Mode 分类器** → 评估规则未覆盖的所有操作，基于上下文推理而非语法匹配

**第 3 层：Hooks**（`PreToolUse`）→ 在权限系统之前运行自定义逻辑；Permission Hook 提供三级处理（快速放行/快速拦截/LLM 分析），与 Auto Mode 可共存，Hook 先运行

**第 4 层：OS 沙盒**（Sandboxing）→ 内核级别限制文件系统和网络访问，即使分类器漏掉某个操作，沙盒也阻止 Bash 命令触及边界外的资源；分类器评估意图，沙盒执行硬边界

**第 5 层：Self-validating agents / Stop hooks** → 确保 Agent 正确完成任务，不偏离分配的范围

每一层捕获其他层遗漏的部分。

---

## 局限性（Research Preview）

- **不保证安全**：用户意图模糊或 Claude 缺少环境上下文时，分类器可能漏掉风险操作，也可能误拦正常操作
- **Token 更多**：每个被检查的操作发送一段对话记录 + 待执行操作给分类器，主要额外开销来自 shell 命令和网络操作（只读操作和工作目录内文件编辑跳过分类器）
- **有延迟**：每次分类器检查增加一个往返，快速连续的 shell 命令序列中开销明显
- **可用性受限**：目前仅 Team 计划，需要 Sonnet 4.6 或 Opus 4.6
- **不能替代对敏感操作的人工审查**：生产环境、凭证、共享基础设施，手动审查仍是正确选择

---

> 原文来源：https://claudefa.st/blog/guide/development/auto-mode

## Related

- [[Scaling Managed Agents: Decoupling the Brain from the Hands]]
- [[Harness Design for Long-Running Application Development]]
- [[Eval Awareness in Claude Opus 4.6's BrowseComp Performance]]
