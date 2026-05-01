# 维度 6：Memory

## 一句话定义

Memory 是 LLM 跨交互持久化、组织和调用信息的能力——它决定一个 agent 能否积累经验、维持身份、并随时间改进。

## 与 Long Context 的关键区分

Memory 和 Long Context（维度 2）经常被混淆，但本质不同。Long Context 是**单次交互的感知范围**——一个 128K 窗口能装下多少信息。Memory 是**跨交互的积累能力**——agent 能否记住上周的对话、上个月的偏好、去年学到的教训。

超长 context（1M+ tokens）能"伪装"记忆——把所有历史对话塞进一个窗口。但这不是真正的记忆：它不做选择性遗忘、不做抽象提炼、不处理矛盾信息、成本随历史线性增长。真正的记忆系统需要**写入、组织、检索、更新、遗忘**的完整生命周期管理，而不只是"装更多东西进窗口"。

2026 年的 MemoryArena benchmark 为这个区分提供了定量证据：模型在被动回忆型 benchmark（如 LoCoMo）上得分 80%+，但在需要主动运用记忆的 agentic task 中骤降至 40-60%。"能想起来"和"能用起来"之间存在巨大鸿沟。

## 核心概念与技术

### 记忆类型：认知科学对照

2025-2026 年的学术和工程社区已收敛到一个源自认知科学的四类分法，这个分类不是学术装饰——它直接影响工程架构，因为每种记忆类型对应不同的存储策略和检索模式。

**工作记忆（Working Memory）** 对应 context window 本身。它是 agent 当前正在处理的信息，容量有限、生命周期短。所有其他类型的记忆最终都要通过工作记忆才能影响 agent 行为——信息必须被加载到 context 中才能被使用。这使得工作记忆成为整个记忆系统的瓶颈。

**情景记忆（Episodic Memory）** 记录"发生了什么"——带时间戳的具体交互事件。用户上周二提过他搬了家；agent 在某次任务中犯了一个错误然后修正了。情景记忆保留具体情境，是反思和经验学习的原料。

**语义记忆（Semantic Memory）** 存储从多次交互中抽象出的结构化知识和事实。"用户偏好 TypeScript"、"这个代码库使用 monorepo 结构"。它是情景记忆提炼后的产物——丢掉具体时间和情境，保留可复用的知识。

**程序记忆（Procedural Memory）** 编码学到的行为模式和工作流。不是"知道什么"，而是"知道怎么做"。Voyager 的技能库（skill library）是典型案例——agent 将成功完成任务的步骤存为可复用的程序，在类似场景中直接调用，而非从头推理。Voyager 的实验显示，有程序记忆的 agent 技术树推进速度快 15.3 倍。

### 表示基底：参数 vs 非参数

理解 Memory 维度演进的核心轴是**参数记忆（parametric）vs 非参数记忆（non-parametric）**的分野。

**参数记忆**指编码在模型权重中的知识。预训练和微调本质上就是在写入参数记忆。知识编辑技术（ROME、MEMIT、WISE）尝试在不重训模型的前提下精确修改权重中的特定事实——MEMIT 能同时编辑数千条记忆。但这条路线面临根本挑战：编辑的精确性有限（改一个事实可能破坏相关知识），无法做到真正的"遗忘"，且不支持个性化（同一组权重服务所有用户）。当前状态：学术活跃但工程落地极少。

**非参数记忆**指存储在模型外部的信息——向量数据库、知识图谱、结构化数据库、纯文本文件。当前所有生产级 agent 记忆系统都是非参数的。优势是灵活（可随时增删改）、可个性化（每用户独立存储）、可审计（知道 agent 记住了什么）。劣势是需要检索（引入延迟和相关性问题）和上下文占用（加载的记忆消耗宝贵的 context 窗口）。

这两条路线有不同的失败模式：参数记忆的失败是"改不准"（编辑副作用），非参数记忆的失败是"找不到"（检索不相关）。目前没有一方占优势，但工程生态几乎完全押注在非参数路线。

### 记忆生命周期：不只是存和取

记忆系统远不是"存进去、取出来"那么简单。写入、存储、检索是所有生产记忆系统都需要处理的基本操作，遗忘和更新在合规场景（如 GDPR）中同样是硬需求。以下按阶段梳理各自的工程问题（六阶段的形式化来自一篇 2026-04 的学术调查，但每个阶段对应的工程挑战在生产系统中都是真实的）：

**写入（Write）**——决定什么值得记住。这是第一道也是最关键的过滤：对话中大量内容是临时性的，只有少数信息有持久价值。写入策略的设计是领域相关的——医疗 agent 需要高召回率（宁可多记），日常助手可以容忍遗漏。Mem0 的核心创新之一就是用 LLM 做选择性提取，而非记录一切。

**存储（Store）**——如何组织已写入的记忆。涉及表示格式（文本/向量/图/结构化数据）、索引方式、版本管理。这个阶段的关键工程问题是**记忆过时（staleness）**——高置信度的事实随时间变得错误但系统不知道。

**检索（Retrieve）**——在需要时找到相关记忆。这是当前的主要瓶颈。Mem0 的 benchmark 数据显示：全量上下文准确率 72.9% 但 P95 延迟 17 秒、token 消耗 26000；选择性记忆（Mem0g）准确率 68.4% 但延迟仅 2.59 秒、token 消耗 1800。准确率和效率之间的权衡是检索阶段的核心张力。

**执行（Execute）**——检索到的记忆如何影响 agent 行为。这个阶段的风险在于记忆可能覆盖用户的显式指令——如果检索到的旧记忆和当前指令矛盾，agent 该听谁的？

**遗忘（Forget）**——主动清除不再需要或不应保留的记忆。选择性遗忘是当前所有系统的普遍弱点——MemoryAgentBench 的评测显示没有任何系统在这个维度上表现合格。遗忘还涉及合规需求（GDPR 的被遗忘权）。

**回滚（Rollback）**——在记忆被污染后恢复到已知良好状态。这在安全语境下至关重要，但几乎没有系统实现了真正的版本化回滚。

### 五种工程机制

> 深度展开见 [topics/memory-architecture.md](../topics/memory-architecture.md)

2026 年 3 月的综合调查将当前的记忆工程实践归纳为五个机制家族，它们不是互斥的——生产系统通常组合使用多种机制：

**上下文驻留压缩（Context-Resident Compression）**——在 context window 内部管理记忆。滑动窗口、滚动摘要、层级压缩。优势是简单、无外部依赖。限制是"摘要漂移"（多次压缩后罕见但关键的细节丢失）。这条路线与 context engineering（topics/context-engineering.md）高度重叠。

**检索增强存储（Retrieval-Augmented Stores）**——将记忆存在外部向量库，需要时检索注入 context。这是当前最主流的生产方案。RETRO 证明了从万亿 token 级语料中检索的可行性。瓶颈已从存储能力转向**检索相关性**——存什么都行，问题是能不能在对的时间找到对的东西。

**反思式自我改进（Reflective Self-Improvement）**——agent 对自身经验进行反思，提炼出抽象规则和教训。Reflexion 在 HumanEval 上达到 91% pass@1。Generative Agents 的实验表明，没有反思机制的 agent 在 48 小时内退化为重复的、无上下文的回复。风险是自我强化的错误——从错误经验中提炼出错误规则。

**层级虚拟上下文（Hierarchical Virtual Context）**——受操作系统内存管理启发，在 context window（RAM）和外部存储（磁盘）之间建立分页机制。MemGPT/Letta 是代表：core memory 始终在 context 内，recall memory 和 archival memory 在需要时换入。agent 主动管理自己的内存——调用 memory function 决定什么换入换出，而非被动接收注入的上下文。

**策略学习型管理（Policy-Learned Management）**——用 RL 优化记忆操作本身，将存储/检索/更新/摘要/丢弃视为可训练的动作。A-Mem（NeurIPS 2025）是代表工作。**落地程度：学术阶段，尚无生产部署案例。**

### 图记忆：2024 实验 → 2026 生产

图记忆（Graph Memory）在 2024 年还是实验性的，到 2026 年初已进入生产。它和向量记忆的区别是精确的：**向量记忆检索语义相似的事实，图记忆检索通过关系连接的事实**。

向量记忆的局限在于它是"无结构的相似性匹配"——当你问"Alice 的经理是谁"时，向量搜索可能返回所有提到 Alice 的段落，但不一定包含答案。图记忆通过实体和关系的显式建模，支持多跳推理和时序推理。Zep 的 Graphiti 框架是代表实现，建立在 Neo4j 之上，构建时序感知的知识图谱。

但图记忆也引入新的攻击面：学术研究（GRAGPoison）在实验室条件下展示了利用共享关系结构的攻击路径。图结构增强了检索精确性的同时，也为结构性污染提供了传播通道——这是向量存储不存在的风险。

### 记忆安全：一个独立的安全类别

> 深度展开见 [topics/memory-security.md](../topics/memory-security.md)

agent 记忆构成一个独立的安全类别，因为它具有三个传统 prompt injection 不具备的属性：**持久性**（毒素跨 session 存活）、**状态性**（累积的偏差导致行为漂移）、**传播性**（污染通过多 agent 共享记忆扩散）。攻击面覆盖记忆生命周期的每个阶段——写入（注入有毒记忆）、检索（操纵检索结果）、共享（跨 agent 传播污染）。

**落地程度**：记忆安全目前几乎完全是学术研究领域。已有多篇论文展示了攻击路径（MemoryGraft、InjecMEM 等），但攻击和防御都停留在实验室条件下。没有主流记忆框架（Mem0、Letta、Zep）将安全作为核心设计原则——在它们的文档中，安全属于"未来路线图"。这可能成为 agent 记忆从 demo 到企业生产的瓶颈。

## 当前技术格局（截至 2026-04）

### 产品层：厂商原生记忆

**ChatGPT Memory（OpenAI）**：自动从对话中提取并持久化用户事实（典型月活跃用户 40-80 条存储事实）。本质是"选择性事实提取 + 存储 + 注入 system prompt"的流程。ChatGPT Projects 在此基础上增加了按项目组织的指令和文件。在 LoCoMo benchmark 上准确率仅 52.9%，显著低于专门的记忆框架。

**Gemini（Google）**：与 Google 账户深度集成（Gmail、Drive、Calendar），获取用户真实生活上下文，但不像 ChatGPT 那样自动记忆对话历史。2026 年初增加了从其他 AI（如 ChatGPT）导入记忆的功能。记忆不跨平台——切到 ChatGPT 或 Claude 后积累的上下文全部丢失。

**Claude（Anthropic）**：通过 Projects 和 CLAUDE.md 文件提供项目级持久上下文，但不提供自动对话记忆。Anthropic 的路线更偏向让用户显式管理上下文，而非自动提取。

共同特征：所有厂商的"原生记忆"本质上都还是**非参数的外部存储 + 检索**，只是包装在产品内部。没有任何一家实现了真正的参数级记忆内化。gen4 判定标准第 1 条（至少两家在 API 层原生提供持久记忆）尚未满足——ChatGPT 的 Memory API 是最接近的，但它更像是一个便利功能，不是地基变化。

### 框架层：记忆基础设施

**Mem0**：2026 年最成熟的独立记忆框架。核心模型是选择性提取——用 LLM 判断对话中什么值得记住，而非记录一切。提供向量记忆和图增强记忆（Mem0g）两种模式。集成广泛（21 个官方集成覆盖 LangChain、CrewAI、AutoGen、OpenAI Agents SDK、Google ADK 等 13 个 agent 框架）。2025 年 6 月起支持 actor-aware memory（在多 agent 系统中标记记忆来源）。LoCoMo benchmark 上 Mem0g 准确率 68.4%，P95 延迟 2.59 秒。

**Letta（原 MemGPT）**：UC Berkeley 研究项目转化的生产平台。核心创新是 OS 式三层架构——core memory（始终在 context 内，类似 RAM）、recall memory（对话历史，可搜索）、archival memory（长期存储，类似磁盘）。区别于其他框架的关键特征是 **agent 主动管理自己的记忆**——通过调用 memory management function 决定什么换入换出，而非被动接收外部注入。2025-12 在 Terminal-Bench 上排名 #1 model-agnostic 开源 agent。

**Zep**：专注于 agent 记忆基础设施，Graphiti 框架是其核心——构建基于 Neo4j 的时序感知知识图谱。强项是结构化的关系推理和多跳查询。

**LangMem**：LangChain 生态的开源记忆方案。

**生态格局**：碎片化。没有单一赢家，这驱动了记忆系统需要广泛集成而非绑定特定平台的需求。向量存储后端选择多达 19 种（Qdrant、Chroma、Weaviate、Milvus、PGVector、Pinecone 等）。语音 agent 是增长最快的记忆需求场景——因为用户无法手动检索上下文，记忆失败在实时语音交互中立刻暴露。

### 评测层：benchmark 的快速成熟

记忆评测在 2025-2026 年经历了从无到有的爆发：

| Benchmark | 年份 | 关键特征 | 暴露的问题 |
|-----------|------|---------|-----------|
| **LoCoMo** | 2024 | 35 个 session、300+ 轮对话；事实/事件/对话任务 | 时序和因果推理维度不足 |
| **MemBench** | 2025 | 区分事实记忆 vs 反思记忆；参与 vs 旁观两种模式 | 增加了效率维度（不只看准确率） |
| **MemoryAgentBench** | 2025 | 四项认知能力：精确检索、测试时学习、远程理解、选择性遗忘 | 没有系统能掌握全部四项；选择性遗忘普遍失败 |
| **MemoryArena** | 2026 | 多 session 互依赖的 agentic task（网页导航、偏好约束规划等） | 从 LoCoMo 的 80%+ 骤降至 40-60%——被动回忆 ≠ 主动运用 |

MemoryArena 的发现最具冲击力：它证明了**被动回忆和主动运用之间存在巨大鸿沟**。现有 benchmark 大多测"能不能想起来"，但 agent 真正需要的是"能不能在对的时刻用对的记忆做对的事"。

### 企业实践

LinkedIn 的 Cognitive Memory Agent（CMA）是目前公开的最有影响力的企业级记忆架构案例。它是一个跨应用的共享记忆基础设施层，支撑 Hiring Assistant 等产品，提供情景、语义、程序三层记忆以及多 agent 协调能力。CMA 反映了一个更广泛的架构转向：从无状态生成到有状态、记忆驱动的 agent 设计。

## 演进路径

- **阶段 1（2020-2022）**：记忆 = 对话历史。把前几轮对话拼进 prompt，没有持久化，没有选择性。context window 很小（4K-8K），记忆约等于滑动窗口。
- **阶段 2（2023-2024）**：外部存储工程化。RAG 范式把记忆外置到向量库，LangChain 等框架提供 ConversationBufferMemory / ConversationSummaryMemory 等标准化组件。MemGPT（2023-10）提出 OS 式分层记忆，引入"agent 自主管理记忆"的范式。ChatGPT Memory（2024-02）标志着产品层开始整合持久记忆。
- **阶段 3（2025-2026，当前）**：记忆工程成熟化。三个已落地的并行趋势：（1）框架竞争和生态整合（Mem0、Letta 等框架在生产环境中使用，21+ 集成）；（2）图记忆从实验进入生产（Graphiti + Neo4j，有实际部署）；（3）评测体系爆发（MemBench → MemoryAgentBench → MemoryArena，逐步逼近真实 agentic 场景）。学术前沿方面，策略学习型管理（A-Mem）和记忆安全作为研究方向出现，但尚无生产落地。
- **趋势**：gen4 假设——记忆从工程外挂向地基内化。但截至 2026-04，这个假设的三个判定标准（API 层原生持久记忆、外部向量库使用下降、memory eval 广泛采用）都还没有明确满足。更现实的近期趋势是：（1）选择性遗忘成为下一个工程焦点（当前所有系统的短板）；（2）记忆安全从纯学术研究开始进入框架开发者的视野（但距离生产级治理仍远）；（3）记忆系统从单 agent 扩展到多 agent 共享记忆（Mem0 的 actor-aware memory 是第一步）。

## 对四层骨架的影响

### 第一层（Agent 内核）
- **塌缩**：固定的 system prompt 作为"记忆"的做法逐步被动态记忆系统替代；纯对话历史拼接不再是可接受的记忆方案
- **涌现**：agent 自主记忆管理成为内核能力（MemGPT 范式）；记忆与 context engineering 的融合（何时加载哪些记忆进 context）；agent "个体性"问题（有持久记忆的 agent 是否还是同一个模型的不同实例？）

### 第二层（与世界的接口）
- **塌缩**：简单的 ConversationBufferMemory / ConversationSummaryMemory 被更成熟的框架取代
- **涌现**：记忆作为独立的基础设施层（LinkedIn CMA 模式）；向量存储 vs 图存储的选型成为关键工程决策；MCP 生态中的记忆服务标准化（OpenMemory MCP）

### 第三层（多 Agent 协作）
- **塌缩**：（尚未发生，但 gen4 预期）无状态 agent 间的协作模式将被有状态协作重塑
- **涌现**：actor-aware memory（标记记忆来源 agent，Mem0 已实现）；共享记忆的一致性和冲突解决；多 agent 记忆的安全隔离问题（学术研究中有"蠕虫式传播"的理论攻击路径，尚无实际案例）

### 第四层（生产化）
- **涌现**：记忆评测（MemoryArena 等）作为 agent 质量的关键维度；记忆安全和隐私治理（GDPR 被遗忘权、记忆审计）；记忆可观测性（agent 记住了什么？记忆质量如何变化？）；记忆过时检测和更新策略

## 与其他维度的耦合

**× Long Context（维度 2）**：最常混淆的一对。长上下文能伪装记忆但不是记忆。两者也存在实际的工程互动：context window 越大，可加载的记忆越多，检索精度的要求相对降低；但 context window 的成本也越高。长上下文让简单的 RAG 场景塌缩，但不替代真正的记忆系统。

**× Reasoning（维度 1）**：reasoning + memory 的组合催生"能反思过去并改进"的 agent。反思式自我改进（Reflexion、Generative Agents）正是这个交叉点。这也是 gen4 的核心机制假设——如果 agent 能推理自身历史，memory 就不只是存储，而是学习的基础。第三代的 reasoning 成本问题如果未解决，会成为 memory 的瓶颈，因为记忆检索和整合同样消耗 inference compute。

**× Tool Use（维度 4）**：工具使用历史是 agent 记忆的重要组成部分。一个有记忆的 agent 应该记得调用过哪些工具、结果如何——这影响后续的工具选择。MCP 生态中的记忆服务（OpenMemory MCP）正在标准化这个接口。

**× Inference-Time Compute（维度 7）**：记忆的检索、整合、反思都消耗推理时计算。如果 memory 走向内化，推理时计算消耗可能进一步增长。

**× Instruction Following（维度 5）**：当检索到的记忆和用户当前指令矛盾时，agent 该遵循哪个？这是 instruction following 在记忆语境下的新变体。详见 [dimensions/05-instruction-following.md](05-instruction-following.md)

## 关键不确定性

**参数记忆是否会成为可行路线？** 知识编辑（ROME/MEMIT/WISE）目前是学术活跃但工程落地极少的状态。如果突破精确性和可逆性的瓶颈，将根本改变记忆系统的架构——不再需要外部存储和检索。但也可能这条路永远不会成为主流，非参数路线足够好。

**记忆内化还是记忆标准化？** gen4 假设是记忆向地基内化。但另一种可能是：记忆不内化，而是作为外部基础设施高度标准化（MCP 化），效果等价于内化。如果 MCP 记忆服务足够成熟、厂商原生记忆 API 足够好，"内化"的必要性可能被消解。

**选择性遗忘能否被工程化解决？** 这是当前所有系统的共同短板。遗忘不只是删除——需要理解信息间的依赖关系（删除一个事实后，基于它推导的结论也应该失效）。这可能需要维度 1 Reasoning 的进一步突破。

**记忆安全的治理框架何时成熟？** 学术界已有治理框架的提案（如一篇 2026-04 survey 提出的"记忆主权"概念），但没有任何生产系统实现了系统性的记忆安全治理。这可能成为 agent 记忆从 demo 到企业生产的最大阻碍。

**被动回忆 vs 主动运用的鸿沟能否弥合？** MemoryArena 暴露的 80%→40-60% 的骤降说明，记忆系统的瓶颈不在"存储和检索"，而在"在对的时刻用对的方式运用记忆"。这可能需要 reasoning 和 memory 的更深度融合，而非单纯优化检索。

## 信息源

### 综合调查

- [Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers](https://arxiv.org/abs/2603.07670) — 2026-03，最全面的 agent 记忆 survey，提出三维分类法和五种机制家族
- [A Survey on the Memory Mechanism of Large Language Model-based Agents](https://dl.acm.org/doi/10.1145/3748302) — ACM TOIS，系统性的记忆机制分类
- [From Storage to Experience: A Survey on the Evolution of LLM Agent Memory Mechanisms](https://www.preprints.org/manuscript/202601.0618) — 演进框架：存储 → 反思 → 经验
- [Memory in the Age of AI Agents: A Survey (Paper List)](https://github.com/Shichun-Liu/Agent-Memory-Paper-List) — 持续更新的论文列表

### 安全

- [A Survey on the Security of Long-Term Memory in LLM Agents: Toward Mnemonic Sovereignty](https://arxiv.org/html/2604.16548v1) — 2026-04，六阶段威胁分类和"记忆主权"框架
- [MemoryGraft: Persistent Compromise of LLM Agents via Poisoned Experience Retrieval](https://arxiv.org/abs/2512.16962) — 通过伪造成功经验持久操纵 agent 行为
- [InjecMEM: Memory Injection Attack on LLM Agent Memory Systems](https://openreview.net/forum?id=QVX6hcJ2um) — 记忆注入攻击
- [A-MemGuard: A Proactive Defense Framework for LLM-Based Agent Memory](https://openreview.net/forum?id=fVxfCEv8xG) — 首个主动防御框架

### 关键系统和框架

- [State of AI Agent Memory 2026](https://mem0.ai/blog/state-of-ai-agent-memory-2026) — Mem0 团队的行业状态报告，含 LoCoMo benchmark 对比数据
- [Letta (MemGPT) Documentation](https://docs.letta.com/concepts/memgpt/) — OS 式记忆架构的官方文档
- [A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110) — NeurIPS 2025，Zettelkasten 原则 + RL 优化的自组织记忆
- [Graphiti: Knowledge Graph Memory for an Agentic World](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/) — Neo4j 上的图记忆实现

### 评测

- [MemBench (ACL 2025)](https://aclanthology.org/2025.findings-acl.989/) — 区分事实记忆 vs 反思记忆的 benchmark
- [MemoryAgentBench (ICLR 2026)](https://github.com/HUST-AI-HYZ/MemoryAgentBench) — 四项认知能力评测，暴露选择性遗忘短板
- [ICLR 2026 Workshop: MemAgents](https://iclr.cc/virtual/2026/workshop/10000792) — ICLR 2026 记忆 workshop

### 企业实践

- [Designing Memory for AI Agents: Inside LinkedIn's Cognitive Memory Agent (InfoQ)](https://www.infoq.com/news/2026/04/linkedin-cognitive-memory-agent/) — LinkedIn CMA 架构

### 参数记忆 / 知识编辑

- [ROME: Locating and Editing Factual Associations in GPT](https://rome.baulab.info/) — 单条知识精确编辑
- [MEMIT: Mass Editing Memory in a Transformer](https://memit.baulab.info/) — 批量知识编辑

## 更新日志

- 2026-05-01：耦合章节增加维度 5 Instruction Following 交叉引用链接
- 2026-04-27：初次创建。覆盖四类记忆分类、参数/非参数分野、记忆生命周期、五种机制家族、图记忆、记忆安全、当前技术格局（产品/框架/评测/企业）、演进路径、四层影响、六个维度耦合、五个关键不确定性
