# Agentic RAG

> 锚点：层级 L2（与世界的接口）/ 层级 L1（Agent 内核）/ 维度 1 Reasoning

## 这个概念是什么

Agentic RAG 把 RAG 从"固定管线"变成"动态决策"：不是执行预定义的"检索 → 生成"流程，而是让一个 agent 控制器自主决定——是否检索、检索什么、检索结果是否足够、是否需要再检索、何时停止。最直接的体现是：步骤数不再固定，由查询复杂度决定。

这个 topic 覆盖四种具体技术路线：ReAct 控制循环（基础架构）、Self-RAG（反思令牌）、CRAG（Corrective RAG，检索评估器）、Adaptive RAG（复杂度分类器路由）。四者不是互斥的——生产系统通常把 Adaptive RAG 的路由和 CRAG 的质量评估组合使用。

## 内部结构

### ReAct：Agentic RAG 的基础控制循环

**机制**：ReAct（Reason + Act）是 Yao 等人 2023 年提出的通用 agent 框架（arXiv 2210.03629），在 RAG 语境下就是把"检索"作为一种 tool 融入 agent 的思考-行动循环：

```
Thought: 我需要查询 X 才能回答这个问题
Action: retrieve("X 的相关信息")
Observation: [检索结果]
Thought: 结果提到了 Y，但我还需要了解 Z
Action: retrieve("Z 的背景")
Observation: [检索结果]
Thought: 现在信息足够了
Action: generate_answer()
```

这个循环的步骤数是动态的——简单事实查询可能 1 步结束，多跳推理可能需要 3-5 轮。每一步的检索决策由上一步的 observation 驱动。

**与固定管线的根本区别**：固定管线中，"要不要检索"和"检索几次"是在系统设计阶段就写死的；ReAct 把这些决策权交给了 LLM 本身。这意味着同样的代码在不同查询上会产生完全不同的执行轨迹。

**主流实现**：LangGraph 提供了 ReAct agent 的有状态图实现，可以定义工具集（retrieve / search / calculate），agent 在每一步选择工具并处理 observation。LlamaIndex 的 `ReActAgent` 和 OpenAI Agents SDK 都有对应实现。

落地程度：**生产部署，是 Agentic RAG 最基础的实现框架**。

### Self-RAG：让模型自己评判检索质量

**机制**：Asai 等人 2023 年提出（arXiv 2310.11511，ICLR 2024 最佳论文奖）。与其他方案不同，Self-RAG 需要对 LLM 本身做**微调**——在训练数据中加入特殊的 reflection token，让模型学会在生成时自我评估：

四种 reflection token：
- **[Retrieve]**：模型判断当前是否需要检索（yes / no / continue）
- **[IsRel]**：检索到的文档是否与查询相关（relevant / irrelevant）
- **[IsSup]**：生成的内容是否有检索文档的支持（fully supported / partially / no support）
- **[IsUse]**：最终生成的回答是否有用（1-5 分）

推理时，模型边生成边输出这些 reflection token，根据 token 的值决定是否触发检索、是否重新检索、是否继续生成。

**为什么值得单独关注**：Self-RAG 是唯一一种把"检索决策和质量评估"内化到模型本身的方案——不需要外部评估器，不需要额外的 LLM 调用，一个模型搞定全部。在推理效率上有独特优势。

**局限**：需要微调，这意味着需要专门的训练数据、维护自己的模型版本。对大多数团队来说，Fine-tune 一个 Self-RAG 模型的成本（数据标注、训练、评估）不如直接用 Reasoning 模型 + CRAG 的组合。

**Reasoning 模型时代的位置**：o3 / Claude Opus 4.x 等 reasoning 模型已经内置了复杂的自我评估和规划能力，某种程度上"原生"实现了 Self-RAG 的功能，但通过 inference-time compute 而非微调。落地程度：**学术影响大（ICLR 2024 最佳），生产中直接采用有限，更多作为设计灵感被整合进更大系统**。

### CRAG（Corrective RAG）：外置检索评估器

**机制**：Yan 等人 2024 年提出（ICLR 2024 Workshop）。CRAG 的核心是引入一个**轻量检索评估器（retrieval evaluator）**，对检索结果打分，根据置信度分三个分支：

- **高置信度（相关）**：直接使用检索结果，通过"知识精炼（knowledge refinement）"去除噪声片段后送入生成
- **低置信度（不相关）**：触发 Web 搜索（Bing / Tavily API 等）补充信息，再融合本地检索结果
- **中等置信度（模糊）**：同时使用本地结果和 Web 搜索结果，合并后送入生成

**知识精炼**是 CRAG 的另一个关键操作：对检索到的文档，先用 LLM 把它分解成细粒度知识条（decompose），对每条打相关性分，只保留高相关的条（recompose），去除噪声。这解决了"检索到的文档里有大量无关内容干扰 LLM 生成"的问题。

**生产数据**：一项 2025 年 MDPI benchmark 报告 CRAG 在 Precision@5 = 0.69、hallucination rate = 10.5%，平均延迟 240ms。这使得 CRAG 成为"最实用的第一步 Agentic RAG"——加入一个评估器分类层，而非重架整个 RAG 系统。

落地程度：**生产部署，尤其在需要防止检索错误传播的场景（知识密集型 QA、医疗、法律）**。

### Adaptive RAG：复杂度路由的成本控制

**机制**：Jeong 等人 2024 年提出（arXiv 2403.14403，NAACL 2024），GitHub: starsuzi/Adaptive-RAG。核心思想是**用查询复杂度决定检索策略**，而非对所有查询都走同一套管线：

分类器把查询分为三级：
- **简单（A 类）**：直接生成（不检索），适合模型训练数据覆盖充分的事实性知识
- **中等（B 类）**：单步检索（标准 RAG），适合需要一次外部知识补充的查询
- **复杂（C 类）**：多步迭代检索（agentic RAG），适合多跳推理、需要多次检索合成信息的查询

分类器是一个**小型 LM 微调模型**，而非大模型——推理延迟极低（<10ms），不增加显著开销。训练数据通过"让大模型和小模型在多个 QA 数据集上预测，以预测结果的差异作为复杂度标签"自动生成，无需人工标注。

**为什么这是成本控制的关键**：纯 Agentic RAG 对所有查询都走完整的 ReAct 循环，简单查询（"2024 年奥运会在哪里举办？"）也需要 3-5 轮检索，token 成本和延迟显著偏高。Adaptive RAG 把 70-80% 的简单查询路由到轻量路径，在保持整体召回质量的同时大幅降低平均成本。实测显示 Adaptive RAG 在多跳 QA 数据集上的效率和准确率都优于单一策略的基线。

落地程度：**企业生产部署，是 2025-2026 年 agentic RAG 的事实最佳实践**。

### 三种进阶 Agentic 模式

**FLARE（Forward-Looking Active REtrieval）**：在生成过程中，当模型对即将生成的内容置信度低时主动触发检索——不是在生成前检索，而是在生成过程中实时决策。对长文本生成（报告、分析）尤其有价值，能在模型"知识边界"处动态补充信息。实验室验证阶段。

**Self-Ask**：模型先自问"回答这个问题，我需要先知道什么子问题？"，把这些子问题一一检索回答，最后综合。是 Sub-Question Decomposition 的 agent 变体。LangChain 有 `SelfAskWithSearchChain` 实现。

**多 Agent 分布式检索**：对知识库做功能划分（产品文档 / 代码库 / 法律条文 / 内部 Wiki），每个知识库有专门的检索 agent，一个 orchestrator agent 协调多个检索 agent 并行检索后综合。这模糊了 L2（检索）和 L3（多 agent 编排）的边界。

### Agentic RAG 的失败模式

**跨步骤误差复合**：每一步检索的轻微偏差会在下一步被放大——用错误的子问题查询 → 检索到轻微不相关的文档 → 基于这个文档生成错误的后续查询 → 越走越偏。这是 Agentic RAG 最难处理的失败模式，也是为什么"73% 的失败在检索环节"的比例在 agentic 场景更高。

**循环不收敛**：agent 在无法检索到满意信息时可能陷入循环（不断换查询词重复检索）。需要显式的终止条件（最大步数 / 最小新增信息阈值）。

**工具调用错误**：检索工具的 schema 设计不合理时（参数命名不直觉、描述模糊），LLM 生成的工具调用可能格式错误或语义错误。这是 L1 的工具描述工程问题（详见 [layers/L2-world-interface.md](../layers/L2-world-interface.md)）传导到 Agentic RAG 的表现。

**成本失控**：没有预算上限的 agentic RAG 在复杂查询上可能消耗 10-50 次 LLM 调用。生产中需要设置最大步数（通常 5-10 步）和 token 预算上限。

### 可观测性：Agentic RAG 的调试基础设施

固定管线 RAG 的失败诊断是：查检索结果 → 查 prompt → 查输出。Agentic RAG 的失败诊断需要追踪整个动态执行轨迹，每步的查询、检索结果、推理都需要记录。

**关键指标**：每步检索延迟、工具调用成功率、循环深度（多少步才结束）、每次检索的相关性评分（CRAG 评估器的分数）、最终答案的忠实度（faithfulness）和相关性（answer relevance）。

**主流工具**（截至 2026-05）：
- **LangSmith**（LangChain 生态）：对 LangGraph agent 的原生集成，可视化执行轨迹
- **Langfuse**：开源替代，支持多框架，可自托管
- **Arize Phoenix**：专注 LLM/RAG 可观测性，有专门的 RAG 评测 metric（ARES 框架）
- **Ragas**：专用于 RAG 评测的框架，faithfulness / answer relevancy / context precision / context recall 四个核心 metric

生产目标（MarsDevs 2026 指南）：faithfulness ≥ 0.9、answer relevancy ≥ 0.85、context precision ≥ 0.8。

## 当前状态（截至 2026-05）

生产中的 Agentic RAG 已从"是否值得用"的讨论阶段进入"如何用好"的工程阶段。Adaptive RAG 的路由思想是最广泛采用的成本控制方案；CRAG 的检索评估器是最常见的质量保证组件；ReAct 是最常见的控制循环。Self-RAG 因为需要微调，直接采用较少，但其 reflection token 的设计思路被广泛参考。

Reasoning 模型对 Agentic RAG 的影响是显著的：o3 / Claude Opus 4.x 的 extended thinking 本身就能完成类似 Self-RAG 的自我评估，某些原本需要复杂 agentic 框架的多跳检索现在可以简化——让 reasoning 模型自己决定何时检索什么。但这不意味着 CRAG 和 Adaptive RAG 变得不必要——它们提供了 reasoning 模型自发行为之外的**显式控制和成本管理**。

## 关键权衡

**微调（Self-RAG）vs 提示工程（CRAG/Adaptive）**：Self-RAG 推理效率高但需要维护自有模型；CRAG 和 Adaptive RAG 可以直接在 API 上实现。对大多数团队，提示工程方案阻力更小。

**单 agent vs 多 agent 检索**：单 agent 实现简单、延迟低、调试容易；多 agent 并行检索能处理分布式知识库但引入 L3 的编排复杂度。需要根据知识库分布和延迟需求决定。

**步数上限 vs 检索完整性**：步数上限越低，成本和延迟越可控，但可能在信息不足时截断；上限越高，越能处理复杂查询，但成本失控风险越高。生产起点通常是 5 步上限，根据实际用例调整。

## 信息源

- [ReAct: Synergizing Reasoning and Acting in Language Models — arXiv](https://arxiv.org/abs/2210.03629) — ReAct 原始论文，Yao et al. 2023
- [Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection — arXiv](https://arxiv.org/abs/2310.11511) — Self-RAG 原始论文，Asai et al. ICLR 2024
- [Corrective Retrieval Augmented Generation (CRAG) — OpenReview](https://openreview.net/forum?id=JnWJbrnaUE) — CRAG 论文，Yan et al. ICLR 2024 Workshop
- [Adaptive-RAG: Learning to Adapt Retrieval-Augmented LLMs through Question Complexity — ACL Anthology](https://aclanthology.org/2024.naacl-long.389/) — Adaptive RAG 论文，Jeong et al. NAACL 2024
- [Agentic RAG: The 2026 Production Guide — MarsDevs](https://www.marsdevs.com/guides/agentic-rag-2026-guide) — 2026 年生产实践指南，含目标 metric 和工具栈
- [How to Build a Production RAG Pipeline in 2026 — RoboRhythms](https://www.roborhythms.com/how-to-build-production-rag-pipeline-2026/) — 生产 RAG 管线的五层设计
- [SoK: Agentic RAG — arXiv](https://arxiv.org/abs/2603.07379) — 2026-03 的 Agentic RAG 形式化分类综述（agent 基数、控制结构、自主级别）

## 更新日志

- 2026-05-06：初次创建，从 topics/advanced-rag-variants.md 拆分，覆盖 ReAct / Self-RAG / CRAG / Adaptive RAG 四种技术路线，以及失败模式和可观测性基础设施
