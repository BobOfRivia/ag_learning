# RAG 文档分块策略（Chunking Strategies）

> 锚点：层级 L2（与世界的接口）/ 维度 2 Long Context

## 这个概念是什么

分块（Chunking）是 RAG 的数据预处理阶段：把原始文档切成小片段（chunk），对每个片段生成 embedding 存入向量库，检索时匹配最相关的片段送入 LLM。分块策略决定了片段的边界在哪里、大小是多少、上下文保留多少——它直接影响检索质量的上限，是"RAG 失败时 73% 根因在检索环节"的重要来源之一。

独立成 topic 的理由：从固定分块到语义分块到 Late Chunking，每种策略的机制、成本、适用场景完全不同，还有 Contextual Retrieval 这种正交的增强手段。理解分块是真正做好 RAG 的前提。

## 内部结构

### 为什么分块是 RAG 的关键瓶颈

分块存在两个方向的错误：**chunk 太大**时，embedding 需要表示过多信息，"主题漂移"导致相似度计算被无关内容干扰，top-K 检索时关键段落和噪声内容一起混入；**chunk 太小**时，单个 chunk 缺乏足够的上下文（"公司注册资本为人民币一亿元"——脱离文档结构后这句话的含义需要上下文），语义不完整，检索命中率降低。

典型的 chunk 大小争论是 256 / 512 / 1024 tokens：256 语义颗粒细但上下文少，1024 上下文丰富但 embedding 被稀释，512 是大多数生产系统的起点。没有普适最优——语料特点（段落长度）、查询类型（事实 vs 综合）、embedding 模型的输入长度上限，都影响最优 chunk 大小。

另一个常被忽视的问题是**边界切割**：固定大小分块会在句子中间切断，"cross-chunk sentences"的两半都变得语义不完整。Overlap 是最直接的缓解——让相邻 chunk 共享 10-20% 的 token，但这增加了存储开销和检索时的重复。

### 固定大小分块（Fixed-Size Chunking）

最简单的分块策略：按 token 数或字符数硬切，带可配置的 overlap（通常是 chunk 大小的 10-20%）。

**生产中的残存场景**：（1）语料是纯文本、没有明显结构，任何位置切都同样合理；（2）需要最低预处理成本的冷启动；（3）下游 embedding 模型对输入长度不敏感。除此之外，固定大小分块在实际评测中通常不是最优选项。

落地程度：**已生产（作为 baseline），大多数场景已被更好方案替代**。

### 递归字符分块（Recursive Character Text Splitting）

LangChain 的 `RecursiveCharacterTextSplitter` 让这种方法成为事实标准。它的逻辑是按"语义重要性"降序尝试分隔符，依次是：`\n\n`（段落）→ `\n`（换行）→ `。/./!/?`（句子）→ 空格 → 字符。优先在语义边界切割，当当前层次的边界让 chunk 超过目标大小时，回退到下一层次的分隔符。

相比固定大小分块，递归字符分块**保留了更多的自然语义边界**，不会在句子中间切断。但它仍是纯文本结构感知——它不知道文档是 Markdown 还是 PDF，不知道这是一个表格还是代码块。

落地程度：**生产部署，当前最常用的 baseline 分块策略**。

### 语义分块（Semantic Chunking）

**机制**：不按固定规则切，而是根据**语义连贯性**确定边界。具体实现通常是：先把文档按句子分割，对每个句子生成 embedding，计算相邻句子之间的 embedding 距离，在距离突变（相似度急剧下降）的地方确定 chunk 边界。

Greg Kamradt 的"Levels of Text Splitting"框架（2024 年初）系统化了这个思路，把文本分割分为 5 个层次：字符 → 递归字符 → 文档结构 → 语义（embedding）→ agent 决策。语义分块是第四层。LangChain 的 `SemanticChunker` 和 LlamaIndex 的 `SemanticSplitterNodeParser` 是两个主流实现。

**成本代价**：语义分块需要在**索引阶段**对整个文档做一次 embedding，用于确定 chunk 边界，然后再对切好的 chunk 做第二次 embedding 用于检索。这使得索引时间和成本至少翻倍。对大规模语料库（百万文档级别），这个成本是真实的工程负担。

**评测数据**：一项在多个 RAG 数据集上的评测（LangCopilot，2025-10）发现语义分块在准确率上优于递归字符分块，在 4/7 个数据集上有显著提升。但提升幅度因数据集差异较大（约 5-15 个百分点），对结构化文档（有明确 Markdown 标题）提升较小，对纯文本段落提升较大。

落地程度：**框架可用，生产中选择性使用（适合纯文本语料，成本可接受时）**。

### Document-Aware 分块

**机制**：按文档的**原始结构**切——Markdown 按标题层级切，PDF 按章节 / 页面 / 段落切，HTML 按 DOM 结构切，代码文件按函数 / 类切。关键是：这种分块方式能保证每个 chunk 在语义上是文档里的一个完整逻辑单元。

**特殊内容的处理**：
- **表格**：整个表格作为一个 chunk（而非按行切断），可以额外把表头信息作为 metadata 附加。
- **代码块**：保持函数或类的完整性，不在函数中间切断。
- **公式 / 图表**：图表通常无法直接 embedding，常见方案是（1）用 caption 作为文字代理，（2）用 multimodal 模型对图表生成描述文字。

**工具**：Unstructured.io 是处理 PDF / Word / HTML 等复杂格式的开源工具，能解析出文档结构（段落、表格、标题层级）。Docling（IBM 开源，2024）专注于 PDF 的高质量解析，包括表格的结构化提取。

落地程度：**生产部署，企业知识库（合同、报告、手册）的首选分块方式**。

### Parent-Child / Multi-Resolution 检索

**机制**：用**小 chunk 做检索，返回大 chunk（父文档）给 LLM**。具体做法是：把文档同时切成小 chunk（128-256 tokens，用于 embedding 检索）和大 chunk（512-1024 tokens 或完整段落，用于送入 LLM），两者通过 parent_id 关联。检索时用小 chunk 精准匹配，命中后拿出对应的大 chunk 送入 LLM 生成回答。

**为什么有效**：小 chunk 的 embedding 噪声少，检索更精准（信噪比高）；大 chunk 提供足够的上下文让 LLM 理解和推理。这是在"chunk 太大影响检索 / chunk 太小缺乏上下文"之间取中间值的优雅工程解法。

LangChain 的 `ParentDocumentRetriever` 是参考实现，存储两级文档：`child_splitter` 生成小 chunk 存入向量库，`parent_splitter`（或整个文档）存入 docstore，通过 parent_id 关联。

落地程度：**生产部署，是目前结合检索精度和上下文质量的最实用方案之一**。

### Late Chunking（Jina AI，2024）

**机制**：Late Chunking 提出了一种根本性不同的分块方式：**先整体编码，后分块取平均**。传统分块是先切 chunk 再各自独立 embedding——每个 chunk 在编码时感知不到其他 chunk 的内容，丢失了全局上下文（文档开头的定义对第 10 页的段落理解至关重要，但传统方式让第 10 页的 chunk 独立编码时看不到开头的定义）。

Late Chunking 的做法：把整个文档（或尽可能大的片段）一次性送入支持长上下文的 embedding 模型（如 jina-embeddings-v2，支持 8192 tokens），得到每个 token 的 contextual embedding，然后对属于同一 chunk 的 token embedding 做**平均池化**，得到该 chunk 的 embedding。这样每个 chunk 的 embedding 中都"包含"了文档其他部分的语义信息。

**评测数据**：Jina AI 原始论文（arXiv 2409.04701，2024-09，AAAI 2026 版本 2025-07 更新）报告：相比传统分块，Late Chunking 的平均相对提升是 2.7%（1.5% 绝对值），在句子边界分块上的相对提升约 3.6%。这是在三个 embedding 模型和四个数据集上的平均，不同数据集差异较大。

**局限**：需要支持长上下文的 embedding 模型（普通 BERT 类模型输入上限 512 tokens，完全不可行）。对超长文档（>8K tokens）仍需分段，只是在模型能处理的范围内做整体编码。Elasticsearch 已有 Jina 的 Late Chunking 集成教程，Milvus 也有对应集成。

落地程度：**框架可用，生产采用仍有限（2024 年末方案，需长上下文 embedding 模型支持）**。

### Contextual Retrieval（Anthropic，2024-09）

**机制**：这是对分块的**正交增强**而非新分块方式。传统分块后，每个 chunk 是文档的一个片段，脱离原始文档后语义可能不完整（"公司注册资本为人民币一亿元"——哪家公司？）。Contextual Retrieval 的做法是：在生成 embedding 之前，先用 Claude 为每个 chunk 生成一段上下文说明（chunk-specific context），说明这个 chunk 在整个文档中的位置和含义，然后把 chunk + 上下文说明一起 embedding。BM25 索引也做同样处理（Contextual BM25）。

**评测数据**：Anthropic 2024-09 的博客报告：Contextual Embeddings 将 top-20 检索失败率从 5.7% 降到 3.7%（35% 降幅）；Contextual Embeddings + Contextual BM25 组合将失败率进一步降到 2.9%（49% 降幅）；加入重排器后总体失败率降低 67%。

**成本**：为每个 chunk 生成上下文需要一次 LLM 调用，且需要把整个文档放入 context window。对大规模语料库（百万 chunk 级别），这是相当大的索引成本。Anthropic 建议结合 prompt caching——相同文档的多个 chunk 可以缓存文档内容，只需对每个 chunk 的位置指针部分重新推理，大幅降低成本。

落地程度：**生产部署（有 prompt caching 辅助时成本可接受），是高价值知识库 RAG 的重要增强手段**。

## 当前状态（截至 2026-05）

生产中的分块策略通常不是单选，而是按语料类型和工程成本的组合：
- 纯文本语料 → 递归字符分块（baseline）或语义分块（成本允许时）
- 结构化文档（PDF / Word / Markdown）→ Document-Aware 分块（Unstructured / Docling）
- 高价值知识库（需要极高检索质量）→ Parent-Child + Contextual Retrieval 组合
- 长上下文 embedding 模型已有 → 考虑 Late Chunking

## 关键权衡

**索引成本 vs 检索质量**：语义分块和 Contextual Retrieval 都需要额外的 LLM 调用（前者用 embedding 模型，后者用生成模型），索引成本显著高于固定/递归分块。高频更新的语料库（每天新增大量文档）应优先考虑低成本分块方案；相对静态的高价值知识库值得用高成本方案换取更高检索质量。

**chunk 大小与 overlap**：chunk 大小和 overlap 是任何分块策略都要调的超参，没有通用最优值。建议在自己的语料上用小样本做 ablation（测试不同配置的检索 recall@10）后再决定。

**是否用 Parent-Child**：单层 chunk 实现简单；Parent-Child 需要维护两套索引（向量库 + docstore），存在同步和一致性问题。语料相对静态时 Parent-Child 值得引入；高频更新时工程复杂度需要评估。

## 信息源

- [Late Chunking in Long-Context Embedding Models — Jina AI](https://jina.ai/news/late-chunking-in-long-context-embedding-models/) — Late Chunking 原始介绍，核心机制和评测数据
- [Late Chunking: AAAI 2026 论文 — arXiv](https://arxiv.org/pdf/2409.04701) — arXiv 2409.04701，Jina AI，最新版 2025-07
- [Contextual Retrieval — Anthropic](https://www.anthropic.com/news/contextual-retrieval) — Contextual Retrieval 官方博客，49% / 67% 失败率降低数据
- [Document Chunking for RAG: 9 Strategies Tested — LangCopilot](https://langcopilot.com/posts/2025-10-11-document-chunking-for-rag-practical-guide) — 9 种分块策略的实测对比（2025-10）
- [Reconstructing Context: Evaluating Advanced Chunking Strategies — arXiv](https://arxiv.org/pdf/2504.19754) — 2025-04 分块策略评测论文，含多种先进方案对比
- [Late Chunking in Elasticsearch with Jina Embeddings v2 — Elasticsearch Labs](https://www.elastic.co/search-labs/blog/late-chunking-elasticsearch-jina-embeddings) — Late Chunking 的 Elasticsearch 集成实践
- [Late Chunking: Balancing Precision and Cost in Long Context Retrieval — Weaviate](https://weaviate.io/blog/late-chunking) — Weaviate 对 Late Chunking 的权衡分析

## 更新日志

- 2026-05-06：初次创建，从 topics/advanced-rag-variants.md 拆分，覆盖固定分块、递归字符分块、语义分块、Document-Aware 分块、Parent-Child、Late Chunking、Contextual Retrieval
