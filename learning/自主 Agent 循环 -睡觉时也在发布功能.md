---
title: "自主 Agent 循环：睡觉时也在发布功能"
aliases:
  - "learning-autonomous-loops"
area: learning
tags: []
status: evergreen
source: ""
source_type: personal-learning-note
related:
  - "Multi-Agent 协作机制（基于 v2.1.88 源码）"
  - "Ralph Wiggum 技术：让 Agent 在你睡觉时发布功能"
  - "`/powerup`：Claude Code 内置交互式教程机制解析"
---

# 自主 Agent 循环：睡觉时也在发布功能

## 两个框架的统一

本文把两个框架结合成一套完整系统：

- **Ralph Wiggum 循环**：如何让 Agent 持续自主运行（Loop 机制）
- **Thread-Based Engineering**：如何扩展和衡量这种自主性（扩展框架）

```
Thread 类型 × 验证机制 → 可靠的自主工作
        ↓
Ralph 循环 = L-Thread 的具体实现
        ↓
结果：功能在你睡觉时自动发布
```

---

## 验证是一切的基础

Boris Cherny 的一条规则：**永远给 Claude 一种验证自己工作的方式。**

不同 Thread 类型对应不同验证方式：

| Thread 类型 | 验证方式 |
|------------|---------|
| Base（基础）| 人工审查 |
| P-Thread（并行）| 并行审查、共识比较 |
| C-Thread（链式）| 逐阶段验证 |
| F-Thread（融合）| 多输出对比 |
| B-Thread（大型）| 子 Agent 验证 |
| L-Thread（长时）| 自动化测试 + Stop Hook |

**关键洞察**：Thread 越长、越自主，验证就必须越自动化。一个跑 26 小时的 L-Thread 你没法手动审查，系统必须自我验证。

---

## 完整技术栈的五层搭建

### 第一层：规格说明（锚点）

每次自主运行从规格开始。规格是你的"锚点"——防止 Agent 自由发明。

```markdown
## 功能：用户仪表板

### 范围内
- 展示用户指标
- 显示近期活动
- 添加导出功能

### 范围外
- 实时更新（Phase 2）
- 移动端响应式（Phase 2）

### 验收标准
- [ ] 指标加载时间 < 2 秒
- [ ] 活动记录显示最近 30 天
- [ ] 导出生成有效的 CSV
```

好的规格：尽量引用现有代码、明确告诉 Agent 不要做什么、客观定义"完成"的标准。

### 第二层：测试驱动验证

**先写测试，再实现**。测试成为让 L-Thread 可靠的验证层。

```
tests/
  dashboard/
    metrics.test.ts    # 验证指标加载时间
    activity.test.ts   # 验证活动展示
    export.test.ts     # 验证 CSV 生成
```

Agent 运行时持续执行测试，循环在测试通过前不结束——没有歧义，没有提前退出。

### 第三层：Stop Hook（执行者）

Stop Hook 是强制执行者，不管 Claude 觉得自己做完了没有，它只关心测试是否通过：

```javascript
// stop-hook.js
module.exports = async function(context) {
  // 跑测试套件
  const testResult = await runTests()
  
  if (testResult.failed > 0) {
    return {
      decision: 'block',
      reason: `${testResult.failed} 个测试失败，继续工作。`
    }
  }
  
  // 检查完成承诺
  if (!context.output.includes('complete')) {
    return {
      decision: 'block', 
      reason: '未找到完成承诺，确认所有工作已完成。'
    }
  }
  
  return { decision: 'allow' }
}
```

### 第四层：选择正确的 Thread 类型

| 场景 | Thread 类型 |
|------|------------|
| 小功能，改一个文件 | Base Thread，prompt → 等待 → 审查 |
| 5 个独立功能 | P-Thread，开 5 个终端各跑一个 |
| 数据库迁移分三个阶段 | C-Thread，每个阶段验证后再继续 |
| 重要架构决策 | F-Thread，3 个 Agent 给出方案，对比选最好的 |
| 隔夜功能构建 | L-Thread + Ralph 循环，睡前启动 |
| 多文件重构含子任务 | B-Thread，编排者派生 Worker 各处理一个文件 |

### 第五层：检查点状态

对于 L-Thread，维护外部状态文件：

```markdown
## 进度：用户仪表板

### 已完成
- [x] 搭建测试基础设施
- [x] 实现指标 API 端点
- [x] 创建指标展示组件

### 进行中
- [ ] 实现活动流

### 待完成
- [ ] 添加导出功能
- [ ] 性能优化
```

Agent 工作时更新这个文件。Context 填满重启后，读取进度文件从上次中断的地方继续。

---

## UI 验证：容易被忽略的一环

功能测试通过了，但 UI 可能还是坏的。任何涉及 UI 的 Thread 都要加截图验证：

```
UI 工作的扩展工作流：
1. 完成实现
2. 对受影响的组件截图
3. 逐一审查截图，检查视觉问题
4. 将已验证的截图重命名为 "verified_" 前缀
5. 此时还不能输出完成承诺
6. 再跑一轮循环，确认所有截图都已验证
7. 然后才输出 "complete"
```

强制视觉验证——Claude 不能跳过截图审查就声称完成。

---

## 经济账

用 Sonnet 持续跑 Agent 大约 **$10.42/小时**。

| 方式 | 成本 | 产出 |
|------|------|------|
| 人工开发者 | ~$100/小时 | 8 小时/天 |
| 单个 Agent | ~$10/小时 | 24 小时/天 |
| 5 个并行 Agent | ~$50/小时 | 120 Agent 小时/天 |

约束不是成本，而是**你能定义多少可靠的工作**。掌握验证优先自主循环的团队，发布速度会与没掌握的团队拉开根本性差距。

---

## 四种常用组合模式

### 模式一：规划 + L-Thread
1. C-Thread 规划（你验证计划）
2. 清空 context
3. L-Thread 实现（Ralph 循环）
4. 最终审查

**为什么有效**：规划和实现在独立 context 中。规划需要你的注意力，实现自主运行。

### 模式二：P-Thread 功能冲刺
1. 为多个功能写规格
2. 启动 P-Thread（每个功能一个）
3. 每个 P-Thread 内部作为 L-Thread 运行
4. 功能完成后逐一审查

**为什么有效**：功能层面并行，实现层面自主。

### 模式三：F-Thread 架构决策
1. 定义架构问题
2. 启动 F-Thread（3~4 个 Agent）
3. 每个 Agent 提出一种方案
4. 对比结果，选最好的
5. 用 L-Thread 实现所选方案

**为什么有效**：重要决策获得多角度视角，决策后自主实现。

### 模式四：B-Thread 编排
1. 主 Agent 接收大任务
2. 分解成子任务
3. 派生 Worker Agent（每个跑迷你 L-Thread）
4. 聚合结果
5. 主 Agent 验证并提交

**为什么有效**：分工明确，每个 Worker 专注，主 Agent 协调。

---

## 失败模式与修复

| 失败现象 | 原因 | 修复方法 |
|---------|------|---------|
| Thread 提前结束 | 验证太弱 | 加更多测试，让完成标准客观化，UI 加截图验证 |
| L-Thread 死循环 | 任务不可能完成或缺少完成承诺 | 设最大迭代次数，加明确完成标准 |
| P-Thread 产生冲突 | Agent 修改了同一文件 | 按功能/文件隔离，用 git worktree，明确并行工作边界 |
| B-Thread 失去连贯性 | 子 Agent 偏离主目标 | 更好的规格，更频繁的检查点，编排者验证子 Agent 工作 |
| 验证通过但结果错误 | 测试没覆盖实际需求 | 更好的验收标准，UI 截图验证，前几次手动审查 |

---

## 八周实施路径

| 周次 | 目标 |
|------|------|
| 第 1 周 | 跑可靠的 Base Thread，每次结果人工验证 |
| 第 2 周 | 加 P-Thread，同时跑两个 Agent，处理 context 切换 |
| 第 3 周 | 实现测试驱动验证，先写测试再实现 |
| 第 4 周 | 尝试第一个 L-Thread，用 Stop Hook，设最大迭代数，观察运行 |
| 第 5 周 | 扩展 L-Thread，隔夜运行，信任验证机制 |
| 第 6 周 | 加 B-Thread，让 Agent 派生子 Agent，编排多文件变更 |
| 第 7 周 | 尝试 F-Thread，为架构决策获取多方意见 |
| 第 8 周 | 组合模式：L-Thread 的 P-Thread，含 F-Thread 的 B-Thread |

每周衡量：跑了几个 Thread？运行了多长时间？需要几次检查点？

---

## 演进方向

- **更多 Thread**：每个层级都并行
- **更长 Thread**：以小时和天计，而非分钟
- **更粗 Thread**：Agent 派生 Agent 派生 Agent
- **更少检查点**：验证替代审查

掌握这套体系的开发者不只是"在用 AI"，而是在运营自主软件工厂。

**循环提供机制，Thread 提供扩展框架，验证提供可靠性。三者合一，系统在你睡觉时发布功能。**

---

> 原文来源：https://claudefa.st/blog/guide/mechanics/autonomous-agent-loops

## Related

- [[Multi-Agent 协作机制（基于 v2.1.88 源码）]]
- [[Ralph Wiggum 技术：让 Agent 在你睡觉时发布功能]]
- [[`/powerup`：Claude Code 内置交互式教程机制解析]]
