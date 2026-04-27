# 简历大纲（基于近两年工作复盘）

> 版本：初稿，用于审查经历取舍、表达口径和量化成果。

## 一、个人定位

- 方向：大模型 Infra / AI Infra / 推理基础设施工程师
- 核心关键词：大模型推理基础设施、LisaRT、TensorRT-LLM、TensorRT、CUDA、CUTLASS、FlashInfer、DeepGEMM、MambaViT、MindVLA、MindGPT、GPU/NPU 性能分析
- 主要优势：
  - 具备大模型与多模态模型推理基础设施经验，覆盖模型选型、引擎拉通、算子优化、精度对齐、性能评测、稳定性排障和交付支撑。
  - 熟悉 GPU 推理链路与性能分析方法，能够从框架、算子、内存拷贝、CUDA Graph、NUMA、NCCL 等层面定位性能与稳定性瓶颈。
  - 能将实验、评测、排障和交付过程沉淀为工具流程、评测体系和团队文档，提升大模型基础设施的复用效率。

## 二、简历摘要候选

### 版本 A：大模型 Infra 主线

近两年专注于大模型推理基础设施与多模态模型端云推理能力建设，参与 MindGPT、MindVLA、LisaRT 等核心项目的研发、优化和交付。熟悉 TensorRT-LLM、TensorRT、LisaRT、CUTLASS、FlashInfer、DeepGEMM 等推理框架和算子库，具备从模型评估、引擎拉通、算子融合、精度对齐、性能评测到线上稳定性排障的完整经验。曾在 MindVLA、MindGPT3.x 等项目中实现多项性能与稳定性优化，包括云侧 MambaViT 推理延迟优化至 7.25ms、MOE prefill 加速 11.5%~16.3%、采样层 decode 吞吐提升 22.68%、NUMA 波动和 NCCL 内存泄漏问题定位等。

### 版本 B：大模型 Infra + 高性能计算

大模型 Infra 工程师，重点关注推理服务底层性能、GPU/NPU 架构、CUDA 算子、推理框架和性能分析体系。具备 Nsight Compute、simpleperf、cutlass_profiler 等工具实践经验，能够结合 roofline、指令达成率、内存访问、拷贝链路和调度开销进行系统级性能优化。参与 MindVLA、MindGPT 和 LisaRT 架构升级，完成端侧 ViT、云侧 MambaViT、MOE、MLA、采样、GEMM 等模块优化，并沉淀性能评测、精度对齐、稳定性排障和团队培训方法。

## 三、工作经历结构

### 理想汽车｜大模型 Infra / AI 引擎方向｜约 2 年

建议简历中按 3~4 个重点项目展开，不建议把所有 OKR 都铺开。优先突出：

1. MindGPT3.x 大模型推理基础设施优化
2. MindVLA 端云推理能力建设与交付
3. LisaRT 架构升级与开源框架评测体系
4. 性能分析、精度对齐与稳定性保障体系

## 四、项目经历候选

### 项目一：MindVLA 端云推理能力研发与性能优化

**项目背景**

MindVLA 涉及车端与云端多模态推理能力建设，需要支撑视觉基座升级、模型选型、端云引擎拉通、性能优化、稳定性排查和对外协作交付。

**职责**

- 支撑端侧 ViT 模型选型与性能调优，参与 MindViT / Hybrid ViT 在 OrinX、ThorU 等平台上的优化验证。
- 拉通云侧 MambaViT / Hybrid ViT 推理链路，拆解性能瓶颈并推进算子、流程和框架级优化。
- 完成 MindVLA 精度对齐、稳定性问题排查和 TRT engine 自助转换能力，支撑跨部门交付。
- 与算法团队协作进行模型结构评估，包括 Stage4、Stage5、3D-VIT、分组卷积、Mamba 耗时占比等选型分析。

**关键成果**

- 端侧 ViT 优化：通过 TensorRT 10.13、分层量化和 channel-per-group 配置探索，使 MindViT v1.0 在 OrinX 平台由 15.2ms 优化至 11.1ms，提升 19.7%；在 ThorU 平台由 9.3ms 优化至 8.1ms，提升 12.9%。
- 相比开源 SIGLIP，端侧方案在 ThorU 平台加速约 40%，在 OrinX 平台加速约 73%。
- 云侧 MambaViT 优化：从初版 NHWC 拉通 33.0ms，经过 cudnn workspace 预计算、CUTLASS 深度卷积、cudnnFrontEnd 融合、LayerNorm / BatchNorm / ElemAdd 融合、D2D copy 消除、CUDA Graph、异步流拷贝等优化，最终达到 7.25ms，性能优于未开 CUDA Graph 的 TensorRT（7.25ms vs 7.5ms）。
- 在 2k batch2 场景下完成 Hybrid-ViT Mamba TRT 拉通与优化，P95 TTFT 达到 105ms，优于 1k ViT 的 114ms。
- 定位 MindVLA 稳定性问题根因为 NUMA 节点导致的 CPU 操作波动，将单算子波动指数从 7.44 降至 0.42，下降 94.35%；单路 TTFT 波动指数从 18.07 降至 3.41，下降 81.13%。
- 完成精度对齐与一键转换 TRT engine 脚本，交付 AD 团队使用。

**可放入简历的精简 bullet**

- 负责 MindVLA 端云推理链路拉通与性能优化，覆盖模型选型、TRT engine 转换、精度对齐、稳定性排查和跨团队交付。
- 优化云侧 MambaViT 推理链路，将端到端耗时从 33.0ms 降至 7.25ms，并在 2k batch2 场景实现 P95 TTFT 105ms，优于 1k ViT 方案 114ms。
- 定位 NUMA 导致的推理波动问题，将单算子波动指数降低 94.35%，单路 TTFT 波动指数降低 81.13%。
- 支撑端侧 ViT 选型调优，使 MindViT 在 OrinX / ThorU 分别加速 19.7% / 12.9%，并显著优于开源 SIGLIP。

---

### 项目二：MindGPT3.x 大模型推理基础设施优化

**项目背景**

MindGPT3.0 / 3.1 需要在大模型推理基础设施中持续优化 prefill、decode、MOE、MLA、GEMM、采样等关键链路，并保障线上多硬件环境下的稳定运行。

**职责**

- 分析 TensorRT-LLM MOE 流程，识别 workspace、group GEMM 和前后处理开销，接入 DeepGEMM / deep-groupgemm 方案。
- 优化 MLA kernel 与 metadata 流程，验证不同 batch 场景下性能收益与无回退效果。
- 将 FP8 GEMM 实现独立到公共 CUTLASS 库，并升级 CUTLASS 至 3.9。
- 将 attention 实现独立到公共 FlashInfer 库，封装并对齐 batchPrefillPagedAttention、batchPrefillPagedCustomMaskAttention、batchPrefillRaggedAttention 等接口。
- 引入 FlashInfer 采样算子和 TemperatureSoftMax 融合，优化非投机和投机采样层。
- 跟踪内存泄漏、显存踩踏等潜在稳定性问题，推动线上风险收敛。

**关键成果**

- 接入 deep-groupgemm 后，初版 group GEMM kernel 相比 CUTLASS 版本加速约 55%。
- MOE 前处理从 14ms 优化至 279us，后处理从 3ms 优化至 339us；相比 TensorRT 前处理加速 85%，后处理加速 21.6%。
- TP16 场景 prefill 时间从 712ms 降至 612ms，加速 16.3%；EP16 场景从 484ms 降至 434ms，加速 11.5%。
- 优化 flash MLA metadata 流程，高 batch 下 MLA kernel 从 161.88us 降至 148.96us，加速 7.98%，多场景无回退。
- 优化采样层：非投机采样耗时从 2.21ms 降至 0.11ms；单路 decode 吞吐从 78.37 token/s 提升至 96.15 token/s，加速 22.68%。
- 投机采样层在 K=50/P=1 场景从 5.82ms 降至 2.43ms，在 K=20/P=0.95 场景从 4.14ms 降至 2.43ms，最高加速约 70%。
- 使用 cutlass_profiler 对高 batch GEMM 调优，GEMM 加速 50%~70%。
- 定位 CPU 内存泄漏根因为 NCCL 版本问题，升级后在 A800、H800、H100 上压测 24 小时均无 CPU 内存泄漏。

**可放入简历的精简 bullet**

- 参与 MindGPT3.x 大模型推理基础设施优化，覆盖 MOE、MLA、GEMM、Attention、采样层和内存稳定性排查。
- 接入 deep-groupgemm 优化 MOE 链路，使 TP16 prefill 加速 16.3%、EP16 prefill 加速 11.5%，并将 MOE 前处理从 14ms 降至 279us。
- 基于 FlashInfer 优化采样层，将非投机采样耗时从 2.21ms 降至 0.11ms，decode 吞吐提升 22.68%；投机采样最高加速约 70%。
- 使用 cutlass_profiler 调优高 batch GEMM，取得 50%~70% GEMM 性能收益。
- 定位 NCCL 版本导致的 CPU 内存泄漏问题，并完成 A800/H800/H100 24 小时压测验证。

---

### 项目三：LisaRT 架构升级、开源框架评测与推理基础设施交付

**项目背景**

LisaRT 需要支撑多模态、VLA、强化学习、数据生产等多种场景，同时需要跟进 TensorRT-LLM、SGLang 等开源方案，建立性能评测和问题跟踪机制。

**职责**

- 对 TensorRT-LLM、SGLang、LisaRT 在 DeepSeek、Qwen 等模型下的单路性能与吞吐进行评测对比。
- 跟进开源框架在 DeepSeek V2.5 / V3、Qwen 等模型上的效果问题，为内部 LisaRT 优化方向提供依据。
- 支撑车端 LLM 实车测试，验证高通 TZ、LLCC、带宽优先级调整等方案对 CPU loading、语音链路和视觉链路阻塞的影响。
- 与高通团队现场协作，验证 LLCC 优化效果并寻找带宽临界点，支撑车端可部署模型大小判断。
- 整理 sysProfiler、sysMon 等带宽测试工具使用说明，沉淀车机端 LLM 测试流程。

**关键成果**

- 建立 TensorRT-LLM、SGLang、LisaRT 在 DeepSeek / Qwen 等模型上的性能评测对比流程，为 LisaRT 后续优化提供基线。
- 完成高通 TZ 带宽增幅控制有效性验证，覆盖实车重负载测试、CPU loading 观察、语音链路和视觉链路阻塞分析。
- 参与上海现场联调，与高通方共同验证 LLCC 优化效果并寻找带宽临界点，支撑车端模型部署决策。
- 沉淀车机端 LLM 测试指引和带宽测试工具说明，提高后续复用效率。

**可放入简历的精简 bullet**

- 建立 TensorRT-LLM、SGLang、LisaRT 在 DeepSeek / Qwen 等模型上的性能评测流程，为 LisaRT 架构升级和优化方向提供基线。
- 支撑车端 LLM 实车测试与高通现场联调，验证 TZ / LLCC / 带宽优先级方案对 CPU loading 和链路阻塞的影响。
- 沉淀车机端 LLM 测试指引及 sysProfiler / sysMon 带宽测试工具说明，提升团队测试复用效率。

---

### 项目四：性能分析体系、算子单测与团队能力建设

**项目背景**

推理引擎优化需要系统性的性能分析方法和可靠的算子测试体系，以提升定位效率、精度正确性和团队整体能力。

**职责**

- 学习并分享 Nsight Compute、simpleperf、roofline、GPU/NPU 架构、warp 指令生命周期、理论/实际占用率、内存访问合并等内容。
- 以矩阵 transpose、MatMul 等算子为例，进行带宽、指令达成率、TCC 利用率等指标分析。
- 在自研 Hybrid ViT LisaRT 拉通过程中建设算子单测体系，用于 cudnn、cudnnFrontEnd、torch-tensorrt 等算子的精度对齐和延时测量。
- 跟进线性注意力、TensorRT-LLM、DeepGEMM、FlashInfer 等前沿技术并沉淀源码分析文档。

**关键成果**

- 完成 Nsight Compute 工具分享，覆盖理论占用率、实际占用率、合并访问、roofline 模型等性能分析方法。
- 在 Nsight Compute 二期中进一步讲解计算 pipeline、warp 指令生命周期、指令达成率等指标，并以 transpose 矩阵为例达到约 83% 带宽利用率。
- 以 MatMul 算子为例，在 MNK=32x3584x128 场景下 TCC 指令达成率达到 90.6%，可作为充分利用算力的分析案例。
- 建立算子级单测与精度对齐流程，改进早期只拉通、不校验正确性的不足。

**可放入简历的精简 bullet**

- 建设推理性能分析与算子单测体系，覆盖 Nsight Compute、roofline、指令达成率、内存访问、精度对齐和延时测量。
- 组织 Nsight Compute 系列分享，以 transpose、MatMul 等算子为例沉淀 GPU 性能分析方法，帮助团队提升问题定位效率。
- 在 Hybrid ViT LisaRT 拉通过程中建立 cudnn / cudnnFrontEnd / torch-tensorrt 算子精度对齐和延时测量流程。

## 五、技能栈整理

### 大模型 Infra / 推理基础设施

- 大模型推理链路优化、模型服务性能评测、线上稳定性排查、跨硬件交付
- LisaRT、TensorRT-LLM、TensorRT、SGLang
- MindGPT、MindVLA、MindViT、Hybrid ViT、MambaViT、DeepSeek、Qwen
- TRT engine 转换、模型精度对齐、推理链路性能评测

### GPU / CUDA / 算子优化

- CUDA、CUTLASS、DeepGEMM / deep-groupgemm、FlashInfer、cuDNN、cuDNN Frontend、cuBLASLt
- GEMM、MOE、MLA、Attention、Sampling、LayerNorm、BatchNorm、Depthwise Conv、Elementwise Fusion
- CUDA Graph、异步拷贝、Pinned Host Memory、D2D copy 消除、算子融合

### 性能分析与稳定性

- Nsight Compute、simpleperf、cutlass_profiler、sysProfiler、sysMon
- roofline、occupancy、warp 指令生命周期、内存访问合并、指令达成率、NUMA、NCCL 内存泄漏排查
- 端侧 OrinX / ThorU，云侧 A800 / H800 / H100

### 工程能力

- 性能评测体系搭建
- 单元测试与精度对齐
- 问题定位与线上风险排查
- 技术文档沉淀与团队分享
- 跨团队协作与外部厂商联调

## 六、成就数字池

后续写正式简历时可按岗位 JD 选择使用：

- MindViT OrinX：15.2ms → 11.1ms，加速 19.7%
- MindViT ThorU：9.3ms → 8.1ms，加速 12.9%
- 对比 SIGLIP：ThorU 加速约 40%，OrinX 加速约 73%
- 云侧 MambaViT：33.0ms → 7.25ms
- MambaViT vs TensorRT：7.25ms vs 7.5ms
- 2k batch2 VIT：P95 TTFT 105ms vs 1k VIT 114ms
- NUMA 波动优化：单算子波动指数 7.44 → 0.42，下降 94.35%
- TTFT 波动优化：18.07 → 3.41，下降 81.13%
- group GEMM kernel：相较 CUTLASS 加速 55%
- MOE 前处理：14ms → 279us
- MOE 后处理：3ms → 339us
- TensorRT 对比：前处理加速 85%，后处理加速 21.6%
- TP16 prefill：712ms → 612ms，加速 16.3%
- EP16 prefill：484ms → 434ms，加速 11.5%
- MLA kernel：161.88us → 148.96us，加速 7.98%
- 非投机采样：2.21ms → 0.11ms
- Decode 吞吐：78.37 token/s → 96.15 token/s，加速 22.68%
- 投机采样：5.82ms → 2.43ms / 4.14ms → 2.43ms，最高加速约 70%
- 高 batch GEMM：加速 50%~70%
- NCCL 内存泄漏：A800 / H800 / H100 压测 24 小时无 CPU 内存泄漏
- transpose 算子：达到约 83% 带宽利用率
- MatMul 指令达成率：MNK=32x3584x128 场景 TCC 指令达成率 90.6%

## 七、建议正式简历呈现方式

### 如果投递方向是大模型 Infra

优先保留：

1. MindGPT3.x 大模型推理基础设施优化
2. LisaRT / TensorRT-LLM / SGLang 评测体系与架构升级
3. MindVLA 云侧 MambaViT 优化与跨团队交付
4. 稳定性排障、精度对齐、性能评测体系和算子单测

弱化：

- 过细的单个 CUDA kernel 实现细节
- 车端实车测试细节
- 个人复盘类表达
- 过多文档沉淀内容

### 如果投递方向是自动驾驶 AI / 端云多模态

优先保留：

1. MindVLA 端云推理能力研发
2. 端侧 ViT / 3D-VIT / ThorU / OrinX 优化
3. 车端 LLM 实车测试和高通联调
4. MindGPT 推理引擎优化作为底层能力补充

### 如果投递方向是 CUDA / HPC / 算子优化

优先保留：

1. MOE / GEMM / MLA / Sampling / Attention 算子优化
2. CUTLASS / FlashInfer / DeepGEMM / cuDNN Frontend 实践
3. Nsight Compute / roofline / 指令达成率 / 带宽利用率案例
4. 算子单测与精度对齐体系

## 八、已确认的简历约束

1. 目标岗位：大模型 Infra。
2. 项目名可直接出现：MindGPT、MindVLA、LisaRT 等无需脱敏。
3. 篇幅：一页。
4. 语言：中文。
5. 内容风格：只保留硬成果，弱化软实力、成长复盘和文档沉淀。
