# 维度 3：Multimodality（多模态）

## 一句话定义

模型原生理解和生成多种信息形态（文本、图像、音频、视频）的能力，以及由此扩展的 agent 感知和行动边界。

## 核心概念与技术

### 关键区分：适配器式 vs 原生多模态

这是理解当前多模态格局的核心架构区分。

**适配器式多模态（Adapter-based）。** 绝大多数现有多模态模型采用这种架构：一个预训练的视觉编码器（ViT、CLIP 或 SigLIP）将图像转换为向量表示，通过一个中间连接器（线性投影、perceiver resampler 等）投射到语言模型的嵌入空间。视觉能力是"嫁接"到已有语言模型上的。训练过程分阶段进行：视觉编码器预训练 → 视觉-语言对齐 → 多模态微调 → RLHF/DPO 对齐。LLaVA、Claude 的视觉能力、Qwen2.5-VL 都属于这种架构。优势是可以复用已有的强语言模型，训练效率高；劣势是模态之间的融合深度受限于连接器的表达能力，跨模态推理能力有天花板。

**原生多模态（Native Multimodal）。** 从头在混合模态数据上训练单一 transformer，不依赖独立的视觉编码器或适配器。Gemini 系列是这种架构的代表——文本、图像、音频、视频在同一个模型内部以统一的 token 流处理。GPT-4o 也采用了原生多模态设计，这使它能在同一个模型内同时理解和生成文本与图像。2025-2026 年的最新一代模型正在从适配器式向原生多模态过渡，但这个过渡并不彻底——很多"原生"宣称实际上仍在内部使用某种形式的模态专用编码器。

两种架构的差异在实际使用中的体现：原生多模态模型在需要跨模态推理的任务上（如"理解图表中的趋势并用语言解释其含义"）通常更强，因为模态之间的信息交换不经过瓶颈层。但适配器式架构在纯视觉理解任务上并不一定弱于原生架构——模态专用编码器在自己的领域内可能更精准。

### 理解 vs 生成：不对称的成熟度

多模态能力有两个方向：理解（模型接收非文本输入）和生成（模型产出非文本输出）。两者的成熟度差距很大。

**理解端已相当成熟。** 视觉理解是最成熟的非文本能力。前沿模型在 OCR、图表分析、文档理解、空间关系理解、截图解读等任务上已达到商用水平。InternVL3-78B 在 MMMU benchmark 上达到 72.2，创下开源 SOTA。音频理解也在快速成熟——GPT-4o 原生支持音频输入，Qwen3.5-Omni 支持 113 种语言/方言的语音识别。视频理解可用但尚处发展期——Gemini 3.1 Pro 原生支持长视频输入（利用 1M context window），但视频理解的精度和时间推理能力仍有差距。

**生成端仍在分化。** 图像生成在 2025 年出现了路径分化。GPT-4o 在 2025 年 3 月将图像生成直接内建到模型中——不是后台调用 DALL-E，而是原生的多模态生成，用户可以在对话中迭代修改图像，模型理解跨轮次的上下文。这是"原生多模态生成"的里程碑。Claude 走了另一条路：不生成像素级图像，而是通过生成 SVG 代码、React 组件、交互式 artifacts 来创建视觉内容——本质上是用代码生成能力"间接"生产视觉输出。语音生成已有实时交互能力——GPT-4o 的 Voice Mode、Qwen3.5-Omni 的 Talker 模块都支持流式语音生成。视频生成仍处早期。

### 模态覆盖谱系

当前模型的多模态覆盖形成一个清晰的谱系：

**Text-only** → **Vision + Text**（视觉理解，最普及） → **Vision + Text + Audio**（加入语音理解和生成） → **Omni**（文本、图像、音频、视频的全方位理解和生成）

大多数生产系统停留在 Vision + Text 层级。完整的 Omni 能力目前由少数模型提供：Gemini 3.1 Pro（最广的模态覆盖，原生支持文本+图像+音频+视频输入）、GPT-4o（文本+图像+音频的双向能力）、Qwen3.5-Omni（开源 Omni 模型标杆，Thinker-Talker 架构支持实时语音交互）。

### Omni 模型的架构创新

2025-2026 年涌现的 Omni 模型在架构上有值得注意的创新。

Qwen3.5-Omni 采用 **Thinker-Talker 架构**：Thinker 模块使用 Hybrid-Attention Mixture of Experts 跨所有模态进行理解和推理，Talker 模块接收 Thinker 的表示生成流式语音。这种"思考-说话"分离的设计使得推理和生成可以并行，支持实时交互。

NVIDIA Nemotron 3 Nano Omni 采用 **模态路由 MoE**：30B 总参数但每次推理仅 3B 活跃，根据输入模态路由到不同的专家模块。在文档理解、视频理解、音频理解六个 benchmark 上排名第一，声称比同级开源模型有 9 倍吞吐量优势。设计目标是在单 GPU 上运行，适合边缘部署。

这些架构创新的共同方向是：**用更高效的参数利用方式覆盖更多模态，而非简单地堆砌参数。**

### Computer Use：Multimodality × Tool Use 的交叉产物

Computer Use 是这个维度对 agent 工程影响最大的具体产物。它让 agent 第一次能操作没有 API 的系统——理论上任何人能用的软件，agent 都能用。这个能力的实现依赖视觉理解（感知屏幕内容）和工具操作（生成鼠标键盘动作）的交叉，因此在维度 4 Tool Use 和本维度都有覆盖。维度 4 从工具形态的角度讨论 Computer Use 的三种调用模式和 MCP 生态定位；这里从多模态能力的角度讨论其感知架构和成熟度。

**三种感知架构。** GUI agent 的核心设计选择是如何感知界面：

截图视觉（Screenshot-based Vision）：把 UI 当作图像输入视觉语言模型，模型输出点击坐标。通用性最强（任何界面都能用），但 token 消耗高（每个动作 15,000+ tokens），推理速度慢（2-5 秒/动作），对小元素的坐标精度有限。OpenAI Operator 采用这种方案。

可访问性树解析（Accessibility Tree Parsing）：通过平台 API（Windows UI Automation、macOS Accessibility API、Android Service）获取界面的结构化语义表示。高效（token 消耗比视觉方案低 50-100 倍）、定位精确，但覆盖有限——自定义 UI、Canvas 绘制的界面对可访问性 API 不可见。Agent-E 在静态网站上达到 95.7%，但在动态网站上降到 27.3%。

DOM/View Hierarchy 操纵：直接访问 web 页面的 DOM 或移动应用的视图层级。最精确、延迟最低，但仅限 web 或特定平台，且受反爬措施、Shadow DOM 等技术障碍影响。

**2026 行业共识：混合架构。** 单一感知方案各有盲区，行业已收敛到混合架构。Microsoft UFO² 的做法是代表：先从可访问性树提取控件，再用视觉模型（OmniParser-v2 + YOLO-v8）识别可访问性 API 覆盖不到的自定义控件，最后通过边界框重叠去重。Browser-Use（混合）在 WebVoyager 上达到 89.1%，而 Agent-E（纯可访问性）仅 73.1%。

**各平台成熟度差异显著。** Web 浏览器 agent 最成熟——OpenAI ChatGPT agent 在复杂 JavaScript 网站上达到 87%，Google Project Mariner 在 WebVoyager 上达到 83.5%。多家已向付费用户开放（$20-30/月）。Mobile agent 在受控 benchmark 上表现突出——Mobile-use 框架在 AndroidWorld 上达到 100% 成功率——但真实环境的应用碎片化仍是挑战。Desktop agent 差距最大——OSWorld 上最优 agent 仅 20.58%（人类基线 72.36%），跨应用工作流（"从邮件取数据→更新表格→发 Slack"）成功率仅 12-20%。

**当前定位：增强而非替代。** 2026 年的 Computer Use 擅长有人类监督的受限工作流（信息检索、表单填写），但无法胜任无人监督的高风险操作、开放式跨应用自动化、或需要 99.9%+ 可靠性的任务。行业已从"能不能用"进入"怎么让它可靠"的阶段，但从 70% 监督演示到 99% 自主生产，预期需要数年而非数月。

## 当前技术格局（截至 2026-04）

### 闭源模型

**Gemini 3.1 Pro** 拥有最广的模态覆盖。原生支持文本、图像、音频、视频输入，1M context window 可处理长视频。在视频理解和多模态推理上的综合表现最强。但不支持原生图像生成。

**GPT-4o** 是原生多模态生成的先行者。2025 年 3 月内建图像生成，支持文本+图像+音频的双向交互。在截图理解、图表分析等视觉理解任务上表现领先。Voice Mode 提供实时语音对话。

**Claude Opus 4.7 Adaptive** 在视觉理解方面强于文档和多页推理场景。不支持原生图像生成或音频处理——走代码生成路线（SVG、React artifacts）产出视觉内容。Computer Use 以公开 beta 形式提供，要求在 VM/容器中运行以保证安全。

### 开源模型

**InternVL3-78B** 在 MMMU 上达到 72.2，创开源多模态 SOTA。视觉理解全面，但不支持音频或视频。

**Qwen3.5-Omni** 是开源 Omni 模型标杆。Thinker-Talker 架构支持文本+图像+音频+视频理解，以及实时语音生成。113 语言语音识别、36 语言语音合成。

**Qwen2.5-VL** 系列在视觉语言任务上表现突出，32B 规模在多个 benchmark 上与闭源模型竞争。

**Molmo 72B** 在学术 benchmark 上超过 Gemini 1.5 Pro 和 Claude 3.5 Sonnet，值得关注。

**NVIDIA Nemotron 3 Nano Omni** 面向边缘部署的 Omni 模型，30B 参数 3B 活跃，单 GPU 可运行。

### 关键 benchmark

**视觉理解**：MMMU（多学科多模态理解，当前开源 SOTA 72.2）、MathVista（视觉数学推理）、DocVQA（文档问答）、ChartQA（图表理解）。

**Computer Use**：WebArena（web 导航，top 71.2%，多数 50-60%）、OSWorld（桌面操作，top 20.58%，人类 72.36%）、AndroidWorld（移动端，top 100%）、ScreenSpot（多模态 UI 理解，top 84%）、WebVoyager（真实 web 任务，top 87%）。

**Omni 能力**：OmniBench（跨模态理解和推理）。

### 格局判断

视觉理解已成为前沿模型的标准能力，开源与闭源的差距小于 reasoning 维度——视觉编码器的训练方法更成熟、更可复现。原生多模态生成（GPT-4o 路线）和代码间接生成（Claude 路线）的分化将持续。Omni 模型（同时覆盖所有模态）正从研究走向可用，但完整的 Omni 能力尚未成为 agent 系统的标准配置——大多数生产 agent 仍只使用 Vision + Text。Computer Use 的 web 端已进入早期生产，desktop 端仍需数年。

## 演进路径

- **2020-2022（第一代）**：纯文本。CLIP（2021）证明了视觉-语言对齐的可行性，但尚未进入 LLM 主流。
- **2023（第二代）**：GPT-4V（2023-09）引入视觉理解，适配器架构。Gemini 1.0（2023-12）以原生多模态架构面世。多模态能力从"有无"变成"好不好"。
- **2024**：GPT-4o（2024-05）标志原生多模态的产品化——同一模型同时处理文本、图像、音频。Anthropic 推出 Computer Use beta（2024-10），Tool Use × Multimodality 的交叉产物。视觉理解在前沿模型中普及。
- **2025**：GPT-4o 内建图像生成（2025-03），原生多模态生成的里程碑。实时语音交互成熟。Qwen3.5-Omni 等开源 Omni 模型涌现。Browser agent 向付费用户开放。
- **2026**：视觉理解商品化。Omni 模型架构创新（MoE 路由、Thinker-Talker）。Computer Use 的混合感知架构成为共识。Mobile agent 在受控环境达到 100%。Desktop agent 仍是难题。
- **趋势**：多模态能力的扩展方向从"更多模态"转向"更深融合"和"更可靠的行动"——模型已经能看、能听、能说，下一步的价值创造在于让这些感知能力可靠地转化为行动（Computer Use 的可靠性提升）和推理（跨模态推理的精度提升）。

## 对四层骨架的影响

### 第一层（Agent 内核）

**涌现**：
- 多模态输入改变 context engineering 的计算。图像 token 消耗显著——一张截图 15,000+ tokens，agent 如果频繁截图感知环境，context 膨胀速度远快于纯文本场景。截图频率和分辨率成为需要权衡的参数
- 视觉推理作为 reasoning 的新维度。模型需要同时理解视觉信息和文本信息，然后做跨模态推理（如"这个图表的趋势说明什么"）。这在 reasoning model 时代尤其重要——thinking tokens 中的视觉推理过程是否和文本推理一样可靠？

### 第二层（与世界的接口）

**塌缩**：
- 部分需要 API 集成的场景可以通过 Computer Use 直接操作 GUI 替代——尤其是没有 API 的遗留系统。但 Computer Use 的可靠性（50-70%）远低于 API 调用（99%+），所以这个塌缩是局部的、有条件的

**涌现**：
- Computer Use 作为全新的工具形态，需要全新的基础设施：截图管道、坐标映射系统、操作确认机制、失败恢复策略
- 视觉理解让 agent 能处理之前纯文本 agent 无法触及的信息类型——图表、UI 界面、手写文档、物理环境照片
- 端到端视觉工具使用（GLM-4.6V 的创新）——图像直接作为工具参数传入，工具的视觉输出直接被模型解释，不需要中间的文本转换

### 第三层（多 Agent 协作）

**涌现**：
- 模态专长分工成为多 agent 的新动机。视觉 agent（擅长截图理解和 GUI 操作）和文本 agent（擅长推理和代码生成）的分工可能比纯认知分工更有实际价值——因为不同模态确实需要不同的能力，不像"研究员 vs 批评者"那样可以被单个 reasoning model 取代

### 第四层（生产化）

**涌现**：
- Computer Use 的安全模型是全新的 L4 问题。Agent 操作 GUI 时的权限控制没有成熟方案——API 调用可以精确限制函数和参数，GUI 操作是视觉驱动的坐标动作，难以做细粒度权限控制。当前实践是 VM/容器沙箱 + 分层权限（Silent/Logged/Confirmed/Blocked）
- GUI 操作的评测体系。WebArena、OSWorld、AndroidWorld 等 benchmark 为 Computer Use 建立了标准化评测，但 benchmark 与真实场景的差距仍然显著
- 13.4% 的 GUI agent 技能存在至少一个关键级安全问题，攻击成功率达 84-85%——Computer Use 的安全问题比 API 调用严峻得多

## 与其他维度的耦合

- **× Tool Use（维度 4）**：Computer Use 是两者的交叉产物——没有视觉理解就无法感知 GUI，没有工具操作能力就无法执行动作。两个维度的能力水位共同决定了 Computer Use 的可行性上限。结构化 API 和 Computer Use 的选择是一个开放决策——当前没有清晰的判断框架（在维度 4 的开放问题中讨论）
- **× Long Context（维度 2）**：图像和视频 token 大量消耗 context window。一张截图 15,000+ tokens，一段视频可能消耗数十万 tokens。多模态 agent 面临比纯文本 agent 更尖锐的 context 管理压力
- **× Reasoning（维度 1）**：视觉推理能力（理解图表趋势、空间关系推理、UI 状态判断）是 reasoning + multimodality 的组合。Reasoning model 的 thinking 中是否能做可靠的视觉推理，是一个开放问题
- **× Memory（维度 6）**：多模态记忆（记住之前看到的截图、听到的语音）比纯文本记忆更复杂。当前的记忆系统几乎都是文本化的——视觉信息必须先转成文本描述才能存储和检索，这个转换过程有信息损失

## 关键不确定性

- **原生多模态 vs 适配器，哪条路线最终胜出？** Gemini 的原生路线和 Claude 的适配器+代码生成路线代表了两种不同的工程哲学。原生路线的上限可能更高（跨模态推理更自然），但训练成本也更高。这个竞争尚未有定论
- **Computer Use 能达到 API 级别的可靠性吗？** Desktop agent 20% vs 人类 72% 的差距巨大。这个差距是当前模型能力的限制（更强的模型就能解决），还是 GUI 环境本身的不确定性决定了上限？
- **图像生成的路径分化会收敛吗？** GPT-4o 的原生生成和 Claude 的代码生成服务于不同的使用场景。随着原生生成能力的进步，代码路线是否会被边缘化？还是两条路线会长期共存，服务于不同用户群？
- **Omni 模型会成为 agent 的标准配置吗？** 当前大多数 agent 只使用 Vision + Text。完整 Omni 能力（音频、视频双向）对 agent 的实际价值有多大？还是它更多服务于消费者产品（语音助手）而非 agent 工程？
- **多模态会成为下一个地基质变吗？** gen4-memory-era.md 已经提到，如果下一个地基跳变不是 memory 而是 multimodality（原生视觉-动作闭环催生 embodied agent），第四代的定义需要修正。embodied agent 的涌现信号值得持续关注

## 信息源

- **模型发布追踪**：各厂商的模型技术报告（GPT-4o system card、Gemini technical report、Qwen 技术博客）——每次发布关注多模态能力章节
- **Benchmark 追踪**：MMMU leaderboard、WebArena leaderboard、OSWorld leaderboard——用于判断视觉理解和 Computer Use 的能力水位变化
- **开源生态**：Hugging Face 的 Open VLM Leaderboard——跟踪开源视觉语言模型的进展
- **arXiv 关键词**：vision language model、multimodal LLM、GUI agent、computer use、omni model、native multimodality

## 参考资料

### 多模态模型全景

- [Multimodal LLMs: Complete Guide to Vision & Text Models (2026) — LLM Trust](https://www.llmtrust.com/blog/multimodal-llm-guide) — 架构演进（适配器 vs 原生）、模型能力对比矩阵
- [Best Multimodal AI Model 2026: Gemini vs Others — AIZolo](https://aizolo.com/blog/best-multimodal-ai-model-2026-gemini-vs-others/) — 2026 年多模态模型横向对比
- [Top 10 Vision Language Models in 2026 — DataCamp](https://www.datacamp.com/blog/top-vision-language-models) — 视觉语言模型排名和 benchmark 数据
- [The Best Open-Source Vision Language Models in 2026 — BentoML](https://www.bentoml.com/blog/multimodal-ai-a-guide-to-open-source-vision-language-models) — 开源多模态模型指南

### Computer Use / GUI Agent

- [Computer Use and GUI Agents in 2026: State of the Art — Zylos Research](https://zylos.ai/research/2026-02-08-computer-use-gui-agents) — 最全面的 2026 Computer Use 全景综述，含三种感知架构对比、各平台成熟度、安全数据
- [The 2025-2026 Guide to AI Computer-Use Benchmarks and Top AI Agents — O-Mega](https://o-mega.ai/articles/the-2025-2026-guide-to-ai-computer-use-benchmarks-and-top-ai-agents) — Benchmark 综述和 agent 排名
- [Computer-Using Agent — OpenAI](https://openai.com/index/computer-using-agent/) — OpenAI 的 Computer Use agent 技术说明
- [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://os-world.github.io/) — Desktop 操作 benchmark

### Omni 模型

- [Qwen3.5-Omni — Alibaba](https://github.com/QwenLM/Qwen3-Omni) — 开源 Omni 模型标杆，Thinker-Talker 架构
- [NVIDIA Nemotron 3 Nano Omni — NVIDIA Blog](https://blogs.nvidia.com/blog/nemotron-3-nano-omni-multimodal-ai-agents/) — 模态路由 MoE 架构，边缘部署 Omni 模型

### 学术综述

- [A Survey of State of the Art Large Vision Language Models — arXiv](https://arxiv.org/abs/2501.02189) — 2025 年初的 VLM 全景综述
- [Survey on Multimodal Large Language Models — National Science Review](https://academic.oup.com/nsr/article/11/12/nwae403/7896414) — 学术视角的多模态 LLM 综述

## 更新日志

- 2026-05-01：初次创建。覆盖适配器 vs 原生架构、理解 vs 生成的不对称成熟度、模态覆盖谱系、Omni 模型架构、Computer Use 三种感知架构和各平台成熟度、技术格局、四层影响
