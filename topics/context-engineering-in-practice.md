# Context Engineering 实践：以 Claude Code 为活体案例

> 锚点：L1 Agent Core / [topics/context-engineering.md](context-engineering.md)（理论框架的实践补充）

## 这个概念是什么

[context-engineering.md](context-engineering.md) 覆盖了 context engineering 的学科全景——四种操作、压缩策略对比、KV-cache 原理。这个文件从一个不同的角度切入：**把一个正在运行的生产 agent 拆开，看它的 context 管理在一次真实工作会话中具体如何运作。**

案例主体是 Claude Code——不是指底层的 Claude 模型，而是围绕模型构建的完整 agent 系统：包含工具调用循环、子 agent 调度、文件系统交互、记忆持久化、会话压缩等机制的端到端 agent。以下分析基于一次跨越多个 session、涉及 26 个文件创建和编辑的知识体系构建会话（即本知识体系的构建过程本身）。

## Agent 的 Context 架构分层

一个生产 agent 的 context window 不是一个均匀的平面空间，而是由多个生命周期不同的信息层叠加而成。以 Claude Code 为样本，可以拆解出四个层次，每层有不同的加载时机、持久性和压缩策略：

**系统层（System Layer）。** 包含 system prompt 和工具定义（tools schema）。这一层在会话开始时加载，整个会话期间不变。它是 KV-cache 命中率最高的部分——因为前缀完全稳定。工具定义中有一个值得注意的优化：**deferred tools**（延迟工具）。Claude Code 的可用工具列表中，部分工具只暴露名称而不加载完整 schema，需要时通过 ToolSearch 动态拉取。这避免了数十个工具的完整 JSON schema 始终占据 context——在 MCP 生态下可用工具达到数百个时，这个优化的价值是决定性的。

**持久记忆层（Persistent Memory Layer）。** 跨 session 存活的信息。在 Claude Code 中有两个载体：CLAUDE.md（项目级，始终加载，编码了协作协议、文件结构、操作约束——本质上是程序记忆）和 auto-memory 文件（用户级，~/.claude/projects/.../memory/，存储用户偏好和反馈——本质上是语义记忆和情景记忆的混合）。这一层的特殊性在于它横跨 session 边界：上一个会话中用户的纠正（"调研资料必须写入文件"）被持久化为 memory 文件，下一个会话自动加载并影响行为。这是 context engineering 中 Write 操作的跨 session 体现。

**会话层（Session Layer）。** 当前会话的对话历史和工具调用结果的累积。这是 context 消耗的主战场——每一轮对话、每一次工具调用的输入输出、每一段 thinking trace 都在这里追加。在一次典型的知识体系构建会话中，context 在处理 3-4 个文件后（约 30-50 轮工具调用）就会接近上限。这也是压缩策略发挥作用的主要层次。

**工作记忆层（Working Memory Layer）。** 当前轮次的 tool results，生命周期最短。一个 Read 工具返回的 200 行文件内容在这一轮被使用后，到下一轮就已经只作为历史消息存在，可以被压缩或丢弃。关键洞察：**工具输出是 context 中信息密度最低的部分**——大量原始文本中只有少数几个关键数据点会影响后续决策。这正是为什么 context-engineering.md 中提到的压缩比指南建议工具输出用 10:1 到 20:1 的压缩率。

## 四种操作的实践映射

以下按 [context-engineering.md](context-engineering.md) 的 Write / Select / Compress / Isolate 四操作框架，逐一映射到本会话中可观察的具体行为。

### Write（卸载到外部）

**CLAUDE.md 作为程序记忆的显式持久化。** 这个文件编码了四种协作模式、文件模板、操作约束（"修改文件前先告诉我"、"归档时更新 log"）等规则。它不是每次会话推导出来的，而是一次写入、反复读取。这解决了一个根本问题：agent 的行为一致性不能依赖于每次都在 context 中重新理解协作规则——规则必须被外化为可稳定加载的记忆。

**auto-memory 作为反馈的跨 session 持久化。** 当用户在某次会话中纠正行为（"调研资料带标题写入对应文件"），这个反馈被存为 memory 文件。下次会话加载 MEMORY.md 索引时，这条反馈自动进入 context。和 CLAUDE.md 的区别是：CLAUDE.md 是预设的、结构化的程序记忆；auto-memory 是后天习得的、零散的经验记忆。两者共同构成了 agent 的跨 session 行为稳定性。

**collapse-emergence-log 作为 append-only 事件日志。** 这个日志只记录增量事件（"什么变了"），不记录全量状态（"现在什么样"）。这本身就是一种 Write 操作的压缩形式——增量日志比状态快照节省得多。回溯时通过事件序列重建状态，而非直接读取状态快照。

**文件间交叉引用代替内容复制。** 当 dim05 讨论指令层级与 prompt injection 的关系时，不复制 L4 安全章节的内容，而是写一个链接"详见 [layers/L4-production.md]"。这是 Write 操作的一个变体——**用指针代替拷贝**。在 context 管理中，这意味着 agent 不需要同时加载两个文件的完整内容来理解它们的关系，只需加载一个文件加上另一个文件的路径。

### Select（按需加载）

**Grep → Read 的两阶段检索。** 这是本会话中最频繁使用的 context 管理模式。第一阶段用 Grep 做粗筛——搜索关键词，返回匹配的文件列表和行号（信息量极小，通常几十 byte）。第二阶段用 Read + offset/limit 做精取——只读返回行号附近的具体内容。例如，检查 dim04 是否提到了 instruction following 时：先 grep "instruction.follow" 得到 126 行匹配，然后 Read offset=120 limit=15 只加载那 15 行。相比直接 Read 整个 192 行的文件，context 消耗减少约 92%。

这个模式本质上是**索引-内容分离**：grep 结果是索引（文件名+行号），Read 结果是内容。索引极其廉价，内容按需加载。这和数据库的 B-tree 索引 → 回表查询是同构的。

**offset + limit 的选择性读取。** 这不只是"少读一点"。它反映了一个关于信息分布的判断：**文件中的信息对当前任务的相关性不是均匀分布的**。读 L4-production.md 时，如果当前任务是检查安全章节的交叉引用，只需读 70-94 行（安全章节），不需要读 1-69 行（评测和可观测性）或 95-224 行（人机协同和可靠性）。这要求 agent 对文件结构有先验知识——它知道安全章节大概在哪个位置。这个先验来自之前的全文阅读或文件模板的结构规律。

**Deferred tools 的惰性加载。** Claude Code 的可用工具列表中，部分工具（如 CronCreate、NotebookEdit、WebFetch 等）只暴露名称。完整的参数 schema 在需要调用时才通过 ToolSearch 拉取。这解决了一个 MCP 时代的实际问题：当可用工具从十几个增长到上百个时，所有工具的完整 schema 同时驻留 context 是不可接受的。惰性加载将"工具发现"和"工具使用"解耦——context 中始终只有工具名列表（几百 byte），具体 schema 在确认要使用时才支付 context 成本。

### Compress（压缩）

**会话 compaction 的结构化摘要。** 当 context 接近上限时，系统生成一份结构化的会话摘要。本会话的 compaction 摘要包含 9 个固定分类：

1. Primary Request and Intent（用户要做什么）
2. Key Technical Concepts（涉及的关键技术概念）
3. Files and Code Sections（已修改的文件，含具体行数和内容描述）
4. Errors and Fixes（遇到的错误和修复）
5. Problem Solving（解决问题的方法）
6. All User Messages（用户的所有原始消息——逐条保留）
7. Pending Tasks（待完成的任务）
8. Current Work（当前工作状态）
9. Optional Next Step（建议的下一步）

这正是 [context-engineering.md](context-engineering.md) 中描述的**锚定式迭代摘要**的实例。固定的分类结构迫使压缩器显式处理每个维度——不会出现"自由文本摘要静默丢失文件路径"的问题，因为第 3 类（Files and Code Sections）强制要求列出所有修改过的文件。

**摘要中保留什么、丢弃什么。** 观察 compaction 摘要的实际内容可以提炼出一个模式：

保留的信息具有以下特征：（1）恢复后继续工作所必需——待完成任务的精确状态、当前工作的位置；（2）不可从外部恢复——用户的决策和偏好（"继续05吧"）、讨论中达成的共识；（3）精确标识符——文件路径、行号、具体数据点（"GPT-5: 660 violations 88%"）。

丢弃的信息具有以下特征：（1）可从外部重新获取——文件的完整内容（可以再 Read）、web 搜索结果（可以再搜）；（2）中间推理过程——为什么选择某个结构、如何组织某段内容的思考过程；（3）已完成任务的执行细节——dim02 创建时的具体 web 搜索查询和原始结果。

**这揭示了一个压缩的核心原则：保留"做了什么决策"和"还要做什么"，丢弃"怎么执行的"和"看到了什么原始数据"。** 决策是不可重建的（除非重新和用户讨论），执行细节是可重建的（再读一次文件、再搜一次网页）。

### Isolate（隔离）

**子 agent 并行调度的 context 经济学。** 本会话中最典型的隔离操作是 L2 补充章节时并行派出的 3 个研究 agent（RAG 模式、Agent 框架、向量数据库）。每个 agent 独立做 web 搜索和信息整合，各自消耗约 23,000-33,000 token（从 task notification 的 total_tokens 可以看到）。但返回给主 context 的结果摘要各约 500 字（~700 token）。

这意味着：3 个 agent 总共消耗约 82,000 token 做研究，但主 context 只接收约 2,100 token 的压缩结果。**有效压缩比约 39:1。** 如果这些研究在主 context 中串行执行，82,000 token 的累积会直接导致 context 溢出。隔离不只是节省空间——它使某些任务**在物理上成为可能**。

隔离还有一个常被忽视的价值：**防止 context 交叉污染**。RAG 研究中的大量 benchmark 数据不会干扰 agent 框架研究的推理，反之亦然。如果在同一个 context 中串行执行，早期研究的细节可能会影响后续研究的注意力分配——这就是 context-engineering.md 中提到的 "context distraction"。

**隔离的代价。** 子 agent 无法访问主 context 的完整历史。它不知道之前的 session 中发生了什么，不知道用户的偏好，不知道已有文件的具体内容。这就是为什么派出子 agent 时需要写一份自包含的 prompt 简报——简报本身就是一次信息压缩，把主 context 中与当前子任务相关的部分提炼成几句话。简报写得太简略，子 agent 缺少必要上下文；写得太详细，失去了隔离的 token 节省优势。这个权衡没有通用解法，取决于子任务的独立性。

## 理论之外的实践技术

以下技术在 [context-engineering.md](context-engineering.md) 的四操作框架中没有显式覆盖，但在实际 agent 运行中对 context 效率有显著影响。

### 并行工具调用的 context 效率

当多个工具调用之间没有依赖关系时，在同一轮中并行发出。本会话中频繁出现的模式是"同时读 4 个文件"或"同时编辑 5 个文件"。这不只是执行速度的优化——它也是 context 效率的优化。

串行模式：Request₁ → Result₁ → Request₂ → Result₂ → Request₃ → Result₃（6 条消息追加到 context）
并行模式：[Request₁, Request₂, Request₃] → [Result₁, Result₂, Result₃]（2 条消息追加到 context）

在 context 中，每条消息除了内容本身，还有角色标记、消息边界等结构性开销。并行调用将 N 次独立操作的结构性开销从 2N 降到 2。当 N=5 时，这意味着 10 条消息压缩为 2 条——结构性开销减少 80%。

### 任务完成即剪枝信号

当一个文件（如 dim05）创建完成并经用户确认后，该文件的详细规划信息（结构方案、研究数据、措辞选择的理由）就不再需要保留在 context 中。只有两类信息仍然有价值：（1）文件已完成这个事实（影响后续的体检和计划）；（2）文件中的交叉引用关系（影响后续文件的一致性检查）。

这是一种**任务感知的压缩**——压缩策略不是机械地按时间或位置决定，而是根据信息是否仍然与未完成的任务相关来决定。compaction 摘要中，已完成文件只保留文件名和关键数据点，不保留创建过程。

### Git 状态快照作为 point-in-time checkpoint

会话开始时，系统自动注入一份 git status 快照（当前分支、最近提交、工作区状态）。这是一种预计算的 context 注入——避免 agent 在会话过程中反复运行 git status 来确认当前状态。快照是只读的（"this status is a snapshot in time, and will not update during the conversation"），agent 知道它可能过时，会在需要精确信息时重新查询。

这反映了 Select 操作的一个变体：**前置加载 + 过时声明**。预先加载高频需要的信息以节省后续查询，同时显式标注其时效性以防止 agent 依赖过时数据。

### 结构化模板的注意力引导效应

CLAUDE.md 中定义了维度文件、层级文件、topic 文件的统一模板。这些模板不只是内容组织工具——它们还有 context engineering 的功能。当 agent 需要检查一个文件的"耦合"章节时，它知道这个章节在模板中的位置（通常在文件后半部分），可以直接用 offset 跳到那里。**模板将文件结构变成了可预测的，使 Select 操作的精度大幅提高。**

反过来说，如果文件没有统一模板，agent 每次都需要先全文扫描以理解结构，才能定位到需要的部分——这本身就是一笔 context 开销。模板是对未来 Select 操作的预投资。

## 失败模式与权衡

### Compaction 的信息损耗

本会话经历了一次 compaction。从摘要中可以观察到几类信息损耗：

**工具输出的完整内容丢失。** 摘要中标注了"Note: .../05-instruction-following.md was read before the last conversation was summarized, but the contents are too large to include"。这意味着 compaction 后，agent 不再拥有这些文件的完整内容——需要重新 Read。对于已完成的文件这不是问题（不需要再读），但对于正在进行的工作（dim05 的后续步骤）需要额外的工具调用来恢复上下文。

**中间推理过程丢失。** 为什么 dim05 的结构方案采用了"横切维度"的定位？为什么选择了特定的 benchmark 数据？这些决策的推理过程在 compaction 中被丢弃。如果用户事后质疑某个决策，agent 无法回溯当时的思考——只能重新推理。

**这揭示了 compaction 的根本局限：它保留了"做了什么"但丢失了"为什么这样做"。** 对于连续执行的工作流（本会话的"按顺序完成"模式）这足够了，但对于需要回溯和修正的工作流可能不够。

### 子 agent 结果的压缩损失

三个研究 agent 各返回约 500 字的摘要。以向量数据库 agent 为例，它执行了 8 次工具调用（web 搜索和页面获取），处理了约 23,000 token 的信息，最终压缩为一份结构化报告。报告中保留了关键数据点（Pinecone $14M 收入、pgvector 471 QPS vs Qdrant 41 QPS），但丢失了数据来源的原始上下文——例如 pgvector 的 benchmark 在什么硬件配置下运行、Pinecone 收入下降的具体时间线。

如果后续发现某个数据点可疑，无法回到子 agent 的原始 context 去追溯——子 agent 的完整 transcript 保存在临时文件中，但读取它会溢出主 context（JSONL 格式的完整对话记录）。这是隔离的不可逆性：**一旦子 agent 完成并返回摘要，详细信息就只能通过重新执行来恢复。**

### 跨 Session 的连续性挑战

本会话是从上一个 session 延续的。连续性依赖两个机制：compaction 摘要（包含前 session 的完整状态快照）和持久文件（CLAUDE.md、auto-memory、已提交的文件）。但这两个机制覆盖的信息类型不同：

摘要覆盖了工作状态（"dim05 已创建但后续步骤未完成"），但不覆盖隐性知识（agent 在前 session 中逐步建立的对体系内部关系的理解）。auto-memory 覆盖了用户偏好（"参考资料必须写入文件"），但不覆盖项目特定的工作模式（"用户的模式是：创建文件→检查关联文档→提交推送→开始下一个"）。

结果是：session 恢复后，agent 需要一段"重新热身"的过程——重新读文件、重新建立对结构的理解。这个成本是不可避免的，但可以通过更好的 compaction 摘要设计来降低（例如，在摘要中显式保留"工作模式"部分）。

## 对 context-engineering.md 理论的验证与补充

### 验证的理论预测

**锚定式迭代摘要的优势得到验证。** Compaction 摘要的固定 9 分类结构确实防止了信息静默丢失——文件路径、待完成任务、用户原始消息都被显式保留。这与 Factory.ai 的评测结论一致。

**隔离的经济性得到验证。** 39:1 的有效压缩比证明了子 agent 隔离在 token 经济学上的价值。更关键的是，隔离使某些任务从"context 不够用"变成"可以完成"——这是质变而非量变。

**简单方案常常优于复杂方案得到验证。** 两阶段检索（Grep → Read）是极其简单的模式，但它的 context 效率（减少约 92% 的无关内容加载）超过了大多数花哨的检索优化。

### 理论中未充分覆盖的实践洞察

**模板作为 Select 操作的预投资。** 统一模板使文件结构可预测，让 offset-based 的精准读取成为可能。这在理论框架中没有显式讨论，但在实践中是 Select 效率的重要前提。

**Write 操作的指针-拷贝权衡。** 交叉引用链接（指针）vs 内容复制（拷贝）是 Write 操作的一个重要设计决策。指针节省存储但需要额外的 Select 操作来解引用；拷贝增加冗余但减少 Select 需求。当前体系选择了指针模式，这在文件数量可控（26 个）时是正确的——解引用成本低于冗余维护成本。

**并行工具调用作为 context 结构性开销的优化。** 这在理论框架中被归入"执行效率"而非"context engineering"，但它对 context 增长速度的影响是实质性的。

## 信息源

- 本文件的分析基于本知识体系构建过程中的真实会话观察，不依赖外部参考
- 理论框架来自 [topics/context-engineering.md](context-engineering.md) 及其引用的信息源
- Claude Code 的架构描述基于 [Anthropic 官方文档](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview) 和 [Martin Fowler 的分析文章](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

## 更新日志

- 2026-05-01：初次创建。基于知识体系构建会话的第一人称观察，覆盖 context 架构分层、四操作实践映射、理论外的实践技术、失败模式分析、理论验证与补充
