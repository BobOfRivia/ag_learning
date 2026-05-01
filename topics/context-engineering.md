# Context Engineering

> 锚点：层级 L1（Agent 内核）/ 与维度 2 Long Context、维度 6 Memory、维度 7 Inference-Time Compute 均有直接耦合
>
> 实践案例补充：[topics/context-engineering-in-practice.md](context-engineering-in-practice.md) — 以 Claude Code 为活体 agent，从第一人称视角映射本文理论框架的实际运作

## 这个概念是什么

Context engineering 是为 LLM 的每一次推理调用策划和维护最优信息集合的工程学科。Andrej Karpathy 的定义被广泛引用："the delicate art and science of filling the context window with just the right information for the next step." Cognition AI（Devin 的开发团队）更直接地说："Context engineering is effectively the #1 job of engineers building AI agents."

它和 prompt engineering 的区别不是程度上的，而是范畴上的。Prompt engineering 关注的是"怎么写好一段指令"——措辞、结构、few-shot 示例的选取。Context engineering 关注的是"模型在每一步看到的全部信息如何设计"——包括 system prompt，但也包括对话历史、工具定义、工具调用结果、检索到的文档、agent 的自我笔记、压缩后的摘要。用一个类比：prompt engineering 是在 context window 内做什么，context engineering 是决定什么进入 context window。或者更直觉地说：prompt engineering 是怎么问问题，context engineering 是确保模型在开始思考之前已经有了正确的教科书、计算器和上次对话的笔记。

这个概念在 2025 年中期从一个实践观察上升为工程共识。背后的驱动力是 agent 的兴起——当 LLM 从单轮问答变成几十甚至上百轮的持续任务执行时，context window 变成了一个需要动态管理的稀缺资源。每一轮工具调用的结果、每一段 reasoning trace、每一次错误的堆栈信息都在往 context 里追加内容。不加管理，agent 在执行到第 30 步时，context 已被早期的无关信息淹没。Gartner 在 2025 年的判断是："context engineering is in, and prompt engineering is out."

## 内部结构

### 四种核心操作

LangChain 在 2025 年提出了一个被广泛采纳的分类框架，将 context engineering 的操作分为四类。这四类不是互斥的阶段，而是可以在 agent loop 的每一步组合使用的正交操作。

**Write（写出）：把信息保存到 context window 之外。** Agent 在执行过程中主动将信息持久化到外部存储，以便后续检索。具体形态包括 scratchpad（执行中的临时笔记，如 Manus 的 todo.md）、长期记忆（跨 session 的持久化，如 Claude Code 的 CLAUDE.md 和 memory 系统）、以及结构化状态对象。记忆可以进一步分为三种类型：episodic（过去的具体经历和示例）、procedural（操作规程和指令）、semantic（事实和知识）。ChatGPT、Cursor、Windsurf 等产品都已实现从用户交互中自动生成记忆的能力。

**Select（选入）：把相关信息拉回 context window。** 与 Write 互补——Write 是卸载，Select 是按需加载。关键实践是 just-in-time 检索：不预先加载所有数据，而是在 context 中保留轻量标识符（文件路径、URL、查询引用），在需要时动态检索。Claude Code 的做法是混合模式——CLAUDE.md 在启动时加载（前置），但代码文件通过 glob/grep 按需检索（即时）。对工具描述也是同理：当可用工具达到数百个时，不把所有工具定义塞进 context，而是用 RAG 动态选择与当前步骤相关的工具子集。

**Compress（压缩）：只保留必要的 token。** 这是四种操作中技术上最复杂的一类，下文单独展开。核心问题是：如何在减少 token 数量的同时保留 agent 继续执行任务所需的关键信息。

**Isolate（隔离）：将 context 分布到独立的处理空间。** 通过 sub-agent 架构实现——每个子 agent 维护自己的 context window，专注于特定子任务，完成后只向主 agent 返回浓缩的结果（通常几万 token 的探索压缩为 1,000-2,000 token 的摘要）。隔离的价值不只是节省 token，更关键的是防止 context 污染——一个子任务的中间状态不会干扰其他子任务的推理。代码 agent 的沙箱环境也是一种隔离：token 密集的运行时对象留在沙箱内，只有必要的输出进入 LLM 的 context。

### 压缩策略：最技术密集的子领域

压缩是 context engineering 中研究最活跃、方案最多样的领域。2025-2026 年出现了多种路径，各有适用场景和局限。

**锚定式迭代摘要（Anchored Iterative Summarization）。** 这是目前评测表现最好的方案。核心思路是维护一个持久化的结构化摘要（"锚"），每次需要压缩时不重新生成完整摘要，而是只对新增的被驱逐消息段做摘要，然后合并到锚中。锚的结构通常按任务意图、已做的修改、已做的决策、下一步计划等维度组织。Factory.ai 在 36,000 条真实工程会话消息上的评测显示，这种方法在准确性上得分 4.04（满分 5），显著优于 Anthropic 的 3.74 和 OpenAI 的 3.43——优势主要体现在技术细节的保留（文件路径、错误码、具体决策的理由）。关键洞察是：**结构化的分类格式迫使摘要器显式处理或显式忽略每个类别，防止自由文本摘要中常见的信息静默丢失。**

**观察遮蔽（Observation Masking）。** JetBrains Research 在 2025 年末提出的方法，思路更简单：保留 agent 的推理和行动记录，但用占位符替换较早的环境观察结果（如测试日志、命令输出）。保持最近 N 轮（研究发现 10 轮最优）的完整交互，更早的只保留决策骨架。在 SWE-bench Verified 上的测试表明，观察遮蔽在 5 组设置中的 4 组优于 LLM 摘要，同时成本降低 52%。一个重要发现是：LLM 摘要反而导致 agent 运行时间延长 13-15%——摘要可能遮蔽了停止信号，让 agent 在应该放弃时继续尝试。

**Provider 原生压缩。** Anthropic 在 2026 年初推出了 `compact-2026-01-12` API，提供开箱即用的自动压缩。在配置的 token 阈值处触发，生成压缩块插入对话中，支持检查和回放。跨 Claude API、AWS Bedrock、Google Vertex AI、Microsoft Foundry 可用。Google 的 ADK（Agent Development Kit）也内置了 compaction 原语。这标志着压缩从"每个框架自建"走向"基础设施标准化"。

**ACON（Failure-Driven Guideline Optimization）。** 2025 年 10 月的研究工作，将压缩视为优化问题：分析成对的执行轨迹（完整 context 成功 vs 压缩 context 失败），用 LLM 识别失败是因为丢失了什么信息，然后迭代优化压缩 prompt。无需微调，兼容任何 API 可访问的模型。在 AppWorld、OfficeBench 等 benchmark 上实现 26-54% 的峰值 token 减少，同时保持 95%+ 的任务准确率。

**嵌入式压缩（Embedding-Based Compression）。** 将历史交互存为稠密嵌入向量，每一轮根据语义相关性重建必要的片段。token 缩减可达 80-90%，但对需要精确细节（逐字错误信息、具体文件路径）的场景有精度损失。

**生产实践的压缩比参考。** 针对不同内容类型，业界逐步形成了经验性的压缩比指导：对话历史（旧轮次）3:1 到 5:1；工具输出/观察结果 10:1 到 20:1；最近 5-7 轮不压缩；system prompt 不压缩。触发阈值的共识是在 context 利用率超过 70% 时开始压缩——研究表明性能退化在超过约 30,000 token 后加速，即使模型的 context window 远大于此。

### KV-Cache 优化：生产环境的第一优先级

KV-cache 命中率是生产 agent 系统最重要的单一性能指标——缓存 token 和未缓存 token 的成本差约 10 倍（以 Anthropic 为例，cached input 价格是 uncached 的 10%），延迟差异同样显著。这使得 KV-cache 优化成为 context engineering 中投资回报率最高的实践。

核心原则只有一条：**保持 context 前缀稳定。** 由于 LLM 的自回归特性，即使一个 token 的差异也会使该位置之后的全部缓存失效。具体做法包括：

不在 system prompt 中放入时间戳或随机 ID——一个字符的差异就会使整个下游缓存失效。已经发生的 action 和 observation 只追加不修改——不要事后编辑历史消息。确保 JSON 序列化的确定性——字段顺序、空格、换行都要一致，否则语义相同但字节不同的内容会导致缓存未命中。工具定义的顺序和内容保持稳定——不要在每轮动态增删工具列表。

Manus 团队的实践验证了这些原则的价值：他们将 KV-cache 命中率作为最优先优化的指标，所有 context 管理决策都以"是否影响缓存前缀稳定性"为首要考量。

在基础设施层面，KV-cache 本身也在快速演进。2026 年的进展包括：Google Research 的 TurboQuant 将 KV-cache 压缩至 3.5-bit，缩减 6 倍且几乎无精度损失，无需重新训练；ICMSP 在 CES 2026 发布的 4 层存储层级将 NVMe 纳入 KV-cache 地址空间，使缓存可以跨推理持久化；SideQuest（2026 年 2 月）专门为 agentic 工作负载设计的 KV-cache 管理方案，支持多步推理场景的缓存复用。

### 注意力管理：Lost in the Middle 问题

2023 年 Stanford 的 "Lost in the Middle" 研究揭示了一个影响深远的发现：LLM 对 context window 中信息的关注度分布不均匀——开头和结尾的信息被充分利用，中间的信息被显著忽视。后续研究确认，这个问题在 128K+ context window 的模型上依然存在，且准确率在关键信息位于中间时可下降 30% 以上。

根本原因与位置编码有关。当前主流模型使用的 RoPE（Rotary Position Embedding）引入了距离衰减效应，使得远离序列开头和结尾的 token 落入低注意力区。

对 agent 系统而言，这意味着 context 的组织方式和内容同样重要。当前的应对策略包括：让 agent 周期性更新任务清单，把当前目标重新推到 context 末尾（进入高注意力区）；在 system prompt 和运行时消息中对关键信息做冗余放置；对长工具输出做摘要而非原文拼接；把最重要的指令放在 system prompt 的最开头和最后一条用户消息的最后。

### 实际系统的 Context Engineering 实践

**Claude Code** 的方案是"最小循环 + 多层 context 机制"的组合。核心循环是单线程 while 循环，context 管理依赖多个协作机制：CLAUDE.md 在启动时加载（提供持久化的项目知识和行为指令）；代码文件通过 glob/grep 按需检索；context 接近上限（约 95%）时触发自动压缩器；sub-agent 提供 context 隔离；TodoWrite 机制将任务清单写入 context 起到注意力锚定作用。Martin Fowler 网站的分析将 Claude Code 的 context 接口分为六类：CLAUDE.md（始终加载的指导）、Rules（路径相关的自动加载规则）、Skills（懒加载的领域知识）、Sub-agents（隔离的并行 context）、MCP Servers（结构化工具/数据访问）、Hooks（生命周期事件触发的确定性脚本）。

**Manus** 的方案强调 KV-cache 优先和文件系统卸载。核心原则是"context 只追加不修改"以最大化缓存命中；信息超过即时需要时卸载到文件系统，context 中只保留路径引用；todo.md 作为持续更新的任务锚，同时服务于 context 管理和用户可见性。Manus 将 KV-cache 命中率视为最重要的单一指标。

**Anthropic 的长时程任务建议**提供了一个按任务类型选择策略的决策框架：大量来回对话 → 压缩（compaction）；有明确里程碑的迭代开发 → 结构化笔记（note-taking）；需要并行探索的复杂调研 → 多 agent 隔离（multi-agent）。三种策略不互斥，复杂任务通常组合使用。

## 当前状态（截至 2026-04）

Context engineering 作为独立学科已获得广泛承认。行业共识是它对 agent 成功的影响超过模型选择——同一个模型在不同的 context 管理策略下，表现差距可以非常大。

**压缩方案正在标准化。** Provider 原生压缩 API（Anthropic compact、Google ADK compaction）的出现标志着从"框架自建"向"基础设施服务"的转变。但最优压缩策略仍然高度依赖任务类型，缺乏通用方案。

**评测方法从文本相似度转向功能性评估。** Factory.ai 的 probe-based 评测框架代表了这个方向：不用 ROUGE 等传统指标衡量摘要的文本保真度，而是测试压缩后 agent 能否回答关于事实回忆、文件追踪、任务续接、决策推理的探测问题。这种评测方式更贴合 agent 的实际需求。

**Context drift 被识别为主要失败模式。** 有分析指出，2025 年企业 AI 失败中约 65% 可归因于 context drift 或 memory loss，而非 context window 的物理耗尽。这意味着"有多大的 context window"远不如"context 里的信息质量多高"重要。

**简单方案常常优于复杂方案。** JetBrains 的研究表明，观察遮蔽在多数场景下优于 LLM 摘要。Anthropic 的建议是"do the simplest thing that works"。这与整个 agent 工程领域的一个更大趋势一致——最成功的系统往往是最简洁的（Claude Code 的单线程循环、最小 agent loop 等）。

**关键数据点。**
- Context rot 在超过约 30,000 token 后加速，即使模型标称 context window 为 128K-2M
- 缓存 token 成本约为非缓存的 10%（Anthropic 定价）
- 在 95% per-step 可靠性下，20 步的组合成功率只有 36%——context 管理的每一步误差都在累积
- 锚定式迭代摘要在准确性上领先全量重建约 8%（Factory 评测）

## 关键权衡

**压缩比 vs 信息保留。** 更激进的压缩节省 token 和成本，但增加信息丢失风险。OpenAI 的方案压缩率达 99.3% 但功能性评分最低，Factory 的方案多保留 0.7% 的 token 但在任务续接能力上明显领先。真正的指标不是 tokens-per-request（每次请求多少 token），而是 tokens-per-completed-task（完成一个任务总共要多少 token）——过度压缩导致的信息丢失会让 agent 重做工作，反而增加总消耗。

**前置加载 vs 即时检索。** 把所有可能需要的信息预先放入 context，简单但浪费空间且可能导致 context distraction。按需检索节省空间但引入延迟和不确定性（可能检索不到需要的信息）。当前最佳实践是混合模式——高频使用且体积小的信息前置加载，低频使用或体积大的信息即时检索。

**缓存友好性 vs 信息最新性。** 保持 context 前缀稳定以最大化 KV-cache 命中，与随时更新 context 以反映最新状态之间存在张力。修改 context 中任何已有内容都会使下游缓存失效。这就是为什么"只追加不修改"成为核心原则——但当旧信息确实过时或有误时，这个原则需要和信息质量做权衡。

**Thinking tokens 与内容空间的零和竞争。** Reasoning model 的 thinking tokens 和实际内容争夺 context window。一个 128K window 的模型，如果 thinking 占了 60K，实际可用空间只剩 68K。在高 effort 模式下，这个竞争更加尖锐。Context engineering 需要和 reasoning budget 管理协同设计。

**保留错误 vs 清理 context。** 保留失败的工具调用和错误信息，让模型从失败中学习，避免重蹈覆辙。但这些信息也占据 context 空间且可能引起 context confusion（模型误读旧错误为当前状态）。当前的共识是保留——实证表明清除错误后 agent 更容易重复同样的错误。

## 信息源

- [Effective Context Engineering for AI Agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic 官方的 context engineering 指南，覆盖压缩、笔记、sub-agent 三种长时程策略
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — Manus 团队的实战经验，KV-cache 优先和文件系统卸载的一手来源
- [Context Engineering for Coding Agents — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) — 从 coding agent 角度梳理 context 接口的分类和设计选择
- [Context Engineering for Agents — LangChain](https://www.langchain.com/blog/context-engineering-for-agents) — 提出 Write / Select / Compress / Isolate 四操作框架
- [Deep Dive into Context Engineering for AI Agents — Towards Data Science](https://towardsdatascience.com/deep-dive-into-context-engineering-for-ai-agents/) — 技术深度文章
- [Evaluating Context Compression for AI Agents — Factory.ai](https://factory.ai/news/evaluating-compression) — 36,000 条消息的压缩方案对比评测，probe-based 评估方法
- [Cutting Through the Noise: Smarter Context Management — JetBrains Research](https://blog.jetbrains.com/research/2025/12/efficient-context-management/) — 观察遮蔽 vs LLM 摘要的实证对比
- [AI Agent Context Compression Strategies — Zylos Research](https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies) — 压缩策略全景综述，含 CDR 框架和生产指导
- [Context Engineering: From Prompts to Corporate Multi-Agent Architecture](https://arxiv.org/pdf/2603.09619) — 学术视角的 context engineering 框架化
- [Context Engineering — Weaviate](https://weaviate.io/blog/context-engineering) — 从检索和记忆角度的分析
- [What Is Context Engineering — Neo4j](https://neo4j.com/blog/agentic-ai/what-is-context-engineering/) — 知识图谱视角的 context engineering
- [Context Engineering — Gartner](https://www.gartner.com/en/articles/context-engineering) — Gartner 的行业定义和趋势判断
- [SideQuest: Model-Driven KV Cache Management for Long-Horizon Agentic Reasoning](https://arxiv.org/html/2602.22603v1) — Agent 专用的 KV-cache 管理方案
- [KV Cache Optimization Strategies for Scalable and Efficient LLM Inference（ACL 2026 Survey）](https://arxiv.org/html/2603.20397v1) — KV-cache 优化的系统综述

## 更新日志

- 2026-05-01：增加实践案例补充文件 context-engineering-in-practice.md 的交叉引用
- 2026-04-27：初次创建。覆盖四种核心操作、压缩策略深度对比、KV-cache 优化、注意力管理、系统实践对比、关键权衡
