---
tags:
  - agents
  - claude-code
  - sub-agents
  - orchestration
  - design-patterns
source: https://claudefa.st/blog/guide/agents/sub-agent-design
date: 2026-04-21
---

# Sub-Agent Design

> Master sub-agent design in Claude Code to handle complex multi-faceted projects. Learn to orchestrate specialized agents for 10x productivity.

## 为什么单 Agent 分析会失败

让 Claude 同时审查代码、优化性能、检查安全时，得到的是泛泛建议。每个视角在同一上下文窗口中争夺注意力，导致所有领域的分析都流于浅表。

Sub-agent 通过创建专业化上下文解决这一问题——每个 agent 聚焦其专业领域，使用不同工具，再汇总发现进行综合分析。

## 两种 Sub-Agent 方式

1. **Task Tool**：派生拥有独立上下文窗口的隔离 sub-agent，实现真正的并行执行
2. **Perspective Prompting**：在单会话内显式请求多专家视角，模拟专家思维

**快速上手**：

```
Create sub-agents and analyze this from these perspectives:
- Senior engineer: Review architecture decisions
- Security expert: Identify vulnerabilities
- Performance reviewer: Find optimization opportunities
```

## 适合 Sub-Agent 的任务

> [!tip] 适合并行
> - 从多视角进行代码审查
> - 跨技术的研究任务
> - 面向不同受众的文档审查
> - 跨维度的性能分析

> [!warning] 避免并行
> - 有依赖关系的文件修改
> - 串行构建流程
> - 数据库迁移

## 设计专家角色

**代码质量审查：**

```
Analyze this codebase using sub-agents with these specialist roles:
- Factual reviewer: Check technical accuracy against documentation
- Senior engineer: Review architecture decisions and patterns
- Security expert: Identify vulnerabilities and attack vectors
- Consistency reviewer: Check coding standards compliance
- Redundancy checker: Find duplicate logic to consolidate
```

**用户体验分析：**

```
Create sub-agents for UX review of this feature:
- Creative thinker: Suggest innovative interaction solutions
- Beginner user: Test ease of use and onboarding friction
- Designer: Evaluate visual hierarchy and spacing
- Accessibility auditor: Check WCAG compliance
```

## 实施策略

### Plan Mode 安全分析

在关键代码上启动 sub-agent 前，按 **Shift+Tab 两次** 进入 plan mode，确保 sub-agent 只分析不修改。

```
Use sub-agents to validate this API design from:
- Backend perspective: Data flow and scalability
- Frontend perspective: Consumption patterns and DX
- Security perspective: Authentication and authorization gaps
```

### 合并模式（Consolidation Pattern）

Sub-agent 完成后汇总发现：

1. **独立报告**：每个 agent 记录各自发现
2. **冲突解决**：处理相互矛盾的建议
3. **优先级排序**：按影响力排序
4. **行动计划**：创建逐步实施方案

## 常见错误

> [!danger] 错误：用 sub-agent 处理简单任务
> 不要为拼写错误或单行修改派生 sub-agent，开销会抵消所有收益。

> [!danger] 错误：创建过多重叠角色
> 避免"安全专家、渗透测试员、漏洞扫描器、安全架构师"——这些视角高度重叠。

**正确做法**：选择覆盖问题不同维度的互补视角：

```
Use sub-agents to redesign this authentication system:
- Security expert: Audit current vulnerabilities
- UX designer: Simplify the login flow
- Performance engineer: Optimize token handling
```

## 后台执行

Sub-agent 启动后按 `Ctrl+B` 移至后台，继续处理其他任务，详见 [[Async-Workflows]]。

## 成本优化

并行 sub-agent 能在 Sonnet 成本下达到接近 Opus 的分析质量：

- 多专家视角在单会话内完成
- 并行处理减少总耗时
- 专业化上下文避免浅层分析
- 后台执行消除等待

## 高级编排模式

### 角色轮换策略

大型项目中，跨组件轮换 sub-agent 视角：

**第 1 周——核心架构：**
```
Analyze the database design using sub-agents:
- Data architect: Schema optimization and normalization
- Security expert: Access control and encryption
- Performance optimizer: Query patterns and indexing
```

**第 2 周——API 层：**
```
Review API endpoints with these specialist sub-agents:
- Backend engineer: Implementation quality and patterns
- Documentation writer: API clarity and examples
- Integration specialist: Third-party compatibility
```

### 迭代精炼

1. 广视角初步分析
2. 专家角色深度分析
3. 用户视角最终审查

## Sub-Agent 的最佳场景

| 场景 | 价值 |
| --- | --- |
| 架构审查 | 多技术视角 |
| 文档审计 | 不同受众视角 |
| 代码质量门控 | 多质量维度 |
| 产品策略 | 跨职能洞察 |
| 竞品分析 | 不同市场角度 |

## 相关笔记

- [[Agent Fundamentals]]
- [[Sub-Agent-Best-Practices]]
- [[Async-Workflows]]
- [[Custom Agents]]
- [[Agent Patterns]]
