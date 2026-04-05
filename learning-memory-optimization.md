# Claude Code Memory 优化深度解析

> 参考：https://claudefa.st/blog/guide/mechanics/memory-optimization  
> 源码路径：`src/utils/claudemd.ts`、`src/memdir/memdir.ts`

---

## 一、核心概念

CLAUDE.md 是 Claude Code 的**持久化记忆系统**。与会话历史不同，它跨 session 保留，在每次启动时自动加载，内容被视为**高优先级指令**——比聊天消息和 session 中临时读取的文件内容优先级更高。

**本质**：把它当作"配置"而不是"文档"。

---

## 二、五种记忆类型与加载顺序

源码 `src/utils/claudemd.ts:1-9` 明确定义了四种记忆类型，加载顺序如下：

| 优先级 | 记忆类型 | 位置 | 用途 | 共享范围 |
|--------|---------|------|------|---------|
| 最低 | **Managed（托管策略）** | 系统路径（见下） | 组织级指令 | 所有用户 |
| ↑ | **User（用户记忆）** | `~/.claude/CLAUDE.md` | 个人全局偏好 | 仅你自己（所有项目） |
| ↑ | **Project（项目记忆）** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` + `./.claude/rules/*.md` | 团队共享指令 | 团队（版本控制） |
| **最高** | **Local（本地记忆）** | `./CLAUDE.local.md` | 个人项目私有偏好 | 仅你自己（当前项目） |

**关键加载机制**（源码 `claudemd.ts:9`）：
> "Files are loaded in reverse order of priority, i.e. the latest files are highest priority"

- **越靠近当前目录 = 越晚加载 = 优先级越高**
- Local > Project（当前目录）> Project（父目录）> User > Managed

### Managed 托管策略路径（平台差异）

通过 `getManagedFilePath()` 确定：
- macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
- Linux: `/etc/claude-code/CLAUDE.md`
- Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`

---

## 三、目录树遍历机制

Claude Code 从**当前工作目录向上遍历**到根目录，在每一级目录检查：
- `CLAUDE.md`
- `.claude/CLAUDE.md`
- `.claude/rules/*.md`（Rules 目录）

每一级都可能有一个 CLAUDE.md，**形成记忆层级**，适合 monorepo 或多包项目。

### 子目录 CLAUDE.md：懒加载

子目录（低于工作目录）的 CLAUDE.md **不在启动时加载**，只有当 Claude 读取该子目录下的文件时才动态加载。

示例：`packages/auth/CLAUDE.md` 只有在 Claude 操作 `packages/auth/` 内的文件时才会生效，保持启动轻量。

---

## 四、`@import` 指令（文件组合）

源码 `claudemd.ts:18-25`，支持把 CLAUDE.md 从单一文件变成**可组合的指令网络**：

```
@path/to/file           # 相对路径（@不带前缀等价于@./）
@./relative/path        # 显式相对路径（相对于包含该指令的文件，不是工作目录）
@~/home/path            # home 目录相对路径
@/absolute/path         # 绝对路径
```

**重要规则**：
- 只在叶节点文本中生效，**代码块（```）内的 @path 不触发 import**（安全机制）
- 循环引用自动检测并阻断
- 不存在的文件**静默忽略**（不报错）
- 最大递归深度：`MAX_INCLUDE_DEPTH = 5`（源码 `claudemd.ts:537`）
- 支持所有主流文本格式（.md/.ts/.py/.json 等，排除二进制文件）

**外部 import 审批**：第一次遇到项目外的 import 时弹出批准对话框，结果持久化到 `config.hasClaudeMdExternalIncludesApproved`，**一次决定，永久有效**（拒绝后不再弹出）。

**特例**：User 记忆（`~/.claude/CLAUDE.md`）可**始终** import 外部文件，无需审批（源码 `claudemd.ts:833`）。

---

## 五、`CLAUDE.local.md`

- **自动 gitignore**（源码 `claudemd.ts:865`），无需手动配置
- 与 `CLAUDE.md` 同等加载优先级（实际上更高，因为是 Local 类型）
- 存放：sandbox URL、个人测试数据、个人工作流偏好等不应提交的内容
- git worktree 场景：多个 worktree 共享 home 目录 import 是推荐做法

---

## 六、`.claude/rules/` 目录（条件规则）

解决"CLAUDE.md 单文件问题"：当所有指令堆在一个文件时，无关指令也占据高优先级上下文。

Rules 目录让你**按文件路径 glob 范围化**指令生效时机：

```
.claude/rules/
  api-guidelines.md       # 可在 frontmatter 指定 globs: ["src/api/**/*.ts"]
  react-components.md     # globs: ["src/**/*.tsx"]
  security.md             # 全局生效（无 glob）
```

**同等优先级**：Rules 文件与 CLAUDE.md 优先级相同，但 glob 限定了"何时"高优先级生效。

**用户级 rules**：`~/.claude/rules/` 适用于所有项目的个人规则，项目级 rules 优先级更高（后加载）。

**brace expansion** 支持：`src/**/*.{ts,tsx}`、`{src,lib}/**/*.ts`。

**symlinks** 支持：可用 symlink 在多个项目间共享 rules，循环 symlink 自动处理。

---

## 七、文件大小限制

源码 `claudemd.ts:92`：

```typescript
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

**单个 CLAUDE.md 文件上限是 40,000 字符**（不是文章说的 400 行）。超出时工具会给出警告。

实践建议：超出时用 `@import` + Rules 目录拆分，而不是压缩内容。

---

## 八、Auto Memory（MEMORY.md）

Claude 自己记录的笔记，存储在 `~/.claude/projects/<project>/memory/`。

启动时加载 `MEMORY.md` 的**前 200 行**（源码 `memdir/memdir.ts:35`：`MAX_ENTRYPOINT_LINES = 200`）。超出时会有警告提示保持条目精简（每条约 200 字符以内）。

与 CLAUDE.md 的关系：并行加载，互补而非竞争。

---

## 九、`/memory` 命令

session 中随时执行，作用：
- 查看当前已加载的所有 CLAUDE.md 文件及 imports
- 在系统编辑器中直接打开任意记忆文件
- 排查指令缺失或不生效的问题

---

## 十、最佳实践

### 结构建议

```markdown
# 项目技术栈
...永久固定内容...

# 当前焦点（每次 session 手动更新）
...session 特定上下文...
```

"Current Focus" 区域分离临时 context，避免污染永久指令。

### 清理上下文

```
/clear    # 清空对话历史，CLAUDE.md 保留
exit      # 完全重启（彻底清除所有上下文）
```

### 不同任务用不同目录

每个目录有自己的 CLAUDE.md，形成隔离上下文，适合 feature branch 或实验工作。

### Git 版本控制 CLAUDE.md

指令更改出问题时可以立即回滚：

```bash
git checkout HEAD~1 -- CLAUDE.md
```

---

## 十一、常见问题排查

| 症状 | 原因 | 解决方案 |
|------|------|---------|
| 指令被忽略 | CLAUDE.md 过大或内容混乱 | 保持 40,000 字符以内；用 Rules 目录拆分 |
| 项目间 context 混淆 | 没有项目级 CLAUDE.md | 每个项目根目录创建独立 CLAUDE.md |
| 上下文变得陈旧 | 对话积累无关历史 | `/clear` 重置对话，CLAUDE.md 保留 |
| 外部 import 不生效 | 未批准外部 includes | 第一次访问时在审批对话框中确认 |

---

## 十二、与 Session Memory 的关系

| 特性 | CLAUDE.md | Session Memory |
|------|-----------|---------------|
| 创建者 | 用户（手动） | Claude（自动） |
| 范围 | 永久持久 | 每 session 快照 |
| 优先级 | 高优先指令 | 背景参考 |
| 最佳用途 | 规范、架构、命令 | 跨 session 连续性 |

`/remember` 命令是两者的桥梁：把 Session Memory 中反复出现的模式提升为 CLAUDE.md 永久规则。
