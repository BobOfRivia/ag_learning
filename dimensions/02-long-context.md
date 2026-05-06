# 维度 2：Long Context（长上下文）

## 一句话定义

模型在单次调用中能感知和处理的信息量上限，以及在这个范围内的实际利用质量。

## 核心概念与技术

### 最重要的区分：标称容量 vs 有效容量

这是理解这个维度的关键入口。每个模型都有一个"标称 context window"——API 接受的最大 token 数。但模型在这个范围内的实际表现远非均匀。有效容量（effective context）衡量的是模型能在其中可靠推理的真实范围，它往往远低于标称值。

NoLiMa benchmark（LMU Munich + Adobe Research，ICML 2025）提供了最有说服力的数据：当去除问题和答案之间的表面关键词匹配后（迫使模型做真正的语义理解而非模式匹配），13 个前沿模型中有 11 个在仅 32K tokens 时就降到基线性能的 50% 以下。GPT-4o 从近乎完美的 99.3% 基线骤降至 69.7%。这意味着在复杂任务上，有效容量可能不到标称值的 5%。

这个鸿沟的根源有三个：注意力稀释（sequence 越长，每个 token 分到的注意力越少）、位置编码漂移（超出训练分布的位置编码精度下降）、以及 Lost in the Middle 效应（context 中部的信息被系统性忽略）。

生产建议是：**不要在未对自己的数据做测试的情况下使用超过标称窗口 80% 的容量。** 对需要精确推理的任务，实际可靠范围可能只有 30K-64K tokens，无论标称值有多大。

### 容量 vs 利用率：瓶颈的转折

Long context 的故事不是"窗口越来越大"的线性进步史。它经历了一次关键的瓶颈转折：

第一代和第二代早期，瓶颈是**容量**——context window 太小（4K-8K），稍微复杂的任务就放不进去。大量工程精力花在"怎么把信息塞进有限的空间"（RAG、摘要、分块）。

第二代中后期到第三代，容量问题基本解决——1M tokens 成为前沿模型的标准配置。但瓶颈转向了**利用率**——信息放进去了，模型不一定能用好。Context rot（性能随 context 长度退化）、Lost in the Middle（中部信息被忽略）、working memory bottleneck（模型同时跟踪的变量数有限）成为新的核心问题。

这个转折直接驱动了 context engineering 作为独立工程学科的诞生（详见 [topics/context-engineering.md](../topics/context-engineering.md)）。当"放进去"不再是问题时，"放什么进去"和"怎么组织"成了关键。

### Long Context vs Memory

这两个概念经常被混淆，但本质不同。Long context 是单次调用的感知范围——模型在一次请求中能看到多少信息。Memory 是跨次的持久积累——模型如何记住之前的交互并在未来使用。

超长 context（1M+ tokens）可以"伪装"记忆——把所有历史对话塞进一个窗口。但这不是真正的记忆：它不做选择性遗忘、不做抽象提炼、不处理矛盾信息、成本随历史线性增长。真正的记忆系统需要写入、组织、检索、更新、遗忘的完整生命周期管理。详见 [dimensions/06-memory.md](06-memory.md) 的区分讨论。

两者也存在实际的工程互动：context window 越大，可加载的记忆越多，对检索精度的要求相对降低；但 context window 的成本也越高，全量加载的经济性随历史长度线性恶化。

### Lost in the Middle：持续存在的结构性问题

2023 年 Stanford 和 UC Berkeley 的 "Lost in the Middle" 研究揭示了 LLM 对 context window 中信息的关注度分布不均匀：开头和结尾的信息被充分利用，中间的信息被系统性忽视。关键信息从首尾位置移到中间时，准确率下降超过 30%。

到 2026 年，这个问题在所有前沿模型上依然存在。TokenMix 在 2026 年 4 月的跨模型测试显示，每个模型对中部信息都有 10-25% 的精度退化。128K+ context window 的模型同样受影响。

根本原因与位置编码有关。当前主流模型使用的 RoPE（Rotary Position Embedding）引入了距离衰减效应，使得远离序列开头和结尾的 token 落入低注意力区。这不是一个可以通过简单工程手段完全解决的问题——它根植于当前 transformer 架构的注意力机制中。

应对策略在 context engineering 中详述：关键信息做冗余放置（首尾各放一份）、周期性更新任务清单（把目标推到 context 末尾）、对长工具输出做摘要而非原文拼接。

### 使长上下文成为可能的核心技术

**位置编码。** RoPE 是当前几乎所有前沿模型的位置编码方案。它通过旋转矩阵编码 token 的位置信息，自然支持相对位置依赖。RoPE 的扩展方法（NTK-Aware interpolation 等）可以将模型的外推能力扩展到训练长度的 32 倍。2025 年末 MIT-IBM 提出的 PaTH Attention 将位置信息从静态变为自适应和上下文感知的，在推理 benchmark 上优于 RoPE。

**注意力效率。** 标准 attention 的计算复杂度是序列长度的平方（O(n²)），这是长 context 的核心计算瓶颈。FlashAttention 系列通过最小化 GPU 内存读写来加速注意力计算，FlashAttention-3 在 H100 上达到 1.3 PFLOPs/s（FP8 精度），比 FlashAttention-2 快 1.5-2.0 倍。Ring Attention 将注意力计算分布到多个 GPU，通过通信和计算的完全重叠实现 context 长度随设备数线性扩展。稀疏注意力（如 α-entmax 替代 softmax，让不相关 token 获得精确的零注意力权重）在极长序列上的泛化能力显著优于密集注意力。

**KV-cache 基础设施。** 推理时的关键瓶颈是 KV-cache 的加载速度。KV-cache 高度可压缩——Google Research 的 TurboQuant 将其压缩到 3.5-bit，缩减 6 倍且几乎无精度损失。ICMSP 在 CES 2026 发布的 4 层存储层级将 NVMe 纳入 KV-cache 地址空间，使缓存可以跨推理持久化。SideQuest（2026 年 2 月）专门为 agentic 工作负载设计的 KV-cache 管理方案，支持多步推理场景的缓存复用。KV-cache 优化的工程实践在 [topics/context-engineering.md](../topics/context-engineering.md) 的 KV-Cache 章节详述。

**Test-Time Training（TTT）。** 一种新兴方法，通过在推理时对模型做轻量微调来适应长 context，实现与 context 长度无关的常数推理延迟。TTT-E2E 在 2M context 上相比标准 transformer 有 35 倍加速。当前仍处于研究阶段。

## 当前技术格局（截至 2026-04）

### 前沿模型 context window 对比

| 模型 | 标称窗口 | 最大输出 | 输入价格（$/M tokens） | 缓存价格 |
|------|---------|---------|----------------------|---------|
| Gemini 3.1 Pro | 1M | 64K | $2.00（≤200K）/ $4.00（>200K） | — |
| GPT-5.5 | 1M | 128K | $5.00 | ~$2.50（batch）/ 90% 缓存折扣 |
| Claude Opus 4.7 Adaptive | 1M | 64K | $5.00 | $0.50（90% prompt cache） |
| DeepSeek V4 Pro | 1M | 384K | $1.74 | $0.145（cache hit） |

四个前沿模型都标称 1M tokens。差异主要在三个方面：最大输出长度（DeepSeek V4 Pro 的 384K 遥遥领先）、超长 context 的阶梯定价（Gemini 在 200K 以上翻倍）、以及缓存策略和折扣幅度。

### 有效容量的实际数据

标准 NIAH（Needle-in-a-Haystack）测试在 2026 年已成为入门门槛——前沿模型都能做到单事实检索接近完美。但更复杂的评测揭示了真实水位：

RULER benchmark 测试多跳追踪、聚合等复杂任务。历史数据显示 GPT-4-1106 从 4K 的 96.6 降到 128K 的 81.2，Llama 3.1-70B 从 96.5 降到 66.6。Gemini 1.5 Pro 表现最稳（128K 时仍有 94.4，仅降 2.3 分）。Gemini 3.1 Pro 在 500K-1M 范围有效容量评分最高，但第三方独立验证尚不充分。

多事实检索（更接近真实使用场景）的平均召回率约为 60%，远低于单事实 benchmark 的 99.7%。

### 格局判断

行业普遍预期 context window 的标称值将在 1-2M tokens 稳定。创新重心已从继续扩大窗口转向三个方向：压缩（用更少的 token 表达同样的信息）、缓存（避免重复计算已处理的 context）、记忆增强（用外部记忆系统扩展感知范围，而非硬塞进 context）。

## 演进路径

- **2020-2022（第一代）**：4K-8K tokens。Context window 是硬约束——几轮对话就撑满。所有工程精力花在"怎么在有限空间内塞更多信息"。
- **2023（第二代早期）**：GPT-4 的 32K（2023-03）打开扩展空间。Lost in the Middle 问题被发现和研究。
- **2023 末**：GPT-4 Turbo 128K（2023-11）、Claude 2.1 200K（2023-11）。10-25 倍扩展开始侵蚀简单 RAG 场景。
- **2024 初**：Gemini 1.5 Pro 1M（2024-02）。百万级 context 在技术上证明可行。RAG 塌缩信号明确。
- **2024 末-2025**：瓶颈转折点。容量不再是问题，利用率成为焦点。FlashAttention-3 发布，Ring Attention 成熟。NoLiMa 等 benchmark 揭示有效容量远低于标称值。
- **2025-2026**：Context engineering 确立为独立学科。Provider 原生压缩 API 出现（Anthropic compact、Google ADK compaction）。KV-cache 基础设施快速发展。行业共识：窗口大小趋于稳定，工程重心转向利用率优化。
- **趋势**：context window 的边际扩展价值递减——从 4K 到 128K 的提升是质变（解锁了全新的使用场景），从 1M 到 2M 的提升是量变（只服务于少数边缘场景）。下一步价值创造不在"放进更多 token"，而在"更聪明地使用已有空间"。

## 对四层骨架的影响

### 第一层（Agent 内核）

**塌缩**：
- "信息放不进 context 所以必须精心裁剪 prompt"的约束大幅放松——但完全消除这种约束的想法是危险的幻觉，有效容量仍远低于标称值
- 简单的对话历史管理（滑动窗口、硬截断）让位于更精细的策略

**涌现**：
- Context engineering 整个学科的驱动力。当 context window 从稀缺资源变成需要动态管理的大空间时，"什么该放进去、什么该拿出来、怎么组织"成为 L1 最核心的工程问题
- Thinking tokens 与内容空间的零和竞争。Reasoning model 的 thinking 可能占据大量 context 空间（128K 窗口中 thinking 占 60K，实际可用仅 68K），context engineering 需要和 reasoning budget 管理协同设计
- 压缩策略成为关键技术分支——锚定式迭代摘要、观察遮蔽、provider 原生压缩等方案的选型和组合

### 第二层（与世界的接口）

**塌缩**：
- 简单 RAG pipeline 大面积塌缩。"检索-拼接-回答"的基本 RAG 流程在很多场景下可以直接用长 context 替代——把文档全部塞进 context，让模型自己找
- RAG 中间环节（query rewriting、re-ranking）被 reasoning model + 长 context 的组合吸收

**残余 RAG 价值——五因素判断框架**：RAG 在以下条件下仍不可替代：（1）语料超出 context 容量——数百万文档级别的知识库物理上塞不进 1M 窗口；（2）相关比例低——当不到 20% 的数据与查询相关时，RAG 的精准检索优于全量塞入的注意力稀释；（3）延迟 SLO 严格——1M token 请求延迟 30-60 秒，RAG pipeline 约 1 秒，对实时交互场景差距决定性；（4）数据高频更新——文档持续变化时，每次全量塞入成本不可接受；（5）查询量大——高并发场景下长 context 的 1250 倍每查询成本无法承受。满足任一条件时 RAG 仍是正确选择。这个框架回答"该不该做 RAG"；如果决策结果是该做，下一个问题是"做哪一层 RAG"——四个成熟度层级（Naive / Advanced / Modular / Agentic / Adaptive）的内部机制和落地程度判断详见 [topics/advanced-rag-variants.md](../topics/advanced-rag-variants.md)。

**涌现**：
- 混合架构成为生产默认——RAG 做精准检索，长 context 做全局推理，根据查询路由选择
- 工具描述在 MCP 生态下可能占据大量 context（数百个工具），催生工具检索需求

### 第三层（多 Agent 协作）

**涌现**：
- Context 隔离成为多 agent 的新动机。长 context + 长 reasoning trace 会迅速膨胀单 agent 的 context，把独立子任务交给子 agent 处理是一种"用 agent 隔离管理 context"的策略

### 第四层（生产化）

**塌缩**：
- 基于固定 context 大小的成本预算模型失效——同一个模型，不同任务的 context 消耗可能差几十倍

**涌现**：
- 超长 context 的阶梯定价引入新的成本工程问题（Gemini 在 200K 以上每 token 翻倍）
- Context drift 被识别为主要失败模式——约 65% 的企业 AI 失败可归因于 context drift 或 memory loss，而非 context 的物理耗尽
- 有效容量的评测成为生产上线前的必要步骤（不能信任标称值）

## 与其他维度的耦合

- **× Inference-Time Compute（维度 7）**：Thinking tokens 和 context window 争夺空间，存在直接的零和竞争。这是第三代最尖锐的工程张力之一——更深的思考和更多的上下文不可兼得。
- **× Memory（维度 6）**：最常混淆的一对。长上下文能伪装记忆但不是记忆。两者的工程互动在于：context 越大，可加载的记忆越多，但成本也越高。长 context 让简单 RAG 塌缩，但不替代真正的记忆系统。
- **× Tool Use（维度 4）**：工具描述消耗 context。MCP 生态下可用工具达到数百个时，工具描述可能占据大量空间，催生"工具检索"需求——根据当前任务动态选择相关工具子集加载。
- **× Reasoning（维度 1）**：Reasoning model 让 RAG 进一步塌缩——模型自己决定何时再查，复杂多跳 RAG 编排被简化。但 reasoning 也加剧了 context 压力——thinking tokens 本身就消耗 context 空间。
- **× Instruction Following（维度 5）**：Context 越长，指令和内容的混合越复杂，IF 难度越高。研究发现指令经常和冗长的知识片段争夺注意力，context 长度加剧指令遵循的失败率。详见 [dimensions/05-instruction-following.md](05-instruction-following.md)

## 关键不确定性

- **有效容量会随模型迭代自然提升吗？** 还是 Lost in the Middle 是 transformer 注意力机制的结构性限制，需要架构级突破？PaTH Attention 等自适应位置编码能否根本解决这个问题？
- **1-2M 的窗口稳定点是否成立？** 如果 TTT 等常数延迟推理技术成熟，窗口大小可能继续扩展到 10M+，这会进一步改变 RAG 的残余价值图景。
- **长 context 的成本曲线如何演进？** 当前超长 context 的阶梯定价是否会随基础设施成熟而拉平，还是注意力复杂度的二次方本质意味着长 context 永远更贵？
- **Context rot 的退化曲线能否被改善？** 当前研究表明性能退化在约 30K token 后加速。这个拐点是可以通过训练推后的，还是在给定模型规模下有固定上限？

## 信息源

- **Benchmark 追踪**：RULER leaderboard（多维度长 context 评测）、NoLiMa（语义理解而非模式匹配的长 context 评测）、LongBench v2、MRCRv2
- **技术深度**：FlashAttention 系列论文（Tri Dao）、Ring Attention 论文（Hao Liu et al.）、RoPE 扩展分析（ACL 2025 COLING）
- **模型定价追踪**：BenchLM.ai 的 context window 对比页面（持续更新各模型的标称/有效窗口和定价）
- **arXiv 关键词**：long-context evaluation、lost in the middle、KV cache compression、sparse attention、test-time training、context engineering

## 参考资料

- [LLM Context Window Comparison 2026: Advertised vs Effective — BenchLM.ai](https://benchlm.ai/blog/posts/context-window-comparison) — 2026-04 各前沿模型的标称/有效 context window 对比，含定价和输出上限
- [LLM Context Window Limitations in 2026 — Atlan](https://atlan.com/know/llm-context-window-limitations/) — 三大核心局限（advertised-effective gap、working memory bottleneck、context rot）的系统分析
- [Long-Context Models vs. RAG: Production Decision Framework — TianPan.co](https://tianpan.co/blog/2026-04-09-long-context-vs-rag-production-decision-framework) — 五因素决策框架，含延迟和成本对比数据（RAG ~1s vs 长 context 30-60s，1250x 成本差）
- [LLM Context Window Management and Long-Context Strategies 2026 — Zylos Research](https://zylos.ai/research/2026-01-19-llm-context-management) — 管理策略综述，含 FlashAttention-3、Ring Attention、TTT 性能数据
- [RULER: What's the Real Context Size of Your Long-Context Language Models? — NVIDIA](https://github.com/NVIDIA/RULER) — 多维度长 context benchmark，超越单一 NIAH 测试
- [Lost in the Middle: How Language Models Use Long Contexts — Stanford/UC Berkeley](https://arxiv.org/abs/2307.03172) — 原始论文，揭示 context 中部信息被忽视的现象
- [Understanding the RoPE Extensions of Long-Context LLMs — ACL 2025](https://arxiv.org/abs/2406.13282) — 从注意力视角分析 RoPE 扩展方法
- [How LLMs Scaled from 512 to 2M Context: A Technical Deep Dive](https://amaarora.github.io/posts/2025-09-21-rope-context-extension.html) — 从 512 到 2M 的技术路径全景
- [FlashAttention-3 — PyTorch](https://pytorch.org/blog/flashattention-3/) — FlashAttention-3 技术细节和性能数据

## 更新日志

- 2026-05-05：在"残余 RAG 价值"段末尾加反向链接指向 topics/advanced-rag-variants.md，把"该不该做 RAG"（本文件）和"做哪一层 RAG"（topic）两个正交决策串起来
- 2026-05-01：耦合章节增加维度 5 Instruction Following 交叉引用
- 2026-05-01：初次创建。覆盖标称 vs 有效容量、Lost in the Middle、核心技术（RoPE/FlashAttention/Ring Attention/KV-cache）、技术格局、RAG 残余价值五因素框架、四层影响
