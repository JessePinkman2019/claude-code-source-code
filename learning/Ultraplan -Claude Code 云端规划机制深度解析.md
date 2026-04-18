---
title: "Ultraplan：Claude Code 云端规划机制深度解析"
aliases:
  - "learning-ultraplan"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Multi-Agent 协作机制（基于 v2.1.88 源码）"
  - "自主 Agent 循环：睡觉时也在发布功能"
---

# Ultraplan：Claude Code 云端规划机制深度解析

## 核心问题：终端被规划阶段锁死

每次 Claude Code 进入 Plan 模式，终端就会被占用直到规划完成。处理 40 个文件、横跨三个服务的迁移策略，你只能盯着光标转几分钟。

**Ultraplan 的解法**：把整个规划阶段移到 Anthropic 云端基础设施，在远程容器中运行 Opus 4.6，最长 30 分钟，本地终端完全释放，规划在后台进行，你在浏览器里审阅并给出反馈。

---

## 与本地 Plan 模式的核心差异

| 维度 | 本地 Plan 模式 | Ultraplan |
|------|--------------|-----------|
| 运行位置 | 本地机器 | Anthropic 云端基础设施 |
| 终端占用 | 是 | 否，终端保持自由 |
| 审阅方式 | 终端滚动浏览 | 浏览器 UI，支持内联评论 |
| 执行位置 | 仅本地 | 云端（生成 PR）或本地均可 |
| 时间上限 | 受本地资源限制 | Opus 4.6，最长 30 分钟 |

**前置条件**：需要 Claude Code on the web 账户（Pro/Max/Team/Enterprise）且已连接 GitHub 仓库。

---

## 三种调用方式

### 1. Slash 命令
```
/ultraplan <你的任务描述>
```
最明确，触发前弹出确认对话框。

### 2. 关键词触发
在普通 prompt 中包含 `ultraplan` 关键词，Claude 自动检测并弹出确认对话框。适合用自然语言描述任务的场景。

### 3. 从本地规划升级（推荐）
本地 Plan 完成后，在批准对话框选择 **"No, refine with Ultraplan on Claude Code on the web"**，将草稿发送到云端。**云端 session 直接以本地规划为起点，而非从零开始**，是质量最高的路径。

---

## 状态追踪

启动后，CLI 显示实时状态：

| 状态 | 含义 |
|------|------|
| `◇ ultraplan` | 正在研究代码库、起草规划 |
| `◇ ultraplan needs your input` | 有澄清问题，需打开 session 链接回复 |
| `◆ ultraplan ready` | 规划完成，打开浏览器审阅 |

在终端运行 `/tasks` 可查看 ultraplan 条目，也可在此 **Stop ultraplan** 终止云端 session。

> 注意：若 Remote Control 功能处于激活状态，Ultraplan 启动时会断开，两者共用同一 `claude.ai/code` 接口，不能同时连接。

---

## 浏览器审阅 UI

这是 Ultraplan 与本地规划最大的差异所在：

**内联评论**：高亮规划中任意段落并留下定向反馈。不再是模糊的"某部分需要更多细节"，而是直接指向具体段落说明缺什么。Claude 只修订那个部分。

**Emoji 反应**：对各节标注认可或存疑，无需写完整评论，快速标记哪些好、哪些需改。

**大纲侧边栏**：长规划的导航利器，免去痛苦的滚动。

可以多轮迭代：留评论 → Claude 修订 → 再审阅 → 再留评论，直到满意为止。

---

## 四种执行路径

### 路径 1：云端执行，生成 PR
浏览器选择 **"Approve Claude's plan and start coding"**。云端 session 完成实现后，通过 Web UI 审阅 diff 并创建 Pull Request。
适合：自包含的变更，无本地依赖。

### 路径 2：传送回当前终端
选择 **"Approve plan and teleport back to terminal"**，规划注入当前会话，选 **"Implement here"** 继续。
适合：规划涉及本地环境变量、本地服务、仓库外文件等云端无法访问的内容。

### 路径 3：开启新本地 Session
同样传送回终端，选 **"Start new session"**，以干净的上下文窗口专注执行。Claude 会打印 `claude --resume` 命令供后续回到旧会话。
适合：当前 session 上下文已膨胀，需要干净起点。

### 路径 4：保存规划、暂不执行
选 **"Cancel"**，Claude 将规划保存到文件并打印路径。可留待后续执行、分享给团队审阅，或作为手动实现的规格文档。

---

## 源码泄露揭示的内部架构

2026 年 3 月 31 日，Claude Code 源码（512,000+ 行 TypeScript）因 npm 打包失误泄露，其中包含 Ultraplan 的 system prompt。

**Ultraplan 不是单一系统，至少有三个变体，通过 A/B 测试静默分配：**

### Variant 1：`simple_plan`
轻量版。无子 Agent。本质是在云端硬件上运行的常规 Plan 模式。速度快，但深度有限。

### Variant 2：`visual_plan`
在 `simple_plan` 基础上，额外指示 Claude 为结构性变更包含 Mermaid 或 ASCII 图表（依赖顺序、数据流、变更轮廓）。如果你的 Ultraplan 结果包含架构图，说明你分到了这个变体。

### Variant 3：`three_subagents_with_critique`
深度版（Piebald-AI 在 v2.1.88 追踪到）：
- **三个并行研究 Agent** 同时探索代码库的不同维度
- **第四个 Agent** 审查合并后的规划，找出遗漏和风险

这是"探索-综合-批判"多 Agent 模式的实际落地，也是 30 分钟时间窗口存在的真正理由。对于深度架构工作，多 Agent 并行分析风险面、依赖链、现有模式，产出质量远超单次分析。

### 变体抽奖问题
三个变体静默分配，用户无法选择，UI 也不显示当前运行哪个。此外还有一个 `tengu_pewter_ledger` A/B 测试（四个变体：null/trim/cut/cap），影响规划输出的详细程度。

**实际结果**：部分用户得到有风险分析、多节结构的详尽规划，另一些人只得到勉强优于本地规划的简单提纲。这个差异是真实存在的，在早期 Hacker News 讨论中有大量记录。

---

## 版本演变历史

| 版本 | 变化 |
|------|------|
| v2.1.83 | 首次出现，以 "System Reminder: Ultraplan mode" 形式 |
| v2.1.85 | 新增保密指令（不披露工作原理） |
| v2.1.88 | 批准后可在同一 session 执行，新增 teleport sentinel |
| v2.1.89 | `/ultraplan` 从 CLI 命令中移除 |
| v2.1.91 | `/ultraplan` 恢复 |
| v2.1.92 | 规划格式化指南更新 |

v2.1.89→v2.1.91 的移除再恢复，说明 Anthropic 曾短暂测试纯关键词触发模式后回退。该功能至今未出现在官方 CHANGELOG.md，完全在 feature flag 后面，**随时可能变更**。

---

## 最佳实践

### 1. 前置尽量多的上下文
云端 session 可以读你的代码库，但从冷启动开始。对比：
- 差：`/ultraplan migrate auth to JWTs`
- 好：`/ultraplan 将认证系统从 session token 迁移到 JWT，当前实现在 src/auth/，需要保持与旧 token 的向后兼容，影响 12 个 API endpoint`

### 2. 重要工作先走本地规划再升级
先用本地 Plan 得到初步轮廓，再发给 Ultraplan 精炼。云端 session 以更强的基础起步，同时你已在提交云端资源前验证了方向。

### 3. 激进使用内联评论
别只浏览一遍就批准。针对具体段落留定向反馈：
- "这里假设了单数据库，实际有三个：users、analytics、billing"
- "测试策略需要 webhook handler 的集成测试，不只是单元测试"
- "第 4 步依赖第 7 步，需要重排序"

定向评论比"让规划更详细一点"产出更好的修订。

### 4. 按场景选择执行路径

| 场景 | 最佳路径 |
|------|---------|
| 自包含变更，无本地依赖 | 云端执行，生成 PR |
| 需要本地环境变量/Docker/本地服务 | 传送回终端 |
| 当前 session 上下文使用 80%+ | 开启新 session |
| 规划需要团队审阅后再执行 | 保存到文件 |

### 5. 知道什么时候不该用
Ultraplan 有启动延迟（克隆仓库、探索代码库、起草规划）。**简单变更、已知方向时，`Shift+Tab` 两次的本地 Plan 更快。**

用 Ultraplan 的时机：规划本身是难点——迁移、大范围重构、架构决策有多个合理路径、出了问题修复代价极高的任务。

---

## Claude Code 四层规划体系

| 层级 | 触发方式 | 适用场景 |
|------|---------|---------|
| **无规划** | 直接 prompt | 简单明确的任务 |
| **Plan 模式** | `Shift+Tab` 两次 | 中等复杂度，需要执行前审阅 |
| **自动规划** | 复杂 prompt 自动触发 | Claude 自行判断何时需要规划 |
| **Ultraplan** | `/ultraplan` 或关键词 | 高复杂度，需要富审阅和并行探索 |

每层增加能力的同时增加开销。技巧在于匹配正确的层级给你的任务。

Ultraplan 还有一个额外的 context 优势：规划发生在与工作 session 独立的上下文窗口中，本地 context 保持干净供实现阶段使用。

---

## 当前局限性

- **无变体选择**：不能指定 simple/visual/deep 规划，完全由 A/B 测试决定
- **仅支持 GitHub**：GitLab、Bitbucket 和纯本地仓库不支持
- **只能从 CLI 启动**：无法从 Web 界面直接发起 Ultraplan
- **UX 摩擦**：早期用户反映内联评论入口难以发现，整体流程"有些迟钝"
- **机制不透明**：本地文件与云端 session 的同步方式没有文档，中途放弃 session 的后果不明
- **无官方 changelog 记录**：完全在 feature flag 后面，可能随时变更或消失

---

## 战略信号

Anthropic 愿意为**仅规划阶段**投入一个 30 分钟、Opus 4.6 级别的云端 session，说明他们认为规划是 AI 辅助开发的最高杠杆点，而非代码生成、测试或部署。

三变体架构也暗示了演化方向：支持"选择规划深度"的基础设施已经建好，未来版本很可能允许用户根据任务复杂度显式请求 simple/visual/deep 规划。

目前，Ultraplan 最适合作为**高复杂度、高风险工作的专用规划工具**，日常任务继续用本地 Plan 模式，把 Ultraplan 留给那些必须第一次就做对的规划。

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/ultraplan

## Related

- [[Multi-Agent 协作机制（基于 v2.1.88 源码）]]
- [[自主 Agent 循环：睡觉时也在发布功能]]
