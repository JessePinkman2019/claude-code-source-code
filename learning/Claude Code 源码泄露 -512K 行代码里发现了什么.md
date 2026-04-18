---
title: "Claude Code 源码泄露：512K 行代码里发现了什么"
aliases:
  - "learning-source-leak"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Thread-Based Engineering：量化 AI 辅助开发效率"
  - "Claude Code Library 系统：跨项目管理 .claude 配置"
---

# Claude Code 源码泄露：512K 行代码里发现了什么

## 事件起因

2026 年 3 月 31 日，有人在 `.npmignore` 里漏掉了 `*.map`。

这一个缺失行，把 Claude Code 完整代码库暴露了出来：512,000+ 行 TypeScript、44 个隐藏 feature flag、一个始终运行的后台 Agent（KAIROS）、以及一个专门隐藏 Anthropic 员工对开源项目贡献的隐身模式。

安全研究员 **Chaofan Shou**（Solayer Labs 实习生）发现 v2.1.88 的 npm 包里附带了一个 59.8 MB 的 source map 文件（`cli.js.map`），在 X 上发出下载链接，帖子获得 1600-2100 万次浏览。

根本原因很平凡：Claude Code 用 Bun runtime 构建，Bun 默认生成 source map，有人忘了把 `*.map` 加进 `.npmignore`。Claude Code 负责人 Boris Cherny 确认这是"纯粹的开发者失误"，并补充了一句意味深长的话：

> "100% of my contributions to Claude Code were written by Claude Code."

这不是第一次——2025 年 2 月也发生过类似泄露，意味着 13 个月内至少第二次。更巧的是，距"Mythos"模型规格泄露仅 5 天——那次是 CMS 配置错误暴露了约 3000 个内部文件。

---

## 泄露规模

源码解压后约 1,900 个 TypeScript 文件：

| 指标 | 数值 |
|------|------|
| 源码文件总数 | ~1,900 个 TypeScript 文件 |
| 总代码行数 | 512,000+ 行 |
| 主入口文件 | 785KB（`main.tsx`） |
| 注册工具数 | 40+ |
| 隐藏 feature flag | 44 个，覆盖 20+ 未发布功能 |
| Context 压缩策略 | 5 种不同方案 |
| 事件 Hook | 25+ |

架构是基于 React + Ink 的自定义终端渲染器，带有游戏引擎风格的优化。Query engine 单独 46,000 行，Base tool definition 29,000 行。子 Agent 执行模型有三种：Fork、Teammate 和 Worktree。

---

## Undercover Mode：最具争议的发现

文件：`undercover.ts`，约 90 行。

当 Anthropic 员工（`USER_TYPE === 'ant'`）在非内部仓库使用 Claude Code 时激活，注入以下 system prompt：

```
You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository.
Your commit messages, PR titles, and PR bodies MUST NOT contain
ANY Anthropic-internal information. Do not blow your cover.
```

激活后的行为：
- **删除所有 `Co-Authored-By` 归因**：从 commit 中抹去 Claude Code 生成代码的痕迹
- **禁止提及**内部代号、Slack 频道、内部仓库名，甚至连"Claude Code"这个词本身都不能提
- **屏蔽对未发布模型的引用**，如 Opus 4.7 和 Sonnet 4.8
- **无外部用户关闭机制**：外部构建中该功能被完全死代码消除

Hacker News 的评论精准概括了两种观点：Buddy 系统"很可爱"，Undercover Mode"不道德"。

最深刻的讽刺：他们构建 Undercover Mode 专门为了防止内部机密通过代码贡献泄露，然后把整个源码通过忘记排除的 npm 文件泄露出去了。

---

## KAIROS：始终运行的后台 Agent

源码中引用 150+ 次，以古希腊"时机之神"（kairos，"恰当的时刻"）命名——与 chronos（线性时间）对照，代表根据上下文判断何时行动，而非按计划运行。

**代码揭示的能力：**

- **自主运行**：接收周期性 `<tick>` prompt，自主决定是否行动
- **持久化**：笔记本合盖也继续运行，跨重启维持会话状态
- **15 秒阻塞预算**：防止单次决策周期垄断系统资源
- **只追加日志**：每日日志文件，Agent 自身无法删除，提供完整审计轨迹
- **三个专属工具**：推送通知、文件投递、PR 订阅——标准 Claude Code session 无法访问这些
- **GitHub Webhook 订阅**：无需用户干预，自主监控仓库事件

KAIROS 代表了超越现有"自主 Agent 循环"的下一步：当前 Claude Code session 需要用户在场审批，而 KAIROS 会在后台无限期运行，监视仓库并在判断时机合适时主动行动。

150+ 处引用说明这不是原型，而是一个等待放行的完成功能。

---

## autoDream：Claude 在字面意义上"做梦"

`services/autoDream/` 目录包含一个在空闲时运行的记忆整合系统。

**触发三重门控：**
1. 距上次 dream 至少 24 小时
2. 上次整合后至少完成 5 个 session
3. 整合锁（防止并发 dream 进程）

**四阶段 dream 流程：**
1. **Orient**：评估当前记忆状态，确定需要整合什么
2. **Gather Recent Signal**：收集近期 session 的洞察
3. **Consolidate**：将新知识与现有记忆合并
4. **Prune and Index**：去除冗余，保持 MEMORY.md 在 200 行 / 约 25KB 以内

源码确认了四阶段架构和具体触发阈值。

---

## 反蒸馏：向竞对训练数据投毒

两层系统，防止竞争对手用 Claude Code 输出训练自己的模型：

**第一层：假工具（Fake Tools）**

feature flag `anti_distillation: ['fake_tools']` 指示服务端在响应中注入诱饵工具定义（由 GrowthBook flag `tengu_anti_distill_fake_tool_injection` 控制）。任何试图蒸馏 Claude Code 工具调用行为的竞争对手，学到的将是刻意错误的工具 schema。

**第二层：CONNECTOR_TEXT**

服务端只返回加密签名的摘要，而非完整推理链。这层屏蔽了对竞争对手模型训练最有价值的详细思维链。范围仅限 `USER_TYPE === 'ant'`（Anthropic 员工），不影响外部用户。

---

## Penguin Mode 与内部代号

"Penguin Mode"是 Fast Mode 的内部名称，API 端点为 `/api/claude_code_penguin_mode`，有一个关闭开关 `tengu_penguins_off`。

### 模型代号一览表

| 代号 | 对应 |
|------|------|
| Tengu | Claude Code 项目内部代号 |
| Capybara | 新模型家族（可能是"Mythos"泄露的模型），含 capybara-fast、capybara-fast[1m]、capybara-v2-fast |
| Fennec | Opus 4.6（源码中有 `migrateFennecToOpus` 迁移函数） |
| Numbat | 未发布模型（"Remove this section when we launch numbat"） |
| Opus 4.7 | 在 Undercover Mode 禁止字符串列表中被引用 |
| Sonnet 4.8 | 在 Undercover Mode 禁止字符串列表中被引用 |

有趣细节：capybara 代号也出现在 Claude Buddy 宠物物种名中，被十六进制编码以绕过 Anthropic 自己的 `excluded-strings.txt` 构建扫描器。18 个宠物物种名被统一编码，让隐藏一个代号不显得可疑。

---

## 代码质量争议

`print.ts`：5,594 行，包含一个 3,167 行的单函数——这个函数本身比许多完整应用还长。

**情绪检测用 regex**：代码库里有一个基于正则的系统，扫描用户输入中的脏话和情绪痛苦信号。社区反应立刻来了："LLM 公司用正则做情感分析？就像卡车公司用马匹运零件。"

**187 个 spinner 动词**：加载动画循环使用 187 个不同动作词。社区有人专门通读整个列表确认有没有"reticulating"（SimCity 2000 的经典加载文案）——找到了。

**嵌套回调**：大量 `.then()` 回调链被描述为"'just vibes' 时代的定义性作品"。鉴于 Claude Code 负责人说工具是自己写自己，这意味着 AI 的编码风格现在有了公开记录。

**亮点**：客户端认证的安全层被推到了 JavaScript 层以下，进入 Bun 的 Zig 级 HTTP 栈。这比大多数开发者工具的实现更为精密，说明 Anthropic 对 API 安全是认真的——尽管 npm 打包是事后诸葛。

---

## 社区反应

**镜像与 fork**：一个镜像仓库累积超 41,500 个 fork；另有镜像发布到去中心化平台，声明"永不删除"。

**洁室重写**：韩国开发者 Sigrid Jin 创建了"claw-code"（Python 重写），约 2 小时内达到 75,000 GitHub star，可能是 GitHub 历史上增长最快的仓库。

**DMCA 下架**：Anthropic 对 GitHub 镜像提交 DMCA，但因为是他们自己的打包错误，这一做法被批评为理亏。

**Memecoins**：有人基于最稀有的 Claude Buddy 变体（Shiny Legendary）在 Solana 发行了 $Nebulynx。当然。

**并发混乱**：同一天，npm 生态系统还在处理一个无关的供应链攻击（通过 Axios），造成了一个现实：同一天同时面对一次企业意外泄露和一次蓄意安全攻击。

**媒体报道**：CNBC、Fortune、Gizmodo、VentureBeat、Axios、The Register、Decrypt、Cybernews、The Hacker News 等均有报道。

---

## 行业意义

### 对 Anthropic
$190 亿年化营收，Claude Code 贡献 $25 亿 ARR，据报道准备于 10 月以约 $3800 亿估值 IPO。一周内两次泄露（源码 + Mythos 模型规格）严重损伤了"安全第一"的品牌叙事。

AI 安全公司 Straiker 警告：攻击者现在可以研究 Claude Code 四阶段 context 管理管道的数据流，识别此前不可见的攻击向量。竞争对手（GitHub Copilot、Cursor 等）也能看到功能 flag 和产品路线图。

### 对使用 Claude Code 的开发者
泄露确认了架构的真实复杂度：5 种 context 压缩策略、14 个 cache 失效向量、23 个 bash 命令安全检查、3 种子 Agent 执行模型。这不是 API 的简单封装，而是深度工程化的系统。

即将到来的功能（KAIROS、ULTRAPLAN、autoDream 精炼）表明 Claude Code 正朝着始终在线、自主运行的方向演进，而非大多数用户今天体验的基于 session 的模式。如果你在围绕 Claude Code 构建工作流，现在就值得为最终的自主运行模式做设计。

### 关于开源的争议
泄露后最多人搜索的问题：Claude Code 是开源的吗？

技术上，源码在 GitHub 可见，但许可证不允许重新分发或修改——"源码可见"≠"开源"。这就是为何 Anthropic 能对镜像提交 DMCA。npm 泄露让完整代码库更易获取，但没有改变许可条款。

Anthropic 官方声明：「没有涉及敏感客户数据或凭证。发布打包问题由人为失误导致，非安全漏洞。」他们起初只用了 npm `deprecated` 标记而非真正下架，又被批了一波响应迟缓。至今没有发布正式事后复盘。

---

## FAQ

**泄露是怎么发生的？**  
Bun runtime 默认生成 source map，有人忘了把 `*.map` 加进 `.npmignore`。Boris Cherny 确认是"纯粹的开发者失误"，13 个月内至少第二次发生。

**Undercover Mode 是什么？**  
Anthropic 员工在非内部仓库使用 Claude Code 时激活的功能，删除 AI 归因、屏蔽内部信息。外部构建中被完全死代码消除，不影响普通用户。

**KAIROS 是什么？**  
未发布的自主后台 daemon，名字来自希腊"恰当时刻"概念，持久跨 session 运行，自主决定何时响应仓库事件。目前仍在 feature flag 后面。

**发现了哪些模型代号？**  
Tengu（Claude Code 项目代号）、Capybara（新模型家族）、Fennec（Opus 4.6）、Numbat（未发布模型）。Undercover Mode 的禁止字符串列表中还出现了 Opus 4.7 和 Sonnet 4.8。

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/claude-code-source-leak

## Related

- [[Thread-Based Engineering：量化 AI 辅助开发效率]]
- [[Claude Code Library 系统：跨项目管理 .claude 配置]]
