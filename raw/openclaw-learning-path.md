# AI Agent 工程师 JOB MODEL × OpenClaw 项目学习路径

> 基于 2026 年 AI Agent 工程师岗位能力模型（JOB MODEL），将 OpenClaw 项目中的核心模块、设计模式和技术实现映射到每个能力项，形成一套通过源码阅读和实践掌握行业核心竞争力的系统化学习路径。

---

## 一、2026 AI Agent 工程师岗位能力模型（JOB MODEL）

基于对当前行业 JD、技术社区、招聘趋势的综合调研，2026 年 AI Agent 工程师核心能力可归纳为 **11 大支柱**：


| #   | 能力支柱                            | 关键词                                       |
| --- | ------------------------------- | ----------------------------------------- |
| 1   | **Agent 系统架构设计**                | Gateway、Agent Runtime、消息路由、通道抽象           |
| 2   | **Tool Calling 与协议工程（MCP/A2A）** | Schema 设计、工具管线、MCP 协议、安全审批                |
| 3   | **Context Engineering（上下文工程）**  | 系统提示词构造、上下文引擎、Compaction、Token 预算、信息流设计  |
| 4   | **RAG 与记忆系统**                   | 向量嵌入、混合检索、MMR、查询扩展、时间衰减                   |
| 5   | **多模型管理与可靠性工程**                 | 模型降级/Failover、Auth 轮转、重试/断路器              |
| 6   | **多 Agent 编排**                  | 子 Agent 生命周期、Supervisor-Worker、ACP 协议     |
| 7   | **可观测性与评测**                     | 日志/追踪/指标、Token 成本、缓存追踪、诊断、Agent 行为评测     |
| 8   | **安全与合规**                       | 沙箱隔离、路径策略、输入/输出过滤、SSRF 防护                 |
| 9   | **流式工程（Streaming）**             | 多 Provider 流归一化、SSE/WebSocket、分段推送、流中断恢复 |
| 10  | **多模态与新兴能力**                    | Extended Thinking、图片/媒体处理、Canvas 交互式工件    |
| 11  | **成本优化工程**                      | Prompt Caching、Cache TTL、Token 预算、模型选型策略  |


---

## 二、项目概览：OpenClaw 是什么

OpenClaw 是一个 **生产级多通道 AI Agent 网关**（Multi-channel AI Gateway），核心特征：

- **多通道统一接入**：Telegram、Discord、Slack、WhatsApp、Signal、iMessage、LINE、Teams 等 20+ 通道
- **多模型支持**：OpenAI、Anthropic Claude、Google Gemini、Ollama、Mistral、Moonshot 等 30+ 模型提供商
- **完整 Agent 运行时**：工具调用、MCP 集成、子 Agent 编排、流式输出
- **插件/扩展架构**：通过 Plugin SDK 扩展通道、模型、工具
- **生产级基础设施**：沙箱隔离（Docker/SSH）、Auth 管理、速率限制、审计日志

```
openclaw/
├── src/
│   ├── agents/          # Agent 运行时核心（~596 文件，最大模块）
│   ├── gateway/         # Gateway 服务器（WebSocket + HTTP）
│   ├── routing/         # 消息路由
│   ├── channels/        # 通道抽象层
│   ├── memory/          # RAG / 记忆系统
│   ├── plugin-sdk/      # 插件 SDK
│   ├── plugins/         # 插件加载与管理
│   ├── config/          # 配置系统
│   ├── security/        # 安全审计
│   ├── logging/         # 日志子系统
│   ├── infra/           # 基础设施（重试、退避、归档等）
│   ├── media/           # 媒体管线
│   ├── acp/             # Agent Communication Protocol
│   ├── hooks/           # 钩子系统
│   └── cli/             # CLI 入口
├── extensions/          # 30+ 扩展插件
├── apps/                # iOS / Android / macOS 客户端
└── docs/                # 文档
```

---

## 三、能力 × 项目模块映射（学习主线）

### 能力 1: Agent 系统架构设计

**要掌握的核心能力：** 理解一个生产级 Agent 从接收消息到返回结果的完整生命周期。

**JOB MODEL 考点：** 系统设计面试、Agent 架构图绘制、消息流转链路分析。

#### 核心模块


| 模块          | 路径                                              | 行数      | 学什么                                               |
| ----------- | ----------------------------------------------- | ------- | ------------------------------------------------- |
| Gateway 服务器 | `src/gateway/server.impl.ts`                    | 1,354   | WebSocket + HTTP 混合服务架构，消息接入与分发                   |
| Agent 运行时引擎 | `src/agents/pi-embedded-runner/`                | ~14,700 | Agent 执行闭环：session → model call → tool → response |
| 消息路由        | `src/routing/resolve-route.ts`                  | ~130    | 从发送者身份解析到目标 Agent/Session                         |
| 通道抽象        | `src/channels/` + `src/plugin-sdk/channel-*.ts` | —       | 20+ 通道的统一抽象层                                      |
| 通道回复管线      | `src/plugin-sdk/channel-reply-pipeline.ts`      | —       | 消息从 Agent 回到通道的完整管线                               |
| 配置系统        | `src/config/`                                   | —       | 运行时配置热重载、多层级配置合并                                  |


#### 关键数据流

```
用户消息 → [通道插件] → Gateway WebSocket/HTTP → 消息路由(routing)
→ Session 解析 → Agent Runner(pi-embedded-runner)
→ 模型调用 + 工具执行循环 → 流式输出
→ 通道回复管线(channel-reply-pipeline) → [通道插件] → 用户
```

#### 推荐阅读路径

1. **入口理解**：`src/gateway/server.impl.ts` — 看 Gateway 如何启动、注册路由、处理 WebSocket 连接
2. **消息进入**：`src/gateway/server-chat.ts` — 看消息如何被接收和处理
3. **路由解析**：`src/routing/resolve-route.ts` — 看消息如何找到目标 Agent
4. **Agent 执行**：`src/agents/pi-embedded-runner/runs.ts` — 看 Agent 一次运行的编排逻辑
5. **流式输出**：`src/agents/pi-embedded-subscribe.ts`（726 行）— 看流式响应如何组装和推送
6. **通道回复**：`src/plugin-sdk/channel-reply-pipeline.ts` — 看回复如何通过通道返回

#### 动手实践

- **任务 A**：跟着推荐阅读路径，画一张完整的系统架构图（从用户消息进入到回复返回）
- **任务 B**：在本地启动 Gateway（`pnpm dev`），用 WebSocket 客户端发送一条消息，追踪日志中的完整链路
- **任务 C**：阅读 `src/channels/` 和一个通道扩展（如 `extensions/telegram/`），对比通道抽象层的接口设计

#### 面试应用

- "画一个 AI Agent 系统的架构图" — 直接用 OpenClaw 的 Gateway → Routing → Runner → Channel 架构
- "设计一个支持多通道的 AI 助手" — 展示通道抽象层的插件化设计
- "如何处理长连接和高并发" — 展示 WebSocket 管理和 Session 隔离

---

### 能力 2: Tool Calling 与协议工程（MCP）

**要掌握的核心能力：** 精通工具定义、Schema 设计、工具执行管线，理解 MCP 协议集成。

**JOB MODEL 考点：** Function Calling 原理、MCP 协议理解、工具安全审批、Schema 设计规范。

#### 核心模块


| 模块           | 路径                                         | 行数  | 学什么                                            |
| ------------ | ------------------------------------------ | --- | ---------------------------------------------- |
| 工具目录系统       | `src/agents/tool-catalog.ts`               | 342 | 工具分类、Profile 管理（minimal/coding/messaging/full） |
| 工具 Schema 定义 | `src/agents/pi-tools.schema.ts`            | 209 | TypeBox 定义 JSON Schema，LLM 友好的 Schema 规范       |
| 工具定义适配器      | `src/agents/pi-tool-definition-adapter.ts` | 236 | before/after 钩子机制                              |
| 工具权限策略       | `src/agents/tool-policy.ts`                | 210 | 工具级别的权限控制                                      |
| 策略管线         | `src/agents/tool-policy-pipeline.ts`       | 132 | 多策略组合执行                                        |
| 工具循环检测       | `src/agents/tool-loop-detection.ts`        | 623 | 检测 Agent 陷入工具调用死循环                             |
| Bash 工具执行    | `src/agents/bash-tools.exec.ts` 系列         | —   | PTY 管理、超时、安全审批                                 |
| MCP stdio 传输 | `src/agents/mcp-stdio.ts`                  | 79  | MCP 协议 stdio 传输层实现                             |
| MCP 工具打包     | `src/agents/pi-bundle-mcp-tools.ts`        | 229 | 将 MCP Server 工具转换为 Agent 可用工具                  |
| MCP 嵌入集成     | `src/agents/embedded-pi-mcp.ts`            | 29  | MCP 嵌入式集成入口                                    |


#### 关键设计思想

**1. 工具 Profile 分级（tool-catalog.ts）**

项目按场景将工具分为 4 个 Profile：

- `minimal` — 最小工具集
- `coding` — 编码场景（read/write/edit/bash/search）
- `messaging` — 消息场景（send/reply/channel tools）
- `full` — 完整工具集

这是一个非常实用的设计模式：不同场景给 Agent 不同的工具集，避免工具过多导致 LLM 选择困难。

**2. Schema 设计规范（pi-tools.schema.ts）**

项目规范明确禁止在工具 Schema 中使用 `anyOf`/`oneOf`/`allOf`（许多 LLM 对这些复杂 Schema 支持不好），改用 `stringEnum` 和 `Type.Optional`。这是实战中的重要经验。

**3. 工具执行管线（before/after 钩子）**

```
工具调用请求 → before_tool_call 钩子 → 权限策略检查 → 循环检测
→ 实际执行 → after_tool_call 钩子 → 结果返回给 LLM
```

**4. MCP 集成模式**

OpenClaw 将外部 MCP Server 通过 stdio 连接，获取其 `tools/list`，然后将这些工具"翻译"为 Agent 内部的工具格式，无缝集成到工具目录中。

#### 推荐阅读路径

1. `tool-catalog.ts` — 理解工具如何组织（分组、Profile、注册）
2. `pi-tools.schema.ts` — 学习 JSON Schema 在 Function Calling 中的最佳实践
3. `pi-tool-definition-adapter.ts` — 理解 before/after 钩子如何拦截工具调用
4. `tool-policy.ts` + `tool-policy-pipeline.ts` — 理解工具权限控制管线
5. `tool-loop-detection.ts` — 理解如何防止 Agent 工具调用死循环
6. `bash-tools.exec.ts` — 追踪一个 Shell 命令从定义到执行的完整流程
7. `mcp-stdio.ts` → `pi-bundle-mcp-tools.ts` — 理解 MCP 协议集成

#### 动手实践

- **任务 A**：阅读 `tool-catalog.ts`，列出所有 Profile（minimal/coding/messaging/full）包含的工具清单
- **任务 B**：仿照 `pi-tools.schema.ts` 中的现有工具，用 TypeBox 定义一个新的工具 Schema（如 `weather_query`）
- **任务 C**：阅读 `tool-loop-detection.ts`，理解检测算法后，设计一个能触发循环检测的测试场景

#### 面试应用

- "如何设计一个工具调用系统" — 展示 catalog + schema + policy + pipeline 的分层设计
- "MCP 协议是什么，如何集成" — 展示 stdio 传输 + 工具翻译的实现细节
- "如何防止 Agent 陷入工具调用死循环" — 展示 loop detection 算法
- "工具 Schema 设计有什么注意事项" — 展示避免 `anyOf` 等 LLM 不友好模式

---

### 能力 3: Context Engineering（上下文工程）

**要掌握的核心能力：** 动态系统提示词构造、可插拔上下文引擎、上下文窗口管理、对话压缩（Compaction）、信息流设计。

**JOB MODEL 考点：** System Prompt 设计、上下文窗口优化、Token 管理、信息流时机控制、上下文生命周期管理。

> **为什么是 Context Engineering 而不是 Prompt Engineering？** 2025-2026 年行业已从单纯的"提示词工程"演进为更广泛的"上下文工程"（Andrej Karpathy 等人提出）。Context Engineering 关注的不仅是 System Prompt 怎么写，而是**什么信息、在什么时机、以什么形式注入到模型上下文中**，以及上下文的完整生命周期管理（创建 → 组装 → 压缩 → 归档）。

#### 核心模块


| 模块           | 路径                                           | 行数  | 学什么                                |
| ------------ | -------------------------------------------- | --- | ---------------------------------- |
| 系统提示词构造      | `src/agents/system-prompt.ts`                | 719 | 动态 System Prompt 模块化组装的最佳实践       |
| 可插拔上下文引擎     | `src/context-engine/`                        | 7 文件 | 上下文引擎的抽象层：assemble/compact/ingest |
| 上下文引擎注册表     | `src/context-engine/registry.ts`             | 340 | 引擎工厂注册、按 slot 解析、session 兼容代理     |
| 上下文引擎类型      | `src/context-engine/types.ts`                | 178 | 核心类型：AssembleResult/CompactResult  |
| 遗留引擎兼容       | `src/context-engine/legacy.ts`               | —   | 旧 session 到新引擎的过渡策略               |
| 上下文窗口守卫      | `src/agents/context-window-guard.ts`         | 75  | 多源 context window 大小解析与阈值控制        |
| 对话压缩         | `src/agents/compaction.ts`                   | 464 | Token 超限时的分块摘要 + 合并策略              |
| Bootstrap 预算 | `src/agents/bootstrap-budget.ts`             | —   | System Prompt + 工具定义的 Token 预算管理   |
| Prompt 组合场景  | `src/agents/prompt-composition-scenarios.ts` | —   | 不同场景下的 Prompt 组合方式                 |
| 提示词安全化       | `src/agents/sanitize-for-prompt.ts`          | —   | 防止提示词注入的输入清洗                       |
| 系统提示词参数      | `src/agents/system-prompt-params.ts`         | —   | 提示词参数的解析与验证                        |


#### 关键设计思想

**1. 可插拔上下文引擎（context-engine/）**

这是 2026 年 Context Engineering 的核心设计模式。OpenClaw 抽象了一个 `ContextEngine` 接口，定义了上下文生命周期的 5 个阶段：

```
bootstrap → ingest → assemble → compact → afterTurn
  初始化      消息摄入    上下文组装    压缩归档    轮次后处理
```

通过 `ContextEngineFactory` 注册机制，可以插入不同的引擎实现（Legacy、Plugin 提供的新引擎等），而 Runner 代码无需感知具体实现。这是"信息流设计"思想的工程化体现。

**2. 系统提示词的模块化组装（system-prompt.ts）**

这是本项目最值得深入学习的文件之一。它将 System Prompt 拆分为多个独立段落（Section），按需动态组装：

```
System Prompt = Identity（身份）
              + Skills（技能/SKILL.md）
              + Memory Recall（记忆检索指令）
              + Authorized Senders（授权发送者）
              + Tooling（可用工具列表）
              + Workspace（工作空间上下文）
              + Runtime（运行时信息）
```

**3. PromptMode 分级**

- `"full"` — 主 Agent，包含所有段落
- `"minimal"` — 子 Agent，仅包含 Tooling/Workspace/Runtime（减少 Token 消耗）
- `"none"` — 仅基本身份行

**4. 对话压缩（Compaction）策略**

当会话 Token 超出上下文窗口时：

1. 将历史消息分为多个 chunk
2. 每个 chunk 独立生成摘要
3. 合并多个摘要为一个连贯的总结
4. 关键约束：
  - **标识符保留**：UUID、URL、文件路径等不能被摘要丢失
  - **安全边际**：20% buffer 应对 `estimateTokens()` 的不精确
  - **重试容错**：摘要生成失败时的指数退避重试

**5. Token 预算管理**

```
总 Context Window = System Prompt 占用 + 工具定义占用 + 用户对话空间
                    ↓
              bootstrap-budget 负责计算前两项
              剩余空间留给用户对话历史
              超限 → 触发 compaction
```

#### 推荐阅读路径

1. `context-engine/types.ts` — 先理解上下文引擎的抽象接口（AssembleResult/CompactResult/IngestResult）
2. `context-engine/registry.ts` — 理解引擎工厂如何注册和解析（插件化设计）
3. `context-engine/legacy.ts` — 理解新旧引擎的过渡兼容策略
4. `system-prompt.ts` — 逐行阅读，理解每个 Section 的构建逻辑（**黄金文件**）
5. `context-window-guard.ts` — 理解多源 context window 解析（模型默认 → 配置覆盖 → Agent Cap）
6. `compaction.ts` — 理解 Token 超限时的工程化解决方案
7. `bootstrap-budget.ts` — 理解 Token 预算如何分配
8. `sanitize-for-prompt.ts` — 理解防注入的输入清洗

#### 动手实践

- **任务 A**：画出 `ContextEngine` 接口的 5 个生命周期阶段图，标注每个阶段的输入/输出类型
- **任务 B**：阅读 `system-prompt.ts`，列出所有 Section 及其开关条件，用 Markdown 表格记录
- **任务 C**：尝试在 Compaction 逻辑中增加一种新的标识符保留规则（如保留 JSON 路径）

#### 面试应用

- "什么是 Context Engineering" — 展示上下文引擎的 5 阶段生命周期：bootstrap → ingest → assemble → compact → afterTurn
- "如何设计一个 AI Agent 的 System Prompt" — 展示模块化组装模式
- "上下文窗口不够用怎么办" — 展示 Compaction + Token 预算管理
- "如何处理超长对话" — 展示分块摘要 + 标识符保留策略

---

### 能力 4: RAG 与记忆系统

**要掌握的核心能力：** 向量嵌入、混合检索（Dense + Sparse）、语义分块、记忆管理。

**JOB MODEL 考点：** Embedding 模型选型、检索策略、分块策略、向量数据库、记忆持久化。

#### 核心模块


| 模块          | 路径                                     | 行数  | 学什么                                                  |
| ----------- | -------------------------------------- | --- | ---------------------------------------------------- |
| 记忆管理器       | `src/memory/manager.ts`                | 840 | 嵌入生成、文件监听、增量同步、原子重建索引                                |
| 嵌入层抽象       | `src/memory/embeddings.ts`             | 324 | 多 Provider 嵌入抽象（OpenAI/Gemini/Mistral/Voyage/Ollama） |
| 混合检索        | `src/memory/hybrid.ts`                 | 155 | Dense + Sparse 混合搜索策略                                |
| 搜索管理器       | `src/memory/search-manager.ts`         | 253 | 搜索请求编排和结果融合                                          |
| MMR 算法      | `src/memory/mmr.ts`                    | —   | Maximal Marginal Relevance，检索结果多样性                   |
| 查询扩展        | `src/memory/query-expansion.ts`        | —   | 查询改写/扩展技术                                            |
| 语义分块        | `src/plugin-sdk/text-chunking.ts`      | —   | 文本分块策略                                               |
| 时间衰减        | `src/memory/temporal-decay.ts`         | —   | 越旧的记忆权重越低                                            |
| SQLite 向量存储 | `src/memory/sqlite-vec.ts`             | —   | 本地向量存储实现                                             |
| 嵌入分块限制      | `src/memory/embedding-chunk-limits.ts` | —   | 各 Provider 的分块大小限制管理                                 |
| Agent 记忆工具  | `src/agents/memory-search.ts`          | —   | Agent 运行时调用记忆搜索的工具定义                                 |
| 批量嵌入        | `src/memory/batch-runner.ts`           | —   | 批量嵌入生成的调度和执行                                         |


#### 关键设计思想

**1. Provider-Agnostic 嵌入层**

嵌入层统一抽象了多个 Provider，每个 Provider 有独立实现：

- `embeddings-openai.ts` — OpenAI text-embedding-3-small/large
- `embeddings-gemini.ts` — Google Gemini embedding
- `embeddings-voyage.ts` — Voyage AI
- `embeddings-mistral.ts` — Mistral embed
- `embeddings-ollama.ts` — 本地 Ollama

**2. 混合检索（Hybrid Search）**

```
用户查询 → 查询扩展(query-expansion)
         → Dense 检索(向量相似度) + Sparse 检索(全文匹配)
         → 结果融合(hybrid.ts)
         → MMR 重排序(mmr.ts) — 确保多样性
         → 时间衰减加权(temporal-decay.ts)
         → 返回 Top-K 结果
```

这是一个教科书级别的完整 RAG 管线。

**3. 增量同步与原子重建**

记忆管理器支持：

- 文件变更监听 → 增量更新嵌入
- 原子性重建索引 — 保证在重建过程中旧索引仍可用
- 向量去重 — 避免重复文档产生冗余向量

#### 推荐阅读路径（自底向上）

1. `text-chunking.ts` — 理解文本如何被分块
2. `embeddings.ts` → `embeddings-openai.ts` — 理解嵌入生成
3. `sqlite-vec.ts` — 理解向量如何存储和检索
4. `hybrid.ts` — 理解 Dense + Sparse 混合检索
5. `search-manager.ts` — 理解检索请求的编排
6. `mmr.ts` — 理解 MMR 多样性重排序算法
7. `query-expansion.ts` — 理解查询扩展技术
8. `temporal-decay.ts` — 理解时间衰减权重
9. `manager.ts` — 理解记忆管理器的全局编排

#### 动手实践

- **任务 A**：阅读 `hybrid.ts`，画出 Dense + Sparse 检索结果融合的流程图
- **任务 B**：阅读 `mmr.ts`，用数学公式写出 MMR 的计算过程，并用简单的 Python 脚本验证
- **任务 C**：对比 `embeddings-openai.ts` 和 `embeddings-ollama.ts`，列出两者在 API 调用、分块限制、错误处理上的差异

#### 面试应用

- "设计一个 RAG 系统" — 展示从分块到检索到重排序的完整管线
- "Dense 检索和 Sparse 检索有什么区别" — 展示 hybrid.ts 的融合策略
- "如何保证检索结果的多样性" — 展示 MMR 算法实现
- "如何选择 Embedding 模型" — 展示多 Provider 嵌入层的抽象设计

---

### 能力 5: 多模型管理与可靠性工程

**要掌握的核心能力：** 模型路由/降级策略、Auth 管理、重试与断路器、故障转移。

**JOB MODEL 考点：** 多模型管理、Failover 设计、API Key 管理、成本优化、可靠性保障。

#### 核心模块


| 模块              | 路径                                                | 行数  | 学什么                     |
| --------------- | ------------------------------------------------- | --- | ----------------------- |
| 模型降级/故障转移       | `src/agents/model-fallback.ts`                    | 828 | 多模型降级链、超时处理、错误分类        |
| 模型选择            | `src/agents/model-selection.ts`                   | —   | 模型别名、配置解析、Provider 路由   |
| 模型目录            | `src/agents/model-catalog.ts`                     | —   | 统一的模型注册表                |
| 模型配置            | `src/agents/models-config.ts`                     | 128 | 多 Provider 模型配置管理       |
| 模型兼容层           | `src/agents/model-compat.ts`                      | —   | 不同 Provider API 差异的兼容处理 |
| Auth Profile 管理 | `src/agents/auth-profiles.ts` 系列                  | —   | 多 Auth Profile 管理与轮转    |
| API Key 轮转      | `src/agents/api-key-rotation.ts`                  | —   | 密钥自动轮转机制                |
| Failover 错误分类   | `src/agents/failover-error.ts`                    | —   | 区分可重试/不可重试错误            |
| 重试基础设施          | `src/infra/retry.ts`                              | 136 | 通用异步重试框架                |
| 退避策略            | `src/infra/backoff.ts`                            | 28  | 指数退避算法                  |
| Provider 发现     | `src/agents/models-config.providers.discovery.ts` | —   | 运行时自动发现可用模型             |
| 模型工具支持检测        | `src/agents/model-tool-support.ts`                | —   | 检测模型是否支持工具调用            |


#### 关键设计思想

**1. 模型降级链（model-fallback.ts）**

这是可靠性工程的核心文件（828 行），设计精妙：

```
主模型调用 → 失败（超时/限流/服务不可用）
  → 判断是否 AbortError（用户主动取消则不降级）
  → 判断是否 FailoverError（可降级错误）
  → 尝试下一个候选模型
  → 所有候选模型耗尽 → 报告最终错误
```

关键概念：

- **ModelCandidate** — 候选模型（provider + model + auth profile）
- **FallbackAttempt** — 每次尝试的记录（模型、耗时、错误原因）
- **Cooldown 机制** — 失败的 Auth Profile 进入冷却期，暂时跳过

**2. Auth Profile 轮转**

支持多个 API Key（Auth Profile）的自动轮转：

- 按优先级排序
- 失败时记录冷却期
- 冷却期过后自动恢复
- 运行时快照保存，重启后恢复状态

**3. 错误分类**

```
错误 → 是否 AbortError? → 是 → 停止，不降级
     → 是否 FailoverError? → 是 → 降级到下一个模型
        → 是否 TimeoutError? → 是 → 降级 + 标记超时
        → 是否 ContextOverflow? → 是 → 降级 + 标记上下文溢出
     → 未知错误 → 尝试降级
```

#### 推荐阅读路径

1. `failover-error.ts` — 理解错误分类体系
2. `model-selection.ts` — 理解模型是如何被选择的
3. `model-fallback.ts` — 重点文件，理解完整的降级链
4. `auth-profiles.ts` 系列 — 理解 API Key 管理与轮转
5. `infra/retry.ts` + `infra/backoff.ts` — 理解通用重试和退避
6. `models-config.providers.discovery.ts` — 理解运行时模型发现

#### 动手实践

- **任务 A**：阅读 `model-fallback.ts`，画出完整的降级链决策树（包括 AbortError、FailoverError、TimeoutError 的分支）
- **任务 B**：阅读 `infra/retry.ts` + `infra/backoff.ts`，用 Python 实现一个等价的指数退避重试函数
- **任务 C**：模拟一个 3 个 API Key 的轮转场景，画出 Key1 失败 → Cooldown → Key2 → 成功的时序图

#### 面试应用

- "模型 API 挂了怎么办" — 展示 Failover 降级链设计
- "如何管理多个 API Key" — 展示 Auth Profile 轮转 + Cooldown 机制
- "重试和断路器如何设计" — 展示 retry.ts + backoff.ts + cooldown 组合
- "如何优化 LLM 调用成本" — 展示模型选择 + 降级策略

---

### 能力 6: 多 Agent 编排

**要掌握的核心能力：** 子 Agent 生命周期管理、Supervisor-Worker 模式、Agent 间通信。

**JOB MODEL 考点：** 多 Agent 系统设计、任务分解与委托、Agent 间消息传递、深度限制。

#### 核心模块


| 模块          | 路径                                        | 行数    | 学什么                              |
| ----------- | ----------------------------------------- | ----- | -------------------------------- |
| 子 Agent 注册表 | `src/agents/subagent-registry.ts`         | 1,705 | 子 Agent 的完整生命周期管理                |
| 子 Agent 生成  | `src/agents/subagent-spawn.ts`            | 841   | 子 Agent 的创建、参数配置、工作空间分配          |
| 子 Agent 控制  | `src/agents/subagent-control.ts`          | 827   | 暂停/恢复/终止子 Agent                  |
| 子 Agent 通知  | `src/agents/subagent-announce.ts`         | 1,508 | 子 Agent 完成后通知父 Agent             |
| 深度限制        | `src/agents/subagent-depth.ts`            | 176   | 防止子 Agent 无限递归                   |
| 子 Agent 能力  | `src/agents/subagent-capabilities.ts`     | 156   | 子 Agent 可用能力定义                   |
| 生命周期事件      | `src/agents/subagent-lifecycle-events.ts` | —     | Agent 生命周期事件定义                   |
| 孤儿恢复        | `src/agents/subagent-orphan-recovery.ts`  | —     | 处理父 Agent 崩溃后的孤儿子 Agent          |
| ACP 客户端     | `src/acp/client.ts`                       | 638   | Agent Communication Protocol 客户端 |
| ACP 翻译器     | `src/acp/translator.ts`                   | 1,100 | ACP 消息格式翻译                       |
| ACP 会话      | `src/acp/session.ts`                      | 190   | ACP 会话管理                         |
| ACP 策略      | `src/acp/policy.ts`                       | 70    | ACP 访问控制策略                       |


#### 关键设计思想

**1. 子 Agent 注册表（Supervisor-Worker 模式）**

注册表是多 Agent 编排的核心（1,705 行），实现了：

```
Parent Agent（Supervisor）
├── 注册子 Agent（subagent-spawn）
├── 管理子 Agent 状态（SubagentRunRecord）
├── 监听子 Agent 完成/超时/失败
├── 接收子 Agent 执行结果（subagent-announce）
└── 清理资源（subagent-registry-cleanup）
```

关键类型 `SubagentRunRecord` 记录：

- 运行 ID、会话 Key、请求者信息
- 运行状态、超时设置
- 父-子关系链（支持嵌套子 Agent）

**2. 深度限制（subagent-depth.ts）**

防止子 Agent 无限递归生成更多子 Agent，类似操作系统的进程树深度限制。

**3. 孤儿恢复（subagent-orphan-recovery.ts）**

当父 Agent 异常终止时，其子 Agent 会变成"孤儿"。孤儿恢复机制确保这些子 Agent 被正确清理或重新关联。

**4. ACP（Agent Communication Protocol）**

ACP 是 OpenClaw 内部的 Agent 间通信协议：

- `client.ts` — 发送消息到其他 Agent
- `translator.ts`（1,100 行）— 消息格式翻译（不同 Agent 可能使用不同的消息格式）
- `session.ts` — 跨 Agent 的会话管理
- `policy.ts` — 通信权限控制

#### 推荐阅读路径

1. `subagent-lifecycle-events.ts` — 理解 Agent 生命周期事件定义
2. `subagent-spawn.ts` — 理解子 Agent 如何被创建
3. `subagent-registry.ts` — 重点文件，理解完整的注册表机制
4. `subagent-control.ts` — 理解如何控制运行中的子 Agent
5. `subagent-announce.ts` — 理解结果如何回传给父 Agent
6. `subagent-depth.ts` — 理解深度限制的设计
7. `acp/client.ts` → `acp/translator.ts` — 理解 Agent 间通信协议

#### 动手实践

- **任务 A**：阅读 `subagent-registry.ts`，画出 `SubagentRunRecord` 的状态机（pending → running → completed/failed/timeout）
- **任务 B**：阅读 `subagent-depth.ts`，设计一个测试场景验证深度限制是否生效
- **任务 C**：阅读 `acp/client.ts` + `acp/translator.ts`，画出两个 Agent 之间通信的消息流转图

#### 面试应用

- "设计一个多 Agent 系统" — 展示 Supervisor-Worker 模式的注册表实现
- "如何管理 Agent 的生命周期" — 展示 spawn → run → announce → cleanup 完整链路
- "如何处理 Agent 无限递归" — 展示深度限制机制
- "Agent 之间如何通信" — 展示 ACP 协议设计

---

### 能力 7: 可观测性与评测

**要掌握的核心能力：** 全链路日志追踪、Token 成本监控、缓存命中分析、诊断系统。

**JOB MODEL 考点：** LLM 可观测性、Token 使用分析、日志/追踪/指标、自动化评测。

#### 核心模块


| 模块          | 路径                                         | 行数    | 学什么                  |
| ----------- | ------------------------------------------ | ----- | -------------------- |
| Usage 统计    | `src/agents/usage.ts`                      | 190   | Token 使用量归一化、成本计算    |
| 缓存追踪        | `src/agents/cache-trace.ts`                | 260   | 完整的缓存追踪系统（7 阶段）      |
| 模型降级观测      | `src/agents/model-fallback-observation.ts` | —     | 降级决策的日志记录            |
| Provider 归因 | `src/agents/provider-attribution.ts`       | —     | 追踪每次调用使用了哪个 Provider |
| 子系统日志       | `src/logging/subsystem.ts`                 | 425   | 带子系统标签的结构化日志         |
| 诊断系统        | `src/logging/diagnostic.ts`                | 433   | 运行时诊断信息收集            |
| 日志脱敏        | `src/logging/redact.ts`                    | —     | 日志中的敏感信息脱敏           |
| Payload 脱敏  | `src/agents/payload-redaction.ts`          | —     | API 请求/响应中的敏感信息脱敏    |
| 安全审计日志      | `src/security/audit.ts`                    | 1,318 | 安全事件的完整审计链           |
| 工具调用显示      | `src/agents/tool-display.ts`               | —     | 工具调用的人类可读格式化         |
| Agent 追踪基础  | `src/agents/trace-base.ts`                 | —     | Agent 运行追踪的基础元数据     |
| 会话转录策略      | `src/agents/transcript-policy.ts`          | —     | 控制哪些内容写入转录日志         |


#### 关键设计思想

**1. 缓存追踪（cache-trace.ts）— 7 阶段追踪**

每次 Agent 运行都会记录 7 个阶段的完整追踪：

```
session:loaded    → 原始会话加载
session:sanitized → 消息清洗后
session:limited   → 历史截断后
prompt:before     → 发送前的完整 Payload
prompt:images     → 图片处理阶段
stream:context    → 流式输出上下文
session:after     → 运行结束后的会话状态
```

每个阶段记录：时间戳、序号、运行 ID、会话 Key、Provider、Model、消息内容等。这是生产级 Agent 系统排查问题的关键基础设施。

**2. Usage 归一化（usage.ts）**

不同 Provider 返回的 Token 使用量字段名不同：

- OpenAI：`prompt_tokens` / `completion_tokens`
- Anthropic：`input_tokens` / `output_tokens`
- 其他：各种变体

`usage.ts` 将所有格式归一化为统一的 `NormalizedUsage`，并计算成本（含 cache read/write）。

**3. 分层日志系统**

```
subsystem.ts → 创建带标签的日志器（如 "agents/subagent-registry"）
diagnostic.ts → 收集运行时诊断信息
redact.ts → 脱敏处理
audit.ts → 安全审计事件
```

#### 推荐阅读路径

1. `usage.ts` — 理解 Token 使用量归一化和成本计算
2. `cache-trace.ts` — 理解全链路缓存追踪系统
3. `logging/subsystem.ts` — 理解结构化日志设计
4. `logging/diagnostic.ts` — 理解运行时诊断
5. `logging/redact.ts` — 理解日志脱敏
6. `provider-attribution.ts` — 理解 Provider 归因追踪
7. `security/audit.ts` — 理解安全审计日志

#### Agent 评测（Eval）— 补充能力项

**为什么重要：** 2026 年行业越来越重视 Agent 系统的可评测性。"不能衡量就不能改进"——Agent 评测是从实验到生产的关键门槛。

**项目中的评测实践：**

OpenClaw 在 `pi-embedded-runner/` 目录下有 **31 个测试文件**，覆盖了 Agent 系统评测的多个维度：


| 评测维度     | 测试文件示例                                          | 学什么               |
| -------- | ----------------------------------------------- | ----------------- |
| 模型行为回归   | `model.forward-compat.test.ts`                  | 模型 API 变更后的兼容性测试  |
| 工具调用正确性  | `tool-result-truncation.test.ts`                | 工具结果截断是否影响正确性     |
| 上下文溢出恢复  | `run.overflow-compaction.test.ts`               | Compaction 后行为是否正常 |
| 流式输出一致性  | `proxy-stream-wrappers.test.ts`                 | 多 Provider 流式归一化验证 |
| 子 Agent 编排 | `sessions-yield.orchestration.test.ts`          | 多 Agent 协作的端到端验证  |
| 系统提示词正确性 | `system-prompt.test.ts`                         | Prompt 变更的回归测试    |
| 成本统计准确性  | `usage-reporting.test.ts`                       | Token 计量和成本归因验证   |


**评测方法论总结：**

1. **快照测试（Snapshot Testing）**：通过 `.test-harness.ts` 和 `.fixture.ts` 文件保存 Agent 行为的预期输出快照
2. **集成测试**：`skills-runtime.integration.test.ts` 验证端到端的工具 + 模型协作
3. **边界条件测试**：`run.overflow-compaction.loop.test.ts` 测试反复 compaction 的极端场景
4. **多 Provider 对比**：`google.test.ts`、`kilocode.test.ts` 等确保不同模型后端行为一致

#### 动手实践

- **任务 A**：选择 `cache-trace.ts`，追踪一次完整的 7 阶段缓存记录，用 JSON 格式记录每个阶段的数据
- **任务 B**：阅读 `usage-reporting.test.ts`，理解 Token 使用量的归一化测试方法
- **任务 C**：为 `run.overflow-compaction.test.ts` 增加一个新的边界条件测试用例

#### 面试应用

- "如何监控 LLM 应用" — 展示 7 阶段缓存追踪 + Usage 归一化
- "如何计算 Token 成本" — 展示 Usage 归一化 + 成本计算
- "如何保护日志中的敏感信息" — 展示脱敏管线
- "LLM 应用的可观测性包含哪些方面" — 展示 trace + usage + audit 三层体系
- "如何评测 Agent 系统" — 展示快照测试 + 集成测试 + 多 Provider 对比的组合策略
- "Prompt 改了怎么保证不出问题" — 展示 system-prompt.test.ts 的回归测试方法

---

### 能力 8: 安全与合规

**要掌握的核心能力：** 输入校验、输出过滤、沙箱隔离、权限控制、审计追踪。

**JOB MODEL 考点：** Prompt Injection 防护、工具执行安全、代码沙箱、SSRF 防护、权限模型。

#### 核心模块


| 模块        | 路径                                               | 行数    | 学什么                        |
| --------- | ------------------------------------------------ | ----- | -------------------------- |
| 沙箱系统      | `src/agents/sandbox/` 目录                         | —     | Docker/SSH 沙箱隔离 Agent 工具执行 |
| 路径策略      | `src/agents/path-policy.ts`                      | 134   | 文件系统路径访问控制                 |
| 工具策略      | `src/agents/tool-policy.ts`                      | 210   | 工具级别的执行权限                  |
| 工具策略管线    | `src/agents/tool-policy-pipeline.ts`             | 132   | 多策略组合执行                    |
| SSRF 防护   | `src/plugin-sdk/ssrf-policy.ts`                  | 87    | 服务端请求伪造防护                  |
| 安全审计      | `src/security/audit.ts`                          | 1,318 | 完整的安全事件审计链                 |
| 安全扫描      | `src/security/skill-scanner.ts`                  | —     | Skill 文件安全扫描               |
| 危险命令检测    | `src/security/dangerous-tools.ts`                | —     | 危险工具/命令的检测与拦截              |
| 危险配置标志    | `src/security/dangerous-config-flags.ts`         | —     | 危险配置项检测                    |
| 输入白名单     | `src/gateway/input-allowlist.ts`                 | —     | 输入来源白名单控制                  |
| Auth 速率限制 | `src/gateway/auth-rate-limit.ts`                 | —     | API 认证速率限制                 |
| 提示词安全化    | `src/agents/sanitize-for-prompt.ts`              | —     | 防止 Prompt Injection 的输入清洗  |
| 图片安全化     | `src/agents/image-sanitization.ts`               | —     | 图片输入的安全处理                  |
| 密钥管理      | `src/secrets/` 目录                                | —     | 密钥安全存储与访问控制                |
| Bash 执行审批 | `src/agents/bash-tools.exec-approval-request.ts` | —     | Shell 命令执行前的审批机制           |


#### 关键设计思想

**1. 沙箱隔离架构**

OpenClaw 支持多种沙箱后端：

```
Agent 工具调用（bash/write/edit）
  → 沙箱策略检查(sandbox-tool-policy.ts)
  → 选择沙箱后端:
     ├── Docker 容器沙箱 — 完整隔离
     ├── SSH 远程沙箱 — 远程执行
     └── 本地沙箱 — 限制性执行
  → 在沙箱内执行
  → 结果返回
```

**2. 多层安全防线**

```
Layer 1: 输入层
├── input-allowlist — 来源白名单
├── auth-rate-limit — 速率限制
└── sanitize-for-prompt — 防 Prompt Injection

Layer 2: 工具执行层
├── tool-policy — 工具权限控制
├── tool-policy-pipeline — 多策略管线
├── path-policy — 文件路径访问控制
├── dangerous-tools — 危险工具拦截
└── bash-tools.exec-approval — Shell 执行审批

Layer 3: 输出/网络层
├── ssrf-policy — SSRF 防护
├── image-sanitization — 图片安全处理
└── payload-redaction — 响应脱敏

Layer 4: 审计层
└── security/audit.ts — 全量安全审计日志
```

**3. Bash 执行审批流**

Agent 执行 Shell 命令时的安全审批：

1. 分析命令内容 → 判断风险等级
2. 低风险命令（read-only）→ 自动允许
3. 高风险命令（write/delete/network）→ 需要人工审批
4. 记录审计日志

#### 推荐阅读路径

1. `tool-policy.ts` + `tool-policy-pipeline.ts` — 理解工具权限控制
2. `path-policy.ts` — 理解文件路径访问控制
3. `sandbox/` 目录 — 理解沙箱隔离设计
4. `bash-tools.exec-approval-request.ts` — 理解命令审批流
5. `ssrf-policy.ts` — 理解 SSRF 防护
6. `sanitize-for-prompt.ts` — 理解 Prompt Injection 防护
7. `security/audit.ts` — 理解审计日志系统

#### 动手实践

- **任务 A**：阅读 `tool-policy.ts` + `tool-policy-pipeline.ts`，画出多策略管线的执行流程图
- **任务 B**：阅读 `sandbox/docker.ts` 和 `sandbox/ssh.ts`，对比两种沙箱后端的隔离机制差异
- **任务 C**：阅读 `bash-tools.exec-approval-request.ts`，设计一个命令风险等级分类方案

#### 面试应用

- "如何防止 Prompt Injection" — 展示输入清洗 + 工具策略双重防线
- "Agent 执行代码如何保证安全" — 展示沙箱隔离 + 执行审批设计
- "如何设计 Agent 的权限系统" — 展示多层安全防线架构
- "SSRF 防护如何实现" — 展示 ssrf-policy.ts 的 URL 校验逻辑

---

### 能力 9: 流式工程（Streaming）

**要掌握的核心能力：** 多 Provider 流式响应归一化、SSE/WebSocket 适配、流式分段推送、流中断与恢复。

**JOB MODEL 考点：** 流式输出架构、不同 Provider 流格式差异处理、用户体验优化、流式 + 工具调用的交织处理。

> **为什么重要：** 在生产级 Agent 系统中，流式输出直接决定用户体验（首 Token 延迟、打字机效果）。不同 Provider 的流格式完全不同（OpenAI SSE、Anthropic SSE、Google 自定义格式、WebSocket 实时流），如何归一化、处理流中断、实现段落级分段推送是核心工程挑战。

#### 核心模块


| 模块             | 路径                                                        | 学什么                              |
| -------------- | --------------------------------------------------------- | -------------------------------- |
| OpenAI 流式适配    | `src/agents/pi-embedded-runner/openai-stream-wrappers.ts` | OpenAI SSE 流 + 服务分层 + 快速模式适配     |
| Anthropic 流式适配 | `src/agents/pi-embedded-runner/anthropic-stream-wrappers.ts` | Anthropic SSE 流 + Beta 头 + 缓存保留   |
| Google 流式适配    | `src/agents/pi-embedded-runner/google-stream-wrappers.ts` | Google Gemini 流格式适配               |
| Moonshot 流式适配  | `src/agents/pi-embedded-runner/moonshot-stream-wrappers.ts` | Moonshot/SiliconFlow Thinking 适配 |
| 代理流式适配         | `src/agents/pi-embedded-runner/proxy-stream-wrappers.ts`  | OpenAI 兼容代理的流式处理                 |
| WebSocket 实时流  | `src/agents/openai-ws-stream.ts`                          | WebSocket 双向实时流（Realtime API）     |
| 流式订阅中枢         | `src/agents/pi-embedded-subscribe.ts`                     | 726 行，流式事件订阅、分段推送、工具事件处理         |
| 原始流处理          | `src/agents/pi-embedded-subscribe.raw-stream.ts`          | 底层原始流事件处理                        |
| 流消息共享          | `src/agents/stream-message-shared.ts`                     | 跨组件的流消息类型定义                      |
| 流式参数解析         | `src/agents/pi-embedded-runner/extra-params.ts`           | Provider 特定的流参数（temperature 等）解析 |
| Ollama 流       | `src/agents/ollama-stream.ts`                             | 本地 Ollama 模型的流式处理                |


#### 关键设计思想

**1. 多 Provider 流式归一化**

每个 Provider 返回的流格式不同，项目通过 `*-stream-wrappers.ts` 模式实现统一：

```
streamSimple（统一流接口）
  → resolveExtraParams（解析 Provider 特定参数）
  → createXxxWrapper（Provider 特定的流包装器）
  → wrapProviderStreamFn（插件级流包装）
  → 归一化的流事件输出
```

**2. 流包装器的组合模式（extra-params.ts）**

`extra-params.ts` 是流式参数解析的中枢，它将多个流包装器按 Provider 组合：

- OpenAI：`ServiceTierWrapper` + `FastModeWrapper` + `ResponsesContextManagementWrapper`
- Anthropic：`BetaHeadersWrapper` + `FastModeWrapper` + `ToolPayloadCompatibilityWrapper`
- Moonshot：`ThinkingWrapper`

**3. 流式分段推送（EmbeddedBlockChunker）**

`pi-embedded-subscribe.ts` 中实现了段落级分段推送（soft chunking），而非逐 Token 推送：

- 积累文本直到遇到段落边界（换行、标点等）
- 整段推送给通道，减少消息碎片
- 对不同通道适配不同的分段策略（Telegram 有消息长度限制，Discord 有 Embed 限制等）

**4. 流式 + 工具调用的交织**

Agent 运行时需要处理流式文本输出和工具调用交替出现的场景：

```
[流式文本] "让我查看一下..." → [工具调用] bash("ls") → [工具结果] → [流式文本] "文件列表是..."
```

`pi-embedded-subscribe.ts` 负责将这些交织事件正确路由给通道回复管线。

#### 推荐阅读路径

1. `extra-params.ts` — 入口文件，理解多 Provider 流参数如何解析和组合
2. `openai-stream-wrappers.ts` — 理解最通用的 OpenAI 流适配
3. `anthropic-stream-wrappers.ts` — 对比 Anthropic 的差异（Beta 头、缓存保留等）
4. `google-stream-wrappers.ts` — 理解 Google 流的特殊处理
5. `pi-embedded-subscribe.ts` — 重点文件，理解流事件订阅和分段推送
6. `openai-ws-stream.ts` — 理解 WebSocket 实时流的双向通信

#### 动手实践

- **任务 A**：对比 `openai-stream-wrappers.ts` 和 `anthropic-stream-wrappers.ts`，列出两者的关键差异
- **任务 B**：追踪一个流式 Token 从 Provider 返回到用户通道的完整路径，画出数据流图
- **任务 C**：阅读 `proxy-stream-wrappers.test.ts`，理解如何测试流式输出的正确性

#### 面试应用

- "流式输出怎么实现的" — 展示多 Provider 流归一化 + 分段推送
- "不同 LLM Provider 的流格式有什么区别" — 展示 OpenAI/Anthropic/Google 三种 stream wrapper 的差异
- "流式输出和工具调用如何交替处理" — 展示 `pi-embedded-subscribe.ts` 的交织事件处理
- "如何优化首 Token 延迟" — 展示流式参数（service_tier、fast mode）和连接复用

---

### 能力 10: 多模态与新兴能力

**要掌握的核心能力：** Extended Thinking（扩展推理）、图片/媒体输入处理、Canvas 交互式工件。

**JOB MODEL 考点：** 推理模型使用、多模态输入处理、交互式输出、新模型能力适配。

> **为什么重要：** 2026 年 LLM 领域的三大趋势——推理模型（o1/o3/Claude thinking）、多模态（Vision/Audio）、交互式工件（Artifacts/Canvas）——都需要 Agent 系统做相应的工程适配。

#### 核心模块


| 模块           | 路径                                               | 学什么                               |
| ------------ | ------------------------------------------------ | --------------------------------- |
| Thinking 块处理 | `src/agents/pi-embedded-runner/thinking.ts`      | 推理模型 thinking blocks 的剥离与保留策略     |
| 图片安全处理       | `src/agents/image-sanitization.ts`               | 图片输入的安全校验（大小、格式、恶意内容）             |
| Agent 运行时图片  | `src/agents/pi-embedded-runner/run/images.ts`    | Agent 运行中的图片上下文管理                 |
| 图片历史裁剪       | `pi-embedded-runner/run/history-image-prune.test.ts` | 历史消息中图片的智能裁剪（节省 Token）            |
| 工具图片处理       | `src/agents/tool-images.ts`                      | 工具执行产生图片（截图等）的处理管线                |
| 媒体管线         | `src/media/`                                     | 完整的媒体处理管线（转码、压缩、存储）              |
| Canvas 服务器   | `src/canvas-host/server.ts`                      | 交互式工件（Canvas/A2UI）的宿主服务器          |
| Canvas 文件解析  | `src/canvas-host/file-resolver.ts`               | Canvas 文件的解析和加载                   |
| 推理标签检测       | `src/utils/provider-utils.ts`（isReasoningTagProvider） | 检测 Provider 是否使用推理标签（`<thinking>`） |
| 模型能力检测       | `src/agents/model-tool-support.ts`               | 检测模型是否支持特定能力（工具调用、Vision 等）       |


#### 关键设计思想

**1. Extended Thinking 的工程处理**

推理模型（Claude 3.5 Opus、OpenAI o3 等）会在输出中包含 `thinking` 内容块。项目的处理策略：

```
模型输出 → 包含 thinking blocks
  → 实时：推理过程可选展示给用户（内部 UI）
  → 存储：thinking blocks 保留在 session 中用于上下文连续性
  → 传递：发送给下一轮模型时需要剥离 thinking blocks（某些 Provider 不接受）
  → 通道：外部通道（Telegram 等）只发送最终回复，不发 thinking 内容
```

`thinking.ts` 的 `dropThinkingBlocks()` 函数在保留 turn 结构的同时剥离推理块。

**2. 图片输入的多层处理**

```
用户发送图片 → image-sanitization.ts（安全校验 + 大小限制）
  → run/images.ts（嵌入到消息上下文）
  → history-image-prune（历史轮次图片裁剪，节省 Token）
  → 模型调用（Vision 模式）
```

关键约束：`MAX_IMAGE_BYTES` 限制单张图片大小；历史消息中的图片会被智能裁剪（只保留最近 N 轮的图片）。

**3. Canvas/A2UI 交互式工件**

`src/canvas-host/` 实现了类似 Claude Artifacts 的交互式工件宿主：

- `server.ts` — 启动本地服务器托管 Canvas 内容
- `file-resolver.ts` — 解析 Agent 生成的 Canvas 文件
- `a2ui.ts` — A2UI（Agent-to-UI）协议的实现

#### 推荐阅读路径

1. `pi-embedded-runner/thinking.ts` — 理解 thinking blocks 的处理逻辑
2. `image-sanitization.ts` — 理解图片输入的安全处理
3. `pi-embedded-runner/run/images.ts` — 理解图片如何嵌入 Agent 上下文
4. `tool-images.ts` — 理解工具产生的图片（截图等）如何处理
5. `model-tool-support.ts` — 理解模型能力检测
6. `canvas-host/server.ts` — 理解交互式工件的宿主架构

#### 动手实践

- **任务 A**：阅读 `thinking.ts`，理解 `dropThinkingBlocks()` 如何在剥离推理块的同时保留 turn 结构
- **任务 B**：追踪一张图片从用户发送到进入模型上下文的完整路径
- **任务 C**：阅读 `canvas-host/server.ts`，理解 Canvas 服务器如何将 Agent 输出渲染为可交互界面

#### 面试应用

- "推理模型（o3/Claude thinking）的输出怎么处理" — 展示 thinking blocks 的剥离/保留/传递策略
- "Agent 怎么处理图片输入" — 展示多层图片处理管线 + 历史图片裁剪
- "如何支持新的模型能力" — 展示 model-tool-support.ts 的能力检测模式
- "什么是 Artifacts/Canvas" — 展示 A2UI 的宿主服务器设计

---

### 能力 11: 成本优化工程

**要掌握的核心能力：** Prompt Caching、Cache TTL 管理、Token 预算控制、模型选型策略。

**JOB MODEL 考点：** LLM 调用成本优化、缓存策略、模型降级与成本的平衡、Token 效率。

> **为什么重要：** 生产级 Agent 系统的 LLM 调用成本往往是最大的运营开支。Prompt Caching 可以节省 60-90% 的 System Prompt Token 成本；合理的模型选型和降级策略可以在保证质量的同时大幅降低成本。

#### 核心模块


| 模块              | 路径                                                         | 学什么                               |
| --------------- | ---------------------------------------------------------- | --------------------------------- |
| Bootstrap 缓存    | `src/agents/bootstrap-cache.ts`                            | Session 级别的 Bootstrap 文件缓存，避免重复加载 |
| Cache TTL 管理    | `src/agents/pi-embedded-runner/cache-ttl.ts`               | Provider 级缓存 TTL 资格判定与时间戳管理       |
| 缓存保留策略          | `pi-embedded-runner/anthropic-stream-wrappers.ts`（resolveCacheRetention） | Anthropic 缓存保留控制                  |
| OpenRouter 缓存控制 | `pi-embedded-runner/extra-params.openrouter-cache-control.test.ts` | OpenRouter 特定的缓存控制                |
| Token 预算管理      | `src/agents/bootstrap-budget.ts`                           | System Prompt + 工具定义的 Token 占用预算  |
| 工具结果截断          | `src/agents/pi-embedded-runner/tool-result-truncation.ts`  | 过长的工具结果智能截断，节省 Token              |
| 工具结果上下文守卫       | `src/agents/pi-embedded-runner/tool-result-context-guard.ts` | 工具结果的上下文窗口守卫                      |
| 模型降级 + 成本       | `src/agents/model-fallback.ts`                             | 降级到更便宜的模型以控制成本                    |
| 模型选择            | `src/agents/model-selection.ts`                            | 根据场景选择合适（成本效率最优）的模型               |
| Usage 统计        | `src/agents/usage.ts`                                      | Token 使用量归一化和成本计算                 |


#### 关键设计思想

**1. Prompt Caching（缓存保留）**

Anthropic 和 OpenAI 都支持 Prompt Caching——如果连续请求的 System Prompt 前缀相同，后续请求只需支付极少的缓存读取费用。

```
首次请求：System Prompt（1000 tokens）→ 全价计费 → 缓存写入
后续请求：System Prompt（1000 tokens）→ 缓存命中 → 仅计 10% 费用

节省率 = 90%（对于稳定的 System Prompt）
```

`resolveCacheRetention` 控制 Anthropic 的 `cache_control` 参数，`cache-ttl.ts` 管理缓存过期策略。

**2. 工具结果截断（tool-result-truncation.ts）**

Agent 调用工具（如 bash 命令）可能返回巨量文本。项目实现了智能截断：

- 估算字符数 → Token 数的映射
- 超过阈值时截断并附加说明
- 保留头尾关键信息（错误信息通常在末尾）

**3. 多层成本控制策略**

```
Layer 1: 缓存层
├── bootstrap-cache — 避免重复加载 Bootstrap 文件
├── Prompt Caching — 减少重复 System Prompt Token 费用
└── cache-ttl — 管理缓存生命周期

Layer 2: 裁剪层
├── tool-result-truncation — 截断过长工具输出
├── tool-result-context-guard — 守卫工具结果的上下文预算
├── compaction — 压缩过长对话历史
└── history-image-prune — 裁剪历史图片

Layer 3: 模型层
├── model-selection — 选择性价比最优的模型
├── model-fallback — 降级到更便宜的模型
└── PromptMode.minimal — 子 Agent 使用最小 System Prompt
```

#### 推荐阅读路径

1. `usage.ts` — 理解 Token 成本如何计算（含缓存读/写价格）
2. `bootstrap-cache.ts` — 理解 Session 级缓存
3. `cache-ttl.ts` — 理解 Provider 级缓存 TTL 管理
4. `anthropic-stream-wrappers.ts`（`resolveCacheRetention`）— 理解 Anthropic Prompt Caching
5. `tool-result-truncation.ts` — 理解工具结果的智能截断
6. `bootstrap-budget.ts` — 理解 Token 预算分配

#### 动手实践

- **任务 A**：阅读 `usage.ts`，整理一张表：各 Provider 的 Token 价格字段名称差异
- **任务 B**：阅读 `cache-ttl.ts`，画出缓存 TTL 判定流程图
- **任务 C**：计算一个典型 Agent 会话的成本（假设 System Prompt 1000 tokens、5 轮对话、3 次工具调用），对比启用/不启用 Prompt Caching 的成本差异

#### 面试应用

- "如何优化 LLM 调用成本" — 展示三层成本控制策略（缓存 + 裁剪 + 模型选型）
- "什么是 Prompt Caching" — 展示 Anthropic/OpenAI 缓存保留机制和节省率
- "工具结果太长怎么办" — 展示 tool-result-truncation 的智能截断策略
- "如何平衡质量和成本" — 展示 model-selection + model-fallback 的组合策略

---

## 四、插件/扩展系统（横切能力）

理解插件系统是贯穿所有能力项的横切关注点。

#### 核心模块


| 模块            | 路径                                             | 学什么          |
| ------------- | ---------------------------------------------- | ------------ |
| Plugin SDK    | `src/plugin-sdk/`                              | 插件的公共 API 面  |
| 插件加载          | `src/plugins/`                                 | 插件发现、加载、注册   |
| 通道插件运行时       | `src/plugin-sdk/channel-runtime.ts`            | 通道插件的运行时抽象   |
| Agent 运行时 SDK | `src/plugin-sdk/agent-runtime.ts`              | Agent 插件 SDK |
| 扩展示例          | `extensions/openai/`, `extensions/telegram/` 等 | 真实插件实现参考     |


#### 关键设计思想

- **最小 API 面原则**：插件只能通过 `openclaw/plugin-sdk/`* 访问核心功能
- **依赖隔离**：插件的 `dependencies` 独立管理，不污染核心
- **运行时解析**：`openclaw/plugin-sdk` 路径在运行时通过 jiti alias 解析

---

## 五、Python 生态映射（面试与工程必备）

> **背景：** 2026 年 AI Agent 工程师岗位 90%+ 要求 Python 技能。通过 OpenClaw（TypeScript）学习概念后，需要将每个概念对应到 Python 主流框架，才能在面试和实际工作中无缝切换。

### 概念 → Python 框架对照表


| OpenClaw 概念                        | Python 对应框架/库                              | 说明                        |
| ---------------------------------- | ------------------------------------------ | ------------------------- |
| Agent Runner（pi-embedded-runner）   | LangGraph Agent / CrewAI Agent / AutoGen   | Agent 执行循环                |
| Tool Catalog + Schema              | LangChain Tools / OpenAI Function Calling  | 工具定义与注册                   |
| MCP 集成（mcp-stdio.ts）              | MCP Python SDK (`mcp`)                     | 官方 Python MCP 客户端         |
| System Prompt 组装（system-prompt.ts） | LangChain PromptTemplate / Jinja2 模板       | 提示词模板化                    |
| Compaction（compaction.ts）          | LangChain ConversationSummaryBufferMemory  | 对话摘要压缩                    |
| Context Engine（context-engine/）    | LlamaIndex IngestionPipeline               | 上下文摄入/组装/压缩管线             |
| Model Fallback（model-fallback.ts） | LiteLLM Router / fallback 配置               | 多模型路由与降级                  |
| RAG Pipeline（memory/）              | LlamaIndex / LangChain RAG                 | 检索增强生成                    |
| 向量存储（sqlite-vec.ts）               | ChromaDB / FAISS / Pinecone / Qdrant       | 向量数据库                     |
| 混合检索（hybrid.ts）                   | LlamaIndex HybridRetriever                 | Dense + Sparse 混合检索       |
| Multi-Agent（subagent-*）            | CrewAI / AutoGen / LangGraph Multi-Agent   | 多 Agent 编排                |
| ACP 协议（acp/）                      | Google A2A Protocol / LangGraph Send API   | Agent 间通信                 |
| Plugin SDK（plugin-sdk/）            | LangChain Tool/Retriever 接口                | 可扩展插件架构                   |
| 流式输出（stream-wrappers）             | LangChain Streaming / OpenAI stream=True   | 流式响应处理                    |
| Token 计算（usage.ts）                | tiktoken / LiteLLM cost tracking           | Token 计量与成本计算             |
| Prompt Caching                     | Anthropic `cache_control` / OpenAI caching | Python SDK 原生支持           |
| 沙箱隔离（sandbox/）                    | Docker SDK / E2B / Modal                   | 代码执行沙箱                    |


### 建议的 Python 实践项目

学完 OpenClaw 的每个能力项后，尝试用 Python 实现一个简化版：

1. **LangGraph Agent**：实现一个带工具调用和降级策略的 Agent
2. **RAG Pipeline**：用 LlamaIndex 实现 Dense + Sparse 混合检索 + MMR 重排序
3. **MCP Server**：用 Python MCP SDK 实现一个自定义工具服务器
4. **Multi-Agent**：用 CrewAI 实现 Supervisor-Worker 模式的任务分解
5. **Streaming App**：用 FastAPI + SSE 实现一个支持流式输出的 Agent API

---

## 六、学习进阶路线图

### 第一阶段：架构认知（2-3 周）

- 跑通项目：`pnpm install` → `pnpm build` → `pnpm test`
- 阅读能力 1 全部推荐文件，画出系统架构图
- 追踪一条消息从 Gateway 到 Agent Response 的完整链路
- 完成能力 1 的动手实践任务
- **输出**：一份架构分析文档 + 架构图

### 第二阶段：核心引擎（3-4 周）

- 深读能力 2（Tool Calling）：理解工具目录 → Schema → 执行管线 → MCP
- 深读能力 3（Context Engineering）：理解上下文引擎 + 系统提示词组装 + Compaction
- 深读能力 5（可靠性）：理解模型降级链 + Auth 轮转
- 深读能力 9（流式工程）：理解多 Provider 流归一化 + 分段推送
- 完成各能力的动手实践任务
- **输出**：能独立设计一个带工具调用、流式输出和降级能力的 Agent 系统

### 第三阶段：高级能力（3-4 周）

- 深读能力 4（RAG）：从分块到检索的完整管线
- 深读能力 6（多 Agent）：子 Agent 编排 + ACP 协议
- 深读能力 10（多模态与新兴能力）：Thinking/Image/Canvas
- 完成各能力的动手实践任务
- **输出**：能设计多 Agent + RAG + 多模态的复合系统

### 第四阶段：工程化（2-3 周）

- 深读能力 7（可观测性与评测）：追踪 + Usage + 审计 + Agent 评测
- 深读能力 8（安全）：沙箱 + 策略 + 审批
- 深读能力 11（成本优化）：Prompt Caching + Cache TTL + 裁剪策略
- 完成各能力的动手实践任务
- **输出**：能设计生产级 Agent 系统的安全、监控和成本控制方案

### 第五阶段：Python 生态迁移（2 周）

- 对照"Python 生态映射"章节，用 Python 实现每个能力项的简化版
- 完成至少 3 个 Python 实践项目（LangGraph Agent、RAG Pipeline、MCP Server 等）
- **输出**：一套 Python 版 Agent 系统 demo + 对比分析报告

### 第六阶段：实战与面试（持续）

- 为 OpenClaw 项目贡献代码（fix bug / add feature）
- 基于所学设计并实现一个自己的 Agent 系统
- 准备面试：每个能力项准备 3-5 个面试回答（参考面试速查表）
- 定期复习：关注行业新动态，更新知识体系

---

## 七、面试速查表


| 面试问题                          | 对应能力 | 关键文件                                                        |
| ----------------------------- | ---- | ----------------------------------------------------------- |
| 画一个 AI Agent 系统架构图            | 1    | `server.impl.ts`, `pi-embedded-runner/`                     |
| Function Calling 怎么实现的        | 2    | `tool-catalog.ts`, `pi-tools.schema.ts`                     |
| MCP 协议是什么                     | 2    | `mcp-stdio.ts`, `pi-bundle-mcp-tools.ts`                    |
| 什么是 Context Engineering       | 3    | `context-engine/`, `system-prompt.ts`                       |
| System Prompt 怎么设计            | 3    | `system-prompt.ts`                                          |
| 上下文窗口不够怎么办                    | 3    | `compaction.ts`, `context-window-guard.ts`                  |
| RAG 系统怎么设计                    | 4    | `memory/` 目录全部                                              |
| 模型 API 挂了怎么办                  | 5    | `model-fallback.ts`                                         |
| 多个 API Key 怎么管理               | 5    | `auth-profiles.ts`                                          |
| 多 Agent 系统怎么设计                | 6    | `subagent-registry.ts`, `subagent-spawn.ts`                 |
| Agent 间怎么通信                   | 6    | `acp/` 目录                                                   |
| 如何监控 LLM 应用                   | 7    | `cache-trace.ts`, `usage.ts`                                |
| 如何评测 Agent 系统                 | 7    | `pi-embedded-runner/*.test.ts`                              |
| 如何防止 Prompt Injection         | 8    | `sanitize-for-prompt.ts`, `tool-policy.ts`                  |
| Agent 执行代码安全吗                 | 8    | `sandbox/`, `bash-tools.exec-approval-request.ts`           |
| 流式输出怎么实现的                     | 9    | `*-stream-wrappers.ts`, `pi-embedded-subscribe.ts`          |
| 不同 Provider 流格式有什么区别          | 9    | `openai-stream-wrappers.ts`, `anthropic-stream-wrappers.ts` |
| 推理模型（o3/Claude thinking）怎么处理 | 10   | `pi-embedded-runner/thinking.ts`                            |
| Agent 怎么处理图片输入                | 10   | `image-sanitization.ts`, `run/images.ts`                    |
| 如何优化 LLM 调用成本                 | 11   | `cache-ttl.ts`, `tool-result-truncation.ts`, `usage.ts`     |
| 什么是 Prompt Caching            | 11   | `anthropic-stream-wrappers.ts`（resolveCacheRetention）       |


---

## 八、核心代码统计


| 模块                              | 文件数(估) | 核心代码行数   | 重要性   | 对应能力         |
| ------------------------------- | ------ | -------- | ----- | ------------ |
| Agent 运行时 (`src/agents/`)       | ~596   | ~50,000+ | ★★★★★ | 1,2,3,5,9,10,11 |
| Gateway (`src/gateway/`)        | ~200   | ~15,000+ | ★★★★☆ | 1,8           |
| 记忆系统 (`src/memory/`)            | ~100   | ~8,000+  | ★★★★☆ | 4             |
| Plugin SDK (`src/plugin-sdk/`)  | ~195   | ~10,000+ | ★★★☆☆ | 横切           |
| 安全 (`src/security/`)            | ~30    | ~3,000+  | ★★★☆☆ | 8             |
| ACP (`src/acp/`)                | ~30    | ~3,000+  | ★★★☆☆ | 6             |
| 路由 (`src/routing/`)             | ~11    | ~1,000+  | ★★☆☆☆ | 1             |
| 基础设施 (`src/infra/`)             | ~200   | ~15,000+ | ★★★☆☆ | 5             |
| 上下文引擎 (`src/context-engine/`)   | 7      | ~500+    | ★★★★☆ | 3             |
| 浏览器自动化 (`src/browser/`)         | ~132   | ~10,000+ | ★★★☆☆ | 2（工具执行）      |
| Canvas 宿主 (`src/canvas-host/`)  | 5      | ~500+    | ★★☆☆☆ | 10            |
| 媒体管线 (`src/media/`)             | —      | ~2,000+  | ★★☆☆☆ | 10            |
| 扩展 (`extensions/`)              | 70+ 包  | ~50,000+ | ★★★☆☆ | 横切           |


