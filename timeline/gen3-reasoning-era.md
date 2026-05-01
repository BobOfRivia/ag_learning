# 第三代：Reasoning Era（2024 末–2026）

## 定义性变化

第三代的标志是**地基的第二次质变：reasoning 从 prompt 外挂转向训练内化。** 标志性事件是 OpenAI o1 的发布（2024-09）。模型不再需要工程层教它怎么想——它自己会想，会回溯，会自我纠错，会在难题上"想得更久"。

这是迄今为止塌缩-涌现最剧烈的一次转代。冲击波的烈度远超第一代→第二代的过渡：第二代主要是在 L2 层新增了能力（工具调用），原有工程实践大部分保留；第三代则直接废弃了 L1 层的一批核心技术（CoT prompting、Self-Consistency、Reflexion 等一夜贬值），同时在 L4 层催生了全新的工程学科（成本工程、推理时计算可观测性）。

除了 reasoning 内化这个核心驱动力，第三代还叠加了一个生态事件：MCP 协议的发布和爆发（2024 末–2025），把第二代碎片化的工具生态引向标准化。这两股力量——一个来自地基能力变化，一个来自生态标准化——共同塑造了当前的 agent 工程格局。

## 地基能力水位

**Reasoning：从外挂到内化。** 这是第三代的核心变量。reasoning model（OpenAI o 系列、Claude extended thinking、Gemini Deep Think、DeepSeek-R1）通过 RLVR（Reinforcement Learning from Verifiable Rewards）训练，让模型自发学会长 CoT、回溯、自我验证。模型在生成最终答案前先产生 thinking tokens——一段内部思考过程，可能包含规划、分支探索、自我纠错。到 2025 年，几乎所有前沿模型都具备了 reasoning 能力。

**Inference-Time Compute：从固定到可调。** Reasoning 内化的直接代价是推理时计算量的爆炸式增长。单次请求成本可能飙升 20-50 倍。这催生了 effort 控制接口的标准化——Anthropic 的 `effort`（low/medium/high/max）、OpenAI 的 `reasoning_effort`、Google 的 `thinkingBudget`。推理时计算从"固定成本"变成了"可调参数"，成为与训练时计算并列的独立 scaling 维度。

**Tool Use：从能力到生态。** 模型层面的 tool use 能力在第二代已基本到位。第三代的变化不在模型能力，而在生态标准化：MCP 统一了工具定义和连接协议，A2A 标准化了 agent 间通信。简单函数调用的准确率达到 99%+，tool use 的瓶颈从"模型能不能调工具"转向"工具生态怎么管理"。Computer Use（模型通过视觉操作 GUI）在 2024 末由 Anthropic 率先推出，到 2026 年初三大 AI 公司都提供了某种形式的 computer use 能力。

**Long Context：从稀缺到充裕。** 1M tokens 的 context window 成为前沿模型的标准配置。但 thinking tokens 和 context window 形成零和竞争——reasoning model 的 thinking 可能占据大量 context 空间，留给实际内容的反而变少。context 的瓶颈从"容量"转向"利用效率"。

**Memory：框架层成熟化。** 厂商原生记忆（ChatGPT Memory、Gemini 的 Google 集成、Claude 的 Projects/CLAUDE.md）都还是非参数的外部存储；独立框架（Mem0、Letta、Zep）进入生产环境；评测体系从无到有爆发（LoCoMo → MemBench → MemoryAgentBench → MemoryArena）。Memory 是第三代的活跃发展前沿，但尚未引发地基级的质变——第四代的判定标准都还没有满足。

**Instruction Following：与 reasoning 协同提升。** Reasoning model 在遵循复杂指令方面明显更强——因为它能"想一想"指令的含义再执行。但也出现了新问题：模型在 thinking 中可能"想偏"，绕过 guardrail 或重新解读用户意图。

## 工程重心与格局

第三代的工程重心经历了一次深刻的重新分配：

**L1 经历大面积塌缩后重生。** 第一代和第二代积累的大量 L1 技术（CoT prompting、Self-Consistency、Reflexion、显式 Plan-and-Execute）被 reasoning model 内化，工程层不再需要操心。但塌缩之后涌现出新的 L1 核心主题：**context engineering**（管理 agent 在长循环中看到的全部信息）取代 prompt engineering 成为最重要的工程技能；**reasoning budget 管理**和 **model routing** 成为 agent 内核的新设计维度；**最小 agent loop**（单线程 while 循环 + 扁平消息列表）被验证为最有效的架构模式，复杂的 chain/graph 编排反而证明是过度工程。

**L2 从能力建设转向生态治理。** 工具调用的基本能力已解决，焦点转向 MCP 生态管理（工具的发现、质量、权限、版本）、工具描述工程（如何写出让模型准确理解的描述）、Computer Use 基础设施。RAG 在长 context + reasoning model 的双重冲击下大面积塌缩——简单的"检索-拼接-回答"pipeline 越来越多地被直接塞进 context 解决。

**L3 的设计动机被根本重塑。** 第二代的多 agent 主要为了"提升质量"（多角色讨论模拟团队思考）。reasoning model 让这个动机消失——一个模型自己就能做到。第三代的多 agent 转向三个新动机：并行化加速（reasoning model 单次调用慢，多 agent 并行补偿延迟）、context 隔离（长 reasoning trace 需要隔离防止污染）、成本混合（高 effort orchestrator + low effort worker）。

**L4 重要性急剧上升。** 这是第三代变化最大的一层。reasoning model 带来的成本飙升催生了"成本工程"这个全新的学科。延迟从秒级跳到分钟级，传统的容量规划和 SLA 模型全部失效。reasoning trace 的可观测性（折叠/摘要/语义分段）成为新需求。新的安全面出现——模型在 thinking 中"想偏"绕过 guardrail。

## 关键事件与里程碑

**2024-09：OpenAI o1 发布。** 分水岭事件。第一个商用 reasoning model，reasoning 从 prompt 技巧转向训练内化。没有用户可调的算力控制——模型自己决定想多久。单次调用成本飙升，L1 层的 CoT prompting、Self-Consistency 等技术立即开始贬值。

**2024-10：Anthropic 推出 Computer Use beta。** Tool use 的概念从 API 调用扩展到 GUI 操作。模型通过视觉感知屏幕、生成鼠标键盘操作。这是 Tool Use × Multimodality 的交叉产物，开辟了全新的 L2 子领域。

**2024-11：Anthropic 发布 MCP。** 生态事件。统一的工具连接协议，解决了第二代每框架一套工具格式的碎片化问题。

**2024-12："Building Effective Agents"（Anthropic）。** 提出 workflow vs agent 的核心区分和六种可组合构建块（Prompt Chaining、Routing、Parallelization、Orchestrator-Workers、Evaluator-Optimizer、Autonomous Agent）。这篇文章成为第三代 agent 架构设计的事实标准参考。

**2025-01：DeepSeek-R1 发布。** 开源 reasoning model 标杆。证明了 reasoning 内化不是闭源模型的专利，RLVR 训练方法可复现。

**2025 上半年：reasoning model 全面普及。** Claude extended thinking、Gemini Deep Think 相继推出。几乎所有前沿模型都具备 reasoning 能力。effort 控制接口（budget_tokens / reasoning_effort / thinkingBudget）开始出现，工程师首次获得对推理时计算的直接控制。

**2025-03：OpenAI Agents SDK 发布。** 极简架构（Agent、Handoff、Session、Tracing），设计哲学是"closer to the metal"。标志着框架生态从"过度抽象"向"轻量原语"的转向。

**2025-04：Google 发布 A2A 协议。** 标准化 agent-to-agent 通信，与 MCP（agent-to-tool）互补。MCP + A2A 构成完整的 agent 互操作层。

**2025 年中：Context engineering 概念确立。** 取代 prompt engineering 成为 agent 工程的核心概念。Manus 团队的实践文章、多个学术框架（ACE 等）同期出现。核心洞察：agent 长循环中，"决定什么进入 context window"比"在 context window 内怎么写 prompt"更重要。

**2025-12：MCP 捐赠给 Linux Foundation / AAIF。** OpenAI、Google、Microsoft、AWS 共同参与治理。工具生态标准化从单家推动变为行业共治。

**2025-12：Claude Code、Codex CLI 等 coding agent 成为开发者日常工具。** Coding agent 验证了最小 agent loop 架构——单线程主循环 + 严格的工具设计 + 良好的 context 管理 = 可控的自主性。

**2026 年初：effort 参数演化为多级分级。** Anthropic 的 effort 统一控制 thinking、text 和 tool call 三类输出。自动路由（ARES 等）开始出现。推理时计算的分配从"全有或全无"走向精细化。

**2026 年初：Memory 评测体系爆发。** MemBench、MemoryAgentBench、MemoryArena 相继发布。MemoryArena 暴露了"被动回忆 vs 主动运用"的巨大鸿沟（80%→40-60%），标志着 memory 评测从玩具走向真实场景。

## 塌缩与涌现

### 塌缩

**L1 层塌缩最为剧烈——这是第三代最显著的特征。**

- **CoT prompting 大幅贬值。** "Let's think step by step" 对 reasoning model 不仅无效，某些情况下反而有害。对非 reasoning model 仍有效，但 L1 的工程设计不再以 CoT 为核心前提。
- **Self-Consistency 边际收益骤降。** Reasoning model 内部已经在做类似的搜索和验证，外部多次采样的增量价值很小。
- **Reflexion / Self-Refine 大幅简化。** 外部反思循环的大部分价值被 reasoning model 内化。残余价值仅限于需要引入模型自身不具备的外部评估标准的场景。
- **经典 ReAct 的紧密交错被双相位取代。** 模型从"想一步做一步"变成"想完再做"——先在 extended thinking 中完成推理，再在 output 中生成工具调用。
- **Plan-and-Execute 的 planning 阶段部分塌缩。** Reasoning model 可以在一次 thinking 中完成原本需要单独 planning 步骤的分解和排序。
- **"Researcher + Critic + Writer" 类纯认知性多角色编排。** 单个 reasoning model 已足够强，用多 agent 模拟团队讨论来提升质量的性价比骤降。

**L2 层部分塌缩：**

- **ChatGPT Plugins 及封闭工具市场模式。** 被 MCP 的开放协议路线替代。
- **每框架一套的工具适配格式。** MCP 统一了碎片化格局。
- **简单 RAG pipeline。** 长 context + reasoning model 让很多"检索-拼接-回答"场景可以直接塞进 context 解决。复杂多跳 RAG 的编排也被简化——reasoning model 自己决定何时再查。
- **Query rewriting 等 RAG 中间环节。** 被 reasoning model 吸收。

**L4 层局部塌缩：**

- **传统的延迟基线和容量规划模型。** 假设响应时间在秒级的所有设计全部失效。
- **按请求数计费的简单成本模型。** 不再适用。
- **显式 CoT 评估方式。** reasoning trace 可能被隐藏，无法直接评估。

### 涌现

**L1 层的新核心主题：**

- **Context engineering。** 从"写好一个 prompt"升级为"设计一个动态系统管理 agent 在每一轮看到的全部信息"。关键实践包括：KV-cache 优化（缓存 vs 未缓存 token 成本差 10 倍）、文件系统作为外部记忆、注意力操纵（应对 Lost-in-the-Middle）、错误保留、受控多样性。
- **Reasoning budget 管理。** 模型在每个请求上花多少推理算力成为可调参数，agent 内核需要决定每一步"值得想多久"。
- **两层路由决策。** 选模型（Level 1）× 选 effort 级别（Level 2），组合空间远大于传统的单一模型选择。
- **最小 agent loop 验证为最佳实践。** 单线程 while 循环 + 扁平消息列表，复杂性放在 context 管理而非循环结构。
- **Sub-agent 派发与隔离。** 独立子问题交给有限能力的子 agent，结果回收到主循环。

**L2 层的新主题：**

- **MCP 生态管理。** 工具的标准化发现、连接、质量评估、权限控制。17,000+ MCP server 的治理问题。
- **工具描述工程。** 从简短 schema 描述转向更长的"教学式"描述，工具描述的质量直接影响调用准确率。
- **Computer Use 基础设施。** 截图管道、坐标映射、操作确认、失败恢复——全新的 L2 子领域。
- **A2A 协议。** 标准化 agent 间暴露能力和协作。

**L3 层动机重塑：**

- **并行化加速。** Reasoning model 单次调用慢（秒级→分钟级），多 agent 并行执行补偿延迟。
- **Context 隔离。** 长 reasoning trace 需要隔离，防止污染主循环的 context。
- **成本混合编排。** 高 effort orchestrator + low effort worker 的异构成本结构。

**L4 层大量涌现——这是第三代被低估最严重的变化：**

- **成本工程。** 全新的学科。Token 消耗追踪、per-feature 成本归因、成本上限设置、effort 分布优化。
- **推理时计算的可观测性。** Thinking tokens 消耗、effort 级别分布、路由命中率、缓存命中率——传统 APM 工具不具备这些维度。
- **容量规划重做。** Reasoning model 响应时间方差极大，传统 P99 和限流策略需要重新设计。
- **成本失控防护。** Agent 陷入推理循环时可能产生天量 thinking tokens，需要 per-request 和 per-session 的 token 上限。
- **Reasoning trace 可观测性。** 折叠/摘要/语义分段等工具需求。OpenAI 隐藏 trace 对调试和安全审计构成障碍。
- **新的安全面。** 模型在 thinking 中"想偏"绕过 guardrail；prompt injection 通过工具结果注入恶意指令；Computer Use 的权限模型尚无成熟方案。
- **Process eval。** 从"答对了吗"到"思考路径合理吗"——需要评估推理过程的质量，不只是最终结果。
- **Memory 评测。** 从 LoCoMo 到 MemoryArena，agent 记忆的评测体系在这个时期从无到有建立。

## 代表性技术与架构

**Reasoning Model 家族。** OpenAI o 系列（o1 → o3）、Claude extended thinking、Gemini Deep Think、DeepSeek-R1 系列。共同特征：通过 RLVR 训练内化推理能力，生成 thinking tokens 作为内部思考过程。

**最小 Agent Loop。** `while(response 包含 tool_call) → 执行 → 追加结果 → 再调用`。Claude Code 是这种模式的标杆实现。

**Context Engineering 实践。** KV-cache 优化、文件系统卸载、注意力操纵、错误保留。Manus 团队的实践和 Claude Code 的 CLAUDE.md + 压缩器是两个被广泛引用的案例。

**MCP + A2A。** 工具层（MCP）+ Agent 通信层（A2A）的双协议互操作架构。

**Effort 控制 + Model Routing。** 两层路由决策：手工规则 / 基于失败信号升级 / 自动路由（ARES）。

## 遗产与局限

第三代正在进行中，但已经可以看到一些清晰的模式。

**遗产**：reasoning 内化证明了一个重要的规律——**当模型地基能力足够强时，工程层的复杂技巧会被吸收。** 第一代和第二代花大量工程精力做的 reasoning 外挂（CoT、Self-Consistency、Reflexion），在 reasoning model 出现后几乎一夜贬值。这暗示着当前 L2 层的 memory 外挂和 RAG 工程，也可能在未来某次地基质变后遭遇类似的命运（这正是第四代假设的核心）。context engineering 作为一门新的工程学科，可能是第三代最持久的遗产——即使 reasoning 和 memory 继续内化，agent 的 context 管理问题不太可能完全消失。

**当前局限**：

推理成本仍然很高。尽管每 token 价格以每年约 10 倍的速度下降，单次请求的总 token 消耗同步增长，导致实际花费不一定下降。这限制了 reasoning model 在成本敏感场景的应用。

Memory 仍然是工程外挂。所有生产级记忆系统都是非参数的外部存储，没有任何一家实现了真正的参数级记忆内化。选择性遗忘是所有系统的共同短板。"被动回忆 vs 主动运用"的鸿沟（MemoryArena 暴露的 80%→40-60% 骤降）尚未被弥合。

L3（多 Agent 协作）的层级文件已创建（[layers/L3-multi-agent.md](../layers/L3-multi-agent.md)），覆盖了单 agent vs 多 agent 决策框架、五种编排模式、四种交接模式、第三代动机转变、框架选型。L4（生产化）的层级文件已创建（[layers/L4-production.md](../layers/L4-production.md)），覆盖了评测、可观测性、成本工程、安全、人机协同、可靠性六个子领域。

Computer Use 可用但不可靠。WebArena 上最优表现约 71.2%，多数生产系统在 50-60% 区间。离可信赖的自主操作还有距离。

## 与相邻代际的关系

**← 承接第二代**：第二代建立的 tool use 基础能力被完整保留并进一步发展（生态标准化）。但第二代的 RAG 工程在长 context + reasoning model 的冲击下大面积收缩。第二代的 agent 框架生态（LangChain 等）经历了从"过度抽象"到"轻量原语"的反思和调整。

**→ 第四代的过渡信号**：如果 memory 从工程外挂向地基内化，将标志第四代的开始。截至 2026-04，三个判定标准（API 层原生持久记忆、外部向量库使用下降、memory eval 广泛采用）都还没有明确满足。但一些早期信号值得关注：memory 评测体系的快速成熟（MemoryArena 等）在为 memory 能力的评判建立基线；图记忆从实验进入生产（Graphiti + Neo4j）表明 memory 工程的复杂度在上升；LinkedIn CMA 等企业级实践开始把 memory 视为独立的基础设施层而非附属功能。第三代的 reasoning 成本问题如果未解决，会拖慢第四代——因为 memory 检索/整合同样消耗推理时计算。

## 更新日志

- 2026-05-01：更新"当前局限"章节，反映 L3、L4 文件已创建的事实
- 2026-04-27：初次创建
