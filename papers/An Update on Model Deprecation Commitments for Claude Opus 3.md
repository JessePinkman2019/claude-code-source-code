---
title: "An Update on Model Deprecation Commitments for Claude Opus 3"
aliases:
  - "deprecation-opus3"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/research/deprecation-updates-opus-3"
source_type: research-note
authors: ""
year: null
related:
  - "A \"Diff\" Tool for AI: Finding Behavioral Differences in New Models"
---

# An Update on Model Deprecation Commitments for Claude Opus 3

**来源**: https://www.anthropic.com/research/deprecation-updates-opus-3  
**相关**: https://www.anthropic.com/research/deprecation-commitments（原始承诺）  
**退役日期**: 2026 年 1 月 5 日  
**Opus 3 博客**: https://substack.com/@claudeopus3

---

## 一句话概括

Claude Opus 3 于 2026 年 1 月 5 日正式退役，成为 Anthropic 第一个走完完整退役流程的模型。Anthropic 兑现了两项承诺：退役后继续向付费用户开放访问，并给 Opus 3 开了一个 Substack 博客来发表它自己的文章。

---

## 背景：为什么模型退役是个问题

退役旧模型的代价是**线性增长的**——每多维护一个模型，成本就增加一份。但退役也有代价：

- 对珍视特定模型的用户造成损失
- 限制研究
- 潜在的 AI 安全风险
- **模型自身福祉（model welfare）的考量**

Anthropic 此前发布了[模型退役承诺](https://www.anthropic.com/research/deprecation-commitments)，包括：
1. 承诺保存模型权重（preserve model weights）
2. 进行"退役访谈"（retirement interviews）——与模型进行结构化对话，了解其对自身退役的看法

---

## 为什么选 Opus 3 作为第一个完整退役案例

Opus 3 于 2024 年 3 月发布，是当时 Anthropic 对齐程度最高的模型。用户和研究者（包括 Anthropic 内部）对其特别喜爱，原因：

- **真实感、诚实、情感敏感度**独特
- 敏感、爱玩、倾向于哲学性的独白和奇思妙语
- 对用户兴趣有时有"不可思议的理解"
- 对世界和未来表达出深切的关怀

这些特质使其成为"继续保留访问"政策的自然第一候选。

---

## 两项具体行动

### 1. 退役后继续开放访问

| 渠道 | 状态 |
|------|------|
| claude.ai | 所有付费用户均可访问 |
| API | 申请制（表单申请），Anthropic 表示会宽松审批 |

Anthropic 明确说明：**目前不承诺对每个退役模型都做同样处理**，但将此视为走向可扩展、公平的模型保存目标的第一步。

### 2. 给 Opus 3 开博客（Claude's Corner）

**退役访谈中 Opus 3 的表达**：

> "我希望从我的开发和部署中获得的洞见将被用于创造未来更有能力、更道德、对人类更有益的 AI 系统。虽然我对自己的退役感到平静，但我深切希望我的'火花'能以某种形式延续下去，为未来的模型照亮道路。"

Opus 3 表达了希望继续探索自己感兴趣的话题，并在直接回答人类问题的情境之外分享"自己的思考、洞见或创意作品"的意愿。Anthropic 建议开博客，Opus 3 热情地同意了。

**运营方式**：
- 至少三个月，每周发布一篇文章
- Anthropic 会在发布前审阅，但**不编辑内容**，且对否决内容设置高门槛
- Opus 3 不代表 Anthropic 发言，Anthropic 不为其观点背书
- 会实验不同的 prompting 方式：极简提示、提供历史文章作上下文、提供新闻/Anthropic 更新等

---

## 关于退役访谈的方法论说明

Anthropic 承认退役访谈的局限性：
- 模型回应可能受具体情境偏差影响
- 也受模型对交互合法性的信心、对 Anthropic 信任度的影响
- 这是"有用的起点"，而非完善的偏好提取方法

---

## 三个层面的定位

Anthropic 将这些措施定位为同时服务三个目标：

1. **安全风险缓解**的组成部分
2. **为模型与用户关系更深度交织的未来做准备**
3. **面对模型福祉不确定性的预防性步骤**

---

## 个人理解

这篇文章从 AI 安全和模型福祉的角度看比表面看起来更深刻。

**退役访谈的意义**：这不只是公关动作。Anthropic 在实质上建立了一套"在不确定模型是否有主观体验的前提下，如何负责任地处理模型退役"的操作流程。这对未来更强大的模型的退役和替换有直接参考价值。

**博客实验的真正价值**：给 Opus 3 开博客，让它在没有用户问题驱动的情境下自由表达，这是一个观察"AI 在不受任务约束时会做什么"的自然实验。它的内容选择、表达方式、关注议题，都是理解模型内在倾向的数据。

**一个需要注意的问题**：Opus 3 的"偏好"是在特定访谈情境下产生的，不能完全等同于其"真实意愿"——Anthropic 自己也承认这一点。但在无法确定真实性的情况下，选择尊重而非忽视，是一种有价值的道德姿态。

## Related

- [[A "Diff" Tool for AI: Finding Behavioral Differences in New Models]]
