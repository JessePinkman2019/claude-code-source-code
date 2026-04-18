---
title: "Designing AI-Resistant Technical Evaluations"
aliases:
  - "AI-resistant-technical-evaluations"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Demystifying Evals for AI Agents"
  - "Eval Awareness in Claude Opus 4.6's BrowseComp Performance"
---

# Designing AI-Resistant Technical Evaluations

**来源**: https://www.anthropic.com/engineering/AI-resistant-technical-evaluations  
**作者**: Tristan Hume（Anthropic 性能优化团队负责人）  
**开放挑战**: https://github.com/anthropics/original_performance_takehome  
**性质**: 工程实践——随模型能力提升持续重新设计技术面试题

---

## 一句话概括

Anthropic 性能工程团队用于招聘的 take-home 测试，被 Claude Opus 4 超越大多数人类、被 Opus 4.5 超越最强人类，两次被迫重新设计——最终放弃"逼真度"转向"足够出分布"，以确保人类仍能在 AI 辅助下体现差异。

---

## 背景

**使用场景**：Anthropic 性能工程招聘，用于筛选能优化 Trainium 集群、参与所有 Claude 模型训练的工程师。

**时间线**：
- 2023 年 11 月：设计第一版 take-home
- 2024 年起：超过 1,000 名候选人完成测试，招募了数十名工程师
- 2025 年 5 月：Claude Opus 4 在 4 小时内超越几乎所有人类
- Opus 4.5：在 2 小时内匹配最强人类表现
- 被迫进行根本性重新设计

**重要前提**：Anthropic 明确**允许**候选人在 take-home 中使用 AI 工具（正如他们工作中会使用一样）。

---

## 原始版本设计

### 核心目标

| 目标 | 说明 |
|------|------|
| 代表真实工作 | 给候选人体验实际工作的感觉 |
| 高信噪比 | 避免单一洞察决定成败，给候选人多次展示能力的机会 |
| 无特定领域知识要求 | 好的基础可以学习具体领域，不应限制候选人池 |
| 有趣 | 快速开发循环、有深度的有趣问题、创造空间 |

**为什么选 take-home 而非现场面试**：
- 更长时间窗口（4 小时，后缩短为 2 小时）更接近实际工作节奏
- 真实环境（自己的编辑器，无人旁观）
- 有时间理解系统和构建调试工具
- 与 AI 辅助兼容（候选人可以像工作中一样使用 AI）

### 技术内容

用 Python 模拟一个类似 TPU 的假加速器，包含以下特性：
- **手动管理的 scratchpad 内存**（与 CPU 不同，加速器通常需要显式内存管理）
- **VLIW**（多执行单元并行运行，需要高效的指令打包）
- **SIMD**（向量操作）
- **多核**（跨核分配工作）

**任务**：并行树遍历——故意不是深度学习风格，因为当时大多数性能工程师没有深度学习经验。

候选人从全串行实现开始，逐步利用并行性：
1. 热身：多核并行
2. 候选人选择：SIMD 向量化 OR VLIW 指令打包
3. 原版还包含一个需要调试的 bug

---

## 被 Claude 击败的过程

### 第一次：Claude Opus 4 击败 V1

- **结果**：在 4 小时内超越几乎所有人类
- **应对**：用 Claude Opus 4 找到它开始挣扎的地方，以此为新起点重写 V2
  - 更干净的起始代码
  - 新机器特性增加深度
  - 移除多核（Claude 已解决，且降低信噪比）
  - 时间缩短为 2 小时

### 第二次：Claude Opus 4.5 击败 V2

Claude Code 工作 2 小时的过程：
1. 解决初始瓶颈，实现所有常见微优化
2. 不到 1 小时达到通过门槛
3. 停下来，认定遭遇了无法逾越的内存带宽瓶颈
4. （被告知目标 cycle 数后）思考了一会，找到了利用问题结构绕过瓶颈的技巧
5. 调试、调参、进一步优化
6. 到 2 小时时，达到最强人类（大量使用 Claude 4 辅助的人类）的水平

**基准数据（时钟周期，越低越好）**：

| 测试情况 | Cycles |
|---------|--------|
| Claude Opus 4 跑数小时（test-time compute harness） | 2164 |
| Claude Opus 4.5 普通 Claude Code session | 1790（接近最强人类 2 小时表现） |
| Claude Opus 4.5 在 harness 跑 2 小时 | 1579 |
| Claude Sonnet 4.5 跑更长时间 | 1548 |
| Claude Opus 4.5 在 harness 跑 11.5 小时 | **1487** |
| Claude Opus 4.5 在改进版 harness 跑更多小时 | **1363** |

---

## 重新设计的探索过程

### 被否决的选项

**方案一：禁止 AI 辅助**
- 被否决理由：执行困难；更重要的是，作者不想接受"人类只在几小时以上的任务有优势"的前提，希望找到能在有 AI 的环境下区分人类差异的方法

**方案二：要求大幅超越 Claude Code 的表现**
- 被否决理由：Claude 工作速度快；人类通常花一半时间理解问题；人类引导 Claude 可能总是落后，主导策略可能变成"坐着看"

### 尝试一：换一道优化题（失败）

选了一个作者自己做过的真实优化问题：在 TPU 2D 寄存器上高效转置数据，避免 bank conflict。

- Claude Opus 4.5 找到了作者没想到的优化：**转置整个计算而非转置数据**，重写了整个程序
- 修补问题后，Claude 无法找到最高效解法，看起来可以用
- 但用更长 thinking budget 验证后……Claude 解决了，甚至知道 bank conflict 的修复技巧

**失败原因**：转置和 bank conflict 是很多平台的工程师都研究过的问题，Claude 有大量训练数据。作者从第一原理找到的解法，Claude 可以从更大的经验库中提取。

### 尝试二：变得更"奇怪"（最终方案）

灵感来源：**Zachtronics 游戏**——这类编程谜题游戏使用极不寻常、高度受限的指令集，迫使你以非常规方式编程。

**新 take-home 设计**：
- 使用极小、高度受限的指令集
- 优化目标：最小化指令数
- **故意不提供可视化或调试工具**——起始代码只检查解法是否有效
- 构建调试工具本身是被测试内容的一部分：可以插入打印语句，或用 AI 在几分钟内生成交互式调试器
- 由多个独立子问题组成

**优点**：人类推理能力相对于 Claude 更大的经验库有优势，因为问题足够出分布。

**代价**：
- 放弃了原版的真实感和多维深度
- 作者承认"现实性可能是我们已经负担不起的奢侈品"

---

## 结论与反思

**原版有效，因为它像真实工作。新版有效，因为它模拟了新颖工作。**

作者的坦白：
> "我很遗憾放弃了原版的真实感和多样深度……但现实性可能是我们已经负担不起的奢侈品了。"

**招聘人员面临的根本性困境**：
- Anthropic 的性能工程师现在的工作更像：艰难的调试、系统设计、性能分析、验证 AI 代码的正确性、让 Claude 的代码更简洁优雅
- 这些很难在短时间内以客观方式测试
- 设计代表真实工作的面试题一直很难，现在比以往更难

---

## 开放挑战

Anthropic 开放了原版 take-home 供任何人尝试（无时间限制）。

- 人类专家在足够长的时间窗口内仍优于当前模型
- 有史以来最快的人类解法大幅超越 Claude 即使有大量 test-time compute 的成绩
- 如果你能低于 **1487 cycles**（击败 Claude 发布时的最佳表现），发送代码和简历到 performance-recruiting@anthropic.com

---

## 个人理解

### 最深刻的设计教训

**"Claude 擅长有训练数据的问题，哪怕是你从第一原理独立推导出的答案"**——这是作者在 bank conflict 转置问题上学到的代价最高的教训。这意味着：要设计 AI 难以解决的问题，不能只靠"罕见"，还要靠"真正的分布外"。

### 关于评测设计的范式转变

文章描述的困境不只是面试题设计问题，它是一个更广泛的**评测完整性问题**的缩影：任何基于"解题能力"的评测，一旦 AI 能可靠解题，就失去了区分度。这与前面 eval-awareness 文章描述的基准污染问题有相似的深层结构。

### 对 AI 时代能力评估的启示

作者最终走向"奇怪的"Zachtronics 风格谜题，实际上反映了一个重要原则：**在 AI 能力快速扩张的时代，保持"出分布"比保持"真实"更重要**。这对任何需要评估人类能力的场景都有参考价值——无论是招聘、教育还是资格认证。

## Related

- [[Demystifying Evals for AI Agents]]
- [[Eval Awareness in Claude Opus 4.6's BrowseComp Performance]]
