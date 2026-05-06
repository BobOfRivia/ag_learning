# RAG 重排序（Reranking）

> 锚点：层级 L2（与世界的接口）/ 维度 4 Tool Use

## 这个概念是什么

重排序（Reranking）是 RAG 管线的后检索阶段：先用 bi-encoder 做快速召回（top-50 ~ top-100），再用一个更精准但更慢的 cross-encoder 对候选集做精排（top-10 ~ top-20），最终送入 prompt 的是精排后的 top-5 左右。它是 RAG 管线里投入产出比排名第二的单项改进（第一是混合检索），本质是用"二阶段计算成本"换取"召回端遗漏的语义相关性"。

独立成 topic 的理由：bi-encoder vs cross-encoder 的架构差异、主流重排模型的技术对比（BGE / Jina / Cohere / Voyage）、listwise vs pairwise vs pointwise 三种打分范式各有独立的内部结构和取舍逻辑。

## 内部结构

### 架构核心：为什么需要两阶段

**Bi-encoder**（召回阶段）的工作方式是把查询和文档各自独立编码成向量，在向量空间用 ANN（近似最近邻）搜索。优势是**可以预计算文档 embedding**，查询时只需编码查询一次，检索延迟与语料库大小几乎无关（毫秒级）。代价是查询和文档的编码完全独立，模型无法在编码过程中感知到两者之间的交互关系——余弦相似度只是两个向量在空间里的几何距离，无法捕捉"这篇文档用了不同的词表达了和查询完全一致的意思"这种细粒度语义匹配。

**Cross-encoder**（精排阶段）的工作方式是把查询和文档**拼接在一起**送入 encoder，让 attention 机制在两者之间充分交互，最后输出一个相关性评分。这种"联合编码"能捕捉到 bi-encoder 遗漏的细粒度语义匹配，精排质量显著更高。代价是**无法预计算**——每次查询都需要对所有候选文档逐一计算一遍，推理成本是 O(候选集大小)。这就是 cross-encoder 不能做召回的原因：对百万级语料库逐一计算是不现实的。

两阶段管线的设计逻辑：bi-encoder 用低成本把相关文档从百万候选中召回到 top-50 ~ top-100，cross-encoder 在这个小集合上精排。候选集越小，cross-encoder 越快；候选集越大，召回率越高。典型的生产配比是初始召回 top-50，精排后保留 top-10。

### 打分范式：Pointwise / Pairwise / Listwise

这三种范式描述 cross-encoder 如何给候选文档打分，是理解重排模型技术路线的基础框架。

**Pointwise**：对每个文档独立打出一个与查询的相关性分数（0-1 分），按分数排序。大多数早期 cross-encoder 都是 pointwise。实现最简单，但分数没有文档间的对比信息。

**Pairwise**：对每对文档判断哪个更相关，把成对比较的结果聚合成排名（如 TrueSkill 算法）。学习信号比 pointwise 更丰富，但计算复杂度是 O(n²)，候选集稍大就不实用。

**Listwise**：同时处理整个候选列表，在列表层面做优化。比 pairwise 效率更高，且能捕捉文档之间的相对关系。Jina Reranker v3 采用的是 listwise 方案，是 2025 年之后的主流方向。

### 主流重排模型对比（截至 2026-05）

**BGE Reranker 系列（BAAI）**

BGE-reranker-v2-m3 是当前最常被部署的开源重排模型。基于 bge-m3 做 LoRA 微调，278M 参数，支持 100+ 语言，BEIR benchmark 上 nDCG@10 达 51.8，最后一次更新是 2025 年 8 月。关键优势是**极低部署成本**：单张 A10 GPU 可流畅运行，延迟约 50-100ms（top-50 候选集），开源可自托管。bgereranker-v2-minicpm-layerwise 提供了层级动态截断能力——可以用更少的 transformer 层换取更低延迟，是对延迟敏感场景的替代选项。

落地程度：**生产部署，开源最常用的重排模型**。

**Jina Reranker v3（Jina AI，2025-09，AAAI 2026）**

jina-reranker-v3 是当前 BEIR 上最强的开源重排模型之一，61.94 nDCG@10，比 v2 提升 4.88%。技术路线是"last but not late interaction"——基于 Qwen3-0.6B（0.6B 参数，28 transformer 层），通过同一 context window 内的 causal self-attention 让查询和多个文档同时交互（listwise），从最后一个 token 提取各文档的 contextual embedding，再用轻量 MLP 投影（1024→512→256）打分。支持同时处理 64 个文档，context window 131K tokens。

多语言 MIRACL 上 66.50（18 语言平均），多跳检索 HotpotQA 78.56，事实验证 FEVER 93.95，代码检索 CoIR 63.28。

落地程度：**框架可用，生产采用中（2025 年末发布）**。

**Cohere Rerank 4（2025-12）**

Rerank 4 是 Cohere 最新一代，两个变体：rerank-v4.0-pro（最高质量）和 rerank-v4.0-fast（低延迟）。核心升级是 **32K context window**，是 v3.5 的 4 倍——使得长文档（合同、报告、代码文件）可以整段送入排名，而无需切碎再拼接。支持 JSON 文档重排（结构化数据不再需要序列化为文本）。最重要的差异化是**自学习能力**：Rerank 4 能在无需标注数据的情况下基于用户行为自适应调整，这在企业垂直域快速冷启动上有明显优势。

在金融、医疗、制造业 benchmark 上强于 Voyage Rerank 2.5 和 Jina v3（Cohere 自测数据）。Cohere Rerank 3.5 的指标是金融服务数据集上比混合检索高 23.4%，比 BM25 高 30.8%——v4 在此基础上进一步提升。

落地程度：**生产部署（API 服务）**。

**Voyage Rerank 2.5（MongoDB / Voyage AI）**

Voyage AI 在被 MongoDB 收购后的重排产品。强项是长文档和多语言场景，作为 MongoDB Atlas Vector Search 的原生集成被广泛使用。性能在 Jina v3 和 Cohere Rerank 4 的发布后被追平，仍是生态内的常用选项。

落地程度：**生产部署（API 服务）**。

**LLM-as-Reranker**

直接用 GPT-4 / Claude 等大模型对候选集打分，prompt 中给出查询和文档，让模型输出相关性评分或排名。优点是不需要专门的重排模型、能利用大模型的推理能力处理复杂查询。缺点是成本是专用重排器的 10-100 倍，延迟高（1-5 秒 vs 50-200ms），且大模型的评分一致性不如专用模型。

当前主要用途：（1）为专用重排模型生成训练标注数据（GPT-4 打分后蒸馏到小模型）；（2）对极高价值查询做 fallback 精排（agentic 场景）。落地程度：**有限生产使用，主要作为数据生成和 fallback**。

### 重排在管线中的位置和参数调优

**候选集大小的权衡**：初始召回 top-20 和重排是一种组合，top-50 和重排是另一种。召回 top-20 再重排延迟低但召回率差（关键文档可能在 top-20 之外）；召回 top-100 再重排召回率高但重排延迟线性增加。实测中 top-50 是大多数场景下的最优权衡起点，对延迟不敏感的场景可以扩大到 top-100。

**重排后取多少个进 prompt**：通常 top-5 是生产默认，超过 top-10 的文档数增加 context 成本但对答案质量的边际收益快速递减。

**重排 vs 不重排的实测数据**：根据多个 RAG 评测，加入 cross-encoder 重排后，端到端问答准确率典型提升 5-15 个百分点，具体幅度取决于初始召回质量和任务类型。召回质量越差（初始检索不准），重排的提升幅度越大；初始检索已经很准时，重排的边际收益下降。

**重排器的自托管 vs API**：BGE-reranker-v2-m3 单卡可运行，月处理量大时自托管明显比 Cohere/Voyage API 便宜。月处理量小或需要快速上线时，API 是阻力最小的选择。

## 当前状态（截至 2026-05）

cross-encoder 重排是 Advanced RAG 的标准组件，生态已成熟。技术演进方向在两端：开源端向 listwise 架构（Jina v3 路线）演进；闭源 API 端向自学习和长文档支持（Cohere Rerank 4 路线）演进。LLM-as-reranker 的研究仍在推进，但在生产成本可接受之前不太可能替代专用重排模型。

## 关键权衡

**开源自托管 vs 闭源 API**：低流量冷启动 → 闭源 API；高流量稳定场景 → 开源自托管（BGE-reranker-v2-m3 性价比最优）；需要自适应微调 → Cohere Rerank 4。

**候选集大小**：top-50 是延迟/召回的起点。延迟 SLO 宽松（>2s 可接受）→ 扩大到 top-100；严格（<500ms）→ 缩减到 top-20 并评估是否还需重排。

**listwise vs pointwise**：Jina v3 的 listwise 方案在多文档对比类查询（"以下文档中哪个最相关"）上有明显优势；pointwise 在候选集质量已经很高时差距不大。

## 信息源

- [Jina Reranker v3: 0.6B Listwise Reranker for SOTA Multilingual Retrieval — Jina AI](https://jina.ai/news/jina-reranker-v3-0-6b-listwise-reranker-for-sota-multilingual-retrieval/) — v3 技术细节和 benchmark 数据
- [jina-reranker-v3: Last but Not Late Interaction for Document Reranking — arXiv](https://arxiv.org/pdf/2509.25085) — v3 的 AAAI 2026 论文
- [BAAI/bge-reranker-v2-m3 — Hugging Face](https://huggingface.co/BAAI/bge-reranker-v2-m3) — 模型卡，包含 benchmark 和部署说明
- [Introducing Cohere Rerank 4.0 — Cohere](https://cohere.com/blog/rerank-4) — Rerank 4 官方发布，32K context、自学习、Pro/Fast 变体
- [Cohere's Rerank 4 quadruples the context window — VentureBeat](https://venturebeat.com/orchestration/coheres-rerank-4-quadruples-the-context-window-to-cut-agent-errors-and-boost) — 第三方评测和市场分析
- [Best Rerankers for RAG Leaderboard — Agentset](https://agentset.ai/rerankers) — 多模型重排器对比 leaderboard
- [Top 5 Reranking Models to Improve RAG Results — MachineLearningMastery](https://machinelearningmastery.com/top-5-reranking-models-to-improve-rag-results/) — 各模型实用对比

## 更新日志

- 2026-05-06：初次创建，从 topics/advanced-rag-variants.md 拆分，覆盖 bi/cross-encoder 架构、pointwise/pairwise/listwise 范式、BGE/Jina v3/Cohere Rerank 4/Voyage 对比、调优参数
