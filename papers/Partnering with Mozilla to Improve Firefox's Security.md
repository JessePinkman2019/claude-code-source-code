---
title: "Partnering with Mozilla to Improve Firefox's Security"
aliases:
  - "mozilla-firefox-security"
area: papers
tags: []
status: evergreen
source: "https://www.anthropic.com/news/mozilla-firefox-security"
source_type: research-note
authors: ""
year: null
related:
  - "Trustworthy Agents in Practice"
---

# Partnering with Mozilla to Improve Firefox's Security

**来源**: https://www.anthropic.com/news/mozilla-firefox-security  
**相关**: https://blog.mozilla.org/en/firefox/hardening-firefox-anthropic-red-team/  
**时间**: 2026 年（Firefox 148.0 修复发布）  
**模型**: Claude Opus 4.6

---

## 一句话概括

Claude Opus 4.6 在两周内发现了 Firefox 的 22 个漏洞，其中 14 个被评为高危——占 2025 年全年 Firefox 高危漏洞修复总数的近五分之一。AI 已成为世界级漏洞研究员。

---

## 关键数据

| 指标 | 数量 |
|------|------|
| 发现的漏洞总数 | 22 个（提交 112 份报告） |
| 其中高危（high-severity） | **14 个** |
| 占 2025 年全年 Firefox 高危漏洞修复比例 | **~1/5** |
| 扫描的 C++ 文件数 | ~6,000 |
| 发现第一个漏洞耗时 | **20 分钟** |
| 验证提交第一个漏洞期间 Claude 又发现的额外崩溃输入 | 50+ 个 |
| 漏洞利用（exploit）开发耗费 API 费用 | ~$4,000 |
| exploit 成功率 | 极低（数百次尝试中仅 2 次成功） |

---

## 事件经过

### 起点：CyberGym benchmark 接近满分

2025 年底，Anthropic 注意到 Opus 4.5 接近解决 CyberGym（测试 LLM 能否复现已知安全漏洞的 benchmark）的所有任务。他们需要更难的评测，于是：

1. 构建了 Firefox 历史 CVE 数据集，测试 Claude 能否复现
2. 发现 Opus 4.6 能**复现高比例的历史 CVE**（每个都曾耗费人类大量时间才发现）
3. 担忧：历史 CVE 可能已在训练数据中，测试不公平

→ 于是改为寻找**当前版本 Firefox 的全新未知漏洞**

### 选择 Firefox 的原因

- 极度复杂的代码库
- 全球最受测试的开源项目之一
- 数亿用户依赖
- 浏览器漏洞特别危险（每天处理不可信外部内容）

### 第一个漏洞：20 分钟

从 JavaScript 引擎入手（独立子系统，攻击面宽）：
- Claude 报告发现一个 **Use After Free**（内存漏洞：攻击者可用任意恶意内容覆盖数据）
- Anthropic 研究员在独立虚拟机中验证
- 提交 Bugzilla，附带漏洞描述 + Claude 编写的候选修复补丁

提交第一个漏洞时，Claude 已经额外发现了 50+ 个独特崩溃输入。

### Mozilla 合作模式

Mozilla 主动联系 Anthropic，建议：**不需要每个都手动验证，直接批量提交所有崩溃用例**，即使不确定是否有安全影响。最终提交 112 份报告，修复在 Firefox 148.0 中发布。

---

## 漏洞发现 vs 漏洞利用的能力差距

Anthropic 进一步测试：Claude 能不能把发现的漏洞**变成实际 exploit**（攻击者用来执行恶意代码的工具）？

### 测试设计
- 给 Claude 已提交的漏洞报告
- 要求证明能读写目标系统的本地文件（真实攻击场景）
- 运行数百次，耗费约 $4,000 API 费用

### 结果
- **仅 2 次成功**开发出可用的 exploit
- 成功的 exploit 只能在**移除了沙箱等安全机制的测试环境**中运行
- Firefox 的"纵深防御（defense in depth）"对这些 exploit 有效

### 关键结论

| 能力 | 水平 |
|------|------|
| 发现漏洞 | 世界级 |
| 利用漏洞开发 exploit | 远弱于发现能力 |
| 发现漏洞的成本 | 比开发 exploit **低一个数量级** |

→ **当前阶段防御者占优**，但这个差距不会持续太久。

---

## 技术最佳实践

### 1. Task Verifier（任务验证器）

Claude 在有工具**自我验证**时效果最好。对于漏洞修复（patching agent），verifier 需要确认两件事：
1. 原始漏洞在修复后无法再被触发
2. 修复没有破坏程序原有功能（防止回归）

### 2. 向维护者提交报告时需包含

Mozilla 团队明确指出，以下三点是他们信任报告的关键：
1. **最小化测试用例（minimal test cases）**
2. **详细的概念验证（proof-of-concept）**
3. **候选修复补丁（candidate patches）**

### 3. 协调漏洞披露（CVD）原则

Anthropic 发布了[协调漏洞披露操作原则](https://www.anthropic.com/coordinated-vulnerability-disclosure)，遵循行业标准，随能力提升可能调整。

---

## 紧迫性与后续计划

> "前沿语言模型现在已是世界级漏洞研究员。"

Anthropic 的警告：**发现漏洞与利用漏洞之间的差距不会持久**。一旦未来模型突破这道屏障，需要额外的防护措施防止被恶意利用。

**给开发者的建议**：利用这个窗口期，加速提升软件安全性。

**Anthropic 的后续计划**：
- 与开发者合作继续搜索开源软件漏洞
- 开发工具帮助维护者分类处理 bug 报告
- 直接提出修复补丁
- 已发布 **Claude Code Security**（限量研究预览），将漏洞发现和修复能力直接开放给用户

---

## 个人理解

这篇文章的核心意义在于**量化了一个拐点**：AI 不再只是辅助安全研究，而是在两周内独立贡献了 Firefox 全年近 20% 的高危漏洞修复——而 Firefox 是全球测试最充分的开源项目之一。

**最值得关注的不是能发现漏洞，而是成本结构的变化**：
- 发现漏洞：很便宜（20 分钟发现第一个）
- 开发可用 exploit：$4,000 仍基本失败

这意味着**防御者可以用极低成本批量发现并修复漏洞**，而攻击者要将漏洞武器化仍需大量额外工作。这个窗口期是安全界的宝贵机会，但不知道能持续多久。

## Related

- [[Trustworthy Agents in Practice]]
