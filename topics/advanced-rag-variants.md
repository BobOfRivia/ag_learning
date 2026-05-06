# 进阶 RAG 变体（Advanced RAG Variants）

> 锚点：层级 L2（与世界的接口）/ 维度 2 Long Context / 维度 1 Reasoning

## 这个概念是什么

RAG（Retrieval-Augmented Generation）从"检索 top-K 块然后拼进 prompt"的朴素形态出发，在 2024-2026 年间分化出多个成熟度层级。每一层都不是新发明，而是对上一层失败模式的工程响应——naive RAG 在生产中约 40% 的检索失败率推动了 advanced RAG 的预/后检索阶段；advanced RAG 的管线僵化推动了 modular RAG 的可插拔架构；静态管线无法处理多步推理需求推动了 agentic RAG。

独立成 topic 的理由：L2 骨架文件里"RAG 模式"是一个小节，只能用几段话概括四个层级。但每个层级都有自己的内部结构（什么是预检索、什么是 RRF 融合、agentic RAG 的 ReAct 循环如何工作）、关键生产模式（混合检索、交叉编码器重排序、语义分块），以及 2026 年的落地数据。把这些放进骨架会喧宾夺主，独立成 topic 才能讲清楚。

这个 topic 不覆盖 RAG 的整体生态判断（"RAG vs 长上下文"的分层协作、企业 RAG 部署增长 280%）——那部分留在 L2 和 dimensions/02-long-context.md。这里聚焦"如果你要做 RAG，技术上有哪些成熟度选择"。

## 内部结构

### 四层架构分类

2026 年的 RAG 实践可以分为四个成熟度层级。它们是演进关系而非互斥关系——advanced RAG 包含 naive 的所有元素，agentic RAG 通常构建在 modular 的可插拔架构之上。

#### 1. Naive RAG

**机制**：用户查询 → embedding 检索 top-K 块 → 拼入 prompt → 生成回答。这是 2023 年早期 RAG 的事实形态，所有教程都从这里讲起。

**生产现状**：在生产场景中 **已淘汰**。约 40% 的检索失败率使其不可接受——查询和文档之间的词汇不匹配、单一相似度度量遗漏语义相关性、固定 top-K 在简单和复杂查询上同样僵硬。

**残存场景**：原型验证、教学演示。当工程师说"用 RAG 解决"时，他们通常意指 advanced RAG 或更高，naive RAG 已经不再是默认选项。

#### 2. Advanced RAG

**机制**：在 naive 的检索-生成两阶段之间，加入预检索（pre-retrieval）和后检索（post-retrieval）阶段。

**预检索阶段**——查询变换是核心。
- **查询扩展**：把原始查询改写为 2-3 个变体（同义词替换、视角转换），分别检索后合并结果，解决用户查询和文档措辞不匹配的问题。
- **HyDE（Hypothetical Document Embeddings）**：让 LLM 先根据查询生成一个"假设的答案文档"，用这个假设文档的 embedding 去检索。比直接用查询 embedding 更接近文档分布，对短查询尤其有效。
- **查询路由**：判断查询类型（事实型 / 推理型 / 摘要型），路由到不同的检索策略。

**后检索阶段**——重排序和过滤。
- **交叉编码器重排序（cross-encoder reranking）**：对初始 top-20 或 top-50 结果用一个更强但更慢的模型做精排。bi-encoder（用于初步检索）独立编码查询和文档，cross-encoder 把两者拼接后联合编码，能捕捉 bi-encoder 遗漏的细粒度语义关系。这是投入产出比第二高的改进。
- **元数据过滤**：基于时间、来源、权限等结构化字段过滤检索结果。
- **去重和聚合**：消除冗余块，合并来自同一文档的相邻块。

**生产现状**：**生产部署，当前最常见的层级**。混合检索（BM25 + 向量搜索，通过 Reciprocal Rank Fusion 融合）是这一层的事实标准——实测 recall@10 达 91%，显著优于纯向量（78%）和纯稀疏（65%）。所有主流向量数据库现在都内建混合搜索能力，这不再是差异化特性而是入门门槛。

#### 3. Modular RAG

**机制**：把 RAG 管线分解为可插拔模块——分块器、检索器、重排器、生成器、评估器——每个模块可独立替换、组合、优化。

**核心价值**：advanced RAG 把预/后检索阶段固化在管线里，要切换重排器或加入查询路由需要重写整个流程。modular RAG 把这些步骤抽象为统一接口下的可替换组件，让管线本身成为配置而非代码。

**典型实现**：LlamaIndex 和 Haystack 的架构天然支持这种模式——LlamaIndex 的 QueryPipeline、Haystack 的 Pipeline DAG。LangChain 的 LCEL（LangChain Expression Language）也朝这个方向演进，但抽象层更厚。

**生产现状**：**框架可用，生产采用中**。在需要持续迭代检索质量的团队中是主流选择——能 A/B 测试不同重排器、能给不同查询类型配不同管线、能把新论文里的检索技巧快速接入。

**与 advanced 的关系**：modular RAG 不是替代 advanced RAG，而是它的工程化形态。一个成熟的 modular 系统里跑的仍然是 advanced RAG 的那套预/后检索逻辑，只是模块化重组。

#### 4. Agentic RAG

**机制**：引入一个 agent 控制器，由它自主决策——何时检索、用什么查询、检索结果是否够用、是否需要重新检索、何时停止。控制循环用 ReAct（Reason + Act）实现：agent 推理出"我需要查 X"，调用检索工具，看到结果后再推理"还缺 Y"，继续检索，直到信息充分才生成最终回答。

**与前三层的根本区别**：naive/advanced/modular 都是固定管线——查询进、答案出，中间步骤数固定。agentic RAG 的步骤数是动态的，由查询复杂度决定。简单事实查询可能 1 步检索就结束，多跳推理可能需要 3-5 轮检索。

**Adaptive RAG 作为最佳实践**：纯 agentic RAG 的成本控制是个真问题——简单查询也走完整 ReAct 循环是浪费。Adaptive RAG 是解决方案：前置一个查询分类器，把简单查询路由到轻量管线（甚至直接生成不检索），把复杂查询路由到完整的 agentic/graph 检索。这让 agentic RAG 的成本和延迟变得可控。

**生产现状**：**企业生产部署，但失败模式跨推理步骤复合，调试更难**。Microsoft、Google 和大型企业的复杂检索场景普遍采用。失败诊断比固定管线难——需要追踪每一步推理、每次检索的查询和结果、判断是哪一步出了问题。这推动了 L4 层的 RAG 可观测性需求。

**与多 agent 的边界**：agentic RAG 通常是单 agent 自循环，不是多 agent 协作。但当检索任务足够复杂时（跨多个知识库、需要并行检索后合成），会延伸到 L3 的多 agent 编排。这两层的边界正在模糊。

### 关键生产模式（概览）

每种关键技术已独立成 topic，包含算法机制、模型对比、调优参数：

- **混合检索**（BM25 + 向量 + RRF）→ [topics/rag-hybrid-retrieval.md](rag-hybrid-retrieval.md)
- **重排序**（bi/cross-encoder，BGE / Jina v3 / Cohere Rerank 4 对比）→ [topics/rag-reranking.md](rag-reranking.md)
- **查询变换**（HyDE / Multi-Query / Step-Back / Adaptive 路由）→ [topics/rag-query-transformation.md](rag-query-transformation.md)
- **分块策略**（固定 / 递归 / 语义 / Late Chunking / Contextual Retrieval）→ [topics/rag-chunking.md](rag-chunking.md)
- **Agentic RAG 技术路线**（ReAct / Self-RAG / CRAG / Adaptive RAG）→ [topics/agentic-rag.md](agentic-rag.md)

**73% 失败在检索环节**：这是贯穿所有层级的核心生产判断——RAG 失败时约 73% 根因在检索阶段而非生成阶段。工程投资应优先放在检索质量（混合搜索、重排序、分块策略），而非生成端 prompt 优化。

## 当前状态（截至 2026-05）

四层之间的产业分布：原型层有少量 naive；生产主流是 advanced（带混合检索 + 重排序）；持续迭代检索质量的团队走 modular；复杂多跳场景走 agentic（通常以 adaptive 形态出现以控制成本）。

GraphRAG 是这个分类之外的正交维度——它解决基于块检索无法处理多跳关系推理的根本短板，可以叠加在 advanced/modular/agentic 任何一层之上。详见 [layers/L2-world-interface.md](../layers/L2-world-interface.md) 的"图 + 向量融合"小节。

## 关键权衡

**复杂度 vs 检索质量**：从 naive 到 agentic，检索质量阶梯式上升，工程复杂度也阶梯式上升。盲目追求最高层级是常见错误——大多数企业知识库查询用 advanced RAG 已经足够，agentic 的额外复杂度只对真正需要多步推理的场景有正收益。

**延迟 vs 准确率**：交叉编码器重排序提升准确率但增加 100-300ms 延迟；agentic RAG 多轮检索把单次响应从秒级推到 10 秒以上。生产系统需要按查询类型分层——简单查询走轻量路径，复杂查询走完整路径，即 Adaptive RAG 的核心思想。

**成本 vs 召回率**：top-K 增大、查询变体增多、重排序候选集扩大，都能提升召回率但线性推高成本。一个常见的生产配比是初始检索 top-50、重排序保留 top-10、最终送入 prompt top-5。

**调试难度的非线性增长**：固定管线（naive/advanced）的失败诊断相对直接——查检索结果、查 prompt、查输出。agentic RAG 的失败诊断需要追踪整个推理-检索循环，每一步的查询变换都可能引入误差，跨步骤的误差会复合。这把 L4 层的可观测性投资变成必需而非可选。

## 信息源

- [LaRA: Benchmarking RAG and Long-Context LLMs — OpenReview](https://openreview.net/forum?id=CLF25dahgA) — 2326 测试用例的 RAG vs 长上下文对照，确认无普适赢家
- [SoK: Agentic RAG — arXiv](https://arxiv.org/abs/2603.07379) — Agentic RAG 的形式化分类：agent 基数、控制结构、自主级别、知识表示
- [RAG Production Guide 2026 — LushBinary](https://lushbinary.com/blog/rag-retrieval-augmented-generation-production-guide/) — 2026 RAG 生产实践指南，覆盖混合检索、重排序、73% 检索失败定律
- [RAG Techniques Compared: Practical Guide 2026 — Starmorph](https://blog.starmorph.com/blog/rag-techniques-compared-best-practices-guide) — 各种 RAG 技术的实践对比，含 Adaptive RAG 的查询路由模式
- [Best RAG Frameworks 2026 — iTernal](https://iternal.ai/blockify-rag-frameworks) — LangChain vs LlamaIndex vs DSPy vs Haystack 的 RAG 能力对比，含 modular RAG 实现差异

## 更新日志

- 2026-05-06：瘦身"关键生产模式"章节，各技术细节已拆分到独立 topic（混合检索 / 重排序 / 查询变换 / 分块 / Agentic RAG）
- 2026-05-05：从 L2-world-interface.md 拆分独立成文，承载四层架构分类和关键生产模式的深度展开
