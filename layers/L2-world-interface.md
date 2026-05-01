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

（待补充。这个子领域在第三代经历了大面积塌缩——reasoning model + 长上下文窗口让很多 RAG 场景不再必要。需要梳理哪些 RAG 模式仍然有价值、哪些已经塌缩。）

### Memory 工程

（待补充。这个子领域可能是第四代的核心主题。需要梳理当前的工程方案：向量库、对话历史压缩、MemGPT 类架构等。与维度 6 Memory 的关系是：维度文件写能力演进，这里写工程实践。）

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

（待补充。LangChain / LangGraph、CrewAI、AutoGen、Semantic Kernel 等主流框架的当前定位和演进方向。重点关注它们如何适配 MCP/A2A。）

### 向量数据库与检索基础设施

（待补充。Pinecone、Weaviate、Qdrant、Chroma 等。关注第三代塌缩的影响。）

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

## 更新日志

- 2026-05-01：开放问题中增加 L4 安全章节交叉引用
- 2026-04-26：初次创建骨架。Tool Integration 三种模式、MCP、A2A 已填充；RAG、Memory 工程、Agent 框架、向量数据库待补充
