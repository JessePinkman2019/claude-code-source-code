---
title: "April 23 Postmortem"
aliases:
  - "april-23-postmortem"
  - "Claude Code April 23 Postmortem"
area: engineering
tags:
  - claude-code
  - postmortem
  - evals
  - context-engineering
status: evergreen
source: "https://www.anthropic.com/engineering/april-23-postmortem"
source_type: engineering-article
date: 2026-04-27
related:
  - "Effective Context Engineering for AI Agents"
  - "Harness Design for Long-Running Application Development"
  - "Demystifying Evals for AI Agents"
---

# April 23 Postmortem

> 原文：https://www.anthropic.com/engineering/april-23-postmortem  
> 主题：Claude Code / Claude Agent SDK / Claude Cowork 近期质量退化问题复盘。API 与推理层未受影响。

---

## 一句话概括

Anthropic 将部分用户感知到的 Claude 质量退化归因于三个相互独立但时间上重叠的产品层变更：默认 reasoning effort 下调、旧 thinking 清理逻辑 bug、以及过强的“降低 verbosity”系统提示；这些问题暴露出 Claude Code 这类 agentic harness 对 effort、context、system prompt 与 eval 覆盖面的高度敏感。

---

## 影响范围

受影响产品：

- Claude Code
- Claude Agent SDK
- Claude Cowork

未受影响：

- Anthropic API
- 推理层

已修复版本：

- 2026-04-20，v2.1.116 起全部修复

---

## 三个根因

### 1. 默认 reasoning effort 从 high 改为 medium

时间线：

- 2026-03-04：Claude Code 默认 reasoning effort 从 `high` 改为 `medium`
- 2026-04-07：回滚该决策

背景：

- Opus 4.6 在 high effort 下偶发非常长的 thinking latency
- 用户看到 UI 像卡死，同时 token / usage limit 消耗偏高
- 内部 eval 显示 medium effort 在大多数任务上智能略低但 latency 明显更好

问题：

- 这是错误 tradeoff
- Claude Code 的核心用户更愿意默认更高智能，再主动为简单任务选择低 effort
- 用户虽然能通过 `/effort` 修改，但多数用户保留了默认 medium

修复：

- Opus 4.7 默认 `xhigh`
- 其他模型默认 `high`

启发：

- 对 coding agent 来说，默认值是产品行为的一部分，不只是性能参数
- intelligence / latency / usage limit 的 tradeoff 不能只看平均任务
- 对高价值复杂任务，用户对智能下降的敏感度高于对 latency 的敏感度

---

### 2. 缓存优化错误清理历史 thinking

时间线：

- 2026-03-26：上线 idle session thinking 清理优化
- 2026-04-10：在 v2.1.101 修复

原始设计：

- Claude reasoning 通常会保留在 conversation history 中
- 这样后续 turn 能看到之前为什么做了某些 edits / tool calls
- 对 idle 超过 1 小时的 session，由于 prompt cache 可能已经 miss，原计划只在恢复时清理一次旧 thinking，减少 uncached token 成本
- 使用 `clear_thinking_20251015` API header 与 `keep:1`

实际 bug：

- thinking 不是只清理一次，而是在该 session 后续每一轮都继续清理
- 一旦 session 跨过 idle threshold，后续请求都会只保留最近一个 reasoning block
- 如果用户在 tool use 中途追加消息，当前 turn 的 reasoning 也可能被清掉

用户感知：

- Claude 变得健忘
- 重复执行类似动作
- 工具选择异常
- usage limit 消耗更快

为什么难以发现：

- 只发生在 stale session 角落场景
- 与 message queue 内部实验、thinking 展示变化叠加，导致复现困难
- 通过了人工 review、自动 review、单测、端到端测试、自动验证和 dogfooding

后续改进：

- Anthropic 用 Opus 4.7 回测 offending PR；在提供完整相关仓库上下文后，Opus 4.7 找到了该 bug，而 Opus 4.6 没有
- Code Review 将支持更多 repo 作为 review context

与 [[Effective Context Engineering for AI Agents]] 的关系：

- thinking history 是 agent 的关键上下文资产
- context pruning 必须是精确、可验证、可恢复的
- 错误清理 reasoning 会破坏 agent 的行动连续性，即使模型本身没有变差

---

### 3. 降低 verbosity 的系统提示伤害 coding 质量

时间线：

- 2026-04-16：随 Opus 4.7 发布上线系统提示变更
- 2026-04-20：回滚

触发背景：

- Opus 4.7 相比前代更 verbose
- verbosity 往往伴随更强的问题解决能力，但也带来更多输出 token
- Anthropic 同时使用训练、prompting、thinking UX 来降低 verbosity

有问题的系统提示：

> “Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.”

问题：

- 内部测试与当时 eval 未发现回归
- 后续更广 eval + system prompt ablation 显示，对 Opus 4.6 和 Opus 4.7 都带来约 3% drop
- 简短输出约束与其他 prompt 变化组合后，伤害 coding quality

启发：

- coding agent 的“少说话”不能简单等同于“更高效”
- 对复杂任务，必要的 reasoning handoff、状态解释、工具间沟通和最终总结都可能承载任务质量
- system prompt 每一行都可能对模型行为产生非线性影响，需要逐行 ablation

---

## 为什么整体看起来像“广泛退化”

三个问题影响范围、模型和时间线不同：

| 问题 | 时间 | 影响模型 / 产品 | 用户感知 |
|------|------|-----------------|----------|
| effort 默认下调 | 03-04 至 04-07 | Sonnet 4.6 / Opus 4.6 | 变笨、思考不深 |
| thinking 清理 bug | 03-26 至 04-10 | Sonnet 4.6 / Opus 4.6 | 健忘、重复、工具选择异常 |
| verbosity prompt | 04-16 至 04-20 | Sonnet 4.6 / Opus 4.6 / Opus 4.7 | coding 质量下降 |

这些变化分别作用于不同流量切片，因此聚合反馈表现为“广泛但不稳定的质量下降”。

---

## 工程改进

Anthropic 后续将采取：

1. 让更多内部员工使用与公开用户完全一致的 Claude Code build，而不是内部测试 build
2. 改进内部 Code Review 工具，并对外发布改进版
3. 对 Claude Code system prompt 变更加 tighter controls
4. 每次 system prompt 变更都运行更广的 per-model eval suite
5. 继续做 prompt ablation，理解每一行的影响
6. 建立更容易 review / audit prompt change 的新工具
7. 在 CLAUDE.md 中加入 guidance，确保 model-specific changes 只 gate 到目标模型
8. 对任何可能牺牲 intelligence 的变更增加 soak period、更广 eval、渐进 rollout

与 [[Demystifying Evals for AI Agents]] 的关系：

- eval 覆盖面不足时，产品层看似小的变更可能绕过测试
- agent eval 不只要测最终答案，还要覆盖长期会话、stale session、tool use 中断、cache miss 等动态路径

---

## 对 Claude Code / Agent Harness 的启发

### 1. Harness 参数也是模型行为的一部分

模型没变，产品层 effort 默认值变化也会改变用户感知到的智能。

对自己的 harness 设计而言：

- 默认 effort / thinking budget 要匹配任务类型
- 简单任务可降 effort，复杂工程任务应默认高 effort
- effort 是任务策略的一部分，不是纯性能优化项

### 2. Context pruning 必须可验证

thinking 清理 bug 本质是 context engineering 事故。

对于 long-running agent：

- pruning 不能只看 token 成本
- 必须验证 agent 是否仍能解释自己的前序决策
- session log / reasoning history / tool trace 的删除策略需要有回归测试

### 3. System prompt 是高风险配置

“限制输出长度”这种看似 harmless 的系统提示，也可能降低 coding intelligence。

对于 [[Harness Design for Long-Running Application Development]]：

- generator、evaluator、planner 的 prompt 应单独 ablation
- 不能只测 happy path
- prompt diff 应像代码 diff 一样 review

### 4. Public build dogfooding 很重要

内部 build 与外部 build 不一致，会让真实用户问题难以复现。

这对应 agent 产品的一个通用原则：

- dogfooding 必须覆盖真实配置、真实权限、真实缓存、真实发布路径
- 否则内部 eval 很容易测的是“另一个系统”

---

## 关键 takeaways

- Claude Code 的质量由模型、effort、context、system prompt、cache、tool use 与 product harness 共同决定。
- 用户感知的“模型变笨”未必来自模型权重或推理层，也可能来自默认 effort、上下文管理或系统提示。
- 对 agentic coding 产品，thinking history 是关键状态，错误 pruning 会直接破坏任务连续性。
- system prompt 变更需要像代码一样做 review、ablation、soak 和 per-model eval。
- eval 必须覆盖长期会话、idle resume、tool use 中断、cache miss 等真实 agent 路径。
- 默认更高 intelligence，再允许用户 opt into 更低 effort，可能更符合 Claude Code 核心用户预期。

---

## 相关笔记

- [[Effective Context Engineering for AI Agents]]
- [[Harness Design for Long-Running Application Development]]
- [[Demystifying Evals for AI Agents]]
