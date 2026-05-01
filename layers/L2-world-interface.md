# 第 2 层：与世界的接口（World Interface）

## 这一层解决什么问题

Agent 内核（L1）决定 agent 怎么思考和规划，这一层决定它能接触什么、能做什么。L2 是 agent 的手和眼——它包含 agent 与外部世界交互的所有接口：调用工具、检索知识、操作系统、持久化记忆。

## 关键工程模式

### Tool Integration 模式

这是 L2 的核心子领域，有三种主要工程模式对应三种工具调用形态：

**结构化 API 集成**：通过 JSON schema 定义工具接口，模型生成结构化调用。这是最成熟的模式，MCP 正在将其标准化。工程重点在于：工具描述的质量（直接影响调用准确率）、schema 设计（参数类型、约束条件）、错误处理（工具调用失败后的恢复策略）。当前状态：**成熟**。

**CLI / Shell 集成**：通过 shell 命令操作系统。工程重点在于沙箱设计（限制破坏性操作）、输出解析（从非结构化文本中提取信息）、权限控制（哪些命令允许自动执行，哪些需要人工确认）。Coding agent 的事实标准是混合模式——关键操作用结构化工具，灵活操作用 shell。当前状态：**成熟但缺乏标准化**。

**Computer Use / GUI 集成**：通过视觉感知和鼠标键盘操作 GUI。工程重点在于截图频率与分辨率权衡、坐标映射准确性、操作确认机制、失败恢复（GUI 状态不可预测）。当前状态：**上升期，尚未成熟**。

### RAG 模式

第三代的长上下文 + reasoning model 让简单 RAG 大面积塌缩，但"RAG 已死"的叙事被市场数据反驳——企业 RAG 部署在 2025 年增长 280%，向量数据库类别在 2025 年吸引了 $8 亿以上的风投。2026 年的生产共识已从"RAG vs 长上下文"的对立收敛为**分层协作**：RAG 做精准检索预筛选，长上下文做全局推理。一家欧洲银行的对照实验发现，长上下文在简单查询上准确率高 34%，RAG 在跨文档时序综合上准确率高 67%。LaRA benchmark（2326 测试用例）确认不存在普适赢家——最优选择取决于模型、context 长度、任务类型和检索特征。RAG 残余价值的五因素判断框架详见 [dimensions/02-long-context.md](../dimensions/02-long-context.md) 的第二层影响章节。

**四层架构分类。** 2026 年的 RAG 实践可以分为四个成熟度层级：

**Naive RAG**——检索 top-K 块，拼入 prompt，生成回答。仍用于原型验证，但在生产中约 40% 的检索失败率使其不可接受。落地程度：**已淘汰**（生产场景）。

**Advanced RAG**——在 naive 基础上增加预检索（查询扩展、HyDE 假设文档嵌入）和后检索（交叉编码器重排序、元数据过滤）阶段。混合检索（BM25 + 向量搜索，通过 Reciprocal Rank Fusion 融合）是生产标准，实测 recall@10 达 91%，显著优于纯向量（78%）和纯稀疏（65%）。落地程度：**生产部署，当前最常见的层级**。

**Modular RAG**——将管线分解为可插拔模块（分块器、检索器、重排器、生成器），每个模块可独立替换和优化。LlamaIndex 和 Haystack 的架构天然支持这种模式。落地程度：**框架可用，生产采用中**。

**Agentic RAG**——agent 控制器自主决策何时检索、用什么查询、是否需要重新检索、何时停止。使用 ReAct 循环。涌现的最佳实践是 **Adaptive RAG**——查询分类器将简单查询路由到轻量管线，复杂查询路由到完整的 agentic/graph 检索。这解决了 agentic RAG 的成本控制问题。落地程度：**企业生产部署，但失败模式跨推理步骤复合，调试更难**。

**关键生产模式：**

混合检索（BM25 + 向量 + RRF 融合）是投入产出比最高的单项改进，已是所有主流向量数据库的标配能力。交叉编码器重排序（对初始 top-20/50 结果做精排）是第二高投入产出比的改进——弥补嵌入相似度遗漏的语义相关性。查询变换（生成 2-3 个查询变体）解决用户查询和文档之间的词汇不匹配。语义分块（按文档结构而非固定 token 数切分）在一项评测中将准确率从基线提升到 71%。

一个重要的生产发现：**RAG 失败时，73% 的原因在检索环节而非生成环节。** 这意味着 RAG 的工程投资应优先放在检索质量（混合搜索、重排序、分块策略），而非生成端的 prompt 优化。

### Memory 工程

维度 6 文件覆盖记忆的能力演进、机制分类和评测体系（详见 [dimensions/06-memory.md](../dimensions/06-memory.md)），这里聚焦 L2 层的工程实践——agent 如何在架构上集成记忆。

**两种集成范式。** 当前的 agent 记忆集成分为两种根本不同的范式，选择哪种决定了系统的架构复杂度和 agent 的自主性边界。

被动注入（Passive Injection）：系统在外部检索相关记忆，注入到 agent 的 context 中，agent 本身不感知记忆系统的存在。这是最简单的模式——记忆检索逻辑在 agent 之外，agent 看到的只是一段被塞进 system prompt 的文本。ChatGPT Memory、大多数 LangChain 的 ConversationBufferMemory/ConversationSummaryMemory 实现都属于这种范式。优势是实现简单、agent 侧零改造。劣势是 agent 无法主动请求特定记忆、无法决定什么值得记住。

主动管理（Active Management）：agent 自己调用记忆 function，决定什么时候读、什么时候写、什么换入换出。Letta（原 MemGPT）是这个范式的代表——agent 通过 core_memory_append / core_memory_replace / archival_memory_search 等 function call 主动管理三层记忆。这本质上是把记忆管理变成了一种 tool use。优势是 agent 能根据任务需求主动检索特定记忆，并决定什么信息值得持久化。劣势是增加了 agent 的认知负担和调用复杂度，记忆管理的失误会累积。

**存储后端的工程选型。** 记忆的存储后端选择与 RAG 的检索基础设施高度重叠，但有不同的权衡重点：向量存储（Qdrant、Chroma、pgvector 等）适合语义相似性检索，是当前默认选择；图存储（Neo4j + Graphiti）适合需要关系推理和多跳查询的场景——"Alice 的经理是谁"这类问题用图存储比向量检索准确得多；混合存储组合两者，但引入同步和一致性的工程复杂度。Mem0 的生态整合覆盖 19 种向量存储后端，反映了这个选择的碎片化现实。详见下方向量数据库章节。

**MCP 记忆服务。** OpenMemory MCP 等正在将记忆接口标准化——让任何 MCP client 能够以统一方式读写记忆，而非绑定特定框架。这与 MCP 在工具层的标准化逻辑一致：一次实现记忆服务，所有 agent 框架都能使用。当前仍在早期，但方向已明确。

**核心工程权衡。** Mem0 的 benchmark 数据量化了记忆工程的核心张力：全量上下文（把所有历史塞进 context）准确率 72.9% 但 P95 延迟 17 秒、token 消耗 26,000；选择性记忆（Mem0g 图增强）准确率 68.4% 但延迟仅 2.59 秒、token 消耗 1,800。4.5% 的准确率差距换来 6.5 倍的延迟改善和 14.4 倍的 token 节省——这个权衡在生产环境中几乎总是值得的。

## 工具与框架

### MCP（Model Context Protocol）

MCP 是 L2 层当前最重要的基础设施标准化事件。

**解决的问题**：在 MCP 之前，每个 agent 框架都有自己的工具定义格式——LangChain Tools、AutoGPT Plugins、Semantic Kernel Connectors——工具开发者需要为每个框架适配一次。MCP 提供了统一的协议：一个工具实现一次 MCP server，任何 MCP client 都能使用。

**当前生态（2026-04）**：月度 SDK 下载量 9700 万；独立索引的 MCP 服务器 17,000+；OpenAI、Google、Microsoft、Amazon 等均已支持。2025 年 12 月捐赠给 Linux Foundation 下的 Agentic AI Foundation（AAIF）。

**架构要点**：MCP server 暴露 tools（可调用的函数）、resources（可读取的数据）、prompts（可复用的提示模板）。传输层从最初的 stdio（本地进程）扩展到 Streamable HTTP（远程服务）。

**2026 路线图的四个方向**：
1. Streamable HTTP 规模化——有状态 session 与负载均衡的冲突、水平扩展方案
2. Tasks 原语生产化——长时间运行的工具调用的生命周期管理（重试、过期策略）
3. 治理成熟度——SEP（Spec Enhancement Proposals）流程、Working Groups
4. 服务器发现机制——标准化方式让 registry 或 crawler 了解一个 server 提供什么能力

**当前挑战**：服务器质量参差不齐、缺乏有效的信任和发现机制、安全模型（server 的权限边界）尚不成熟。

标注状态：**上升期（爆发增长中，但生产化基础设施仍在补齐）**

### A2A（Agent-to-Agent Protocol）

A2A 与 MCP 互补：MCP 标准化 agent → tool 的纵向连接，A2A 标准化 agent → agent 的横向通信。

**核心概念**：Agent Card（描述一个 agent 的能力和接口，v1.2 引入了加密签名用于域验证）、Task（agent 间协作的基本单位）、Message（task 内的通信）。

**当前状态（2026-04）**：v1.2，150+ 组织在生产环境使用，归入 AAIF 治理。由 Google 于 2025 年 4 月发起，Salesforce、SAP、Accenture 等首批 50+ 企业伙伴。

标注状态：**上升期（已有真实生产使用，但规模和成熟度不及 MCP）**

### Agent 框架

L3 文件覆盖了各框架的多 agent 编排能力（详见 [layers/L3-multi-agent.md](L3-multi-agent.md)），这里关注它们在 L2 层（工具集成、检索、世界接口）的定位和能力。

**当前格局（截至 2026-04）：**

| 框架 | 状态 | L2 层核心能力 | MCP | A2A |
|------|------|-------------|-----|-----|
| **LangGraph**（LangChain） | 成熟 | 图状态机 + 检查点，复杂有状态工具编排 | ✓ | ✓ |
| **CrewAI** | 成熟/上升 | 角色驱动的工具分配，学习曲线最低 | ✓ | ✓ |
| **Microsoft Agent Framework** | 成熟 | SK + AutoGen 合并，2026-04 GA 1.0，.NET + Python | ✓ | ✓ |
| **OpenAI Agents SDK** | 上升 | 原生沙箱执行，模型原生工具调用 | ✓ | 部分 |
| **Claude Agent SDK** | 上升 | 内建工具执行循环、子 agent、hooks | ✓（原生） | — |
| **Google ADK** | 上升 | Go 1.0 + TS + Python，一键 Cloud 部署 | ✓ | ✓ |
| **Haystack**（deepset） | 成熟/小众 | 类型化 I/O，DAG 管线，最低 token 消耗（1.57K/调用） | ✓ | — |
| **LlamaIndex** | 成熟/小众 | 最深的索引/分块/检索栈 | ✓ | — |

**采纳规模。** LangChain 100K+ GitHub stars，LangGraph 30.9K stars。CrewAI 47.8K stars、27M+ PyPI 下载、150+ 企业客户、12 个月内 2B 次 agent 执行。Datadog 2026 年报告显示框架采纳率同比翻倍（9% → 18% 的组织），使用框架的服务数量增长超过两倍。

**MCP 标准化的影响。** MCP 已成为 L2 层的通用协议：月度 SDK 下载量 9700 万（18 个月增长 970 倍），10,000+ 已发布的 MCP server。所有上表中的框架都已原生或通过扩展支持 MCP。自定义工具格式尚未完全消亡，但其相关性正在快速下降——MCP 的标准化消除了"基于工具生态选框架"的锁定逻辑。

**框架疲劳与整合趋势。** 三个信号表明生态正在整合：厂商合并（Microsoft 将 AutoGen 和 Semantic Kernel 合并为一个 SDK，LangChain 将 LangGraph 确立为唯一的 agent runtime）；协议标准化（MCP 和 A2A 消除了框架间的工具和通信壁垒）；操作层面的反弹（Datadog 报告框架引入了"agent sprawl"——隐藏的复杂性使调试更难，Gartner 警告 2027 年前 40%+ 的 agentic AI 项目将因成本升级而取消）。团队正在从"选哪个框架"转向"多薄的框架"——直接使用 MCP server 加最少的编排层，而非采纳重量级抽象。

**L2 层的框架选型建议。** 如果核心需求是检索质量，LlamaIndex 的索引和分块能力最深（框架开销约 6ms）。如果核心需求是复杂的工具编排和状态管理，LangGraph 提供最强的可控性和可视化（开销约 10-14ms）。如果在合规行业需要可审计的管线，Haystack 的类型化管线和最低 token 消耗是优势。生产中常见的组合是 LlamaIndex 做检索 + LangGraph 做编排。如果团队已深度绑定某个云生态（Azure/GCP/OpenAI），对应厂商框架是阻力最小的选择。

### 向量数据库与检索基础设施

**主要玩家（截至 2026-04）：**

| 产品 | 状态 | 核心差异化 | 规模信号 |
|------|------|----------|---------|
| **Pinecone** | 成熟/下行 | 全托管、零运维 | $14M 收入（较 2024 年 $26.6M 下降），$138M+ 融资 |
| **Weaviate** | 成熟/上升 | 混合搜索最强（BM25 + 向量内建） | $50M C 轮（2025-10），$12.3M 收入 |
| **Qdrant** | 上升 | sub-5ms 过滤搜索，Rust 实现 | $50M B 轮（2026-03），$87.8M 总融资 |
| **Chroma** | 上升/小众 | 开发者友好，嵌入式部署 | $18M 种子轮，原型和小规模场景 |
| **Milvus / Zilliz** | 成熟 | 大规模（十亿级向量） | $100M+ 总融资，大型中国科技公司部署 |
| **pgvector** | 颠覆者 | PostgreSQL 扩展，50M 向量以下性价比最优 | pgvectorscale 基准：471 QPS vs Qdrant 41 QPS（99% recall，50M 向量） |
| **Neo4j** | 转型 | 向量 + 图的交叉点，GraphRAG 枢纽 | HNSW 向量索引 + Cypher 图遍历，Microsoft GraphRAG 基于 Neo4j |

**商品化：这个品类的核心趋势。** 向量正在从"数据库品类"变成"数据类型"。pgvector 在 50M 向量以下的场景中性价比碾压专用向量数据库——Supabase（pgvector 原生支持）在 2025-10 达到 $5B 估值。传统数据库（MongoDB Atlas Vector Search、Oracle 23ai）纷纷加入原生向量支持。Pinecone 收入下降是前哨信号——纯向量存储是一场"向底部赛跑"。专用向量数据库的分化方向是向上走——检索编排、混合搜索、GraphRAG，而非存储本身。Snowflake 和 Databricks 在 2025 年花了约 $12.5 亿收购 PostgreSQL-first 的公司，进一步验证了"向量嵌入传统数据库"的趋势。

**混合搜索成为标配。** 向量搜索（语义匹配）+ BM25（关键词匹配）的混合检索，通过 RRF 融合排序，在所有主流场景中一致优于单一方法（91% recall@10 vs 纯向量 78% vs 纯稀疏 65%）。每个主要向量数据库现在都支持混合搜索。这不再是差异化特性，而是入门门槛。

**图 + 向量融合。** 每个认真做 RAG 的团队都在悄悄构建图层。GraphRAG 解决了基于块检索无法处理的多跳问题。Neo4j 在这个交叉点上占据有利位置（原生 HNSW 向量索引 + Cypher 图遍历）。代价是索引成本比纯向量 RAG 高 10-40 倍，适用于关系密集的数据而非通用检索。

**Embedding 模型格局。** 闭源方面：OpenAI text-embedding-3-large（3072 维）、Cohere embed-v4.0（企业级多语言 + 多模态）、Google Gemini Embedding 2（支持文本/图像/视频/音频/PDF 五种模态，支持 MRL 可变维度）。开源方面：Qwen3-Embedding-8B（阿里，正在逼近闭源模型水平）、BGE-M3（BAAI，100+ 语言，单张 A10 GPU 可运行）。月处理量超过 1 亿 token 时，自托管开源模型的成本显著低于 API。

## 塌缩与涌现

### 塌缩

- **ChatGPT Plugins 及封闭工具市场模式**（2023 年高调推出，2024 年退役）——证明了「平台方控制的工具市场」不是正确的标准化路径，社区驱动的开放协议（MCP）才是
- **每框架一套的工具适配格式**——MCP 统一了这个碎片化格局
- **简单的 RAG pipeline**——长上下文 + reasoning model 让很多"检索-拼接-回答"的场景可以直接用长 context 解决
- **手工编排的多步工具链**——reasoning model 自己决定调用顺序，减少了显式编排的需要

### 涌现

- **MCP 生态管理**：工具的发现、质量评估、版本管理、权限控制成为新的工程主题
- **工具描述工程**：如何写出让模型理解且准确调用的工具描述，成为一个非平凡的工程实践
- **Computer Use 基础设施**：截图管道、坐标系统、操作确认、失败恢复——全新的基础设施需求
- **CLI 安全沙箱**：防止 agent 执行破坏性命令的沙箱和权限机制
- **工具调用可观测性**：追踪整个工具调用链的执行情况、耗时、成本

## 开放工程问题

- MCP server 的信任模型：如何验证一个第三方 MCP server 是安全的、行为符合预期？
- 大工具集的管理：当一个 agent 可以访问数百个 MCP server 暴露的数千个工具时，如何有效地组织和检索？
- CLI tool use 的标准化：是否需要一个专门的协议，还是 MCP 可以扩展覆盖？
- Computer Use 的权限模型：agent 操作 GUI 时，如何实现细粒度的权限控制？
- RAG 的残余价值：在长上下文 + reasoning model 时代，哪些 RAG 模式仍不可替代？

注：以上多个安全相关问题在 [layers/L4-production.md](L4-production.md) 的安全章节有更系统的覆盖，包括 OWASP Top 10 for Agentic Applications 2026 框架和间接 prompt injection 的生产防御。

## 参考资料

### MCP 协议与生态

- [The 2026 MCP Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) — MCP 官方 2026 路线图，四大优先方向
- [MCP Roadmap](https://modelcontextprotocol.io/development/roadmap) — 官方路线图页面，实时更新
- [MCP's Biggest Growing Pains for Production Use Will Soon Be Solved](https://thenewstack.io/model-context-protocol-roadmap-2026/) — The New Stack 对 MCP 生产化痛点的分析
- [The Model Context Protocol's Impact on 2025](https://www.thoughtworks.com/en-us/insights/blog/generative-ai/model-context-protocol-mcp-impact-2025) — Thoughtworks 对 MCP 2025 年影响的回顾
- [Model Context Protocol - Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol) — 基础概念和历史
- [2026: The Year for Enterprise-Ready MCP Adoption](https://www.cdata.com/blog/2026-year-enterprise-ready-mcp-adoption) — 企业级 MCP 采纳的现状与挑战

### A2A 协议

- [Announcing the Agent2Agent Protocol (A2A)](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) — Google 官方发布文章
- [Developer's Guide to AI Agent Protocols](https://developers.googleblog.com/developers-guide-to-ai-agent-protocols/) — Google 官方的 MCP + A2A 开发者指南
- [A2A Protocol Explained: How Google's Standard Grew to 150+ Organizations](https://stellagent.ai/insights/a2a-protocol-google-agent-to-agent) — A2A 一年来的采纳增长
- [MCP vs A2A: A Guide to AI Agent Communication Protocols](https://auth0.com/blog/mcp-vs-a2a/) — Auth0 对两个协议互补关系的技术分析

### MCP vs A2A 对比

- [MCP vs A2A: The Two Protocols Shaping the AI Agent Ecosystem (2026)](https://devtk.ai/en/blog/mcp-vs-a2a-comparison-2026/) — 2026 视角下的对比
- [MCP vs A2A: The Complete Guide to AI Agent Protocols in 2026](https://dev.to/pockit_tools/mcp-vs-a2a-the-complete-guide-to-ai-agent-protocols-in-2026-30li) — 开发者社区的完整指南

### RAG 模式与架构

- [LaRA: Benchmarking RAG and Long-Context LLMs — OpenReview](https://openreview.net/forum?id=CLF25dahgA) — 2326 测试用例的 RAG vs 长上下文对照，确认无普适赢家
- [SoK: Agentic RAG — arXiv](https://arxiv.org/abs/2603.07379) — Agentic RAG 的形式化分类：agent 基数、控制结构、自主级别、知识表示
- [RAG Production Guide 2026 — LushBinary](https://lushbinary.com/blog/rag-retrieval-augmented-generation-production-guide/) — 2026 RAG 生产实践指南
- [RAG Techniques Compared: Practical Guide 2026 — Starmorph](https://blog.starmorph.com/blog/rag-techniques-compared-best-practices-guide) — 各种 RAG 技术的实践对比
- [Best RAG Frameworks 2026 — iTernal](https://iternal.ai/blockify-rag-frameworks) — LangChain vs LlamaIndex vs DSPy vs Haystack 的 RAG 能力对比

### Agent 框架

- [Datadog State of AI Engineering 2026](https://www.datadoghq.com/state-of-ai-engineering/) — 框架采纳率同比翻倍（9%→18%），agent sprawl 的运维挑战
- [Microsoft Agent Framework 1.0 GA — DevBlogs](https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-version-1-0/) — SK + AutoGen 合并后的 1.0 正式版
- [OpenAI Agents SDK: The Next Evolution — OpenAI](https://openai.com/index/the-next-evolution-of-the-agents-sdk/) — Agents SDK 的演进方向
- [Claude Agent SDK Overview — Anthropic](https://platform.claude.com/docs/en/agent-sdk/overview) — Claude Agent SDK 的架构和能力
- [Google ADK Go 1.0 — Google Developers](https://developers.googleblog.com/adk-go-10-arrives/) — Go 1.0 发布，双周发布节奏
- [CrewAI Platform Statistics — Panto](https://www.getpanto.ai/blog/crewai-platform-statistics) — 47.8K stars、27M+ 下载、2B 次执行
- [LLM Frameworks Compared 2026 — Morph](https://www.morphllm.com/llm-frameworks) — 框架开销和 token 消耗对比

### 向量数据库与检索基础设施

- [RAG vs Long Context: Do Vector Databases Still Matter in 2026? — MarkAICode](https://markaicode.com/vs/rag-vs-long-context/) — 企业 RAG 部署增长 280%，向量数据库品类 $800M+ VC
- [The Vector Database Hype is Over — Estuary](https://estuary.dev/blog/the-vector-database-hype-is-over) — 向量商品化趋势分析
- [The Shift to Hybrid RAG: Graph Layers in 2026 — Earezki](https://earezki.com/ai-news/2026-04-26-why-every-rag-company-is-quietly-building-a-graph-layer-in-2026/) — 图+向量融合趋势
- [Which Embedding Model Should You Use in 2026? — Dev.to](https://dev.to/chen_zhang_bac430bc7f6b95/which-embedding-model-should-you-actually-use-in-2026-i-benchmarked-10-models-to-find-out-58bc) — 10 个嵌入模型的基准对比
- [Embedding Models in 2026 — StackSpend](https://www.stackspend.app/resources/blog/embedding-models-2026-options-pros-cons) — 嵌入模型选型指南

## 更新日志

- 2026-05-01：补充四个待补充章节——RAG 模式（四层架构分类、关键生产模式）、Memory 工程（被动注入 vs 主动管理范式）、Agent 框架（8 框架格局、MCP 标准化影响、整合趋势）、向量数据库（7 玩家格局、商品化趋势、混合搜索、图+向量融合、embedding 模型）
- 2026-05-01：开放问题中增加 L4 安全章节交叉引用
- 2026-04-26：初次创建骨架。Tool Integration 三种模式、MCP、A2A 已填充；RAG、Memory 工程、Agent 框架、向量数据库待补充
