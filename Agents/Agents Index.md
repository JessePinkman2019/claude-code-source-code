---
title: Agents Index
aliases:
  - agents-index
area: agents
tags:
  - moc
  - agents
  - claude-code
status: evergreen
source_type: index-note
---

# Agents Index

Claude Code Agent 系统全体系笔记索引，覆盖从基础概念到高级编排的完整知识链。

## 基础概念

- [[Agent-Fundamentals]] — Sub-agent 原理、Task tool、并行执行基础
- [[Custom-Agents]] — Slash commands、`.claude/agents/` YAML 定义、allowedTools 限制
- [[Human-Like-Agents]] — CLAUDE.md personality 注入，让 agent 像资深开发者一样思考

## 编排模式

- [[Agent-Patterns]] — 六种编排策略：Orchestrator / Fan-Out / Validation Chain / Specialist Routing / Progressive Refinement / Watchdog
- [[Team-Orchestration]] — Builder-Validator 配对模式，`addBlockedBy` 依赖链，Meta-Prompt 自动生成计划
- [[Sub-Agent-Best-Practices]] — 并行 vs 顺序路由决策
- [[Sub-Agent-Design]] — Sub-agent 架构设计原则
- [[Task-Distribution]] — 多 agent 任务分发策略
- [[Async-Workflows]] — 异步工作流模式

## Agent Teams（原生多 Agent 协作）

- [[Agent-Teams]] — 总览：启用方式、与 Subagents 的核心区别、架构组成
- [[Agent-Teams-Controls]] — 显示模式、Delegate Mode、Plan Approval、Hooks、Token 成本
- [[Agent-Teams-Use-Cases]] — 10+ 场景即用 prompt（代码审查、Debug、全栈实现、架构决策…）
- [[Agent-Teams-Best-Practices]] — 最佳实践、Troubleshooting 速查表、版本修复记录
- [[Agent-Teams-Workflow]] — 端到端 7 步生产构建流程（Brain Dump → 契约链 → 波次执行）

## 知识地图

```
Agent 基础
  └─ Agent-Fundamentals
       ├─ Sub-Agent-Design
       ├─ Sub-Agent-Best-Practices
       ├─ Task-Distribution
       └─ Async-Workflows

编排模式
  ├─ Agent-Patterns（六种通用模式）
  ├─ Team-Orchestration（Builder-Validator）
  └─ Custom-Agents（slash commands & agent defs）

Agent Teams（实验性原生功能）
  ├─ Agent-Teams（入口）
  ├─ Agent-Teams-Controls（配置）
  ├─ Agent-Teams-Use-Cases（场景）
  ├─ Agent-Teams-Best-Practices（实战）
  └─ Agent-Teams-Workflow（完整流程）

辅助
  └─ Human-Like-Agents（personality 注入）
```
