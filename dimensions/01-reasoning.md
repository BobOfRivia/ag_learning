# 维度 1：Reasoning（推理能力）

## 一句话定义

模型在生成答案前进行「思考」的能力——具体表现为：是否会做长链中间推理、是否会自我纠错回溯、是否会在难题上「想得更久」。

## 核心概念与技术

### 外挂式 reasoning（第一代/第二代技术，仍有使用场景）

- **Chain-of-Thought (CoT)**：通过 prompt 引导模型逐步推理。核心区分：zero-shot CoT（"Let's think step by step"）vs few-shot CoT（给示例推理链）。对 reasoning model 已大幅贬值，但对非 reasoning model 仍有效。
- **Self-Consistency**：同一问题多次采样，对最终答案投票取多数。用推理路径的多样性换准确率。在 reasoning model 时代边际收益骤降。
- **Tree-of-Thoughts (ToT)**：将推理建模为搜索树，每个节点是一个思考步骤，可以回溯和分支。工程复杂度高，已被 reasoning model 的内部搜索能力大幅替代。
- **ReAct**：reasoning 和 acting 交错进行——想一步，做一步，观察结果，再想。是早期 agent 的核心范式。现在被「思考相位 + 行动相位」模式取代。
- **Reflexion / Self-Refine**：外部反思循环——让模型审视自己的输出并改进。reasoning model 已内化了大部分这类能力。

### 内化式 reasoning（第三代核心技术）

- **RLVR（Reinforcement Learning from Verifiable Rewards）**：用可验证结果（数学题答案、代码测试通过等）作为奖励信号，训练模型自发学会长 CoT、回溯、验证。是当前 reasoning model 的核心训练范式。
- **Thinking tokens / Extended thinking**：模型在输出最终答案前，先产生一段内部思考过程。这段思考可能对用户可见（如 DeepSeek-R1）或部分隐藏（如 OpenAI o 系列）。
- **Thinking budget**：控制模型在一个请求上花多少推理算力。是 reasoning 可控性的核心工程接口。
- **Process Reward Model (PRM)**：不只评估最终答案对不对，而是评估每个推理步骤是否合理。用于训练和评估 reasoning model 的推理过程质量。

### 关键区分

- **Reasoning vs Generation**：reasoning 是"在输出前思考"，generation 是"直接输出"。同一个模型可以在两种模式间切换。
- **忠实 CoT vs 不忠实 CoT**：模型展示的思考过程是否真的反映了它的内部计算？这是开放问题，直接影响可解释性和安全性。

## 当前技术格局（截至 2026-04）

Reasoning 已经从「外挂式 prompt 技巧」全面转向「训练内化」。前沿模型（o3、Claude with extended thinking、Gemini 3 Deep Think、DeepSeek-R1 系）都将 reasoning 作为训练目标。

下一阶段焦点是 **reasoning 的可控性和经济性**：thinking budget 调节、按任务难度分配算力、reasoning trace 的处理。

**主要玩家**：OpenAI（o 系列）、Anthropic（Claude extended thinking）、Google DeepMind（Gemini Deep Think）、DeepSeek（R1 系列，开源标杆）。

**关键 benchmark**：GPQA（研究生级别问答）、MATH-500（数学推理）、ARC-AGI（抽象推理）、LiveCodeBench（代码推理）。

## 演进路径

- **2022 - 2023**：外挂式 reasoning 时代。CoT、Self-Consistency、ToT、ReAct 等技术涌现。共同特征：模型本身不会思考，工程层搭脚手架模拟。
- **2024 末**：分水岭。OpenAI o1 发布，reasoning 从 prompt 技巧转向训练内化。模型自发出现回溯、自我纠错行为。
- **2025**：reasoning model 普及到几乎所有前沿模型（o3、Claude extended thinking、Gemini Deep Think、DeepSeek-R1）。
- **2026**：reasoning budget 成为可调参数；模型在 reasoning 强度上提供分级（如 Deep Think 模式）。焦点从"能不能想"转向"想多少、花多少"。

## 对四层骨架的影响

### 第一层（Agent 内核）：塌缩最严重

**塌缩**：
- CoT prompting 大幅贬值（对 reasoning model 写 "step by step" 反而有害）
- Self-Consistency 类采样投票边际收益骤降
- Reflexion / Self-Refine 这类外部反思循环大幅简化
- Plan-and-Execute 部分塌缩（planning 和 execution 可一次完成）
- 经典 ReAct 紧密交错被「思考相位 + 行动相位」取代

**涌现**：
- Reasoning budget 管理成为新工程问题
- Model routing：简单任务 → 快模型，难任务 → reasoning model
- Reasoning trace 的处理（保留多少到后续 context）
- 思考-行动协同模式（边想边做 vs 想完再做）

### 第二层（与世界的接口）：影响中等

**塌缩**：
- 复杂多跳 RAG 编排链路被简化（reasoning model 自己决定何时再查）
- Query rewriting 等小工程动作被吸收

**涌现**：
- 工具描述的写法变化：从严格 schema 转向更长的「教学式」描述
- 工具结果格式的「思考友好度」成为新指标

### 第三层（多 Agent 协作）：双向变化最深刻

**塌缩**：
- 用多 Agent 模拟「团队思考」提升质量这一动机大幅消失
- 「Researcher + Critic + Writer」这类纯认知性多 agent 编排

**涌现（出于新动机）**：
- 多 Agent 用于并行化加速（reasoning model 单次慢）
- 多 Agent 用于隔离 context 污染（长 reasoning trace 需要隔离）
- 多 Agent 用于成本混合（贵的 reasoning + 便宜的执行）

**关键洞察**：多 Agent 的设计动机从「提升质量」转向「提升效率和隔离性」。

### 第四层（生产化）：变化最易被低估

**塌缩**：
- 显式 CoT 评估方式失效（trace 隐藏）
- 旧的延迟基线（秒级 → 分钟级）

**涌现**：
- 成本工程：单次调用成本可能 20-50 倍增长
- Process eval：从「答对了吗」到「思考路径合理吗」
- Reasoning trace 的可观测性工具（折叠 / 展开 / 语义分段）
- 容量规划重做（rate limit 频繁触发）
- 新的安全面：模型在思考链中可能「想偏」绕过 guardrail

## 与其他维度的耦合

- **× Inference-Time Compute**：本质上 reasoning = 花更多 inference compute。这两个维度高度耦合
- **× Tool Use**：reasoning model 的工具调用模式和普通模型不同，需要重新设计
- **× Memory**：reasoning + memory 的组合是「能反思过去并改进」的 agent，是下一代关键耦合点
- **× Long Context**：reasoning trace 本身消耗 context，两者存在竞争关系

## 关键不确定性

- Reasoning model 的 trace 是否值得 / 应该让用户看到？目前各家做法不一
- Reasoning 能力是否有 ceiling？还是可以通过更多 RL 持续提升？
- 「假装思考」（reasoning trace 看起来对但结论错）的检测方法
- Reasoning 在 tool use 任务上的优势是否会随模型能力均衡而消失？

## 信息源

跟踪这个维度的少量优质来源：
- **模型技术报告**：OpenAI（o-系列 system card）、Anthropic（Claude model card）、DeepMind（Gemini technical report）——每次新模型发布时读 reasoning 相关章节
- **个人博客**：Lilian Weng（lilianweng.github.io，综述质量极高，尤其关注她的 agent / reasoning 相关文章）；Sebastian Raschka（Ahead of AI newsletter，周更，善于追踪 reasoning benchmark 变化）
- **Benchmark 追踪**：GPQA、MATH-500、ARC-AGI 的 leaderboard 变动——用于判断 reasoning 能力是否仍在提升
- **arXiv 关键词**：process reward model、verifiable reward、test-time compute scaling、chain-of-thought faithfulness

## 更新日志

- 2026-04-26：初次创建（基于和 Claude 的推演）

