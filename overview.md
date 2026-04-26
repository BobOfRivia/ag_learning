# 体系总览

这份文档是整个知识体系的地图。任何深入阅读应该从这里开始定位，再跳转到具体维度或层级文件。

## 核心抽象：地基与建筑

```
                  Agent 工程（建筑）
        ┌──────────────────────────────────┐
   L4   │  生产化                            │  Evals, Tracing, Cost, Safety
   L3   │  多 Agent 协作                     │  Orchestration, Communication
   L2   │  与世界的接口                      │  Tools, MCP, RAG, Memory
   L1   │  Agent 内核                        │  Reasoning loop, Context engineering
        ├──────────────────────────────────┤
   L0   │  LLM 本身（地基）                  │  七大维度 + 准维度
        └──────────────────────────────────┘

         地基变化 ─────────► 建筑塌缩 / 涌现
```

> **L2 的内部结构**：L2 包含的四个子领域（Tools/MCP、RAG、Memory、其他外部接口）演进速度和复杂度差异很大。RAG 在第三代已大面积塌缩，MCP 正处标准化爆发期，Memory 可能是下一代核心主题。因此 L2 层级文件内部按子领域分节，而非作为单一主题叙述。

**核心命题**：Agent 工程的所有复杂性，本质上都在解决「在不可靠的 LLM 上构建可靠系统」这一根本张力。当 LLM 本身（L0）的能力变化时，上层四层的某些工程必要性会消失（塌缩），新的工程问题会出现（涌现）。

## 七大维度速览

| # | 维度 | 一句话定义 | 当前演进阶段 |
|---|---|---|---|
| 1 | Reasoning | 模型「思考多深」的能力 | 从 prompt 外挂转向训练内化（已发生） |
| 2 | Long Context | 单次能感知多少 tokens | 1M+ 普及，瓶颈转向利用率 |
| 3 | Multimodality | 跨模态原生处理 | early-fusion 普及中 |
| 4 | Tool Use | 调用外部工具的原生能力 | 原生 + 并行调用 |
| 5 | Instruction Following | 指令遵循 / 可控性 | 分层化 + 抗注入 |
| 6 | Memory | 跨 session 记忆 | 工程外挂为主，向地基迁移中（最热） |
| 7 | Inference-Time Compute | 推理时算力可控 | thinking budget 已可调 |

**准维度**（边界模糊但值得跟踪）：Agentic Capability、Self-Evolution、Speed、Cost、Ecosystem Maturity。

> **关于 Ecosystem Maturity**：部分工程变化不由 LLM 能力驱动，而由生态标准化驱动——MCP 出现让 Tool Use 工程方式剧变，但触发因素是外部协议而非模型能力变化。这类事件同样会造成塌缩/涌现，需要被跟踪。在塌缩-涌现日志中，此类事件的"触发"字段标注为生态事件而非某个维度变化。

## 代际演进速览

```
第一代 (2020-2022)  prompt era
    raw LLM + prompt engineering，工程几乎全在 prompt 层

第二代 (2023-2024 中)  tool-rag era
    function calling 工程化，RAG 成为标配，第二层成为工程重点

第三代 (2024 末-2026)  reasoning era
    o1/Claude/DeepSeek-R1 等让 reasoning 内化
    第一层大量塌缩，第四层（成本/延迟/可观测性）重要性凸显
    
第四代 (2026-)  memory era（猜测）
    memory 从工程外挂向地基内化
    第三层（多 Agent）的设计动机重塑
```

## 双索引使用方式

任何一个知识点应该回答两个问题：
- **空间锚点**：它在哪一层？涉及哪个维度？
- **时间锚点**：它属于哪一代？是塌缩还是涌现？

例：
> 「Reasoning Model 的 thinking budget」
> - 空间：维度 7（Inference-Time Compute），影响 L1 和 L4
> - 时间：第三代涌现物
> - 性质：涌现（同时让 L1 的 self-consistency 类技术塌缩）

## 归位规则：当一个概念同时跨 L0 和工程层

某些概念（典型如 Memory、Tool Use）同时存在"地基能力"和"工程方案"两个面向。归位时遵循以下约定：

- **维度文件（dimensions/）记录能力本身的演进**：模型原生能做什么、做到什么程度、趋势是什么
- **层级文件（layers/）记录工程应对方式的演进**：工程上怎么用、怎么补、怎么绕
- 同一个事件可以同时触发两边的更新。例如"某公司发布 persistent memory API"：维度文件更新 Memory 能力的当前状态，L2 文件更新工程层的选型和方案变化
- 在塌缩-涌现日志中，用关联文件字段标注同时更新了哪些文件

## 关键耦合关系

维度之间不是独立的。重点关注以下耦合：

- **Reasoning × Memory**：会催生「能反思过去并学习」的 agent 形态
- **Multimodality × Tool Use**：催生 GUI / browser agent 的范式转变
- **Long Context × Memory**：经常被混淆但本质不同（前者是单次感知，后者是跨次积累）
- **Reasoning × Inference-Time Compute**：reasoning 能力本质上是花更多 inference compute 换来的

## 核心机制：塌缩 - 涌现

每次地基变化，会同时产生两类影响：

- **塌缩（collapse）**：原本需要工程解决的问题，被模型自身能力消解
- **涌现（emergence）**：新的工程问题因新能力出现

经典塌缩 - 涌现循环：

| 塌缩了什么 | 因为什么内化 | 涌现了什么 |
|---|---|---|
| Chain-of-Thought prompting | Reasoning 内化 | Reasoning budget 管理 |
| 多跳 RAG 编排 | Long context + reasoning | Context 利用率工程 |
| ReAct 模拟工具调用 | 原生 function calling | 千级工具池的检索 |
| 向量库选型（猜测） | Memory 内化（猜测） | Memory eval / 治理 |

完整记录见 `logs/collapse-emergence-log.md`。

## 元原则

1. **先全景，再追踪**：先建立完整的知识地图（概念、技术、格局），再在此基础上追踪变化
2. **空白是待填的**：覆盖缺失应被识别和补上，但完整指的是结构性覆盖，不是细节穷举
3. **双索引同步**：每个条目同时挂在维度和时间线上
4. **全景 + 变化**：既描述「当前是什么样」，也追踪「最近发生了什么变化」
5. **结构稳定，内容生长**：维度框架可以稳定较长时间，文件内容随学习逐步丰满

