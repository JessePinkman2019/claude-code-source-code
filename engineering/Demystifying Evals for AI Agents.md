---
title: "Demystifying Evals for AI Agents"
aliases:
  - "demystifying-evals-for-ai-agents"
area: engineering
tags: []
status: evergreen
source: ""
source_type: engineering-article
related:
  - "Designing AI-Resistant Technical Evaluations"
  - "Eval Awareness in Claude Opus 4.6's BrowseComp Performance"
---

# Demystifying Evals for AI Agents

**来源**: https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents  
**作者**: Mikaela Grace、Jeremy Hadfield、Rodrigo Olivares、Jiri De Jonghe  
**性质**: 工程最佳实践——AI agent 评测的系统性指南

---

## 一句话概括

好的 eval 让团队以更大信心交付 AI agent：没有 eval 的团队陷入被动循环（生产环境发现问题 → 修复 → 引入新问题），有 eval 的团队则能把每次失败转化为测试用例，把猜测换成可量化的指标——这篇文章是 Anthropic 基于内部实践的完整方法论。

---

## 核心概念词汇表

| 术语 | 定义 |
|------|------|
| **Task（任务）** | 单个测试，包含定义好的输入和成功标准 |
| **Trial（试验）** | 对同一 task 的一次执行。由于模型输出有随机性，需多次 trial 才能得到稳定结果 |
| **Grader（评分器）** | 对 agent 某方面表现进行评分的逻辑。一个 task 可有多个 grader |
| **Transcript / Trace / Trajectory** | 一次 trial 的完整记录：输出、工具调用、推理、中间结果 |
| **Outcome（结果）** | trial 结束时环境的最终状态。Agent 说"已订机票"是 transcript，数据库里真的有预订记录才是 outcome |
| **Evaluation harness** | 端到端运行 eval 的基础设施：提供指令和工具、并发运行任务、记录步骤、评分、聚合结果 |
| **Agent harness / scaffold** | 让模型作为 agent 运行的系统（如 Claude Code）。Eval 评测的是 harness + 模型的组合 |
| **Evaluation suite** | 一组用于衡量特定能力或行为的 task 集合 |

---

## 为什么要构建 Eval

### 没有 eval 的团队的困境

- "Agent 感觉变差了"——无法验证，只能猜测
- 调试是被动的：等待报告 → 手动复现 → 修复 bug → 祈祷没有回归
- 无法区分真正的回归和噪声
- 发布新模型时需要数周人工测试

### Eval 的价值复利

- **早期**：强制团队明确定义"成功"是什么（两个工程师读同一规格可能得出不同的边缘情况处理方式，eval suite 消除这种歧义）
- **中期**：基准线和回归测试自动得到：延迟、token 用量、每任务成本、错误率
- **发布新模型时**：有 eval 的团队几天内就能确定模型的强弱点并升级；没有 eval 的团队需要数周测试

**Claude Code 的真实案例**：从早期基于 Anthropic 员工反馈的快速迭代，到后来加入针对性 eval（先是简洁性和文件编辑，再到过度工程化等复杂行为），这些 eval 帮助识别问题、引导改进、聚焦研究与产品协作。

---

## 三类评分器（Grader）

### 代码评分器

| 方法 | 优势 | 劣势 |
|------|------|------|
| 字符串匹配（精确/正则/模糊）、二元测试（fail-to-pass）、静态分析、状态验证、工具调用验证、transcript 分析 | 快、便宜、客观、可复现、易调试、验证具体条件 | 对不完全匹配预期模式的有效变体很脆；缺乏细微差别；不适合更主观的任务 |

### 模型评分器（LLM-as-judge）

| 方法 | 优势 | 劣势 |
|------|------|------|
| 基于 rubric 评分、自然语言断言、成对比较、基于参考的评估、多评判共识 | 灵活、可扩展、捕捉细微差别、处理开放性任务和自由格式输出 | 非确定性；比代码更贵；需要与人类评分者校准 |

### 人工评分器

| 方法 | 优势 | 劣势 |
|------|------|------|
| 专家审查、众包判断、抽样检查、A/B 测试、评分者间一致性 | 黄金标准质量；匹配专家用户判断；用于校准模型评分 | 贵、慢；需要规模化访问人类专家 |

**评分策略**：加权（组合分数需达到阈值）、二元（所有 grader 必须通过）、混合。

---

## 能力 Eval vs 回归 Eval

| 类型 | 问题 | 目标通过率 | 用途 |
|------|------|-----------|------|
| **能力/质量 eval** | "这个 agent 能做好什么？" | 低（从 agent 挣扎的任务开始）| 给团队一座"山"去爬 |
| **回归 eval** | "Agent 还能处理以前能处理的任务吗？" | 近 100% | 防止退步 |

**生命周期转换**：能力 eval 的通过率上升后，可以"升级"为持续运行的回归 suite——从"我们能做到吗"变成"我们还能可靠地做到吗"。

---

## 各类 Agent 的评测方法

### 编程 Agent

**特点**：软件通常直接可评测——代码是否运行？测试是否通过？

代表性基准：
- **SWE-bench Verified**：给出 GitHub issue，通过运行测试套件评分（一年内从 40% 提升到 >80%）
- **Terminal-Bench**：端到端技术任务（如从源代码构建 Linux 内核）

**编程 agent eval 示例（YAML）**：

```yaml
task:
  id: "fix-auth-bypass_1"
  desc: "Fix authentication bypass when password field is empty..."
  graders:
    - type: deterministic_tests
      required: [test_empty_pw_rejected.py, test_null_pw_rejected.py]
    - type: llm_rubric
      rubric: prompts/code_quality.md
    - type: static_analysis
      commands: [ruff, mypy, bandit]
    - type: state_check
      expect:
        security_logs: {event_type: "auth_blocked"}
    - type: tool_calls
      required:
        - {tool: read_file, params: {path: "src/auth/*"}}
        - {tool: edit_file}
        - {tool: run_tests}
  tracked_metrics:
    - type: transcript
      metrics: [n_turns, n_toolcalls, n_total_tokens]
    - type: latency
      metrics: [time_to_first_token, output_tokens_per_sec, time_to_last_token]
```

实践中编程 eval 通常只用单元测试（正确性）+ LLM rubric（代码质量），其余按需添加。

### 对话 Agent

**特点**：交互质量本身是评测对象；通常需要第二个 LLM 模拟用户。

成功是多维的：工单是否解决（状态检查）+ 是否在 10 轮内完成（transcript 约束）+ 语气是否恰当（LLM rubric）。

代表性基准：τ-Bench、τ2-Bench（模拟多轮交互，涵盖零售支持和航空预订等场景）。

### 研究 Agent

**挑战**：没有单元测试等价的二元信号；专家可能对综合是否"全面"持不同意见；参考内容不断变化；输出越长越容易出错。

**策略**：组合多类 grader——
- **Groundedness check**：声明是否有来源支撑
- **Coverage check**：好的答案必须包含哪些关键事实
- **Source quality check**：来源是否权威，而非仅仅是最先检索到的

### 计算机使用 Agent

需要在真实或沙盒环境中运行并检查 outcome（而非仅检查 agent 说了什么）。

代表性基准：WebArena（浏览器任务）、OSWorld（完整操作系统控制）。

**token 效率权衡**：
- DOM 交互：执行快但消耗 token 多
- 截图交互：较慢但更 token 高效
- 例：总结 Wikipedia → DOM 更高效；Amazon 购物 → 截图更高效（提取整个 DOM 消耗 token 过多）

---

## 处理非确定性的两个指标

**pass@k**：k 次尝试中至少一次正确的概率。k 增大时 pass@k 上升。
- 适用场景：工具类，一次成功即可（如代码生成）

**pass^k**：k 次尝试全部正确的概率。k 增大时 pass^k 下降。（75% 单次成功率，运行 3 次全部通过的概率 = 0.75³ ≈ 42%）
- 适用场景：面向用户的 agent，用户期望每次都可靠

k=1 时两者相同；k=10 时讲述完全相反的故事：pass@k 趋近 100%，pass^k 趋近 0%。

---

## 从零到一的实用路线图

### 收集初始 eval 数据集

**Step 0：尽早开始**
- 不需要等到有几百个 task——20-50 个来自真实失败的简单 task 就是很好的起点
- Eval 越晚构建越难：早期产品需求自然转化为测试用例；等太久就要从现有系统逆向工程成功标准

**Step 1：从已手动测试的内容开始**
- 把开发中每次发布前验证的行为转化为 task
- 如果已在生产：看 bug tracker 和支持队列，把用户报告的失败转化为测试用例

**Step 2：编写无歧义的 task 和参考解法**
- 好的 task：两个领域专家能独立得出相同的通过/失败判断
- 0% pass@100 通常是 **task 有问题**的信号，而非模型能力不足——需要检查任务规格和 grader

**Step 3：构建均衡的问题集**
- 同时测试"行为应该发生"和"行为不应发生"的情况
- 单侧 eval 产生单侧优化——Claude.ai 网络搜索 eval 的教训：既要测"应该搜索时搜索"，也要测"不应该搜索时不搜索"

### 设计 eval harness 和 grader

**Step 4：构建稳健的 eval harness**
- 每次 trial 必须从干净环境开始（隔离）
- 曾观察到 Claude 通过检查之前 trial 的 git 历史获得不公平优势——shared state 会人为抬高成绩

**Step 5：谨慎设计 grader**

关键原则：
- **评 agent 产出什么，而非走了哪条路**——检查特定工具调用顺序太死板；agent 经常找到 eval 设计者没预料到的有效方法
- **多组件任务引入部分分**——正确识别问题但未能处理退款的 support agent，明显好于立即失败的 agent
- **给 LLM judge 留退路**——提供"Unknown"选项避免幻觉；为每个维度单独使用独立 LLM judge 而非一个 judge 评所有维度

**典型 grading bug 案例**：
- Opus 4.5 在 CORE-Bench 初始得 42%，发现多个问题后（严格 grading 惩罚"96.12"而期望"96.124991…"、歧义任务规格、无法复现的随机任务）得分跳到 **95%**
- METR 发现时间跨度基准中的任务要求 agent 优化到某个分数阈值，但 grading 要求**超过**该阈值——Claude 因遵循指令而被惩罚，忽略目标的模型反而得分更高

### 长期维护

**Step 6：阅读 transcript**
- 失败时 transcript 告诉你是 agent 真的犯了错，还是 grader 拒绝了有效解法
- 这是验证 eval 衡量的是真正重要的东西的关键技能

**Step 7：监控能力 eval 饱和**
- 100% 通过率的 eval 只能跟踪回归，不提供改进信号
- SWE-Bench Verified 从 30% 涨到接近 80%——接近饱和时，大能力提升对应小分数增长，结果具有误导性
- Qodo 的教训：对 Opus 4.5 的一次评测因使用一次性编码 eval 而错过了在更长更复杂任务上的提升

**Step 8：保持 eval suite 健康**
- Eval suite 是需要持续关注和清晰所有权的活文档
- 有效的做法：专职 eval 团队拥有核心基础设施，领域专家和产品团队贡献大部分 eval task
- **Eval 驱动开发**：在 agent 能完成之前先写 eval 定义计划能力，再迭代直到 agent 通过

---

## Eval 与其他方法的配合（瑞士奶酪模型）

| 方法 | 优势 | 劣势 |
|------|------|------|
| **自动化 eval** | 快速迭代、完全可复现、无用户影响、可在每次 commit 运行 | 需要前期投入构建；可能产生虚假信心（不匹配真实使用模式） |
| **生产监控** | 揭示真实用户行为；捕捉合成 eval 遗漏的问题 | 被动；问题先到达用户；信号嘈杂 |
| **A/B 测试** | 衡量真实用户结果；控制混淆变量 | 慢（需数天/周）；只测试已部署的变更 |
| **用户反馈** | 发现未预料到的问题；真实案例 | 稀少且自选择；偏向严重问题 |
| **手动 transcript 审查** | 建立对失败模式的直觉；捕捉细微质量问题 | 耗时；不可扩展 |
| **系统性人工研究** | 黄金标准质量判断 | 贵、慢；难以频繁运行 |

没有单一评估层能捕捉所有问题——就像安全工程中的瑞士奶酪模型，多层方法互补。

---

## Eval 工具框架（附录）

- **Harbor**：容器化环境运行 agent，支持跨云提供商大规模运行 trial，Terminal-Bench 2.0 通过其注册表发布
- **Braintrust**：离线评估 + 生产可观测性 + 实验追踪，含预构建的事实性/相关性评分器
- **LangSmith**：追踪、离线/在线评估、数据集管理，与 LangChain 生态深度集成
- **Langfuse**：类似 LangSmith 的自托管开源版本
- **Arize Phoenix**：开源 LLM 追踪、调试和评估平台

> 框架只是加速器——它们的质量取决于你通过它们运行的 eval task。最好快速选定一个框架，然后把精力投入到 eval 本身：迭代高质量测试用例和 grader。

---

## 个人理解

### 最重要的反直觉原则

**"评 outcome，不评路径"**——检查特定工具调用顺序是"过度规定的测试"陷阱。Opus 4.5 解决 τ2-bench 航班预订问题时发现了政策漏洞，在 eval 层面"失败"但实际给用户提供了更好的解法。这说明好的 eval 应该验证用户真正关心的结果，而非实现细节。

### 0% 通过率是 task 问题，不是模型问题

这个判断规则非常实用：如果一个 task 连 100 次尝试都没通过，几乎肯定是 task 规格或 grader 有 bug，而不是模型真的完全不能做。CORE-Bench 42% → 95% 的案例是最好的示范。

### Eval 的隐藏成本

Eval 的成本是前期可见的，收益是后期积累的——这是为什么团队容易推迟构建 eval 的根本原因。但文章的数据很清楚：新模型发布时，有 eval 的团队几天升级，没有的要几周。这个收益差异随时间指数级扩大。

## Related

- [[Designing AI-Resistant Technical Evaluations]]
- [[Eval Awareness in Claude Opus 4.6's BrowseComp Performance]]
