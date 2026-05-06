# RAG 查询变换（Query Transformation）

> 锚点：层级 L2（与世界的接口）/ 维度 1 Reasoning / 维度 5 Instruction Following

## 这个概念是什么

查询变换是 RAG 的预检索阶段：在把查询送入检索系统之前，先对它做变换——扩展、改写、分解、或抽象——以弥合"用户的查询语言"和"文档的写作语言"之间的语义距离。它解决的核心问题是**词汇不匹配（vocabulary mismatch）**：用户用口语化的提问，文档用专业术语写作；或者用户查询太短、意图不完整，向量空间里离真正相关文档很远。

独立成 topic 的理由：HyDE / Multi-Query / Step-Back / Sub-Question / Adaptive Routing 五种技术有各自完全不同的机制、适用场景和失败模式，每一种都值得单独理解。

## 内部结构

### HyDE（Hypothetical Document Embeddings）

**机制**：Gao 等人 2022 年提出（arXiv 2212.10496）。不直接用查询的 embedding 做检索，而是先让 LLM 根据查询生成一段"假设答案文档"，然后用这个假设文档的 embedding 做检索。

**为什么有效**：这个技术的核心洞察是 embedding 空间中的**分布不对称**问题。训练 bi-encoder 时，正负样本是（查询, 文档）对——查询的 embedding 和文档的 embedding 是在同一个目标下优化的，但两者的分布在空间里未必对齐。短查询（"如何提高员工积极性？"）在向量空间里的坐标离相关长文档往往比较远，因为查询的语言模式和文档的语言模式本身就不同。HyDE 的假设文档用的是"文档语言"写出来的文本，embedding 分布和真实文档更接近，因此检索更准确。

**适用场景**：短查询、零样本信息检索（没有大量训练数据）、查询意图模糊（用假设文档"展开"意图）。

**失败模式**：当 LLM 的领域知识不足时，生成的假设文档可能包含错误事实——用一份错误的假设文档做检索会召回错误的文档，并进一步传导到最终回答（hallucination 放大）。在领域知识高度专业化（法律、医疗、金融）的场景，HyDE 的风险尤其需要评估。

落地程度：**框架可用（LangChain HyDE chain、LlamaIndex HyDEQueryTransform），生产中选择性使用**。

### Multi-Query（多查询变体）

**机制**：用 LLM 根据原始查询生成 N 个同义词变体查询（通常 3-5 个），分别检索，结果合并去重后进入后续阶段。LangChain 的 `MultiQueryRetriever` 是最常见的实现。

**为什么有效**：单一查询在向量空间里是一个点，不同的表达方式可能在空间里散布开来，各自命中不同但相关的文档。多查询扩展了检索的覆盖范围，等价于从多个角度对同一个问题做检索。

**成本**：每次检索多一次 LLM 调用（生成变体）+ N 次检索调用（N 通常是 3-5）。总延迟增加约 200-500ms（变体生成），适合延迟不敏感场景。

**实现细节**：去重是关键步骤——不同变体可能检索到相同文档，需要在融合前去重；融合算法一般用简单 union（取所有变体结果的并集后截取 top-K）或 RRF。LangChain 的实现是 union，LlamaIndex 提供 RRF 选项。

落地程度：**生产部署，实现简单，是使用最广泛的查询变换方法**。

### Step-Back Prompting

**机制**：Google DeepMind 2023 年底提出（arXiv 2310.06117）。面对具体的、细节化的查询，先让 LLM 生成一个"更高层抽象"的查询（step back），用抽象查询检索更通用的背景知识，再把原始查询和背景知识一起送入最终生成。

**示例**：
- 原始查询："爱因斯坦 1905 年发表了什么论文？"
- Step-back 查询："爱因斯坦的主要科学贡献是什么？"
- 逻辑：先检索广泛的背景知识，再在这个背景下回答具体问题。

**适用场景**：多跳推理查询（回答一个具体问题需要先建立背景认知）、专业知识领域的问答（具体事实查询需要领域框架支撑）。

**与 Multi-Query 的区别**：Multi-Query 是横向扩展（同一层次的不同表达方式），Step-Back 是纵向抽象（从具体问题到更高层概念）。两者可以组合使用。

落地程度：**框架可用，但需要更多 LLM 调用，生产采用比 Multi-Query 少**。

### Sub-Question Decomposition（子问题分解）

**机制**：对复杂的多跳查询，先让 LLM 将其分解为若干个独立的子问题，并行检索各个子问题，最后将所有子问题的结果合并，统一生成最终回答。LlamaIndex 的 `SubQuestionQueryEngine` 是参考实现。

**示例**：
- 原始查询："比较 GPT-4 和 Claude 3 在代码生成和数学推理上的表现"
- 子问题 1："GPT-4 在代码生成上的表现如何？"
- 子问题 2："Claude 3 在代码生成上的表现如何？"
- 子问题 3："GPT-4 在数学推理上的表现如何？"
- 子问题 4："Claude 3 在数学推理上的表现如何？"

**与 Agentic RAG 的区别**：Sub-Question Decomposition 是**静态规划**——在检索之前就把问题分解好，子问题的数量和内容在分解时固定。Agentic RAG 的多步检索是**动态推理**——每一步检索的结果决定下一步检索什么，是自适应的。前者适合结构清晰、可预知子问题的查询；后者适合需要探索性推理的查询。

落地程度：**框架可用，生产中用于结构化多跳查询场景**。

### 查询路由（Query Routing）

**机制**：在查询进入检索之前，先对其分类，根据类型路由到不同的检索策略或知识源。比如：事实类问题 → 向量数据库；结构化数据查询 → SQL 生成；最新事件 → Web 搜索；多文档综合 → 图检索。

**分类器实现方式**：
- **小型 LLM 分类**：将查询送给一个小模型（Llama 3.1 8B 或更小），让它输出分类标签。速度较快，延迟约 50-100ms。
- **Embedding 分类器**：将查询 embedding 送入训练好的分类头，延迟极低（<10ms），但需要标注数据训练。
- **关键字规则**：用正则或关键词匹配做简单分类（"最新"→web 搜索，"图表"→SQL），零延迟，但覆盖率有限。

**Adaptive RAG（Jeong et al., NAACL 2024）** 是查询路由的系统性理论化版本：训练一个小型分类器预测查询的复杂度（简单/中等/复杂），分别路由到"不检索直接生成"、"单步检索"、"多步 agentic 检索"三种管线。这是在保持召回质量的同时控制成本的关键设计，详见 [topics/agentic-rag.md](agentic-rag.md)。

落地程度：**生产部署，尤其在多知识源（向量库 + SQL + Web）的混合检索系统中**。

### RAG Fusion：结合多查询与 RRF

**机制**：一种综合方案——生成多个查询变体，分别检索，用 RRF（Reciprocal Rank Fusion）融合排名，再送重排器精排。LangChain 有 `RAGFusion` 链的参考实现。

**与 Multi-Query 的关系**：RAG Fusion 是 Multi-Query + RRF 融合的组合，而非简单的 union 合并。RRF 比 union 更好地处理了不同变体检索结果的置信度加权。

落地程度：**框架可用，实际生产中逐渐成为 Multi-Query 的升级选项**。

## 当前状态（截至 2026-05）

Multi-Query 是生产中采用最广泛的查询变换技术，实现简单且效果稳定。HyDE 在零样本场景有独特价值，但需要小心领域知识不足的风险。Step-Back 和 Sub-Question 适用于特定查询类型，通常作为 Multi-Query 的补充而非替代。Query Routing 是多知识源系统的必要组件。

**Reasoning 模型的影响**：有一个重要的趋势正在发酵——o3、Claude Opus 4.x 等 reasoning 模型内置了规划和多步推理能力，模型自己会"决定先查什么、再查什么"。这在一定程度上让显式的查询变换变得可选：如果 agentic RAG 中使用了 reasoning 模型，它会自发地做类似 Sub-Question 分解和 Step-Back 的推理。但 Multi-Query 和 Query Routing 的价值相对稳定——前者是召回层的覆盖扩展（与模型推理无关），后者是知识源的路由决策（需要显式配置）。

## 关键权衡

**变换质量 vs LLM 调用成本**：每一种查询变换都需要一次或多次额外的 LLM 调用。延迟敏感场景（实时对话）通常只能承受 Multi-Query（一次额外调用）；批量处理场景可以叠加 HyDE + Multi-Query + Step-Back。

**覆盖率 vs 噪声**：生成更多变体提高召回率，但变体质量差时会引入噪声文档，冲淡相关文档的比例，最终可能降低生成质量。

**HyDE 的风险收益**：通用领域收益高风险低，专业垂直领域风险高（需要评估 LLM 的领域知识质量后决定是否启用）。

## 信息源

- [Precise Zero-Shot Dense Retrieval without Relevance Labels (HyDE) — arXiv](https://arxiv.org/abs/2212.10496) — HyDE 原始论文，Gao et al. 2022
- [Take a Step Back: Evoking Reasoning in LLMs via Abstraction — arXiv](https://arxiv.org/abs/2310.06117) — Step-Back Prompting 论文，Google DeepMind 2023
- [Adaptive-RAG: Learning to Adapt Retrieval-Augmented LLMs through Question Complexity — ACL Anthology / NAACL 2024](https://aclanthology.org/2024.naacl-long.389/) — Adaptive RAG 论文（Jeong et al.），复杂度分类器路由
- [MultiQueryRetriever — LangChain Docs](https://python.langchain.com/docs/modules/data_connection/retrievers/MultiQueryRetriever/) — LangChain 的 Multi-Query 实现
- [SubQuestionQueryEngine — LlamaIndex Docs](https://docs.llamaindex.ai/en/stable/examples/query_engine/sub_question_query_engine/) — LlamaIndex 的子问题分解实现
- [RAG Techniques Compared: Practical Guide 2026 — Starmorph](https://blog.starmorph.com/blog/rag-techniques-compared-best-practices-guide) — 各变换技术的实践对比，含 Adaptive RAG 路由说明

## 更新日志

- 2026-05-06：初次创建，从 topics/advanced-rag-variants.md 拆分，覆盖 HyDE / Multi-Query / Step-Back / Sub-Question / Query Routing / RAG Fusion 六种技术
