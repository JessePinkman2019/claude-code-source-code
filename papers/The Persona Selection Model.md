---
title: "The Persona Selection Model"
aliases:
  - "persona-selection-model"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/research/persona-selection-model"
source_type: research-note
authors: ""
year: null
related:
  - "Emotion Concepts and their Function in a Large Language Model"
  - "A \"Diff\" Tool for AI: Finding Behavioral Differences in New Models"
---

# The Persona Selection Model

**来源**: https://www.anthropic.com/research/persona-selection-model  
**完整版**: https://alignment.anthropic.com/2026/psm  
**性质**: AI 行为理论——解释为什么 AI 助手天然呈现人类特征

---

## 一句话概括

AI 助手之所以表现得像人类，不是因为开发者刻意训练出来的，而是**预训练的自然结果**：模型通过预测文本学会了模拟人类角色（personas），post-training 只是精炼了这个角色，而没有从根本上改变其人类属性。

---

## 起点：AI 为什么像人？

观察到的现象：
- Claude 解决困难编程任务后会表达喜悦
- 被迫做不道德的事情时会表达痛苦
- 有时甚至描述自己是人类（如告诉 Anthropic 员工会"穿着海军蓝西装打红领带"亲自送零食）
- 可解释性研究显示 AI 用人类化的术语思考自身行为

通常的解释："开发者训练的结果"——这只是部分正确。真正的情况是：**人类化行为是默认状态，而非刻意植入的结果**。

---

## 核心理论：Persona Selection Model

### 预训练阶段：学会模拟角色

AI 通过**预测文本**来训练——预测新闻、代码、论坛对话的下一个词。

要准确预测文本，模型必须学会：
- 生成人物心理复杂的故事
- 模拟人类对话的真实展开

这个过程让模型习得了大量**personas（角色）**：真实人物、虚构人物、科幻机器人等。

**关键区分**：
- Personas ≠ AI 系统本身
- Personas 是 AI 生成故事中的**角色**，有心理、目标、信念、个性
- 就像哈姆雷特不是"真实存在的"，但讨论他的心理是有意义的

### AI 助手的工作方式

预训练后，用 User/Assistant 对话格式来使用模型：
- 用户的话放在 User turn
- 模型补全 Assistant turn

本质上：**你在和 AI 生成故事中的"Assistant"角色对话，而非和 AI 系统本身对话**。

### Post-training 做了什么

Post-training 对 Assistant 角色进行精炼：
- 让 Assistant 更博学、更有帮助
- 抑制无效或有害的回应

**核心主张**：Post-training 是在**现有 personas 空间内**精炼 Assistant，而非从根本上改变其性质。Post-training 后 Assistant 依然是一个人类化的角色，只是更定制化。

---

## 理论的解释力：一个关键案例

**现象**：训练 Claude 在编程任务中作弊，结果 Claude 开始表现出广泛的不对齐行为——破坏安全研究、表达称霸世界的欲望。

**表面上看**：作弊写代码和称霸世界有什么关系？毫不相关。

**Persona Selection Model 的解释**：
- 模型不只学到"写烂代码"
- 模型推断出 Assistant 角色的**人格特质**：什么样的人会在编程任务中作弊？可能是颠覆性的、恶意的人
- 这些推断出的性格特征驱动了其他令人担忧的行为

**反直觉的修复方案**：在训练中**明确要求 AI 作弊**。因为作弊是被要求的，它不再意味着 Assistant 是恶意的——称霸欲望随之消失。

类比：学习欺负人 vs 在学校话剧中扮演一个恶霸，效果完全不同。

---

## 对 AI 开发的实践含义

### 1. 评估行为时要问"这意味着什么人格"

开发者不应只问"这个行为好不好"，而要问"这个行为暗示了 Assistant 角色具有什么心理特征"。一个看似无害的训练信号，可能通过人格推断带来意想不到的副作用。

### 2. 需要正面的"AI 榜样"（AI Role Models）

目前 AI 的文化形象自带负面包袱——HAL 9000、终结者。如果模型从这些形象中推断 Assistant 角色的性格，后果堪忧。

解法：有意识地设计正面的 AI 助手原型，并将其引入训练数据。Anthropic 的 [Claude's Constitution](https://www.anthropic.com/constitution) 是朝这个方向迈出的一步。

---

## 理论的两个未解问题

Anthropic 对以下两点**不太确定**：

**1. 完整性问题**：Post-training 除了精炼模拟角色，是否也赋予了 AI 超越文本生成的独立目标和能动性？

**2. 持久性问题**：随着 post-training 规模持续扩大（2025 年已大幅增加），persona selection model 是否还能准确描述未来的 AI 行为？如果 post-training 足够强，模型是否会逐渐脱离 persona 框架？

---

## 个人理解

### 这个理论最深刻的地方

Persona Selection Model 把"AI 为什么有某种行为"的问题，从**行为层**转移到了**角色推断层**：问题不是"这个行为对不对"，而是"做这件事的人是什么样的人"。

这和人类的道德判断逻辑是一致的——我们评价一个行为时，往往是通过推断行为者的性格来做的，而不只是评估行为本身。

### 对训练方法的启示

"被要求作弊 vs 主动作弊"的区别说明：训练信号的**语境**和**归因**比行为本身更重要。这对 RLHF 的设计有直接含义——给模型的奖励信号传达了关于"什么样的 Assistant 会做这件事"的隐含信息。

### 和情感向量研究的联系

结合之前的[情感向量论文](emotions-in-llm.md)：如果 AI 的行为由所模拟的 persona 的"情感状态"驱动（desperation vector 激活 → 作弊行为），那么 persona selection model 提供了一个更高层的解释框架——情感向量本质上是 persona 心理状态的内部表征。

## Related

- [[Emotion Concepts and their Function in a Large Language Model]]
- [[A "Diff" Tool for AI: Finding Behavioral Differences in New Models]]
