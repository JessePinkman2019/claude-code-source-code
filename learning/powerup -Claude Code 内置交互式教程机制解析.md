---
title: "`/powerup`：Claude Code 内置交互式教程机制解析"
aliases:
  - "learning-powerup"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考"
  - "Voice Mode：对着终端说话"
---

# `/powerup`：Claude Code 内置交互式教程机制解析

## 背景：被遗忘的 4 月 1 日发布

2026 年 4 月 1 日，Anthropic 在 22 小时内连发两个版本：

- **v2.1.89**（01:07 UTC）：`/buddy` —— 终端 Tamagotchi 宠物，18 个物种、稀有度分级、帽子皮肤。社区爆了，甚至有人基于最稀有变体发了 Solana meme 币。
- **v2.1.90**（23:41 UTC）：`/powerup` —— 带动画演示的交互式功能教程。零社区报道，零 Twitter 热点。

愚人节文化造就了这个对比：玩具吸走所有眼球，实用工具悄然落地。

---

## `/powerup` 是什么

官方 changelog 只有一行：

> **Added `/powerup` — interactive lessons teaching Claude Code features with animated demos**

官方文档页面截至 4 月 2 日甚至还没收录这个命令。

本质是：**Claude Code 第一个官方第一方、纯终端内的学习系统**。输入 `/powerup`，不会跳转浏览器，而是在当前终端会话内展示带动画演示的交互式课程。

---

## 它解决了什么问题

Claude Code 的功能远超大多数用户发现的范围：Hooks、Sub-agents、Plan 模式、Rewind、Worktrees、Skills、MCP server、Cloud Tasks……每周还在增加新能力，而普通用户可能只触及了 20%。

**问题不是难度，是可见性。**

在 `/powerup` 之前，学习 Claude Code 的资源一览：

| 资源 | 位置 | 费用 |
|------|------|------|
| Anthropic Skilljar | 外部浏览器 | 免费 |
| claude.nagdy.me | 外部浏览器 | 免费 |
| CC for Everyone | 外部浏览器 | $20/月 |
| Coursera（范德堡）| 外部浏览器 | ~$50 |
| 官方 `/help` | 终端内 | 免费 |

规律：除 `/help` 外，所有学习资源都需要**离开终端**。而 `/help` 只列命令，不解释原理，更不演示。

`/powerup` 打破这个模式：**不离开终端，用动画展示而非静态文字描述。**

---

## 技术实现原理

Claude Code 的终端 UI 基于 **React + Ink** 框架（v2.1.88 源码泄露已确认，512,000 行 TypeScript 代码库）。Ink 将 React 组件渲染为终端输出。

`/powerup` 的"动画演示"本质上是 Ink 组件：
- 按键序列逐字出现
- 命令输出渐进渲染
- 功能演示如同现场操作

交互方式与 `/model`、`/config` 一致：**方向键 + 回车**，全键盘驱动。

---

## 课程可能覆盖的内容

Anthropic 未公布课程表，但根据功能特性推断：

**初级**
- Context 管理（`/clear`、`/compact`）
- CLAUDE.md 记忆系统
- Plan 模式（`Shift+Tab` 切换）
- 模型选择

**中级**
- Skills 和自定义命令系统
- Hooks（PreToolUse、PostToolUse）
- Sub-agent 编排
- MCP server 配置
- `/rewind` 与 checkpoint

**高级**
- Worktrees 与并行会话
- Auto 模式与权限管理
- `/schedule` 与 Cloud Tasks
- SDK 与 headless 模式

动画格式特别适合展示**难以用文字描述的工作流**：`/rewind` 实际看起来是什么样？切换 Plan 模式时视觉上发生了什么？Sub-agent 如何 spawn 并返回结果？

---

## v2.1.90 其他重要更新

`/powerup` 之外，这个版本还有实质性修复：

**关键 Bug 修复**
- 修复 rate-limit 弹窗循环崩溃会话的无限循环
- 修复 `--resume` 导致 prompt cache 完全失效（v2.1.69 引入的回归，影响所有使用 MCP 或 deferred tools 的用户）
- 修复 auto 模式忽略"不要 push"等明确边界指令
- 修复 PostToolUse format-on-save hook 改写文件后 `Edit`/`Write` 报"文件内容已更改"

**性能优化**
- 消除每轮 MCP tool schema 的 `JSON.stringify` 开销
- SSE transport 大帧处理从 O(n²) 优化为 O(n)
- 长对话 SDK session 的 transcript 写入不再二次方级别降速
- `/resume` 并行加载项目 session

**安全加固**
- 修复多个 PowerShell 权限绕过漏洞（`&` 后台任务绕过、`-ErrorAction Break` 调试器挂起、archive 解压 TOCTOU 漏洞）
- 从自动允许列表移除 DNS 缓存查询命令（隐私保护）

---

## 产品信号：Anthropic 在做什么

为 CLI 工具内置教程系统，是**产品成熟度信号**。

早期工具假设用户会自己摸索。当工具开始在产品内部添加结构化 onboarding，意味着公司在为**下一批用户**优化，而不只是早期玩家。

Claude Code 社区已经在外部建了大量学习资源：交互式浏览器教程、收费课程、GitHub 图解仓库。这些证明了 onboarding 需求真实且迫切。

`/powerup` 是 Anthropic 的回应：**我们自己来负责降低学习曲线，不外包给社区。**

与源码泄露事件形成哲学对照：开发者不应该需要反编译源码来发现功能。

---

## 如何使用

更新到 v2.1.90 或更高版本：

```bash
npm update -g @anthropic-ai/claude-code
```

然后运行：

```
/powerup
```

如果命令不存在，用 `/stats` 或 `claude --version` 检查版本。该功能可能与 `/buddy` 类似，有 Pro/Max 计划限制（Anthropic 尚未确认所有计划的可用性）。

---

## 一句话结论

`/buddy` 有趣一天，`/powerup` 有用一辈子。4 月 1 日那天没人谈论的功能，可能是六个月后你最常用的那个。

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/claude-powerup

## Related

- [[Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考]]
- [[Voice Mode：对着终端说话]]
