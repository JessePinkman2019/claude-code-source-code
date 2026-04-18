# Claude Code 知识库

这是一个以 **Obsidian** 为体系的知识库项目，记录 Claude Code 的源码分析、工程实践与 AI 研究笔记。

## Obsidian 规则

**所有新增 md 文件必须遵循 Obsidian 格式**：

1. 文件顶部加 YAML frontmatter（`tags`、`source`、`date` 等字段）
2. 用 `[[文件名]]` 语法建立内部链接（wiki links），仅在原文有明确引用关系时使用
3. 合理使用 `#标签` 标注主题
4. 相关笔记之间互相 wiki link，形成知识网络

目录结构：
- `learning/` — Claude Code 机制笔记（源码级分析）
- `engineering/` — Anthropic 工程博客摘要
- `papers/` — AI 研究论文笔记

## Obsidian Skills 使用规则

本项目使用 `obsidian-skills` 套件处理所有笔记相关操作：

| 场景 | 使用工具 | 说明 |
|------|----------|------|
| 从网页抓取文章 | `defuddle parse <url> --md` | 替代 WebFetch，输出更干净，节省 token |
| 创建/编辑笔记 | `obsidian-markdown` skill | 确保 wikilink、callout、frontmatter 格式正确 |
| 操作 Obsidian vault | `obsidian-cli` | 需要 Obsidian 处于运行状态 |
| 创建数据库视图 | `obsidian-bases` skill | `.base` 文件，table/cards/list 视图 |
| 创建白板 | `json-canvas` skill | `.canvas` 文件 |

**网页抓取强制规则**：用户提供 URL 要求写入笔记时，必须优先使用 `defuddle`，不使用 WebFetch。`.md` 结尾的 URL 除外（直接用 WebFetch）。

## 飞书操作规则

以 bot 身份创建飞书文档后，**必须**立即为王朗（open_id: `ou_48ce500f9446f3c2043d5200fd158bef`）授予 `full_access` 权限：

```bash
lark-cli drive permission.members create \
  --params '{"token":"<doc_id>","type":"docx","need_notification":"false"}' \
  --data '{"type":"user","member_type":"openid","member_id":"ou_48ce500f9446f3c2043d5200fd158bef","perm":"full_access","perm_type":"container"}'
```
