# 第二代：Tool-RAG Era（2023–2024 中）

## 定义性变化

第二代的标志是**地基发生第一次实质性质变：模型获得原生 tool use 能力，context window 从 4K 扩大到 100K+ tokens。** 这两个变化的冲击波传导到 L2（与世界的接口），让 agent 从"只能思考"变成"能操作世界"。RAG 和 Tool 工程化成为这个时代的核心主题，工程重心从 L1 向 L2 大幅转移。

与第一代的根本区别在于：第一代的 agent 只能在 prompt 内部模拟行动（文本格式的伪工具调用），第二代的 agent 能真正执行外部操作（API 原生的函数调用）。这不是程度之差，而是性质之差——agent 第一次有了真正的"手"。

## 地基能力水位

**Tool Use：从零到原生。** OpenAI 在 2023 年 6 月推出 function calling API，模型可以在生成过程中主动决定调用外部函数并输出结构化的 JSON 参数。不再需要文本格式模拟和正则解析。这是第二代最重要的单一地基变化。随后，Anthropic、Google 等相继跟进，原生 tool use 在 2024 年中期成为所有主流模型的标准能力。

**Context Window：10-25 倍扩展。** GPT-4 Turbo 的 128K（2023-11）、Claude 2.1 的 200K（2023-11）、Gemini 1.5 Pro 的 1M（2024-02）。这个扩展的直接影响是：许多之前因为"放不进 context"而必须用 RAG 的场景，现在可以直接塞进去。但"能放进去"和"能用好"是两回事——Lost-in-the-Middle 问题在这个时期被发现和研究。

**Reasoning：仍然是外挂。** 模型没有内化的推理能力。CoT prompting 仍然是让模型做多步推理的标准方法。Self-Consistency、Reflexion 等第一代技术在这个时期被广泛采用和工程化。GPT-4（2023-03）在推理能力上比 GPT-3.5 有质的提升，但这是模型整体能力的提升，不是推理的内化——模型仍然需要 prompt 引导才会做多步推理。

**Instruction Following：显著改善。** GPT-4 和 Claude 2 在遵循复杂指令方面比前代明显更强，这使得更复杂的 agent 行为成为可能（能给模型更长的系统指令，模型能更可靠地遵循）。

**Memory：仍然是拼接。** 没有持久化记忆。LangChain 等框架提供了 ConversationBufferMemory / ConversationSummaryMemory 等标准化组件，本质上还是"把对话历史以某种方式拼进 prompt"。ChatGPT Memory 在 2024 年 2 月推出，是产品层整合持久记忆的第一次尝试。

## 工程重心与格局

**L2 成为主战场。** 这个时代的工程能量集中在两件事：怎么让 agent 用好工具（Tool Integration），怎么让 agent 获取外部知识（RAG）。L1 的基本范式（ReAct 循环）没有根本变化，只是从手工文本解析升级到了 API 原生调用。L3 开始出现（AutoGen 等框架推动多 agent 实验），但更多是探索而非生产。L4 刚刚萌芽——LangSmith（2023）和 Langfuse 等可观测性工具出现，但生产化的工程体系尚未建立。

**框架爆发期。** LangChain（2022 末发布，2023 年爆发增长）成为这个时代的标志性框架——它把 LLM call + prompt template + tool + memory + chain 封装成标准化组件，大幅降低了构建 agent 的门槛。随后 LlamaIndex（RAG 专精）、AutoGen（多 agent）、CrewAI（角色扮演多 agent）等框架涌现。这些框架共同的问题是**过度抽象**——为了通用性牺牲了灵活性，且每个框架都有自己的工具定义格式，导致工具生态碎片化。

**RAG 的工程化。** 2023-2024 年是 RAG 的黄金期。基本的 RAG pipeline（文档分块 → 向量化 → 存入向量库 → 用户提问时检索相关块 → 拼入 context → 模型生成答案）被大规模工程化部署。在此基础上衍生出大量优化技术：查询重写（query rewriting）、混合检索（sparse + dense）、re-ranking、多跳 RAG（先检索→基于结果再检索）、Graph RAG（用知识图谱增强检索）。向量数据库（Pinecone、Weaviate、Qdrant、Chroma、Milvus 等）成为热门基础设施赛道。

## 关键事件与里程碑

**2023-03：GPT-4 发布。** 在推理、instruction following、多模态理解上全面超越 GPT-3.5。不是单一维度的突破，而是地基整体水位的大幅抬升。

**2023-03：ChatGPT Plugins 推出。** OpenAI 试图构建一个平台方控制的工具市场——第三方开发者注册 plugin，用户在 ChatGPT 中启用。这是"agent 使用工具"的第一次大规模产品化尝试。

**2023-06：OpenAI 推出原生 function calling API。** 这是第二代的分水岭事件。从此，工具调用从 prompt hack 变成了 API 原生支持。

**2023-10：MemGPT 论文发表。** 提出 OS 式分层记忆架构——core memory（始终在 context 内）、recall memory（可搜索的对话历史）、archival memory（长期存储）。agent 主动管理自己的记忆而非被动接收注入。这个概念在第二代是学术探索，到第三代（Letta）开始进入生产。

**2023-11：GPT-4 Turbo（128K context）和 Claude 2.1（200K context）发布。** Context window 的大幅扩展开始侵蚀简单 RAG 场景的存在理由。

**2024-02：Gemini 1.5 Pro（1M context）发布。** 百万级 context 让"把所有文档塞进去"在技术上成为可能。RAG 的塌缩信号开始明确。

**2024-02：ChatGPT Memory 推出。** 产品层整合持久记忆的首次尝试——自动从对话中提取用户事实并跨 session 保存。

**2024-03：ChatGPT Plugins 退役。** 推出不到一年即关闭。失败原因：发现率低（用户不知道有什么 plugin）、体验差（plugin 质量参差）、商业模式不清（开发者无收入激励）。证明了"封闭工具市场"不是正确的标准化路径。

**2024-中：并行函数调用成为标准能力。** 模型可以在一次响应中生成多个独立的工具调用，框架并行执行。性能提升显著。

**2024-中：Structured Outputs / strict mode 出现。** OpenAI 推出 strict mode，通过受限解码保证 100% 的 schema 合规率。这解决了 tool use 可靠性的最后一个技术障碍（在简单调用场景下）。

## 塌缩与涌现

### 塌缩（第一代的技术/模式消失或贬值）

- **文本格式的伪工具调用**被 API 原生 function calling 完全替代。不再需要在 prompt 中定义 `Action: tool_name[params]` 格式，不再需要正则解析模型输出。
- **手工解析模型输出**大幅减少。结构化输出（JSON mode → strict mode）让模型输出可靠地符合预定义 schema。
- **简单的信息检索场景**开始被长 context 侵蚀。当 context window 从 4K 扩大到 128K-1M 时，"文档放不进 context 必须用 RAG"这个前提在很多场景下不再成立。
- **专用意图分类模型**。在 agent 需要判断用户意图并路由到不同处理器时，之前需要训练专门的分类模型；现在用一次 LLM 调用做 routing 即可。

### 涌现

- **Tool Integration 工程化**。工具定义 schema 设计、工具描述编写、错误处理策略、并行调用编排——一整套围绕工具集成的工程实践从零建立。
- **RAG 工程化**。从基础 pipeline 到查询重写、混合检索、re-ranking、Graph RAG——一个完整的 RAG 优化技术栈在这个时期形成。
- **向量数据库赛道**。Pinecone、Weaviate、Qdrant、Chroma、Milvus、PGVector 等大量向量数据库产品涌现，各自差异化定位。
- **Agent 框架生态**。LangChain、LlamaIndex、AutoGen、CrewAI 等框架建立了 agent 开发的标准化工具链。
- **多 Agent 探索**。AutoGen（微软）和 CrewAI 推动了多 agent 的早期实验。这个时期的多 agent 主要动机是"用多个角色讨论提升质量"——Researcher + Critic + Writer 这类纯认知性编排是典型模式。
- **可观测性萌芽**。LangSmith、Langfuse 等工具开始提供 agent 执行链路的追踪和可视化，L4 的雏形出现。

## 遗产与局限

第二代最重要的遗产是**让 agent 从玩具变成了工具**。原生 tool use + 大 context window 让 agent 第一次能做有实际价值的事情——查询数据库、调用 API、操作文件系统。RAG 的工程化让 agent 能访问组织的私有知识。这些基础能力的到位，是后续所有更复杂的 agent 应用的前提。

另一个重要遗产是**ChatGPT Plugins 的失败经验**。它证明了平台方控制的封闭工具市场行不通，为后来 MCP 的开放协议路线提供了反面教材。Plugins 的失败原因——发现率低、质量不可控、激励不对齐——在 MCP 的设计中被有意识地规避。

局限在于：**推理能力仍然是瓶颈。** 工具虽然有了，但 agent 在"决定什么时候用什么工具、怎么组合使用、出错了怎么调整"这些需要推理能力的问题上仍然脆弱。复杂的多步任务需要精心编排的 chain 或 graph 才能可靠执行，这本身就是 reasoning 不够强的症状——如果模型自己能想清楚执行策略，就不需要工程师预先编排。

另一个局限是**框架层的过度抽象和碎片化**。每个框架都有自己的工具格式、自己的 chain/graph 定义方式、自己的 memory 组件。这些抽象在 reasoning model 时代被证明是多余的——当模型自己能规划执行路径时，大部分预定义的 chain 和 graph 失去了存在价值。

## 与相邻代际的关系

**← 承接第一代**：第二代把第一代用 prompt 模拟的工具调用能力（ReAct 的 Action 部分）变成了地基原生支持。ReAct 的概念框架不变，但实现从文本解析升级为 API 调用。第一代的 reasoning 技术（CoT、Self-Consistency 等）在第二代仍然是主力——因为推理还没有内化。

**→ 第三代的过渡信号**：2024 年下半年出现了两个明确的信号。一是 OpenAI o1 的发布（2024-09），宣告 reasoning 内化的开始，直接对第一代和第二代积累的 reasoning 外挂技术构成替代。二是 Anthropic MCP 的发布（2024 末），把第二代碎片化的工具生态引向标准化。这两个事件——一个是地基质变（reasoning 内化），一个是生态事件（工具标准化）——共同标志着第三代的开始。

**对第四代的远程影响**：第二代建立的 RAG 工程和向量数据库基础设施，如果第四代的 memory 内化假设成立，将面临大面积塌缩。但第二代在 memory 工程化上的探索（MemGPT 的分层架构、LangChain 的 memory 组件）为第四代的记忆系统设计提供了经验基础。

## 更新日志

- 2026-04-27：初次创建
