---
title: "CLAUDE.md 编写指南（基于 v2.1.88 源码）"
aliases:
  - "learning-claudemd"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Claude Code Memory 优化深度解析"
  - "Claude Code Library 系统：跨项目管理 .claude 配置"
  - "Claude Code 自动记忆：AI 的项目笔记本"
---

# CLAUDE.md 编写指南（基于 v2.1.88 源码）

> 所有建议均直接对应源码，标注具体文件和行号。

---

## 1. 文件体系：4 层加载顺序

源码 `src/utils/claudemd.ts:1-26` 注释明确了加载顺序，**越靠后优先级越高**（后加载的内容在 prompt 中位置更靠后，模型更关注）：

```
优先级（低→高）
1. Managed   /etc/claude-code/CLAUDE.md          企业/系统管理员下发，用户无法覆盖
2. User      ~/.claude/CLAUDE.md                 个人全局配置，对所有项目生效
3. Project   ./CLAUDE.md 或 .claude/CLAUDE.md    项目配置，入库共享
4. Local     ./CLAUDE.local.md                   本地私有配置，最高优先级，不入库
```

**实践建议**：
- 团队规范 → `CLAUDE.md`（入库）
- 个人偏好、本地路径、调试开关 → `CLAUDE.local.md`（加入 `.gitignore`）
- 全局习惯（如语言偏好、回复风格）→ `~/.claude/CLAUDE.md`

---

## 2. 注入到 system prompt 的格式

`src/utils/claudemd.ts:89-90` 定义了注入前缀：

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.
```

`src/utils/claudemd.ts:1185` 每个文件的注入格式：

```
Contents of /path/to/CLAUDE.md (project instructions, checked into the codebase):

<你的内容>
```

**实践建议**：
- 不需要在文件里重复写"请严格遵守"，系统已经加了
- 直接写规则本身，越具体越好
- 空文件或纯空白内容会被跳过（`claudemd.ts:652`：`!memoryFile.content.trim()` 则返回空）

---

## 3. 字符数上限

`src/utils/claudemd.ts:92`：

```ts
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

超过 40000 字符会被标记为 large file（`getLargeMemoryFiles` 函数，`claudemd.ts:1132`），但不会被截断（截断只针对 AutoMem/TeamMem 类型，`claudemd.ts:383`）。

**实践建议**：
- 单个 CLAUDE.md 控制在 40k 字符以内
- 超出时用 `@include` 拆分（见第 4 节）

---

## 4. `@include` 指令：拆分大型规则

`src/utils/claudemd.ts:18-26` 和 `extractIncludePathsFromTokens`（`claudemd.ts:451`）实现了 `@path` 语法。

### 支持的路径格式（`claudemd.ts:477-483`）

```markdown
@./relative/path.md          相对路径（推荐）
@~/home/path.md              home 目录路径
@/absolute/path.md           绝对路径
@bare-filename.md            等同于 @./bare-filename.md
```

### 路径中有空格时

用反斜杠转义（`claudemd.ts:473`）：

```markdown
@./my\ rules\ file.md
```

### 最大嵌套深度

`src/utils/claudemd.ts:537`：

```ts
const MAX_INCLUDE_DEPTH = 5
```

超过 5 层的 include 会被静默忽略。

### 外部路径（项目目录之外）

`claudemd.ts:667-670`：Project 类型的文件默认**不允许** include 项目目录之外的文件，需要用户在对话中手动批准（`hasClaudeMdExternalIncludesApproved`）。User 类型（`~/.claude/CLAUDE.md`）始终允许 include 外部文件（`claudemd.ts:833`）。

### 支持 include 的文件类型

`claudemd.ts:96-227` 定义了白名单，包含 `.md`、`.ts`、`.py`、`.json`、`.yaml` 等几乎所有文本格式，**不支持**二进制文件（图片、PDF 等）。

### 示例

```markdown
# CLAUDE.md
@./docs/coding-standards.md
@./docs/testing-rules.md
@~/.claude/personal-prefs.md
```

---

## 5. `.claude/rules/` 目录：条件规则

`src/utils/claudemd.ts:697-788`（`processMdRules`）支持在 `.claude/rules/` 下放置带 frontmatter 的 `.md` 文件，只在操作特定文件时才注入上下文。

### frontmatter `paths` 字段

`src/utils/frontmatterParser.ts:52` 和 `claudemd.ts:254-278`：

```markdown
---
paths:
  - src/api/**/*.ts
  - src/routes/**
---

# API 文件规则
所有 handler 必须返回统一的 { code, message, data } 格式。
```

也支持逗号分隔的单行写法（`frontmatterParser.ts:185`）：

```markdown
---
paths: src/api/**/*.ts, src/routes/**
---
```

支持花括号展开（`frontmatterParser.ts:240`）：

```markdown
---
paths: src/**/*.{ts,tsx}
---
```

### glob 匹配的基准目录

`claudemd.ts:1377-1380`：
- Project 类型：相对于 `.claude` 目录的**父目录**（即项目根）
- Managed/User 类型：相对于原始 CWD

### 无 `paths` frontmatter = 无条件加载

没有 `paths` 字段的 `.claude/rules/*.md` 文件会在每次会话都加载（`claudemd.ts:773`：`conditionalRule: false` 时只取无 globs 的文件）。

---

## 6. HTML 注释：写给人类的备注

`src/utils/claudemd.ts:292-334`（`stripHtmlComments`）在加载时自动剥离块级 HTML 注释，这些内容**不会进入 context**：

```markdown
<!-- 这段是给维护者看的说明，Claude 读不到 -->
<!-- 上次修改：2024-01-15，原因：规范了错误处理格式 -->

# 实际规则
所有函数必须有错误处理。
```

**注意**：只剥离块级注释（独占一行或多行的 `<!-- -->`），行内注释（在段落中间的）会保留。代码块内的注释也不会被剥离（`claudemd.ts:496-498`）。

---

## 7. frontmatter 本身会被剥离

`claudemd.ts:356-358`：

```ts
const { content: withoutFrontmatter, paths } = parseFrontmatterPaths(rawContent)
```

frontmatter（`---` 之间的 YAML 块）会被解析后从内容中移除，不会出现在注入的 prompt 里。

---

## 8. 目录向上遍历：父目录的 CLAUDE.md 也会加载

`claudemd.ts:850-857`：从当前工作目录向上遍历到根目录，每一层都会尝试加载 `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md`、`CLAUDE.local.md`。

加载顺序是**从根到 CWD**（`dirs.reverse()`，`claudemd.ts:878`），所以越靠近 CWD 的文件优先级越高。

**实践建议**：
- monorepo 场景下，根目录放通用规则，子包目录放专属规则，子包规则会覆盖根目录规则
- 不想让父目录规则生效，可以在 `settings.json` 里配置 `claudeMdExcludes`

---

## 9. 禁用 CLAUDE.md 的方式

### 完全禁用

`claudemd.ts:165-167`（`context.ts`）：

```bash
CLAUDE_CODE_DISABLE_CLAUDE_MDS=1 claude
```

### 排除特定文件

`settings.json` 中配置 `claudeMdExcludes`（`claudemd.ts:552`），支持 glob 模式：

```json
{
  "claudeMdExcludes": [
    "/path/to/project/CLAUDE.md",
    "**/legacy/**/*.md"
  ]
}
```

只对 User、Project、Local 类型生效，Managed 类型不受影响（`claudemd.ts:547-550`）。

### bare 模式

`claudemd.ts:166-167`：`--bare` 模式下跳过自动发现，但仍然加载通过 `--add-dir` 显式指定的目录。

---

## 10. 去重机制

`claudemd.ts:629-631`：用 `processedPaths` Set 追踪已处理的文件路径，同一个文件不会被加载两次（即使被多个文件 `@include`）。

---

## 11. 完整文件结构示例

```
project/
├── CLAUDE.md                    # 团队共享规则（入库）
├── CLAUDE.local.md              # 个人本地规则（不入库，加入 .gitignore）
└── .claude/
    ├── CLAUDE.md                # 也会被加载（与根目录 CLAUDE.md 并列）
    └── rules/
        ├── general.md           # 无 paths frontmatter，每次都加载
        ├── api-rules.md         # 有 paths frontmatter，只在操作 API 文件时加载
        └── frontend-rules.md    # 有 paths frontmatter，只在操作前端文件时加载
```

`~/.claude/CLAUDE.md` 对所有项目生效，优先级低于项目级文件。

---

## 12. 关键源码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 加载顺序说明 | `src/utils/claudemd.ts` | 1-26 |
| 注入前缀文本 | `src/utils/claudemd.ts` | 89-90 |
| 字符数上限常量 | `src/utils/claudemd.ts` | 92 |
| `@include` 正则 | `src/utils/claudemd.ts` | 459 |
| 最大嵌套深度 | `src/utils/claudemd.ts` | 537 |
| HTML 注释剥离 | `src/utils/claudemd.ts` | 292-334 |
| frontmatter 解析 | `src/utils/frontmatterParser.ts` | 130-175 |
| paths 字段解析 | `src/utils/frontmatterParser.ts` | 189-232 |
| 条件规则匹配 | `src/utils/claudemd.ts` | 1354-1397 |
| claudeMdExcludes | `src/utils/claudemd.ts` | 547-573 |
| 目录向上遍历 | `src/utils/claudemd.ts` | 850-934 |
| 注入格式拼接 | `src/utils/claudemd.ts` | 1153-1195 |

## Related

- [[Claude Code Memory 优化深度解析]]
- [[Claude Code Library 系统：跨项目管理 .claude 配置]]
- [[Claude Code 自动记忆：AI 的项目笔记本]]
