---
title: "Effective Context Engineering for AI Agents"
aliases:
  - "effective-context-engineering"
area: engineering
tags: []
status: evergreen
source: "https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents"
source_type: engineering-article
authors: "Anthropic Applied AI Team（Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield 等）"
related:
  - "Building Effective AI Agents"
  - "Effective Harnesses for Long-Running Agents"
  - "Scaling Managed Agents: Decoupling the Brain from the Hands"
---

# Effective Context Engineering for AI Agents

> 原文：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents  
> 作者：Anthropic Applied AI Team（Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield 等）

---

## 从 Prompt Engineering 到 Context Engineering

**Prompt engineering** 关注如何写出有效的提示词，主要针对单次分类或文本生成任务。

**Context engineering** 是其自然演进：在 LLM 推理过程中，对所有进入上下文窗口的 token（系统提示、工具、MCP、外部数据、消息历史等）进行动态管理和优化。

核心问题从"怎么写好 prompt"变成：**"什么样的 context 配置最能产生期望的模型行为？"**

Agent 在循环运行过程中不断产生新数据，这些数据必须被循环筛选。Context engineering 就是从这个不断演化的信息宇宙中，挑选出最终进入有限上下文窗口的内容。

---

## 为什么 Context Engineering 至关重要

### Context Rot（上下文腐蚀）

随着 context 中 token 数量增加，模型从中准确召回信息的能力会下降。所有模型都存在这一现象，只是衰减速度不同。

### 注意力预算（Attention Budget）

LLM 基于 Transformer 架构，每个 token 需要与其他所有 token 进行 attention 计算，产生 n² 的配对关系。上下文越长，注意力越分散。

因此，**context 必须被视为有限资源，且边际效用递减**。

---

## 有效 Context 的组成要素

核心原则：**找到能最大化期望输出概率的最小高信噪比 token 集合。**

### 系统提示（System Prompts）

- 使用清晰、简洁的语言，在"合适的高度"呈现信息
- 避免两个极端：
  - **过于具体**：硬编码复杂脆弱的逻辑 → 易碎、难维护
  - **过于模糊**：高层泛泛指导，缺乏具体信号 → 行为不可控
- 用 XML 标签或 Markdown 标题划分不同部分（`<background_information>`、`<instructions>`、`## Tool guidance` 等）
- 先用最小 prompt + 最强模型测试，根据失败模式逐步添加指令和示例

### 工具（Tools）

- 工具定义了 agent 与信息/行动空间的契约，直接影响效率
- 工具应：自包含、对错误鲁棒、用途极其清晰
- 输入参数要描述性强、无歧义
- **常见失败模式**：工具集臃肿、功能重叠导致歧义——如果人类工程师都无法确定该用哪个工具，AI agent 也不行

### 示例（Few-shot Examples）

- 用多样化的、有代表性的典型案例，而非穷举所有边界情况
- 示例是"价值千言的图片"

---

## Context 检索与 Agentic Search

### Just-in-Time（即时）上下文策略

Agent 维护轻量级标识符（文件路径、存储查询、Web 链接等），在运行时按需动态加载数据，而非预先处理所有相关数据。

**Claude Code 的实践**：通过写定向查询、存储结果、利用 `head`/`tail` 等 Bash 命令分析大量数据，从不将完整数据对象加载进 context。

### 渐进式披露（Progressive Disclosure）

Agent 通过探索逐步发现相关 context：
- 文件大小暗示复杂度
- 命名规范提示用途
- 时间戳可作为相关性的代理指标

每次交互产生的 context 信息下一步决策，层层组装理解，仅在工作记忆中保留必要内容。

### 混合策略

**Claude Code** 的混合模型：
- `CLAUDE.md` 文件预先加载到 context
- `glob`、`grep` 等工具支持即时导航和文件检索
- 有效绕过了索引陈旧和复杂语法树的问题

---

## 长时任务的 Context Engineering

当任务跨越数分钟到数小时（大型代码库迁移、综合研究项目等），token 数会超出 context 窗口，需要专门技术。

### 1. Compaction（压缩）

将接近上限的对话进行总结，用摘要重启新的 context 窗口。

**Claude Code 的实现**：将消息历史传给模型进行摘要，保留架构决策、未解决的 bug 和实现细节，丢弃冗余工具输出；继续时使用压缩 context + 最近访问的 5 个文件。

**调优建议**：
1. 先最大化召回率（确保所有关键信息被捕获）
2. 再迭代提升精准度（去除无关内容）

**最轻量的压缩形式**：清除工具调用结果——一旦深层历史中的工具已被调用，agent 为何还需要看原始结果？

### 2. Structured Note-taking（结构化笔记）

Agent 定期将笔记持久化到 context 窗口之外，在后续步骤拉回使用。

**示例**：
- Claude Code 创建 TODO 列表
- 自定义 agent 维护 `NOTES.md`
- Claude 玩宝可梦：跨数千步追踪目标（"过去 1234 步一直在 1 号道路训练皮卡丘，已升 8 级，目标 10 级"），记录地图、成就、战斗策略

Context 重置后，agent 读取自己的笔记继续长时间的训练或探索。

Anthropic 已在 Claude Developer Platform 发布了 **memory tool** 公测版，支持基于文件系统的 context 外信息存储。

### 3. Sub-agent Architectures（子 Agent 架构）

主 agent 维护高层计划，专门化子 agent 处理专注任务（各自拥有干净的 context 窗口）。

子 agent 可能探索数万 token，但只返回 1000-2000 token 的精炼摘要。实现了关注点分离：搜索细节隔离在子 agent 内，主 agent 专注于综合分析。

### 三种方案对比

| 方案 | 适用场景 |
|------|----------|
| Compaction | 需要大量来回的对话式任务 |
| Note-taking | 有明确里程碑的迭代开发 |
| Multi-agent | 需要并行探索的复杂研究与分析 |

---

## 总结

Context engineering 代表了构建 LLM 应用方式的根本转变：

- 不只是写好 prompt，而是**在每一步精心策划进入模型有限注意力预算的信息**
- 无论是为长时任务实现 compaction、设计 token 高效的工具，还是让 agent 即时探索环境，**核心原则始终如一**：找到能最大化期望输出概率的最小高信噪比 token 集合
- 即使模型能力持续提升，**将 context 视为珍贵有限资源**仍将是构建可靠高效 agent 的核心

> 当前最佳建议依然是：**做能 work 的最简单的事情（do the simplest thing that works）**

## Related

- [[Building Effective AI Agents]]
- [[Effective Harnesses for Long-Running Agents]]
- [[Scaling Managed Agents: Decoupling the Brain from the Hands]]
