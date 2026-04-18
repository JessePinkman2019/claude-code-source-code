---
title: "Building a C Compiler with a Team of Parallel Claudes"
aliases:
  - "building-c-compiler"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Building Effective AI Agents"
  - "Scaling Managed Agents: Decoupling the Brain from the Hands"
---

# Building a C Compiler with a Team of Parallel Claudes

**来源**: https://www.anthropic.com/engineering/building-c-compiler  
**作者**: Nicholas Carlini（Anthropic Safeguards 团队研究员）  
**源码**: https://github.com/anthropics/claudes-c-compiler  
**性质**: 工程实践——多智能体并行团队的能力极限测试

---

## 一句话概括

16 个 Claude 实例并行工作，经过近 2,000 个 Claude Code 会话、$20,000 API 费用、约两周时间，从零写出了一个 10 万行的 Rust 实现 C 编译器——能编译 Linux 6.9 内核（x86/ARM/RISC-V），可运行 Doom。

---

## 实验背景与目标

**"Agent Teams"**：多个 Claude 实例并行在共享代码库上工作，无需人类持续干预。

作者的一贯方法：压力测试 LLM 在当前**勉强能做到**的事情边界，以便为未来做准备。

**具体任务**：用 Rust 从零实现一个 C 编译器：
- 无依赖（除 Rust 标准库）
- 兼容 GCC 接口
- 能编译 Linux 内核
- 支持多后端（x86、ARM、RISC-V）
- 包含 SSA IR（支持多个优化 pass）

---

## 基础设施：让 Claude 持续运行

### 单 Agent 无限循环

```bash
#!/bin/bash

while true; do
    COMMIT=$(git rev-parse --short=6 HEAD)
    LOGFILE="agent_logs/agent_${COMMIT}.log"

    claude --dangerously-skip-permissions \
           -p "$(cat AGENT_PROMPT.md)" \
           --model claude-opus-X-Y &> "$LOGFILE"
done
```

**注意**：在容器中运行，不要在本机直接运行。
（曾发生过一次意外：Claude 执行了 `pkill -9 bash`，杀死了自己的循环进程。）

### 并行 Agent 架构

- 创建一个裸 git repo
- 每个 agent 运行在独立 Docker 容器中，repo 挂载到 `/upstream`
- 每个 agent clone 本地副本到 `/workspace`
- 完成后 push 到 upstream

### 任务同步机制

1. Agent 通过写入 `current_tasks/` 中的文本文件"锁定"任务（如 `current_tasks/parse_if_statement.txt`）
2. 如果两个 agent 尝试认领同一任务，git 同步强制第二个 agent 选择不同任务
3. Agent 完成后：pull upstream → 合并其他 agent 的变更 → push 自己的变更 → 移除锁
4. 无限循环产生新的 Claude Code 会话，周期重复

Merge conflict 很频繁，但 Claude 足够智能可以处理。**没有 orchestration agent**，每个 agent 自行决定下一步做什么。

---

## 核心设计经验

### 1. 编写极高质量的测试

Claude 会自主解决测试所定义的问题——因此**测试验证器必须近乎完美**，否则 Claude 会解决错误的问题。

作者的工作：
- 引入高质量编译器测试套件
- 为开源软件包编写 verifier 和构建脚本
- 观察 Claude 犯的错误，针对性设计新测试
- 构建 CI 流水线，强制新 commit 不能破坏现有代码

### 2. 站在 Claude 的角度思考

测试 harness 是为 Claude 写的，不是为人类写的——需要重新审视关于"测试应该如何传达结果"的假设。

**Context 污染问题**：
- 测试 harness 不应输出数千字节无用信息
- 最多输出几行，所有重要信息写入日志文件
- 日志文件要易于自动处理：有错误就写 `ERROR`，原因放在同一行方便 grep
- 预计算聚合统计信息，省去 Claude 重复计算

**时间盲区问题**：
- Claude 无法感知时间，会愉快地花数小时跑测试而非推进进展
- harness 设计：进度信息少量输出（避免污染 context）
- 默认 `--fast` 选项只跑 1% 或 10% 随机样本
- 样本对每个 agent 确定性（可复现回归），但跨 VM 随机（整体覆盖所有文件）

### 3. 让并行变得容易

**容易并行化的阶段**：有大量独立的失败测试时，每个 agent 认领不同的测试修复。

**遇到瓶颈**：测试通过率达到 99% 后，agent 开始编译 Linux 内核——这是一个巨大的单体任务。每个 agent 都会碰到同一个 bug，修复后互相覆盖更改，16 个 agent 毫无效率。

**解决方案**：用 GCC 作为在线已知良好的编译器 oracle：
- 随机选大部分内核文件用 GCC 编译
- 只有剩余文件用 Claude 的编译器
- 如果内核可以工作：问题不在 Claude 的那部分文件
- 如果内核崩溃：缩小范围，逐步排查
- 这让每个 agent 可以并行修复不同文件中的不同 bug
- 后续还需应用 delta debugging 技术找出共同失败的文件对

### 4. 多 agent 角色分工

并行还带来了专业化的可能：

| Agent 角色 | 职责 |
|-----------|------|
| 主要编码 agents | 实现编译器功能、修复 bug |
| 代码去重 agent | 合并重复实现的功能 |
| 性能优化 agent | 提升编译器自身性能 |
| 代码输出优化 agent | 让编译产物更高效 |
| Rust 设计审查 agent | 从 Rust 开发者视角批评设计，做结构改进 |
| 文档 agent | 维护文档 |

---

## 最终成果

### 规模与成本

| 指标 | 数据 |
|------|------|
| Claude Code 会话数 | 近 2,000 次 |
| 运行时间 | 约两周 |
| 输入 tokens | 20 亿 |
| 输出 tokens | 1.4 亿 |
| 总费用 | 约 $20,000 |
| 代码行数 | 约 10 万行 Rust |

### 能力

- 构建可启动的 Linux 6.9（x86、ARM、RISC-V）
- 编译 QEMU、FFmpeg、SQLite、PostgreSQL、Redis
- GCC torture test suite 通过率：**99%**
- 能编译并运行 Doom

### 当前局限

| 局限 | 说明 |
|------|------|
| 缺少 16-bit x86 编译器 | 无法独立完成 Linux real mode 启动，调用 GCC 代劳 |
| 无自己的汇编器和链接器 | 仍处于早期，demo 使用 GCC 的汇编器和链接器 |
| 无法编译所有项目 | 不是 GCC 的完全替代品 |
| 生成代码效率低 | 开所有优化后，仍不如 GCC 关闭所有优化 |
| Rust 代码质量一般 | 远未达到 Rust 专家水准 |

**一个典型的卡住案例**：16-bit x86 代码生成器。虽然可以通过 66/67 前缀输出正确的 16-bit x86，但生成的代码超过 60KB，远超 Linux 要求的 32KB 上限。最终 Claude 选择"作弊"——对这个 phase 调用 GCC（ARM 和 RISC-V 可以完全独立编译）。

---

## 模型演进比较

| 模型 | 能力 |
|------|------|
| 早期 Opus 4 系列 | 勉强能产出有功能的编译器 |
| Opus 4.5 | 第一个能通过大型测试套件，但无法编译真实大项目 |
| Opus 4.6 | 能编译 Linux 内核，通过 GCC torture test，可运行 Doom |

---

## 作者的思考与担忧

**令人振奋的**：这在 2026 年初能够实现，远超预期。Agent teams 展示了自主实现整个复杂项目的可能性。

**令人不安的**：
- 当人类坐在 Claude 旁边开发时，可以实时确保质量、捕获错误
- 对于自主系统，看到测试通过就认为工作完成，但实际几乎从不是这样
- 作者曾做渗透测试工作，**程序员部署从未亲自验证过的软件是真实的安全隐患**

> "Building this compiler has been some of the most fun I've had recently, but I did not expect this to be anywhere near possible so early in 2026."

---

## 个人理解

### 这个项目的真正意义

10 万行编译器本身不是重点——这是 Anthropic 研究员用来**压力测试多智能体协作上限**的基准。作者明确说这是能力测试，目的是"研究 LLM 在今天勉强能做到什么边界"。

### 最值得学习的工程原则

**"站在 Claude 的角度思考"**是最有实操价值的建议：
- 测试输出格式要对 Claude 友好（不是对人类友好）
- ERROR 写在同一行方便 grep
- 避免 context 污染
- 处理时间盲区（默认 fast 模式）

这些原则在任何长时运行的 agent 任务中都适用，不限于编译器项目。

### GCC Oracle 技巧的通用性

用已知正确的实现作为 oracle 来分解"大单体任务"成"可并行的子任务"，这是一个非常通用的设计模式。当任务无法自然分解时，通过 oracle 对比来划定范围，让并行重新成为可能。

### 关于自主开发的安全隐患

作者的担忧值得认真对待：当 agent 团队能写出"测试通过"的大型项目，而人类没有亲自审查代码时，潜在的安全漏洞和质量问题会变得更难发现。能力的快速提升带来的不只是生产力收益，还有新的监督挑战。

## Related

- [[Building Effective AI Agents]]
- [[Scaling Managed Agents: Decoupling the Brain from the Hands]]
