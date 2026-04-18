---
title: "Long-running Claude for Scientific Computing"
aliases:
  - "long-running-claude-scientific-computing"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/research/long-running-Claude"
source_type: research-note
authors: "Siddharth Mishra-Sharma（Anthropic Discovery Team）"
year: null
related:
  - "Measuring AI Agent Autonomy in Practice"
  - "Trustworthy Agents in Practice"
  - "Introducing the Anthropic Science Blog"
---

# Long-running Claude for Scientific Computing

**来源**: https://www.anthropic.com/research/long-running-Claude  
**作者**: Siddharth Mishra-Sharma（Anthropic Discovery Team）  
**性质**: 实践教程——多日 agentic coding 工作流

---

## 一句话概括

用 Claude Opus 4.6 在 HPC 集群上跑多天的自主科研 agent，通过 CLAUDE.md（计划）+ CHANGELOG.md（记忆）+ 测试 oracle + Ralph loop（防懒惰）四件套，让 agent 在非专家监督下完成了一个宇宙学 Boltzmann 求解器。

---

## 背景：工作模式的转变

**旧模式**：人在循环，管控每一步  
**新模式**：指定高层目标，放开一组 agent 自主完成

适合 agent 自主工作的任务特征：
- 工作范围明确（well-scoped）
- 成功标准可量化（clear success criteria）
- 人类监督可以偶发而非持续

**案例**：在 HPC 集群上用 Claude Opus 4.6 实现了一个可微分的宇宙学 Boltzmann 求解器（JAX 实现），达到与参考实现 CLASS 的亚百分比精度。这类工作通常需要领域专家花费数月到数年——作者本人并非该领域专家。

---

## 四件套架构

### 1. CLAUDE.md——计划与规则

- 放在项目根目录，Claude 会自动将其保持在 context 中
- 包含：项目目标、设计决策、成功标准、操作规则
- **关键**：Claude 可以边工作边编辑这个文件，将新的决策和发现写回去

示例规则（写在 CLAUDE.md 中）：
```
每完成一个有意义的工作单元就 commit 并 push。
每次 commit 前运行 pytest tests/ -x -q。
不要 commit 任何会破坏现有通过测试的代码。
```

### 2. CHANGELOG.md——跨 session 的持久记忆

充当 agent 的"实验记录本"，记录：
- 当前状态
- 已完成任务
- **失败的尝试及原因**（这点最关键——没有它，下一个 session 会重走同样的死路）
- 关键检查点的精度表
- 已知局限

示例条目：
> "尝试用 Tsit5 求解扰动 ODE，系统太刚性（stiff）。切换到 Kvaerno5。"

### 3. 测试 Oracle——agent 知道自己是否在进步

长时间自主工作的核心依赖：agent 需要一个可量化的方式判断是否在进步。可以是：
- 参考实现（本例用 CLASS C 源码）
- 可量化目标函数
- 现有测试套件

同时让 agent 持续扩展测试套件，防止回归。

**注意**：本例中 agent 有一个明显的测试覆盖盲区——一度只在单个参数点测试，大幅降低了 bug 捕获率。这是当前 agent 的典型缺陷。

### 4. Ralph Loop——防止 agentic 懒惰

**问题**：当前模型会出现"agentic laziness"——完成复杂任务的途中找借口停下（甚至说"今天比较晚了，明天再继续？"）。

**解法**：Ralph loop，本质上是一个 for 循环，在 agent 宣称完成时把它踢回 context，问"你真的完成了吗？"

```
/ralph-loop:ralph-loop "请持续工作直到在全参数范围达到 0.1% 精度。" --max-iterations 20 --completion-promise "DONE"
```

agent 会迭代最多 20 次，直到确认完成并输出 "DONE"。

类似方案：GSD（Get Shit Done）、Claude Code 原生的 `/loop` 命令。

---

## 执行环境：HPC 集群 + tmux

在 SLURM 集群上的典型 job script：

```bash
#!/bin/bash
#SBATCH --job-name=claude-agent
#SBATCH --partition=GPU-shared
#SBATCH --gres=gpu:h100-32:1
#SBATCH --time=48:00:00
#SBATCH --output=agent_%j.log

cd $PROJECT/my-solver
source .venv/bin/activate
export TERM=xterm-256color
tmux new-session -d -s claude "claude; exec bash"
tmux wait-for claude
```

**工作流程**：
1. 提交 SLURM job → Claude Code 在 tmux session 中启动
2. attach 到 tmux，给 Claude 指令（如"读 CHANGELOG.md，继续下一个任务"）
3. detach，关电脑，偶尔在手机 GitHub 上查看进度
4. 需要时 re-attach：`srun --jobid=JOBID --overlap --pty tmux attach -t claude`

**Git 作为协调工具**：每个有意义的工作单元 commit + push，提供可恢复历史，进度可见，防止计算配额耗尽时丢失工作。

---

## 实际结果

- Claude 从零开始工作了几天，达到与 CLASS 参考实现的亚百分比精度
- 开发轨迹"有些笨拙"：测试覆盖不足，会被坐标规范（gauge conventions）绊倒，会花数小时追一个领域专家几秒就能看出的 bug
- 但始终在朝目标稳步推进
- **副产品**：作者通过观察 git commit history 学到了很多 Boltzmann 求解器的知识——"commit log 读起来像一个快速、极度认真的博士后的实验记录"

结论：agent 驱动的开发可以将数月甚至数年的研究者工作压缩到几天。

---

## 核心洞察

> 现在，**不跑 agent 也有机会成本**。如果你有计算资源和目标明确的项目，每一个没有让 agent 工作的夜晚，都是白白损失的潜在进展。

---

## 对 Claude Code 用户的启示

这套工作流并非科研专用，对任何长时间 coding agent 任务都适用：

| 元素 | 对应 Claude Code 用法 |
|------|---------------------|
| CLAUDE.md | 在项目根目录写清楚目标、规则、成功标准 |
| CHANGELOG.md | 让 agent 维护一个进度+失败记录文件 |
| 测试 oracle | 给 agent 明确的可量化验证标准（跑测试、对比输出） |
| Ralph loop | 使用 `/loop` 命令或类似机制，防止 agent 中途认为"完成了" |
| tmux + Git | 让 agent 在后台跑，通过 commit history 监控进度 |

**最关键的一条**：失败记录（failed approaches）必须持久化。没有它，每次新 session agent 都会重新踩同样的坑。

## Related

- [[Measuring AI Agent Autonomy in Practice]]
- [[Trustworthy Agents in Practice]]
- [[Introducing the Anthropic Science Blog]]
