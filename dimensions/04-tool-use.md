# 维度 4：Tool Use（工具调用）

## 一句话定义

模型在生成过程中主动决定调用外部函数或操作外部系统的原生能力——这是 LLM 从「只能说」到「能做事」的关键跃迁，也是 agent 之所以成为 agent 的基本前提。

## 核心概念与技术

### 基础机制：函数调用如何工作

Tool use 的核心循环只有四步：定义工具 schema → 模型决定调用并生成参数 → 外部执行 → 结果回注到上下文。这个循环看似简单，但每一步都有工程上不平凡的地方。

**训练侧**：模型的 tool use 能力不是凭空出现的，而是通过几种手段训练出来的。首先是代码语料的基础作用——大量的代码训练让模型天然理解 JSON、函数签名、参数结构。在此基础上，主流的做法是在 SFT（监督微调）阶段注入合成的 tool-call 数据。典型方法如 ToolFormer（2023）：用 LLM 自己生成工具调用标注，然后用加权交叉熵筛选出真正有帮助的调用——只保留那些让模型后续预测更准的调用样本。Meta 的 Llama 系列采用类似思路：从真实代码中抽取函数调用和定义，清洗后生成配对的自然语言查询。更晚近的 ToolACE 管线则通过自动化的自演进合成过程生成了 26,000+ 多样化 API 的训练数据。

**推理侧**：即使训练到位，模型也可能生成格式不合法的调用。这催生了**受限解码（constrained decoding）**技术：在生成时动态约束 token 概率分布，确保输出符合 JSON schema。OpenAI 在 2024 年推出的 strict mode 就是这个思路——通过语法约束保证 100% 的 schema 合规率。当前的生产标准是混合方案：训练保证高基线合规率，推理时对关键字段（函数名、必填参数）施加额外约束。

**模型需要回答的三个问题（Three Ws 框架）**：Whether——这个请求是否需要调工具？Which——从可用工具集中选哪个？How——用什么参数调用？三者中，Whether 最常被忽略，但在实际系统中最关键——一个误判"需要调工具"的 false positive 可能比选错工具更危险。

### 三种调用形态

Tool use 在「模型怎么操作外部世界」这个维度上存在三种形态，各自的特征差异很大。这是理解当前 tool use 全景的一个核心分类框架。

**结构化 API 调用**是目前最标准化的路径。模型输出 JSON 格式的函数调用（函数名 + 参数），由外部框架执行后返回结构化结果。优势在于 schema 严格、可验证、易观测、安全边界清晰——你可以精确控制模型能调用哪些函数、参数范围是什么。MCP 协议本质上是在标准化这条路径。几乎所有 tool use 的学术研究和 benchmark 都针对这种形态。

**CLI / Shell 调用**是实际 agent 系统（尤其是 coding agent）的主力接口，但在标准化和学术讨论中被严重低估。模型生成 shell 命令，在终端执行后拿回文本输出。比 API 灵活得多——任何已安装的 CLI 工具即插即用，不需要预先定义 schema。但也更危险：命令注入风险、破坏性操作（`rm -rf`）、输出是非结构化文本需要模型自行解析。Claude Code 是这种形态的典型代表：它混合使用专用的结构化工具（Read、Edit、Grep）和通用的 Bash 工具执行任意 shell 命令。这种**结构化 + CLI 混合**的模式是当前 coding agent 的事实标准。值得注意的是，CLI 路径目前几乎没有专门的标准化协议或评测体系，这是一个显著的覆盖空白。

**Computer Use / GUI 操作**是最新的形态，把「工具」的概念从 API 扩展到了整个图形界面。模型通过视觉感知屏幕内容，生成鼠标移动和键盘操作，观察下一帧结果。这条路径依赖多模态能力（本质是 Tool Use × Multimodality 的交叉），延迟最高，可靠性目前最低，但覆盖面最广——理论上任何人能用的软件它都能用。2024 年末 Anthropic 率先推出 Computer Use beta，随后 OpenAI（Operator）、Google、Perplexity 等相继跟进。到 2026 年初，三家所有主要 AI 公司都推出了某种形式的 computer use 能力。

三者的权衡轴线：**结构化程度和安全性递减，灵活性和覆盖面递增。** 在实际系统中它们不互斥——最强的 agent 同时使用多种形态，根据任务性质选择最合适的路径。

### 能力进阶：从单次调用到复杂编排

**并行函数调用（Parallel Function Calling）**：模型在一次响应中生成多个独立的函数调用，框架并行执行后一起返回。性能提升显著——四个各 300ms 的 API 调用从串行的 1.2 秒降到并行的约 300ms。LLMCompiler（ICML 2024）进一步提出在运行时自动融合相似类型的工具操作，实现最多 4 倍的并行度提升，同时降低 40% 的 token 消耗。

**结构化输出（Structured Outputs）**：从 JSON Mode（只保证语法合法）到 Strict Mode（保证 schema 合规），是 tool use 可靠性的关键一步。OpenAI 2024 年推出 strict 模式后，JSON Mode 已被视为 legacy。当前生产标准是 schema 级别的严格输出。

**大工具集的工具选择**：当可用工具从几个增长到几百上千个（MCP 生态下这很常见），模型需要从大量候选中选出正确的工具。这变成了一个检索问题——工具描述的质量和组织方式比工具的数量更重要。

**多轮工具使用（Multi-turn Tool Use）**：agent 在多次对话轮次中持续调用工具，需要维持上下文、记住之前的调用结果、处理工具调用失败。这是当前 benchmark 显示模型表现仍有明显差距的区域——BFCL 的数据表明，前沿模型在单轮调用上接近满分，但在多轮和记忆依赖场景中明显下降。

### 生态标准化：MCP 与 A2A

Tool use 在 2025-2026 年经历了一波**生态事件驱动**的剧变——不是模型能力的突破，而是工具生态的标准化。

**MCP（Model Context Protocol）**由 Anthropic 于 2024 年末发布，2025 年爆发增长，到 2026 年 3 月月度 SDK 下载量达 9700 万，独立索引的公开 MCP 服务器超过 17,000 个。MCP 解决的核心问题是：让任何工具只需实现一次，就能被任何支持 MCP 的 agent 使用——打破了之前每个 agent 框架都要自己写工具适配层的碎片化格局。2025 年 12 月，Anthropic 将 MCP 捐赠给 Linux Foundation 下新成立的 Agentic AI Foundation（AAIF），OpenAI、Google、Microsoft、AWS 等共同参与治理。2026 年路线图聚焦四个方向：Streamable HTTP 传输的规模化、Tasks 原语的生产化、治理成熟度提升、以及远程服务器发现机制。

**A2A（Agent-to-Agent Protocol）**由 Google 于 2025 年 4 月发布，解决的是不同平面的问题：MCP 标准化的是 agent-to-tool 的连接，A2A 标准化的是 agent-to-agent 的通信。二者是互补关系而非竞争——一个 agent 内部通过 MCP 访问工具，外部通过 A2A 与其他 agent 协作。A2A 在 2026 年初达到 v1.2，已有 150+ 组织在生产环境使用，同样归入 AAIF 治理。

这两个协议的出现意味着 tool use 的瓶颈已从模型能力转向生态基础设施。这一点在维度文件里标注，在 L2 层级文件里详细展开。

## 当前技术格局（截至 2026-04）

**简单函数调用已经商品化。** Gemini 3.1 Pro、Claude Opus 4.6 等前沿模型在标准 benchmark 上都达到 99%+ 的准确率。简单的单轮、单工具调用不再是区分模型能力的维度。

**复杂场景仍有差距。** BFCL V4 的数据讲了一个清晰的故事：模型在单轮调用上接近满分，但在多轮对话、记忆依赖、动态决策（判断何时不该调用工具）等场景中仍有明显不足。整体平均分约 0.717，离满分还有相当距离。

**Computer Use 可用但不可靠。** WebArena benchmark 上当前最优表现约 71.2%，多数生产系统在 50-60% 区间。Mobile-use 框架在 AndroidWorld 上达到 100% 成功率是个里程碑，但通用桌面环境仍有大量边界情况。

**CLI 工具使用缺乏标准化评测。** 尽管它是 coding agent 的事实主力接口，目前没有专门的 benchmark。SWE-Bench 间接评估了这个能力（coding agent 必须通过 CLI 操作来解决 GitHub issue），Claude Mythos 在 SWE-Bench Verified 上达到 93.9%。

**开源模型在追赶。** Llama 3.1 405B 在 BFCL 上名列前茅（0.885），Qwen 系列在 tool calling 上也表现突出。开源和闭源的差距比 reasoning 维度小得多——tool use 的训练方法更成熟、更可复现。

**主要 benchmark**：BFCL（Berkeley Function Calling Leaderboard，最全面的函数调用评测）、Nexus（工具选择）、API-Bench、τ-Bench（多轮对话 tool use）、SWE-Bench（间接评估 CLI tool use）、WebArena / OSWorld（Computer Use）。

## 演进路径

- **2023 年中**：OpenAI 在 GPT-3.5/4 API 中推出原生 function calling，这是 tool use 从 prompt hack 转向 API 原生支持的分水岭。同期 ChatGPT Plugins 高调推出，试图构建工具市场。
- **2023 年末 - 2024 年初**：Plugins 生态未能起飞（发现率低、用户体验差、商业模式不清）。但 function calling 本身在所有主要模型中快速普及。
- **2024 年中**：并行函数调用成为标准能力。OpenAI 推出 Structured Outputs / strict mode。Anthropic 推出 Computer Use beta。tool use 的能力维度基本到位。
- **2024 年末 - 2025 年初**：MCP 发布并爆发。焦点从"模型能不能调工具"转向"工具生态怎么标准化"。
- **2025 年中**：A2A 发布。MCP + A2A 构成完整的 agent 互操作层（tool 层 + agent 层）。CLI agent（Claude Code、Codex CLI、Aider 等）开始成为开发者日常工具。
- **2026 年初**：MCP 和 A2A 都归入 Linux Foundation 治理。简单 tool calling 商品化。前沿转向复杂多轮编排、Computer Use 可靠性、以及大规模工具集的管理。

## 对四层骨架的影响

### 第一层（Agent 内核）

**塌缩**：
- 经典 ReAct 中"行动"相位的实现从 prompt 模拟变为 API 原生调用，大量的格式解析和错误处理代码不再需要
- 手动编写的"工具选择"逻辑（if-else 路由）被模型原生的 tool selection 取代

**涌现**：
- Reasoning model 的思考-行动协同模式成为新设计问题——模型先在 thinking 中规划工具调用策略，再在 output 中执行，两个相位之间的协调需要新的工程考量
- "何时不调用工具"（Whether 问题）的重要性凸显，过度调用（tool call hallucination）成为需要专门处理的问题

### 第二层（与世界的接口）

这一层的存在根基就是 tool use 能力。

**塌缩**：
- ChatGPT Plugins 及类似的封闭工具市场模式
- 每个 agent 框架自定义的工具适配格式（LangChain Tools、AutoGPT Plugins 等）——MCP 正在统一这个碎片化格局
- 对非 reasoning model 而言，复杂的多跳工具编排链路（reasoning model 自己决定调用顺序）

**涌现**：
- MCP 生态：工具的标准化发现、连接、和权限管理成为新的工程主题
- 工具描述工程：从简短的 schema 描述转向更长的「教学式」描述，工具描述的质量直接影响调用准确率
- Computer Use 作为新的工具接口类型，需要全新的基础设施（屏幕截图、坐标映射、操作确认）

### 第三层（多 Agent 协作）

**塌缩**：
- 多 agent 共享工具需要自建适配层的模式——MCP 让工具天然可共享

**涌现**：
- A2A 协议让 agent 间可以互相暴露能力（以"工具"的形式），催生新的多 agent 协作模式
- 工具权限的隔离问题：不同 agent 应该能调用哪些工具？这在多 agent 系统中比单 agent 复杂得多

### 第四层（生产化）

**塌缩**：
- 简单的工具调用成功率监控已经不够（因为成功率接近 100%）

**涌现**：
- 工具调用安全性成为核心问题：agent 能执行任意工具 = 新的攻击面。Prompt injection 通过工具结果注入恶意指令是已知威胁
- CLI 工具使用的安全边界：如何防止模型执行破坏性 shell 命令？当前主要靠沙箱和人工确认
- 工具调用的可观测性：调用链追踪、成本归因（哪个工具调用花了多少时间和 token）
- Computer Use 的安全模型：agent 操作 GUI 时的权限控制尚无成熟方案

## 与其他维度的耦合

**× Reasoning**：Reasoning model 改变了 tool use 的交互模式。经典 ReAct 是「想一步→做一步」紧密交错，reasoning model 倾向于「先在 thinking 中完整规划→再批量执行工具调用」。这种模式更高效但更不透明。此外，reasoning model 在工具描述理解上明显更强，允许更复杂的工具定义。

**× Multimodality**：Computer Use 本质上是 Tool Use 和 Multimodality 的交叉产物——没有视觉理解能力就无法操作 GUI。两个维度的能力水位共同决定了 Computer Use 的可行性上限。

**× Instruction Following**：Tool use 的可靠性高度依赖模型遵循 schema 约束的能力。strict mode 本质上是在推理时强制提升 instruction following。工具描述中的约束条件（参数范围、调用前提）需要模型精确遵循。

**× Long Context**：工具描述消耗上下文窗口。当可用工具数量从几个增长到几百个（MCP 生态下的常态），工具描述可能占据大量 context。这催生了「工具检索」需求——不把所有工具描述塞进 context，而是根据当前任务动态选择相关工具。

**× Memory**：工具使用历史是 agent 记忆的重要组成部分。一个有记忆的 agent 应该记得之前调用过哪些工具、结果如何——这影响后续的工具选择策略。

## 关键不确定性

- **CLI tool use 会被标准化吗？** 目前 MCP 标准化了 API 路径，Computer Use 有 benchmark，但 CLI 路径既没有标准协议也没有专门评测。这个空白是会被填补（出现 CLI 版的 MCP），还是会被 Computer Use 的进步消解（直接操作终端 GUI）？
- **Computer Use 的可靠性天花板在哪？** 当前 50-70% 的成功率足够某些用例，但离可信赖的自主操作还有距离。这个上限由模型能力决定还是由 GUI 本身的复杂性决定？
- **MCP 生态会不会出现碎片化？** 17,000+ 服务器的质量参差不齐，缺乏有效的发现和信任机制。这会走向统一市场还是出现几个竞争的子生态？
- **Reasoning model 会让 tool use 范式再变一次吗？** 当前迹象是 reasoning model 倾向于「想完再做」而非「边想边做」。如果这个趋势持续，工具编排的设计范式可能需要根本性调整。
- **结构化 API 和 Computer Use 的边界在哪？** 什么时候该用 API，什么时候该用 GUI？当前没有清晰的决策框架。

## 信息源

- **Benchmark 追踪**：BFCL V4（gorilla.cs.berkeley.edu/leaderboard，最全面的函数调用评测）；WebArena / OSWorld（Computer Use 评测）；SWE-Bench（间接评估 CLI tool use）
- **协议与标准**：MCP 官方博客（blog.modelcontextprotocol.io）和路线图（modelcontextprotocol.io/development/roadmap）；A2A 规范（Google Developers Blog）
- **个人博客 / Newsletter**：Lilian Weng（综述质量极高）；Martin Fowler 网站有一篇优秀的 function calling 架构文章；Sebastian Raschka 的 Ahead of AI
- **arXiv 关键词**：function calling, tool learning, tool-augmented LLM, constrained decoding, computer use agent, GUI agent
- **行业追踪**：OpenRouter 的 tool calling models 页面（openrouter.ai/collections/tool-calling-models）提供实时的模型 tool use 能力对比

## 参考资料

### 综述与框架性文章

- [LLM-Based Agents for Tool Learning: A Survey](https://link.springer.com/article/10.1007/s41019-025-00296-9) — Springer 上的系统综述，提出了 "Three Ws"（Whether / Which / How）框架
- [Function Calling: Structured Tool Use for Large Language Models](https://mbrenndoerfer.com/writing/function-calling-llm-structured-tools) — 对 function calling 机制的交互式深度解析
- [Function calling using LLMs](https://martinfowler.com/articles/function-call-LLM.html) — Martin Fowler 网站，偏架构视角，适合理解 tool use 在系统设计中的位置
- [The Guide to Structured Outputs and Function Calling with LLMs](https://agenta.ai/blog/the-guide-to-structured-outputs-and-function-calling-with-llms) — 结构化输出与函数调用的关系梳理，含 strict mode 的技术细节

### 训练与内部机制

- [How LLMs Are Trained for Function Calling](https://simplicityissota.substack.com/p/how-llms-are-trained-for-function) — 从 ToolFormer 到 Llama 系列的训练方法演进，技术细节丰富
- [ToolACE: Winning the Points of LLM Function Calling](https://openreview.net/forum?id=8EB8k6DdCU) — 自动化工具训练数据生成管线，26,507 API 的合成方法
- [Inside the Black Box: LLM Tool Calling Decisions](https://dasroot.net/posts/2026/04/inside-black-box-llm-tool-calling-decisions/) — 2026 年文章，分析模型内部如何做出工具调用决策
- [R2IF: Aligning Reasoning with Decisions via Composite Rewards for Interpretable LLM Function Calling](https://arxiv.org/html/2604.20316v1) — 用复合奖励对齐推理与工具调用决策

### 并行调用与编排优化

- [An LLM Compiler for Parallel Function Calling](https://arxiv.org/pdf/2312.04511) — LLMCompiler，ICML 2024，自动优化并行工具调用的调度
- [An LLM-Tool Compiler for Fused Parallel Function Calling](https://arxiv.org/html/2405.17438) — 进一步的融合优化，运行时合并同类工具操作
- [Natural Language Tools: A Natural Language Approach to Tool Calling](https://arxiv.org/html/2510.14453v1) — 提出用自然语言替代 JSON 格式做工具调用，准确率提升 18.4%，输出方差降低 70%

### Benchmark 与评测

- [Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html) — 最全面的函数调用 benchmark，覆盖 AST 评估、并行调用、多轮交互
- [BFCL: From Tool Use to Agentic Evaluation of LLMs](https://openreview.net/forum?id=2GmDdhBdDk) — BFCL 的学术论文，ICML 2025
- [LLM Agent & Tool-Use Benchmarks Rankings 2026](https://benchlm.ai/llm-agent-benchmarks) — BenchLM 的综合排名，含 agentic 能力评分
- [Best AI for Tool Calling 2026](https://llm-stats.com/leaderboards/best-ai-for-tool-calling) — 实时更新的 tool calling 模型排名

### Computer Use / GUI Agent

- [Computer Use and GUI Agents in 2026: State of the Art](https://zylos.ai/research/2026-02-08-computer-use-gui-agents) — 2026 年 2 月的全景综述
- [The 2025-2026 Guide to AI Computer-Use Benchmarks and Top AI Agents](https://o-mega.ai/articles/the-2025-2026-guide-to-ai-computer-use-benchmarks-and-top-ai-agents) — benchmark 导览
- [Agentic Computer Use: Ultimate Deep Guide 2026](https://o-mega.ai/articles/agentic-computer-use-the-ultimate-deep-guide-2026) — 深度实践指南

### CLI Agent 与 Coding Agent

- [Inside the Agent Harness: How Codex and Claude Code Actually Work](https://medium.com/jonathans-musings/inside-the-agent-harness-how-codex-and-claude-code-actually-work-63593e26c176) — 拆解 coding agent 的内部工作机制
- [Claude Code vs Codex App in 2026: Local Agent Pairing vs Cloud Agent Orchestration](https://www.developersdigest.tech/blog/claude-code-vs-codex-app-2026) — 两种架构范式的对比分析
- [I Tested 13 Local LLMs on Tool Calling — 2026 Eval Results](https://www.jdhodges.com/blog/local-llms-on-tool-calling-2026-pt1-local-lm/) — 本地模型 tool calling 能力实测，发现训练方法和架构比参数量更重要

## 更新日志

- 2026-04-26：初次创建。覆盖基础机制、三种调用形态、能力进阶、MCP/A2A 生态、当前格局、四层影响
