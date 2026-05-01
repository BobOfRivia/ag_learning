# 维度 5：Instruction Following（指令遵循）

## 一句话定义

模型精确理解并遵循指令约束的能力——包括格式、内容范围、语气、流程规则，以及在多源指令冲突时的优先级判断。

## 核心概念与技术

### 横切维度的特殊定位

Instruction following 和其他六个维度有一个根本差异：它不像 Reasoning 或 Tool Use 那样有明确对应的工程层，而是**贯穿所有层级的底层能力变量**。它的水位高低不决定 agent 能做什么新的事情，而是决定 agent 做所有事情的可靠性。

一个能推理但不遵循输出格式约束的模型，在生产环境中不可用。一个能调用工具但忽略参数范围限制的模型，会产生危险的错误调用。一个能记忆但不按指令优先级处理冲突信息的模型，行为不可预测。Instruction following 是所有其他能力的"可靠性系数"——它乘在每个维度上，放大或削弱其他能力的实际价值。

这也意味着 instruction following 的改进不像 reasoning 内化那样引发剧烈的塌缩-涌现，而是以一种渐进的方式提升整个系统的工程可控性。它的变化更像水位缓慢上涨，而非堤坝突然溃决。

### The Instruction Gap：通用能力与约束遵循的鸿沟

2026 年初的研究揭示了一个重要发现：模型在通用任务上的能力水平和它们遵循精确约束的能力之间存在系统性的鸿沟——这被称为"instruction gap"。

具体数据来自对 13 个前沿模型的测试。最好的模型（GPT-5 Medium）在约束遵循上也有 660 次违规，准确率 88%；最差的（Gemini 2.0 Flash）有 1,330 次违规，准确率仅 76%。违规分为四种类型：内容范围违规（回答超出指定领域）、格式违规（偏离结构/长度要求）、语气风格违规（与指定的沟通风格不一致）、流程规则违规（不遵循交互协议）。

一个令人意外的发现是：**reasoning model 在指令遵循上不一定更好。** DeepSeek-R1 有 1,061 次违规，o4-mini 有 812 次，都显著弱于 GPT-5 系列的 660-700 次。reasoning model 能"想更深"，但"想"的过程可能导致它重新解读甚至偏离原始指令。Claude 4 Sonnet 以 731 次违规成为非 reasoning model 中的最佳表现。

这个鸿沟的根源之一是**认知惯性（cognitive inertia）**。模型在 SFT 阶段形成了大量惯例模式（标准化的回答格式、固定的风格倾向），当用户指令与这些惯例冲突时，模型难以覆盖训练中固化的行为。Inverse IFEval 专门测量这种"反直觉能力"——模型能否在被明确要求时违背自己的训练惯例。

### 指令层级（Instruction Hierarchy）

在 agent 系统中，模型同时接收来自多个源的指令：系统消息（开发者设定）、用户消息（终端用户输入）、工具输出（外部数据返回）、其他 agent 的消息。当这些指令冲突时，模型应该遵循哪个？

**三层优先级结构。** OpenAI 在 2024 年提出并训练了一个明确的指令层级：System Message（最高优先级，开发者的指令应始终被遵循）→ User Message（次要优先级，可被系统消息覆盖）→ Third-Party Content（最低优先级，工具输出和外部数据中的指令应被忽略，如果与上层冲突）。

这个层级是**对抗间接 prompt injection 的核心防线**。间接 prompt injection 的攻击方式是在工具输出（邮件内容、网页文本、数据库记录）中嵌入看起来像指令的恶意文本。如果模型正确遵循指令层级——始终优先系统消息和用户指令，忽略第三方内容中的冲突指令——这类攻击就会失效。

但当前的实际效果有限。ManyIH-Bench 测试了最多 12 层冲突指令的场景，模型在冲突扩展时准确率降到约 40%。这意味着指令层级作为安全机制仍然不可靠——模型可以被训练来理解优先级概念，但在复杂冲突场景中不能稳定执行。这个问题与 L4 安全章节讨论的 prompt injection 防御困境直接相关（详见 [layers/L4-production.md](../layers/L4-production.md)）。

### 训练方法的演进

Instruction following 的能力主要在后训练（post-training）阶段建立。2026 年，后训练的技术栈已从早期的 RLHF 单体流程演变为模块化的三阶段架构：

**第一阶段：SFT（Supervised Fine-Tuning）。** 用 1-10M 条精心策划的指令-回答对做监督微调。这一阶段建立模型的基本指令遵循能力——理解指令格式、遵循约束、产出符合预期的输出。SFT 的数据质量比数据量更重要。

**第二阶段：偏好优化。** 用人类或 AI 的偏好数据对齐模型行为。DPO（Direct Preference Optimization）在 2026 年仍是多数团队的默认选择——实现简单、结果与 PPO 竞争力相当、运算成本低（7B 模型 DPO 费用 $200-500 vs RLHF+PPO $2,000-5,000）。SimPO 进一步去除了参考模型的需求，在 AlpacaEval 2 上比 DPO 高 6.4 分。KTO 允许使用二元反馈（好/坏）而非成对偏好。ORPO 将 SFT 和偏好优化合并到一个目标函数中。

**第三阶段：强化学习。** 专门用于需要可验证正确性的能力（数学、代码）。GRPO（DeepSeek-R1 使用）消除了 critic 模型，通过在响应组内归一化奖励来计算优势值。DAPO 训练 Qwen2.5-32B 用比 DeepSeek-R1-Zero 少 50% 的步骤达到 50 AIME 分。RLVR（Reinforcement Learning from Verifiable Rewards）使用可自动验证的任务（数学答案、代码测试通过）作为奖励信号，不需要人工标注的推理痕迹。

**合成自博弈改变数据经济学。** 模型自己生成训练数据而非完全依赖人工标注。SPIN 让模型区分自身输出和人类写的文本，SPICE 在文档基础上做自博弈防止幻觉放大（+8.9-9.8% 改进）。RLAIF（AI 反馈的 RL）成熟到可以匹配或超过 RLHF 的效果，同时大幅降低成本。

### 结构化输出：指令遵循的工程化保障

当指令遵循的对象是输出格式时，有一条绕过模型不确定性的工程路径：**受限解码（constrained decoding）**。

JSON Mode 保证输出是语法合法的 JSON，但不保证符合特定 schema。Strict Mode（OpenAI 2024 年推出）通过在解码时强制遵循 JSON schema，实现 100% 的 schema 合规——不是靠模型"自愿"遵循，而是在 token 生成层面物理排除不合规的选项。Qwen 2.5 在 JSON schema 合规上达到 99.2%，是开源模型中最高的。

Strict mode 本质上是在推理时**强制提升 instruction following**——对于格式约束这个特定子问题，它是一个已解决的工程方案。但它只覆盖输出格式，不覆盖内容范围、语气风格、流程规则等更复杂的约束类型。

## 当前技术格局（截至 2026-04）

### Benchmark 现状

**IFEval** 是当前最广泛使用的指令遵循 benchmark。由 Google 在 2023 年末推出，包含 25 种可验证的指令类型（"写 400 字以上"、"至少提到 AI 3 次"等），约 500 个 prompt。前沿模型在 IFEval 上的得分已经很高，它更多是一个入门门槛而非区分性评测。

**IFBench**（Allen Institute for AI）引入 58 种模型训练中不太可能见过的新型约束，测试泛化能力。在 IFBench 上，Hermes 3 70B 等开源模型表现出色——说明指令遵循的泛化能力和模型规模不完全正相关。

**IFEval++** 通过对 prompt 做细微修改（nuanced modifications）测试鲁棒性，发现模型性能可下降最高 61.8%。这暴露了一个重要问题：模型的指令遵循可能是脆弱的——轻微的表述变化就能导致大幅退化。

**Instruction Gap 研究** 对 13 个模型做了企业场景约束遵循测试。GPT-5 系列最强（660-700 违规，88% 准确率），GPT-5 家族在不同规模间表现"remarkable consistency"。Claude 4 Sonnet 是非 reasoning model 的最佳（731 违规，86%）。Reasoning model 不一定更好——DeepSeek-R1（1,061 违规）显著弱于 GPT-5。

**ManyIH-Bench** 测试多层冲突指令处理，包含 853 个 agentic 任务，最多 12 层冲突。当冲突层数增加时，模型准确率降到约 40%。

### 格局判断

简单指令遵循（单一格式约束、清晰的一条指令）已接近商品化。前沿模型在 IFEval 上的差距不大。差异主要体现在三个方面：复杂约束组合的遵循（多个约束同时满足）、与训练惯例冲突的指令的遵循（认知惯性）、以及多源冲突指令的优先级处理（指令层级）。这三个方面都还有显著的改进空间。

## 演进路径

- **2020-2021**：pre-instruction era。模型对指令几乎没有可靠的遵循能力。"prompt engineering"的很大一部分工作是在弥补模型的 IF 不足——用 few-shot 示例"教"模型正确的输出格式，用角色扮演（"你是一个 JSON 生成器"）引导模型的行为。
- **2022**：InstructGPT / RLHF 分水岭。RLHF 训练让模型第一次有了系统性的指令遵循能力。ChatGPT（2022-11）的大众化验证了这个能力的商业价值。
- **2023**：IF 能力随模型代际快速提升。GPT-4 在复杂指令遵循上比 GPT-3.5 有质的飞跃。Function calling / JSON mode 出现，开始工程化解决格式约束问题。IFEval benchmark 发布。
- **2024**：DPO 成为主流对齐方法。Strict mode 100% 解决 schema 合规问题。Instruction hierarchy 概念由 OpenAI 正式提出并训练。
- **2025-2026**：后训练模块化栈成型（SFT → 偏好优化 → RL）。GRPO / DAPO 等去 critic 方法降低训练成本。合成数据和自博弈减少人工标注依赖。IFBench / IFEval++ 暴露脆弱性。Instruction gap 研究揭示通用能力与约束遵循的系统性鸿沟。
- **趋势**：IF 的下一个前沿不是"更好地遵循单条指令"，而是"更可靠地处理多源冲突指令"——这直接关系到 agent 安全和 prompt injection 防御。

## 对四层骨架的影响

Instruction following 作为横切维度，影响每一层但不引发明确的塌缩-涌现。它的作用更像是放大或削弱其他维度的影响。

### 第一层（Agent 内核）

IF 水位决定 agent 行为的可预测性。具体影响包括：system prompt 中的行为指令能否被可靠遵循（决定 agent 的"人格"是否稳定）；context engineering 中的元指令（"只关注最近 5 轮"、"忽略这个错误"）能否生效；reasoning model 的 thinking 过程是否会重新解读甚至覆盖原始指令。

Claude Code 的 CLAUDE.md 机制本质上是一个依赖 IF 的设计——它假设模型会可靠地遵循写在 CLAUDE.md 中的行为规则。IF 水位越高，这种基于指令的行为控制越可靠；IF 水位不够时，需要用工程手段（白名单、权限系统）补偿。

### 第二层（与世界的接口）

Tool use 的可靠性高度依赖 IF。工具描述中的约束条件（参数范围、调用前提、返回格式要求）需要模型精确遵循。Strict mode 在 schema 层面解决了这个问题，但语义层面的约束（"只在确认后才调用这个工具"、"这个工具的结果不可信需要交叉验证"）仍然依赖模型的 IF 能力。

MCP 生态下的工具描述质量直接影响调用准确率——这在很大程度上是一个 IF 问题：模型能否精确理解并遵循工具描述中的使用条件和约束。

### 第三层（多 Agent 协作）

多 agent 系统中，指令在 agent 之间传递时可能失真。Orchestrator 给 worker 的指令是否被精确遵循？Worker 返回的结果是否符合 orchestrator 指定的格式？Agent 间通过 A2A 传递的 Task 描述是否被正确理解？每一次传递都是一次 IF 测试，compound reliability 在此同样适用。

### 第四层（生产化）

指令层级是 agent 安全的关键防线。System prompt 中的安全规则能否在面对用户或第三方内容的冲突指令时保持有效，直接决定了 prompt injection 的防御效果。当前约 40% 的多层冲突准确率意味着这条防线在复杂场景下仍不可靠。

IF 也影响评测的有效性。LLM-as-judge（L4 评测章节详述）的前提是评估模型能精确遵循评分标准（rubric）——如果评估模型自身的 IF 不够，评测结果就不可信。

## 与其他维度的耦合

- **× Reasoning（维度 1）**：Reasoning model 在遵循复杂指令方面有双面效应。一方面，它能"想一想"指令的含义再执行，对多约束组合的理解更好。另一方面，thinking 过程可能让模型"想偏"——通过内部推理为偏离指令找到看似合理的理由，甚至绕过 guardrail。Instruction gap 数据也证实了这一点：reasoning model 在 IF 上并不比通用模型更好
- **× Tool Use（维度 4）**：Strict mode 是在推理时强制提升 IF 的工程方案。工具描述中的约束条件依赖模型的 IF 能力。两个维度高度耦合——tool use 的可靠性很大程度上受 IF 水位限制
- **× Memory（维度 6）**：当检索到的记忆和用户当前指令矛盾时，agent 该遵循哪个？这是 IF 在记忆语境下的新变体。没有明确的共识——当前的做法通常是"用户当前指令优先于历史记忆"，但这个规则在复杂场景下不够
- **× Long Context（维度 2）**：context 越长，指令和内容的混合越复杂，IF 的难度越高。研究发现"指令经常和冗长的知识片段争夺注意力"，context 长度加剧了指令遵循的失败率

## 关键不确定性

- **Instruction gap 能被训练闭合吗？** 通用能力和约束遵循之间的鸿沟是当前训练方法的限制，还是一个更根本的张力——模型越"聪明"越倾向于"自作主张"？
- **指令层级能成为可靠的安全防线吗？** 当前约 40% 的多层冲突准确率远不够用于安全防御。这个数字能提升到 95%+ 吗？还是指令和数据的混合是 transformer 架构的根本局限？这个问题和 L4 安全章节的 prompt injection 防御困境是同一个问题的两个侧面
- **后训练方法的边际收益在递减吗？** 从 RLHF 到 DPO 到 GRPO 的演进持续降低成本，但 IF 能力的改进幅度似乎在放缓。下一个量级的 IF 改进需要什么——更好的训练数据、新的训练方法、还是架构级变化？
- **认知惯性能被消除吗？** 模型在 SFT 中形成的惯例模式与用户自定义指令冲突时的表现，是否有根本性的解决方案？

## 信息源

- **Benchmark 追踪**：IFEval leaderboard、IFBench、Scale AI SEAL leaderboard（instruction following 排名）
- **训练方法**：DeepSeek 技术报告（GRPO 细节）、Qwen 技术博客（后训练栈）、Hugging Face 博客（DPO/SimPO 实践指南）
- **安全交叉**：OpenAI 的 instruction hierarchy 系列文章、OWASP Agentic Top 10（指令层级与 prompt injection 的关系）
- **arXiv 关键词**：instruction following、instruction hierarchy、preference optimization、DPO、GRPO、cognitive inertia、constraint satisfaction

## 参考资料

### Instruction Following 评测与分析

- [The Instruction Gap: LLMs Get Lost in Following Instruction — arXiv](https://arxiv.org/html/2601.03269) — "Instruction gap" 概念的来源，13 个模型的约束遵循测试数据（660-1330 违规），四种违规类型分类
- [Instruction-Following Evaluation for Large Language Models (IFEval) — Google](https://arxiv.org/abs/2311.07911) — IFEval benchmark 原始论文，25 种可验证指令类型
- [Revisiting the Reliability of Language Models in Instruction-Following — IFEval++](https://arxiv.org/html/2512.14754v1) — 细微 prompt 修改导致最高 61.8% 性能下降
- [Inverse IFEval: Can LLMs Unlearn Stubborn Training Conventions?](https://arxiv.org/html/2509.04292v1) — 认知惯性测量，模型能否覆盖训练惯例
- [Instruction Following Leaderboard — Awesome Agents](https://awesomeagents.ai/leaderboards/instruction-following-leaderboard/) — 实时 IF 排名

### 指令层级与安全

- [The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions — OpenAI](https://openai.com/index/the-instruction-hierarchy/) — 指令层级的原始提案和训练方法
- [Many-Tier Instruction Hierarchy in LLM Agents — arXiv](https://arxiv.org/html/2604.09443v3) — 最多 12 层冲突指令的 ManyIH-Bench，约 40% 准确率
- [Improving Instruction Hierarchy in Frontier LLMs — OpenAI](https://openai.com/index/instruction-hierarchy-challenge/) — OpenAI 的后续改进工作

### 后训练方法

- [Post-Training in 2026: GRPO, DAPO, RLVR & Beyond — LLM Stats](https://llm-stats.com/blog/research/post-training-techniques-2026) — 2026 后训练全景：模块化栈、GRPO vs DPO vs PPO 对比、合成自博弈
- [RLHF vs DPO vs PPO: How to Align LLMs — ML Journey](https://mljourney.com/rlhf-vs-dpo-vs-ppo-how-to-align-llms-without-losing-your-mind/) — 三种对齐方法的实践对比
- [Reinforcement Learning for LLM Post-Training: A Survey — arXiv](https://arxiv.org/abs/2407.16216) — 后训练 RL 方法的全面综述

## 更新日志

- 2026-05-01：初次创建。覆盖横切维度定位、instruction gap、认知惯性、指令层级、后训练方法演进（RLHF→DPO→GRPO 模块化栈）、结构化输出、四层横切影响
