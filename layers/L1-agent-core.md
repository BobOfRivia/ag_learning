# 第 1 层：Agent 内核（Agent Core）

## 这一层解决什么问题

让模型从"回答问题"变成"完成任务"。一个 LLM 本身是无状态的请求-响应函数——给它一段文本，它生成下一段文本。Agent 内核要在这个基础上构造出一个持续运行的循环：感知当前状态、决定下一步行动、执行、观察结果、再决定——直到任务完成或明确失败。

这一层是整个 Agent 工程离 LLM 最近的地方。它不关心 agent 能调用什么工具（那是 L2）、如何与其他 agent 协作（那是 L3）、或如何在生产环境中可靠运行（那是 L4）。它只关心一件事：**单个 agent 的思考-行动循环如何工作。**

## 关键工程模式

### Agent Loop：思考-行动循环

Agent loop 是 L1 的原子结构。所有 agent 系统——无论多复杂——最终都建立在某种形式的循环之上。这个循环的基本骨架只有一句话：**模型生成下一步动作 → 外部执行 → 结果回注上下文 → 模型再生成 → 直到任务结束。** 差异在于循环内部的思考和行动如何组织。

**经典 ReAct（2022–2024，仍在非 reasoning model 上使用）。** Thought → Action → Observation 三步紧密交错。模型先用自然语言"想一步"（Thought），然后生成一个工具调用（Action），拿回结果（Observation），再想下一步。每一轮思考只规划一步行动，思考和行动的粒度严格对齐。这个模式的核心价值在于：思考过程外显，便于调试和干预；每一步都有 grounding（基于真实观察而非幻觉）。局限也明显：串行执行导致延迟线性增长，模型需要在每一轮都重新"读"完整个上下文。

**思考-行动双相位（2024 末至今，reasoning model 时代的主导模式）。** Reasoning model 的出现改变了循环的内部结构。模型不再一步一想，而是在 extended thinking / thinking tokens 中进行一段完整的内部推理——可能包括规划多步、预判分支、自我纠错——然后在输出中一次性生成一个或多个工具调用。思考和行动不再紧密交错，而是分成两个相位：先集中想，再集中做。这个模式更高效（一次思考可以规划多步），但也更不透明（思考过程可能被隐藏或截断）。Anthropic 的 Claude 系列在 extended thinking 模式下就是典型的双相位行为：thinking 中完成推理，output 中生成工具调用。

**最小 agent loop（当前生产系统的事实标准）。** 剥掉所有抽象层，当前最成功的 agent 系统的核心循环惊人地简单：`while(model_response 包含 tool_call) → 执行工具 → 将结果追加到消息历史 → 再次调用模型`。循环在模型生成纯文本（不含工具调用）时自然终止。Claude Code 的内部实现就是这个模式——一个单线程的主循环，没有多 agent 竞争、没有显式的状态机，只有一个扁平的消息列表和一个 while 循环。这种简洁性不是简陋，而是一个经过验证的架构判断：**单线程主循环 + 严格的工具设计 + 良好的 context 管理 = 可控的自主性。** 复杂性不在循环本身，而在循环运行时模型看到的 context 里。

三种模式不是互斥的演进替代关系。经典 ReAct 对非 reasoning model 仍然有效；双相位是 reasoning model 的自然行为；最小 loop 是工程实现层面的共识，无论模型内部用哪种思考模式，外层都是这个循环。

### Context Engineering：L1 当前最核心的工程主题

> 深度展开见 [topics/context-engineering.md](../topics/context-engineering.md)

Context engineering 在 2025 年中期迅速取代 prompt engineering 成为 agent 工程的核心概念。这个转变的本质是：**当 agent 从单轮对话变成几十甚至上百轮的持续任务执行时，"写好一个 prompt"远不够——你需要设计一个动态系统来管理模型在每一轮看到的全部信息。**

Prompt engineering 是你在 context window 内做什么；context engineering 是你决定什么进入 context window。

这个区分之所以重要，是因为 agent 的 context window 是一个**稀缺资源，且随循环推进持续膨胀**。每一轮工具调用的结果、每一段 reasoning trace、每一次错误的堆栈信息都在往 context 里追加内容。不加管理，agent 在执行到第 30 步时，context 已被早期的无关信息淹没，模型的注意力被稀释，决策质量下降。

当前 context engineering 的关键技术和实践：

**KV-cache 优化。** 在生产环境中，context engineering 的第一优先级不是"放什么进去"，而是"确保缓存命中"。Manus 团队的经验表明，KV-cache 命中率是 agent 系统最重要的单一性能指标——缓存 token 和未缓存 token 的成本差 10 倍。具体做法包括：保持 system prompt 前缀稳定（不要在 system prompt 里放时间戳，哪怕一个 token 的差异都会使下游缓存失效）；context 只追加不修改（已经发生的 action 和 observation 不要事后编辑）；确保 JSON 序列化的确定性（字段顺序、空格都要一致）。

**文件系统作为外部记忆。** 当 context window 不够用时，一种被验证有效的策略是把信息卸载到文件系统，只在 context 中保留引用路径。Claude Code 的 CLAUDE.md、Manus 的 todo.md 都是这个思路的具体实例。关键原则是**压缩必须可逆**——你可以从 context 中移除一段内容，但前提是 agent 能通过 URL 或文件路径把它找回来。

**注意力操纵。** "Lost in the middle" 问题在长 context agent 中尤其严重：模型倾向于关注 context 的开头（system prompt）和末尾（最近的消息），中间的信息容易被忽略。应对方式包括：让 agent 周期性地创建和更新任务清单（把目标重新推到 context 末尾，进入模型的近期注意力范围）；对长工具输出做摘要而非原文拼接；关键信息在 system prompt 和运行时消息中做冗余放置。

**错误保留。** 一个反直觉但被验证有效的做法：不要从 context 中清除失败的工具调用和错误信息。当模型看到一个失败的操作及其堆栈信息时，它会隐式更新内部信念，避免在后续步骤中重复同样的错误。清除错误反而会让 agent 重蹈覆辙。

**受控多样性。** 如果 context 中所有的工具调用示例都用完全相同的格式和措辞，模型会过度拟合这种模式，面对略有不同的场景时变得脆弱。在 action 和 observation 的序列化中引入少量结构化变异（不同的模板、轻微的措辞差异）可以提升鲁棒性。

### 编排模式谱系

Agent loop 是原子操作，编排模式是如何组合这些原子操作来完成复杂任务。Anthropic 在 2024 年末的 "Building Effective Agents" 中提出了一个被广泛引用的分类框架，核心区分是 **workflow**（LLM 和工具通过预定义的代码路径编排）和 **agent**（LLM 动态控制自身的执行流程和工具使用）。Workflow 用可预测性换灵活性，agent 用灵活性换可预测性。

Anthropic 识别出的可组合构建块，按复杂度递增：

**Prompt Chaining。** 把任务分解为串行的步骤，每步的输出是下一步的输入，步骤之间可以插入程序化的质量门控。适用于任务可以被清晰分解、且每一步的质量需要独立验证的场景。这是最简单的编排模式。

**Routing。** 对输入进行分类，路由到专门的下游处理器。本质上是一个 if-else 分发，但分类由 LLM 完成。典型应用：根据问题难度路由到不同能力（和成本）的模型。Reasoning model 时代，routing 的重要性大幅上升——因为 reasoning model 贵 20-50 倍，不值得把简单任务也交给它。

**Parallelization。** 把独立的子任务同时交给多个 LLM 执行。两种形态：sectioning（把任务切成不重叠的子任务并行处理）和 voting（同一任务多次执行，结果投票或聚合）。voting 在 reasoning model 时代的边际收益已大幅下降（模型内部已经在做类似的搜索）。

**Orchestrator-Workers。** 中央 LLM 动态分解任务并分发给 worker LLM。比 parallelization 更灵活——子任务不是预先确定的，而是 orchestrator 根据任务性质实时决定。Claude Code 的子 agent 派发就是这个模式：主循环遇到需要独立探索的子问题时，派发一个 sub-agent 处理，sub-agent 不能再派发自己的 sub-agent（防止递归爆炸）。

**Evaluator-Optimizer。** 一个 LLM 生成输出，另一个 LLM 评估质量，评估结果反馈给生成者迭代改进。这是 agent 版的测试驱动开发。在 reasoning model 时代，这个模式的价值有微妙变化：reasoning model 内部已经有自我评估和修正能力，外部的 evaluator 更多用于施加模型自身不具备的评估标准（如业务规则、合规要求）。

**自主 Agent。** LLM 在循环中自主运行，通过环境反馈决定下一步，处理不可预测的任务，可能持续多轮。这是上述所有模式中自主性最高的，也是不确定性最大的——模型可能走入死循环、产生不可预期的行为。

**Plan-then-Execute vs ReAct vs 混合。** 这是一个正交于上述分类的设计选择。Plan-then-Execute 先制定完整计划再逐步执行，适合任务可预先分解的场景，有数据显示在某些 benchmark 上比 ReAct 有 3.6 倍的速度优势。ReAct 边想边做，适合需要根据中间结果调整策略的场景。当前生产系统几乎都采用混合方式：用 planning 提供结构，用 ReAct 提供适应性。Claude Code 的 TodoWrite 机制就是这种混合的具体实例——agent 在任务开始时制定一个结构化的任务清单（plan），但执行过程中可以随时根据观察修改计划（adapt）。

### 错误恢复与自我修正

Agent 的可靠性很大程度上取决于它如何处理失败。这在第三代发生了显著变化。

**第二代：外部反思循环。** Reflexion（2023）和 Self-Refine 代表了一种思路——让模型审视自己的输出，生成反思，然后重试。这是"用另一轮 LLM 调用修复上一轮 LLM 调用的错误"。工程上需要额外的 prompt、额外的调用、以及判断何时停止反思的终止条件。

**第三代：reasoning model 内化了大部分自我修正。** Reasoning model 在 extended thinking 中自发出现回溯、验证、纠错行为——不需要外部循环触发。这让 Reflexion 类技术大幅贬值。但并没有让所有错误恢复变得不必要：模型的自我修正只覆盖推理层面的错误，不覆盖环境层面的错误（工具调用超时、API 返回异常、文件系统状态变化等）。

**当前的生产错误恢复模式。** 分层处理：首先让模型自身尝试修正（reasoning model 天然支持）；如果是工具执行层面的错误，按指数退避重试；如果有替代工具可用，尝试替代路径；如果能接受部分结果，优雅降级；最后才升级到人工干预。一个关键的生产经验是**必须显式禁止无限重试**——这是最常见的生产故障模式，模型陷入重试循环而不知道放弃。

另一个值得注意的实践是**保留失败上下文**。如前文 context engineering 部分所述，把失败的 action 和错误信息保留在 context 中，让模型从失败中学习，比清除错误后让模型"干净地"重试更有效。

## 工具与框架

这里关注的是各框架在 **L1 层**（agent 内核的循环和编排设计）的架构选择，不是它们的完整功能比较。

**Claude Code** 采用单线程主循环（内部代号 nO）。核心设计：一个 while 循环 + 一个扁平的消息列表。没有多 agent 竞争、没有显式状态机。context 接近上限（~92%）时触发压缩器（wU2），将对话摘要写入长期记忆。支持通过异步队列（h2A）实现运行中的用户干预——用户可以在 agent 执行过程中注入新指令而不需要中断重启。复杂任务通过 TodoWrite 生成结构化任务清单。子 agent 派发有严格限制：子 agent 不能再派子 agent。这种约束的本质是用简洁性换可调试性和可控性——这在当前 agent 系统中被证明是一个成功的权衡。状态：**成熟，事实标准之一（coding agent 领域）**。

**OpenAI Agents SDK**（2025 年 3 月发布）采用极简架构：Agent、Handoff、Session、Tracing 四个核心概念。设计哲学是"closer to the metal"——只提供 80% 的 agent 模式真正需要的基础设施，拒绝过度抽象。Handoff 机制允许 agent 之间的控制权转移，本质上是一种轻量的 orchestrator-worker 模式。状态：**上升期**。

**LangGraph** 采用状态机架构。Agent 的行为被建模为图中的节点和边——每个节点是一个处理步骤，边是条件转移。这提供了最强的可控性和可视化能力，代价是学习曲线更陡、灵活性受限于图的预定义结构。适合需要精确控制执行流程的企业场景。状态：**成熟**。

**CrewAI** 采用角色扮演架构。每个 agent 被分配角色、目标和背景故事，agent 之间通过对话协作完成任务。本质上是一种 managerial process 模式——有管理者 agent 分配任务给专家 agent。适合需要模拟团队协作的场景，但在 reasoning model 时代，纯认知性的多角色讨论这一动机已大幅削弱。状态：**成熟，但核心使用动机在收窄**。

**AWS Strands Agents**（2025）支持在同一个 agent 内切换多种编排模式——同一个 agent 可以在不同步骤中使用 ReAct、Plan-then-Execute 或自定义的控制流。这种"编排模式是运行时参数而非编译时决定"的设计思路值得关注。状态：**上升期**。

**Microsoft Agent Framework**（2025 年末由 AutoGen 和 Semantic Kernel 合并而成，2026 Q1 GA）试图统一微软的 agent 基础设施。AutoGen 本身已进入维护模式。合并后的框架强调企业集成（Azure 生态、Microsoft 365）。状态：**转型期**。

一个跨框架的观察：**弱模型 + 好编排经常胜过强模型 + 差编排。** 2026 年 2 月的一组测试显示，三个框架使用相同的底层模型，在 731 个问题上的得分差距达 17 分。Benchmark 分数应该和编排架构一起读，不能只看模型标签。

## 塌缩与涌现

### 塌缩（第三代 reasoning model 对 L1 的冲击）

**CoT prompting 大幅贬值。** "Let's think step by step" 对 reasoning model 不仅无效，某些情况下反而有害——它干扰了模型自身的推理节奏。对于非 reasoning model，CoT 仍然有效，但 L1 的工程设计不再以 CoT prompting 为核心前提。

**Self-Consistency 边际收益骤降。** 多次采样投票的收益建立在"模型推理路径多样性"的前提上。Reasoning model 内部已经在做类似的搜索和验证，外部再做一层的增量价值很小。

**Reflexion / Self-Refine 大幅简化。** 外部反思循环的大部分价值已被 reasoning model 内化。还有残余价值的场景是：需要引入模型自身不具备的外部评估标准（如业务规则验证、合规检查）。

**经典 ReAct 的紧密交错被双相位取代。** 模型不再"想一步做一步"，而是"想完再做"。这不是 ReAct 概念的消亡，而是它的实现粒度变了——从单步交错变成多步批量。

**Plan-and-Execute 的 planning 阶段部分塌缩。** Reasoning model 可以在一次 thinking 中完成原本需要单独 planning 步骤才能做到的分解和排序。显式的"先生成计划再执行"的两阶段流程在简单任务上不再必要。

**"Researcher + Critic + Writer" 类纯认知性多角色编排。** 这种用多个 agent 模拟团队讨论来提升输出质量的模式，在单个 reasoning model 已经足够强的情况下，性价比大幅下降。

### 涌现

**Context engineering 成为 L1 的核心工程学科。** 当 agent 从几轮对话变成几十上百轮的持续执行时，context 管理从"写好 prompt"升级为"设计一个动态的信息架构系统"。这是第三代最重要的 L1 涌现。

**Reasoning budget 管理。** 模型在每个请求上花多少推理算力成为可调参数。L1 需要决定：哪些步骤需要深度思考，哪些步骤只需要快速执行。这催生了任务内部的 model routing——同一个 agent 在不同步骤中使用不同强度的推理。

**Model routing。** 简单任务 → 快模型（成本低、延迟小），复杂任务 → reasoning model（成本高、质量高）。这在 L1 层表现为：agent 内核需要具备"判断当前步骤的难度并选择合适模型"的能力，或者至少需要支持外部系统做这个路由。

**Reasoning trace 的处理。** Reasoning model 的 thinking tokens 是否保留到后续 context？保留太多会挤占 context 空间，丢弃太多会丧失中间推理的价值。当前没有共识做法。

**运行时用户干预。** Agent 执行过程中允许用户注入新指令、修正方向，而不需要中断重启。Claude Code 的 h2A 异步队列是这个涌现的一个工程实现。这改变了 agent 的交互范式——从"发出指令然后等结果"变成"持续协作"。

**Sub-agent 派发与隔离。** 当主循环遇到需要独立探索的子问题时，派发一个有限能力的子 agent 处理，结果回收到主循环。关键工程约束：子 agent 不能递归派发、context 需要隔离（防止污染主循环的 context）、需要资源上限（防止子 agent 无限消耗）。

## 开放工程问题

- **Context engineering 的最优策略是什么？** 当前各家的做法（Claude Code 的 CLAUDE.md + 压缩器、Manus 的 todo.md + 文件系统卸载）都是经验驱动的，缺乏系统化的理论框架。什么时候该压缩、压缩到什么粒度、哪些信息值得保留——这些决策还没有被 benchmark 化。
- **Reasoning trace 保留多少？** 在 context 空间和推理价值之间的权衡没有定论。完全保留会挤占 context，完全丢弃会丧失推理链的价值。是否应该对 trace 做选择性摘要？标准是什么？
- **单 agent 的能力边界在哪？** 什么任务一个 agent + 好的 context engineering 就能解决，什么任务必须拆成多 agent？当前缺乏清晰的判断框架。这个问题直接决定 L1 和 L3 的边界。[layers/L3-multi-agent.md](L3-multi-agent.md) 的"核心决策"章节提供了一个基于任务结构和 token 经济学的启发式决策规则。
- **Agent 循环的终止条件。** 除了"模型不再调用工具"这个隐式终止外，如何设计显式的终止条件？超时、步数上限、成本上限、质量检查——各种终止策略的权衡没有标准答案。关于终止策略和复合可靠性问题的生产视角，详见 [layers/L4-production.md](L4-production.md) 的可靠性章节。
- **用户干预的粒度。** 运行时干预是一个有价值的涌现，但最优的干预粒度是什么？太细（每步确认）等于人工操作，太粗（只在开始和结束）等于完全自主。当前 Claude Code 的权限系统（白名单 + 人工确认）是一种实用方案，但不是通用解法。关于风险分层的 HITL 架构，详见 [layers/L4-production.md](L4-production.md) 的人机协同章节。

## 参考资料

### 架构与设计模式

- [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — Anthropic 2024 年末的 agent 架构指南，提出 workflow vs agent 的核心区分和六种可组合构建块，被广泛引用
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — Manus 团队的 context engineering 实战经验，KV-cache 优化、文件系统作为记忆、注意力操纵等关键技术的一手来源
- [Claude Code: Behind the Scenes of the Master Agent Loop](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/) — Claude Code 内部架构的逆向分析，单线程主循环、压缩器、子 agent 派发的实现细节
- [Claude Code Agent Architecture: Single-Threaded Master Loop](https://www.zenml.io/llmops-database/claude-code-agent-architecture-single-threaded-master-loop-for-autonomous-coding) — 对 Claude Code 架构的另一个视角，强调"简洁性即架构优势"

### Context Engineering

- [Context Engineering for AI Agents — Towards Data Science](https://towardsdatascience.com/deep-dive-into-context-engineering-for-ai-agents/) — context engineering 的技术深度文章
- [Context Engineering — Weaviate](https://weaviate.io/blog/context-engineering) — 从检索和记忆角度看 context engineering
- [Context Engineering: From Prompts to Corporate Multi-Agent Architecture](https://arxiv.org/pdf/2603.09619) — 学术视角的 context engineering 框架化
- [Context Engineering — Gartner](https://www.gartner.com/en/articles/context-engineering) — Gartner 对 context engineering 的行业定义和趋势分析

### Agent 框架

- [Anthropic 2026 Agentic Coding Trends Report](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf) — Anthropic 的 agentic coding 趋势报告
- [Inside the Agent Harness: How Codex and Claude Code Actually Work](https://medium.com/jonathans-musings/inside-the-agent-harness-how-codex-and-claude-code-actually-work-63593e26c176) — 对比 Claude Code 和 Codex 的架构差异
- [Top 5 Open-Source Agentic AI Frameworks in 2026](https://aimultiple.com/agentic-frameworks) — 框架横向对比，含架构差异分析
- [Agentic AI Architecture: Patterns and Production](https://thinking.inc/en/pillar-pages/agentic-ai-architecture/) — 从架构模式到生产部署的系统梳理

### 学术综述

- [Agentic AI: Architectures, Taxonomies, and Evaluation of LLM Agents](https://arxiv.org/html/2601.12560v1) — 2026 年 1 月的综合综述，统一分类体系（Perception / Brain / Planning / Action / Tool Use / Collaboration）
- [Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models](https://arxiv.org/abs/2510.04618) — ACE 框架，将 context 视为可演化的 playbook

## 更新日志

- 2026-05-01：开放问题中增加 L4 交叉引用（终止策略、HITL 架构）和 L3 交叉引用（单 agent 能力边界决策框架）
- 2026-04-27：初次创建。覆盖 agent loop 三种模式、context engineering、编排模式谱系、错误恢复、框架 L1 设计选择、塌缩与涌现
