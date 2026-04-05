# Claude Code 梦境机制：Anthropic 的新记忆功能

Claude Code Auto Dream 会整合记忆文件，清除过时笔记并合并洞察。就像 AI Agent 的 REM 睡眠。

Anthropic 悄悄为 Claude Code 上线了一个尚未正式公告的功能，叫做 **Auto Dream**。它解决了自动记忆最大的问题：随时间推移的衰减。

**问题所在：** 自动记忆是一次突破——Claude 终于能自己给项目做笔记了。但经过 20+ 个 session 之后，这些笔记变成了一团乱麻。矛盾条目不断堆积，"昨天"这样的相对日期失去意义，过时的调试方案引用了早已不存在的文件。原本用来帮助 Claude 记忆的笔记本，反而成了干扰它的噪声。

**快速检查：** 在任意 Claude Code session 中运行 `/memory`，查找选择器中是否有 "Auto-dream: on"。如果有，Claude 已在 session 之间后台整合你的记忆文件了。

```
# 检查当前记忆状态
/memory

# 查找：
# Auto-dream: on
```

如果 Auto Dream 已开启，你不需要做任何事。Claude 会周期性地审查项目中的每个记忆文件，清除过时内容，解决矛盾，重新整理其余内容。如果还没看到这个开关，继续往下读——你可以手动触发它。

---

## 为什么叫"做梦"？REM 睡眠的类比

这个命名是刻意的，而且这个比喻出奇地准确。

白天，大脑吸收原始感官输入并存为短期记忆。REM 睡眠期间，大脑回放当天的事件，强化重要的连接，丢弃不重要的，把一切整理成长期记忆。REM 睡眠不足的人难以形成持久记忆——信息进来了，但从未被整合。

自动记忆是 Claude 的白天大脑：边工作边做笔记，记录调试模式、构建命令、架构决策、你的偏好。每个 session 都会增加新条目。但没有整合步骤，这些笔记就像未整合的短期记忆一样不断堆积——矛盾持续存在，过时条目久久不去，信噪比随每个 session 下降。

Auto Dream 就是 REM 睡眠周期。它审查自动记忆收集的内容，强化仍然相关的部分，移除过时的部分，把其余内容重新整理成干净、有索引的主题文件。没有 Auto Dream 的 Claude Code，本质上是睡眠剥夺状态——不断添加随机笔记，却从不清理。

---

## Auto Dream 的四个阶段

Auto Dream 运行时遵循结构化的四阶段流程。每个阶段有其特定目的，共同将分散的 session 笔记转化为有组织的项目知识。

### 阶段一：定向（Orientation）

Claude 读取当前记忆目录，盘点现有内容。打开 `MEMORY.md`（索引文件），扫描主题文件列表，建立当前记忆状态的心智地图。

这个阶段回答："我已经知道什么，它是如何组织的？"

### 阶段二：收集信号（Gather Signal）

Auto Dream 在 session 记录（Claude 为每个 session 本地存储的 JSONL 文件）中搜索，但不会逐字读完每条记录——对于有数百个 session 的项目来说代价太高。它只针对性地搜索特定模式：

- **用户纠正：** 你告诉 Claude 它错了或重新引导它的时刻
- **明确保存：** 你说"记住这个"或"保存到记忆"的时刻
- **反复出现的主题：** 跨多个 session 出现的模式
- **重要决策：** 架构选择、工具选型、工作流变更

针对性搜索是刻意为之的。穷举读取 500+ 条 session 记录只会浪费 token 和时间，效益递减。有针对性的 grep 式搜索能高效提取最高价值的信号。

### 阶段三：整合（Consolidation）

核心阶段。Claude 将新信息合并到现有主题文件中，并进行关键维护：

- 将相对日期转为绝对日期。"昨天我们决定用 Redis"变为"2026-03-15 我们决定用 Redis"，防止记忆随时间推移产生时间混淆。
- 删除被推翻的事实。如果三周前从 Express 切换到了 Fastify，旧的"API 使用 Express"条目会被移除。
- 清除过时记忆。关于重构中已删除文件的调试笔记毫无用处，直接剪掉。
- 合并重叠条目。如果三个不同 session 都记录了同一个构建命令怪癖，合并为一个干净条目。

### 阶段四：剪枝与索引（Prune and Index）

最后阶段专注于 `MEMORY.md` 索引文件。这个文件保持在 200 行以内，因为这是启动时加载的上限。阶段四更新 `MEMORY.md` 以准确反映所有主题文件的当前状态：

- 移除指向不再存在的主题文件的指针
- 添加整合过程中新建主题文件的链接
- 解决索引与实际文件内容之间的矛盾
- 按相关性和时效性重排条目

整合过程中不需要修改的文件保持不变。Auto Dream 不会每次运行都重写所有内容，它是外科手术式的。

---

## 完整系统提示词

以下是驱动 Auto Dream 的实际系统提示词，这是梦境周期启动时 Claude 收到的内容：

```
# Dream: Memory Consolidation

You are performing a dream - a reflective pass over your memory files.
Synthesize what you've learned recently into durable, well-organized
memories so that future sessions can orient quickly.

Memory directory: `~/.claude/projects/<project>/memory/`
This directory already exists - write to it directly with the Write tool
(do not run mkdir or check for its existence).

Session transcripts: `~/.claude/projects/<project>/`
(large JSONL files - grep narrowly, don't read whole files)

## Phase 1 - Orient

- `ls` the memory directory to see what already exists
- Read `MEMORY.md` to understand the current index
- Skim existing topic files so you improve them rather than creating
  duplicates
- If `logs/` or `sessions/` subdirectories exist (assistant-mode layout),
  review recent entries there

## Phase 2 - Gather recent signal

Look for new information worth persisting. Sources in rough priority order:

1. **Daily logs** (`logs/YYYY/MM/YYYY-MM-DD.md`) if present - these are
   the append-only stream
2. **Existing memories that drifted** - facts that contradict something
   you see in the codebase now
3. **Transcript search** - if you need specific context (e.g., "what was
   the error message from yesterday's build failure?"), grep the JSONL
   transcripts for narrow terms:
   `grep -rn "<narrow term>" <project-transcripts>/ --include="*.jsonl" | tail -50`

Don't exhaustively read transcripts. Look only for things you already
suspect matter.

## Phase 3 - Consolidate

For each thing worth remembering, write or update a memory file at the
top level of the memory directory. Use the memory file format and type
conventions from your system prompt's auto-memory section - it's the
source of truth for what to save, how to structure it, and what NOT
to save.

Focus on:

- Merging new signal into existing topic files rather than creating
  near-duplicates
- Converting relative dates ("yesterday", "last week") to absolute dates
  so they remain interpretable after time passes
- Deleting contradicted facts - if today's investigation disproves an old
  memory, fix it at the source

## Phase 4 - Prune and index

Update `MEMORY.md` so it stays under 200 lines. It's an **index**, not a
dump - link to memory files with one-line descriptions. Never write memory
content directly into it.

- Remove pointers to memories that are now stale, wrong, or superseded
- Demote verbose entries: keep the gist in the index, move the detail into
  the topic file
- Add pointers to newly important memories
- Resolve contradictions - if two files disagree, fix the wrong one

---

Return a brief summary of what you consolidated, updated, or pruned. If
nothing changed (memories are already tight), say so.
```

关于这个提示词有几点值得注意：它明确要求 Claude 进行针对性 grep 而非读完整个记录；强制执行 `MEMORY.md` 200 行限制；并要求 Claude 在源头解决矛盾，而不仅仅是标记它们。

---

## Auto Dream 何时运行

触发整合周期需要同时满足两个条件：

- 距上次整合已过 **24 小时**
- 距上次整合已发生 **超过 5 个 session**

两个条件缺一不可。如果你在两天内只跑了一个长 session，Auto Dream 不会触发（session 数不够）。如果你在两小时内跑了 10 个短 session，也不会触发（时间不够）。这个双重门槛防止轻度使用的项目进行不必要的整合，同时确保活跃项目定期清理。

作为参考：有记录的案例中，Auto Dream 整合了 913 个 session 的记忆，耗时约 8-9 分钟。

---

## 安全保障

Auto Dream 运行时有有意义的约束来防止意外。

- **项目代码只读。** 梦境周期中，Claude 只能写入记忆文件，无法修改源代码、配置、测试或任何其他项目文件。梦境进程被沙箱化在记忆目录内。
- **锁文件防止并发运行。** 如果同一项目打开了两个 Claude Code 实例，只有一个能运行 Auto Dream。锁文件确保不会有两个整合进程同时运行，防止记忆文件产生合并冲突。
- **后台执行。** Auto Dream 运行时你可以继续在 Claude Code 中工作，它不会阻塞你的 session 或要求你等待，整合在独立进程中进行。

---

## 如何手动触发

你不必等待自动触发。如果你知道记忆文件需要清理（比如大重构之后），可以强制整合。

`/dream` 命令存在，但尚未向所有人开放。目前你可以直接告诉 Claude 来手动触发：

```
"dream"
"auto dream"
"consolidate my memory files"
```

Claude 会识别意图并执行完整的四阶段流程。这在项目发生重大变化后特别有用，无需等待下一个自动周期就能获得干净的记忆。

---

## 整合前后对比

### 整合前（30+ Session 积累的混乱状态）

```
~/.claude/projects/<project>/memory/
├── MEMORY.md            # 280 行，超出 200 行限制
├── debugging.md         # 包含 3 条关于 API 错误的矛盾条目
├── api-conventions.md   # 引用 Express（两周前已切换到 Fastify）
├── random-notes.md      # 过时与当前信息混杂
├── build-commands.md    # "昨天"出现 6 次，没有具体日期
└── user-preferences.md  # 与 MEMORY.md 中的条目重复
```

### 整合后（干净状态）

```
~/.claude/projects/<project>/memory/
├── MEMORY.md            # 142 行，带链接的干净索引
├── debugging.md         # 去重，只保留当前有效方案
├── api-conventions.md   # 已更新以反映 Fastify 迁移
├── build-commands.md    # 所有日期为绝对日期，无重复
└── user-preferences.md  # 已与 MEMORY.md 相关条目合并
```

`random-notes.md` 消失了。MEMORY.md 从 280 行降至 142 行。

---

## 自动记忆 vs Auto Dream：同一系统的两个层次

| 维度 | Auto Memory | Auto Dream |
|---|---|---|
| 角色 | 收集新信息 | 整合现有信息 |
| 运行时机 | 每个 session 期间 | 周期性（24h + 5 sessions） |
| 做什么 | 写入发现的模式笔记 | 剪枝、合并、重组笔记 |
| 触发方式 | 自动、持续 | 自动或手动 |
| 输出 | 记忆文件中的新条目 | 清理后重组的记忆文件 |
| 跳过的风险 | 遗漏有用信息 | 记忆质量随时间下降 |

---

## 四大记忆系统全景

| 维度 | CLAUDE.md | Auto Memory | Session Memory | Auto Dream |
|---|---|---|---|---|
| 谁来写 | 你 | Claude（每 session） | Claude（自动） | Claude（周期性） |
| 用途 | 指令与规则 | 项目模式与经验 | 对话摘要 | 记忆整合 |
| 运行时机 | 你手动编辑 | 每个 session 期间 | 后台，每 ~5K token | 每 24h + 5 sessions |
| 范围 | 项目级或全局 | 项目级 | session 级 | 项目级 |
| 启动时加载 | 完整文件 | MEMORY.md 前 200 行 | 相关历史 session | 不加载（session 间运行） |
| 存储位置 | `./CLAUDE.md` | `~/.claude/projects/<project>/memory/` | `~/.claude/projects/<project>/<session>/session-memory/` | 同 Auto Memory |
| 最适合 | 规范、架构、命令 | 构建模式、调试、偏好 | session 间的连续性 | 保持记忆干净准确 |
| 人类类比 | 使用手册 | 白天记笔记 | 短期对话回忆 | REM 睡眠整合 |

---

## 实用建议

- **大多数项目让它自动运行。** 默认触发条件（24h + 5 sessions）对活跃开发来说效果很好。
- **大重构后强制触发。** 如果你重命名了一半代码库、迁移了框架或改变了 API 结构，手动触发一次梦境周期。
- **偶尔检查输出。** 梦境周期运行后，快速浏览你的 `MEMORY.md`。Auto Dream 很好，但并不完美，文件是纯 Markdown，随时可以自由编辑。
- **结合上下文管理实践。** 干净的记忆文件意味着 Claude 启动时加载的噪声更少，给实际工作留出更多上下文窗口空间。

---

## 来源

原文：https://claudefa.st/blog/guide/mechanics/auto-dream
