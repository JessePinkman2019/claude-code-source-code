---
title: "Harness Design for Long-Running Application Development"
aliases:
  - "harness-design-long-running-apps"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Effective Harnesses for Long-Running Agents"
  - "Effective Context Engineering for AI Agents"
  - "Auto Mode：用 AI 分类器替代手动权限审批"
---

# Harness Design for Long-Running Application Development

**来源**: https://www.anthropic.com/engineering/harness-design-long-running-apps  
**作者**: Prithvi Rajasekaran（Anthropic Labs 团队成员）  
**性质**: 工程实践文章——多智能体 harness 设计，用于自主构建完整应用

---

## 一句话概括

通过 GAN 启发的生成器-评估器多智能体架构（加上规划器），Claude 可在无人干预下自主运行数小时，构建出功能完整的全栈应用——关键突破是将"自评"改为"他评"，并随模型能力提升持续精简 harness。

---

## 背景与问题

### 起点

作者此前已有两项基础工作：
- **前端设计技能**（frontend design skill）
- **长时运行编码 agent harness**（effective-harnesses-for-long-running-agents）

两者都通过 prompt 工程和 harness 设计超越了基线，但最终遇到天花板。需要寻找新的突破。

### 两个持续性失败模式

**模式一：Context 焦虑（Context Anxiety）**

模型在上下文窗口将满时开始过早收尾工作，而非继续完成任务。

解决方案对比：
| 方式 | 原理 | 效果 |
|------|------|------|
| **Compaction（压缩）** | 摘要早期对话，保持同一 agent 继续 | 不彻底，context 焦虑仍可持续 |
| **Context Reset（重置）** | 清空上下文，启动新 agent，通过交接文件传递状态 | 提供干净的 slate，解决根本问题 |

注：Context Reset 引入了额外的编排复杂度、token 开销和延迟，但对 Sonnet 4.5 是必要的。Opus 4.6 显著改善了此问题，因此可以完全移除 Context Reset。

**模式二：自评偏差（Self-Evaluation Bias）**

当 agent 评估自己生成的工作时，会倾向于过度称赞——即使质量明显平庸。主观任务（如设计）尤甚，因为没有等价于软件测试的二元校验。

解决方案：**将生成者与评估者分离**。
- 独立的评估 agent 比让生成者自我批评更容易调教成"挑剔"
- 有了外部反馈，生成者才有具体的迭代目标

---

## 第一步：前端设计的生成器-评估器循环

### 四项评分标准

作者设计了四个评分维度，同时放入生成器和评估器的 prompt：

| 标准 | 说明 | 权重 |
|------|------|------|
| **设计质量** | 颜色、排版、布局、图像构成整体一致的氛围和身份 | 高 |
| **原创性** | 有定制化决策，而非模板布局/库默认值/AI 套路（如紫色渐变+白卡片）| 高 |
| **工艺** | 排版层级、间距一致性、色彩和谐、对比度 | 低（默认就好） |
| **功能性** | 可用性，用户能否理解界面、找到主要操作 | 低（默认就好） |

原创性标准明确惩罚"AI 特征"输出，权重更高，推动模型在设计上冒更多风险。

### 工作流

- 生成器：创建 HTML/CSS/JS 前端
- 评估器：通过 **Playwright MCP** 直接与运行中的页面交互（截图、导航、研究），然后打分+详细批评
- 循环 5–15 次迭代，每次推动生成器朝更独特的方向
- 评估器用少样本示例校准，减少分数漂移
- 每次完整运行可长达 4 小时

### 关键观察

- 即使是第一次迭代（没有任何评估器反馈），输出也比完全没有 prompt 时明显更好——说明评分标准本身的语言就在引导模型
- 分数并非线性改善，有时中间迭代优于最终迭代
- 后期实现复杂度趋于增加，生成器会寻求更有野心的方案

**典型案例**：为荷兰艺术博物馆生成网站，前九轮产出了一个干净的深色主题着陆页。第十轮彻底推翻，重新构想为：CSS 透视渲染的 3D 房间+棋盘格地板+墙壁上的画作+走廊导航——这种创意飞跃从未在单次生成中出现过。

---

## 第二步：扩展到全栈编码

### 三智能体架构

在前端实验经验的基础上，构建了一个三智能体系统：

```
用户 prompt (1-4 句话)
        ↓
  [规划器 Planner]
  扩展为完整产品规格
  （大胆定义范围，聚焦产品上下文和高层技术设计，
    避免过度规定实现细节）
        ↓
  [生成器 Generator]
  按 sprint 实现功能（每次一个 feature）
  每个 sprint 前与评估器协商 sprint 契约
  自我评估后交接给 QA
        ↓
  [评估器 Evaluator]
  通过 Playwright MCP 像用户一样点击测试运行中的应用
  检查 UI 功能、API 端点、数据库状态
  按标准打分，任意维度低于阈值则 sprint 失败
  返回详细反馈供生成器修复
```

### Sprint 契约机制

每个 sprint 开始前，生成器和评估器协商 **sprint 契约**：
- 生成器提出本次将构建什么、如何验证成功
- 评估器审核提案，确认构建的是正确的东西
- 双方通过**文件**通信（一方写文件，另一方读取并响应）
- 达成一致后，生成器才开始编码

作用：弥合高层产品规格与可测试实现之间的鸿沟，在不过度规定实现细节的同时保持 faithful。

### V1 harness 实测（视频游戏制作器）

**Prompt**: "Create a 2D retro game maker with features including a level editor, sprite editor, entity behaviors, and a playable test mode."

| Harness 类型 | 时长 | 费用 |
|-------------|------|------|
| Solo agent | 20 分钟 | $9 |
| 完整 harness | 6 小时 | $200 |

**Solo 结果**的问题：
- 布局浪费空间，固定高度面板
- 工作流僵硬，没有引导顺序
- 核心游戏功能损坏——实体出现在屏幕上但对输入无响应

**Harness 结果**：
- 规划器将一句话扩展为 16 个 feature、分 10 个 sprint 的规格
- 包括精灵动画系统、行为模板、音效音乐、**AI 辅助精灵生成器和关卡设计师**、游戏导出+分享链接
- 使用了前端设计技能文档，创建了一致的视觉设计语言
- 实际可以移动实体并玩游戏（Solo 版本根本无法运行）

**评估器发现的真实问题示例**：

| 契约标准 | 评估器发现 |
|---------|-----------|
| 矩形填充工具允许拖动填充矩形区域 | **失败** — 只在拖动起止点放置 tile，`fillRectangle` 函数存在但未在 mouseUp 时正确触发 |
| 用户可以选择和删除已放置的实体 | **失败** — `LevelEditor.tsx:892` 的 Delete 键处理需要同时设置 `selection` 和 `selectedEntityId`，但点击实体只设置 `selectedEntityId` |
| 用户可以通过 API 重新排序动画帧 | **失败** — `PUT /frames/reorder` 路由定义在 `/{frame_id}` 路由之后，FastAPI 将 `reorder` 解析为整数 frame_id，返回 422 |

---

## 第三步：迭代精简 harness

### 核心原则

> **"Find the simplest solution possible, and only increase complexity when needed."**（Building Effective Agents）

Harness 的每个组件都编码了"模型自己无法做到"的假设——这些假设值得压力测试，因为：
1. 假设可能本来就是错的
2. 模型改善后假设会迅速过时

### 移除 Sprint 结构

Opus 4.6 的改进（来自发布博客）：
- 更谨慎地规划
- 能更长时间维持 agent 任务
- 在更大代码库中更可靠运行
- 更好的代码审查和调试技能，能捕捉自己的错误
- 长上下文检索大幅改善

这意味着 Opus 4.5 需要 Sprint 分解才能处理的工作，4.6 可以原生完成。因此：
- 移除 Sprint 构造
- 评估器改为在整个运行结束后的单次检查（而非每个 sprint 一次）
- Opus 4.6 也消除了 Context Anxiety，不再需要 Context Reset——整个构建用一个连续会话，Agent SDK 的自动 compaction 处理上下文增长

### V2 harness 实测（数字音频工作站 DAW）

**Prompt**: "Build a fully featured DAW in the browser using the Web Audio API."

| 阶段 | 时长 | 费用 |
|------|------|------|
| 规划器 | 4.7 分钟 | $0.46 |
| 构建 Round 1 | 2 小时 7 分钟 | $71.08 |
| QA Round 1 | 8.8 分钟 | $3.24 |
| 构建 Round 2 | 1 小时 2 分钟 | $36.89 |
| QA Round 2 | 6.8 分钟 | $3.09 |
| 构建 Round 3 | 10.9 分钟 | $5.88 |
| QA Round 3 | 9.6 分钟 | $4.06 |
| **总计** | **3 小时 50 分钟** | **$124.70** |

Builder 连续运行超过 2 小时，没有 Sprint 分解。

**QA 第一轮反馈**：
> 这是一个强大的应用，设计保真度出色，AI agent 扎实，后端良好。主要失败点是功能完整性——应用看起来令人印象深刻，AI 集成运作良好，但几个核心 DAW 功能只是展示性的：片段无法在时间线上拖动/移动，没有乐器 UI 面板（合成器旋钮、鼓垫），没有视觉效果编辑器（EQ 曲线、压缩器计量表）。这些不是边缘情况——它们是使 DAW 可用的核心交互。

**最终结果**：包含可工作的编排视图、混音器和传输的完整音乐制作程序。可以通过 prompting 完整创作一个短曲片段——AI agent 设置节拍和调性、铺旋律、构建鼓点、调整混音电平、添加混响。

---

## 关键洞察总结

### 评估器是否值得引入

评估器的价值取决于任务与模型当前能力边界的关系：

| 任务难度 | 评估器价值 |
|---------|-----------|
| 任务在模型能力边界内 | 不必要的开销 |
| 任务在模型能力边界处/以外 | 提供真实价值 |

→ **不是固定的 yes/no 决策，而是随模型能力动态调整**

### 规划器的关键约束

- 聚焦产品上下文和高层技术设计
- **避免规定详细的技术实现**（规格中的错误会级联到下游实现）
- 约束"要交付什么"，让 agent 自己找路径

### 调教评估器的现实

Claude 默认是糟糕的 QA agent——早期运行中，评估器会发现合理的问题，然后自己说服自己这不是大问题而批准工作。需要多轮开发循环：
1. 阅读评估器日志
2. 找到其判断与自己判断分歧的例子
3. 更新 QA prompt 解决这些问题

---

## 展望

随着模型持续改进：
- 某些 scaffold 的必要性会降低（如 Context Reset 已不再需要）
- 但模型能力越强，可以构建的 harness 能完成更复杂的任务

**每当新模型发布时的最佳实践**：
1. 重新审视现有 harness
2. 剥离不再必要的组件
3. 增加新组件以实现新模型支持的更强能力

> "有趣的 harness 组合空间不会随模型改善而缩小。它会移动，AI 工程师的有趣工作是持续找到下一个新颖的组合。"

---

## 个人理解

### 最核心的工程原则

**"自评有偏差，他评才有效"**——这条原则超越了 AI，在人类软件开发中就有充分体现（代码审查、QA 独立于开发）。这篇文章最有价值的贡献是用实验证据量化了这个差距有多大。

### 关于 Sprint 契约的设计

Sprint 契约的精妙之处在于：既避免了规格过于详细（级联错误风险），又避免了纯粹执行（可能偏离意图）。两个 agent 在动工前就"对齐"，是一种轻量级的需求澄清机制——对应于 Claude Code 中 plan mode 的作用。

### 技术变化的节奏

这篇文章记录了从 Sonnet 4.5 → Opus 4.5 → Opus 4.6 过程中，相同的 harness 组件如何从"必要"变成"可选"再变成"冗余"。这提示：**阅读 harness 教程时要注意其基于哪个模型**——不同模型版本下的最优设计可能大相径庭。

### 对 Claude Code 用户的启示

`评分标准文档 → 生成器 + 评估器分离` 的模式完全可以在 Claude Code 中手动复现：
- 写一个 REVIEW_CRITERIA.md 定义什么是"好的"
- 让 Claude 生成后，另开一个会话，给它 REVIEW_CRITERIA.md，让它作为评估者审查
- 这比"让它检查自己的工作"效果好得多

## Related

- [[Effective Harnesses for Long-Running Agents]]
- [[Effective Context Engineering for AI Agents]]
- [[Auto Mode：用 AI 分类器替代手动权限审批]]
