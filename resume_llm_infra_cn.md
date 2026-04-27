# 王朗｜大模型 Infra / AI Agent 工程化 / 推理基础设施工程师

## 个人简介

近两年专注大模型 Infra、推理基础设施与 AI Agent 工程化，参与 MindGPT、MindVLA、LisaRT 等核心项目研发与交付。Vibe Coding 分享者与拥护者，系统实践 Claude Code / Managed Agents / long-running harness，将工程师角色从直接编码扩展为 Agent 监督者与 Harness 设计者；关注 Planner/Generator/Evaluator 分离、Brain/Hands/Memory 解耦、持久化 session log、自动评测闭环和上下文工程。熟悉 TensorRT-LLM、TensorRT、CUTLASS、FlashInfer、DeepGEMM、cuDNN Frontend 等推理框架与算子库，具备 GPU/NPU 架构优化、模型评估、引擎拉通、算子优化、精度对齐、性能评测、稳定性排障和跨硬件交付经验。

## 教育经历

- 中国科学院大学｜网络空间安全学院｜硕士研究生｜高性能计算方向｜2021.09 - 2024.06
- 北京信息科技大学｜计算机科学与技术｜本科｜2015.09 - 2019.06

## 工作经历

### 理想汽车｜AI 引擎 / 大模型 Infra｜2024.07 - 至今

#### MindGPT3.x 大模型推理基础设施优化

- 参与 MindGPT3.0 / 3.1 推理基础设施优化，覆盖 MOE、MLA、GEMM、Attention、Sampling、内存稳定性等关键链路，支撑大模型推理性能与线上稳定性提升。
- 分析 TensorRT-LLM MOE 流程并接入 deep-groupgemm，初版 group GEMM kernel 相比 CUTLASS 版本加速约 55%；MOE 前处理从 14ms 降至 279us，后处理从 3ms 降至 339us。
- 优化 MOE prefill 链路：TP16 场景从 712ms 降至 612ms，加速 16.3%；EP16 场景从 484ms 降至 434ms，加速 11.5%。
- 基于 FlashInfer 优化采样层：非投机采样耗时从 2.21ms 降至 0.11ms，单路 decode 吞吐从 78.37 token/s 提升至 96.15 token/s，加速 22.68%；投机采样最高加速约 70%。
- 使用 cutlass_profiler 调优高 batch GEMM，取得 50%~70% GEMM 性能收益；定位 NCCL 版本导致的 CPU 内存泄漏问题，并完成 A800 / H800 / H100 24 小时压测验证。

#### MindVLA 端云推理能力建设与交付

- 负责 MindVLA 端云推理链路拉通与优化，覆盖模型选型、TRT engine 转换、精度对齐、稳定性排查和跨团队交付。
- 优化云侧 MambaViT / Hybrid ViT 推理链路，从初版 NHWC 拉通 33.0ms 优化至 7.25ms，性能优于未开 CUDA Graph 的 TensorRT（7.25ms vs 7.5ms）。
- 在 2k batch2 场景完成 Hybrid-ViT Mamba TRT 拉通与优化，P95 TTFT 达到 105ms，优于 1k ViT 方案 114ms。
- 定位 NUMA 节点导致的推理耗时波动问题，将单算子波动指数从 7.44 降至 0.42，下降 94.35%；单路 TTFT 波动指数从 18.07 降至 3.41，下降 81.13%。
- 支撑端侧 MindViT 选型与调优，使 OrinX 平台从 15.2ms 降至 11.1ms，加速 19.7%；ThorU 平台从 9.3ms 降至 8.1ms，加速 12.9%。

#### M100 MoE2.0 / Attention 同步优化与 TM 地址分配

- 参与 M100 MoE2.0 Attention 层适配与同步优化，将 barrier 同步逐步改造为信号量同步，完成关键 barrier removal，推动初版有效 token 输出与 Golden bit-wise 对齐，并进入指令并行执行优化阶段。
- 设计基于生命周期冲突图与贪心装箱的 TM 基址生成方法，建模 loop transient / persistent 变量生命周期、良性/恶性踩踏、非 in-place 约束和 allowed overlaps，在 2MB Tile Memory 约束下自动生成变量起始地址。
- 编写 barrier-removal harness，用于约束 Claude Code 移除 barrier 的工作流，沉淀从观测、假设、修改到 bit-wise 验证的同步优化流程。

#### AI Agent / Vibe Coding 工程化与技术分享

- Vibe Coding 分享者与拥护者，围绕 Claude Code 工作流、上下文工程、CLAUDE.md / Memory / Skills / Subagents、Plan Mode、worktree 隔离、提示词工程等主题沉淀系统化方法论，推动团队从“使用 AI 编码”升级为“设计 Harness 管理 Agent”。
- 设计并实践面向长时间系统优化任务的 Agent Harness，将 Planner、Generator、Evaluator、Memory 解耦，通过多 worktree 并行探索、独立评估、session log 持久化和自动化回归验证支持复杂优化任务持续收敛。
- 在 VLIW-SIMD 虚拟机优化实验中，通过多 Agent 竞争 + Perf Agent trace 评估 + 增量融合，将 cycles 从 147,734 降至 1,391，取得 106.2x 加速，8/9 测试通过，并超过 Claude Opus 4.5 发布时最佳记录 1,487 cycles。
- 在低人工干预 Managed-Agents 实验中，构建 Brain / Hands / Memory 三层解耦架构，使用 Sonnet 4.6 长时间优化 VLIW-SIMD kernel，从 9,936 cycles 推进至 2,337 cycles candidate，并沉淀 Planner 决策、Evaluator 事实验证、worktree 生命周期管理等执行规则。
- 基于 AI Agent 主导的 CUDA kernel 优化 harness，在 H800 dense attention forward 场景将 Round 17 latency 优化至 0.303ms，超过 FlashInfer 0.6.8 baseline 0.311ms，验证 long-running agent harness 在高性能 kernel 优化中的工程可行性。

## 技术栈

- 推理基础设施：LisaRT、TensorRT-LLM、TensorRT、SGLang、TRT engine 转换、精度对齐、性能评测、稳定性排障、信号量同步、barrier removal
- CUDA / 算子优化：CUDA、CUTLASS、DeepGEMM、FlashInfer、cuDNN Frontend、cuBLASLt、CUDA Graph、异步拷贝、算子融合、TM 地址分配
- AI Agent 工程化：Vibe Coding、Claude Code、Managed Agents、long-running harness、Planner/Generator/Evaluator、Brain/Hands/Memory、多 worktree 并行、session log、上下文工程、Memory/Skills/Subagents、自动评测闭环
- 性能分析：Nsight Compute、simpleperf、cutlass_profiler、sysProfiler、sysMon、roofline、occupancy、NUMA、NCCL
