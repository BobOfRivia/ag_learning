# Agent 知识系统学习计划

> 这份文档记录知识体系的**学习推进计划**，与知识内容本身（dimensions/layers/topics）分离。
> 学习产出物 = 知识库被填满到可工作的程度，而不是"我读了多少东西"。

---

## 总体节奏

预计 4-6 周（业余时间每天 1-2 小时），分四个阶段。前三个阶段串行+部分并行，第四阶段永久持续。任何一段中断都不浪费——每阶段都有独立的可交付产物。

| 阶段 | 任务 | 时长 | 产出 |
|------|------|------|------|
| 0 | 自诊断 | 半天 | `phase-0-self-diagnosis.md` 完成填写，得到优先级清单 |
| 1 | 骨架填充 | 1-2 周 | 4 个 layer 文件 + ≥3 个 dimension 文件达到"能教别人"状态 |
| 2 | 生产系统逆向 | 2 周 | layers 中每个关键模式有具体系统作为参照；topics/ 新增 3-5 个 |
| 3 | 项目逼出盲区 | 2-3 周（与阶段 2 可并行） | 1-2 个 side project + open-questions 实质性增长 |
| 4 | 稳态运行 | 永久（每周 1-2h） | 持续的 collapse-emergence 日志、每月体检报告 |

---

## 阶段 0：自诊断

**目的**：在读任何资料之前，建立当前认知的 baseline，识别真实薄弱区域。

**做法**：对 7 个 dimensions 和 4 个 layers，凭记忆写出能讲什么，再自评等级。详见 `phase-0-self-diagnosis.md`。

**完成标准**：自诊断表填完，末尾的"阶段 1 优先级排序"已写出。

---

## 阶段 1：骨架填充

**目的**：让每个 dimension 和 layer 文件达到"能教别人"的标准——合上电脑能给新人讲清楚这一层在解决什么问题、有哪些主流方案、各自的取舍。

**优先级**：先 layers 后 dimensions。dimensions 的深度归因留给 llm-learning 项目；在 agent 视角只回答"这个能力变化对工程的影响是什么"。

**资源池**（按层级组织，每个资源带具体问题去读，不要通读）：

### L1 Agent 内核
- Anthropic "Building effective agents"
- Anthropic "Effective context engineering for AI agents"
- Dex Horthy "12-factor agents"
- ReAct 原论文（重点看抽象，不抠实验）
- Reflexion 原论文（同上）
- Lilian Weng "LLM Powered Autonomous Agents"

### L2 与世界的接口
- MCP 官方文档 + Anthropic 关于 MCP 的 blog
- Claude tool use 文档
- Eugene Yan "Patterns for Building LLM-based Systems"
- Jason Liu RAG 系列
- Letta（前 MemGPT）和 Mem0 设计文档

### L3 多 Agent 协作
- Anthropic "How we built our multi-agent research system"（必读）
- Cognition "Don't build multi-agents"（必读，与上一篇并读，注意对立）
- LangGraph Multi-Agent 文档（看模式分类，不抠 API）
- OpenAI Swarm 源码（已 deprecate 但代码极简，2 小时读完精髓）

### L4 生产化
- Hamel Husain "Your AI Product Needs Evals"
- Hamel "Creating a LLM-as-a-Judge that Drives Business Results"
- Eugene Yan "Evaluating Long-form Generation"
- Braintrust / LangSmith / Phoenix 任选一个看其设计哲学
- Anthropic prompt caching 成本优化文档

**完成标准**：所有 4 个 layer 文件 + 至少 3 个 dimension 文件达到"能教别人"状态。每个文件末尾有更新日志。骨架写入触发的塌缩/涌现事件已进 `logs/collapse-emergence-log.md`。

---

## 阶段 2：生产系统逆向

**目的**：把"抽象模式"接地为"X 系统在 Y 约束下选择了 Z，因为 W"。

**系统选择**（挑 2 个，按工作相关性）：
- 编码 + 通用范式：Claude Code / OpenCode（覆盖 L1+L2+部分 L4）
- 通用 agent：OpenHands / Devin-like 开源实现（覆盖 L3+L4）
- RAG / 知识 agent 方向：Verba / RAGFlow
- Browser / computer use 方向：browser-use

**读法**：每个系统带 5-10 个具体问题去读，不要通读。问题示例：
- context 是怎么组装的？哪些信息每轮都重发，哪些是 cache？
- 工具的 schema 是怎么生成的？模型选错工具时怎么处理？
- 长程任务的中间状态存在哪里？checkpoint 和恢复怎么做？
- 错误是怎么回流给模型的？怎么避免死循环？

**完成标准**：layers 文件中每个关键模式至少有 1 个具体系统作为参照；topics/ 新增 3-5 个（当某概念内部结构远超骨架文件能容纳时）。

---

## 阶段 3：用项目逼出盲区

**目的**：把"觉得自己懂了"打回原形。读 100 篇关于 context poisoning 的文章，不如自己被坑一次。

**项目选择**（基于阶段 0 自诊断的薄弱区，挑 1-2 个）：
- Eval 薄弱 → 给现有 agent 建真实 eval pipeline（不是跑 benchmark，是设计能反映真实失败模式的 case set + LLM-as-judge）
- Multi-agent 薄弱 → 用 LangGraph 或裸写 3-agent 协作系统，故意让它失败，分析失败模式
- Memory 薄弱 → 给 chatbot 加跨 session memory，对比有无 memory 的行为差异
- Context 工程薄弱 → 拿现有 long-running agent 做一次 context 重设计，量化 token 占用和效果变化

**时长**：每个项目控制在 3-5 天。

**完成标准**：每个失败模式都进 `logs/open-questions.md`。修订相关 dimension/layer 文件。

---

## 阶段 4：稳态运行

每周固定 1-2 小时，三件事：

1. **扫信息源**（30-45 分钟）：Anthropic engineering blog、Latent Space、Simon Willison weeknotes、Hamel Husain、Eugene Yan、AI Engineer 会议 talks。值得归档的进 `logs/new-arrivals.md`。
2. **归位 new-arrivals**（15-30 分钟）：处理上周积累的待归位项，更新 dimensions/layers/topics，写 collapse-emergence 日志。
3. **每月体检一次**（1 小时）：触发模式 D，扫整个体系找空白、过时、矛盾。

---

## 与 llm-learning 项目的协调

为了不让 dimension 部分卡壳，建议：阶段 1 推进 layers 时，llm-learning 项目至少把**维度 1（Reasoning）、维度 4（Tool Use）、维度 7（Inference-Time Compute）**推到中等水位。这三个对当下 agent 工程影响最直接。其余 4 个维度可晚 1-2 个月。

---

## 执行原则

- **不追求"读完"**。每个资源带问题去读，问题之外略过。
- **第一遍 70 分**。骨架先粗后细，阶段 2 和 3 会自然带回来修订。
- **归位犹豫先进 new-arrivals.md**，不硬塞。每周归位时一起处理。
- **计划本身要被修订**。每周回顾时勾掉完成项、调整剩下的部分，修订也是学习的一部分。

---

## 进度跟踪

- [ ] 阶段 0：自诊断完成
- [ ] 阶段 1：layers 骨架完成
- [ ] 阶段 1：dimensions 骨架完成（≥3 个）
- [ ] 阶段 2：系统逆向完成（系统 1）
- [ ] 阶段 2：系统逆向完成（系统 2）
- [ ] 阶段 3：side project 完成
- [ ] 阶段 4：稳态节奏建立
