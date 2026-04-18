---
title: "Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考"
aliases:
  - "learning-interactive-mode"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Voice Mode：对着终端说话"
  - "输出格式与文件变更审批：d/e/y/n 四个快捷键"
  - "`/powerup`：Claude Code 内置交互式教程机制解析"
---

# Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考

## 两个立刻能用的快速提升

```bash
# 立刻杀掉所有后台 agent（按两次确认）
Ctrl+F

# Claude 处理中，问一个旁白问题，不影响对话历史
/btw 那个报错在哪个文件里？
```

---

## /btw：不打断主流程的旁白提问

这是 2026 年 3 月发布的最新交互功能（Claude Code 团队 Erik Schluntz 旁项目，发布推文 220 万浏览）。

**它解决什么问题**：Claude 正在执行长任务时，你突然需要确认一件事，但又不想打断当前任务、污染对话历史。

```bash
# Claude 正在重构模块时，随时插入：
/btw 认证中间件的测试文件在哪？

# 快速澄清概念，不进入对话历史：
/btw useEffect 和 useLayoutEffect 有什么区别？

# 确认已做的工作，不转移当前任务：
/btw 我们之前更新过 types.ts 里的错误类型吗？
```

**关键特性**：
- Claude 处理响应**期间**就能用，不需要等它回复完
- 回答**不进入对话历史**，不占用 context window
- **无工具访问**：只能基于当前 session 已有的上下文回答，不能读文件、搜网页、执行代码
- **复用 prompt cache**，token 消耗极低
- 按 Space、Enter 或 Escape 关闭，继续工作

心智模型：`/btw` 是子 Agent 的反面——子 Agent 有完整工具访问但从空 context 起步；`/btw` 有完整 session 可见性但没有任何工具。

---

## 键盘快捷键

> **macOS 用户注意**：`Alt+P`、`Alt+T`、`Alt+B`、`Alt+F` 等需要在终端配置 Option 键为 Meta。iTerm2：Settings → Profiles → Keys → Left/Right Option 设为 "Esc+"。Terminal.app：Profiles → Keyboard → 勾选 "Use Option as Meta Key"。

### Session 控制

| 快捷键 | 作用 | 使用时机 |
|--------|------|---------|
| `Ctrl+C` | 取消当前响应 | Claude 走偏了 |
| `Ctrl+F` | 杀掉所有后台 agent（按两次确认）| 子 Agent 在烧 token |
| `Ctrl+D` | 退出 Claude Code | 结束 session |
| `Ctrl+L` | 清屏 | 终端太乱 |
| `Esc Esc` | 回退到某个时间点或生成摘要 | Claude 犯了错误，撤销它 |

`Esc Esc` 双击非常被低估——它能恢复代码和对话到之前的状态，比 `Ctrl+C` 更适合"撤销"而非"停止"的场景。

### 输入与导航

| 快捷键 | 作用 |
|--------|------|
| `Up/Down` | 滚动 prompt 历史 |
| `Ctrl+R` | 反向搜索历史（输入片段匹配过去的 prompt）|
| `Ctrl+G` | 打开外部文本编辑器（写长 prompt 的最佳方式）|
| `Ctrl+V` / `Cmd+V` | 从剪贴板粘贴图片（分享截图调试）|
| `Shift+Tab` 或 `Alt+M` | 切换权限模式（plan/auto-accept/normal）|

### 模型与思考控制

| 快捷键 | 作用 |
|--------|------|
| `Alt+P` | 切换模型（session 中途换 Sonnet/Opus）|
| `Alt+T` | 开关 extended thinking（复杂问题深度推理）|
| `Ctrl+O` | 开关详细输出（查看完整工具调用细节）|
| `Ctrl+T` | 开关任务列表 |
| `Ctrl+B` | 将当前任务移到后台 |

### 输入行文字编辑

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+K` | 删除光标到行尾 |
| `Ctrl+U` | 删除整行 |
| `Ctrl+Y` | 粘贴上次删除的内容 |
| `Alt+Y` | 循环粘贴历史 |
| `Alt+B` | 向前跳一个词 |
| `Alt+F` | 向后跳一个词 |

---

## 多行输入的四种方式

| 方式 | 操作 |
|------|------|
| 反斜杠 + 回车 | 行末输入 `\`，然后 Enter |
| Option+Enter（macOS）/ Shift+Enter | 插入换行不发送 |
| `Ctrl+J` | 在输入中插入换行 |
| 粘贴多行文本 | 自动进入多行模式 |

长 prompt 仍推荐 `Ctrl+G` 打开外部编辑器——在真正的编辑器里写，保存关闭后自动发送。

---

## 三种前缀输入模式：`/`、`!`、`@`

### `/`：Slash 命令与 Skills

输入 `/` 查看所有可用命令和项目中定义的自定义 skills，边输入边过滤。

### `!`：Bash 模式

输入 `!` 加任意 shell 命令，直接在 shell 执行，输出进入对话 context。

```bash
# 不离开 session 运行 git 命令
! git status

# 查看文件内容
! cat src/config.ts

# 跑测试，输出自动进入 context
! npm test
```

Claude 建议了某个命令，你想跑一个修改版本？直接 `!` 去。跑完后可以接着说"帮我修这次测试跑出来的报错"，Claude 看得到完整输出。Tab 补全对 `!` 后的命令也有效。

### `@`：文件路径引用

输入 `@` 加文件路径，明确告诉 Claude 看这个文件。支持 Tab 补全。

---

## Slash 命令完整索引

### Session 管理

| 命令 | 作用 |
|------|------|
| `/add-dir` | 添加新工作目录到当前 session |
| `/clear` | 重置对话历史（同时重置该目录的命令历史）|
| `/compact` | 压缩 context 释放 token 空间 |
| `/exit` | 退出 Claude Code |
| `/fork` | 把对话分支到新 session |
| `/resume` | 恢复之前的 session |
| `/rename` | 重命名当前 session |
| `/rewind` | 撤销上一轮 |

> `/compact` 是长 session 的必备操作。context 窗口快满时主动运行，而不是等自动压缩触发。注意：`/clear` 同时清除命令历史，想保留历史访问能力用 `/compact`。

### 信息与状态

| 命令 | 作用 |
|------|------|
| `/cost` | 查看本 session token 用量和费用 |
| `/usage` | 查看计划用量和速率限制 |
| `/stats` | 查看 session 统计 |
| `/diff` | 查看最近文件变更 |
| `/doctor` | 诊断常见配置问题 |
| `/context` | 以彩色网格可视化 context 用量 |

### 配置

| 命令 | 作用 |
|------|------|
| `/config` | 打开设置 |
| `/memory` | 编辑 CLAUDE.md 记忆文件 |
| `/model` | 切换当前模型 |
| `/permissions` | 管理工具权限 |
| `/theme` | 更改语法高亮和显示主题 |
| `/vim` | 切换 vim 编辑模式 |
| `/terminal-setup` | 配置终端集成 |

### 工具与集成

| 命令 | 作用 |
|------|------|
| `/mcp` | 管理 MCP server 连接 |
| `/hooks` | 管理自动化 hooks |
| `/ide` | 连接 IDE 集成 |
| `/chrome` | 连接 Chrome 进行浏览器自动化 |

### 协作

| 命令 | 作用 |
|------|------|
| `/agents` | 管理运行中的子 Agent |
| `/tasks` | 查看和管理任务列表 |
| `/plan` | 进入规划模式 |
| `/sandbox` | 在隔离沙盒中运行 |
| `/pr-comments` | 获取 GitHub PR 的评论 |
| `/security-review` | 分析分支变更的安全漏洞 |
| `/btw` | 问旁白问题，不影响对话历史 |

> `/review` 已废弃，替代品是 `/simplify`（并行三 Agent 代码审查）。

---

## Vim 模式：输入框内的完整模态编辑

`/vim` 启用 vim 键位绑定。不是简化版模拟，是完整实现：

| 类别 | 命令 |
|------|------|
| 模式切换 | `i`、`I`、`a`、`A`、`o`、`O`、`Esc` |
| 导航 | `h/j/k/l`、`w/b/e`、`0/$`、`^`、`gg/G` |
| 字符跳转 | `f{char}`、`F{char}`、`t{char}`、`T{char}`、`;`、`,` |
| 编辑 | `d`、`dd`、`D`、`c`、`cc`、`C`、`x`、`J`、`.`、`>>`、`<<` |
| 文本对象 | `iw`、`aw`、`iW`、`aW`、`i"`、`a"`、`i(`、`a(`、`i{`、`a{` |
| 剪贴板 | `y`、`yy`、`p`、`P` |

再次 `/vim` 关闭。模式在 session 内跨 prompt 保持，新 session 重置。

---

## 后台任务（Ctrl+B）

Claude 处理长响应时，按 `Ctrl+B` 把当前任务推到后台，Claude 继续工作，你拿回 prompt 可以干别的。

> Tmux 用户：按 `Ctrl+B` 两次（tmux 占用了同一前缀键）。

- 输出在后台缓冲，切回时显示
- 多个任务可同时运行
- `Ctrl+F` 杀掉所有后台 Agent
- `Ctrl+T` 查看后台任务状态

适合后台的场景：大型代码生成、跑测试套件、Claude 读取多个文件的研究任务、任何盯着输出流没有价值的任务。

---

## Prompt 建议

Claude Code 根据 git 历史和对话 context 自动生成 prompt 建议，显示在输入框下方，Tab 接受。

- 刚 commit 后可能建议"为上次 commit 的变更写测试"
- 有未提交改动时可能建议"审查当前 diff 的问题"
- 复用 cache，额外成本极低
- 随对话深入越来越准确

---

## 命令历史

- Up/Down 箭头滚动历史，`Ctrl+R` 反向增量搜索
- **历史按目录隔离**，不同项目不互相污染
- **`/clear` 同时清除命令历史**；想保留历史用 `/compact`

---

## 任务列表（Ctrl+T）

`Ctrl+T` 切换任务列表 overlay，跟踪 session 内多步工作。任务列表在 context 压缩后依然保留。

团队工作流：设置环境变量 `CLAUDE_CODE_TASK_LIST_ID` 在多个 Claude Code session 间共享任务列表，适合并行处理同一项目不同部分时统一查看进度。

---

## PR 审阅状态

在有开放 PR 的分支工作时，footer 显示 PR 状态，链接下划线颜色代表审阅状态：

| 颜色 | 状态 |
|------|------|
| 绿色 | 已批准 |
| 黄色 | 等待审阅 |
| 红色 | 需要修改 |
| 灰色 | 草稿 PR |
| 紫色 | 已合并 |

每 60 秒自动更新，不用切浏览器。需要安装并认证 `gh` CLI（`gh auth login`）。

---

## 功能组合模式

**深度工作**：`Ctrl+B` 把复杂实现推后台 → `/btw` 问旁白问题不污染主任务 context → `Ctrl+T` 查进度

**快速迭代**：`/fast` 开快速模式 → `!` 跑测试 → `Ctrl+R` 召回历史 prompt 修改后重跑

**严谨工作**：Plan 模式分析 → Vim 模式写详细 prompt → 退出 Plan 模式执行；长指令用 `Ctrl+G` 打开完整编辑器

**多 Agent 协调**：`/agents` 查运行中的子 Agent → `Ctrl+T` 追踪任务 → `Ctrl+F` 杀掉偏离的 → `/btw` 询问状态不打断主对话

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/interactive-mode

## Related

- [[Voice Mode：对着终端说话]]
- [[输出格式与文件变更审批：d/e/y/n 四个快捷键]]
- [[`/powerup`：Claude Code 内置交互式教程机制解析]]
