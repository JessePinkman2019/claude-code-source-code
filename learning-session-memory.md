# Claude Code Session Memory 深度解析

> 参考：https://claudefa.st/blog/guide/mechanics/session-memory  
> 源码路径：`src/services/SessionMemory/`

---

## 一、是什么

Session Memory 是 Claude Code 的**自动后台记忆系统**，在会话过程中持续将对话内容提炼成结构化摘要并写入磁盘，供下次会话自动加载。

与 `CLAUDE.md`（人工维护的项目规则）不同，Session Memory **完全自动运行，无需用户干预**。

---

## 二、工作原理（源码级）

### 2.1 触发机制

Session Memory 通过**后处理钩子**（post-sampling hook）运行，在每次 Claude 回复后检查是否应该提取记忆。

核心判断逻辑在 `src/services/SessionMemory/sessionMemory.ts:134` `shouldExtractMemory()`：

```
触发提取的条件（两者满足其一）：
1. token 增量阈值 AND tool call 次数阈值 同时满足
2. token 增量阈值满足 AND 最后一轮没有 tool call（自然对话断点）

注意：token 增量阈值是必要条件，始终要求满足。
```

### 2.2 阈值参数（默认值）

源码 `src/services/SessionMemory/sessionMemoryUtils.ts:32`：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `minimumMessageTokensToInit` | **10,000** | 初次触发的上下文 token 数阈值 |
| `minimumTokensBetweenUpdate` | **5,000** | 两次更新之间最小 token 增量 |
| `toolCallsBetweenUpdates` | **3** | 两次更新之间最少 tool call 次数 |

这些值可通过 GrowthBook 远程配置 `tengu_sm_config` 动态覆盖。

### 2.3 存储路径

源码 `src/utils/permissions/filesystem.ts:259`：

```
~/.claude/projects/<project-hash>/<session-id>/session-memory/summary.md
```

- 每个 session 有独立的 summary.md
- 文件权限：目录 0o700，文件 0o600
- 多个 session 积累历史，形成项目记忆链

### 2.4 提取过程

Session Memory 提取通过 `runForkedAgent()` 以独立子 agent 运行（`querySource: 'session_memory'`），不阻塞主对话线程。子 agent 只被授权用 Edit 工具修改 summary.md，其他工具一律拒绝（`src/services/SessionMemory/sessionMemory.ts:460`）。

### 2.5 Feature Flag（前置条件）

Session Memory 被 Statsig feature flag `tengu_session_memory` 控制。  
此外，`initSessionMemory()` 还要求 **auto-compact 必须开启**，否则直接跳过：

```typescript
// src/services/SessionMemory/sessionMemory.ts:369
if (!autoCompactEnabled) {
  return
}
```

**Bedrock / Vertex / Foundry 用户无法使用此功能**，需要 Anthropic 原生 API 基础设施。

---

## 三、Summary 文件结构

源码 `src/services/SessionMemory/prompts.ts:11` 定义了默认模板，共 **9 个固定 section**：

```markdown
# Session Title       - 5-10词的信息密集描述
# Current State       - 正在进行的工作、未完成任务、下一步
# Task specification  - 用户要求构建什么、设计决策
# Files and Functions - 重要文件及其作用
# Workflow            - 常用命令及执行顺序
# Errors & Corrections- 遇到的错误及修复方法
# Codebase and System Documentation - 系统组件
# Learnings          - 有效/无效方法
# Key results        - 用户要求的具体输出（完整表格、答案等）
# Worklog            - 逐步行动记录
```

**关键限制**（源码 `src/services/SessionMemory/prompts.ts:8`）：
- 每个 section 最大 **2,000 tokens**
- 整个文件最大 **12,000 tokens**
- 超限时 AI 会被强制要求压缩旧内容

### 自定义模板

可在 `~/.claude/session-memory/config/template.md` 放置自定义模板，  
可在 `~/.claude/session-memory/config/prompt.md` 放置自定义提取指令（支持 `{{currentNotes}}` 和 `{{notesPath}}` 占位符）。

---

## 四、Session Memory Compact（即时压缩）

当 `tengu_session_memory` 和 `tengu_sm_compact` 两个 flag 同时开启时，`/compact` 变成**即时操作**。

传统 compact 需要重新分析整个对话（~2分钟），Session Memory Compact 直接加载已有的 summary.md 作为压缩结果，无需重新分析。

源码 `src/services/compact/sessionMemoryCompact.ts:514` `trySessionMemoryCompaction()`：

1. 等待当前正在进行的 session memory 提取完成（最多 15 秒超时）
2. 读取 summary.md 内容
3. 计算保留消息的起点（从最后一条已总结消息之后开始，向前扩展到满足最小要求）
4. 构建压缩结果，summary.md 内容直接作为摘要注入

压缩后保留的消息量（默认配置，`DEFAULT_SM_COMPACT_CONFIG`）：
- 最少 10,000 tokens
- 最少 5 条含文本的消息
- 最多 40,000 tokens

---

## 五、跨会话 Recall

当新会话启动时，系统会查找该项目目录下历史 session 的 summary.md 文件，并将内容注入上下文。注入时带有标注，提醒 Claude 这是"来自过去会话的背景参考，不一定与当前任务相关"，避免盲目遵循旧决策。

终端启动时会显示：
```
Recalled 3 memories (ctrl+o to expand)
```

---

## 六、与 `/remember` 命令的关系

`/remember` 是从 Session Memory 到 `CLAUDE.local.md` 的**提升通道**：

- Session Memory = 原始历史快照（自动，临时）
- CLAUDE.local.md = 永久项目规则（手动确认，持久）
- `/remember` = 识别跨 session 的重复模式，提议写入 CLAUDE.local.md

---

## 七、对比表

| 特性 | Session Memory | CLAUDE.md |
|------|---------------|-----------|
| 创建者 | Claude（自动） | 用户（手动） |
| 范围 | 每个 session 快照 | 持久项目规则 |
| 优先级 | 背景参考 | 高优先指令 |
| 最佳用途 | 跨 session 连续性 | 标准规范、架构、命令 |

---

## 八、最佳实践

1. **明确声明意图**：会话开头说"我在用 Stripe 构建支付集成"，有利于生成准确的 Session Title
2. **显式总结决策**：说"我们决定用 webhook 而不是轮询"，这会被记录为 Key results
3. **主动触发记录**：说"记录我们刚才的架构决策"，可以触发更丰富的提取
4. **查看已存储记忆**：
   ```bash
   ls ~/.claude/projects/
   cat ~/.claude/projects/<hash>/<session-id>/session-memory/summary.md
   ```
5. **利用即时 compact**：长会话中可以随时 `/compact`，不再需要担心等待时间

---

## 九、注意事项

- Session Memory **需要 auto-compact 开启**才会初始化（源码强制要求）
- 短会话（< 10,000 tokens）不会触发任何提取
- 提取是**串行化**的（`sequential()` 包装），同时只能有一个提取在运行
- Remote 模式（SSH 远程会话）下 Session Memory **不启用**
- 子 agent、teammate 等非主线程查询**不触发** Session Memory 提取
