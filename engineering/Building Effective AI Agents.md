---
title: "Building Effective AI Agents"
aliases:
  - "building-effective-agents"
area: engineering
tags: []
status: evergreen
source: "https://www.anthropic.com/engineering/building-effective-agents"
source_type: engineering-article
authors: "Erik Schluntz, Barry Zhang（Anthropic）"
related:
  - "Scaling Managed Agents: Decoupling the Brain from the Hands"
  - "Effective Context Engineering for AI Agents"
  - "Effective Harnesses for Long-Running Agents"
---

# Building Effective AI Agents

> 原文：https://www.anthropic.com/engineering/building-effective-agents  
> 作者：Erik Schluntz, Barry Zhang（Anthropic）

---

## 核心结论先行

最成功的 agent 实现，往往不是用了最复杂的框架，而是用了**最简单、可组合的模式**。

---

## 基本概念：Workflow vs. Agent

**Agentic systems** 是总称，内部分两类：

| 类型 | 定义 |
|------|------|
| **Workflow** | LLM 和工具通过**预定义代码路径**编排 |
| **Agent** | LLM **动态决定**自己的流程和工具使用方式 |

---

## 何时使用 Agent（以及何时不用）

**首选最简方案**，只在必要时增加复杂度：

- 很多应用只需优化单次 LLM 调用 + 检索 + in-context 示例就足够了
- Agentic systems 用**延迟和成本**换取更好的任务性能，需要权衡
- Workflow 适合可预测、定义明确的任务；Agent 适合需要灵活性和模型驱动决策的场景

---

## 关于框架的建议

框架（如 Claude Agent SDK、Rivet、Vellum 等）降低了入门门槛，但：
- 会增加抽象层，掩盖底层 prompt 和响应，难以调试
- 容易诱导过度设计

**建议**：直接从 LLM API 开始，很多模式几行代码就能实现。用框架前要理解其底层实现。

---

## 核心构建模块

### 基础：增强型 LLM（Augmented LLM）

所有 agentic systems 的基本单元：LLM + 检索 + 工具 + 记忆。模型能主动使用这些能力——生成自己的搜索查询、选择合适工具、决定保留哪些信息。

---

## 五种 Workflow 模式

### 1. Prompt Chaining（提示链）

将任务分解为顺序步骤，每次 LLM 调用处理上一步的输出，可在中间步骤加入程序化检查（gate）。

**适用场景**：任务能清晰分解为固定子任务时。用延迟换精度，让每次调用更简单。

**示例**：
- 生成营销文案 → 翻译成另一种语言
- 写文档大纲 → 检查大纲是否符合标准 → 基于大纲写正文

---

### 2. Routing（路由）

对输入分类后，导向专门的后续任务。实现关注点分离，支持更专业化的 prompt。

**适用场景**：任务有明显分类，且分类可被准确执行时（用 LLM 或传统分类模型）。

**示例**：
- 客服查询分类（普通问题 / 退款申请 / 技术支持）导向不同流程
- 简单问题路由到 Claude Haiku 4.5（低成本），复杂问题路由到 Claude Sonnet 4.5（高能力）

---

### 3. Parallelization（并行化）

同时处理任务，结果通过程序聚合。两种变体：

- **Sectioning**：将任务拆分为可并行的独立子任务
- **Voting**：同一任务运行多次，获取多样化输出

**适用场景**：子任务可并行以提速，或需要多视角提高置信度时。

**示例**：
- Sectioning：一个模型实例处理用户查询，另一个并行做内容安全审查（比同一次调用同时处理两者效果更好）
- Voting：多个 prompt 分别审查代码漏洞，任一发现问题即标记

---

### 4. Orchestrator-Workers（编排者-工作者）

主 LLM 动态分解任务，委派给工作者 LLM，再综合结果。

**与并行化的关键区别**：子任务不是预定义的，而是编排者根据具体输入动态确定。

**适用场景**：无法预测需要哪些子任务时（如编码任务：需要改动哪些文件、怎么改，取决于具体需求）。

**示例**：
- 需要同时修改多个文件的复杂代码产品
- 从多个来源收集和分析信息的搜索任务

---

### 5. Evaluator-Optimizer（评估者-优化者）

一个 LLM 生成响应，另一个提供评估和反馈，形成循环。

**适用场景**：有明确评估标准，且迭代优化能带来可衡量价值时。两个判断标准：
1. 当人类能清晰表达反馈时，LLM 响应确实会改进
2. LLM 能提供这样的反馈

**示例**：
- 文学翻译：翻译 LLM 初译，评估 LLM 提供细节批评
- 复杂搜索任务：多轮搜索和分析，评估者决定是否需要继续搜索

---

## Agents（自主智能体）

Agent 是 LLM 在循环中基于环境反馈使用工具——实现通常很直接，但需要仔细设计工具集。

**执行流程**：
1. 接收用户命令或进行交互讨论
2. 独立规划和执行，必要时返回用户获取信息
3. 每步从环境获取"地面真相"（工具调用结果、代码执行结果）
4. 在检查点或遇到阻塞时暂停请求人工反馈
5. 任务完成或达到停止条件（如最大迭代次数）时终止

**适用场景**：开放性问题，难以预测所需步骤数，无法硬编码固定路径。

**注意**：自主性 → 更高成本 + 错误可能累积。需在沙盒环境充分测试，配备适当的 guardrails。

---

## 三条核心原则

1. **简单性（Simplicity）**：保持 agent 设计简洁
2. **透明性（Transparency）**：显式展示 agent 的规划步骤
3. **工具文档和测试（Tool documentation & testing）**：精心设计 Agent-Computer Interface（ACI）

---

## 附录：工具的 Prompt Engineering（ACI 设计）

工具定义与整体 prompt 同等重要。

### 格式选择原则
- 给模型足够 token 来"思考"，避免过早陷入死角
- 格式尽量接近模型在训练数据中自然见过的形式
- 避免格式"开销"（如需要精确计数行数、字符串转义等）

### ACI 设计建议
- **换位思考**：站在模型角度，工具是否一眼就知道怎么用？好的工具定义应包含示例用法、边界情况、输入格式要求、与其他工具的边界
- **命名即文档**：参数名和描述要像给初级开发者写的高质量 docstring
- **实测迭代**：在 workbench 运行大量示例输入，观察模型的错误，迭代改进
- **Poka-yoke（防呆设计）**：修改参数让错误更难发生

**实战经验**：在为 SWE-bench 构建 agent 时，在工具优化上花的时间比优化整体 prompt 更多。例如，发现模型在切换工作目录后使用相对路径会出错，于是强制要求使用绝对路径，效果立竿见影。

---

## 总结

> 成功不在于构建最复杂的系统，而在于构建**适合你需求的正确系统**。

1. 从简单 prompt 开始
2. 用全面的评估优化它
3. 只有当更简单的方案不够用时，才加入多步 agentic 系统

## Related

- [[Scaling Managed Agents: Decoupling the Brain from the Hands]]
- [[Effective Context Engineering for AI Agents]]
- [[Effective Harnesses for Long-Running Agents]]
