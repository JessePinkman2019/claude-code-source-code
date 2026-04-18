---
title: "Claude Code 记忆机制学习路径"
aliases:
  - "learning-tutorial"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Claude Code Memory 优化深度解析"
  - "Claude Code Session Memory 深度解析"
  - "Claude Code 自动记忆：AI 的项目笔记本"
  - "Claude Code 梦境机制：Anthropic 的新记忆功能"
  - "CLAUDE.md 编写指南（基于 v2.1.88 源码）"
  - "Multi-Agent 协作机制（基于 v2.1.88 源码）"
---

# Claude Code 记忆机制学习路径

本文是 Claude Code 记忆机制系列学习文档的索引，按推荐阅读顺序排列。

---

## 推荐学习顺序

### 1. [[Claude Code Memory 优化深度解析]]
**CLAUDE.md 记忆体系**

起点。理解 Claude Code 的五种记忆类型（Managed / User / Project / Local / Rules）、加载顺序与优先级、`@import` 指令、`.claude/rules/` 条件规则目录。这是整个记忆系统的基础架构。

### 2. [[Claude Code Session Memory 深度解析]]
**Session Memory 跨会话自动记忆**

在 CLAUDE.md（手动维护）之外，Claude Code 还有一套自动运行的后台记忆系统。理解触发阈值（10,000 tokens 初始化 / 5,000 tokens 间隔更新 / 3 次 tool call）、summary.md 的 9 个固定 section、与即时 compact 的关系，以及 feature flag 前置条件。

### 3. [[Claude Code 自动记忆：AI 的项目笔记本]]
**Auto Memory Claude 自主笔记**

Claude 在项目目录下维护的自动笔记（`~/.claude/projects/<project>/memory/MEMORY.md`）。与 Session Memory 不同，这是跨 session 的长期项目知识积累，启动时自动加载前 200 行。

### 4. [[Claude Code 梦境机制：Anthropic 的新记忆功能]]
**Auto Dream 后台深度整理**

在上述三层记忆之上，Auto Dream 是 Claude Code 利用空闲时间对记忆进行深度整理和提炼的后台机制。

---

## 其他学习文档

- [[CLAUDE.md 编写指南（基于 v2.1.88 源码）]] — CLAUDE.md 进阶用法与最佳实践
- [[Multi-Agent 协作机制（基于 v2.1.88 源码）]] — 多 Agent 协作机制

---

## 记忆层级总览

```
┌─────────────────────────────────────────────────┐
│  Auto Dream（后台深度整理，对上层记忆做提炼）        │  ← 4
├─────────────────────────────────────────────────┤
│  Auto Memory（MEMORY.md，Claude 自主长期笔记）     │  ← 3
├─────────────────────────────────────────────────┤
│  Session Memory（summary.md，自动跨 session 快照）  │  ← 2
├─────────────────────────────────────────────────┤
│  CLAUDE.md / Rules（用户手动维护的持久化指令）       │  ← 1（基础）
└─────────────────────────────────────────────────┘
```

越底层越基础、越持久；越上层越自动、越动态。

## Related

- [[Claude Code Memory 优化深度解析]]
- [[Claude Code Session Memory 深度解析]]
- [[Claude Code 自动记忆：AI 的项目笔记本]]
- [[Claude Code 梦境机制：Anthropic 的新记忆功能]]
- [[CLAUDE.md 编写指南（基于 v2.1.88 源码）]]
- [[Multi-Agent 协作机制（基于 v2.1.88 源码）]]
