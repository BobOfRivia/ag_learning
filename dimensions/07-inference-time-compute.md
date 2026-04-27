# 维度 7：Inference-Time Compute（推理时计算）

## 一句话定义

模型在推理阶段为一次请求投入多少计算资源——这是 Reasoning 维度的成本面和可控性面。Reasoning 问的是"模型能想多深"，Inference-Time Compute 问的是"想这么深要花多少，谁来控制"。

## 核心概念与技术

### 两种 scaling 的博弈：训练时 vs 推理时

传统的 scaling law（Chinchilla 及其前身）只优化训练侧：给定总计算预算，如何分配模型大小和训练数据量。推理时的计算被视为固定成本——一次前向传播，结束。Reasoning model 打破了这个假设：模型在推理时主动"多想"，推理不再是单次前向传播，而是一段可长可短的内部搜索过程。这意味着推理时计算变成了一个独立的、可调节的 scaling 维度。

这个转变的核心发现来自 2024 年的一系列研究。DeepMind 的工作（Snell et al., 2024）表明，在中等难度任务上，一个小模型配合更多的推理时计算，可以追平甚至超过 14 倍大的模型。但这个结论有明确的边界条件：对困难任务，训练时计算仍然不可替代——小模型无论花多少推理时间，如果它从未在训练中学到相关能力，推理时再怎么搜索也搜索不到。

2026 年 4 月的 Train-to-Test（T²）scaling law 进一步推进了这个方向。T² 框架同时优化模型大小、训练数据量和推理时采样次数，结论是：当把推理成本纳入考量后，最优的训练策略会大幅偏向过度训练（overtrain）——训练一个更小的模型，用远超 Chinchilla 比例的数据量训练它，然后把节省下来的计算预算花在推理时。这对传统 scaling law 是一个实质性修正。

一个实用的量化参考来自 Epoch AI 的分析：训练时和推理时计算的兑换比率因任务类型而异。对一般性的 scaling 方法，每增加 1-2 个数量级的推理计算，大约可以节省 1 个数量级的训练计算。对可验证问题（数学、代码），兑换比率更激进：5-6 个数量级的推理计算增加可以换来 3-4 个数量级的训练节省。但这些技术不能无限叠加，组合使用时边际收益递减。

### 推理时计算的两大策略族

推理时计算不是一个单一的技术，而是一个策略谱系。当前的研究将其分为两大族群。

**搜索与采样策略（外部编排）。** 这一族的共同思路是：让模型生成多个候选答案，然后用某种机制选出最好的。

Best-of-N 是最简单的形式：同一个问题采样 N 次，用一个独立的评估器（通常是 reward model）选出最佳答案。它的优势是实现简单、与模型解耦，但计算成本线性增长，且强烈依赖评估器的质量。

Beam Search 在生成过程中维护多条候选路径，每一步只保留得分最高的 K 条，逐步收敛到最优解。比 Best-of-N 更高效（在生成过程中就剪枝，而非生成完再选），但需要能对中间步骤评分的 Process Reward Model（PRM）。

MCTS（Monte Carlo Tree Search）将推理建模为搜索树，结合探索（尝试新路径）和利用（深化已知好路径）。实验数据表明，在充足的计算预算下，MCTS 一致优于 beam search、Best-of-N 和 majority vote。但它的工程复杂度最高，且对 reward model 的质量要求最严格。

**内化式推理（模型内部行为）。** 这一族不依赖外部编排，而是模型自身在生成过程中进行延长的内部推理——表现为 thinking tokens 或 extended thinking。模型在输出最终答案前，先生成一段内部思考过程，这段思考中可能出现回溯、自我验证、分支探索等行为。

当前的前沿模型（OpenAI o 系列、Claude extended thinking、Gemini Deep Think、DeepSeek-R1）都采用内化式推理作为主要的推理时计算策略。从工程角度看，内化式推理对用户更透明（只需调用 API，不需要自己编排搜索循环），但可控性更低——模型自己决定"想多久"，工程师只能通过 thinking budget 间接调节。

**两族的关系不是替代而是互补。** 内化式推理是当前的主流，但搜索与采样策略在特定场景下仍有价值：当模型内部推理不够可靠时，外部多次采样提供额外的保险；当任务有可验证的正确答案（数学、代码）时，Best-of-N + verifier 的组合可以进一步提升准确率。2025 年末的大规模研究（"The Art of Scaling Test-Time Compute"）的核心结论是：没有单一策略在所有场景下占优，最优策略取决于任务难度、模型类型和计算预算。

### Thinking Tokens 的经济学

Thinking tokens 是内化式推理的计费单位，也是当前推理时计算成本的核心矛盾所在。

**计费模式因厂商而异，且差异显著。** OpenAI 的 reasoning tokens 按输出 token 价格计费，但不对用户可见——你为模型的"思考过程"付费，但看不到它想了什么（除非启用 reasoning summary 功能获取摘要）。Anthropic 的 thinking tokens 对用户完全可见，同样按输出 token 价格计费。Gemini 对 thinking tokens 单独标价。

**核心矛盾：成本黑洞。** Reasoning model 的 thinking tokens 消耗通常远超最终输出。一个简单的问题，模型可能生成几百个 thinking tokens 但只输出一句话；一个复杂问题，thinking tokens 可能达到数万。由于 thinking tokens 按输出价计费（输出价通常是输入价的 3-5 倍），实际成本结构与传统模型完全不同。有分析指出，reasoning model 的实际输出成本通常是标称价格的 2-5 倍——因为大量 token 花在了不可见的思考上。

**价格下降趋势与消耗增长的赛跑。** Epoch AI 的数据显示，等性能水位的 LLM 推理价格以每年约 10 倍的速度下降（2024 年初以来中位数达每年 50-200 倍，取决于性能基准线）。但这个暴跌主要体现在非 reasoning 模型和较低性能水位上。Reasoning model 的每 token 价格也在下降（o3 比 o1 便宜 87%），但单次请求的总 token 消耗同步增长，导致单次请求的总成本并不一定下降。这就是 LLM 领域的成本悖论：每 token 越来越便宜，但每次请求的总花费可能越来越高。

### Reasoning Effort 控制：工程师的直接旋钮

当前三大厂商都提供了 API 级别的推理算力控制参数，这是工程师管理推理时计算的直接接口。

**Anthropic 的 `effort` 参数。** 提供 low / medium / high / max 四个级别（Claude Opus 4.7 额外支持 xhigh）。默认值为 high。effort 影响所有输出 token——包括文本回复、工具调用和 extended thinking。这意味着 low effort 不仅让模型想得少，还让它做得少（更少的工具调用、更简短的回复）。一个关键的实用数据点：Claude Opus 4.5 在 medium effort 下的表现与 Sonnet 4.5 在 high effort 下持平，但输出 token 减少 76%。这说明 effort 参数不仅是成本旋钮，也是模型选型的替代维度——调低大模型的 effort 可能比用满小模型更划算。

**OpenAI 的 `reasoning_effort` / `reasoning.effort`。** 提供 none / minimal / low / medium / high / xhigh（模型相关）。GPT-5.5 默认值为 medium。Reasoning tokens 不可见但可通过 `output_tokens_details.reasoning_tokens` 查询消耗量。OpenAI 还提供 reasoning summary 功能（concise / detailed），让用户获取推理过程的摘要而非完整 trace。

**Google 的 `thinkingBudget`。** Gemini 模型允许直接设置 thinking tokens 的上限。

三家的共同趋势是：从早期的"全有或全无"（要么不思考、要么全力思考）转向细粒度的分级控制。这直接催生了一个新的工程实践——**两层路由决策**。

### 两层路由：模型选择 × 算力分配

在 reasoning model 时代，每个 API 请求需要做两个决策：选哪个模型（Level 1）和分配多少推理算力（Level 2）。这两层是正交的，组合空间远大于传统的"选模型"单一决策。

当前的实践路线有三种。

**手工规则。** 按任务类型预设路由表：分类/提取等简单任务 → 快模型 + low effort；一般推理任务 → 中等模型 + medium effort；复杂规划/调试 → reasoning model + high effort。简单、可解释、但无法适应任务内部难度变化。

**基于失败信号的升级。** 默认用最低算力，遇到失败信号（低置信度、工具调用错误、自相矛盾的输出）时自动升级到更高 effort 或更强模型。Claude Code 的实践接近这个思路——主循环用默认 effort，子 agent 用 low effort。

**自动路由。** ARES（Adaptive Reasoning Effort Selection，2026）是这个方向的代表性工作：训练一个轻量路由器，在 agent 执行的每一步动态决定使用哪个 effort 级别。经过强化学习优化后，路由器将 high effort 的使用比例从 50%+ 降到 20% 以下，同时保持任务完成质量。这表明大量步骤被过度分配了推理算力。

## 当前技术格局（截至 2026-04）

**推理时计算作为独立 scaling 维度已成共识。** 从学术界到产业界，"花更多推理时间换更好结果"不再是一个需要论证的命题，而是一个需要优化的工程参数。

**控制接口已标准化。** 三大厂商都提供了 effort / reasoning_effort / thinkingBudget 参数，API 层的控制粒度已经从"开关"演进到"分级旋钮"。liteLLM 等中间件已经统一了不同厂商的参数映射。

**成本结构剧变但正在被消化。** 2024 年末 reasoning model 初登场时，单次调用成本飙升 20-50 倍的冲击引发了广泛焦虑。到 2026 年，通过 effort 分级、model routing、prompt caching（缓存 token 成本降低约 90%）等手段，工程师已经有了一套可操作的成本控制工具箱。但"最优分配策略"仍然是经验驱动的，缺乏系统化理论。

**当前的代表性价格点（2026-04）：**

| 模型 | 输入（$/M tokens） | 输出（$/M tokens） | 备注 |
|------|-------|-------|------|
| Claude Opus 4.6 | 5.00 | 25.00 | thinking tokens 可见，按输出价计费 |
| Claude Sonnet 4.6 | 3.00 | 15.00 | 推荐 medium effort 作为默认 |
| GPT-5.5 | — | — | 默认 medium effort |
| o3 | 2.00 | 8.00 | reasoning tokens 不可见 |
| o4-mini | 1.10 | 4.40 | 性价比路线 |
| Gemini 3.1 Pro | 2.00 | 12.00 | thinking tokens 单独计价 |

注意：以上是标称每 token 价格，reasoning model 的实际单次请求成本需要乘以 thinking tokens 的消耗量，实际花费通常是标称价格的数倍。

## 演进路径

- **2024 年中之前**：推理时计算不被视为独立维度。模型的推理时间近似固定（一次前向传播），成本优化集中在训练侧和部署侧（量化、蒸馏、推测解码）。
- **2024 年 8 月**：DeepMind 发表"Scaling LLM Test-Time Compute Optimally"，首次系统性论证推理时计算作为独立 scaling 维度的价值。同期多篇工作（Inference Scaling Laws 等）从不同角度验证这一结论。
- **2024 年 9 月**：OpenAI o1 发布。推理时计算从学术概念变成产品现实。但此时没有用户可调的算力控制——模型自己决定想多久。
- **2025 年**：effort 控制接口出现。Anthropic 推出 budget_tokens，OpenAI 推出 reasoning_effort，Google 推出 thinkingBudget。工程师首次获得对推理时计算的直接控制。Model routing 从概念变成必需品。
- **2026 年初**：effort 参数演化为多级分级（low / medium / high / max / xhigh）。Anthropic 的 effort 参数统一控制 thinking、text 和 tool call 三类输出。自动路由（ARES 等）开始出现。T² scaling law 将训练时和推理时计算纳入统一优化框架。
- **趋势**：推理时计算的分配将从"全局设置"走向"逐步骤自适应"——agent 在执行过程中根据每个步骤的难度动态调整 effort，而非对整个任务使用同一个级别。

## 对四层骨架的影响

### 第一层（Agent 内核）

**塌缩**：
- "一律用最强模型 + 最高算力"的默认假设不再成立——这在第二代是合理的（成本差异小），在第三代是不可持续的
- 静态的模型选择（整个 agent 用一个模型）让位给动态路由

**涌现**：
- Reasoning budget 管理成为 agent 内核的一等公民——agent loop 的每一步都需要决定"这一步值得想多久"
- Model routing 逻辑嵌入 agent 内核：简单步骤用 low effort 或快模型，关键决策步骤用 high effort 或 reasoning model
- 思考-行动的资源分配权衡：thinking tokens 消耗 context 空间，与工具结果、历史记录争夺有限的 context window

### 第二层（与世界的接口）

**塌缩**：
- 工具调用结果的复杂后处理部分被 reasoning model 吸收（模型自己能从嘈杂的工具输出中提取关键信息）

**涌现**：
- 工具调用本身的成本考量——在 reasoning model 下，每次工具调用都触发一轮 thinking，工具调用的"隐性成本"（thinking tokens）可能远超工具执行本身的成本
- 工具描述的精简成为成本优化手段——更短的工具描述 = 更少的输入 token = 更低的每次调用成本

### 第三层（多 Agent 协作）

**塌缩**：
- 用多个弱 agent 投票提升质量的动机大幅消失——reasoning model 内部已经做了类似的搜索

**涌现**：
- 成本混合编排：orchestrator 用高 effort reasoning model，worker 用 low effort 或小模型。这种异构成本结构是多 agent 在第三代的核心设计动机之一
- 并行化加速：reasoning model 单次调用慢（秒级 → 分钟级），多 agent 并行执行成为补偿延迟的手段

### 第四层（生产化）：冲击最大的一层

**塌缩**：
- 传统的延迟基线和容量规划模型（假设响应时间在秒级）完全失效
- 按请求数计费的简单成本模型不再适用

**涌现**：
- **成本工程**成为 L4 的核心新学科。需要回答的问题包括：每个功能/每个用户的推理计算成本是多少？成本预算如何分配？如何设置成本上限而不严重损害质量？
- **推理时计算的可观测性**：需要实时追踪 thinking tokens 消耗、effort 级别分布、路由命中率、缓存命中率。传统的 APM 工具不具备这些维度
- **容量规划重做**：reasoning model 的响应时间方差极大（同一个模型，简单问题秒级响应，复杂问题分钟级），传统的 P99 延迟指标和限流策略需要重新设计
- **成本失控防护**：agent 陷入推理循环时可能产生天量 thinking tokens。需要设置 per-request 和 per-session 的 token 上限
- **Reasoning trace 的可观测性困境**：OpenAI 隐藏 reasoning tokens，工程师无法直接检查模型"想了什么"——这对调试和安全审计都是障碍。Anthropic 的 thinking tokens 可见，但体积巨大，需要折叠/摘要/语义分段等工具支持

## 与其他维度的耦合

**× Reasoning（维度 1）**：本质上是同一件事的两个面。Reasoning 是"能力"，Inference-Time Compute 是"代价"。每一次 reasoning 能力的提升（更深的思考、更长的 trace），都直接转化为 inference-time compute 的增加。两个维度应该始终一起读。

**× Long Context（维度 2）**：Thinking tokens 和 context window 争夺空间。一个 128K context window 的模型，如果 thinking tokens 占了 60K，留给实际内容的空间就只剩 68K。这意味着推理时计算和上下文容量之间存在直接的零和竞争。在 agent 场景下，这个竞争尤其尖锐——长对话历史 + 大量工具结果 + 长 thinking trace，三者同时争夺有限空间。

**× Tool Use（维度 4）**：Reasoning model 改变了工具调用的成本结构。每次工具调用-结果回注循环都触发新一轮 thinking，工具调用的"真实成本"= 工具执行成本 + 触发的 thinking tokens 成本。在高 effort 模式下，后者可能远超前者。

**× Memory（维度 6）**：如果 memory 走向内化（第四代假设），memory 的检索和整合同样需要推理时计算。Reasoning + Memory 的组合意味着推理时计算的消耗可能进一步增长。第三代的推理成本问题如果未解决，会成为第四代的瓶颈。

## 关键不确定性

- **Thinking tokens 应该对用户透明还是隐藏？** OpenAI 和 Anthropic 做了截然相反的选择。隐藏有利于产品体验的简洁性和 IP 保护，但损害可调试性和安全审计能力。哪种模式会成为行业共识目前不明。
- **推理时计算有 ceiling 吗？** 当前证据显示 thinking 越长、结果越好的趋势仍在持续（单调递增），但是否存在收益归零的拐点尚无定论。如果存在 ceiling，那么超过某个 thinking budget 就是纯浪费；如果不存在，成本控制将持续是核心挑战。
- **成本下降速度能否跑赢消耗增长？** 每 token 价格每年降 10 倍，但 reasoning model 的 token 消耗也在随能力提升而增长。这两条曲线的交叉点决定了 reasoning model 在什么时候变得"对大多数应用经济可行"。
- **逐步骤自适应 effort 在工程上可行吗？** ARES 等工作展示了可能性，但在生产环境中为 agent 的每一步单独做 effort 决策的延迟和复杂度是否可接受？
- **推理时搜索策略（Best-of-N、MCTS 等）在内化式推理时代还有多少残余价值？** 随着模型内部推理能力持续提升，外部搜索策略的增量价值是否会趋近于零？

## 信息源

- **学术核心文献**：Snell et al. 2024（"Scaling LLM Test-Time Compute Optimally"，DeepMind，推理时计算作为 scaling 维度的奠基工作）；"The Art of Scaling Test-Time Compute"（2025 末，30B+ tokens 的大规模 TTS 策略对比）；T² Scaling Laws（2026-04，训练时-推理时联合优化）
- **综合论文列表**：GitHub ThreeSR/Awesome-Inference-Time-Scaling（持续更新的推理时 scaling 论文索引）
- **厂商文档**：Anthropic effort 参数文档（platform.claude.com/docs/en/build-with-claude/effort）；OpenAI reasoning models 指南（developers.openai.com/api/docs/guides/reasoning）
- **成本追踪**：Epoch AI 推理价格趋势（epoch.ai/data-insights/llm-inference-price-trends）；Artificial Analysis 模型对比（artificialanalysis.ai/models）
- **Model routing**：ARES 论文（arxiv.org/pdf/2603.07915）；Augment Code 的 model routing 指南
- **arXiv 关键词**：test-time compute scaling, inference scaling laws, reasoning effort, compute-optimal inference, process reward model

## 更新日志

- 2026-04-27：初次创建。覆盖训练时/推理时计算权衡、两大策略族、thinking tokens 经济学、effort 控制接口、两层路由、四层影响、成本悖论
