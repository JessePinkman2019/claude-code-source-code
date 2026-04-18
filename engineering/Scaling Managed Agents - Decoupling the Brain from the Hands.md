---
title: "Scaling Managed Agents: Decoupling the Brain from the Hands"
aliases:
  - "managed-agents"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Building Effective AI Agents"
  - "Effective Harnesses for Long-Running Agents"
  - "Auto Mode：用 AI 分类器替代手动权限审批"
---

# Scaling Managed Agents: Decoupling the Brain from the Hands

**来源**: https://www.anthropic.com/engineering/managed-agents  
**作者**: Lance Martin, Gabe Cemaj, Michael Cohen（Anthropic）  
**性质**: 工程架构——Managed Agents 托管服务的设计原理

---

## 一句话概括

通过将 agent 的"大脑"（Claude + harness）与"双手"（sandbox/工具）和"记忆"（session log）彻底解耦，
Managed Agents 构建了一套能随模型能力演进而持续稳定的托管基础设施——借鉴操作系统对硬件虚拟化的思路，用稳定的抽象接口隔离不断变化的底层实现。

---

## 背景：Harness 假设会过时

此前系列文章（building-effective-agents、effective-harnesses、harness-design-long-running-apps）的共同线索：
**harness 编码了"Claude 自己做不到"的假设**。

但这些假设会随模型改进迅速过时。典型案例：

- Sonnet 4.5 有"context anxiety"（上下文将满时提前收尾）
- 为此在 harness 中加入了 context reset 机制
- 换成 Opus 4.5 后，该行为消失——context reset 成了死重

结论：**与其不断修补特定 harness，不如构建一套接口稳定、实现可替换的元基础设施。**

---

## 核心类比：操作系统对硬件的虚拟化

操作系统通过将硬件抽象为 _进程_、_文件_ 等接口，实现了"为尚未被想到的程序设计"的目标。

- `read()` 命令对 1970 年代的磁盘包和现代 SSD 都适用
- 上层抽象稳定，底层实现可以自由替换

**Managed Agents 遵循同样的模式**，将 agent 的三个组件虚拟化为接口：

| 组件 | 抽象 | 含义 |
|------|------|------|
| **Session** | append-only 事件日志 | 所有发生过的事情的完整记录 |
| **Harness** | 调用 Claude 并路由工具调用的循环 | 大脑 |
| **Sandbox** | Claude 运行代码、编辑文件的执行环境 | 双手 |

三者各自有独立接口，可以单独失败、单独替换，互不干扰。

---

## 问题一：不要养宠物（Don't Adopt a Pet）

### 最初的耦合设计

初始方案将 session、harness、sandbox 全部放在一个容器里。

**问题**：容器变成了"宠物"（pet）——必须精心照料、不能丢失的单点。

- 容器失败 → session 丢失
- 容器无响应 → 工程师要手动进去 debug
- 容器里有用户数据 → debug 本质上违反了安全边界
- harness 假设所有资源都在容器旁边 → 客户要接入自己 VPC 时必须做网络对等（network peering）

**宠物 vs 牛群类比**：
- 宠物：有名字、手工照料、不能失去的个体
- 牛群：可互换的个体，某一头死了换一头

### 解决方案：三个组件全部变成牛群

**大脑离开容器**

harness 不再住在容器里，而是像调用其他工具一样调用容器：

```
execute(name, input) → string
```

容器变成牛群：容器死了 → harness 捕获到工具调用错误 → 传给 Claude → Claude 决定是否重试 → 用标准配方重新初始化：`provision({resources})`

**Harness 失败也可恢复**

session log 存在 harness 外部，所以 harness 本身不需要存活任何状态。

- harness 崩溃 → 用 `wake(sessionId)` 重启新 harness
- 用 `getSession(id)` 取回事件日志
- 从最后一个事件恢复执行

harness 运行期间，用 `emitEvent(id, event)` 持续写入 session，保证事件的持久记录。

**安全边界**

耦合设计的安全问题：Claude 生成的代码和凭据（credentials）在同一容器里。一旦 prompt injection 成功，攻击者可以读取环境变量拿到 token，然后用 token 创建新 session 并委托工作。

解耦后的两种凭据处理模式：
1. **Git**：用 access token 在 sandbox 初始化时 clone 仓库并写入 git remote，之后 push/pull 直接从 sandbox 工作，agent 永远不接触 token 本身
2. **自定义工具（MCP）**：OAuth token 存在 harness 外部的安全 vault 里，Claude 通过专用 proxy 调用 MCP 工具，proxy 用 session 关联的 token 去 vault 取对应凭据再调用外部服务，harness 全程不知道任何凭据

---

## 问题二：Session ≠ Claude 的 Context Window

### 传统 context 管理的局限

长时任务超出 context window 后，常见方案都涉及**不可逆的决策**：
- Compaction：摘要早期对话，但被压缩的消息难以恢复
- Context trimming：选择性删除旧工具结果、thinking blocks
- 难以预判未来 turn 需要哪些 token

### Session 作为 Context 对象

Managed Agents 的 session log 提供了类似"REPL 中的 context 对象"的能力——context 存在于 context window **外部**，可以持久保存并按需查询。

核心接口：`getEvents()` — 通过位置切片查询事件流

使用方式：
- **向前恢复**：从上次读到的位置继续
- **回滚查看**：在某个时刻前倒回几个事件，了解前因
- **重读上下文**：在某个动作前重新读取相关上下文

拉取的事件可以在 harness 中做任意变换后再传入 Claude 的 context window，包括：
- 提高 prompt cache 命中率的 context 组织
- 任意 context engineering

**设计哲学**：可恢复的 context 存储（session）和任意 context 管理（harness）分离，因为无法预测未来模型需要什么样的 context engineering 策略。

---

## 多脑多手：扩展能力

### 多脑（Many Brains）

**解耦前的问题**：
- 每个大脑需要一个容器 → N 个大脑需要 N 个容器
- 每次 session 都要先等容器启动才能开始推理
- TTFT（time-to-first-token）包含了全量容器启动成本

**解耦后**：
- 容器按需通过工具调用 `execute(name, input)` 触发
- 不需要 sandbox 的 session 不等容器
- 推理在 orchestration layer 拉取 session 事件后立即开始
- 多大脑 = 启动多个无状态 harness，按需连接到 hands

**实测性能提升**：
- p50 TTFT 下降约 **60%**
- p95 TTFT 下降超过 **90%**

同时解决了 VPC 连接问题：harness 不在容器里，不再假设所有资源都在附近，客户无需网络对等。

### 多手（Many Hands）

早期模型不具备在多个执行环境间推理的能力，因此限制在单容器。随着智能提升，单容器反而成了瓶颈：那个容器失败，大脑伸向的所有手都丢失了状态。

解耦后，每只手都是一个工具：

```
execute(name, input) → string
```

- 支持任何自定义工具、任何 MCP server、Anthropic 自己的工具
- harness 不知道 sandbox 是容器、手机还是 Pokémon 模拟器
- 大脑之间可以互相传递手（hands）

---

## 核心设计原则总结

### Meta-Harness 的立场

Managed Agents 是一个 **meta-harness**：

- **有立场的地方**：Claude 需要什么样的接口（session、sandbox）
- **没立场的地方**：具体用什么 harness 实现

它兼容：
- Claude Code（通用 coding harness）
- 任务特定的专用 harness
- 任何未来出现的新 harness

### 两个设计假设（仅这两个）

1. Claude 需要操作状态的能力 → **session**
2. Claude 需要执行计算的能力 → **sandbox**

此外，不对大脑/手的数量和位置做任何假设。

---

## 与系列文章的关系

```
building-effective-agents（agent 设计原则）
        ↓
effective-harnesses-for-long-running-agents（跨 context window 的基础设施）
        ↓
harness-design-long-running-apps（生成器-评估器多 agent 架构）
        ↓
managed-agents（本文）：为所有这些 harness 提供稳定底座的元基础设施
```

---

## 个人理解

### 最深刻的架构洞察

**"Harness 假设会过时"** 这个洞察是整篇文章的核心驱动力。context reset 的例子说明：为某个模型版本的弱点设计的机制，可能在下一个版本成为累赘。

这提示了一个更普遍的原则：**应对 AI 能力提升的正确姿势不是不断打补丁，而是设计接口足够稳定、实现足够可替换的基础设施。**

### Session ≠ Context Window 的意义

把 session 从 context window 中解放出来，本质上是把"记忆"从"工作记忆"升级为"长期记忆"。这和人类认知架构的区别类似：工作记忆有限且易失，长期记忆持久且可检索。

### 安全设计的根本原则

凭据永远不应该出现在 Claude 生成的代码所能访问的环境里。这不是"缩小权限范围"的优化，而是结构性隔离——无论 Claude 多聪明，它都拿不到 token，因为 token 根本就不在它的沙箱里。

## Related

- [[Building Effective AI Agents]]
- [[Effective Harnesses for Long-Running Agents]]
- [[Auto Mode：用 AI 分类器替代手动权限审批]]
