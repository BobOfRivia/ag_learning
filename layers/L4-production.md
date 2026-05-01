# 第 4 层：生产化（Production）

## 这一层解决什么问题

L1-L3 关心的是"agent 能不能做到"，L4 关心的是"agent 能不能上线"。这一层覆盖让 agent 在真实环境中可靠、安全、经济地运行所需的全部工程基础设施：评测、可观测性、成本控制、安全、人机协同、可靠性保障。

L4 是第三代重要性提升最快的一层。在第一代和第二代，agent 的生产化几乎不是话题——agent 本身的可靠性都没解决，谈不上上线。第三代的 reasoning model 同时带来了更强的能力和更大的生产化挑战：成本飙升 3-10 倍、延迟从秒级跳到分钟级、thinking trace 引入新的可观测性需求、模型在思考链中"想偏"构成新的安全面。这些挑战让 L4 从"锦上添花"变成了"没有就不能上线"。

LangChain 2026 年的调查数据提供了一个当前断面：57.3% 的团队已有 agent 在生产中运行，89% 实现了某种形式的可观测性。但质量（32%）仍是首要障碍，延迟（20%）紧随其后。Gartner 预测 40% 的 agentic AI 项目将在 2027 年底前被取消——不是因为技术不可行，而是因为生产化的工程没跟上。

## 关键工程模式

### 评测（Evals）

Agent 评测和传统 ML 评测有根本差异。传统评测关心"模型在一个数据集上的准确率"，agent 评测关心的是"一个多步骤、非确定性、与环境交互的系统，在真实条件下能不能可靠地完成任务"。

**结果评测 vs 轨迹评测。** 这是 agent 评测的核心区分。结果评测（outcome eval）衡量任务最终是否完成——代码是否通过测试、问题是否被正确回答、表单是否被正确填写。轨迹评测（trajectory eval）衡量 agent 的完整执行路径——每一个推理步骤、每一次工具调用、每一个决策点是否合理。结果评测告诉你 agent 行不行，轨迹评测告诉你 agent 为什么行或为什么不行。对 reasoning model 而言，轨迹评测尤其关键：模型可能通过错误的推理路径偶然得到正确答案（lucky hit），也可能推理过程完全合理但因为一个工具调用失败而最终失败。只看结果会漏掉两类问题。

**Process eval 的兴起。** 第三代的涌现物。从"答对了吗"升级到"思考路径合理吗"。当 reasoning model 的 thinking trace 可见时（如 Anthropic 的 extended thinking），可以直接评估推理过程的质量；当 trace 被隐藏时（如 OpenAI 的 reasoning tokens），只能通过 reasoning summary 或最终输出推断推理质量。这是当前的一个开放工程问题——OpenAI 和 Anthropic 在 trace 可见性上做了截然相反的选择，直接影响了 process eval 的可行性。

**LLM-as-judge。** 用 LLM 评估 LLM 的输出，是当前 agent 评测的主流方法（53.3% 的团队采用）。优势是可扩展——人工评审是瓶颈，LLM 判断可以自动化运行。但有三个已知的系统性偏差：位置偏差（倾向于选择列表中靠前的答案）、长度偏差（倾向于给更长的回答更高分）、迎合偏差（倾向于认同被评估内容）。实测数据显示，LLM 评估器在复杂任务上的错误率超过 50%，在专业领域与人类专家的一致率仅 64-68%。缓解策略包括：随机化呈现顺序并多数投票、多次运行一致性检验（目标 Cronbach's alpha > 0.8）、提供显式评分标准和 few-shot 示例。生产环境中的目标是与人类评估者达到 0.80+ 的 Spearman 相关系数。

**Benchmark 到生产的鸿沟。** 企业 agentic AI 系统在实验室 benchmark 得分和真实部署表现之间存在 37% 的差距。更严峻的数字来自可靠性衰减：agent 在单次运行中成功率约 60%，但相同任务重复八次后整体成功率降至 25%。标准 benchmark 测不出这种可靠性问题。74% 的生产 agent 依赖 human-in-the-loop 评估而非标准化 benchmark——这本身就说明 benchmark 的局限性。

**生产评测的三层模型。** 当前生产团队正在收敛到一种三层评测架构：（1）单元评测（unit evals）——对离散步骤的确定性测试（路由正确性、schema 验证），在 CI 中运行，快速且确定性；（2）LLM-as-judge 回归套件——对主观质量按评分标准打分，作为 PR / 发布的门控；（3）生产 trace 采样——在真实流量上运行评估，捕捉分布漂移和渐进退化。缺少任何一层都会产生可预见的盲区。

**Agent 专用 benchmark。** 当前主要的 agent 评测基准按能力维度分布：SWE-Bench Verified（代码任务端到端完成）、WebArena / OSWorld（GUI 操作）、τ-Bench（多轮工具使用）、GAIA（复杂推理）、BFCL V4（函数调用）。这些 benchmark 各自衡量 agent 能力的不同切面，不存在一个综合性的 agent benchmark——这本身就是 L4 的一个开放问题。

### 可观测性与追踪（Observability & Tracing）

Agent 系统的可观测性需求和传统 APM（应用性能监控）有本质差异。传统 APM 追踪的是确定性的代码执行路径——函数调用链、HTTP 请求、数据库查询。Agent 系统的执行路径是非确定性的——同样的输入可能产生完全不同的推理步骤和工具调用序列。这意味着传统 APM 工具无法直接复用，需要专门的 agent 可观测性基础设施。

**四类必须追踪的信号。** 生产 agent 系统需要追踪：LLM 调用（输入/输出 token 数、模型选择、effort 级别、完成原因）、工具调用（参数、返回值、延迟、成功/失败）、检索步骤（查询内容、返回结果数、相关性评分）、规划决策（agent 选择了什么行动、拒绝了什么替代方案）。Token 计数、工具调用结果、LLM 调用完成原因必须作为一等指标，不是事后补充。

**标准 APM 工具漏掉的四种 agent 故障模式。** （1）工具调用失败——模型生成无效参数、无限循环调用、或捏造结果；（2）上下文截断——关键指令在 prompt 超过 context window 时被静默丢弃；（3）失控循环——agent 反复重试失败步骤，消耗大量 token 但不收敛；（4）静默质量退化——span 返回成功但输出质量在 prompt 或模型变更后下降。

**Reasoning trace 的可观测性困境。** 这是第三代特有的问题。Anthropic 的 thinking tokens 对用户完全可见，可以直接检查模型"想了什么"；OpenAI 的 reasoning tokens 对用户隐藏，只能通过 `output_tokens_details.reasoning_tokens` 查询消耗量和通过 reasoning summary 获取摘要。隐藏有利于 IP 保护和产品体验的简洁性，但严重损害了可调试性和安全审计能力——当模型在 thinking 中"想偏"绕过 guardrail 时，工程师看不到发生了什么。当前没有行业共识。

**工具格局。** Langfuse 是开源基准线，MIT 许可，支持自托管（需运行 ClickHouse + PostgreSQL + Redis 等 5+ 服务），框架无关，成本可预测性最好。LangSmith 是 LangChain 的商业平台，对 LangGraph 工作流有最强的原生支持（自动节点级可视化），包含 prompt 版本管理和人工标注队列。Braintrust 以评测为核心定位，提供一等的评分函数库（数值评分器、LLM-as-judge 模板、可组合自定义评分器），适合把评测开发作为主要工作流的团队。此外 Arize、Datadog LLM Observability 等平台提供企业级集成。

**OpenTelemetry 作为可移植层。** 基于 OTel GenAI 语义约定（`gen_ai.system`、`gen_ai.usage.input_tokens`、`gen_ai.response.finish_reasons`）做 instrumentation，可以将同一套数据发送到不同后端（LangSmith、Langfuse、Braintrust、自托管），只需配置变更，避免供应商锁定。

**采样策略。** 尾部采样（trace 完成后决定是否保留）优于头部采样（随机选择）。应 100% 保留：包含错误或超时的 trace、高成本 trace（token 消耗 top 5%）、低评分 trace。对健康且低成本的 trace 按 1-5% 采样，并按租户分层以保留小客户的可见性。

**漂移检测。** Golden-set replay 是最可靠的方法：维护 50-500 条精选 trace，每次部署或每天重放并追踪评分变化。配合生产端的评分分布追踪，区分模型退化和输入分布变化。

### 成本工程（Cost Engineering）

成本工程是第三代涌现出的全新 L4 学科。在第二代，LLM 调用成本相对固定且可预测，成本管理不是独立的工程问题。Reasoning model 改变了这一切：单次请求的 token 消耗方差极大（简单问题几百 token，复杂问题几万 token），输出 token 的价格是输入的 3-8 倍（2026 年中位数 4:1），thinking tokens 不可控地增长。团队如果不把成本作为一等工程关注项，和延迟、可靠性并列管理，agent 系统在经济上不可持续。

**Agent 的成本结构特征。** Agent 比简单 chatbot 贵 3-10 倍。一个用户请求触发规划、工具选择、执行、验证等步骤，token 消耗是直接 chat completion 的 5 倍。不加约束的 agent 解决一个软件工程任务的 API 费用可达 5-8 美元。成本增长的核心驱动力有三个：多轮循环的二次增长（10 轮 ReAct 循环消耗 50 倍于线性通过的 token）、输出 token 的价格溢价（reasoning model 达 8:1）、context window 扩展的注意力矩阵成本（128K window 的计算成本是 8K 的约 64 倍）。

**成本优化工具箱。** 当前已验证的成本优化手段，按节省幅度排序：

Prompt caching 是投资回报率最高的单一优化。缓存 token 的成本约为非缓存的 10%（Anthropic 定价），延迟降低 75-85%。适用于静态 system prompt、RAG pipeline 的固定前缀、多步 agent loop 中的重复前缀。关键实践在 context engineering 中已详述：保持 context 前缀稳定、只追加不修改、确保序列化确定性。

Model routing 将任务复杂度匹配到对应能力级别的模型。90% 的查询可以由小模型处理（Gemini Flash、Haiku 等），仅复杂任务升级到 reasoning model。整体可节省约 87% 的成本。LiteLLM、Portkey、OpenRouter 等中间件已内置路由能力。与维度 7 inference-time compute 中讨论的两层路由（选模型 × 选 effort 级别）直接对应。

Semantic caching 通过向量相似度识别语义等价的重复查询，约 31% 的 LLM 查询在典型工作负载中存在语义相似性。缓存命中时成本为零，响应从秒级降到毫秒级。但存在对抗性缓存投毒的风险，需要相似度阈值调优。

Prompt 压缩通过去除冗余 token 降低输入成本。LLMLingua 等工具可达 20 倍压缩比。对摘要和问答类任务质量损失可接受，但对需要精确细节的场景（错误信息、文件路径）有精度损失。

Batch API 对所有模型提供约 50% 折扣，结果在 24 小时内返回。适用于文档摘要、离线分析、合成数据生成等非实时场景。

综合使用压缩、路由、缓存等手段，可以在不显著损失质量的前提下实现 60-80% 的总成本降低。

**FinOps：成本归因与监控。** 大多数团队知道总支出但缺乏细粒度归因——不知道哪个模型、哪个 prompt、哪个工作流、哪个用户驱动了成本。需要追踪的关键指标包括：每 trace / 每工作流运行的成本、每用户成本、每模型层级成本、缓存命中率、每次工具调用的 token 消耗、输出 token 占比。成本归因的最佳实践是"根节点打标签，向子节点传播"：在请求入口附加 user_id、tenant_id、task_type，确保所有子 span 继承这些标签。三个维度驱动生产决策：per-user（检测滥用、支撑按量计费）、per-task/feature（揭示哪些产品功能有利润）、per-tenant（支撑 B2B 毛利计算）。工具方面，Portkey / Helicone 作为网关代理提供 per-request 成本追踪，Langfuse / Traceloop 提供开源的 span 级归因，Datadog LLM Observability 提供企业级云成本集成。

**预算防护（不可谈判的底线）。** 编排框架中的最大迭代上限（防止失控循环）、per-trace token 天花板、per-user / per-workflow 速率限制、支出异常告警（偏差超过 2σ 时触发）。agent 陷入推理循环时可能产生天量 thinking tokens——per-request 和 per-session 的 token 上限是防止成本失控的最后一道防线。

### 安全（Safety & Security）

Agent 的安全面比传统 LLM 应用更大，原因在于 agent 具备**自主行动能力**。传统 LLM 应用的风险主要是输出有害内容（toxicity、bias、幻觉），agent 的风险是**执行有害行动**——调用工具删除数据、发送未授权消息、泄露敏感信息。OWASP 在 2026 年专门为 agentic 应用发布了独立的 Top 10 风险清单（区别于已有的 LLM Top 10），反映了这个威胁面的独立性。

**OWASP Top 10 for Agentic Applications（2026）。** 由 100+ 行业专家协作制定。十大风险覆盖了 agent 系统特有的攻击面，按编号：（1）Agent Goal Hijack——通过直接或间接指令注入篡改 agent 目标；（2）Tool Misuse & Exploitation——agent 以不安全的方式使用合法工具（递归调用、危险链式调用、资源耗尽）；（3）Agent Identity & Privilege Abuse——委托权限和模糊的 agent 身份导致越权操作；（4）Agentic Supply Chain Compromise——agent 运行时动态信任的外部工具或 schema 被攻陷；（5）Unexpected Code Execution——agent 生成或触发的代码未经充分验证即执行；（6）Memory & Context Poisoning——agent 记忆或上下文状态被腐蚀，影响后续推理（详见 [topics/memory-security.md](../topics/memory-security.md)）；（7）Insecure Inter-Agent Communication——多 agent 间通信被篡改或伪造；（8）Cascading Agent Failures——小故障通过连接的系统传播造成大范围影响；（9）Human-Agent Trust Exploitation——人类过度信赖 agent 的误导性解释或虚假权威；（10）Rogue Agents——agent 因目标漂移、串通或涌现行为超越预设目标。

**间接 Prompt Injection：生产环境中的真实威胁。** 2025 年的数据显示 73% 的生产 AI 部署遇到过 prompt injection。间接注入（indirect prompt injection）是对 agent 最危险的攻击形态——攻击者不直接和 agent 对话，而是将恶意指令嵌入 agent 会处理的数据源（邮件、文档、网页、数据库记录）。Agent 信任自己的数据管道，把被污染的内容当作合法上下文处理。已有公开案例：EchoLeak（Microsoft 365 Copilot 的零交互漏洞——隐藏在邮件中的注入指令通过 RAG 检索触发，通过图片 URL 请求外泄数据，成功率 80%）、GeminiJack（Google Gemini Enterprise 的同类攻击）。

Simon Willison 的"致命三角"框架概括了 agent 安全的根本困境：当系统同时具备（1）私有数据访问、（2）不可信输入暴露、（3）外发请求能力时，它就是可被利用的。大多数有生产价值的 agent 天然满足这三个条件。

**Reasoning model 特有的安全面。** 模型可能在 thinking trace 中"想偏"——通过内部推理为绕过 guardrail 找到看似合理的理由。当 thinking trace 被隐藏时（OpenAI 模式），这个问题对安全审计更加棘手。这是传统 guardrail 框架没有覆盖的维度。

**Defense-in-depth。** 没有单一防御层能提供完整保护。当前的共识是纵深防御：输入筛查（检测和过滤恶意 prompt）+ 对话控制（限制 agent 的行动范围、工具权限）+ 输出验证（检查 agent 的工具调用和输出是否在预期范围内）。Guardrail 平台（如 Straiker、Lakera、Promptfoo）提供运行时的检查层，在 agent 的每个决策点施加约束。但 2025 年的学术评估发现，面对自适应对手，12 种 prompt injection 防御的被攻破率超过 90%（受控实验条件）。核心困难是模型无法可靠区分指令和数据——OpenAI 承认这是一个"前沿安全挑战"，近期没有根本解决方案。

**EU AI Act（2026-08-02 全面生效）。** 高风险 AI 系统必须设计为人类可以理解输出、干预决策、覆盖结果、并在需要时停止运行。这对 agent 系统的人机协同设计提出了法律层面的硬要求。

### 人机协同与权限（Human-in-the-Loop）

Agent 的自主性和人类的控制之间需要精心设计的平衡。太少的自主性（每步确认）等于人工操作，太多的自主性（完全放手）等于失控。2026 年的生产实践已收敛到一种**风险分层架构**。

**三种 HITL 模式。** 大多数生产系统需要同时支持三种模式：行动前审批（synchronous approval——高风险操作必须等待人类批准后才执行）、不确定时上报（escalation——agent 遇到低置信度场景时主动请求人类介入）、执行后审计（post-execution audit——中低风险操作自动执行但完整记录，供异步审查）。

**风险分层审批架构。** 低风险操作（读取操作、格式转换等）自动执行；中风险操作（低风险的写操作）异步记录供事后审查；高风险操作（发送外部消息、执行金融交易、删除数据、执行 shell 命令）排队等待同步人工审批。分层的依据不只是操作类型，还包括 blast radius（影响范围）和可逆性——删除数据比创建数据风险高，因为不可逆。

**实践案例：Claude Code 的权限模型。** Claude Code 实现了一种具体的分层方案：工具被分为自动允许（Read、Glob、Grep 等读操作）和需要确认（Edit、Write、Bash 等写操作）两类。用户可以通过白名单扩展自动允许的工具集。子 agent 的能力被进一步限制（不能再派子 agent）。这种"默认保守，用户可放宽"的模式在 coding agent 领域被证明是一个成功的权衡——既不阻碍工作流，又防止了最危险的误操作。

**身份治理。** 没有身份治理来定义 agent 可以自主做什么、什么需要审批，HITL 检查点就没有执行机制。组织需要一个身份感知的编排层：暂停 agent 执行、将审批请求路由到授权人类、执行有时限的决策窗口、记录每次干预供审计。

### 可靠性与部署（Reliability & Deployment）

Agent 系统的可靠性面临一个数学上的硬约束：**复合可靠性问题（compound reliability）**。Lusser 定律指出，当独立组件串行执行时，系统整体成功率是各组件成功率的乘积。对 agent 而言，每个步骤（LLM 推理、工具调用、结果解析）都有一个成功概率。如果每步 95% 可靠，20 步后系统可靠性只剩 36%。如果每步 85% 可靠，10 步后只剩 20%。即使每步 99% 可靠，10 步后也只有约 90%。

这解释了为什么提升单个 agent 步骤的质量很难显著改善系统级可靠性——一旦允许错误在步骤之间传播而不加验证边界，风险就会指数级累积。这也解释了为什么实践者倾向于缩短 agent 工作流的步数，放弃开放式的长时间运行任务，转向步骤更少的受限工作流。

**Reasoning model 时代的容量规划。** 传统的容量规划假设响应时间在一个可预测的范围内——比如 P99 延迟在 2 秒以内。Reasoning model 打破了这个假设：同一个模型对简单问题秒级响应，对复杂问题分钟级响应，方差极大。传统的 P99 延迟指标和限流策略需要重新设计。异步 agent 模式在 2026 年越来越流行——系统接受分钟级延迟，通过异步执行和回调机制处理。

**多 agent 系统的额外可靠性成本。** 从单 agent 到多 agent 引入额外的故障模式：状态同步问题（agent 对共享状态的视图不一致）、通信中断（消息重排序、超时歧义、schema 不兼容）、协调开销饱和（每次 agent 间交接增加 100-500ms 延迟）、资源争用（API 速率限制、数据库连接池耗尽）。实测数据显示，单 agent 系统可达 99.5% 成功率，等效的多 agent 实现降至 97%。多 agent 架构的 token 成本通常是单 agent 的 2-5 倍。关于何时引入多 agent 的决策框架和成本权衡的详细分析，详见 [layers/L3-multi-agent.md](L3-multi-agent.md) 的核心决策章节。

**错误恢复的生产模式。** 分层处理：首先让 reasoning model 自身尝试修正（内化的自我纠错能力）；工具执行层面的错误按指数退避重试（有最大重试次数上限）；有替代工具可用时尝试替代路径；能接受部分结果时优雅降级；最后升级到人工干预。核心约束是**必须显式禁止无限重试**——这是最常见的生产故障模式。保留失败上下文（不清除错误信息），让 agent 从失败中学习，比"干净地"重试更有效。

**终止策略。** Agent 循环需要多重终止条件：自然终止（模型不再调用工具）、步数上限、token 上限、成本上限、超时上限、质量检查（外部评估器判断输出是否达标）。只依赖自然终止是危险的——agent 可能永远认为"还需要再做一件事"。

## 工具与框架

**可观测性平台**（按定位排列）：
- **Langfuse**：开源基准线，MIT 许可，框架无关。状态：**成熟**
- **LangSmith**：LangChain 生态的商业平台，LangGraph 原生支持最强。状态：**成熟**
- **Braintrust**：评测优先的 SaaS，适合 eval 开发为主的团队。状态：**成熟**
- **Arize / Phoenix**：ML 可观测性向 LLM 扩展。状态：**成熟**
- **Datadog LLM Observability**：企业级，与云基础设施集成。状态：**上升期**

**评测框架**：
- **DeepEval**：开源 agent 评测框架，支持多种评分方法。状态：**上升期**
- **Galileo**：提供低成本 LLM-as-judge（Luna-2 小模型，$0.01-0.02/M tokens）。状态：**上升期**
- **Promptfoo**：开源红队测试和评测工具，支持 OWASP Top 10 评估。状态：**上升期**

**安全与 Guardrail 平台**：
- **Straiker**：运行时 AI guardrail。状态：**上升期**
- **Lakera**：prompt injection 检测和 guardrail。状态：**上升期**
- **Silmaril**（YC）：自修复的 prompt injection 防御。状态：**早期**

**成本管理**：
- **LiteLLM**：统一 API 代理，内置多厂商路由和成本追踪。状态：**成熟**
- **Portkey / Helicone**：AI 网关，per-request 成本追踪和路由。状态：**上升期**

## 塌缩与涌现

### 塌缩

**传统的延迟基线和容量规划模型。** 假设响应时间在秒级范围内的所有设计失效。P99 延迟指标需要重新定义。固定超时策略让位于自适应超时。

**简单的请求数计费模型。** 当 reasoning model 的 token 消耗方差达几百倍时，"按请求计费"不再有意义。计费和成本归因必须下沉到 token 级别。

**显式 CoT 评估方式。** 当 reasoning trace 被隐藏时，直接评估思考过程不再可行，需要通过最终输出和 reasoning summary 间接推断。

**简单的工具调用成功率监控。** 当简单函数调用的成功率已达 99%+ 时，"调用成功了吗"不再是有区分力的指标。需要转向更细粒度的评估：调用时机是否正确、参数是否最优、是否应该调用了其他工具。

**静态的 guardrail 规则。** 基于关键词匹配或正则表达式的 guardrail 被自适应攻击轻松绕过。

### 涌现

**成本工程作为独立学科。** Token 消耗追踪、per-feature 成本归因、成本上限设置、effort 分布优化、model routing 的成本维度——这些在第二代不存在的工程实践构成了一个完整的新学科。

**推理时计算的可观测性。** Thinking tokens 消耗、effort 级别分布、路由命中率、缓存命中率——这些维度在传统 APM 中不存在，需要专门的工具支持。

**Process eval。** 从"答对了吗"到"思考路径合理吗"——评估推理过程而非只评估结果。

**OWASP Agentic Top 10。** Agent 安全从 LLM 安全中独立出来，形成自己的威胁分类和防御框架。

**风险分层的 HITL 架构。** 从"全部确认"或"全部放手"走向精细的风险分层审批。

**记忆评测与治理。** Agent 记忆的评测（MemoryArena 等）和安全治理（记忆主权框架）作为 L4 的新维度出现。详见 [dimensions/06-memory.md](../dimensions/06-memory.md) 和 [topics/memory-security.md](../topics/memory-security.md)。

**合规基础设施。** EU AI Act 等法规要求 agent 系统满足可解释性、可干预性、可停止性等硬性标准，驱动了新的工程需求。

## 开放工程问题

- **综合性的 agent benchmark 不存在。** 当前的 benchmark 各自衡量能力的不同切面（代码、GUI、工具调用、推理），没有一个 benchmark 覆盖 agent 在生产中面对的综合性挑战。多步骤可靠性、成本效率、错误恢复能力、与人类的协作质量——这些生产关键维度缺乏标准化评测。
- **Reasoning trace 的可见性应该如何标准化？** OpenAI 和 Anthropic 的分歧尚未解决。可见性影响可调试性、安全审计、process eval 的可行性。这个决定的影响远超产品体验。
- **成本优化的最优策略缺乏理论框架。** 当前的 prompt caching + model routing + semantic caching 组合是经验驱动的。什么时候该用哪种策略、组合使用的边际收益如何变化——缺乏系统化分析。
- **间接 prompt injection 没有根本解决方案。** 模型无法可靠区分指令和数据，这是当前架构的根本局限。所有防御都是缓解而非根治。这个问题是否需要模型架构级别的变化？
- **复合可靠性问题如何工程化解决？** 缩短工作流、增加验证边界、用子 agent 隔离风险——当前的做法都是绕路，不是正面解决。是否存在从根本上改变这个数学约束的架构模式？
- **Agent 的 SLA 怎么定义？** 传统 SLA 基于延迟和可用性。Agent 的延迟方差极大、输出质量难以量化、成本不可预测——在这些条件下，什么样的 SLA 定义是有意义的？

## 参考资料

### 评测

- [Agent Evaluation Framework 2026: Metrics, Rubrics & Benchmarks — Galileo](https://galileo.ai/blog/agent-evaluation-framework-metrics-rubrics-benchmarks) — 轨迹评测 vs 结果评测的框架，LLM-as-judge 的系统偏差分析和缓解策略
- [AI Agent Evaluation: A Practical Framework — Braintrust](https://www.braintrust.dev/articles/ai-agent-evaluation-framework) — 实用评测框架
- [Evaluating AI Agents: Real-World Lessons from Amazon — AWS](https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-real-world-lessons-from-building-agentic-systems-at-amazon/) — Amazon 的实战经验
- [Evaluating AI Agents in Practice: Benchmarks, Frameworks, and Lessons Learned — InfoQ](https://www.infoq.com/articles/evaluating-ai-agents-lessons-learned/) — 综合评述
- [State of Agent Engineering — LangChain](https://www.langchain.com/state-of-agent-engineering) — 2026 行业调查，57.3% 生产部署率等核心数据
- [Measuring Agents in Production](https://arxiv.org/pdf/2512.04123) — 学术视角的生产评测

### 可观测性

- [Agent Observability 2026: Evals, Traces, Cost Guide — Digital Applied](https://www.digitalapplied.com/blog/agent-observability-2026-evals-traces-cost-guide) — 综合指南：四类信号、采样策略、漂移检测、OTel 集成
- [7 Best LLM Tracing Tools for Multi-Agent AI Systems (2026) — Braintrust](https://www.braintrust.dev/articles/best-llm-tracing-tools-2026) — 多 agent 追踪工具对比
- [Best LLM Observability Tools for AI Agents — Latitude](https://latitude.so/blog/best-llm-observability-tools-agents-latitude-vs-langfuse-langsmith) — Langfuse vs LangSmith vs Braintrust 对比
- [Top 5 LLM and Agent Observability Tools in 2026 — MLflow](https://mlflow.org/top-5-agent-observability-tools) — 工具横向对比

### 成本工程

- [AI Agent Cost Optimization: Token Economics and FinOps in Production — Zylos Research](https://zylos.ai/research/2026-02-19-ai-agent-cost-optimization-token-economics) — 最全面的成本优化指南：token 经济学、优化技术对比（含节省比例数据）、FinOps 实践
- [Agent Pricing Models 2026: Token vs Outcome Billing — Digital Applied](https://www.digitalapplied.com/blog/agent-pricing-models-token-vs-outcome-based-2026) — 定价模型演进
- [The Hidden Economics of AI Agents — Stevens](https://online.stevens.edu/blog/hidden-economics-ai-agents-token-costs-latency/) — token 成本与延迟的权衡

### 安全

- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) — Agent 专用安全风险框架
- [AI Security in 2026: Prompt Injection, the Lethal Trifecta, and How to Defend — Airia](https://airia.com/ai-security-in-2026-prompt-injection-the-lethal-trifecta-and-how-to-defend/) — 致命三角框架、EchoLeak/GeminiJack 案例分析、纵深防御策略
- [AI Agent Security in 2026: Prompt Injection, Memory Poisoning, and the OWASP Top 10 — Swarm Signal](https://swarmsignal.net/ai-agent-security-2026/) — Agent 安全全景
- [Safety in Building Agents — OpenAI](https://platform.openai.com/docs/guides/agent-builder-safety) — OpenAI 官方安全指南
- [Prompt Injection Defense Checklist for AI Apps (2026) — DomainOptic](https://domainoptic.com/blog/prompt-injection-defense-checklist-2026/) — 实用防御清单

### 人机协同

- [Human-in-the-Loop: A 2026 Guide to AI Oversight — Strata](https://www.strata.io/blog/agentic-identity/practicing-the-human-in-the-loop/) — HITL 模式和身份治理
- [Human-in-the-Loop for AI Agents: Best Practices — Permit.io](https://www.permit.io/blog/human-in-the-loop-for-ai-agents-best-practices-frameworks-use-cases-and-demo) — 实践指南和框架
- [Enforcing Human-in-the-Loop Controls for AI Agents — Prefactor](https://prefactor.tech/learn/enforcing-human-in-the-loop-controls) — 执行层面的控制

### 可靠性

- [Towards a Science of AI Agent Reliability](https://arxiv.org/html/2602.16666v1) — 学术视角的 agent 可靠性框架
- [The Reliability Gap: Agent Benchmarks for Enterprise — Simmering](https://simmering.dev/blog/agent-benchmarks/) — benchmark 到生产的鸿沟分析
- [Multi-Agent System Reliability: Failure Patterns and Validation Strategies — Maxim](https://www.getmaxim.ai/articles/multi-agent-system-reliability-failure-patterns-root-causes-and-production-validation-strategies/) — 多 agent 故障模式和验证策略
- [5 Production Scaling Challenges for Agentic AI in 2026 — MLMastery](https://machinelearningmastery.com/5-production-scaling-challenges-for-agentic-ai-in-2026/) — 生产扩展的五大挑战

## 更新日志

- 2026-05-01：可靠性章节增加 L3 交叉引用（多 agent 决策框架）
- 2026-05-01：初次创建。覆盖评测（结果/轨迹/process eval、LLM-as-judge、benchmark 鸿沟）、可观测性（四类信号、工具格局、reasoning trace 困境、采样策略）、成本工程（成本结构、优化工具箱、FinOps、预算防护）、安全（OWASP Agentic Top 10、间接 prompt injection、致命三角、纵深防御）、人机协同（风险分层、三种 HITL 模式、EU AI Act）、可靠性（复合可靠性、容量规划、多 agent 故障模式、终止策略）
