---
tags:
  - claude-code
  - best-practices
  - agentic
  - workflow
source: https://claudefa.st/blog/guide/development/agentic-engineering-best-practices
date: 2026-04-16
related:
  - "[[opus-4-7-best-practices]]"
---

# Claude Code Best Practices: 5 Agentic Engineering Techniques

#claude-code #best-practices #agentic #workflow

> 原文：https://claudefa.st/blog/guide/development/agentic-engineering-best-practices  
> 发布时间：2026-04-16

---

## 核心观点

大多数开发者把 Claude Code 当聊天机器人用——描述需求两句话，得到模糊结果，然后怪工具。

获得 10x 结果的开发者做了不一样的事：**他们围绕 Claude Code 建立了系统**，让每次 session 都比上一次更高效。

---

## 1. PRD 优先开发（PRD-First Development）

#prd #context-management

**最大的错误**：没有计划就开始 agentic 编码。没有 PRD 会导致 context drift——Claude 开始做你从未讨论过的架构决策。

PRD 不需要是 50 页的企业规格书，一个 5 分钟写完的轻量 markdown 文件就够：

```markdown
# Feature: User Authentication

## Mission
Add email/password authentication with session management.

## In Scope
- Sign up, login, logout flows
- Password hashing with bcrypt
- JWT session tokens with 24-hour expiry
- Protected route middleware

## Out of Scope
- OAuth providers (Phase 2)
- Two-factor authentication (Phase 3)

## Architecture
- Auth routes: /api/auth/signup, /api/auth/login, /api/auth/logout
- Middleware: src/middleware/auth.ts
- Database: users table with email, password_hash, created_at
```

**PRD 的作用**：
- 每次 session 都有明确边界，告诉 Claude 什么在范围内、什么不在
- 解决多 session 问题——Claude Code 不携带跨 session 记忆，但 PRD 可以；每次新 session 读同一份文档，从上次结束的地方继续
- 适用于新项目（定义 MVP）和已有项目（记录现状 + 指定下一步）

---

## 2. 模块化规则架构（Modular Rules Architecture）

#claude-md #context-engineering

**最常见错误**：把所有内容塞进一个 CLAUDE.md——技术栈、编码规范、测试规则、部署流程全在一起。

问题是**context 浪费**：Claude 在 session 开始时加载整个 CLAUDE.md，每个 token 都在争夺注意力。调试数据库迁移时，React 模式也被加载进来。

**更好的方案**：保持 CLAUDE.md 轻量，指向任务专属文档。

```markdown
# CLAUDE.md（约 15 行）

## Tech Stack
- Next.js 15 with App Router
- TypeScript strict mode
- PostgreSQL with Prisma ORM

## Standards
- Use path aliases (@/components, @/lib, @/utils)
- All functions require explicit return types
- Error handling: guard clauses with early returns

## Reference Docs (load when relevant)
- Frontend: .claude/skills/react/SKILL.md
- API patterns: .claude/skills/api/SKILL.md
- Database: .claude/skills/postgres/SKILL.md
- Deployment: .claude/skills/infra/SKILL.md
```

**目录结构**：
```
.claude/
├── skills/
│   ├── react/       # 组件模式、hooks、状态
│   ├── api/         # 路由规范、验证、认证
│   ├── postgres/    # Schema 模式、查询优化
│   └── infra/       # Docker、CI/CD、部署
└── CLAUDE.md        # 轻量全局规则
```

域知识按需加载，而非每次 session 全部加载。

---

## 3. 将一切命令化（Command-ify Everything）

#slash-commands #automation

**原则**：如果某件事你 prompt 了两次，它就应该成为一个命令。

命令是 `.claude/commands/` 中的 markdown 文件，定义可复用工作流。输入 `/commit`，Claude 读取命令文件并执行。

**示例命令**：
```markdown
# /commit

Review all staged changes with `git diff --cached`.
Write a commit message that:
- Starts with a verb (add, fix, update, remove)
- Summarizes the WHY, not the WHAT
- Stays under 72 characters
- Uses lowercase

Create the commit. Do not push.
```

**5 个值得创建的入门命令**：
- `/commit` — 规范化 git commit
- `/review` — 按项目标准做代码审查
- `/plan` — 写代码前生成结构化实施计划
- `/prime` — 每次对话开始时加载 session 上下文
- `/execute` — 执行上一 session 创建的计划文档

每个命令花 5 分钟写，在项目生命周期内节省数百次 prompt，并强制保持一致性。

---

## 4. Context 重置（The Context Reset）

#context-management #session

**反直觉但重要**：规划和执行应该在**独立的对话**中进行。

**流程**：
1. **规划 session**：研究问题、讨论权衡、探索方案。Claude 输出结构化计划文档（保存为项目中的 markdown 文件）
2. **清除 context**：完全退出对话，终止 session
3. **执行 session**：重新开始。只喂给 Claude 步骤 1 的计划文档，其他什么都不给

**为什么这样做**：

长时间规划对话后，Claude 的 context 充满了探索性的岔路、被否决的方案和已不适用的中间推理。继续说"好，现在开始构建"时，Claude 会携带这些噪声进入执行阶段。

全新 context + 计划文档 = Claude 以干净的心智模型开始执行，没有规划阶段的包袱，也有更多 token 空间用于推理实现细节和自我验证。

**实际操作**：
```bash
# 规划 session
claude "Research auth patterns for our Next.js app and create
an implementation plan. Save it to docs/auth-plan.md"

# （退出，开始新 session）

# 执行 session
claude "Read docs/auth-plan.md and implement Phase 1"
```

计划文档必须足够完整，能够独立使用。好的计划包含：功能描述、用户故事、架构上下文、相关组件引用、逐任务分解。如果 Claude 在执行时需要问澄清问题，说明计划有缺口。

---

## 5. 系统演化思维（System Evolution Mindset）

#system-thinking #continuous-improvement

**核心区别**：每个 bug 都是系统失败，不是一次性错误。

好的 agentic 工程师修复系统，而不只是修复单个实例。

**三个真实案例**：

- **错误的导入风格**：Claude 一直用相对路径而非 path alias。解决：在 CLAUDE.md 加一行规则，永不再发生
- **忘记跑测试**：每次实现完 push 到 CI 才发现测试失败。解决：更新 `/execute` 命令模板，强制在每次执行结尾跑测试
- **错误的认证流程**：Claude 用 cookie 实现 JWT，但项目用 header bearer token。解决：创建 `.claude/skills/auth/SKILL.md` 记录 token 格式、header 规范和中间件模式

**固化这个习惯**：每个功能完成后，让 Claude 审查规则和命令：

> "Read CLAUDE.md and the commands we used. What rules or process changes would have prevented the issues we hit?"

这把系统演化从事后想法变成常规步骤。

---

## 五个实践的系统关系

这五个实践不是独立的技巧，它们构成一个系统：

- **PRD** 限定工作范围
- **模块化规则**保持 context 干净
- **命令化**消除重复
- **Context 重置**保持 session 清晰
- **系统演化**让一切持续变好

从解决你当前最大痛点的那个实践开始，逐步添加其余的。复利效应会随着每个项目积累。
