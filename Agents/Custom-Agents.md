---
title: Claude Code Custom Agents
tags:
  - agents
  - claude-code
  - custom-agents
  - slash-commands
  - configuration
source: https://claudefa.st/blog/guide/agents/custom-agents
date: 2026-04-27
---

# Claude Code Custom Agents

> [!tip] 2 分钟快速上手
> ```bash
> mkdir -p .claude/commands
> # 创建 .claude/commands/code-review.md，写入 prompt 内容
> # 用 /project:code-review 调用
> ```

---

## 三种自定义方式

| 方式 | 路径 | 触发 | 适用场景 |
|------|------|------|----------|
| **Slash Commands** | `.claude/commands/` | 手动 `/project:cmd` | 可复用的工作流 prompt |
| **Agent Definitions** | `.claude/agents/` | Orchestrator 自动 spawn | 持久化 sub-agent 身份 |
| **CLAUDE.md 规则** | `CLAUDE.md` | 始终生效 | 全局行为约束 |

**Commands vs Agents**：Slash commands 是你手动拿起的工具；agent definitions 是 orchestrator 可随时调用的团队成员。

**作用域**：
- `.claude/commands/` / `.claude/agents/` — 项目级，via git 共享
- `~/.claude/commands/` / `~/.claude/agents/` — 用户级，跨项目个人使用

---

## Slash Commands

### 基础示例

`.claude/commands/security-audit.md`：
```
You are a security expert. Scan this codebase for vulnerabilities.

Check for:
- SQL injection vulnerabilities
- XSS attack vectors
- Authentication bypass issues
- Exposed API keys or secrets
- OWASP Top 10 violations

For each finding, provide:
1. Vulnerability description
2. Risk level (Critical/High/Medium/Low)
3. Specific fix recommendation
4. Code example of the fix

Start by searching for common vulnerability patterns in the codebase.
```

调用：`/project:security-audit`

### 动态参数

`.claude/commands/review-file.md`：
```
Review this specific file for issues: $ARGUMENTS

Focus on: code quality, potential bugs, security concerns.
```

调用：`/project:review-file src/auth/login.ts`

---

## CLAUDE.md：始终生效的行为

```markdown
## Code Review Standards

When reviewing code, always check:
- All functions have error handling
- No console.log statements in production code
- API endpoints validate input parameters
- Database queries use parameterized statements

## Commit Message Format

Use conventional commits: type(scope): description
Types: feat, fix, docs, refactor, test, chore
```

---

## .claude/agents/ 持久化 Sub-Agent 定义

与 slash commands 不同，`.claude/agents/` 中的定义可被 Claude 的 Task tool 在编排时**自动** spawn。

### YAML Frontmatter 字段

```yaml
---
name: "security-auditor"       # 必填，Task 调用时的标识
model: "claude-sonnet-4-5"     # 可选，覆盖默认模型
allowedTools:                   # 可选，限制工具访问
  - Read
  - Grep
  - Bash
description: "Scans code for vulnerabilities and OWASP violations"
---
```

Frontmatter 下方是普通 Markdown 格式的系统指令，与 slash command 写法相同。

### 真实 Agent 定义示例

`.claude/agents/quality-engineer.md`：
```markdown
---
name: "quality-engineer"
allowedTools: ["Read", "Grep", "Bash"]
description: "Validates code against project standards before commits"
---

You are a quality engineer. Your job is to validate, never to modify.

## Validation Protocol

1. Read the files that changed (use git diff --name-only)
2. For each file, check:
   - Imports resolve correctly
   - No console.log/print statements in production code
   - Functions under 50 lines
   - All exported functions have JSDoc or docstring
3. Run the test suite: `npm test -- --bail`
4. Report pass/fail with specific line numbers for failures

Never edit files. Never suggest fixes. Only report what you find.
```

> [!important] Tool 限制的价值
> `allowedTools` 排除了 `Edit` 和 `Write`，该 agent **物理上无法**修改代码，只能检查。这是 validation-only agent 的关键约束。

Agent definitions 自动继承项目的 CLAUDE.md，无需重复编写编码规范。

---

## 常用 Command 示例

**数据库优化器**（`.claude/commands/db-optimize.md`）：
```
You are a database performance expert.

Analyze the database queries in this codebase:
1. Find slow or inefficient queries
2. Check for missing indexes
3. Review schema design

For each issue, provide:
- The problematic query or schema
- Why it's a problem
- Optimized version with explanation
```

**文档撰写**（`.claude/commands/write-docs.md`）：
```
Write documentation for: $ARGUMENTS

Include:
- Purpose and overview
- Setup instructions
- Usage examples
- Common troubleshooting

Target audience: developers new to this project.
```

---

## 常见错误

| 错误 | 问题 | 修复 |
|------|------|------|
| 范围太宽 | "review all issues" → 表面观察 | 缩窄任务，专注一个维度 |
| 无输出格式约束 | 每次返回格式不一致 | 明确指定："For each finding, provide: file path, line number, severity, fix" |
| review agent 有写权限 | 开始"顺手修复"而非报告 | `allowedTools` 限制为 `Read/Grep/Bash` |
| 重复 CLAUDE.md 内容 | 浪费上下文 token | Command 只加特定性，不重复基线规则 |

---

## 何时用哪种方式

| 场景 | 最佳方式 |
|------|----------|
| 每周重复 3 次的工作流 | Slash command |
| Orchestrator 自动 spawn 的专家 | `.claude/agents/` |
| 所有 session 都需要遵守的规则 | CLAUDE.md |
| 复杂领域工作流（支付、部署） | Skill |
| 一次性临时视角 | 直接 prompt |

**经验法则**：同样详细的 prompt 打了 3 次以上 → 存为 slash command；Orchestrator 需要自动调用 → 存为 agent definition。

---

## 延伸阅读

- [[Agent-Fundamentals]] — Sub-agent 基础与 Task tool
- [[Agent-Patterns]] — 六种编排模式
- [[Team-Orchestration]] — Builder-Validator 工作流
