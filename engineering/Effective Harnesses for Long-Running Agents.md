---
title: "Effective Harnesses for Long-Running Agents"
aliases:
  - "effective-harnesses-long-running-agents"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Effective Context Engineering for AI Agents"
  - "Scaling Managed Agents: Decoupling the Brain from the Hands"
  - "Harness Design for Long-Running Application Development"
---

# Effective Harnesses for Long-Running Agents

**来源**: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents  
**作者**: Justin Young（Anthropic）  
**性质**: 工程实践——跨多个 context window 的长时运行 agent harness 设计

---

## 一句话概括

通过「初始化 agent + 编码 agent」的两阶段设计，配合特性列表文件、进度文件、git 提交和浏览器自动化测试，让 Claude 能在跨多个 context window 的情况下持续推进复杂软件项目。

---

## 核心问题

长时运行 agent 的根本挑战：**agent 必须在离散的 session 中工作，每个新 session 开始时对之前发生的事情没有任何记忆。**

即使有 compaction（上下文压缩），Opus 4.5 在 Claude Agent SDK 中仅靠高层 prompt（如"构建一个 claude.ai 克隆"）也无法产出生产质量的应用。

### 两个主要失败模式

| 失败模式 | 表现 |
|---------|------|
| **一次性尝试（One-shotting）** | agent 试图一口气完成所有功能，在 context 中途耗尽，留下半实现、未文档化的功能，下一个 session 要花大量时间恢复状态 |
| **提前宣告完成** | 后期 session 看到已有进展，就认为任务完成了 |

---

## 解决方案：两阶段 Agent 架构

### 第一阶段：初始化 Agent（Initializer Agent）

仅在第一次 session 运行，负责建立所有未来 session 需要的环境基础：

1. **`init.sh` 脚本**：启动开发服务器的一键脚本
2. **特性列表文件（`feature_list.json`）**：将用户 prompt 扩展为完整的特性需求列表
3. **进度文件（`claude-progress.txt`）**：记录 agent 工作日志的文件
4. **初始 git 提交**：记录添加了哪些文件的起点快照

**特性列表格式示例**：

```json
{
    "category": "functional",
    "description": "New chat button creates a fresh conversation",
    "steps": [
      "Navigate to main interface",
      "Click the 'New Chat' button",
      "Verify a new conversation is created",
      "Check that chat area shows welcome state",
      "Verify conversation appears in sidebar"
    ],
    "passes": false
}
```

关键设计：
- 所有特性初始标记为 `"passes": false`
- 使用 JSON 而非 Markdown——模型更不容易无意覆盖或修改 JSON 文件
- 强措辞指令："删除或编辑测试是不可接受的，这可能导致功能缺失或有 bug"
- 编码 agent 只允许将 `passes` 字段从 `false` 改为 `true`

### 第二阶段：编码 Agent（Coding Agent）

每个后续 session 的标准化启动流程：

```
1. 运行 pwd，确认工作目录
2. 读取 git log 和进度文件，了解最近的工作
3. 读取特性列表，选择优先级最高的未完成特性
```

**典型 session 开始的 assistant 消息序列**：

```
[Assistant] I'll start by getting my bearings and understanding the current state of the project.
[Tool Use] <bash - pwd>
[Tool Use] <read - claude-progress.txt>
[Tool Use] <read - feature_list.json>
[Assistant] Let me check the git log to see recent work.
[Tool Use] <bash - git log --oneline -20>
[Assistant] Now let me check if there's an init.sh script to restart the servers.
<Starts the development server>
[Assistant] Excellent! Now let me navigate to the application and verify that some fundamental features are still working.
<Tests basic functionality>
[Assistant] Based on my verification testing, I can see that the fundamental functionality is working well...
<Starts work on a new feature>
```

---

## 增量进度策略

### 每次只做一个特性

给定初始环境脚手架后，编码 agent 被要求**每次只处理一个特性**。这是解决"一次性尝试"倾向的关键。

### Session 结束时的"干净状态"

每个 session 结束后，代码应处于可合并到主分支的状态：
- 没有重大 bug
- 代码有序且文档完整
- 开发者可以直接在此基础上开始新特性

实现方式：
- 用**描述性 commit message** 提交 git
- 在进度文件中**写下进度摘要**

这让模型可以用 `git revert` 回滚错误代码，恢复可工作状态。

---

## 测试策略：浏览器自动化

### 失败模式

Claude 倾向于做代码改动、运行单元测试或 curl 命令，但**不能识别端到端功能是否真正工作**。

### 解决方案：Puppeteer MCP

用 Puppeteer MCP 让 Claude 像真实用户一样测试：
- 截图验证
- 页面导航
- 交互操作

效果：戏剧性地提升了性能——agent 能识别和修复仅从代码看不出来的 bug。

**已知局限**：Claude 无法通过 Puppeteer MCP 看到浏览器原生 alert 弹窗，依赖这类弹窗的功能容易有 bug。

---

## 失败模式与解决方案汇总

| 问题 | 初始化 Agent 行为 | 编码 Agent 行为 |
|-----|-----------------|----------------|
| Claude 过早宣告整个项目完成 | 建立特性列表文件：基于输入规格，创建包含端到端特性描述的结构化 JSON | 每次 session 开始时读取特性列表，选择单个特性开始工作 |
| Claude 留下有 bug 或未文档化进度的环境 | 建立初始 git 仓库和进度文件 | session 开始时读取进度文件和 git log，运行基础测试捕捉未记录的 bug；session 结束时写 git commit 和进度更新 |
| Claude 过早将特性标记为完成 | 建立特性列表文件 | 自我验证所有特性，仅在仔细测试后才将特性标记为 passing |
| Claude 要花时间摸索如何运行应用 | 编写 init.sh 脚本启动开发服务器 | session 开始时读取 init.sh |

---

## 未来方向

1. **单一通用编码 agent vs 多 agent 架构**：专门化的 agent（测试 agent、QA agent、代码清理 agent）是否能在软件开发生命周期的子任务上表现更好，目前尚不清楚

2. **泛化到其他领域**：当前 demo 针对全栈 Web 应用优化，这些经验是否能推广到科学研究、金融建模等长时运行任务——可能可以

---

## 与 harness-design-long-running-apps.md 的关系

本文是 `harness-design-long-running-apps.md` 中提到的**前序工作**：

> "作者此前已有两项基础工作：前端设计技能（frontend design skill）、**长时运行编码 agent harness（effective-harnesses-for-long-running-agents）**"

本文描述的是 V1 基础设施（初始化 agent + 编码 agent + 特性列表），后续文章在此基础上加入了生成器-评估器的多 agent 架构。

---

## 个人理解

### 最核心的工程洞察

**"让 agent 用人类工程师每天在做的事情快速上手"**——git log、进度文件、跑测试——这不是 AI 特有的技巧，而是把成熟的工程实践映射到 agent 设计中。

### init.sh 的设计哲学

`init.sh` 的价值不只是便利，它是**一份隐式的环境契约**：初始化 agent 承诺未来的 session 可以用这个脚本恢复环境。这使得每个 session 的开始变成确定性的，而非依赖 agent 自己推断如何启动应用。

### 进度文件 vs git log 的分工

两者都在帮助 agent "上下文恢复"，但作用不同：
- **git log**：结构化、客观的事实记录（做了什么）
- **claude-progress.txt**：主观的意图和状态记录（为什么做、下一步是什么）

这种分离对应于代码注释（为什么）和代码本身（做了什么）的经典原则。

## Related

- [[Effective Context Engineering for AI Agents]]
- [[Scaling Managed Agents: Decoupling the Brain from the Hands]]
- [[Harness Design for Long-Running Application Development]]
