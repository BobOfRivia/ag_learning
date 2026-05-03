# 学习任务清单

> Agent 工程知识体系的**微粒学习地图**。每条任务对应一个值得深入理解的关键知识点——dimensions/ 和 layers/ 中有概括性描述，但真正理解需要自己推导机制和权衡。
>
> - **[ ]** = 待学习
> - **[x]** = 已完成（完成后在对应位置补充细节）
> - **锚点** = 指向现有体系位置
> - 难度：⭐ 基础 / ⭐⭐ 进阶 / ⭐⭐⭐ 需要推导

---

## 第 0 层：LLM 能力维度

### 维度 1：Reasoning

- [ ] **⭐⭐ RLVR 训练机制**：可验证奖励的具体形式（数学题精确匹配、代码测试通过率、形式化证明验证）、训练过程如何让模型自发学会长 CoT 和回溯、与 RLHF 的信号差异
  - 目标：能设计一个 RLVR 训练实验的奖励函数，解释为什么可验证域是 reasoning 训练的天然起点
  - 锚点：`dimensions/01-reasoning.md` 内化式 reasoning

- [ ] **⭐⭐⭐ Process Reward Model (PRM) vs Outcome Reward Model (ORM)**：PRM 对每个推理步骤评分 vs ORM 只评估最终答案、PRM 标注成本和 Math-Shepherd 自动化方案、credit assignment 问题
  - 目标：用具体例子说明 ORM 如何强化"运气正确"的错误推理链，以及 PRM 的标注瓶颈如何被自动化缓解
  - 锚点：`dimensions/01-reasoning.md` → 与 `dimensions/07-inference-time-compute.md` beam search 关联

- [ ] **⭐⭐ Thinking Token 的训练来源**：reasoning model 的 thinking tokens 在训练中如何产生？是 SFT 阶段注入思维链数据，还是 RL 阶段自发涌现？DeepSeek-R1-Zero（纯 RL）vs R1（先蒸馏再 RL）的路径差异
  - 目标：区分"训练出来的推理"和"涌现出来的推理"，判断哪种更可靠
  - 锚点：`dimensions/01-reasoning.md` 内化式 reasoning → 姊妹项目 `llm-learning` 训练轨道

- [ ] **⭐⭐ 忠实 CoT vs 不忠实 CoT**：模型展示的思考过程是否真的反映内部计算？Anthropic 的证据（Lanham et al., 2023）、Turpin et al. 的 biased CoT 实验、对安全性和可解释性的影响
  - 目标：能评估 "thinking trace 可见 = 推理透明" 这个假设在多大程度上成立
  - 锚点：`dimensions/01-reasoning.md` 关键不确定性

- [ ] **⭐⭐ CoT 对 Reasoning Model 的干扰机制**：为什么对 reasoning model 写 "step by step" 反而有害？模型内部推理节奏被外部 prompt 干扰的具体表现
  - 目标：给出实际的 prompt 设计建议——什么时候该用 CoT prompt，什么时候应该移除
  - 锚点：`dimensions/01-reasoning.md` → `layers/L1-agent-core.md` 塌缩

### 维度 2：Long Context

- [ ] **⭐⭐⭐ Lost in the Middle 的注意力机制根源**：RoPE 的距离衰减效应如何导致中部 token 获得更少的注意力权重、位置编码和 attention score 的数学关系
  - 目标：从 RoPE 的旋转矩阵出发，推导为什么序列中部的 token 比首尾更容易被忽略
  - 锚点：`dimensions/02-long-context.md` Lost in the Middle → 姊妹项目 `llm-learning` RoPE 章节

- [ ] **⭐⭐ 标称容量 vs 有效容量的量化方法**：NIAH（Needle-in-a-Haystack）测试的局限、RULER 的多维评测设计（多跳追踪、聚合）、NoLiMa 去除表面匹配后暴露的真实水位
  - 目标：能解释为什么 NIAH 不够用，设计一个更接近真实场景的 context 评测方案
  - 锚点：`dimensions/02-long-context.md` 有效容量

- [ ] **⭐⭐ Context Rot 的退化曲线**：性能在约 30K token 后加速退化的实证数据、退化的原因（注意力稀释 vs 位置编码漂移 vs 工作记忆瓶颈）、这个拐点是否可以通过训练推后
  - 目标：量化理解 context 长度与性能的关系曲线，用于指导生产中的 context budget 分配
  - 锚点：`dimensions/02-long-context.md` 关键不确定性

- [ ] **⭐⭐⭐ Test-Time Training (TTT)**：在推理时对模型做轻量微调以适应长 context 的机制、如何实现与 context 长度无关的常数推理延迟、TTT-E2E 在 2M context 上 35 倍加速的来源
  - 目标：理解 TTT 为什么能突破 transformer 的二次方注意力瓶颈，以及它离生产部署还有多远
  - 锚点：`dimensions/02-long-context.md` 核心技术

- [ ] **⭐⭐ RAG 残余价值的五因素决策框架**：语料规模、相关比例、延迟 SLO、更新频率、查询量——每个因素的阈值和权衡逻辑
  - 目标：面对一个具体的检索场景，能用这个框架做出 RAG vs 长 context 的定量决策
  - 锚点：`dimensions/02-long-context.md` 第二层影响 → `layers/L2-world-interface.md` RAG 模式

### 维度 3：Multimodality

- [ ] **⭐⭐ 适配器式 vs 原生多模态架构**：适配器架构的信息瓶颈（连接器的表达能力上限）、原生架构的训练成本代价、两者在跨模态推理上的能力差异
  - 目标：解释为什么适配器架构在纯视觉理解上不一定弱于原生架构，但在跨模态推理上有天花板
  - 锚点：`dimensions/03-multimodality.md` 关键区分

- [ ] **⭐⭐ Computer Use 三种感知架构的工程权衡**：截图视觉（15,000+ tokens/动作）vs 可访问性树（token 低 50-100 倍但覆盖有限）vs DOM 操纵（最精确但仅限 web），以及混合架构的去重策略
  - 目标：能为一个具体的 GUI 自动化场景选择最优感知架构组合
  - 锚点：`dimensions/03-multimodality.md` Computer Use → `dimensions/04-tool-use.md`

- [ ] **⭐⭐ Omni 模型的 Thinker-Talker 架构**：Qwen3.5-Omni 的思考-说话分离设计、Hybrid-Attention MoE 如何跨模态进行理解和推理、实时语音交互的并行化机制
  - 目标：理解"推理和生成可以并行"对实时交互的架构意义
  - 锚点：`dimensions/03-multimodality.md` Omni 模型

- [ ] **⭐ 视觉 Token 对 Context 的消耗模型**：一张截图 15,000+ tokens 的计算来源、分辨率和 patch size 对 token 数的影响、视觉 agent 的 context budget 管理策略
  - 目标：给出视觉 agent 的 context 消耗估算公式，用于指导截图频率和分辨率的权衡
  - 锚点：`dimensions/03-multimodality.md` 第一层影响

### 维度 4：Tool Use

- [ ] **⭐⭐ 函数调用的训练方法演进**：ToolFormer 的加权交叉熵筛选机制（只保留让模型后续预测更准的调用样本）、Llama 系列的代码抽取+清洗方法、ToolACE 的自演进合成管线
  - 目标：理解为什么"代码预训练是 tool use 的基础"，以及合成数据如何扩展工具调用的训练多样性
  - 锚点：`dimensions/04-tool-use.md` 训练侧

- [ ] **⭐⭐⭐ 受限解码（Constrained Decoding）**：在 token 生成时动态约束概率分布以保证 JSON schema 合规的算法、Strict Mode 100% 合规的实现原理、对生成质量的影响
  - 目标：从 token 生成层面解释为什么 strict mode 是精确保证而非概率性改善
  - 锚点：`dimensions/04-tool-use.md` 推理侧 → `dimensions/05-instruction-following.md` 结构化输出

- [ ] **⭐⭐ Three Ws 框架**：Whether（是否需要调工具）/ Which（选哪个）/ How（用什么参数），为什么 Whether 的 false positive 比选错工具更危险、Tool Call Hallucination 的检测方法
  - 目标：为一个 agent 系统设计 tool call hallucination 的检测和防护机制
  - 锚点：`dimensions/04-tool-use.md` 基础机制

- [ ] **⭐⭐ LLMCompiler 并行调用优化**：运行时自动识别独立工具调用并融合同类操作的机制、4 倍并行度提升 + 40% token 消耗降低的来源
  - 目标：理解编译器思想如何应用于工具调用调度优化
  - 锚点：`dimensions/04-tool-use.md` 并行函数调用 → arXiv 2312.04511

- [ ] **⭐⭐ MCP 协议架构**：Server 暴露的三种原语（tools / resources / prompts）、传输层设计（stdio 本地进程 → Streamable HTTP 远程服务）、有状态 session 与负载均衡的冲突
  - 目标：能设计一个 MCP server 并理解远程部署的工程挑战
  - 锚点：`dimensions/04-tool-use.md` MCP → `layers/L2-world-interface.md` MCP

- [ ] **⭐ MCP vs A2A 的互补关系**：纵向（agent→tool）vs 横向（agent→agent）的标准化分野、Agent Card 的加密签名和域验证、Task 作为协作基本单位的设计
  - 目标：画出一个完整的 agent 互操作协议栈，说明 MCP 和 A2A 各自覆盖哪一层
  - 锚点：`dimensions/04-tool-use.md` → `layers/L2-world-interface.md` A2A → `layers/L3-multi-agent.md` 协议层

### 维度 5：Instruction Following

- [ ] **⭐⭐ Instruction Gap 的量化**：13 个模型的四种违规类型分布（内容范围 / 格式 / 语气 / 流程）、为什么 reasoning model 不一定更好（DeepSeek-R1 1,061 违规 vs GPT-5 660 违规）
  - 目标：解释 reasoning model "想偏"导致偏离指令的机制，以及为什么"更聪明"不等于"更听话"
  - 锚点：`dimensions/05-instruction-following.md` The Instruction Gap

- [ ] **⭐⭐ 认知惯性（Cognitive Inertia）**：SFT 阶段形成的惯例模式如何干扰自定义指令的遵循、Inverse IFEval 的测试方法、IFEval++ 中细微 prompt 修改导致最高 61.8% 性能下降的机制
  - 目标：理解为什么模型在被要求违背自己的训练惯例时表现骤降，以及这对 agent system prompt 设计的影响
  - 锚点：`dimensions/05-instruction-following.md` 认知惯性

- [ ] **⭐⭐⭐ 指令层级的三层优先级**：System Message > User Message > Third-Party Content 的训练方法、ManyIH-Bench 12 层冲突场景下约 40% 准确率的根因分析
  - 目标：评估指令层级作为 prompt injection 防御的可靠性上限，以及为什么"模型能理解优先级但不能稳定执行"
  - 锚点：`dimensions/05-instruction-following.md` 指令层级 → `layers/L4-production.md` 安全

- [ ] **⭐⭐ 后训练模块化栈**：SFT → 偏好优化（DPO/SimPO/KTO/ORPO）→ 强化学习（GRPO/DAPO/RLVR）三阶段各自解决什么问题、DPO vs PPO 的等价性条件和偏离场景
  - 目标：能解释为什么 DPO 在大多数场景下足够好，以及什么场景仍然需要 PPO
  - 锚点：`dimensions/05-instruction-following.md` 训练方法 → 姊妹项目 `llm-learning` 对齐轨道

- [ ] **⭐⭐ 合成自博弈改变数据经济学**：SPIN（区分自身输出和人类文本）、SPICE（文档基础上自博弈防幻觉）、RLAIF 匹配或超过 RLHF 的条件
  - 目标：理解合成数据在后训练中的边界——什么时候可以替代人工标注，什么时候不行
  - 锚点：`dimensions/05-instruction-following.md` 合成自博弈

### 维度 6：Memory

- [ ] **⭐⭐ 四类记忆的工程映射**：工作记忆（context window）、情景记忆（带时间戳的事件）、语义记忆（抽象知识）、程序记忆（可复用的行为模式）——每种对应什么存储策略和检索模式
  - 目标：为一个具体的 agent 设计记忆架构，说明四种记忆类型分别用什么技术实现
  - 锚点：`dimensions/06-memory.md` 记忆类型

- [ ] **⭐⭐⭐ MemGPT/Letta 的 OS 式三层架构**：core memory（RAM，始终在 context）/ recall memory（对话历史，可搜索）/ archival memory（磁盘，长期存储）的换入换出机制、agent 通过 function call 自主管理记忆的设计
  - 目标：理解"agent 主动管理自己的记忆"和"系统被动注入记忆"的本质区别，以及各自的失败模式
  - 锚点：`dimensions/06-memory.md` → `layers/L2-world-interface.md` Memory 工程 → `topics/memory-architecture.md`

- [ ] **⭐⭐ 参数记忆 vs 非参数记忆**：ROME/MEMIT 的知识编辑精确性问题（改一个事实可能破坏相关知识）、非参数路线的检索瓶颈（存什么都行但能不能找到对的东西）
  - 目标：判断参数记忆路线在什么条件下可能变得可行，目前的技术瓶颈是什么
  - 锚点：`dimensions/06-memory.md` 表示基底

- [ ] **⭐⭐ 图记忆 vs 向量记忆**：向量记忆的"无结构相似性匹配"局限（问"Alice 的经理是谁"可能找不到答案）、图记忆通过实体-关系显式建模支持多跳和时序推理、Graphiti 在 Neo4j 上的实现
  - 目标：给出一个决策框架——什么场景用向量记忆，什么场景必须用图记忆
  - 锚点：`dimensions/06-memory.md` 图记忆 → `layers/L2-world-interface.md` 向量数据库

- [ ] **⭐⭐ 被动回忆 vs 主动运用的鸿沟**：MemoryArena benchmark 显示从 LoCoMo 80%+ 骤降至 40-60% 的数据、"能想起来"和"在对的时刻用对的记忆"的差距根源
  - 目标：分析这个鸿沟需要 reasoning 和 memory 的哪种融合才能弥合
  - 锚点：`dimensions/06-memory.md` 评测层

- [ ] **⭐⭐ 记忆安全的六阶段威胁模型**：Write（注入有毒记忆）→ Store → Retrieve（操纵检索结果）→ Execute → Share（跨 agent 传播污染）→ Forget、MemoryGraft 和 InjecMEM 攻击的具体机制
  - 目标：评估当前记忆框架（Mem0、Letta、Zep）的安全缺口，理解"没有生产系统实现综合记忆安全"意味着什么风险
  - 锚点：`dimensions/06-memory.md` 记忆安全 → `topics/memory-security.md`

### 维度 7：Inference-Time Compute

- [ ] **⭐⭐⭐ 训练时-推理时计算的兑换比率**：Epoch AI 分析——一般性方法每增加 1-2 OOM 推理计算换 1 OOM 训练节省、可验证问题 5-6 OOM 换 3-4 OOM、组合使用时边际收益递减的原因
  - 目标：能为一个具体场景（如数学推理 vs 通用对话）计算训练时和推理时计算的最优分配
  - 锚点：`dimensions/07-inference-time-compute.md` 两种 scaling 的博弈

- [ ] **⭐⭐⭐ T² Scaling Law**：同时优化模型大小、训练数据量和推理时采样次数的统一框架、为什么纳入推理成本后最优策略偏向过度训练（overtrain）
  - 目标：理解 T² 对传统 Chinchilla scaling law 的修正方向和幅度
  - 锚点：`dimensions/07-inference-time-compute.md` 两种 scaling 的博弈

- [ ] **⭐⭐ 两大策略族的边界条件**：搜索与采样（Best-of-N / Beam Search / MCTS）vs 内化式推理（thinking tokens），各自的适用条件、"The Art of Scaling Test-Time Compute" 的核心结论——没有单一策略在所有场景下占优
  - 目标：给出一个任务类型 × 计算预算的策略选择矩阵
  - 锚点：`dimensions/07-inference-time-compute.md` 两大策略族

- [ ] **⭐⭐ Effort 参数的跨厂商对比**：Anthropic effort（low/medium/high/max/xhigh）vs OpenAI reasoning_effort vs Google thinkingBudget 的设计差异、Claude Opus medium ≈ Sonnet high 但输出 token 减少 76% 的含义
  - 目标：建立 effort 参数的心智模型——它不只是成本旋钮，也是模型选型的替代维度
  - 锚点：`dimensions/07-inference-time-compute.md` Reasoning Effort 控制

- [ ] **⭐⭐ LLM 推理成本悖论**：每 token 价格每年降 10 倍（Epoch AI 数据），但 reasoning model 的每请求总 token 消耗同步增长，导致单次请求总成本不一定下降
  - 目标：能用数据论证这个悖论在多长时间内会被解决（或者不会被解决）
  - 锚点：`dimensions/07-inference-time-compute.md` 成本悖论

- [ ] **⭐⭐⭐ ARES 自适应 Effort 路由**：训练轻量路由器在 agent 每一步动态决定 effort 级别的 RL 方法、将 high effort 使用比例从 50%+ 降到 20% 以下的机制
  - 目标：理解为什么"大量步骤被过度分配了推理算力"，以及自动路由的延迟和复杂度代价
  - 锚点：`dimensions/07-inference-time-compute.md` 两层路由

---

## 第一层：Agent 内核

### Agent Loop

- [ ] **⭐ 三种循环模式的架构差异**：经典 ReAct（Thought→Action→Observation 紧密交错）vs 双相位（先集中想再集中做）vs 最小 loop（while tool_call → execute → append），它们之间的关系（不是替代而是共存）
  - 目标：用伪代码写出三种循环的核心实现，解释各自的设计权衡
  - 锚点：`layers/L1-agent-core.md` Agent Loop

- [ ] **⭐⭐ Agent 循环终止策略**：自然终止（模型不再调用工具）的不可靠性、步数/token/成本/超时多重上限的设计、失控循环（最常见的生产故障模式）的检测
  - 目标：为一个生产 agent 设计完整的终止条件集合，包含优先级排序
  - 锚点：`layers/L1-agent-core.md` 开放问题 → `layers/L4-production.md` 终止策略

- [ ] **⭐⭐ 错误保留策略**：为什么不清除失败的工具调用和错误信息反而更好、模型如何从 context 中的失败信息隐式更新内部信念、这个策略与 context 空间节约的张力
  - 目标：设计一个"选择性错误保留"策略——哪些错误值得保留，哪些可以安全移除
  - 锚点：`layers/L1-agent-core.md` Context Engineering

### Context Engineering

- [ ] **⭐⭐⭐ KV-Cache 命中率优化**：为什么 KV-cache 命中率是 agent 系统最重要的单一性能指标（缓存 vs 未缓存 10 倍成本差）、保持 system prompt 前缀稳定、只追加不修改、JSON 序列化确定性
  - 目标：列出所有可能导致 KV-cache 失效的场景，并给出对应的防护措施
  - 锚点：`layers/L1-agent-core.md` KV-cache 优化 → `topics/context-engineering.md`

- [ ] **⭐⭐ Context Engineering 四操作框架**：Write（持久化）/ Select（检索）/ Compress（压缩）/ Isolate（子 agent 隔离）——每个操作的具体技术手段和适用时机
  - 目标：对一个长时间运行的 agent 任务，设计一套完整的 context engineering 方案
  - 锚点：`topics/context-engineering.md`

- [ ] **⭐⭐ 压缩策略对比**：锚定式迭代摘要（4.04/5 准确度）vs 观察遮蔽（52% 成本减少）vs Provider 原生压缩（Anthropic compact / Google ADK compaction）——各自的信息损失特征
  - 目标：理解"摘要漂移"（多次压缩后罕见但关键细节丢失）的机制，以及如何最小化信息损失
  - 锚点：`topics/context-engineering.md` 压缩策略

- [ ] **⭐⭐ Thinking Tokens 与 Context 空间的零和竞争**：128K 窗口中 thinking 占 60K 导致实际可用仅 68K、context engineering 和 reasoning budget 管理的协同设计
  - 目标：建立 thinking tokens / context 内容 / 工具结果的三方预算分配模型
  - 锚点：`dimensions/02-long-context.md` 第一层影响 → `dimensions/07-inference-time-compute.md`

- [ ] **⭐ 文件系统作为外部记忆**：把信息卸载到文件系统只在 context 中保留引用路径的策略、CLAUDE.md 和 todo.md 的设计模式、"压缩必须可逆"原则
  - 目标：设计一个文件系统卸载方案，确保 agent 能高效找回卸载的信息
  - 锚点：`layers/L1-agent-core.md` Context Engineering → `topics/context-engineering.md`

### 编排模式

- [ ] **⭐⭐ Anthropic 的六种可组合构建块**：Prompt Chaining / Routing / Parallelization / Orchestrator-Workers / Evaluator-Optimizer / 自主 Agent——各自的适用条件、在 reasoning model 时代的价值变化
  - 目标：面对一个具体任务，能选择最合适的编排模式组合并论证选择
  - 锚点：`layers/L1-agent-core.md` 编排模式谱系

- [ ] **⭐⭐ Workflow vs Agent 的核心区分**：workflow（预定义代码路径）用可预测性换灵活性 vs agent（LLM 动态控制）用灵活性换可预测性、何时选择哪种
  - 目标：给出一个决策树——基于任务的确定性程度和容错要求选择 workflow 或 agent
  - 锚点：`layers/L1-agent-core.md` 编排模式

- [ ] **⭐⭐ Plan-then-Execute vs ReAct 的混合实践**：Plan 提供结构 + ReAct 提供适应性的组合、Claude Code 的 TodoWrite 机制、3.6 倍速度优势的条件
  - 目标：理解什么任务适合 plan-first，什么任务适合 react-first，以及混合的最优比例
  - 锚点：`layers/L1-agent-core.md` Plan-then-Execute

---

## 第二层：与世界的接口

### RAG 工程

- [ ] **⭐⭐ RAG 四层成熟度模型**：Naive（40% 检索失败率）→ Advanced（混合检索 91% recall@10）→ Modular（可插拔模块）→ Agentic（agent 自主决策检索策略），各层的工程投入和收益曲线
  - 目标：评估一个现有 RAG 系统处于哪个层级，以及升级到下一层级的 ROI
  - 锚点：`layers/L2-world-interface.md` RAG 模式

- [ ] **⭐⭐ 混合检索（BM25 + 向量 + RRF 融合）**：BM25 的关键词匹配机制、向量搜索的语义匹配机制、Reciprocal Rank Fusion 的融合排序算法、为什么混合方案一致优于单一方法（91% vs 78% vs 65%）
  - 目标：从算法层面理解 RRF 如何组合两种不同信号，以及阈值调优的方法
  - 锚点：`layers/L2-world-interface.md` RAG 关键生产模式

- [ ] **⭐⭐ 交叉编码器重排序**：初始 top-20/50 → 精排的架构、交叉编码器比双编码器更精确的原因（token 级交互 vs 独立编码后匹配）、延迟和精度的权衡
  - 目标：理解为什么重排序是第二高投入产出比的 RAG 改进
  - 锚点：`layers/L2-world-interface.md` RAG 关键生产模式

- [ ] **⭐⭐ 语义分块策略**：按文档结构切分 vs 固定 token 切分 vs 递归字符切分的对比、Chunk 大小对检索精度和生成质量的影响曲线
  - 目标：能为不同类型的文档（代码、法律文件、技术文档）选择最优分块策略
  - 锚点：`layers/L2-world-interface.md` RAG

- [ ] **⭐ RAG 失败归因**：73% 的 RAG 失败在检索环节而非生成环节的数据、检索失败的常见原因（查询-文档词汇不匹配、分块边界切断关键信息、embedding 模型语义盲区）
  - 目标：建立 RAG 系统的诊断流程——先查检索质量再查生成质量
  - 锚点：`layers/L2-world-interface.md` RAG 模式

### 向量数据库与 Embedding

- [ ] **⭐⭐ 向量数据库商品化趋势**：pgvector 在 50M 向量以下性价比碾压专用数据库（471 QPS vs Qdrant 41 QPS）、Pinecone 收入从 $26.6M 降到 $14M 的前哨信号、"向量从数据库品类变成数据类型"
  - 目标：判断什么场景仍需要专用向量数据库，什么场景 pgvector 足够
  - 锚点：`layers/L2-world-interface.md` 向量数据库

- [ ] **⭐⭐ GraphRAG 架构**：知识图谱如何解决基于块检索无法处理的多跳问题、Neo4j HNSW 向量索引 + Cypher 图遍历的组合、索引成本比纯向量 RAG 高 10-40 倍的代价
  - 目标：给出 GraphRAG 的适用条件——什么类型的数据和查询模式值得 10-40 倍的索引成本
  - 锚点：`layers/L2-world-interface.md` 图+向量融合

- [ ] **⭐ Embedding 模型选型**：闭源（OpenAI text-embedding-3-large / Cohere embed-v4.0 / Gemini Embedding 2）vs 开源（Qwen3-Embedding-8B / BGE-M3）的成本和质量权衡、月处理量超过 1 亿 token 时自托管的经济性
  - 目标：建立 embedding 模型选型的决策矩阵（语言覆盖、维度、成本、部署方式）
  - 锚点：`layers/L2-world-interface.md` Embedding 模型格局

### Memory 工程

- [ ] **⭐⭐ 被动注入 vs 主动管理两种范式**：系统在外部检索注入 context（ChatGPT Memory）vs agent 自己调用 memory function 决定读写（Letta），各自的失败模式和适用场景
  - 目标：为一个新的 agent 项目选择记忆集成范式，论证选择
  - 锚点：`layers/L2-world-interface.md` Memory 工程 → `dimensions/06-memory.md`

- [ ] **⭐⭐ Mem0 的选择性提取机制**：用 LLM 判断对话中什么值得记住（而非记录一切）的实现、准确率-延迟-token 消耗的三角权衡（72.9%/17s/26K vs 68.4%/2.59s/1.8K）
  - 目标：理解"4.5% 准确率差距换 6.5 倍延迟改善和 14.4 倍 token 节省"在什么条件下值得
  - 锚点：`layers/L2-world-interface.md` Memory 工程 → `dimensions/06-memory.md` 检索阶段

---

## 第三层：多 Agent 协作

- [ ] **⭐⭐ 单 Agent vs 多 Agent 决策框架**：任务结构（是否存在真正独立分支）作为决定性变量、token 经济学硬约束（多 agent 15 倍 token 消耗 × reasoning model 3-8 倍单价 = 45-120 倍总成本）、Microsoft Azure 从专家集群回退到通才 agent 的案例
  - 目标：面对一个具体任务，能用这个决策框架做出是否采用多 agent 的定量判断
  - 锚点：`layers/L3-multi-agent.md` 核心决策

- [ ] **⭐⭐ 五种编排模式**：Sequential / Concurrent（扇出-汇聚）/ Group Chat / Handoff / Magentic（动态自适应），各自的控制流特征、适用条件、在 reasoning model 时代的价值变化
  - 目标：能画出每种模式的控制流图，并标注各自的优劣势和适用场景
  - 锚点：`layers/L3-multi-agent.md` 编排模式

- [ ] **⭐⭐ 编排 vs 编舞（Orchestration vs Choreography）**：中央控制点 vs 去中心化事件驱动的架构差异、为什么当前生产实践一边倒选择编排（agent 行为已经足够不确定）
  - 目标：分析什么条件下编舞模式可能变得有价值（提示：事件驱动架构、跨组织协作）
  - 锚点：`layers/L3-multi-agent.md` 编排 vs 编舞

- [ ] **⭐⭐ 第三代动机转变**：从"用多 agent 提升质量"到三个新动机——并行化加速、context 隔离、成本混合编排——的转变逻辑
  - 目标：用具体例子说明每个新动机在什么场景下驱动多 agent 设计
  - 锚点：`layers/L3-multi-agent.md` 第三代动机转变

- [ ] **⭐⭐ Context 隔离作为多 agent 核心价值**：长 reasoning trace 导致单 agent context 膨胀的定量分析、子 agent 不能递归派发的约束逻辑、"用 agent 隔离管理 context"策略
  - 目标：计算在什么 context 消耗率下，拆分子 agent 的隔离收益超过额外 token 成本
  - 锚点：`layers/L3-multi-agent.md` → `layers/L1-agent-core.md` Sub-agent

---

## 第四层：生产化

### 评测（Evals）

- [ ] **⭐⭐ 结果评测 vs 轨迹评测 vs 过程评测**：三者各自衡量什么、为什么 reasoning model 时代轨迹评测尤其关键（lucky hit 问题、合理推理但工具调用失败）、process eval 的可行性困境（trace 可见性差异）
  - 目标：为一个生产 agent 设计三层评测架构（unit evals → LLM-as-judge → 生产 trace 采样）
  - 锚点：`layers/L4-production.md` 评测

- [ ] **⭐⭐ LLM-as-judge 的三种系统性偏差**：位置偏差、长度偏差、迎合偏差的具体表现、复杂任务错误率超 50% 的数据、缓解策略（随机化顺序 + 多次一致性检验 + 显式评分标准）
  - 目标：设计一个减轻偏差的 LLM-as-judge 流程，目标 Spearman 相关系数 > 0.80
  - 锚点：`layers/L4-production.md` LLM-as-judge

- [ ] **⭐⭐ Benchmark 到生产的 37% 鸿沟**：实验室得分和真实部署表现差距的来源、单次 60% 成功率但八次重复后降至 25% 的可靠性衰减
  - 目标：解释这个鸿沟的根源（分布差异、边界情况、复合错误），以及如何在上线前检测
  - 锚点：`layers/L4-production.md` benchmark 鸿沟

### 可观测性

- [ ] **⭐⭐ Agent 专用的四类追踪信号**：LLM 调用 / 工具调用 / 检索步骤 / 规划决策——各自需要记录什么字段、标准 APM 漏掉的四种 agent 故障模式
  - 目标：为一个新的 agent 系统设计可观测性方案，确保四种故障模式都能被检测
  - 锚点：`layers/L4-production.md` 可观测性

- [ ] **⭐ OpenTelemetry GenAI 语义约定**：`gen_ai.system`、`gen_ai.usage.input_tokens`、`gen_ai.response.finish_reasons` 等标准化字段、如何实现供应商无关的 instrumentation
  - 目标：用 OTel 约定为一个 agent 实现可移植的追踪，能切换后端（Langfuse → LangSmith）只需配置变更
  - 锚点：`layers/L4-production.md` OTel

- [ ] **⭐⭐ 尾部采样与漂移检测**：尾部采样优于头部采样的原因、100% 保留错误/高成本/低评分 trace 的策略、Golden-set replay 漂移检测方法
  - 目标：设计一个采样策略，在控制存储成本的同时最大化故障发现率
  - 锚点：`layers/L4-production.md` 采样策略

### 成本工程

- [ ] **⭐⭐ Agent 成本结构分析**：多轮循环的二次增长（10 轮 ReAct 消耗 50 倍 token）、输出 token 价格溢价（reasoning model 达 8:1）、context window 扩展的注意力矩阵成本（128K 约 64 倍于 8K）
  - 目标：能为一个 agent 系统做成本预估，识别成本增长的主要驱动力
  - 锚点：`layers/L4-production.md` 成本工程

- [ ] **⭐⭐ Prompt Caching 实践**：缓存 token 成本约为非缓存的 10%、延迟降低 75-85%、保持前缀稳定的具体做法（不放时间戳、只追加不修改、序列化确定性）
  - 目标：在一个真实的 agent 系统中实现 prompt caching，测量成本和延迟的改善
  - 锚点：`layers/L4-production.md` Prompt Caching → `topics/context-engineering.md` KV-Cache

- [ ] **⭐⭐ Model Routing 的 87% 成本节省**：90% 的查询可由小模型处理的假设验证、路由准确率对总成本的影响曲线、手工规则 vs 失败信号升级 vs 自动路由三种实现路线
  - 目标：设计一个渐进式 model routing 方案——从手工规则起步，逐步过渡到自动路由
  - 锚点：`layers/L4-production.md` Model Routing → `dimensions/07-inference-time-compute.md` 两层路由

- [ ] **⭐⭐ Semantic Caching**：向量相似度识别语义等价查询（31% 查询存在语义相似性）、缓存投毒风险、相似度阈值调优的方法
  - 目标：理解 semantic caching 和 prompt caching 的互补关系，以及各自的适用边界
  - 锚点：`layers/L4-production.md` Semantic Caching

- [ ] **⭐⭐ FinOps 成本归因**：per-user / per-task / per-tenant 三维归因、"根节点打标签向子节点传播"的实践、Portkey / Helicone / Langfuse 的工具对比
  - 目标：为一个多租户 agent 系统设计成本归因架构
  - 锚点：`layers/L4-production.md` FinOps

### 安全

- [ ] **⭐⭐⭐ 间接 Prompt Injection 攻击机制**：攻击者在 agent 数据源（邮件/文档/网页）中嵌入恶意指令、EchoLeak（Microsoft 365 Copilot 零交互漏洞，80% 成功率）和 GeminiJack 的具体技术路径
  - 目标：能复现一个间接 prompt injection 的 PoC，理解为什么"模型无法可靠区分指令和数据"是根本难点
  - 锚点：`layers/L4-production.md` 安全 → `dimensions/05-instruction-following.md` 指令层级

- [ ] **⭐⭐ Simon Willison 的"致命三角"**：（1）私有数据访问 +（2）不可信输入暴露 +（3）外发请求能力 = 可被利用、为什么大多数有生产价值的 agent 天然满足这三个条件
  - 目标：评估一个真实 agent 系统是否落入致命三角，以及如何在不放弃功能的前提下缓解
  - 锚点：`layers/L4-production.md` 安全

- [ ] **⭐⭐ OWASP Top 10 for Agentic Applications**：十大风险各自的攻击面和缓解策略、与传统 LLM Top 10 的区别（agent 特有：自主行动能力带来的额外风险）
  - 目标：能对一个 agent 系统做安全审计，识别它面临的 top 3 风险并设计缓解措施
  - 锚点：`layers/L4-production.md` OWASP Agentic Top 10

- [ ] **⭐⭐ 纵深防御架构**：输入筛查 + 对话控制 + 输出验证的三层防御、12 种 prompt injection 防御被攻破率超 90% 的学术评估数据、guardrail 平台（Straiker / Lakera / Promptfoo）的定位
  - 目标：理解为什么"所有防御都是缓解而非根治"，以及在这个限制下的最优防御组合
  - 锚点：`layers/L4-production.md` Defense-in-depth

### 人机协同与可靠性

- [ ] **⭐⭐ 风险分层审批架构**：低/中/高风险操作的分层标准（blast radius × 可逆性）、三种 HITL 模式（行动前审批 / 不确定时上报 / 执行后审计）的组合
  - 目标：为一个企业 agent 系统设计完整的风险分层和审批流程
  - 锚点：`layers/L4-production.md` 人机协同

- [ ] **⭐⭐⭐ 复合可靠性问题（Lusser 定律）**：每步 95% → 20 步 = 36%、每步 99% → 10 步 = 90% 的数学推导、为什么"提升单步质量很难显著改善系统级可靠性"、缩短工作流和增加验证边界的权衡
  - 目标：计算一个具体 agent workflow 的系统可靠性，识别提升可靠性的最有效杠杆点
  - 锚点：`layers/L4-production.md` 复合可靠性

- [ ] **⭐⭐ Reasoning Model 时代的容量规划**：响应时间方差极大（秒级到分钟级）对 P99 延迟指标的冲击、异步 agent 模式的架构设计、rate limit 策略的重新设计
  - 目标：为一个 reasoning model 驱动的 agent 设计容量规划方案
  - 锚点：`layers/L4-production.md` 容量规划

---

## 跨层概念

### 塌缩-涌现机制

- [ ] **⭐⭐ 塌缩-涌现的因果链推导**：从一个具体的地基变化（如 reasoning 内化）出发，推导它在四层上引发的完整塌缩-涌现链条、区分直接影响和间接传导
  - 目标：能对任何一个新的地基变化（如 memory 内化），预测它在四层上的工程影响
  - 锚点：`overview.md` → `logs/collapse-emergence-log.md`

- [ ] **⭐⭐ 代际转换的判定标准**：什么指标满足了才算进入下一代？gen3→gen4 的三个判定条件（API 层原生持久记忆、外部向量库使用下降、memory eval 广泛采用）为什么目前都还没有满足
  - 目标：建立一个"代际判定仪表盘"——持续追踪的 3-5 个关键指标
  - 锚点：`timeline/gen4-memory-era.md` → `dimensions/06-memory.md`

### Agent 框架选型

- [ ] **⭐⭐ "弱模型 + 好编排 > 强模型 + 差编排"**：2026 年 2 月测试三个框架使用相同模型差距 17 分的数据、编排质量如何被量化和比较
  - 目标：建立 agent 框架评估方法论——不只看模型标签，还看编排架构对任务表现的贡献
  - 锚点：`layers/L1-agent-core.md` 框架 → `layers/L3-multi-agent.md` 框架选型

- [ ] **⭐ 框架疲劳与"多薄的框架"趋势**：从"选哪个框架"到"直接使用 MCP server + 最少编排层"的趋势、Gartner 警告 40%+ 项目因成本升级取消、agent sprawl 问题
  - 目标：评估什么时候用框架，什么时候 bare-metal 实现更优
  - 锚点：`layers/L2-world-interface.md` 框架疲劳

### EU AI Act 与合规

- [ ] **⭐ EU AI Act 对 Agent 系统的硬性要求**：人类可理解输出、干预决策、覆盖结果、停止运行——这四个要求如何映射到 agent 的工程设计
  - 目标：检查一个现有 agent 系统是否满足 EU AI Act 的四个要求，识别合规缺口
  - 锚点：`layers/L4-production.md` EU AI Act
