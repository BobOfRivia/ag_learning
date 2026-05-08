# 塌缩与涌现日志

> 这是体系最重要的活体文件。每次地基有变化或生态发生标准化事件，都在这里追加一条记录。
> 长期看，这个日志本身就是 LLM 演进对 Agent 工程影响的「活的索引」。

## 模板

```
## YYYY-MM-DD｜[简述事件]
- **触发**：[什么进展，地基哪个维度变化 / 或标注"生态事件"表示非能力驱动]
- **塌缩**：[哪些工程模式因此变得不再必要]
- **涌现**：[哪些新工程问题因此出现]
- **影响层级**：L1 / L2 / L3 / L4
- **置信度**：高 / 中 / 低
- **关联文件**：[更新了哪些 dimensions/layers 文件]
```

---

## 已记录条目（倒序，最新在前）

## 2026-05-08｜维度 6 Memory 补充"与 Context Engineering 的关系"章节

- **触发**：体系建设——L2 自诊断中暴露 memory 与 context engineering 概念混淆，dimensions/06-memory.md 已有"与 Long Context 的区分"但缺"与 Context Engineering 的区分"
- **塌缩**：（不适用）
- **涌现**：（不适用，但记录了一个**演化中的争议**）——长上下文 + 持久 KV cache 是否会让 memory engineering 的部分工作塌缩回 context engineering，这直接关系 gen4 假设（memory 内化）的边界。当前无结论，置信度中。
- **影响层级**：（元层 + L2 概念边界澄清）
- **置信度**：（不适用）
- **关联文件**：dimensions/06-memory.md

## 2026-05-04｜L1 编排模式谱系补充：控制权归属框架与选型信号

- **触发**：体系建设——补充 workflow vs agent 选型的「信任边界」视角和实践选型信号
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：layers/L1-agent-core.md

## 2026-05-03｜Agent 产品全景归档：框架整合、定价塌缩、企业轨道成型

- **触发**：生态事件（多个并发）——框架层大整合（LangChain→LangGraph、AutoGen→Microsoft Agent Framework）、coding agent 定价塌缩（Devin $500→$20）、企业轨道独立成型（Agentforce/Harvey/Glean）、M&A 波（Cognition 收购 Windsurf $250M）
- **塌缩**：
  - L3：LangChain 原始 chain-based 架构退出主流，被 LangGraph（graph-based）取代；AutoGen 并入 Microsoft 框架进入维护模式
  - L1："agent 框架提供过度抽象"的模式被证伪——轻量原语（OpenAI Agents SDK 等）验证为更可靠路径
  - 产品层：纯自主执行（Devin 1.x）和纯辅助（早期 Cursor 模式）之间的非此即彼，被混合形态（Windsurf/Cascade，后被 Cognition 整合）取代
- **涌现**：
  - L4：SWE-bench 从学术基准变成行业军备竞赛标尺，驱动了一轮产品优先级扭曲（"让 benchmark 好看"优先于"解决真实用户场景"）；SWE-EVO 的出现是修正信号
  - L3：企业自动化轨道（Agentforce/Glean/Harvey）与开发者工具轨道（Cursor/Claude Code/Devin）形成两条独立格局，彼此暂时不直接竞争但都在向对方地盘延伸
  - L4：pricing 塌缩（Devin $500→$20）使 coding agent 能力的商品化趋势明确，竞争护城河从"能力"向"用户体验 + 分发渠道"转移
- **影响层级**：L1 + L3 + L4
- **置信度**：高（M&A 和定价数据已确认）；中（两轨道的长期分化或收敛尚不确定）
- **关联文件**：topics/agent-product-landscape.md, timeline/gen2-tool-rag-era.md, timeline/gen3-reasoning-era.md

## 2026-05-01｜Context Engineering 实践案例 topic 创建

- **触发**：体系建设——将 Claude Code 作为活体 agent 案例，从第一人称视角分析 context engineering 的实际运作
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：topics/context-engineering-in-practice.md

## 2026-05-01｜L2 四个待补充章节完成

- **触发**：体系建设——L2 骨架文件中 RAG 模式、Memory 工程、Agent 框架、向量数据库四个章节从创建以来一直标记为待补充
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：layers/L2-world-interface.md

## 2026-05-01｜维度 5 Instruction Following 文件创建

- **触发**：体系建设——维度 5 文件缺失，横切维度的定位、instruction gap、指令层级、后训练方法演进需要系统梳理
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：dimensions/05-instruction-following.md

## 2026-05-01｜维度 3 Multimodality 文件创建

- **触发**：体系建设——维度 3 文件缺失，多模态能力全景、Computer Use 成熟度、Omni 模型架构需要系统梳理
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：dimensions/03-multimodality.md

## 2026-05-01｜维度 2 Long Context 文件创建

- **触发**：体系建设——维度 2 文件缺失，长上下文的技术格局、标称 vs 有效容量鸿沟、对 RAG 的塌缩影响需要系统梳理
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：dimensions/02-long-context.md

## 2026-05-01｜L3 多 Agent 协作层级文件创建

- **触发**：体系建设——L3 层级文件缺失，多 agent 协作的工程模式、决策框架和动机转变需要系统梳理
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：layers/L3-multi-agent.md

## 2026-05-01｜L4 生产化层级文件创建

- **触发**：体系建设——L4 层级文件缺失，但其内容被多个维度和时间线文件大量引用
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：layers/L4-production.md

## 2025 ~ 2026-初｜推理时计算成为独立 scaling 维度，effort 控制接口标准化

- **触发**：维度 7 Inference-Time Compute。学术侧（DeepMind TTS 论文、T² Scaling Laws）确立理论基础，产品侧三大厂商相继推出 effort / reasoning_effort / thinkingBudget 控制参数
- **塌缩**：
  - L1："一律用最强模型"的默认策略不再经济可行
  - L4：传统的固定延迟基线和按请求数计费的成本模型失效
  - L3：多 agent 投票提升质量的动机消失（reasoning model 内部已做类似搜索）
- **涌现**：
  - L1：两层路由决策（选模型 × 选 effort 级别）成为 agent 内核的标准设计问题
  - L1：逐步骤 reasoning budget 管理（ARES 等自适应框架）
  - L4：成本工程作为独立学科出现——token 消耗追踪、per-feature 成本归因、成本上限设置
  - L4：推理时计算的可观测性需求（thinking tokens 消耗、effort 分布、缓存命中率）
  - L3：成本混合编排（高 effort orchestrator + low effort worker）成为多 agent 新动机
- **影响层级**：L1 + L3 + L4
- **置信度**：高
- **关联文件**：dimensions/07-inference-time-compute.md, dimensions/01-reasoning.md

## 2024-末 ~ 2025｜MCP 发布与爆发，工具生态标准化

- **触发**：生态事件。Anthropic 发布 MCP，随后 OpenAI、Google、Microsoft 等全面采纳，2025-12 捐赠给 Linux Foundation / AAIF
- **塌缩**：
  - L2：ChatGPT Plugins 及封闭工具市场模式退役
  - L2：每框架一套的工具适配格式（LangChain Tools、AutoGPT Plugins 等各自为政的局面）
- **涌现**：
  - L2：MCP 生态管理——工具的标准化发现、连接、质量评估、权限控制
  - L2：工具描述工程——如何写出让模型准确理解的工具描述
  - L3：A2A 协议（2025-04 Google 发布）开辟 agent-to-agent 标准化通信
  - L4：第三方 MCP server 的信任与安全模型
- **影响层级**：L2 + L3 + L4
- **置信度**：高
- **关联文件**：dimensions/04-tool-use.md, layers/L2-world-interface.md

## 2023-06｜OpenAI 推出原生 function calling API

- **触发**：维度 4 Tool Use，从 prompt hack 转向 API 原生支持
- **塌缩**：
  - L1：手工解析模型输出提取工具调用的 prompt 技巧
  - L2：基于字符串匹配的工具调用路由
- **涌现**：
  - L1：tool use 成为 agent 内核的原生能力，ReAct 模式工程化
  - L2：结构化工具集成成为标准工程实践
  - L2：并行函数调用、工具选择优化等进阶问题
- **影响层级**：L1 + L2
- **置信度**：高
- **关联文件**：dimensions/04-tool-use.md

## 2024-末｜Anthropic 推出 Computer Use beta，工具形态扩展

- **触发**：维度 4 Tool Use × 维度 3 Multimodality 交��
- **塌缩**：
  - L2：部分需要 API 集成的场景可以直接通过 GUI 操作替代（尤其是没有 API 的遗留系统）
- **涌现**：
  - L2：Computer Use 基础设施——截图管道、坐标映射、操作确认、失败恢复
  - L4：GUI agent 的权限模型和安全边界
  - L4：Computer Use 的评测体系（WebArena、OSWorld 等 benchmark）
- **影响层级**：L2 + L4
- **置信度**：中（Computer Use 的长期影响尚不确定）
- **关联文件**：dimensions/04-tool-use.md, layers/L2-world-interface.md

## 2024-09｜OpenAI o1 发布，reasoning 内化分水岭

- **触发**：维度 1 Reasoning，从 prompt 外挂转向训练内化
- **塌缩**：
  - L1：CoT prompting、Self-Consistency、Reflexion 等大幅贬值
  - L1：经典 ReAct 紧密交错被「思考相位 + 行动相位」取代
  - L3：用多 Agent 模拟「团队思考」提升质量这一动机消失
- **涌现**：
  - L1：Reasoning budget 管理
  - L1：Model routing（按难度路由）
  - L4：成本结构剧变（单次 20-50 倍）
  - L4：Process eval（评估思考过程合理性）
  - L4：Reasoning trace 可观测性
- **影响层级**：L1 + L3 + L4
- **置信度**：高
- **关联文件**：dimensions/01-reasoning.md, timeline/gen3-reasoning-era.md

## 2026-04-27｜补充代际演进时间线（gen1-gen3）

- **触发**：体系建设——时间索引（索引 B）的三个已确立代际文件缺失
- **塌缩**：（不适用）
- **涌现**：（不适用）
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：timeline/gen1-prompt-era.md, timeline/gen2-tool-rag-era.md, timeline/gen3-reasoning-era.md

## 2026-04-26｜知识体系初始化

- **触发**：体系本身的建立
- **塌缩**：（不适用）
- **涌现**：双索引结构、四种协作模式、本日志机制
- **影响层级**：（元层）
- **置信度**：（不适用）
- **关联文件**：所有
