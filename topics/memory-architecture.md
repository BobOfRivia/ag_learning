# 记忆架构模式（Memory Architecture Patterns）

> 锚点：层级 L2（与世界的接口）/ 维度 6 Memory / 与 topics/context-engineering.md 有直接关联

## 这个概念是什么

记忆架构是"如何给 agent 装上跨 session 的持久记忆"的工程核心问题。它包含两个正交的设计决策：**用什么机制管理记忆**（五种机制家族）和**用什么基底表示记忆**（向量 vs 图 vs 混合）。这两个轴的组合构成了当前所有生产记忆系统的设计空间。

独立展开的理由：维度文件中每种机制只能写一段，但每种都有完整的技术路线、代表系统、适用场景和失败模式。且 2026 年生产实践已从单一架构转向混合架构（双层模式），需要更深的分析。

## 内部结构

### 五种机制家族

这个分类来自 2026-03 的综合调查（arxiv 2603.07670），它们不是互斥的——生产系统通常组合使用。

#### 1. 上下文驻留压缩（Context-Resident Compression）

在 context window 内部解决记忆问题，不引入外部存储。

**技术路线**：滑动窗口（保留最近 N 轮）→ 滚动摘要（每次溢出时摘要旧内容）→ 层级压缩（多层摘要形成树状结构）→ 锚定迭代摘要（维护持久结构化锚点，只摘要新增内容后合并，Factory.ai 在 36000 条消息评测中达到 4.04/5 准确率）→ 观察遮蔽（JetBrains 方案：保留 agent 推理和动作，用占位符替换旧的环境观察结果，在 4/5 场景中优于 LLM 摘要，成本降低 52%）。

**核心限制**：**摘要漂移（summarization drift）**——多次压缩后，罕见但关键的细节逐步丢失。每次压缩都是有损操作，信息损失随压缩次数累积。这个问题在长时间运行的 agent 中尤其严重。

**适用场景**：单 session 对话、不需要跨 session 持久化的场景。是其他所有机制的"兜底"——即使有外部存储，agent 的 context window 管理仍然需要压缩策略。

**与 context engineering 的关系**：这个机制家族和 context engineering（topics/context-engineering.md）的"压缩"操作完全重叠。区别在于视角——context engineering 从"管理当前 context 里有什么"出发，记忆架构从"跨时间积累信息"出发。在上下文驻留压缩这个交叉地带，两者是同一件事的两个名字。

#### 2. 检索增强存储（Retrieval-Augmented Stores）

将记忆存入外部数据库（通常是向量库），需要时通过语义检索注入 context。

**技术路线**：基础向量检索（embedding + cosine similarity）→ 多粒度索引（按 chunk/document/session 不同粒度建索引）→ 两阶段检索（粗检索 + 精排 reranking）→ 元数据过滤（在语义检索之上叠加结构化属性查询）。

**检索的核心张力**：Mem0 的 LoCoMo benchmark 数据清晰展示了准确率-效率权衡：

| 方法 | 准确率 | P95 延迟 | Token 消耗 |
|------|--------|---------|-----------|
| 全量上下文 | 72.9% | 17.12s | ~26,000 |
| Mem0g（图增强） | 68.4% | 2.59s | ~1,800 |
| Mem0（向量） | 66.9% | 1.44s | ~1,800 |
| RAG | 61.0% | 0.70s | — |
| OpenAI Memory | 52.9% | — | — |

全量上下文准确率最高但在实时生产环境中不可用（17 秒延迟）。选择性记忆以 6 个百分点的准确率代价换取 91% 的延迟降低。这个权衡是所有检索增强系统的基本面。

**瓶颈转移**：RETRO 证明了从万亿 token 级语料中检索的技术可行性，存储不再是限制因素。瓶颈已完全转移到**检索相关性**——不是"能不能存"，而是"能不能在对的时间找到对的东西"。

**适用场景**：当前最主流的生产方案。适合记忆量中到大（数百到数万条）、需要跨 session 持久化、对延迟有一定容忍度的场景。

#### 3. 反思式自我改进（Reflective Self-Improvement）

Agent 对自身经验进行反思，提炼出抽象规则、教训和模式。不只是存储过去发生了什么，而是从过去学到什么。

**技术路线**：事后总结（verbal post-mortem）→ 观察聚类（将相似经验归纳）→ 对比规则提取（从成功和失败中提炼出可泛化的规则）。

**代表系统和数据**：
- **Reflexion**（2023）：agent 在失败后进行语言层面的自我反思，生成改进策略并在下次尝试中应用。HumanEval 上达到 91% pass@1。
- **Generative Agents**（Stanford，2023）：25 个 agent 模拟小镇生活。关键发现是——**没有反思机制的 agent 在 48 小时内退化为重复的、无上下文的回复**。反思是维持有意义行为的必要条件，不是可选优化。
- **Voyager**（2023）：技能库（skill library）存储成功完成任务的可执行程序。有程序记忆的 agent 技术树推进速度快 **15.3 倍**。这是程序记忆的代表案例——记住"怎么做"比记住"发生了什么"更有直接的行为价值。

**核心风险**：**自我强化的错误**。Agent 从一次错误经验中提炼出一条错误规则，这条规则在后续交互中不断被"验证"（因为 agent 按照错误规则行动，环境反馈恰好不足以纠正），最终固化为根深蒂固的错误信念。调查中称之为"overgeneralization"——从有限经验中过度泛化。

**适用场景**：需要长期改进的 agent（coding agent、游戏 agent、个人助手）。通常不单独使用，而是与检索增强存储组合——反思产出的规则和教训存入外部存储，需要时检索。

#### 4. 层级虚拟上下文（Hierarchical Virtual Context）

受操作系统内存管理启发，在 context window（RAM）和外部存储（磁盘）之间建立显式的分层和换入换出机制。

**Letta（原 MemGPT）的三层架构**：

**Core Memory**——始终在 context 内，嵌入 system instructions。包含标签（identifier）、描述（behavioral influence）和值（content）的结构化 memory block。相当于操作系统的常驻内存——agent 的身份、用户的核心偏好、当前任务的关键参数。大小有限且可配置。

**Recall Memory**——对话历史的持久存储。当 context 溢出时，Letta 不是简单丢弃旧内容，而是递归摘要被驱逐的内容并存入 recall storage。通过搜索工具可以找到之前的对话片段。

**Archival Memory**——长期外部存储。存放不常访问的大量数据：文档、策略、知识库、偏好历史。支持外部向量数据库和语义检索。相当于磁盘。

**核心创新**：和其他所有架构的关键区别在于——**agent 主动管理自己的记忆**。Agent 通过调用 `core_memory_append`、`core_memory_replace`、`memory_insert`、`memory_rethink` 等 memory function 来决定什么信息换入 context、什么写入外部存储、什么需要重新组织。它不是被动接收外部系统注入的上下文，而是自己决定记忆策略。这是一个根本性的范式转变：从"系统替 agent 管记忆"到"agent 管自己的记忆"。

**Heartbeat 机制**：Letta 的 agent 有一个内部的 heartbeat 循环，允许多步推理和后台更新。Agent 可以在一次交互中连续调用多个 memory function，执行复杂的记忆管理操作。

**生产状态**：从 UC Berkeley 研究项目转化为生产软件，model-agnostic，开源。2025-12 在 Terminal-Bench 上排名 #1 model-agnostic 开源 agent。2026-01 发布 Conversations API 支持跨并行体验的共享记忆。

**适用场景**：需要细粒度记忆控制和长期自我改进的 agent。尤其适合需要"agent 了解自己知道什么、不知道什么"的场景——因为 agent 自己在管理记忆，它对自身记忆状态有显式的感知。

#### 5. 策略学习型管理（Policy-Learned Management）

用强化学习优化记忆操作本身——不是由人设计记忆策略，而是让 agent 通过训练学出最优的记忆管理行为。

**A-Mem**（NeurIPS 2025）是代表系统。它借鉴了 Zettelkasten 方法的原则：每条记忆是一个包含上下文描述、关键词、标签的结构化笔记，笔记之间通过动态索引和链接形成互连知识网络。当新记忆加入时，不只是被追加——系统会评估新记忆和已有记忆的关系，更新已有记忆的上下文表示和属性，让整个记忆网络持续自我精炼。

**通过 GRPO 训练**，A-Mem 将记忆的存储、检索、更新、摘要、丢弃视为 RL 动作。训练过程中发现了人类不太会想到的策略，比如"在 context 填满前进行预防性摘要"——在上下文压力出现之前就主动压缩和整理，而非等到被迫摘要时才行动。

**在六个基础模型上的实验**显示 A-Mem 优于现有 SOTA 基线。但这条路线的代价是训练复杂性和成本——需要针对特定任务训练 memory policy。

**适用场景**：对记忆管理质量要求极高且有训练预算的场景。目前更多是研究前沿，生产落地案例少于前四种机制。

### 表示基底：向量 vs 图 vs 混合

与五种机制家族正交的另一个设计轴是**记忆以什么形式存储**。

#### 向量记忆

将记忆 embed 为高维向量，通过余弦相似度检索。这是 2023-2025 年的主流方案，工具链最成熟（Pinecone、Qdrant、Chroma、Weaviate、Milvus、PGVector 等 19+ 后端）。

**优势**：语义匹配强、延迟低（Mem0 的 P95 仅 1.44 秒）、工具链成熟、概念简单。

**根本局限**：向量记忆是**无结构的相似性匹配**。它找到的是语义相近的记忆，不是逻辑相关的记忆。当你问"Alice 的经理是谁"时，向量搜索返回所有提到 Alice 的段落，但不一定包含答案——因为"经理"这个关系没有被显式建模。向量记忆无法做多跳推理（A 的经理 B 的部门在哪？）和时序推理（上次会议之后预算改了吗？）。

#### 图记忆

将记忆建模为实体（节点）和关系（边）的知识图谱。实体间的关系被显式表示，支持结构化查询。

**Zep/Graphiti 的架构**是当前图记忆的代表实现，建立在 Neo4j 之上，包含三层子图：

**Episode 子图（𝒢ₑ）**：存储原始对话数据，保持无损。每条消息或文本作为 episode 节点，连接到从中提取的语义实体。

**Semantic Entity 子图（𝒢ₛ）**：包含提取的实体和它们之间的关系。提取过程使用当前消息加前 4 条消息作为上下文，通过 embedding 余弦相似度和全文搜索定位候选，再用 LLM 做实体消歧。

**Community 子图（𝒢ₚ）**：高层级的实体聚类和摘要。使用 label propagation 算法动态扩展。

**时序模型**是 Graphiti 的核心差异点。它使用**双时间线（bi-temporal model）**：Timeline T 记录事件实际发生的时间（包括处理相对日期如"下周四"），Timeline T' 记录数据被系统摄入的时间。每条关系边都带有效性区间（t_valid / t_invalid）和系统区间（t'_created / t'_expired）。当新信息和已有信息冲突时，系统利用时序元数据更新或失效（而非删除）旧信息——旧事实不被丢弃，而是被标记为"在某个时间段内有效"。

**检索**使用三种互补搜索：语义搜索（cosine similarity）、关键词搜索（BM25）、图搜索（BFS 从相关节点出发的 n 跳遍历）。结果通过 Reciprocal Rank Fusion 或 cross-encoder 重排。最终检索约 10 个最相关的节点和边，构建平均约 1600 token 的上下文。

**性能数据**：在 LongMemEval 上，Zep + gpt-4o 达到 71.2% 准确率，比全量上下文的 60.2% 高出 18.5%。同时延迟从 28.9 秒降至 2.58 秒，context token 从 115K 降至 1.6K。在复杂查询类型上收益最大：single-session-preference 提升 184%，temporal-reasoning 提升 48.2%。但在简单的 single-session-assistant 问题上，准确率反而下降 17.7%——图记忆对简单问题有一定的过度结构化开销。

#### 混合架构：2026 的生产趋势

部分技术博客和框架文档（Hermes OS、部分 dev.to 文章）描述了一种"组合多种"的趋势，称为**双层架构（Dual-Layer Architecture）**。这个模式在概念上合理，但需要注意：其"生产趋势"的判断主要来自博客和框架推广材料，缺少大规模生产部署的公开案例数据。以下是其设计思路：

**热路径（Hot Path）**：最近 N 条消息加上压缩后的图状态，处理即时上下文。用的是上下文驻留压缩（机制 1）。

**冷路径（Cold Path）**：外部存储（向量库、图数据库或两者兼有），通过语义检索按需注入。用的是检索增强存储（机制 2）或图记忆。

在 agent 每轮交互完成后，一个后台 Memory Node 负责决定什么值得保存：提取事实、更新用户模型、基于任务结果创建 Skill Documents。大多数实现将这个过程异步化（Mem0 的 v1.0.0 默认 async mode），不阻塞推理管线。

更完整的生产栈是三层的：**向量记忆做快速模糊召回** + **情景缓冲区维持短期连贯** + **图数据库处理实体密集查询**，agent 根据查询类型在三者之间路由。

### 三种生产部署模式

2026-03 的学术 survey 将实践归纳为三种部署模式（这是学术分类，不是行业共识），复杂度递增：

**单体上下文（Monolithic Context）**：全部记忆留在 context window 内。简单，但受限于 context 大小和成本。适合短对话、记忆量小的场景。

**上下文 + 检索（Context + Retrieval）**：context window 管理近期记忆，外部存储管理历史记忆。这是当前的"主力方案"——大多数生产 agent 用这种模式。Mem0 的架构就属于这类。

**分层 + 策略学习（Tiered + Learned）**：Letta 式的多层架构，加上 A-Mem 式的策略优化。最大的提升空间，但也是最高的实现和运维成本。**落地程度：Letta 的分层部分有生产使用，策略学习部分尚无生产案例。**

## 当前状态（截至 2026-04）

### 框架对比

| 维度 | Mem0 | Letta | Zep |
|------|------|-------|-----|
| **核心范式** | 选择性提取 + 向量/图存储 | OS 式三层分页 | 时序知识图谱 |
| **记忆管理者** | 系统（LLM 判断提取什么） | Agent 自身（调用 memory function） | 系统（自动构建图） |
| **强项** | 广泛集成、易上手、个性化 | 细粒度控制、长期自我改进 | 时序推理、实体关系、多跳查询 |
| **弱项** | 复杂关系推理不如图方案 | 实现复杂度高 | 简单查询有过度结构化开销 |
| **LongMemEval 准确率** | 49.0%（gpt-4o） | — | 63.8%（gpt-4o），71.2%（gpt-4o） |
| **集成广度** | 21 个官方集成 | 较少，自成平台 | 以 Neo4j 为核心 |
| **开源** | 是 | 是 | Community Edition |

**选型建议（从开发者博客和框架对比文章中归纳，非权威结论）**：
- 需要个性化和用户偏好记忆 → Mem0
- 需要 agent 自主管理记忆、长期改进 → Letta
- 需要时序推理、实体关系、复杂查询 → Zep
- 生产环境 → 大概率是混合（向量 + 图 + 情景缓冲），而非单一选择

### 关键技术组件的成熟度

**向量检索**：成熟。工具链完善，19+ 后端选择。

**图记忆**：从实验进入生产。2024 年还是实验性的，2026 年在生产系统中有实际部署（Zep、Neo4j Aura Agent）。Neo4j 投入 1 亿美元支持 1000 家 AI 创业公司用图技术。

**Agent 自主记忆管理**：上升期。Letta 证明了可行性和收益，但范式尚未被广泛采纳。

**策略学习型管理**：研究前沿。A-Mem 在学术 benchmark 上表现优秀，但生产落地案例稀少。

**双层/混合架构**：概念上被广泛讨论，但"已成为生产默认"的判断缺少大规模数据支持。博客和框架文档中频繁出现，公开的生产案例仍有限。

## 关键权衡

**1. 准确率 vs 延迟 vs 成本**

这是最基本的三角权衡。全量上下文准确率最高但延迟和成本不可接受；纯向量检索最快最便宜但准确率有天花板；图记忆在复杂查询上准确率最高但增加了构建和维护成本。没有一种架构在三个维度上同时最优。

**2. 系统控制 vs Agent 自治**

Mem0 和 Zep 的模式是"系统替 agent 管记忆"——提取、存储、检索的决策由外部系统做出。Letta 的模式是"agent 管自己的记忆"——agent 通过 tool call 显式操作记忆。前者简单可预测，后者灵活但引入了 agent 记忆管理能力本身可能不靠谱的风险。

**3. 结构化 vs 非结构化**

向量记忆是非结构化的（一切变成 embedding），图记忆是高度结构化的（实体和关系被显式建模）。结构化方案在关系查询上占优势，但在输入端有更重的处理负担（实体提取、消歧、关系构建），且对简单查询有过度工程的问题。

**4. 写入成本 vs 读取质量**

图记忆在写入端做了大量工作（实体提取、关系建模、冲突检测），因此读取时检索质量高。向量记忆在写入端几乎无开销（embed 然后存），但读取时相关性不如图方案。这是一个"前期投入 vs 后期收益"的经典权衡。

**5. 通用性 vs 领域适配**

通用的记忆框架（Mem0、Letta）适配广泛但不针对任何特定领域优化。A-Mem 的策略学习路线可以针对特定任务优化记忆策略，但需要训练投入且不可迁移。LinkedIn 的 CMA 走的是另一条路——为企业特定用例（Hiring Assistant）定制记忆架构。

## 信息源

### 综合调查

- [Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers](https://arxiv.org/abs/2603.07670) — 五种机制家族的分类来源
- [State of AI Agent Memory 2026](https://mem0.ai/blog/state-of-ai-agent-memory-2026) — Mem0 团队的行业报告，LoCoMo benchmark 数据

### 代表系统

- [Letta (MemGPT) Documentation](https://docs.letta.com/concepts/memgpt/) — OS 式三层记忆架构
- [Stateful AI Agents: A Deep Dive into Letta Memory Models](https://medium.com/@piyush.jhamb4u/stateful-ai-agents-a-deep-dive-into-letta-memgpt-memory-models-a2ffc01a7ea1) — Letta 技术细节
- [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956) — Graphiti 的时序知识图谱架构和 benchmark 数据
- [A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110) — Zettelkasten + RL 的自组织记忆，NeurIPS 2025
- [Graphiti: Knowledge Graph Memory for an Agentic World](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/) — Neo4j 上的图记忆实践

### 架构对比

- [AI Agent Memory Systems in 2026: Zep, Mem0, Letta, and Dual-Layer Architectures](https://hermesos.cloud/blog/ai-agent-memory-systems) — 双层架构趋势和框架对比
- [5 AI Agent Memory Systems Compared (2026 Benchmark Data)](https://dev.to/varun_pratapbhardwaj_b13/5-ai-agent-memory-systems-compared-mem0-zep-letta-supermemory-superlocalmemory-2026-benchmark-59p3) — 跨框架 benchmark 对比
- [Mem0 vs Letta (MemGPT): AI Agent Memory Compared](https://vectorize.io/articles/mem0-vs-letta) — 两大框架的直接对比

### 企业实践

- [Designing Memory for AI Agents: Inside LinkedIn's CMA](https://www.infoq.com/news/2026/04/linkedin-cognitive-memory-agent/) — 企业级记忆基础设施

## 更新日志

- 2026-04-27：初次创建。覆盖五种机制家族、向量 vs 图 vs 混合表示基底、Zep/Graphiti 时序架构细节、三种生产部署模式、框架对比（Mem0/Letta/Zep）、五个关键权衡
