# 第 3 层：多 Agent 协作（Multi-Agent Coordination）

## 这一层解决什么问题

当单个 agent 的能力边界不足以完成一个任务时，需要多个 agent 分工协作。L3 的工程职责是：决定何时需要拆分、如何分工、如何通信、如何汇聚结果。

但这一层有一个和其他层不同的特征：**默认答案是"不需要"。** L1-L2-L4 的工程问题是你一旦做 agent 就必须面对的；L3 是你在确认单 agent 不够用之后才进入的。大量生产经验表明，过早引入多 agent 是一个常见的过度工程陷阱——增加了复杂度、成本和故障面，却没有带来对等的收益。

第三代 reasoning model 让这个判断变得更加偏向"不需要"。一个配合良好 context engineering 的单 agent 能处理的任务范围远超第二代。多 agent 的设计动机从"提升质量"（多角色讨论模拟团队思考）转向了"提升效率和隔离性"（并行化、context 隔离、成本混合）——前者已被 reasoning model 内化，后者仍然是多 agent 的真实价值所在。

## 核心决策：单 Agent vs 多 Agent

这是进入 L3 之前的前置判断，也是这一层最重要的工程决策。

**任务结构是决定性变量。** 一个任务是否适合多 agent，取决于它的内部结构是否存在真正独立的分支。Augment Code 的生产数据表明：在可并行化的任务上（如独立文件的代码修改），多 agent 带来最高 81% 的效率提升；在严格顺序依赖的任务上（如需要前一步结果才能执行下一步），多 agent 导致最高 70% 的性能退化（Google Research 数据）。任务结构不是一个光谱——它是一个阶梯：要么存在可并行的独立分支，要么不存在。

**Token 经济学是硬约束。** 多 agent 的 token 开销不是线性增长。Anthropic 的生产数据显示，多 agent 系统的输入 token 是单 agent 的 4-220 倍（取决于 agent 数量和通信频率），整体 token 消耗约为单 agent 的 15 倍。这个倍数来自两个源头：每个 agent 需要自己的 system prompt 和 context 副本（复制成本）；agent 之间的通信本身消耗 token（协调成本）。在 reasoning model 时代，这个倍数的经济后果更加严峻——15 倍的 token 消耗乘以 reasoning model 3-8 倍的单价，总成本可能是单 agent 的 45-120 倍。

**反向案例值得重视。** Microsoft Azure 的 agent 工程团队在 2025 年经历了一次有启示意义的反转：从专家 agent 集群（每个工具一个专家 agent）回退到通才 agent（单 agent 持有所有工具）。原因是专家 agent 之间的路由和通信开销超过了专业化带来的收益。类似地，多个独立团队发现，reasoning model 时代"一个好 agent + 好 context engineering"的上限远高于第二代——很多原本需要拆分的任务现在单 agent 就能完成。

**决策规则。** 综合生产经验，可以提炼一条启发式规则：**默认单 agent。只有当以下两个条件同时满足时，才考虑多 agent：（1）任务中存在真正独立的分支，且这些分支的并行执行能带来可衡量的收益；（2）token 预算可以承受 15 倍以上的成本增长。** 如果只满足其中一个条件——比如有独立分支但预算有限，或者预算充裕但任务是严格顺序的——单 agent 仍然是更好的选择。

这个规则的一个推论是：多 agent 的适用场景比直觉预期的要窄得多。大多数看起来需要多 agent 的场景（"让一个 agent 做研究，一个做写作，一个做审校"），在 reasoning model 时代都可以用单 agent + 好的 context engineering 解决。真正需要多 agent 的场景通常涉及：物理并行（同时操作多个独立系统）、权限隔离（不同 agent 持有不同权限）、或极长任务中的 context 隔离（防止 context 膨胀导致质量下降）。

## 关键工程模式

### 编排模式（Orchestration Patterns）

编排模式决定多个 agent 之间的控制流和数据流如何组织。Microsoft Azure Architecture Center 在 2025 年梳理了五种生产验证的模式，这个分类被广泛采纳。

**Sequential（流水线）。** Agent A 完成后将结果传递给 Agent B，Agent B 完成后传递给 Agent C。最简单的多 agent 模式，适用于任务的各阶段有明确的顺序依赖且每个阶段需要不同专长的场景。优势是控制流清晰、调试简单；劣势是延迟是各 agent 延迟之和，且没有并行收益。在 reasoning model 时代，这个模式的价值进一步收窄——如果阶段之间不需要不同的权限或工具集，单 agent 通常可以串行完成同样的工作。

**Concurrent（扇出-汇聚）。** 把独立的子任务同时分发给多个 agent，各自独立执行，结果汇聚到一个聚合步骤。这是多 agent 价值最明确的模式——它直接利用了任务的并行结构来缩短总延迟。适用条件：子任务之间无数据依赖、结果可以独立生成后合并。Claude Code 的子 agent 派发（同时探索多个文件或多个方案）就是这个模式的实例。关键工程约束：需要设计聚合策略（简单拼接、投票、或由一个 agent 做最终综合）和超时策略（一个 agent 卡住不应阻塞整体）。

**Group Chat（多 agent 讨论）。** 多个 agent 在一个共享的对话空间中轮流发言，一个管理者 agent 决定谁下一个发言。支持 maker-checker 模式（一个 agent 生成、另一个验证）。这个模式在第二代很流行——"让不同角色的 agent 讨论得出更好结论"。在第三代，其核心价值缩窄到需要对抗性验证的场景（如代码生成 + 代码审查，方案设计 + 安全审计），纯认知性的"讨论提升质量"动机已大幅消失。

**Handoff（控制权转交）。** 一个 agent 在执行过程中判断当前步骤需要另一个 agent 的专长，将控制权转交给它。被转交的 agent 完成任务后可以转交回来或转交给第三个 agent。OpenAI Agents SDK 将 handoff 作为核心原语——handoff 本质上是一种轻量的动态路由。适用于对话类场景（客服 agent 根据问题类型转交给专家 agent）和需要不同权限的场景（普通 agent 遇到需要高权限操作时转交给有权限的 agent）。

**Magentic（动态自适应编排）。** 没有预定义的固定流程，一个元编排器根据当前状态和任务需求动态决定调用哪些 agent、以什么顺序。这是灵活性最高但不确定性也最大的模式。Microsoft AutoGen（现已并入 Microsoft Agent Framework）的 Magentic-One 系统是这个模式的代表。适用于需求不可预先分解的开放式任务。当前状态：有学术和框架验证，生产部署有限，不确定性管理是主要挑战。

### 编排 vs 编舞（Orchestration vs Choreography）

上述五种模式都属于编排（orchestration）——有一个中央控制点（orchestrator）决定谁做什么。另一种架构是编舞（choreography）：没有中央控制点，每个 agent 根据接收到的事件自主决定是否行动和如何行动。

编排的优势是可控性和可观测性——所有决策经过一个点，容易追踪和干预。劣势是 orchestrator 是单点瓶颈和单点故障。编舞的优势是去中心化、无单点故障、更易水平扩展。劣势是全局行为难以预测和调试。

当前生产实践几乎一边倒地选择编排。原因是 agent 系统的可调试性和可控性在当前阶段比扩展性更重要——agent 的行为已经足够不确定，再引入去中心化的不可预测性会让调试变得极其困难。编舞模式目前更多出现在学术研究和特定的事件驱动架构中。

### 交接模式（Handoff Patterns）

交接模式关注的是 agent 之间的控制权如何转移。这和编排模式是正交的——一个 Sequential 编排中的两个 agent 也需要交接。

**Sequential Handoff（顺序交接）。** Agent A 完成后把完整的执行上下文传递给 Agent B。最简单的模式，控制流清晰。缺点是 context 传递的粒度难以控制——传太多浪费 token，传太少丢失关键信息。

**Hierarchical Routing（层级路由）。** 一个路由 agent 接收任务，根据任务类型分发给专家 agent。约 60% 的生产多 agent 部署采用这种模式。它的核心优势是路由决策集中化——系统的行为取决于路由 agent 的分类准确率，这是一个可以独立优化和评测的变量。L1 中讨论的 Orchestrator-Workers 模式在工程上通常实现为层级路由。

**Parallel Delegation（并行委派）。** 一个 agent 同时委派多个子 agent 执行独立的子任务。和 Concurrent 编排模式紧密对应。关键工程问题是结果汇聚策略和超时处理。

**Event-Driven（事件驱动）。** Agent 通过事件总线或消息队列通信，不直接感知彼此的存在。每个 agent 订阅自己关心的事件类型，处理后可能发布新事件。这种模式的耦合度最低，最适合需要动态增减 agent 的场景和跨系统集成。A2A 协议的 Task 原语可以视为这种模式的标准化尝试。

### 第三代的动机转变

理解当前多 agent 工程的关键，是理解第三代 reasoning model 如何重塑了多 agent 的设计动机。

**第二代的核心动机：用多 agent 提升质量。** 在 reasoning model 出现之前，单个模型的推理质量有明显的天花板。用多个 agent 扮演不同角色（研究员、批评者、写作者）相互讨论和审校，确实能提升最终输出的质量。这是 CrewAI 等框架的核心设计哲学。

**第三代：质量动机消失，三个新动机涌现。** Reasoning model 内化了"多角色讨论"的大部分认知价值——一个模型在 extended thinking 中自己就会做类似的多视角审查。多 agent "讨论提升质量"的性价比骤降。取而代之的三个新动机：

并行化加速。Reasoning model 的单次调用从秒级跳到分钟级。当任务存在独立分支时，多 agent 并行执行可以显著缩短总延迟。这是纯物理层面的收益，和模型能力无关。

Context 隔离。长 reasoning trace 会迅速膨胀 context。如果所有工作在一个 agent 内完成，50 轮工具调用后 context 可能已经被早期的无关信息淹没。把独立子任务交给子 agent 处理，子 agent 的 context 是隔离的——它的 reasoning trace 不会污染主 agent 的 context。Claude Code 的子 agent 不能再派子 agent 的限制，本质上就是防止 context 隔离层级无限嵌套。

成本混合编排。高 effort（reasoning model / high effort 参数）用于 orchestrator 的决策——选择子任务、分配优先级、评估结果；low effort（快模型 / low effort 参数）用于 worker 的执行——具体的代码修改、数据格式转换、文件操作。这种异构成本结构可以在保持决策质量的同时显著降低总成本。与维度 7 Inference-Time Compute 中讨论的两层路由决策（选模型 × 选 effort 级别）直接对应。

## 工具与框架

这里关注各框架在 **L3 层**（多 agent 协作的编排和通信设计）的架构选择。

**LangGraph** 采用图编排架构。Agent 之间的协作被建模为有状态图——节点是 agent 或处理步骤，边是条件转移，图的状态在节点间传递。这提供了最强的可控性和可视化能力：你可以精确定义哪些 agent 在什么条件下被调用，执行路径可以被可视化检查。代价是灵活性受限于预定义的图结构——动态增减 agent 或改变执行路径需要修改图定义。LangSmith 提供了图执行的原生可视化，是 LangGraph 在生产环境中的核心调试手段。当前 L3 层面生产就绪度最高的框架。状态：**成熟**。

**CrewAI** 采用角色模式。每个 agent 被分配角色（Role）、目标（Goal）和背景（Backstory），agent 之间通过对话协作。核心是一种模拟团队协作的抽象——有管理者 agent 分配任务给专家 agent。学习曲线是所有框架中最低的：定义几个角色和它们的工具就可以运行。但在 reasoning model 时代，CrewAI 的核心使用动机（多角色讨论提升质量）正在收窄。它的最佳场景收敛到：需要不同工具集的角色分工、需要明确的责任边界的任务。状态：**成熟，但核心使用动机在收窄**。

**Microsoft Agent Framework**（2025 年末由 AutoGen 和 Semantic Kernel 合并而成）。AutoGen 本身的核心贡献是 Group Chat 模式和 Magentic-One 系统（动态自适应编排的代表性实现）。合并后的框架强调企业集成（Azure 生态、Microsoft 365）。AutoGen 的原始版本已进入维护模式。值得注意的历史教训是 Microsoft 自己从专家 agent 集群回退到通才 agent 的经验——这在一定程度上反映了当前 agent 框架的过度工程倾向。状态：**转型期**。

**Google ADK（Agent Development Kit）** 采用层级 agent 树架构。每个 agent 可以有子 agent，形成树状结构，任务通过树的层级自然分解和委派。与 Google 生态（Vertex AI、A2A 协议）深度集成。树状结构的优势是递归分解直觉性强，劣势是不擅长处理需要跨分支通信的协作。状态：**上升期**。

**OpenAI Swarm / Agents SDK** 的 handoff 原语提供了最轻量的多 agent 支持。Swarm 本身定位为教育和原型验证工具（不保证生产稳定性），但其 handoff 设计思想——agent 间的控制权转移作为一等公民——被 Agents SDK 吸收。Agents SDK 的多 agent 支持极其克制：只提供 handoff，不提供 group chat 或复杂编排。这种克制本身就是一种设计立场——鼓励开发者在确认需要复杂编排之前先用简单 handoff 解决问题。状态：**上升期（Agents SDK）/ 实验性（Swarm）**。

### 协议层

**A2A（Agent-to-Agent Protocol）** 标准化 agent 之间的横向通信。与 MCP（标准化 agent → tool 的纵向连接）互补。核心概念：Agent Card 描述 agent 的能力和接口（v1.2 引入加密签名），Task 是协作的基本单位，Message 是 task 内的通信。当前 150+ 组织在生产环境使用，归入 AAIF 治理。A2A 解决的核心问题是跨框架、跨组织的 agent 互操作——你的 LangGraph agent 需要调用合作伙伴的 CrewAI agent 时，A2A 提供标准化的发现和通信机制。但 A2A 的成熟度不及 MCP，当前更多用于企业间的 agent 集成场景而非组织内部的 agent 编排。详见 [layers/L2-world-interface.md](L2-world-interface.md) 的 A2A 章节。

### 框架选型的一个交叉观察

L1 文件中已经提到：**弱模型 + 好编排经常胜过强模型 + 差编排。** 这个观察在 L3 有一个推论：框架的编排质量比框架支持的 agent 数量更重要。一个设计精良的两 agent 系统（orchestrator + worker）通常比一个设计粗糙的五 agent 系统更可靠、更便宜、更可调试。选框架时应该评估它的编排控制能力和可观测性，而非它支持多少种多 agent 模式。

## 塌缩与涌现

### 塌缩

**多 agent 投票 / 辩论提升质量的动机。** 这是第二代多 agent 的核心价值主张。Reasoning model 内部已经在做类似的多路径搜索和自我验证，外部再用多个 agent 讨论的增量收益很小。残余价值仅限于需要引入模型自身不具备的外部约束的场景（如合规审查、领域专家知识）。

**固定角色分配的刚性编排。** 预先定义"研究员 agent → 分析师 agent → 写作 agent"这种固定流水线的模式在 reasoning model 时代价值下降。Reasoning model 自己可以在一次调用中完成研究、分析、写作的全流程。固定角色编排仍有价值的场景是角色之间需要不同的工具集或权限——但这时驱动分工的是权限隔离而非认知分工。

**每框架一套的 agent 通信格式。** 各框架（LangChain、AutoGen、CrewAI）各自定义的 agent 间消息格式正在被 A2A 标准化取代。这个塌缩尚未完成——A2A 的采纳率还远低于 MCP，但方向已经明确。

### 涌现

**成本混合编排。** 高 effort orchestrator + low effort worker 的异构成本模式。这在第二代不存在——当时所有模型的调用成本差异不大。Reasoning model 时代，不同 effort 级别之间的成本差可达 20-50 倍，让"用贵模型决策、用便宜模型执行"成为一个有显著经济价值的模式。

**动态 agent 生成。** 不是预先定义所有 agent，而是 orchestrator 根据任务需求在运行时决定需要几个 agent、每个做什么。Magentic 编排模式是这个涌现的具体体现。当前仍在上升期，生产中的应用有限。

**A2A 跨系统协作。** Agent 不再局限于同一个应用内部的协作，而是跨组织、跨框架互操作。这催生了新的工程问题：agent 身份验证、跨边界的权限管理、通信延迟的处理。

**Context 隔离作为多 agent 的核心架构价值。** 第三代涌现出的新认识：多 agent 的价值不只是"做得更多"，还包括"保持每个 agent 的 context 干净"。这改变了多 agent 系统的设计目标——从最大化并行度转向最优化 context 管理。

## 开放工程问题

- **多 agent 的可观测性。** 单 agent 的 trace 已经足够复杂（L4 详述），多 agent 系统的 trace 是分布式的——每个 agent 有自己的 trace，agent 之间的消息传递构成额外的连接。如何在不丢失细节的前提下可视化和检索这些分布式 trace，是一个未解决的工程问题。当前 LangSmith 对 LangGraph 有原生支持，但通用的多 agent 可观测性标准不存在。
- **共享状态管理。** 多个 agent 需要读写的共享上下文（如一个文档的当前状态、一个数据库的查询结果）如何管理？当前的方案要么是把完整状态复制到每个 agent 的 context 中（token 开销大），要么是用外部存储（引入一致性问题）。没有成熟的"agent 间共享状态"抽象。
- **多 agent 系统的评测方法论。** 如何评测一个多 agent 系统的表现？是看最终结果（end-to-end），还是看每个 agent 的独立表现？如何归因——当最终结果错误时，是哪个 agent 的哪个决策导致的？当前缺乏标准化的多 agent 评测框架。与 L4 开放问题中"综合性 agent benchmark 不存在"直接相关。
- **Agent 间信任模型。** 在同一个框架内部，agent 之间可以假设相互信任。但在 A2A 跨组织场景下，一个 agent 如何验证另一个 agent 的身份和意图？Agent Card 的签名机制（A2A v1.2）是第一步，但完整的信任模型尚未建立。与 L4 安全章节的 OWASP #7（Insecure Inter-Agent Communication）直接相关。
- **多 agent 的成本归因。** 当一个请求触发了多个 agent 的协作时，成本如何归因到具体的 agent 和步骤？当前的 token 追踪通常是 per-request 的，不支持跨 agent 的成本传播。与 L4 成本工程章节的 FinOps 讨论相关。

## 参考资料

### 架构与编排模式

- [Multi-Agent AI System Architecture Patterns — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/multi-agent-ai-system-architecture-patterns) — 五种生产验证的编排模式（Sequential、Concurrent、Group Chat、Handoff、Magentic），Microsoft 的官方架构指南
- [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — Orchestrator-Workers 模式的原始定义，workflow vs agent 的核心区分
- [When (and When Not) to Use Multi-Agent Systems — Augment Code](https://www.augmentcode.com/blog/when-and-when-not-to-use-multi-agent-systems) — 单 agent vs 多 agent 的决策框架，含 +81% / -70% 的任务结构数据

### 决策框架与经验教训

- [The Multi-Agent Trap: Why More Agents Don't Mean Better Results](https://www.google.com/search?q=multi+agent+trap+why+more+agents) — 过度使用多 agent 的反面案例和教训
- [Single Agent vs Multi-Agent: A Cost-Benefit Analysis — Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/agentic-systems) — Anthropic 的 agentic 系统指南，包含多 agent 的 token 经济学数据

### 交接与通信

- [Handoff Protocols in Multi-Agent Systems — PepperEffect](https://peppereffect.com/multi-agent-handoff-protocols/) — 四种交接模式的分类和生产比较
- [A2A Protocol — Google Developers](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) — Agent-to-Agent 通信标准化

### 框架

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/) — 图编排框架的官方文档
- [CrewAI Documentation](https://docs.crewai.com/) — 角色模式框架的官方文档
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) — Handoff 原语的官方文档
- [Google Agent Development Kit](https://google.github.io/adk-docs/) — 层级 agent 树架构

### 可靠性与成本

- [Multi-Agent System Reliability: Failure Patterns and Validation Strategies — Maxim](https://www.getmaxim.ai/articles/multi-agent-system-reliability-failure-patterns-root-causes-and-production-validation-strategies/) — 多 agent 故障模式的系统性分析
- [The Hidden Economics of AI Agents — Stevens](https://online.stevens.edu/blog/hidden-economics-ai-agents-token-costs-latency/) — Token 成本与延迟的权衡

## 更新日志

- 2026-05-01：初次创建。覆盖单 agent vs 多 agent 决策框架、五种编排模式、四种交接模式、第三代动机转变、框架 L3 设计选择、A2A 协议层、塌缩与涌现
