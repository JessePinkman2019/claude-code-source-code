---
title: "输出格式与文件变更审批：d/e/y/n 四个快捷键"
aliases:
  - "learning-output-formatting"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考"
  - "Voice Mode：对着终端说话"
---

# 输出格式与文件变更审批：d/e/y/n 四个快捷键

## 核心认知：Claude Code 不是代码生成器

Claude Code 不生成代码让你粘贴，它是一个直接写入你文件系统的 Agent。

两个核心工具：
- **Write**：创建新文件或完全替换现有文件
- **Edit**：对现有文件做精准的局部修改

每次文件修改都需要你审批。这是核心安全机制。

---

## 四个必会快捷键

| 键 | 动作 |
|----|------|
| `y` | 接受变更 |
| `n` | 拒绝变更 |
| `d` | 查看完整 diff（再决定） |
| `e` | 在接受前手动编辑 |

**最重要的习惯**：养成先按 `d` 看 diff，再按 `y` 接受的习惯。Diff 格式：`-` 红色行被删除，`+` 绿色行被添加。

---

## 源码中的审批流

### 文件权限对话框

审批流的 UI 核心在：
- `src/components/permissions/FilePermissionDialog/FilePermissionDialog.tsx` — 主对话框容器
- `src/components/permissions/FilePermissionDialog/useFilePermissionDialog.ts` — 管理接受/拒绝逻辑的 hook（约第 150 行注册 keybinding context "Confirmation"）
- `src/components/permissions/FileEditPermissionRequest/FileEditPermissionRequest.tsx` — Edit 工具的权限请求 UI
- `src/components/permissions/FileWritePermissionRequest/FileWritePermissionRequest.tsx` — Write 工具的权限请求 UI

### Diff 计算与展示

- `src/tools/FileWriteTool/UI.tsx`：`loadRejectionDiff()` 异步计算 diff，`WriteRejectionDiff` 组件渲染
- `src/tools/FileEditTool/UI.tsx`：同样的 `loadRejectionDiff()` + `EditRejectionDiff` 组件
- `src/utils/diff.js`：`getPatchForDisplay()` — 底层 diff 工具函数
- `src/components/FileEditToolUseRejectedMessage.tsx` — 展示被拒绝的编辑及其 diff

---

## Auto-Accept 模式（`Shift+Tab`）

对于信任的操作，不想每次手动审批时使用。

**切换**：`Shift+Tab`（Windows 无 VT 模式下为 `Meta+M`）

源码路径：
- `src/keybindings/defaultBindings.ts:69` — 绑定 `shift+tab` → `'chat:cycleMode'`
- `src/utils/permissions/PermissionMode.ts` — 定义权限模式，含 `autoAccept`
- `src/utils/permissions/getNextPermissionMode.ts` — 循环切换各模式的逻辑

**注意**：Auto-accept 只应在充分信任操作时开启，适合 L-Thread/Ralph 循环等场景。

---

## 多文件操作

Claude 顺序处理多个文件，并在文件间保持上下文：

```
创建 schema.ts → 记住类型定义
    ↓
创建 api.ts → 使用 schema.ts 中的类型，import 路径自动正确
    ↓
创建 ui.tsx → 调用 api.ts 中的函数，签名保持一致
```

**优势**：不需要你手动维护跨文件的 import 路径、类型引用、函数签名一致性。

---

## 控制输出格式

### 想看代码但不写文件

用 Plan Mode（`/plan`）：分析和建议，不触发写文件工作流。

或者 Prompt 前缀用 "show me" / "explain" 替代 "create" / "add"：Claude 输出代码块，不触发 Write 工具。

### 控制响应格式

Claude 默认用 Markdown 回复，通过 Prompt 明确要求格式：
- "以 JSON 格式输出"
- "只输出代码，不要解释"
- "分步骤列出"

### Diff 太长看不过来

- 按 `d` 查看完整 diff
- 或告诉 Claude："把这个变更拆成更小的步骤"

---

## 撤销：`Esc` 两次

Claude 在每次变更前自动创建快照（checkpoint）。

按两次 `Esc` 访问检查点，可以回退到变更前的状态。

源码中双键检测参考 `src/commands/fast/fast.tsx`（约第 175 行），`ctrl+c` / `ctrl+d` 有类似的时间窗口双击检测逻辑。

---

## 常见场景速查

| 场景 | 做法 |
|------|------|
| 不确定 Claude 改了什么 | 先按 `d` 看 diff |
| Claude 改错了文件 | 按 `n` 拒绝，解释你实际需要什么 |
| 想在接受前微调 | 按 `e` 手动编辑 |
| 大量重复操作不想逐一审批 | `Shift+Tab` 开启 auto-accept |
| 后悔接受了某个变更 | 按两次 `Esc` 回滚到检查点 |
| 只想看代码不想写文件 | Prompt 用 "show me" 或开 Plan Mode |

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/output-formatting

## Related

- [[Claude Code Interactive Mode：快捷键、命令与隐藏功能完整参考]]
- [[Voice Mode：对着终端说话]]
