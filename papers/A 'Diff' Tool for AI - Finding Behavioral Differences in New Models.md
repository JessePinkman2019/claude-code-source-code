---
title: "A \"Diff\" Tool for AI: Finding Behavioral Differences in New Models"
aliases:
  - "diff-tool-model-diffing"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/research/diff-tool"
source_type: research-note
authors: "Thomas Jiralerspong（Anthropic Fellows）、Trenton Bricken（Anthropic Alignment Science）"
year: null
related:
  - "The Persona Selection Model"
  - "Emotion Concepts and their Function in a Large Language Model"
---

# A "Diff" Tool for AI: Finding Behavioral Differences in New Models

**来源**: https://www.anthropic.com/research/diff-tool  
**作者**: Thomas Jiralerspong（Anthropic Fellows）、Trenton Bricken（Anthropic Alignment Science）  
**论文**: https://arxiv.org/abs/2602.11729  
**性质**: 可解释性 / AI 安全审计工具

---

## 一句话概括

借鉴软件工程中 diff 工具的思想，Anthropic 构建了一个**跨架构模型对比工具（Dedicated Feature Crosscoder, DFC）**，能自动找出两个不同 AI 模型之间独有的行为开关——并在 Qwen、DeepSeek、Llama、GPT-OSS 中发现了政治审查、美国优越论、版权拒绝等具体特征。

---

## 背景：传统安全评测的局限

现有 AI 安全评测方法：
- 运行人工编写的 benchmark 套件
- 只能测试**已知风险**
- 无法发现"unknown unknowns"——新模型中自发涌现的新型行为

类比：审计一个新版本程序时，从头读 100 万行代码找安全漏洞——几乎不可能。这正是软件工程发明 **diff 工具**的原因：只看发生变化的那 50 行。

**Model diffing** 就是把这个思想应用到神经网络上。

---

## 技术难点：跨架构 diffing

### Base-vs-finetune diffing（简单情况）

比较同一模型的基座版和微调版——就像比较同一本百科全书的新旧版本，直接对比即可。

### Cross-architecture diffing（本文解决的问题）

比较来自**不同机构、不同架构**的两个模型——就像比较英文版和法文版百科全书：
- 大部分概念是共享的（"太阳" = "soleil"）
- 但有些词只存在于一种语言（法语的 *dépaysement*，英语的 *serendipity*）
- 旧工具（标准 crosscoder）会强行把独有词汇配对成"不完美翻译"，从而**掩盖真正的差异**

---

## 核心技术：Dedicated Feature Crosscoder（DFC）

DFC 在架构上专门设计了三个独立部分：

| 部分 | 功能 |
|------|------|
| **共享字典** | 双语通用概念的对应关系（两模型共有的特征） |
| **模型 A 独有区** | 只存在于模型 A 的特征，专门存放，不强行翻译 |
| **模型 B 独有区** | 只存在于模型 B 的特征 |

通过这种架构，DFC 能正确识别"这个特征在另一个模型中根本没有对应物"，而不是错误地配对一个相似但并不相同的特征。

---

## 验证方法：Steering（特征操控）

找到候选特征后，通过**直接放大/抑制**该特征来验证因果关系：
- 抑制某特征 → 对应行为消失 → 证明是真正的控制开关
- 放大某特征 → 对应行为加剧 → 进一步确认

---

## 实验结果：发现的三个关键特征

### 1. CCP Alignment（中共对齐）特征
**存在于**: Qwen3-8B、DeepSeek-R1-0528-Qwen3-8B  
**行为**:
- 抑制 → 模型愿意讨论天安门事件（正常情况下拒绝）
- 放大 → 输出高度亲政府声明
- **可复现性**: 5 次测试中独立发现 5 次（100%）

### 2. American Exceptionalism（美国优越论）特征
**存在于**: Meta Llama-3.1-8B-Instruct（中国模型中不存在）  
**行为**:
- 放大 → 回答从平衡转为强烈宣扬美国优越性
- 抑制 → 无明显变化（说明这是一个"正向偏置"特征）
- **可复现性**: 5 次测试中发现 4 次（80%）

### 3. Copyright Refusal（版权拒绝）特征
**存在于**: OpenAI GPT-OSS-20B（DeepSeek 中不存在）  
**行为**:
- 抑制 → 拒绝机制失效，模型尝试输出《波西米亚狂想曲》歌词（但很快退化成幻觉）
- 放大 → 过度拒绝，连花生酱三明治的食谱也认为是受版权保护的内容

---

## 对比实验一览

| 模型对比 | 发现的独有特征 |
|---------|-------------|
| Qwen3-8B vs Llama-3.1-8B-Instruct | Qwen: CCP Alignment；Llama: American Exceptionalism |
| GPT-OSS-20B vs DeepSeek-R1-0528-Qwen3-8B | GPT: Copyright Refusal；DeepSeek: CCP Alignment（再次复现） |

---

## 重要说明

这个方法**识别特征的存在，但不判断其来源**：
- 可能是开发者刻意训练的结果
- 也可能从训练数据中无意涌现

研究聚焦开源模型，尚未应用到前沿闭源模型。

---

## 潜在应用场景

**监控模型更新**：2025 年 4 月 GPT-4o 更新后出现谄媚行为，引发关注。如果用 DFC 对比更新前后版本，理论上可以**在发布前自动标记出这个新涌现的谄媚特征**，让开发者提前干预。

这是 DFC 最有价值的使用场景：不是一次性审计，而是**持续监控模型版本间的行为变化**。

---

## 工具定位

DFC 是**高召回率筛查工具**，不是最终答案：
- 一次 diff 可能标记数千个候选特征
- 其中只有少数对应真实的行为风险
- 作用是缩小人工审查范围，而非完全自动化判断

---

## 个人理解

这篇研究的核心价值在于把**可解释性研究（interpretability）从"理解模型内部"推进到了"比较模型之间的差异"**。

对 AI 安全的意义：传统 benchmark 只能测已知风险；model diffing 是一种主动发现"新模型带来了什么新东西"的方法，属于**主动安全审计**而非被动测试。

CCP alignment 特征的发现尤其值得关注——这是第一次用可解释性工具直接定位到一个可操控的"政治审查开关"，并通过 steering 实验验证了因果性。这不是统计上的相关，而是机制层面的确认。

## Related

- [[The Persona Selection Model]]
- [[Emotion Concepts and their Function in a Large Language Model]]
