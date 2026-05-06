# RAG 混合检索（Hybrid Retrieval）

> 锚点：层级 L2（与世界的接口）/ 维度 2 Long Context / 维度 4 Tool Use

## 这个概念是什么

混合检索把两种在设计上互补的检索范式融合在一个管线里：BM25 的精确关键词匹配（稀疏检索）和向量检索的语义相似度匹配（稠密检索）。它是投入产出比最高的单项 RAG 改进，recall@10 实测达 91%，显著优于纯向量（78%）和纯 BM25（65%）。

独立成 topic 的理由：BM25 算法、向量检索的 ANN 索引、RRF 融合公式各自有值得深入理解的机制，三者合并后的参数调优也有足够的工程细节，不是骨架文件里一段话能覆盖的。

## 内部结构

### BM25：稀疏检索的工程机制

BM25（Best Match 25）是 TF-IDF 的改进版，1994 年由 Robertson 等人提出，至今仍是稀疏检索的行业基线。它的核心改进是对词频引入了饱和函数——在 TF-IDF 中，词出现 100 次比出现 10 次在相关性评分上的差异是线性的；BM25 通过 k₁ 参数引入饱和：

```
BM25(d, q) = Σ IDF(qᵢ) × [tf(qᵢ, d) × (k₁ + 1)] / [tf(qᵢ, d) + k₁ × (1 − b + b × |d| / avgdl)]
```

关键参数的语义：**k₁**（默认 1.2-2.0）控制词频饱和速度——k₁ 越大，词频增加带来的评分提升越持久；**b**（默认 0.75）控制文档长度归一化强度——b=1 完全按文档长度归一化，b=0 不归一化。这两个参数在不同语料上有不同的最优值，通常需要在验证集上做超参搜索。

BM25 的优势是**字面匹配精确、可解释性强、无需 GPU 推理、索引更新成本低**。它的根本局限是词汇不匹配（vocabulary mismatch）：用户用"汽车"查询，文档里写"轿车"，BM25 打零分。这个局限正是稠密检索的切入点。

### 向量检索（Dense Retrieval）：稠密检索的工程机制

向量检索用 embedding 模型把查询和文档各自映射到高维向量空间，通过向量相似度（余弦相似度或点积）做检索。它的核心能力是**语义相似度**：即使查询词和文档词完全不重叠，只要语义相似，就能检索到。

**Bi-encoder 架构**是向量检索的标准架构：查询和文档各自独立编码（两个共享权重的 encoder），得到各自的 embedding 向量，在索引时预计算文档 embedding 并存储，查询时只需对查询编码一次，然后做向量检索。这使得检索延迟与语料库大小几乎无关（检索是 ANN 近似最近邻操作）。

**ANN 索引工程权衡**：
- **HNSW（Hierarchical Navigable Small World）**：基于图结构的 ANN 算法。查询速度最快（毫秒级），内存占用大（所有向量存在内存），不支持增量更新（添加新文档需重建图）。Qdrant 和 Weaviate 默认使用。
- **IVF-PQ（Inverted File + Product Quantization）**：先用聚类把向量空间划分为 Voronoi 单元（IVF），再用乘积量化压缩向量（PQ），大幅减少内存。适合百亿级别向量但牺牲部分召回率。Faiss 的主力索引类型。
- **pgvector 的 HNSW**：PostgreSQL 扩展，50M 向量以下性价比最优——Timescale 的 pgvectorscale 实测 471 QPS（Qdrant 同场景 41 QPS），99% recall 下。

**当前主流 Embedding 模型（截至 2026-05 MTEB 排名）**：

闭源方面：OpenAI text-embedding-3-large（3072 维，支持 MRL 可变维度截断）、Cohere embed-v4.0（多语言 + 多模态）、Google Gemini Embedding 2（文本/图像/视频/音频/PDF 五模态）、Voyage AI 系列（voyage-3-large 在代码和多语言有强项）。

开源方面：阿里 Qwen3-Embedding-8B 正在逼近闭源水平（MTEB 排名前列），BAAI BGE-M3（100+ 语言，单张 A10 可运行，dense + sparse + multi-vector 三种检索模式），intfloat/e5-mistral-7b-instruct（仍是长文本检索的强选）。

### RRF（Reciprocal Rank Fusion）：融合算法

RRF 是混合检索的标准融合方法，由 Cormack 等人 2009 年提出。它不使用绝对评分，只使用排名位置，因此可以把稀疏检索（BM25 分数）和稠密检索（余弦相似度）这两个量纲不同的分数统一融合：

```
RRF(d) = Σ 1 / (k + rank_i(d))
```

其中 **k=60** 是标准默认值（也是原始论文的推荐值）。k 的作用是防止极高排名的文档过度主导：当 k=60 时，排名第 1 的文档得分是 1/61≈0.0164，排名第 10 的文档是 1/70≈0.0143，差距被压缩；如果 k 很小（如 k=1），第 1 名和第 10 名的差距就会被放大。60 这个值经验上在大多数检索场景下表现稳定。

RRF 的优点是**无需训练**（不需要调整各路检索的权重参数），任意数量的排名列表都可以融合，对单路检索的错误具有一定的鲁棒性。它的局限是无法利用检索评分的绝对大小——当一路检索对某个文档非常确定（评分远高于其他候选）时，RRF 会压平这种信号。

**加权融合的替代方案**：直接对归一化后的 BM25 分数和余弦相似度做加权求和（常见权重是 0.5/0.5 或 0.3 BM25 + 0.7 向量）。这种方式需要调参，但能利用评分的绝对大小信息。在有标注数据时，学习融合权重（Linear Combination with Learned Weights）通常比 RRF 更优，但需要监督数据。

### 生产实现：各向量库的混合检索差异

混合检索在技术上需要同时运行 BM25 索引（通常用 Lucene/Elasticsearch 的倒排索引）和向量索引，然后做 RRF 融合。各主流向量库的实现差异：

- **Weaviate**：混合搜索是第一公民设计，alpha 参数（0-1，0 纯 BM25，1 纯向量）直接在查询 API 中配置，内建 BM25F（字段加权 BM25）。
- **Qdrant**：通过 sparse vectors + dense vectors 并行检索实现，支持 SPLADE 学习型稀疏向量替代传统 BM25。RRF 是内建支持。
- **Elasticsearch / OpenSearch**：Lucene 提供 BM25，dense_vector 字段提供向量检索，Hybrid Scoring API 做 RRF 融合。Elasticsearch 是生产场景下历史最成熟的方案。
- **pgvector + pgvectorscale**：向量检索强，BM25 需要结合 PostgreSQL 全文索引（tsvector + tsrank）实现，融合在应用层做，工程复杂度较高但可行。
- **Pinecone**：2024 年后原生支持稀疏-稠密混合，一个 namespace 里可以同时存稀疏和稠密向量，在查询时指定融合权重。

### 学习型稀疏检索：SPLADE 和 BM42

SPLADE（Sparse Lexical and Expansion Model）用 BERT 做 backbone，对词表中每个 token 生成稀疏权重，相当于"学习型 BM25"——它能自动做词汇扩展（"汽车"→["轿车", "车辆", "驾驶"]），解决了传统 BM25 的词汇不匹配问题。BEIR benchmark 上 SPLADE 稳定优于 BM25。

但当前**生产中完全替换 BM25 的采用率仍低**。主要障碍是推理成本（SPLADE 需要 transformer 推理，BM25 是纯 CPU 倒排索引）和索引大小（SPLADE 的稀疏向量相比 BM25 倒排索引更大）。目前生产中的常见模式是：BM25（稀疏）+ 稠密向量 + RRF，而非 SPLADE + 稠密 + RRF。Qdrant 提供了 SPLADE 支持，但实际采用案例主要集中在对检索质量要求极高且有 GPU 资源的场景。落地程度：**框架可用，生产采用有限**。

### 参数调优

**alpha 参数**（向量检索权重）的直觉：事实类、精确匹配查询（"某法律条文的具体内容"）偏向 BM25（alpha 低）；模糊语义查询（"关于激励员工的方法"）偏向向量（alpha 高）。生产中常见的起点是 alpha=0.5，然后在验证集上二分搜索。

**候选集大小**：两路各检索多少候选？常见配置是各检索 top-50 到 top-100，RRF 融合后取 top-20，再送重排器精排到 top-10，最后 top-5 进入 prompt。候选集越大召回率越高，融合和重排的延迟也越高——通常 top-50 是最佳权衡起点。

**不同查询类型的差异**：代码检索中精确 token 匹配比语义匹配更关键（BM25 权重偏高）；医疗领域的专业术语检索中 BM25 表现差（"心肌梗塞"vs"心脏病发作"）向量权重需偏高；多语言检索中 BM25 的跨语言能力几乎为零，几乎完全依赖稠密检索。

## 当前状态（截至 2026-05）

混合检索是 Advanced RAG 的事实标准，所有主流向量数据库都内建支持。它不再是差异化特性而是入门门槛。生产中的工程重点已从"是否用混合检索"转向"如何调优融合参数"和"是否升级到 SPLADE 类学习型稀疏检索"。

## 关键权衡

**BM25 权重 vs 向量权重**：没有通用最优解，取决于查询类型（精确 vs 模糊）和语料特点（专业术语密度）。在有标注数据前，alpha=0.5 是最保守起点。

**RRF vs 加权融合**：RRF 无需调参、鲁棒，适合冷启动；加权融合有潜力更优但需要标注数据调优。大多数生产系统从 RRF 开始，积累足够数据后再迁移。

**SPLADE vs BM25**：SPLADE 在召回质量上有优势，但引入 GPU 推理依赖。GPU 资源充裕且召回质量是硬需求时值得评估；否则 BM25 的运维成本和延迟优势难以放弃。

**候选集扩大 vs 重排延迟**：候选集 50→100 通常带来约 2-3% 的 recall 提升，但重排延迟线性增加。在延迟 SLO 严格的场景（<500ms），候选集上限通常是 top-50。

## 信息源

- [Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods — Cormack et al. 2009 (SIGIR)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) — RRF 原始论文，k=60 推荐值的来源
- [SPLADE for Sparse Vector Search Explained — Pinecone](https://www.pinecone.io/learn/splade/) — SPLADE 机制与 BM25 对比
- [Modern Sparse Neural Retrieval: From Theory to Practice — Qdrant](https://qdrant.tech/articles/modern-sparse-neural-retrieval/) — SPLADE 生产化的工程视角
- [BM42: New Baseline for Hybrid Search — Qdrant](https://qdrant.tech/articles/bm42/) — Qdrant 提出的 BM42 改进（BERT + BM25 hybrid idea）
- [Hybrid Search Explained — Weaviate](https://weaviate.io/blog/hybrid-search-explained) — Weaviate 混合搜索的 alpha 参数和 RRF 实现
- [RAG Production Guide 2026 — LushBinary](https://lushbinary.com/blog/rag-retrieval-augmented-generation-production-guide/) — 混合检索的 recall@10 对比数据（91% vs 78% vs 65%）
- [pgvectorscale Benchmark — Timescale](https://www.timescale.com/blog/pgvector-vs-pinecone/) — pgvectorscale 在 50M 向量场景的 QPS 对比

## 更新日志

- 2026-05-06：初次创建，从 topics/advanced-rag-variants.md 拆分，覆盖 BM25 算法、向量检索 ANN 索引、RRF 融合、生产实现差异、SPLADE 学习型稀疏检索
