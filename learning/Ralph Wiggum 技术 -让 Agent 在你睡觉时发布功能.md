---
title: "Ralph Wiggum 技术：让 Agent 在你睡觉时发布功能"
aliases:
  - "learning-ralph-wiggum"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "自主 Agent 循环：睡觉时也在发布功能"
  - "Robots-First Engineering：为 AI 设计的工程体系"
---

# Ralph Wiggum 技术：让 Agent 在你睡觉时发布功能

> **2025 年 1 月更新**：Anthropic 已内置原生 [task management](https://claudefa.st/blog/guide/development/task-management)，支持依赖关系、阻断项和多 Session 协调（`CLAUDE_CODE_TASK_LIST_ID`）。很多 Ralph 的 workaround 现在已是内置功能。但核心原则依然适用。

---

## 什么是 Ralph Wiggum？

Ralph 是一个自主 AI 编码循环。核心流程：

```
你给出任务列表
    ↓
Agent 选一个任务 → 实现 → 测试 → 提交
    ↓
自动选下一个任务 → 实现 → 测试 → 提交
    ↓
整晚循环，直到所有任务完成
```

大多数开发者用 Claude Code 的方式：`Prompt → 等待 → 审查 → Prompt`，你是瓶颈。

Ralph 翻转模型：你设置好循环，Agent 工作直到验证通过，你只在开始和结束出现。

---

## 核心机制：完成承诺（Completion Promise）

Ralph 的关键创新是**完成承诺**——一个特定词或短语，表示真正完成。

```
Agent 认为任务完成
    ↓
Stop Hook 拦截退出尝试
    ↓
检查：是否输出了完成承诺（如 "complete"）？
    ↓ 否                        ↓ 是
继续迭代                    允许退出
```

**关键规则**：Agent 只有在输出完成承诺之后才能停止。这迫使 Agent 持续迭代，直到它真正相信工作已完成。

完成承诺防止两种失败：
- 过早退出（Agent 觉得差不多了就停）
- 无限循环（没有明确的退出信号）

---

## 验证驱动：Boris Cherny 的唯一规则

> "永远给 Claude 一种验证自己工作的方式。"

没有验证 = 要么永远跑，要么过早停。三种验证方式：

### 1. 测试驱动验证（最可靠）

先写测试，再实现。测试要么通过要么失败，没有歧义。

```
写测试 → Agent 实现 → 跑测试 → 失败 → 继续实现 → ... → 全部通过 → 输出 "complete"
```

### 2. 后台 Agent 验证

派生一个独立 Agent 检查主 Agent 的工作。独立检查 = 更客观的结果。

```
主 Agent 实现 → 后台验证 Agent 审查 → 发现问题 → 主循环继续
```

### 3. Stop Hook 验证

Stop Hook 本身运行验证命令：检查进度文件、跑 lint、验证构建状态。

---

## 两阶段分离：规划 vs 实现

最常见的错误：在同一个 context window 里规划和实现。

**必须分开：**

```
Phase 1（规划 Session）          Phase 2（实现 Session）
────────────────────             ────────────────────
对话生成规格                      清空 context
手动审查编辑                      只喂规格文档
创建含文件引用的实现计划            跑 Ralph 循环
保存为"锚点"文档                  Agent 迭代直到完成
```

**为什么分开？** Context 窗口越用越脏。经过太多来回，Claude 开始基于早期已不相关的消息做假设。全新 context + 干净计划 = 更清晰的专注。

计划文档是**锚点**，防止 Agent 自由发明。每次循环迭代都参考它，而不是漂移。

---

## Ryan Carson 的任务结构化方法

1. **从 PRD 开始**（产品需求文档）
   - 我们在构建什么？
   - 范围内是什么？
   - 明确范围外是什么？

2. **转化为带验收标准的用户故事**
   - 每个故事是小的可测试单元
   - 验收标准定义"完成"

3. **结构化为 Agent 可消费的格式**
   ```markdown
   - [ ] 用户故事 1：指标展示
     - 验收：加载时间 < 2 秒
     - 相关代码：src/components/Dashboard.tsx
   - [ ] 用户故事 2：活动流
     - 验收：展示最近 30 天
   ```

4. **运行循环**：Agent 选下一个未完成故事 → 实现 → 运行验证 → 标记完成 → 选下一个

---

## UI 验证：截图协议

功能测试通过 ≠ UI 没问题。按钮可能跑到屏幕外，文字可能被截断。

**强制视觉验证的两轮循环：**

```
第一轮：
  实现 UI
  对受影响组件截图
  截图重命名为 "reviewed_" 前缀（表示已审查）
  ← 此时不能输出完成承诺

第二轮：
  确认所有截图已命名为 "reviewed_"
  确认无视觉问题
  ← 现在才可以输出 "complete"
```

**关键细节**：重命名截图后，告诉 Claude 还不能输出承诺，让下一轮循环确认完成。这阻止过早退出。

---

## 检查点状态：context 重置后继续

对于长时运行的 L-Thread，维护外部进度文件：

```markdown
## 进度追踪

### 已完成
- [x] 搭建测试基础设施
- [x] 实现 API 端点

### 进行中
- [ ] 创建 UI 组件

### 待完成
- [ ] 添加导出功能
```

Context 填满后重启，Agent 读取进度文件，从中断处继续。

---

## Stop Hook 配置（`settings.json`）

```json
{
  "hooks": {
    "stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "node /path/to/stop-hook.js"
          }
        ]
      }
    ]
  }
}
```

Stop Hook 逻辑示例：

```javascript
// stop-hook.js
module.exports = async function(context) {
  // 1. 检查测试
  const testResult = await runTests()
  if (testResult.failed > 0) {
    return {
      decision: 'block',
      reason: `${testResult.failed} 个测试失败，继续工作。`
    }
  }

  // 2. 检查完成承诺
  if (!context.output.includes('complete')) {
    return {
      decision: 'block',
      reason: '未找到完成承诺，确认所有工作已完成后再输出 complete。'
    }
  }

  return { decision: 'allow' }
}
```

---

## 失败模式与修复

| 现象 | 原因 | 修复 |
|------|------|------|
| 循环永不结束 | 任务不可能完成，或缺少完成标准 | 设最大迭代次数（如 25 次），加明确完成标准 |
| 循环过早结束 | 完成承诺被过早输出 | 加强验证，让"完成"客观可测，UI 加截图协议 |
| 迭代中质量下降 | Context 被失败尝试填满 | 实现检查点状态，让循环在 context 重置后能恢复 |
| Agent 自由发明 | 规格模糊或缺失 | 规格是锚点，必须具体，包含现有代码引用，明确写清不要做什么 |

---

## 经济账

用 Sonnet 持续跑 Agent：**约 $10.42/小时**（24 小时烧率实测）。

这低于大多数地方的最低工资。一台能清理积压、并行跑多功能、永不疲倦的机器。

约束不是成本，而是**你能定义多少可靠的工作**。

---

## 入门路径

1. 选一个有现有测试的、定义明确的功能
2. 建立 Stop Hook（或用内置 task management）
3. 创建 Prompt 文件（计划文档）
4. 设置约束：最大迭代 25 次，完成承诺 "complete"，质量门：测试 + lint
5. **先看着第一次运行**——还不要走开。如果行为不对，取消，调整 Prompt，重跑
6. 随着信任建立，逐步增加自主度

---

## 与其他机制的关系

- **L-Thread**（`learning-thread-based-engineering.md`）：Ralph 是 L-Thread 的具体实现
- **Stop Hook**（`learning-autonomous-loops.md`）：Ralph 的技术基础，与 Thread-Based Engineering 共用
- **Sub-Agents**（`learning-multi-agent.md`）：B-Thread 里的后台验证 Agent 模式
- **内置 Task Management**：`CLAUDE_CODE_TASK_LIST_ID` 现在原生支持多 Session 协调，是 Ralph 的官方演进

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/ralph-wiggum-technique

## Related

- [[自主 Agent 循环：睡觉时也在发布功能]]
- [[Robots-First Engineering：为 AI 设计的工程体系]]
