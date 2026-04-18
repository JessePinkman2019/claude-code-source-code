---
title: "Claude Code 自动记忆：AI 的项目笔记本"
aliases:
  - "learning-auto-memory"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Claude Code Session Memory 深度解析"
  - "Claude Code 梦境机制：Anthropic 的新记忆功能"
  - "CLAUDE.md 编写指南（基于 v2.1.88 源码）"
---

# Claude Code 自动记忆：AI 的项目笔记本

Claude Code 自动记忆功能让 Claude 自己给项目写笔记。了解它的工作原理、文件位置，以及何时用它 vs CLAUDE.md。

**问题所在：** 你在 CLAUDE.md 里写好了项目规则，但 Claude 仍然反复问同样的问题：构建命令、测试约定、调试技巧。你的指令涵盖了你想让 Claude 怎么做，但没有涵盖 Claude 已经学到的代码库知识。

**快速检查：** 在任意 Claude Code session 中运行 `/memory`，你会看到一个选择器，展示你的 CLAUDE.md 文件和自动记忆开关。如果开关是打开的（默认就是打开的），Claude 一直在后台给你的项目做笔记。

```bash
# 查找项目的自动记忆目录
ls ~/.claude/projects/
```

如果你看到目录，Claude 已经有笔记了。如果记忆文件经过多次 session 后变得杂乱，可以了解 [[Claude Code 梦境机制：Anthropic 的新记忆功能]]——它会周期性地清理和重组自动记忆写入的所有内容。

---

## 自动记忆到底是什么

自动记忆是一个持久化目录，Claude 在工作过程中把学到的知识、发现的模式和洞察记录在这里。核心区别：**CLAUDE.md 是你给 Claude 的指令，MEMORY.md 是 Claude 关于你项目的笔记本。**

你写 CLAUDE.md 来告诉 Claude 诸如"用 pnpm 不用 npm"或"总是先写测试再实现"之类的事。自动记忆则是 Claude 给自己做笔记的地方："构建命令是 `pnpm build`，测试在 `__tests__/`，API 用 Express，中间件在 `src/middleware/`"。

这和 Session Memory 不同——Session Memory 保存对话级别的摘要用于跨 session 回忆。自动记忆在不同层面运作，它捕获的是持久的项目知识，无论哪次对话产生的都会保留。

---

## 三大记忆系统对比

Claude Code 现在有三种独立的记忆系统，理解各自适用场景能避免重复劳动。

| 维度 | **CLAUDE.md** | **Auto Memory** | **Session Memory** |
|---|---|---|---|
| 谁来写 | 你 | Claude | Claude |
| 内容 | 指令与规则 | 项目模式与经验 | 对话摘要 |
| 范围 | 项目级或全局 | 项目级 | session 级 |
| 启动时加载 | 完整文件 | MEMORY.md 前 200 行 | 相关历史 session |
| 优先级 | 高（视为指令） | 后台参考 | 后台参考 |
| 存储位置 | `./CLAUDE.md` 或 `~/.claude/CLAUDE.md` | `~/.claude/projects/<project>/memory/` | `~/.claude/projects/<project>/<session>/session-memory/` |
| 最适合 | 规范、架构决策、命令 | 构建模式、调试洞察、偏好 | 工作 session 之间的连续性 |
| 团队共享 | 是（通过 git） | 否（仅本地） | 否（仅本地） |

最强的配置是三者结合使用：CLAUDE.md 提供权威规则，自动记忆捕获 Claude 工作中学到的内容，Session Memory 维护对话连续性。

---

## 自动记忆存在哪里

每个项目基于 git 仓库根目录获得独立的记忆目录：

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 主索引，每个 session 都会加载
├── debugging.md       # 详细调试模式
├── api-conventions.md # API 设计决策
└── ...                # Claude 创建的任何主题文件
```

几个重要的存储细节：

- **git 仓库根目录决定项目路径。** 同一仓库内的所有子目录共享一个自动记忆目录。如果你 `cd` 到 `src/api/` 启动 Claude，它使用的是与从仓库根目录运行相同的记忆。
- **git worktree 各自有独立的记忆目录。** 这是刻意设计的——不同 worktree 通常代表不同状态的分支。
- **在 git 仓库外**，使用工作目录代替仓库根目录。

---

## 什么会被记住

Claude 在处理你的项目时，会跨多个类别保存笔记：

**项目模式：** 构建命令、测试约定、代码风格偏好。运行一次测试套件后，Claude 就会记下命令和所需的特殊标志。

**调试洞察：** 复杂问题的解决方案和常见错误原因。如果 Claude 花时间诊断了一个 CORS 问题或 webpack 配置问题，它会记录解决方案。

**架构笔记：** 关键文件、模块关系、重要抽象。Claude 绘制地图，这样每次 session 就不需要重新探索你的项目结构。

**你的偏好：** 沟通风格、工作流习惯、工具选择。如果你始终偏好某些方式，Claude 会注意到。

`MEMORY.md` 文件充当简洁索引。当详细笔记堆积时，Claude 会把它们移到专用主题文件中，如 `debugging.md` 或 `patterns.md`。这使主文件保持在 200 行以内，因为那是启动时加载的上限。

---

## 如何使用自动记忆

### 让它自动工作

最简单的方式：什么都不做。自动记忆默认启用。在你工作时，Claude 在后台读写记忆文件。当 Claude 访问记忆目录中的文件时，你会在 session 中看到这个过程。

### 保存特定知识

直接告诉 Claude 要记住什么：

```
"记住我们用 pnpm 不用 npm"
"保存到记忆：API 测试需要本地 Redis 实例"
"记下 staging 环境使用 3001 端口"
```

Claude 会立即把这些写入相应的记忆文件。

### 浏览和编辑

在任意 session 中运行 `/memory` 打开记忆文件选择器，会显示所有记忆文件（CLAUDE.md、自动记忆、本地配置），并让你在系统编辑器中打开任何一个。

也可以直接读取文件：

```bash
# 列出项目的所有记忆文件
ls ~/.claude/projects/<project>/memory/

# 读取主记忆索引
cat ~/.claude/projects/<project>/memory/MEMORY.md

# 读取特定主题文件
cat ~/.claude/projects/<project>/memory/debugging.md
```

这些都是纯 Markdown 文件，随时可以编辑。删除过时的条目，随项目演进重新整理。

---

## 配置与控制

自动记忆默认启用，以下是所有控制方式：

### 按 session 切换

运行 `/memory` 使用自动记忆开关，这是为当前工作流快速开关的最简便方式。

### 对所有项目禁用

在用户设置中添加：

```json
// ~/.claude/settings.json
{ "autoMemoryEnabled": false }
```

### 对单个项目禁用

在项目设置中添加：

```json
// .claude/settings.json
{ "autoMemoryEnabled": false }
```

### 环境变量覆盖

`CLAUDE_CODE_DISABLE_AUTO_MEMORY` 环境变量会覆盖所有其他设置。这是 CI 流水线、自动化环境和托管部署的正确选择：

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1  # 强制关闭
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=0  # 强制开启
```

这优先于 `/memory` 开关和 `settings.json`，是终极开关。

---

## 何时用哪个

有三种记忆系统可用，以下是实用决策框架：

**用 CLAUDE.md**：当你想强制执行规则时。编码规范、架构决策、必需命令和团队约定属于这里。CLAUDE.md 在启动时以高优先级完整加载。想让 Claude 总是遵循某个模式，就放在 CLAUDE.md。

**用自动记忆**：当你想让 Claude 有机地学习时。工作中涌现的项目模式、调试解决方案和隐性偏好非常适合自动记忆，你不需要提前预料所有事情。

**用 Session Memory**：当你需要对话连续性时。Session Memory 跟踪你在特定 session 中讨论和决定的事情，是"我们昨天做了什么"的系统。

**用 rules 目录**：当你的 CLAUDE.md 变得太大时。把指令拆分到 `.claude/rules/` 下的专注文件中，以获得更好的组织而不失去优先级。

这些系统之间的重叠是刻意的：自动记忆捕获你忘记写入 CLAUDE.md 的内容，Session Memory 提供自动记忆的项目级笔记无法捕获的上下文，合在一起创建覆盖项目规则、项目知识和对话历史的分层记忆。

---

## 最佳实践

**保持 MEMORY.md 在 200 行以内。** 启动时只加载前 200 行。Claude 被要求通过把详细笔记移到独立主题文件来保持简洁。手动编辑时也请遵守这个限制。

**定期检查自动记忆。** 像任何笔记一样，这些内容可能会变得过时。重大重构或架构变更后，浏览记忆文件，删除过时条目。

**不要在 CLAUDE.md 和自动记忆之间重复。** 如果某事重要到需要成为规则，放在 CLAUDE.md。如果它是可能改变的学习模式，让自动记忆处理。

**对关键知识使用显式保存。** 当你解决了一个棘手的调试问题或做出重要的架构决策时，告诉 Claude 记住它，不要依赖 Claude 自己注意到所有事情。

**在 CI 中禁用自动记忆。** 自动化流水线不需要 Claude 积累关于构建环境的笔记。在 CI 配置中设置 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。

**与上下文工程结合。** 自动记忆是更广泛上下文工程策略中的一个层次。你越刻意地构建 Claude 在启动时知道的内容，每次 session 的表现就越好。

---

## 来源

原文：https://claudefa.st/blog/guide/mechanics/auto-memory

## Related

- [[Claude Code Session Memory 深度解析]]
- [[Claude Code 梦境机制：Anthropic 的新记忆功能]]
- [[CLAUDE.md 编写指南（基于 v2.1.88 源码）]]
