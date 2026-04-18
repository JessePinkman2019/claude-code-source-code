---
title: "Anthropic Economic Index Report: Learning Curves"
aliases:
  - "economic-index-mar2026-learning-curves"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/research/economic-index-march-2026-report"
source_type: research-note
authors: "Maxim Massenkoff, Eva Lyubich, Peter McCrory, Ruth Appel, Ryan Heller"
year: "2026"
related:
  - "How Australia Uses Claude: Findings from the Anthropic Economic Index"
  - "Labor Market Impacts of AI: A New Measure and Early Evidence"
---

# Anthropic Economic Index Report: Learning Curves

**来源**: https://www.anthropic.com/research/economic-index-march-2026-report  
**发布时间**: 2026 年 3 月 24 日  
**作者**: Maxim Massenkoff, Eva Lyubich, Peter McCrory, Ruth Appel, Ryan Heller  
**数据**: 2026 年 2 月 5–12 日，100 万条 Claude.ai + 100 万条 1P API 对话样本

---

## 一句话概括

与上期报告相比，Claude.ai 的使用场景进一步多样化、任务复杂度略有下降；同时发现**老用户（高 tenure）比新用户成功率高 ~4pp**，与 learning-by-doing 假说一致。

---

## 报告结构

| 章节 | 内容 |
|------|------|
| **第一章** | 自上期报告以来的变化：任务多样性、自动化模式、地理收敛 |
| **第二章** | 学习曲线：模型选择行为 + 老用户 vs 新用户差异 |

---

## 第一章：自上期以来的变化

### 1.1 Claude.ai 任务多样化

| 指标 | Nov 2025 | Feb 2026 |
|------|---------|---------|
| Top 10 任务占比（Claude.ai） | 24% | 19% |
| 学习/课业用途 | 19% | 12% |
| 个人用途 | 35% | 42% |
| 平均任务价值（美元时薪） | $49.3 | $47.9 |
| 提示词理解所需教育年限 | 12.2 年 | 11.9 年 |

**解读**：Claude.ai 任务变得更分散、更日常化——体育结果查询、家居维护、产品比较等简单任务增多。这是标准"技术扩散曲线"现象：早期采用者偏向高价值技术任务（编程），后期普通用户带来更广泛但更低复杂度的用途。

### 1.2 编程任务从 Claude.ai 迁移到 API

- Claude Code 在 API 流量中占据大比例，其 agentic 架构将一个编程任务拆成多个 API 调用，每个都被归类为不同的小任务
- 结果：**编程总量没减少，但在 API 中分散到更多任务类别**
- Claude.ai 的编程占比下降 18%，API 的计算机与数学类占比上升 14%（自 2025 年 8 月起）

**含义**：迁移到 API 的任务更容易被自动化，其对应职业的就业形态可能更快发生改变。

### 1.3 新出现的两类 API 自动化工作流（Feb 2026 占比至少翻倍）

1. **销售 & 业务拓展自动化**：销售素材生成、B2B 线索资质筛选、客户数据丰富化、冷邮件起草
2. **自动化交易 & 市场运营**：监控市场/头寸、提出具体投资建议、向交易员推送市场动态

### 1.4 地理收敛

**美国州级**（收敛继续，但放缓）：
- Top 5 州的人均使用份额：从 30% 降至 24%
- 按当前速度，各州人均使用量达到均等需要 **5–9 年**（上期预测 2–5 年，现在变慢了）

**跨国**（发散加剧）：
- Top 20 国家的人均使用份额：从 45% 升至 48%
- 低采用率国家进一步落后

---

## 第二章：学习曲线

### 2.1 模型选择：用户会按任务复杂度选模型

在 Claude.ai 付费用户中，**Opus 使用率与任务价值正相关**：

| 任务类型 | Opus 使用率 |
|---------|-----------|
| 软件开发（高薪任务） | 34% |
| 辅导/教学（低薪任务） | 12% |
| 整体平均 | 51% |
| 计算机与数学类 | 55%（+4.4pp） |
| 教育类 | 45%（−6pp） |

**API 用户的模型选择更敏感**：任务时薪每提高 $10，Opus 使用率在 Claude.ai 提升 1.5pp，在 API 提升 2.8pp（约为前者 2 倍）。

### 2.2 老用户 vs 新用户（高 tenure = 注册满 6 个月以上）

| 指标 | 低 tenure | 高 tenure | 差异 |
|------|---------|---------|------|
| 工作相关对话比例 | — | — | 高 tenure 高 **7pp** |
| 个人用途比例 | 44% | 38% | 低 tenure 更多个人用途 |
| 提示词教育水平 | — | — | 高 tenure 高 **6%** |
| Top 10 任务占比 | 22.2% | 20.7% | 高 tenure 使用更多样 |
| 对话成功率 | — | — | 高 tenure 高 **~4–5pp** |
| 使用模式 | 更多 directive（指令式） | 更多 iterative（迭代协作） | — |

**tenure 越长，教育水平越高**：注册每增加一年，提示词所需教育年限提升约 1 年。

### 2.3 成功率差异的因果分析

通过逐步增加控制变量的回归验证：

| 规格 | 高 tenure 用户成功率优势 |
|------|----------------------|
| 简单二元回归（无控制） | **+5pp** |
| 加入 O*NET 任务 + 请求类别固定效应 | **+3pp** |
| 再加入模型、使用场景、国家固定效应 | **+4pp** |

即使控制了任务类型、国家、模型选择等因素，高 tenure 用户的成功率依然高出约 **4pp**。

**结论**：这不能仅用"高 tenure 用户做不同任务"来解释，很可能反映了真实的 **learning-by-doing**——使用越多，越懂得如何从 AI 中提取价值。

---

## 两类极端任务（按平均 tenure 排序）

**高 tenure 用户偏好的任务**（早期采用者的技术型需求）：
- AI 研究
- Git 操作
- 修改学术论文
- 创业融资

**低 tenure 用户偏好的任务**（晚期普通用户的日常需求）：
- 写俳句
- 查体育比赛结果
- 派对食物建议

---

## 核心讨论：技能偏向型技术变革（SBTC）的 AI 版本

经济学长期关注**技能偏向型技术变革（Skill-Biased Technological Change）**：技术革新提升高技能劳动者工资，压低低技能劳动者工资。

本报告提供了一个 AI 版本的具体机制：

> **早期高技能采用者 → 更高成功率 → 更大 AI 红利 → 同时也是最"暴露"于 AI 冲击的群体**

这形成了一个自我强化循环：越早用、越擅长用、越受益——但也越可能被 AI 替代。

---

## 方法论说明

- **成功率**：由 Claude 自身评估对话是否成功（详见上期报告定义）
- **任务价值**：用该任务关联职业的美国平均小时工资估算
- **交互类型分类**：directive（指令式）、feedback loop、task iteration、validation、learning → 合并为 automation 和 augmentation 两大类
- **tenure 定义**：高 = 注册至少 6 个月前；低 = 其余

---

## 个人理解

### 对我作为 Claude Code 用户的启示

报告发现高 tenure 用户的核心特征是**更倾向迭代协作（iterative）而非全权委托（directive）**。这与直觉相反——人们以为越熟悉 AI 越会直接丢任务给它，实际上是越用越懂得和 AI 一起做。

具体来说对新手的建议：
- 不要期望一次 prompt 解决所有问题，迭代是正常工作模式
- 成功率差异 4pp 看起来不大，但乘以每天的对话次数就很可观

### 关于 learning-by-doing 的局限

报告自己也承认，高 tenure 用户的优势可能是**cohort effect**（早期用户本来就是程序员等技术人群）或**survivorship bias**（坚持用 6 个月的人本来就更有动力/能力）。无法完全区分这两种解释，需要更长期的数据。

## Related

- [[How Australia Uses Claude: Findings from the Anthropic Economic Index]]
- [[Labor Market Impacts of AI: A New Measure and Early Evidence]]
