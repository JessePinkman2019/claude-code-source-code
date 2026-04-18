---
title: "Claude Code Library 系统：跨项目管理 .claude 配置"
aliases:
  - "learning-library-system"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "CLAUDE.md 编写指南（基于 v2.1.88 源码）"
  - "Claude Code Memory 优化深度解析"
---

# Claude Code Library 系统：跨项目管理 .claude 配置

## 核心问题：.claude 文件夹的分发困境

10 个代码库，每个都有 `.claude` 文件夹，里面放着 skills、agents、commands、hooks 和 CLAUDE.md。有些是最新的，大多数在几周前就已经漂移了。

你在一个项目里改进了 code review agent，忘了同步到另外九个。你的队友构建了一个 planning command，你永远不会知道它存在，因为它只在他的 repo 里。

这是 `.claude` 文件夹的分发问题，每增加一个项目就会更严重。

---

## 解法：私有 Git Library

核心思路：建一个私有 Git 仓库存放所有 `.claude` 内容，用一个 `map.json` 控制每个项目获取哪些条目。

```
library-repo/
├── map.json           ← 大脑：哪个项目得到什么
├── skills/
│   ├── git-commits/
│   ├── react/
│   └── git-commits--blog/   ← 变体（blog 项目专用版本）
├── agents/
├── commands/
├── hooks/
├── claude-mds/
└── settings/
```

运行一次 sync 命令，项目的 `.claude` 文件夹就与 map 保持一致。本地修改某个 skill，push 回 library，所有用到它的项目下次 sync 时自动更新。

---

## map.json：系统的大脑

每个项目条目精确列出它接收哪些内容：

```json
{
  "projects": {
    "my-saas-app": {
      "skills": ["git-commits", "react", "postgres-best-practices", "auth"],
      "agents": ["backend-engineer", "supabase-specialist", "security-auditor"],
      "commands": ["plan", "review", "deploy-check"],
      "hooks": ["format-on-save", "test-on-commit"],
      "claude-mds": ["saas-default"],
      "settings": ["team-standard"]
    },
    "my-blog": {
      "skills": ["git-commits--blog", "seo", "content"],
      "agents": ["content-writer", "seo-specialist"],
      "commands": ["draft", "publish"],
      "claude-mds": ["blog-default"]
    }
  }
}
```

不同项目从同一个 library 取不同的组合。Library 存放所有内容，每个项目按需选取。

### Profiles：公共配置的快捷方式

```json
{
  "profiles": {
    "web-default": {
      "skills": ["git-commits", "react", "typescript"],
      "agents": ["frontend-specialist", "backend-engineer"],
      "hooks": ["format-on-save"]
    }
  }
}
```

初始化新项目时 `--profile web-default`，完整的 web 技术栈一条命令就绪。

---

## 变体（Variants）：不分叉地定制

**场景**：`git-commits` skill 在大多数项目上运行良好，但 blog 项目需要额外支持 MDX commit 模式。你不想把它分叉成完全独立的 skill，你想要同一个 skill 的项目专用版本。

**命名规则**：`name--variant`（双短横线分隔基础名和变体标签）

```
skills/
├── git-commits/           ← 基础版本
└── git-commits--blog/     ← blog 项目专用变体
```

在 `map.json` 中引用完整名称：
```json
"skills": ["git-commits--blog"]
```

同步到项目时，`git-commits--blog` 部署为 `.claude/skills/git-commits/`——`--blog` 后缀是 library 内部概念，项目里看不到。

**Push 时的智能映射**：你修改了 blog 项目的 `.claude/skills/git-commits/`，sync 引擎读取 manifest，发现 `git-commits` 映射到 library 里的 `git-commits--blog`，把改动 push 到正确的地方。其他项目仍然用基础版本。

---

## 八个操作命令

| 操作 | 作用 |
|------|------|
| **sync** | 按 map.json 从 library 复制内容到项目 |
| **push** | 把本地改动推回 library（遵循变体映射） |
| **diff** | 哈希对比每个托管条目，显示哪些有变化 |
| **list** | 展示所有 library 内容及各项目的使用情况 |
| **add** | 在项目 map 中添加条目，然后 sync |
| **remove** | 从 map 删除条目并从项目中删除 |
| **init** | 在 map.json 中注册新项目 |
| **seed** | 从现有项目的 .claude/ 导入内容到 library |

`seed` 是引导启动的方式：指向已有良好配置的 `.claude` 文件夹，它把所有内容导入 library，创建 map.json 条目，写入 manifest。从此 library 成为唯一真实来源。

`diff` 用内容哈希对比 library 和项目版本，为每个条目报告 `in-sync`、`local-changes` 或 `library-ahead`，sync 或 push 之前总能清楚地知道哪里漂移了。

---

## Manifest：本地安全边界

每个 sync 过的项目获得 `.claude/.library-manifest.json`：

```json
{
  "managed": {
    "git-commits": "git-commits--blog",
    "react": "react",
    "backend-engineer": "backend-engineer"
  }
}
```

Manifest 记录部署名到 library 名的映射，这是 push 操作知道把 `git-commits/` 推回 `git-commits--blog/` 而非基础版本的原因。

`.claude/` 里不在 manifest 中的任何内容，library 完全看不到——本地任务、备份、项目实验都在 sync 时原封不动地保留。Library 只管它自己放进去的东西。

---

## /library Slash Command

系统还包含一个 `/library` slash command，无需离开 Claude Code session 就能管理 library：

```
sync my-blog project
push the git-commits skill changes
add the security-auditor agent to my-saas-app
show what's drifted across all projects
```

Claude 读取 manifest，定位 library，理解意图，调用 sync 引擎。对于破坏性操作（删除条目、覆盖），先确认再执行。

这个 command 本身也由 library 管理——改进它后，下次 sync 自动传播到所有项目。分发系统自我分发。

---

## 从零搭建（约 30 分钟）

1. **创建私有 repo**，建好文件夹结构：`skills/`、`agents/`、`commands/`、`hooks/`、`claude-mds/`、`settings/`

2. **从最完善的项目 seed**：把配置最完整的 `.claude` 文件夹导入，这成为初始 library 内容

3. **写 map.json**：先写一个项目条目，列出它应该有的所有内容

4. **构建 sync 脚本**：核心逻辑简单——读 map.json、解析条目名（在 `--` 处分割处理变体）、复制文件到正确位置、写 manifest。sync 前 git pull，push 后 git commit + push

5. **添加 /library command** 到 library 本身，同步到所有项目，从第一天就有自然语言接口

6. **加入第二个项目**（配置不同的选取）——这是系统证明价值的地方：两个项目、同一个 library、不同配置、始终同步

**关键设计决策**：存储实际文件，而非引用。引用增加间接层，使变体更难实现，push 更复杂。文件放在 library 里，sync 就是复制，push 就是复制回去，diff 就是哈希比较。

---

## 使用后的改变

**改进自动传播**：修复 code review agent 的 bug，push 它，下次 sync 带给每个使用它的项目。不再有"我在某处修过这个但记不住在哪"。

**新项目快速启动**：用 profile 初始化，完整的 agent 团队、hooks、settings 和 CLAUDE.md 几秒就绪。不再花第一个小时复制配置文件。

**变体防止隐式分叉**：项目需要某个东西的修改版本时，创建变体而非悄悄发散。基础版本和变体的关系在 library 中明确。一年后还能看清哪些项目用哪个版本。

**团队协作**：Library 是 Git repo，团队克隆并从中 sync。有人改进了一个 skill，push 回去，全队下次 sync 就得到改进。map.json 可以为每个团队成员的项目建立条目。

---

## 与 IndyDevDan 方案的区别

[IndyDevDan 的 the-library](https://github.com/disler/the-library) 在 YAML 目录中存储引用（指向 GitHub repo 或本地路径的指针）。本文描述的方案：
- 在一个 repo 里存储**实际文件**
- 增加了变体命名规范处理项目专用版本
- 包含 manifest 支持双向 sync
- 覆盖所有 `.claude` 内容类别（skills、agents、commands、hooks、rules、CLAUDE.md、settings、任意文件）

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/library-meta-skill

## Related

- [[CLAUDE.md 编写指南（基于 v2.1.88 源码）]]
- [[Claude Code Memory 优化深度解析]]
