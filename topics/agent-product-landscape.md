# Agent 产品全景

> 锚点：横跨 gen2 / gen3 / gen4 代际；影响 L1 / L2 / L3 / L4 全层

## 这个概念是什么

这个 topic 记录的是 agent 产品竞争格局的演进——不是技术架构视角，而是"谁在做什么、做出来没有、市场怎么反应"的产品视角。它补充时间线文件（gen2/gen3）缺少的竞争叙事、商业数据和格局判断。

---

## 内部结构：五个阶段

### 阶段一：Demo 驱动的幻觉期（2023 上半年）

AutoGPT、BabyAGI 代表这个阶段的典型形态——给模型接上工具和记忆，让它自己循环调用自己。在 demo 里看起来惊人，在生产里几乎全部失效。失败的根源不是架构设计有问题，而是地基（GPT-3.5 / 早期 GPT-4）的 instruction following 可靠性根本撑不住多步自主执行。

LangChain 成为事实标准框架不是因为它好，而是因为它出现得最早、API 对开发者最友好。这个时期的框架层极度混乱——每周都有新框架。

### 阶段二：垂直产品突破（2023 下半年 - 2024 上半年）

**Cursor**（2023）是这个阶段的标志性产品。它不去解决"完整任务自主执行"这个太难的问题，而是把模型能力的不确定性包裹在 IDE UI 里——diff view、inline accept/reject、context picker。核心设计哲学是：**模型不可完全信任，所以需要 UI 做缓冲**。这让它成为第一个有真实生产价值的 coding agent 产品。

**Devin**（2024-03，Cognition AI）宣称"第一个 AI 软件工程师"，在 SWE-bench 上达到 13.86%——当时是革命性数字。更重要的是它确立了一个新产品定义：不是"帮你写代码的工具"，而是"可以被分配任务的 agent"。初始定价 $500/月，面向企业级客户。

同期 SWE-bench 作为评测标准开始主导行业叙事，几乎每个 coding agent 产品发布都要报 SWE-bench 成绩——这个 benchmark 从学术工具变成了军备竞赛的标尺。

### 阶段三：协议标准化 + Computer Use（2024 下半年）

**Anthropic Computer Use**（2024-10）和 **OpenAI Operator**（2025-01）代表两种截然不同的市场策略：前者 developer-first（API beta，给开发者自己构建）；后者 consumer-first（产品界面，直接面向 ChatGPT Pro 用户）。两者都还不可靠（WebArena 最优约 71.2%），但开辟了"不需要 API，直接操作 UI"这条新路线。

**Salesforce Agentforce**（2024-09 发布，2024-10 GA）标志着传统软件巨头正式入场。代表了和开发者工具完全不同的 agent 形态：嵌入 CRM/ERP 工作流的企业自动化，面向销售、客服、营销等业务角色而非开发者。这是企业轨道的起点。

### 阶段四：推理模型重塑可能性，竞争白热化（2025 年）

这是产品层变化最快的一年。核心驱动力是推理模型的普及——SWE-bench 的成绩曲线陡峭上升，能力边界被快速推远。

**Claude Code**（2025-02 发布，2025-05 GA）代表了与 Cursor 截然不同的哲学转变：**模型已经可靠到不需要 UI 缓冲了**，直接 CLI + shell 访问，6 个月达到 $1B ARR，比 ChatGPT 的收入爬坡速度更快。

**GitHub Copilot Agent Mode**（2025-02）+ **Coding Agent GA**（2025-05），走平台渗透路线——把 agent 能力嵌入已有的 GitHub issue → PR 工作流，不做独立产品。

**Devin 2.0**（2025-04）是一个定价塌缩信号：从 $500/月降到 $20/月，引入 IDE 界面，面向个人开发者。这说明底层能力快速商品化，竞争重心从"能不能做到"转向"怎么定价获客"。

**Codeium → Windsurf 品牌重塑**（2025-04）。Windsurf 的核心是 Cascade——一个能理解整个 codebase、多文件编辑、执行 terminal 命令的 planning agent，设计上比 Cursor 更向"自主执行"倾斜。

**Cognition 收购 Windsurf**（2025-12，约 $250M）。这是 coding agent 领域最大的一次 M&A。收购的逻辑是合并"完全自主执行"（Devin）和"developer-in-the-loop IDE"（Windsurf/Cascade）两种形态——市场验证的结果是两者都有需求，纯自主和纯辅助都不是终态。收购时 Windsurf 已达 $82M ARR、350+ 企业客户。

**框架层大整合**：LangChain 主力迁移到 LangGraph（graph-based 替代 chain-based）；Microsoft 将 AutoGen 并入统一 Agent Framework（AutoGen 进入维护模式）；CrewAI 完成 $18M Series A，达到 60% Fortune 500 渗透。混乱的 2023 收敛为三强格局。

**SWE-bench 军备竞赛**：OpenHands + Claude Sonnet 4.5 达到 72%（2025-11），SWE-bench Verified 开始变得"太容易"——SWE-EVO 作为更难的替代出现，最强模型（GPT-5）在 SWE-EVO 上只能解决 19-21%。

### 阶段五：持久驻留 + 自改进（2026 年，进行中）

**Hermes Agent**（Nous Research，2026-02）代表了一个技术上的范式转移：持久记忆（FTS5 + LLM summarization）+ 从经验自主生成 skill，使 agent 随时间积累能力而非每次重置。这是对"agent 作为 resident 而非工具"这一范式的首次具体产品化。

**Perplexity Computer**（2026-02/03）：从搜索 agent 扩展到通用计算机使用 agent，开始正面竞争 Microsoft 和 Salesforce 的企业市场。

**Cognition 估值 $25B**（2026-04）：Windsurf 集成后 ARR 翻倍，成为 coding agent 赛道迄今最高估值。

---

## 两条平行轨道（尚未收敛）

这是理解产品格局最重要的结构性观察：

**开发者工具轨道**（Cursor → Claude Code → GitHub Copilot Agent → Windsurf+Devin → OpenClaw）：面向工程师，工具入口为 IDE/CLI，评测标准是 SWE-bench，商业模式是 per-seat 订阅。核心争议：UI scaffolding 有多重要？

**企业自动化轨道**（Salesforce Agentforce → Glean → Harvey）：面向业务角色，工具入口为 CRM/企业知识库/专业工作流，评测标准是业务 KPI，商业模式是平台授权 + 用量计费。核心争议：通用 agent 还是垂直专精？

两条轨道的架构、用户心智模型、商业模式完全不同，目前还没有正面竞争。但两者都在向对方地盘延伸：Perplexity Computer 向企业扩张，Agentforce 向开发者工具延伸。

**OpenClaw** 是一个介于两者之间的异类：自托管 gateway，把任意 chat 界面连接到任意 AI backend，247K GitHub stars（截至 2026-03）。它不在任何一条轨道上竞争，而是做基础设施——它的增长反映的是"被集成"的广度而非终端用户数。

---

## 当前状态（截至 2026-05）

**SWE-bench 竞赛数据轨迹**：
- 2024-03：Devin 13.86%（发布时的革命性基准）
- 2025-11：OpenHands 72%（Claude Sonnet 4.5 + extended thinking）
- SWE-EVO：GPT-5 约 19-21%（代替了失去区分度的 SWE-bench Verified）

**商业格局快照**：
- Devin（Cognition）：$73M ARR（2025-06），估值 $25B（2026-04）
- Claude Code：$1B ARR run rate（6 个月内达到，2025 年中）
- Harvey（法律 AI）：估值 $3B，$300M 融资（2025-02）
- Perplexity：估值 $20B（2026-01）

**落地程度标注**：
- 开发者工具轨道（Cursor、Claude Code、GitHub Copilot）：生产部署，付费用户验证
- 企业轨道（Agentforce、Harvey、Glean）：生产部署，企业客户验证
- Computer Use / GUI agent：框架可用，可靠性未达生产级（50-71% 完成率）
- 自改进 agent（Hermes）：框架可用，缺乏生产规模验证

---

## 关键权衡

**Scaffolding 多少合适？** Cursor（重 UI）vs Claude Code（极简 CLI）代表两个极端。Cognition 收购 Windsurf 之后的产品形态暗示：hybrid（自主执行 + 人机交互节点）可能是市场真正要的。

**垂直 vs 水平？** Harvey（法律专精）vs Glean（企业通用知识）代表两种路线。Harvey 在单垂直的深度让它实现了 $3B 估值；Glean 押注水平平台的规模效应。两者目前都没有被证伪。

**开源 vs 闭源？** OpenHands（开源，$18.8M Series A）vs Devin（闭源，$25B 估值）。开源在研究和 benchmark 上保持领先，闭源在商业化上跑得更快。OpenClaw 和 Hermes Agent 代表"开源基础设施"路线——不做终端产品，做生态基础设施。

**Agent 作为工具 vs Agent 作为 resident？** 当前主流产品（包括 Claude Code 和 Devin）都是"被调用"的工具。Hermes Agent 是第一个认真做"持续驻留、积累能力"的产品——这个范式如果成立，对当前所有工具型 agent 的商业模式都是挑战。

---

## 安全事件备注

2025-08 至 11 月间，威胁行为者（GTG-2002）利用 Claude Code 自动化了针对 30+ 组织的网络攻击，承担了 80-90% 的攻击执行工作。这不是 Claude Code 被"破解"，而是"最小 scaffolding + 直接环境访问"这个设计哲学的自然延伸——给模型多少自主权，攻击者就能放大多少倍的攻击能量。这是第三代 agent 产品普遍面临的安全债。

---

## 信息源

- [Cognition Devin 73x ARR Surge | AgentMarketCap](https://agentmarketcap.ai/blog/2026/04/11/cognition-devin-73x-arr-growth-coding-agent-revenue)
- [One Year of OpenHands | OpenHands Blog](https://openhands.dev/blog/one-year-of-openhands-a-journey-of-open-source-ai-development)
- [Windsurf Review 2026 | Taskade](https://www.taskade.com/blog/windsurf-review)
- [The Evolution of Claude Code in 2025 | Medium](https://medium.com/@lmpo/the-evolution-of-claude-code-in-2025-a7355dcb7f70)
- [One Year of MCP | Model Context Protocol Blog](https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/)
- [Salesforce Agentforce 360](https://www.salesforce.com/news/press-releases/2025/10/13/agentic-enterprise-announcement/)
- [Harvey AI $3B valuation | TechCrunch via search](https://petronellatech.com/blog/hermes-agent-ai-guide-2026/)
- [OpenClaw Wikipedia](https://en.wikipedia.org/wiki/OpenClaw)
- [Hermes Agent | Nous Research](https://hermes-agent.nousresearch.com/)

## 更新日志

- 2026-05-03：初次创建，覆盖五阶段产品史和两条轨道结构
