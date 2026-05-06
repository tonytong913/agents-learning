AI Agent 工程师 Learning Path — cc-haha 项
目
AI Agent 工程师 Learning Path — 基于 cchaha 项目
目标读者：正在转型大模型/AI Agent 工程师的开发者 学习方式：通过精读本项
目（一个类 Claude Code 的 AI Agent CLI + Desktop 产品）的核心模块，掌握
2026 年 AI Agent 工程师 JOB MODEL 所要求的 14 大能力项 预计周期：16-18 周
（每周投入 10-15 小时） 最后更新：2026-05-02 | 版本：v2.0

0. 项目全局架构
0.1 这是一个什么项目
cc-haha 是一个 AI Agent 产品，包含三个层次：
• CLI 终端应用（Bun + React/Ink TUI）— 用户在终端与 AI 对话，AI 可以读写文
件、执行命令、搜索代码
• 本地 HTTP/WebSocket 服务（Bun.serve）— 为桌面应用提供 API 和实时通信
• 桌面应用（React + Vite + Tauri 2）— 图形化界面，通过 WebSocket 连接本地
服务

0.2 技术栈
层级
CLI
服务端

技术

入口文件

Bun, TypeScript, React

bin/claude-haha → src/

19, Ink 6

entrypoints/cli.tsx

Bun.serve, WebSocket

src/server/index.ts

层级
桌面端

LLM API

MCP

技术
React 18, Vite, Tauri 2,
Zustand
@anthropic-ai/sdk , SSE

streaming
@modelcontextprotocol/
sdk

入口文件
desktop/src/main.tsx

src/services/api/claude.ts

src/services/mcp/client.ts

0.3 架构全景与能力项分布
graph TB
subgraph UserLayer [用户交互层]
CLI["CLI TUI (Ink)"]
Desktop["Desktop App (Tauri)"]
IMAdapters["IM 适配器<br/>adapters/feishu/ telegram/"]
end
subgraph CoreEngine [Agent 核心引擎]
QueryLoop["Agent Loop<br/>src/query.ts"]
QueryEngine["QueryEngine<br/>src/QueryEngine.ts"]
ToolSystem["Tool 系统<br/>src/Tool.ts + src/tools/"]
Coordinator["Coordinator 编排<br/>src/coordinator/"]
end
subgraph LLMLayer [LLM 调用层]
Claude["Claude API Client<br/>src/services/api/claude.ts"]
Proxy["协议转换代理<br/>src/server/proxy/"]
Streaming["SSE Streaming"]
TokenEst["Token Estimation"]
end
subgraph ContextLayer [上下文管理层]
Compact["Compact 压缩<br/>src/services/compact/"]
Memory["Memory 记忆<br/>src/services/SessionMemory/"]
Prompts["Prompt Engineering<br/>src/constants/prompts.ts"]
end
subgraph IntegrationLayer [集成层]
MCP["MCP Client/Server<br/>src/services/mcp/"]
MultiAgent["Multi-Agent<br/>src/tools/AgentTool/"]
ComputerUse["Computer Use<br/>src/vendor/computer-use-mcp/"]
WSServer["WebSocket Server<br/>src/server/ws/"]
end

subgraph SafetyLayer [安全层]
Permissions["权限系统<br/>src/utils/permissions/"]
Worktree["Worktree 隔离<br/>src/tools/EnterWorktreeTool/"]
OAuth["OAuth 认证<br/>src/services/oauth/"]
end
subgraph OpsLayer [可观测性]
CostTracker["成本追踪<br/>src/cost-tracker.ts"]
Logging["API 日志<br/>src/services/api/logging.ts"]
VCR["VCR 录制回放<br/>src/services/vcr.ts"]
GrowthBook["Feature Flag<br/>src/services/analytics/growthbook.ts"]
end
CLI --> QueryEngine
Desktop --> WSServer
IMAdapters --> WSServer
WSServer --> QueryEngine
QueryEngine --> QueryLoop
QueryEngine --> Coordinator
Coordinator --> QueryLoop
QueryLoop --> Claude
Claude --> Proxy
QueryLoop --> ToolSystem
ToolSystem --> MCP
ToolSystem --> MultiAgent
ToolSystem --> ComputerUse
Claude --> Streaming
Claude --> TokenEst
QueryLoop --> Compact
QueryLoop --> Memory
QueryLoop --> Prompts
ToolSystem --> Permissions
ToolSystem --> Worktree
Claude --> CostTracker
Claude --> Logging
Claude --> GrowthBook

0.4 核心数据流
理解以下数据流是学习整个项目的基础：
用户输入
→ QueryEngine.processUserInput()
→ query() / queryLoop()
← Agent Loop 核心
→ while(true) {
组装 systemPrompt + messages
→ queryModel()
← 调 LLM API（Streaming）
← 解析 assistant message
if (包含 tool_use) {

→ runTools()
← 执行工具（需权限检查）
← tool_result 追加到 messages
continue
← 继续循环
} else {
break
← 无工具调用，结束
}
}

1. Agent Loop 设计
1.1 为什么重要
Agent Loop 是 AI Agent 最核心的设计模式，也是面试必考第一题。它决定了 Agent
如何“思考-行动-观察”循环执行任务。2026 年几乎所有 AI Agent 岗位都要求深入理
解 ReAct 模式的工程实现。

1.2 代码阅读指引
入口函数和阅读顺序：
顺序
1

文件

关注点

src/query.ts → query()

入口函数，注意它是 async

(第 219 行)

function* 生成器

src/query.ts →

2

queryLoop() (第 241

行)， while (true) 在第

核心循环

307 行

3

4

src/query.ts → State 类

型 (第 204 行)

turnCount,
autoCompactTracking

src/QueryEngine.ts →

上层封装：用户输入处理 →

QueryEngine 类

query 调用 → 结果分发

src/services/tools/

5

跨迭代状态：messages,

toolOrchestration.ts →
runTools()

工具执行编排：只读工具并
发，写入工具串行

顺序
6

文件

关注点

src/services/tools/

流式工具执行器：边流式接收

StreamingToolExecutor.ts

LLM 输出边启动工具

带着这些问题读代码：
1. queryLoop 的 while(true) 循环在什么条件下终止？（找 return { reason: ... } 的
所有出处）
2. 当 LLM 返回 tool_use 时，工具的执行结果是如何追加到 messages 数组并回传
给下一轮 LLM 调用的？
3. AsyncGenerator （ async function* ）模式相比普通 async 函数有什么优势？为
什么 Agent Loop 选用这个模式？
关键设计模式：
• AsyncGenerator 流式管道： queryLoop 用 yield 逐条输出事件
（StreamEvent、Message），上层消费者按需拉取，天然支持背压和中断
• 不可变参数 + 可变状态分离： params （systemPrompt, canUseTool 等）在循
环外解构且不变， state （messages, turnCount）在每次迭代末尾用整体替换
state = next 更新

• 多策略恢复：循环内有 prompt-too-long 恢复、max-output-tokens 恢复、
reactive compact 等容错路径

1.3 面试准备
Q1：请描述一个 AI Agent 的核心循环是如何实现的？
回答思路：以 queryLoop() 为例说明 ReAct 模式的工程实现。核心是一个 while(true)
循环，每次迭代包含：(1) 上下文组装（autoCompact 检查、microCompact、
context collapse）；(2) 调用 LLM API 获取流式响应；(3) 解析响应中的 tool_use
块；(4) 执行工具并收集 tool_result ；(5) 将结果追加到 messages 进入下一迭代。循
环终止条件包括：无 tool_use（任务完成）、maxTurns 到达、用户中断
（AbortController）、Token 预算耗尽。
Q2：为什么用 AsyncGenerator 而不是普通递归或回调？

回答思路：三个原因 —— (1) 天然支持流式输出， yield 让每个中间事件（思考过程、
工具进度）可以被 UI 实时消费；(2) 惰性求值，消费者不拉取就不产生，避免缓冲区
溢出；(3) using 语法（Resource Management）可以在生成器退出时自动清理资源
（如 pendingMemoryPrefetch ）。
Q3：Agent Loop 中如何处理错误恢复？
回答思路：项目实现了多层恢复策略 —— (1) reactiveCompact ：当 API 返回 prompttoo-long 错误时，自动压缩对话历史后重试；(2) max_output_tokens 恢复：当输出被
截断时，注入一条 meta message 让模型继续，最多重试 3 次；(3)
FallbackTriggeredError ：当主模型不可用时切换到 fallback model；(4) context

collapse：在 compact 之前先尝试折叠上下文，以更低成本解决窗口溢出。

1.4 动手练习
目标：在本项目中添加一个自定义 Tool（如 CurrentTimeTool ，返回当前时间）
步骤：
1. 研究模板：阅读 src/tools/GlobTool/GlobTool.ts ，理解一个完整 Tool 的结构
（inputSchema、call、prompt、renderToolUseMessage 等）
2. 创建目录：在 src/tools/ 下新建 CurrentTimeTool/ 目录
3. 定义 prompt：创建 prompt.ts ，导出工具名和描述文字
4. 实现工具：创建 CurrentTimeTool.ts ，使用 buildTool() 构建，定义 Zod
inputSchema（如接收 timezone 参数）、实现 call() 方法返回时间字符串
5. 注册工具：在 src/tools.ts 的 getAllBaseTools() 中添加你的工具
6. 测试运行：用 ./bin/claude-haha 启动项目，对话中要求 “告诉我现在几点”，
观察你的工具是否被调用
验证方法：Agent 在对话中调用了你的 CurrentTime 工具并返回了正确的时间。

1.5 延伸阅读
• ReAct 论文：ReAct: Synergizing Reasoning and Acting in Language Models
(ICLR 2023) — Agent Loop 的理论基础
• Anthropic Agent 文档：Building effective agents — 官方推荐的 Agent 设计模
式
• Agentic Design Patterns：5 AI Agent Design Patterns Every Developer Needs
to Know — 包含 Loop、Sequential、Parallel 等模式

2. Tool Calling / Function Calling
2.1 为什么重要
Tool Calling 是 Agent 从“能说话”进化到“能做事”的关键。2026 年的 AI Agent 工
程师必须精通：工具的定义与 Schema 设计、Zod → JSON Schema 转换、权限控
制、并发执行策略、以及动态工具加载（defer loading）来优化 Token 开销。

2.2 代码阅读指引
顺序

1

文件

关注点

src/Tool.ts → Tool 类

完整的工具接口：

型 (第 362 行)，

inputSchema、call、

ToolUseContext (第

checkPermissions、

158 行)

isConcurrencySafe 等

src/Tool.ts →

2

buildTool() (第 783

行)， ToolDef (第 721
行)
src/tools.ts →

3

getAllBaseTools() (第

194 行)
src/utils/api.ts →

4

toolToAPISchema() (第

119 行)
5

6

{ ...TOOL_DEFAULTS, ...def } 展

开模式，了解默认值

工具注册中心：所有内置工具
在这里汇总

Zod Schema → Anthropic API
BetaToolUnion 的转换

src/tools/GlobTool/

完整示例：一个只读、并发安

GlobTool.ts

全的搜索工具

src/tools/BashTool/

src/services/tools/

7

工具构建器：

toolOrchestration.ts →
runTools() (第 19 行)

复杂示例：涉及权限、沙箱、
危险命令分类
执行编排：只读工具并发，写
入工具串行

带着这些问题读代码：
1. 一个工具从“定义”到“被 LLM 调用”经历了哪些步骤？（Schema 注册 →
API 序列化 → LLM 返回 tool_use → 权限检查 → call() 执行 → result 映射）
2. isConcurrencySafe 如何影响工具的执行策略？哪些工具是安全的，哪些不是？
3. defer_loading 和 ToolSearchTool 是如何配合工作的？为什么不把所有工具都发
给 LLM？

2.3 面试准备
Q1：如何为 AI Agent 设计一套可扩展的工具系统？
回答思路：以 Tool 接口为例，说明关键设计点：(1) 使用 Zod Schema 定义
inputSchema，自动转 JSON Schema 给 LLM；(2) buildTool() 提供默认值（failclosed：默认不并发、默认非只读），子工具只需覆盖必要字段；(3) 分离
checkPermissions 和 call ，权限判断在执行前完成；(4) 通过 isConcurrencySafe 标记

实现自动并发编排；(5) shouldDefer 支持动态加载，避免工具过多时 Token 浪费。
Q2：如何解决工具数量过多导致的 Token 浪费问题？
回答思路：项目通过 defer_loading + ToolSearchTool 解决。当工具数量超出阈值时，
非核心工具标记为 shouldDefer: true ，只把工具名发给 API（不含完整 schema）。当
LLM 需要某个延迟加载的工具时，先调用 ToolSearchTool 搜索，找到后该工具的完整
schema 才会在下一轮注入。这样初始 prompt 的 Token 开销大幅降低。
Q3：工具执行失败时 Agent 如何处理？
回答思路：多层容错 —— (1) validateInput 在调用前校验参数，失败直接返回错误信
息给 LLM；(2) call() 内部异常被捕获，转为 is_error: true 的 tool_result，LLM 可以
看到错误并重试；(3) 用户中断时， StreamingToolExecutor 为所有未完成的工具生成
合成的 tool_result，避免 tool_use/tool_result 不配对导致 API 错误。

2.4 动手练习
目标：深入理解工具执行链路，跟踪一个 GlobTool 调用的完整生命周期
步骤：
1. 在 src/tools/GlobTool/GlobTool.ts 的 call() 方法入口处添加 console.log('GlobTool
called:', input)

2. 在 src/services/tools/toolExecution.ts 的 runToolUse 函数入口处添加
console.log('runToolUse:', toolName)

3. 在 src/utils/api.ts 的 toolToAPISchema 函数入口处添加
console.log('toolToAPISchema:', tool.name)

4. 启动项目，要求 Agent “找出所有 .ts 文件”
5. 观察日志输出顺序，绘制调用链路图
6. 完成后清除所有 console.log
验证方法：你能画出完整的调用链路： getAllBaseTools() → toolToAPISchema() →
LLM 返回 tool_use → runToolUse() → checkPermissions() → GlobTool.call() →
mapToolResultToToolResultBlockParam()

2.5 延伸阅读
• Anthropic Tool Use 文档：Tool use with Claude — 官方 API 规范
• Fine-grained Tool Streaming：Fine-grained tool streaming — 工具参数流式
传输
• JSON Schema 规范：Understanding JSON Schema — 理解 Zod → JSON
Schema 转换的目标格式

2.6 结构化输出 (Structured Output)
本项目用了一种精妙的方式实现结构化输出：不依赖 response_format 参数，而是
通过 Tool 的 inputSchema 间接约束输出格式。
核心代码：
文件
src/tools/SyntheticOutputTool/
SyntheticOutputTool.ts

关注点
“合成输出工具”：模型通过“调用工
具”来输出结构化 JSON，Schema 由 Ajv
校验

src/utils/api.ts →

按模型能力决定是否启用 strict tool

modelSupportsStructuredOutputs

schema

src/QueryEngine.ts → SDK 路径中
jsonSchema 的处理

非交互模式下如何强制结构化输出

设计思路：让模型“调用一个工具”来产出结构化数据，比 response_format: json 更
灵活——可以在同一轮对话中混合自然语言和结构化输出，且兼容不支持 JSON Mode
的模型。
面试考点：Q：如何在 Agent 系统中实现可靠的结构化输出？
回答思路：两种方案对比：(1) response_format: json_schema （OpenAI 风格，强约束
但限制模型行为）；(2) Tool-as-Schema（Anthropic 风格，用 tool_use 的
inputSchema 约束，更灵活）。本项目采用方案 (2)，通过 SyntheticOutputTool 实现

——模型“调用”一个特殊工具，该工具的 schema 就是期望的输出格式，Ajv 校验确
保格式正确。优点：不限制模型的 reasoning 能力，可在同一轮混合文本和结构化输
出。

3. MCP 协议 (Model Context Protocol)
3.1 为什么重要
MCP 是 2025-2026 年 AI Agent 生态最重要的标准化协议。它被类比为“AI Agent 世
界的 USB 接口”，解决了 Agent 与外部工具/数据源集成的碎片化问题。掌握 MCP 的
客户端和服务端实现是 AI Agent 工程师的硬门槛。

3.2 代码阅读指引
顺序

文件
src/services/mcp/

1

client.ts (文件头

import 区)

关注点
三种 Transport：
StdioClientTransport、
SSEClientTransport、
StreamableHTTPClientTransport

src/services/mcp/

2

client.ts →

连接建立流程：Transport 选择

connectToServer() (第

→ Client 初始化 → capabilities

592 行， memoize 包

协商

裹)
src/services/mcp/

3

client.ts →
fetchToolsForClient()

工具发现：list_tools → 转换为
本地 Tool 格式

顺序
4

5

文件

关注点

src/tools/MCPTool/

适配器模式：将 MCP 工具桥接

MCPTool.ts

为项目的 Tool 接口

src/services/mcp/

MCP 服务器配置：如何声明一

config.ts

个 MCP Server

src/entrypoints/mcp.ts

6

→ startMCPServer()
(第 35 行)

7

反向角色：本项目作为 MCP
Server 暴露工具

src/services/mcp/

OAuth 认证流程：MCP Server

auth.ts

的安全接入

带着这些问题读代码：
1. 当用户配置了一个新的 MCP Server 时，从配置文件到工具可用，经历了哪些步
骤？
2. MCP 工具和本地工具（如 GlobTool）在 Tool 接口层面有什么区别？（提示：
看 isMcp 和 mcpInfo 字段）
3. 本项目自身作为 MCP Server 时，暴露了哪些工具？暴露方式和作为 Client 消费
工具有什么对称性？

3.3 面试准备
Q1：解释 MCP 协议的核心架构和通信机制。
回答思路：MCP 基于 JSON-RPC 2.0，定义了 Client 和 Server 两个角色。Client
（Host 应用，如本项目）发起请求，Server 提供工具/资源/提示。支持三种
Transport：Stdio（本地进程，适合 CLI 工具）、SSE（HTTP 长连接，适合远程服
务）、Streamable HTTP（更通用的 HTTP 流式传输）。协议的核心交互包括：
initialize （能力协商）→ tools/list （工具发现）→ tools/call （工具调用）。

Q2：在本项目中，MCP 的双向性是如何体现的？
回答思路：项目同时实现了 MCP Client 和 Server。Client 侧（ src/services/mcp/
client.ts ）连接外部 MCP Server，将远程工具适配为本地 Tool 接口，让 Agent 可以

调用外部服务。Server 侧（ src/entrypoints/mcp.ts ）将项目自身的工具暴露为 MCP 接
口，让其他 AI 应用可以使用本项目的能力。这种双向设计体现了 MCP “通用互操
作”的核心理念。

3.4 动手练习
目标：编写一个简单的 MCP Server 并接入本项目
步骤：
1. 阅读 MCP SDK 文档：访问 https://modelcontextprotocol.io/docs/gettingstarted/intro ，理解 Server 的最小实现
2. 创建项目：在项目外新建一个独立的 Node/Bun 项目
3. 实现 Server：使用 @modelcontextprotocol/sdk 创建一个 Stdio Server，暴露一
个简单工具（如 get_weather ，接收 city 参数，返回模拟天气数据）
4. 注册配置：在项目的 MCP 配置文件中添加你的 Server（stdio 类型，指向你的
脚本路径）
5. 启动测试：用 ./bin/claude-haha 启动，执行 /mcp 查看你的 Server 是否连接成
功
6. 对话验证：让 Agent 查询某个城市的天气，验证你的 MCP 工具是否被调用
验证方法：Agent 成功通过 MCP 协议调用了你自定义的 get_weather 工具并返回结
果。

3.5 延伸阅读
• MCP 官方文档：Model Context Protocol — 协议规范、SDK、教程
• MCP GitHub：modelcontextprotocol/modelcontextprotocol — 源码和规范
• Anthropic MCP 介绍：Introducing the Model Context Protocol — 设计动机和
愿景

3.6 Computer Use：从文本 Agent 到 GUI Agent
本项目包含一个完整的 Computer Use MCP Server（ src/vendor/computer-use-mcp/
），让 Agent 能操控桌面 GUI——截图、点击、打字、滚动、拖拽。这是 Agent 能力
边界从“读写文件 + 执行命令”扩展到“操控任意桌面应用”的关键。
核心代码：
文件

关注点

src/vendor/computer-use-mcp/

工具 Schema 定义： computer_batch （批量

tools.ts

动作）、截图、点击、滚动等

文件

关注点

src/vendor/computer-use-mcp/
executor.ts

动作执行器：将模型指令转为实际桌面操作

src/vendor/computer-use-mcp/

坐标模式：像素坐标 vs 归一化百分比

types.ts

（0-100）

src/vendor/computer-use-mcp/
deniedApps.ts / sentinelApps.ts
src/vendor/computer-use-mcp/
pixelCompare.ts

安全管控：应用黑名单、敏感应用检测

图像比对：判断截图是否变化

带着这些问题读代码：
1. 为什么需要两种坐标模式（像素 vs 归一化）？各自适用什么场景？
2. computer_batch 如何将多个动作编排为原子操作？
3. 安全管控如何防止 Agent 操控系统偏好设置、终端等敏感应用？
面试考点：Q：AI Agent 如何安全地操控桌面 GUI？
回答思路：Computer Use 的核心挑战是安全性。本项目实现了多层管控：(1)
deniedApps 黑名单完全禁止操控特定应用；(2) sentinelApps 检测前台应用是否在允

许列表内；(3) keyBlocklist 禁止危险按键组合（如 Cmd+Q 关闭应用）；(4) 每次操作
前截图对比，确保 Agent “看到”的和实际一致。

3.7 MCP vs A2A：Agent 互联的两种范式
2025-2026 年出现了两个重要的 Agent 互联协议，理解它们的区别是高级 Agent 工程
师的必备知识。

维度

MCP (Model Context
Protocol)

提出方

Anthropic

核心定

Agent ↔ 工具/数据源 的

位

连接协议

类比

“AI 世界的 USB 接口”

A2A (Agent-to-Agent Protocol)
Google
Agent ↔ Agent 的通信协议
“AI 世界的 HTTP 协议”

维度

MCP (Model Context
Protocol)

A2A (Agent-to-Agent Protocol)

交互模

Client 调用 Server 暴露

式

的工具

本项目

完整（Client + Server 双

未实现（但 SendMessageTool + Swarm 是

实现

向）

类似思路）

对等 Agent 之间发送任务和消息

面试考点：Q：MCP 和 A2A 有什么区别？什么场景用哪个？
回答思路：MCP 解决的是“Agent 如何获取外部能力”——通过标准化的 tools/list +
tools/call 让 Agent 发现和调用外部工具。A2A 解决的是“Agent 如何互相协作”——

通过标准化的任务委托和消息传递让不同 Agent 分工。在本项目中，MCP Client 连接
外部工具服务（如数据库、搜索引擎），而多 Agent 协同（ AgentTool +
SendMessageTool ）实现了类似 A2A 的功能但使用自定义协议。未来趋势是两者互

补：MCP 负责“纵向”能力扩展，A2A 负责“横向”Agent 协作。
延伸阅读：
• A2A 协议：Agent-to-Agent Protocol — Google 的 Agent 间通信协议
• MCP vs A2A 对比：Understanding A2A and MCP — Google Cloud 官方博客

4. Context Engineering / Prompt Engineering
4.1 为什么重要
2026 年，行业共识已从“Prompt Engineering”升级为“Context Engineering”。
不再只是写好一段提示词，而是工程化管理传给 LLM 的所有信息：系统提示词、工具
描述、用户上下文、对话历史、附件、记忆注入等。这是区分初级和高级 Agent 工程
师的关键能力。

4.2 代码阅读指引
顺序

文件

关注点
系统提示词模

1

src/constants/prompts.ts

板：观察提示词
的结构化分层

顺序

文件
src/context.ts → getSystemContext() /

2

getUserContext()

src/utils/queryContext.ts →

3

fetchSystemPromptParts()

关注点
动态上下文信息
收集（OS、
cwd、时间等）
提示词组装：多
个来源的提示词
片段如何合并
最终组装：

4

src/services/api/claude.ts → queryModel() (第

systemPrompt +

1017 行)

messages +
tools → API 请求
上下文注入点：

5

src/utils/api.ts → prependUserContext() /

用户上下文前

appendSystemContext()

置、系统上下文
后置
子 Agent 的提示

6

src/tools/AgentTool/prompt.ts

词：观察如何为
不同角色定制
prompt
Effort 级别：如

7

src/utils/effort.ts

何通过参数控制
LLM 的思考深度

带着这些问题读代码：
1. 系统提示词由哪些部分组成？它们的拼接顺序是什么？
2. userContext 和 systemContext 分别注入到 messages 的什么位置？为什么这样
设计？
3. 工具描述是如何嵌入到上下文中的？ defer_loading 工具和正常工具的描述有何
区别？

4.3 面试准备
Q1：在生产级 Agent 中，系统提示词应该如何结构化？

回答思路：以本项目为例，系统提示词分为多层：(1) CLI 环境前缀
（ getCLISyspromptPrefix ）；(2) 核心行为指令（角色、规则、约束）；(3) 工具描述
（由 toolToAPISchema 动态生成）；(4) 用户上下文（ userContext 前置到
messages）；(5) 系统上下文（ systemContext 后置）；(6) 动态附件
（CLAUDE.md、记忆文件等）。这种分层设计让每层可以独立更新，且通过 Prompt
Cache 机制，不变的前缀部分可以跨请求复用。
Q2：什么是 Context Engineering？它和 Prompt Engineering 有什么区别？
回答思路：Prompt Engineering 关注的是“写出好的提示词”，Context
Engineering 关注的是“工程化管理传给 LLM 的所有信息”。在本项目中，上下文管
理包括：(1) 动态工具发现和描述注入；(2) 对话历史的自动压缩（compact）；(3) 记
忆文件的按需加载（memory prefetch）；(4) Token 预算下的信息优先级排序；(5)
Prompt Cache 优化（保持前缀稳定以命中缓存）。这些远超“写好一段提示词”的
范畴。

4.4 动手练习
目标：追踪一次完整 API 调用中，系统提示词的组装过程
步骤：
1. 设置环境变量 CLAUDE_CODE_DUMP_PROMPTS=1 （如果项目支持），或在 src/
services/api/claude.ts 的 queryModel() 中添加日志，打印最终发给 API 的

system prompt 长度和前 500 字符
2. 启动项目，发送一条简单消息（如“你好”）
3. 记录系统提示词的结构：总长度、各部分的占比
4. 再发送一条涉及工具的消息（如“列出当前目录的文件”），对比两次的系统提
示词差异
5. 查看 userContext 注入了哪些环境信息
验证方法：你能画出系统提示词的层次结构图，并标注各层的 Token 占比。

4.5 延伸阅读
• Anthropic Prompt Engineering 指南：Prompt engineering overview — 官方
最佳实践
• Context Engineering 概念：AI 软件开发必备 Skill 与 Agent：2026 工程师核心
能力图谱 — Context Engineering 的系统阐述

• Claude System Prompt 最佳实践：System prompts — 如何设计有效的系统提
示词

5. 上下文窗口管理与 Token 优化
5.1 为什么重要
上下文窗口是 Agent 的“工作记忆”，有限且昂贵。一个 200k token 的窗口，每次
API 调用的输入成本就是真金白银。生产级 Agent 必须精细管理 Token 预算——何时
压缩、如何压缩、如何利用 Prompt Cache 降低成本。这是高薪 Agent 工程师和初级
API 调用者之间的核心差距。

5.2 代码阅读指引
顺序

文件

关注点

src/services/compact/

1

autoCompact.ts →

入口：如何计算“何时

getEffectiveContextWindowSize()

需要压缩”

(第 33 行)
2

src/query.ts 中搜索 autocompact

src/services/compact/compact.ts

3

→ compactConversation() (第 387
行)

4

5

6

压缩在 Agent Loop 中的
调用时机
核心压缩：将完整对话
历史压缩为摘要

src/services/compact/

轻量压缩：针对工具输

microCompact.ts

出的细粒度压缩

src/services/compact/
reactiveCompact.ts

src/services/tokenEstimation.ts

被动压缩：API 报
prompt-too-long 时的
应急压缩
Token 计数：如何估算
消息的 Token 数量

顺序

文件
src/services/api/

7

promptCacheBreakDetection.ts

关注点
Cache 监控：检测
Prompt Cache 命中率
下降

带着这些问题读代码：
1. autoCompact 的触发阈值是如何计算的？（有效窗口 = 模型窗口 - 预留输出空
间）
2. compact 和 microCompact 的区别是什么？什么场景用哪个？
3. cache_creation_input_tokens 和 cache_read_input_tokens 分别代表什么？为什
么 Cache 命中率对成本至关重要？

5.3 面试准备
Q1：如何管理 AI Agent 的上下文窗口？
回答思路：本项目实现了多层压缩策略：(1) autoCompact：当 Token 数接近窗口上
限时，主动调用 LLM 将对话历史压缩为摘要；(2) microCompact：对冗长的工具输
出（如大文件内容）做细粒度裁剪；(3) snipCompact：片段级压缩，移除不再需要的
中间结果；(4) reactiveCompact：当 API 返回 prompt-too-long 错误时的应急压缩；
(5) contextCollapse：将多轮搜索/读取操作折叠为摘要。这些策略形成了从“主动预
防”到“被动恢复”的完整梯度。
Q2：什么是 Prompt Cache？如何优化命中率？
回答思路：Anthropic API 的 Prompt Cache 允许重复的 prompt 前缀跨请求复用，命
中时只收取 cache_read_input_tokens （比 cache_creation_input_tokens 便宜
~90%）。优化命中率的关键是保持 prompt 前缀的稳定性：(1) 系统提示词放在最前
面且不频繁变更；(2) 工具 schema 的序列化顺序保持一致；(3) compact 后重建的消
息尽量复用之前的结构。项目通过 promptCacheBreakDetection 监控命中率下降并告
警。

5.4 动手练习
目标：观察 autoCompact 的触发过程

步骤：
1. 设置较小的上下文窗口（设置环境变量
CLAUDE_CODE_AUTO_COMPACT_WINDOW=8000 ）

2. 启动项目，进行一次较长的对话（让 Agent 读几个文件、搜索代码）
3. 观察 autoCompact 何时触发（控制台应有相关日志）
4. 触发后，对比压缩前后的 messages 数组长度和内容变化
5. 记录 compactionUsage （压缩本身消耗了多少 Token）
验证方法：你能描述 autoCompact 的完整触发流程，包括阈值计算、压缩执行、和
压缩后消息重建。

5.5 延伸阅读
• Anthropic Prompt Caching：Prompt caching — 官方缓存机制文档
• Token 计数指南：Token counting — 如何准确计数和估算 Token

6. Streaming 与实时交互
6.1 为什么重要
流式输出让用户看到 Agent “实时思考”的过程，是优秀 AI 产品体验的基础。从 SSE
协议解析、增量文本渲染、到流式工具执行，Streaming 涉及前后端全链路。

6.2 代码阅读指引
顺序

文件

关注点

src/services/api/claude.ts →

1

queryModelWithStreaming()

流式 API 调用入口

(第 752 行)

2

3

src/services/api/claude.ts →

message_start →

queryModel() (第 1017 行) 内

content_block_delta →

的 SSE 事件处理循环

message_delta

src/services/api/claude.ts →

流式空闲超时看门狗

如何检测和处理挂起的流

顺序
4

文件

关注点

src/services/api/claude.ts →

流式 usage 的累计逻辑（注

updateUsage() (第 2924 行)

意：是累计值不是增量）

src/services/tools/
StreamingToolExecutor.ts (第

5

40 行)

边流式接收边执行工具的优
化
WebSocket 层：CLI →

6

src/server/ws/handler.ts

Server → Desktop 的事件转
发

带着这些问题读代码：
1. 一条流式消息经历了几层传递？（LLM API → SSE → AsyncGenerator →
WebSocket → Desktop UI）
2. 流式超时看门狗的工作原理是什么？为什么需要它？
3. StreamingToolExecutor 如何做到“边流式接收边执行工具”？它和传统的“等
消息完整再执行”有什么性能差异？

6.3 面试准备
Q1：描述一个 AI Agent 的流式输出架构。
回答思路：本项目实现了三层流式架构：(1) SSE 层：Anthropic API 返回 ServerSent Events 流，SDK 解析为 BetaRawMessageStreamEvent ；(2) AsyncGenerator
层： queryModel() 是一个生成器函数，逐个 yield StreamEvent，上层 queryLoop 按
需消费；(3) WebSocket 层：本地服务端通过 WebSocket 将事件实时推送到桌面客户
端。三层解耦使得每层可以独立处理背压、超时和错误。
Q2：如何处理流式连接的异常？
回答思路：项目实现了流式空闲超时看门狗（Watchdog）。当 SSE 流超过
STREAM_IDLE_TIMEOUT_MS （默认 90 秒）没有收到新 chunk 时，看门狗主动 abort

连接并重试。同时还有“半超时警告”机制（45 秒），以及 stall 检测（两个 chunk
之间的间隔过长）。这防止了静默断连导致的会话永久挂起。

6.4 动手练习
目标：追踪一条流式消息从 API 到 UI 的完整路径

步骤：
1. 在 src/services/api/claude.ts 的 SSE 事件处理循环中，为每种事件类型
（ message_start 、 content_block_start 、 content_block_delta 、
message_delta ）添加日志

2. 启动项目，发送一条消息
3. 记录每种事件出现的顺序和频率
4. 特别关注 content_block_delta 的 text_delta ：它是如何被增量拼接成完整文本
的
验证方法：你能画出一条消息从 API SSE 事件到最终屏幕渲染的完整数据流图。

6.5 延伸阅读
• Anthropic Streaming 文档：Streaming Messages — SSE 事件格式详解
• Server-Sent Events 规范：MDN: Server-Sent Events — SSE 协议基础

7. 记忆系统 (Memory)
7.1 为什么重要
记忆是让 Agent 从“一次性对话”进化到“持续协作”的关键。短期记忆（对话上下
文）、工作记忆（compact 摘要）、长期记忆（持久化文件）的分层设计，直接影响
Agent 的智能程度和用户体验。

7.2 代码阅读指引
顺
序
1

文件
src/services/SessionMemory/
sessionMemory.ts

2

src/services/SessionMemory/prompts.ts

3

src/services/extractMemories/

4

关注点

会话记忆核心：什么信息被记住
记忆提取使用的提示词
自动记忆提取：从对话中识别值得
记住的信息

src/services/compact/

compact 时如何保存记忆到持久

sessionMemoryCompact.ts

层

顺
序

文件

5

src/memdir/

6

src/tools/AgentTool/agentMemory.ts

7

关注点
记忆目录系统（memdir）：文件
系统级持久化
Agent 级记忆：子 Agent 的记忆
隔离

src/query.ts 中搜索

记忆预取：在 LLM 推理时并行加

pendingMemoryPrefetch

载相关记忆

带着这些问题读代码：
1. 记忆的三层架构（上下文内 → compact 摘要 → 持久化 memdir）之间是如何转
换的？
2. pendingMemoryPrefetch 是如何在不阻塞 Agent Loop 的情况下预取相关记忆
的？
3. 子 Agent 的记忆和主 Agent 的记忆是共享的还是隔离的？

7.3 面试准备
Q1：如何为 AI Agent 设计记忆系统？
回答思路：分三层设计：(1) 短期记忆：当前对话上下文中的 messages 数组，随时可
被 LLM 访问；(2) 工作记忆：compact 时由 LLM 生成的对话摘要，保留关键信息但大
幅减少 Token；(3) 长期记忆：通过 extractMemories 从对话中提取结构化信息，持久
化到 memdir 文件系统。在新会话开始或 compact 后，通过 memoryPrefetch 机制按
需加载相关记忆回上下文。
Q2：记忆预取如何实现非阻塞加载？
回答思路：使用 startRelevantMemoryPrefetch 在每轮 Agent Loop 开始时启动异步记
忆检索。该 prefetch 使用 using 声明（Resource Management 语法），在 Loop 中
后续位置（工具执行完成后）通过检查 settledAt 判断是否已完成。已完成则消费结
果注入 messages，未完成则跳过，在下一轮迭代重试。这样记忆加载和 LLM 推理/工
具执行重叠执行，不增加延迟。

7.4 动手练习
目标：理解记忆的生命周期

步骤：
1. 找到项目的 memdir 路径（通常在 ~/.claude/ 或项目配置中）
2. 查看当前已有的记忆文件内容
3. 在项目中进行一次有意义的对话（如“我偏好用 TypeScript，请记住”）
4. 触发 compact（用 /compact 命令或等自动触发）
5. 再次查看 memdir，观察新增了什么记忆文件
6. 在新会话中验证记忆是否被加载（问 Agent “你知道我偏好什么语言吗”）
验证方法：你能描述记忆从“对话中产生”到“被持久化”再到“在新会话中被召
回”的完整生命周期。

7.5 延伸阅读
• LLM 记忆系统综述：Building AI Agents with Memory — Anthropic 关于 Agent
记忆的设计建议
• MemGPT 论文：MemGPT: Towards LLMs as Operating Systems — 将 LLM 记
忆管理类比操作系统虚拟内存

8. 多 Agent 协同 (Multi-Agent)
8.1 为什么重要
复杂任务往往需要多个 Agent 分工协作。Multi-Agent 系统涉及任务分解、上下文共
享与隔离、消息传递、并行执行与结果汇总。这是 2026 年高级 Agent 工程师的核心
竞争力。

8.2 代码阅读指引
顺序

文件

关注点

src/tools/AgentTool/

1

runAgent.ts → runAgent()

子 Agent 的完整生命周期

(第 248 行)
2

src/tools/AgentTool/

Fork 模式：共享父 Agent 上

forkSubagent.ts

下文的分叉

顺序
3

文件

关注点

src/tools/AgentTool/

自定义 Agent 加载：从目录

loadAgentsDir.ts

读取 Agent 定义

src/tools/AgentTool/

4

builtInAgents.ts
src/utils/swarm/
inProcessRunner.ts →

5

runInProcessTeammate()

内置 Agent 定义

Swarm 模式：进程内多
Agent 协同

6

src/tools/SendMessageTool/

Agent 间消息传递

7

src/tools/TeamCreateTool/

团队创建与管理

带着这些问题读代码：
1. 子 Agent 和父 Agent 共享了什么（ query() 函数）？隔离了什么（独立的
messages、toolUseContext）？
2. forkSubagent 和 runAgent 的区别是什么？什么场景用 fork？
3. Swarm 模式下多个 Agent 如何协调——是有一个“协调者”还是完全对等？

8.3 面试准备
Q1：如何设计一个 Multi-Agent 系统？
回答思路：本项目实现了两种模式：(1) 主-从模式（AgentTool）：主 Agent 通过
runAgent() 创建子 Agent，子 Agent 拥有独立的 messages 和 toolUseContext，但

共享 query() 核心引擎。子 Agent 完成后结果通过 yield 返回给父 Agent。(2)
Swarm 模式（inProcessRunner）：多个 Agent 在同一进程内并行执行，通过
SendMessageTool 互相通信，由一个“Leader”Agent 协调任务分配。两种模式的关

键共同点是：每个 Agent 都运行独立的 Agent Loop（ query() ），但工具集和权限可
以按角色定制。
Q2：子 Agent 的上下文如何管理？
回答思路：子 Agent 通过 createSubagentContext() 创建独立的 toolUseContext ，包
含：(1) 独立的 abortController （父中断不影响子）；(2) 独立的 readFileState （文件
读取缓存隔离）；(3) 独立的 agentId （用于会话记录分离）。但共享：(1)
getAppState/setAppState （全局状态，如权限配置）；(2) MCP 连接。子 Agent 的对

话记录通过 recordSidechainTranscript() 写入独立的 sidechain 文件，支持后续恢复。

8.4 动手练习
目标：理解子 Agent 的创建和执行流程
步骤：
1. 在对话中要求 Agent 执行一个会触发子 Agent 的任务（如“帮我搜索所有
TODO 并写一个总结”）
2. 观察日志中 runAgent 的调用
3. 在 src/tools/AgentTool/runAgent.ts 的 runAgent() 入口添加日志，打印子 Agent
的 agentDefinition.agentType 和 promptMessages 的数量
4. 对比父 Agent 和子 Agent 的 toolUseContext 中 tools 数组的差异
5. 查看 sidechain transcript 文件，理解子 Agent 的对话是如何记录的
验证方法：你能画出父 Agent 调用子 Agent 的时序图，包括上下文创建、query 调
用、结果回传。

8.5 延伸阅读
• Multi-Agent 设计模式：AI Agent Orchestration Patterns (Microsoft) — 微软的
Agent 编排模式指南
• A2A 协议：Agent-to-Agent Protocol — Google 的 Agent 间通信协议

8.6 Coordinator 模式：高级多 Agent 编排
除了主-从模式和 Swarm 模式，项目还实现了更高级的 Coordinator 模式——一个带
阶段划分、并行 Worker 调度、验证环节的协调者架构。
核心代码：
文件

关注点

src/coordinator/

Coordinator 系统提示：阶段划分（规划 →

coordinatorMode.ts

并行执行 → 验证）、Worker 调度策略

src/utils/agentContext.ts

src/tools/shared/
spawnMultiAgent.ts
src/utils/swarm/permissionSync.ts

AsyncLocalStorage 实现跨并发子 Agent 的归

因追踪
多后端 spawn：分屏/进程内等多种并发模式
并发 Agent 间的权限同步（邮箱转发机制）

三种多 Agent 模式对比：

维度

主-从模式
(AgentTool)

Swarm 模式

Coordinator 模式
协调者→Worker，

拓扑

父→子，树状

对等，网状

任务

父 Agent 显式创建

Leader 协调，

Coordinator 按阶段

分配

子任务

SendMessage 互通

分发

子 Agent 串行

进程内并行

阶段内 Worker 并行

简单的任务委托

多角色协作

复杂项目级任务

并发
度
适用
场景

星状

面试考点：Q：如何编排多个 Agent 处理一个复杂项目级任务？
回答思路：采用 Coordinator 模式——(1) 规划阶段：Coordinator 分析任务，拆解为
可并行的子任务；(2) 执行阶段：每个子任务分配给独立 Worker（通过
spawnMultiAgent 创建），Worker 在独立的 worktree 中工作互不干扰；(3) 验证阶

段：Coordinator 检查所有 Worker 的输出，合并结果。关键工程挑战：
AsyncLocalStorage 解决并发 Agent 的日志归因、 permissionSync 解决权限请求的跨

Agent 转发、每个 Worker 共享模型调用能力但隔离工作目录和对话上下文。

9. 安全、权限与沙箱
9.1 为什么重要
AI Agent 能执行代码、读写文件、发送网络请求，权限不当可能造成严重后果。安全
与权限控制是 Agent 产品化落地的硬性前提，也是面试中展现系统设计能力的好题
目。

9.2 代码阅读指引
顺序

文件
src/utils/permissions/

1

PermissionMode.ts

2

关注点
三种权限模式：default / plan
/ yolo
（bypassPermissions）

src/utils/permissions/

权限检查主逻辑：

permissions.ts

checkPermissions()

src/utils/permissions/
bashClassifier.ts →

3

classifyBashCommand()

Bash 命令危险性分类器

(第 40 行)
4

5

src/utils/permissions/

危险命令模式定义（rm -rf、

dangerousPatterns.ts

chmod 777 等）

src/utils/permissions/

YOLO 模式下的自动安全分类

yoloClassifier.ts

器

src/utils/permissions/

6

shellRuleMatching.ts
src/services/mcp/

7

channelPermissions.ts

Shell 命令与规则模式匹配

MCP 工具的权限管控

带着这些问题读代码：
1. 权限检查的三层模型是什么？（命令分类 → 规则匹配 → 用户确认）
2. 什么命令在 YOLO 模式下仍然会被拦截？
3. MCP 工具和本地工具的权限模型有什么区别？

9.3 面试准备
Q1：如何为 AI Agent 设计安全的权限系统？
回答思路：三层递进模型：(1) 命令分类： bashClassifier 将命令分为安全（ls、
cat）、写入（mkdir、cp）、危险（rm -rf、chmod 777）三级；(2) 规则匹配：用户
配置的 alwaysAllowRules / alwaysDenyRules / alwaysAskRules 对工具+参数进行模式
匹配（如 Bash(git *) 允许所有 git 命令）；(3) 最终决策：未被规则覆盖的写入/危险

操作弹出 UI 询问用户。此外还有三种全局模式： default （正常审核）、 plan （只允
许只读操作）、 yolo （自动放行，但危险模式仍拦截）。
Q2：YOLO 模式如何在“方便”和“安全”之间平衡？
回答思路：YOLO 模式并非完全禁用安全检查，而是引入 yoloClassifier 作为自动审
核。它会将命令分为“自动放行”和“仍需确认”两类。 dangerousPatterns 中定义的
高危命令（如递归删除、权限修改、网络发送）即使在 YOLO 模式下也不会自动放
行。这实现了“日常操作无需确认、高危操作永远确认”的平衡。

9.4 动手练习
目标：理解权限检查的完整流程
步骤：
1. 以默认模式启动项目
2. 让 Agent 执行 ls -la ，观察是否需要确认（应该自动放行）
3. 让 Agent 执行 rm -rf /tmp/test ，观察权限提示（应该要求确认）
4. 在 src/utils/permissions/bashClassifier.ts 中找到分类逻辑，理解 ls 和 rm -rf 被
分到哪一类
5. 配置一条 alwaysAllowRules （如允许 Bash(echo *) ），验证该规则生效
验证方法：你能解释为什么 ls 自动通过而 rm -rf 需要确认，以及规则配置如何覆盖
默认行为。

9.5 延伸阅读
• Agentic AI System 安全设计：The Complete Agentic AI System Design
Interview Guide 2026 — Agent 安全架构面试指南
• OWASP LLM Top 10：OWASP Top 10 for LLM Applications — LLM 应用安全风
险清单

9.6 Worktree 隔离：环境级安全沙箱
除了命令级权限控制，项目还实现了环境级隔离——通过 Git Worktree 为 Agent 创建
独立的工作目录，实验性修改不污染主分支。
核心代码：

文件

关注点

src/tools/EnterWorktreeTool/
EnterWorktreeTool.ts
src/tools/ExitWorktreeTool/
ExitWorktreeTool.ts
src/utils/worktree.ts

创建 worktree、切换 cwd、清除提示词缓存

退出 worktree、恢复原始目录
Worktree 管理：创建、验证 slug、会话绑定

为什么这个设计很重要：
命令级权限（“这条命令能不能执行”）和环境级隔离（“在哪个目录执行”）是两
个正交的安全维度。Worktree 隔离使得： - best-of-N 并行尝试：多个子 Agent 在各
自的 worktree 分支上同时尝试不同方案，最终选最优 - 安全实验：Agent 可以大胆修
改代码而不影响用户的工作目录 - 原子回滚：失败的实验只需删除 worktree，无需
git reset

面试考点：Q：如何让 Agent 安全地做代码实验？
回答思路：两层隔离——(1) Git Worktree 提供独立工作目录和分支，Agent 的修改完
全不影响主分支；(2) 权限系统在 worktree 内仍然生效，危险命令仍需确认。这种设
计支持“best-of-N”策略：让 N 个 Agent 在各自 worktree 中并行尝试，
Coordinator 比较结果后选最优方案合并。

10. LLMOps / 可观测性
10.1 为什么重要
生产级 Agent 需要答得好（质量）、跑得快（性能）、花得少（成本）。LLMOps 覆
盖了 API 监控（TTFT、OTPS）、Token 追踪、成本管理、错误恢复、A/B 测试
（Feature Flag）等全链路可观测能力。

10.2 代码阅读指引
顺
序
1

文件

关注点

src/cost-tracker.ts

成本追踪： getTotalCost() 、 getModelUsage()

顺
序

文件

2

src/services/api/logging.ts

3

src/services/api/usage.ts
src/services/api/

4

withRetry.ts

5

src/services/vcr.ts
src/services/

6

diagnosticTracking.ts

7

src/services/analytics/
src/services/

8

rateLimitMessages.ts

关注点
API 日志：TTFT（首 Token 时间）、OTPS（每
秒输出 Token）
API 用量统计
重试机制：指数退避、模型降级
VCR 模式：录制和回放 API 交互
诊断追踪
分析和 Feature Flag（GrowthBook）
限流消息处理

带着这些问题读代码：
1. VCR 模式是如何录制 API 请求/响应的？录制文件存在哪里？如何回放？
2. 当 API 返回 429（限流）时，系统是如何处理的？重试策略是什么？
3. 成本追踪是实时的吗？如何计算一次对话的总花费？

10.3 面试准备
Q1：如何监控和优化 AI Agent 的性能和成本？
回答思路：本项目从多个维度做监控：(1) 延迟指标：TTFT（Time To First Token，
首 Token 延迟）和 OTPS（Output Tokens Per Second，输出速度）记录在
logging.ts ；(2) 成本追踪： cost-tracker.ts 根据 API usage 中的 input_tokens、

output_tokens、cache_read 等字段实时计算费用；(3) 重试机制： withRetry.ts 实现
指数退避重试，429/500 等暂时性错误自动重试，模型不可用时降级到 fallback
model；(4) VCR 调试：录制完整的 API 请求/响应，可以离线回放调试，不消耗 API
额度。
Q2：什么是 VCR 模式？在 Agent 开发中有什么价值？
回答思路：VCR（Video Cassette Recorder）模式可以录制 LLM API 的完整请求和响
应，存为本地文件。在开发和调试 Agent 时，可以回放录制，不需要真实调用 API。

价值：(1) 调试确定性——同一个录制回放结果完全一致；(2) 节省成本——不消耗 API
额度；(3) 离线开发——没有网络也能运行和测试。

10.4 动手练习
目标：理解成本追踪和 API 监控
步骤：
1. 启动项目，进行几轮对话
2. 使用 /cost 命令（或查看 cost-tracker.ts 的输出）查看当前会话的成本
3. 关注 cache_creation_input_tokens vs cache_read_input_tokens 的比例
4. 计算 Prompt Cache 命中率： cache_read / (cache_read + cache_creation)
5. 尝试理解 withRetry.ts 的重试逻辑，在什么条件下会重试、最多重试几次
验证方法：你能解读一次会话的 API usage 报告，计算总成本，并指出 Prompt
Cache 优化的效果。

10.5 延伸阅读
• Anthropic API Usage：Messages API usage — API 用量字段说明
• LLMOps 实践：AI Agent 核心概念：Agent Loop、Context Engineering、
Tools 注册 — Agent 工程实践综合指南

10.6 Feature Flag 与灰度发布
生产级 Agent 不是“发版一刀切”——新 Prompt、新工具、新模型都需要灰度验证。
项目集成了 GrowthBook 实现完整的特性开关和 A/B 测试框架。
核心代码：
文件

关注点

src/services/analytics/

GrowthBook 客户端：用户属性（平台、组

growthbook.ts

织、订阅类型）→ 特性开关 → 实验曝光日志

src/services/analytics/config.ts

分析系统配置

src/services/analytics/datadog.ts

Datadog 集成：性能指标上报

src/services/analytics/sink.ts /
sinkKillswitch.ts

事件汇集层 + 紧急关停开关

带着这些问题读代码：
1. GrowthBook 的用户属性（ GrowthBookUserAttributes ）包含哪些维度？如何实
现“按组织灰度”？
2. sinkKillswitch 的作用是什么？为什么分析系统需要紧急关停能力？
3. Feature Flag 如何与 useMainLoopModel 配合，实现“按灰度切换模型”？
面试考点：Q：如何安全地为 Agent 灰度发布新功能？
回答思路：通过 Feature Flag 系统（如 GrowthBook）实现多维度灰度：(1) 按用户/
组织/平台分批放开新 Prompt 模板；(2) 按特性开关控制新工具是否可见（与
shouldDefer / defer_loading 配合）；(3) 按实验组切换模型（A 组用 Claude

Sonnet、B 组用 Claude Opus）。曝光日志记录每次特性开关的命中情况，配合
Datadog 的性能指标形成闭环。

10.7 Agent 评估方法论（2026 核心能力）
“没有度量，就没有改进”。Agent 的输出质量评估远比传统软件测试复杂——输出是
概率性的、多维度的、上下文依赖的。2026 年行业共识：评测是 Agent 系统的控制平
面。只有不到 25% 的企业对 Agent 可靠性有信心，掌握评测工程是 AI Agent 工程师
从“能做 Demo”到“能上生产”的核心分水岭。
项目中已有的评估基础设施：
工具

用途

VCR 录制回放

Prompt Cache 监控
成本追踪
性能日志

录制 API 请求/响应，
离线确定性回放

文件
src/services/vcr.ts

检测 prompt 变更导致

src/services/api/

的缓存失效

promptCacheBreakDetection.ts

逐请求计费，计算 ROI

src/cost-tracker.ts

TTFT、OTPS 等延迟指
标

src/services/api/logging.ts

Agent 评估金字塔：
┌──────────────────────────┐
│ Level 4: 人工评估 (最贵) │ ← 复杂场景的最终裁判
├──────────────────────────┤

│ Level 3: LLM-as-Judge │ ← 用更强模型自动化评分
├──────────────────────────┤
│ Level 2: VCR/Golden Set │ ← 录制回放 + 确定性的断言
├──────────────────────────┤
│ Level 1: 单元/快照测试(最便宜)│ ← Schema 校验、格式检查
└──────────────────────────┘

2026 年主流评测框架：
框架

MLflow

agentevals
(Solo.io)

DeepEval

Ragas

定位

核心能力

全栈平台

Trace 级评分、Agent

（3000万

GPA 打分、LLM Judge

+月下载）

对齐

OTel 原生
（2026.3 发
布）

Golden Set 对照、轨迹
匹配、MCP Server 集成

pytest 原生

50+ 预置指标、Span 级

CI/CD 集成

评估

研究驱动

Faithfulness/

RAG +

Relevancy/

Agent

AgentGoalAccuracy

适用场景

企业级全链路评测

分布式 Agent 系统评测

已用 pytest 的团队

RAG 管线专项评测

2026 年评测趋势：闭环改进（Closed-Loop Improvement）
Google Cloud Next 2026 提出的模式：
Planner(规划) → Simulator(模拟执行) → Evaluator(LLM-as-Judge 评分)
↓
不合格 → Refine(优化 Prompt/工具/策略)
↓
合格 → Deploy(发布)

关键指标： - 系统优化周期从“数周人工调试”缩短为“数小时自动迭代” - 通过
agentevals MCP Server ，Claude Code 等 Agent 可以直接运行评测 - Hallucination

Rate 和 Tool Selection Accuracy 是最重要的评测指标 - 采用评测驱动的团队输出一致
性提升可达 40%
面试考点：Q：如何评估 AI Agent 的输出质量？

回答思路：四层评估体系：(1) 单元测试：校验工具输入/输出的 Schema、测试格式
转换函数（项目中 proxy-transform.test.ts 就是范例）；(2) VCR 回归测试：使用
vcr.ts 录制真实 API 交互，后续回放验证输出是否回归——确定性高、成本为零；(3)

LLM-as-Judge：用另一个 LLM（通常用更强的模型）对 Agent 输出打分，适合评
估“回答质量”这类主观维度，通过 MLflow 的 GEPA/MemAlign 算法校准自动裁判
与人工标签的一致性；(4) 人工评估：复杂场景（如多步推理、代码生成）仍需人工审
核，但通过前三层过滤，人工只需审核少量边界 case。
Q：如何搭建 Agent 持续评测流水线？
回答思路：(1) Golden Set 构建：从历史成功会话中提取 50-200 个代表性 case，人
工标注期望输出；(2) CI/CD 集成：每次 Prompt/工具变更触发 DeepEval/agentevals
回归测试；(3) 在线监控：通过 real-time tracing（OTel）持续监控生产环境 Agent 质
量；(4) 闭环优化：评估结果反馈到 Prompt 优化（GEPA/MIPRO 自动优化）。
延伸阅读：
• Anthropic 评估指南：Evaluating AI outputs — 官方测试与评估最佳实践
• LLM-as-Judge：Judging LLM-as-a-Judge — 用 LLM 做评估的方法论
• agentevals：agentevals GitHub — OTel 原生 Agent 评测框架
• MLflow Agent Evaluation：MLflow Agent Eval — 全栈 Agent 评测指南

11. 多 Provider 与模型路由
11.1 为什么重要
2026 年企业级 Agent 几乎不可能只用一个 LLM Provider。团队需要在 Anthropic、
OpenAI、开源模型（DeepSeek、Llama）之间灵活切换——有时是为了成本（小任务
用便宜模型），有时是为了合规（数据不出境用私有部署），有时是为了容灾（主模
型限流时自动降级）。理解多 Provider 架构和协议转换是高级 Agent 工程师的必备能
力。

11.2 代码阅读指引
顺
序
1

文件

src/utils/model/providers.ts

关注点
四种 Provider： firstParty / bedrock / vertex /
foundry

顺
序
2

3

4

文件

关注点

src/utils/model/

动态模型能力发现：从 API 获取

modelCapabilities.ts

max_input_tokens / max_tokens ，本地缓存

src/utils/model/aliases.ts /
configs.ts

src/server/proxy/handler.ts

模型别名系统、默认配置
协议转换代理：Anthropic → OpenAI Chat/
Responses，含流式映射

src/server/proxy/transform/

请求转换：system prompt、messages、

anthropicToOpenaiChat.ts

tool_use → tool_calls、image 格式

src/server/proxy/streaming/

流式响应转换：OpenAI SSE → Anthropic SSE 事

openaiChatStreamToAnthropic.ts

件

7

src/services/api/withRetry.ts

Fallback 降级：主模型失败时切换备用模型

8

src/utils/effort.ts

5

6

Effort 级别：通过参数控制“思考深度”，间接影
响模型选择

带着这些问题读代码：
1. Anthropic 和 OpenAI 的 Tool Use 格式有什么具体差异？
anthropicToOpenaiChat.ts 如何做映射？

2. 流式响应的格式转换有什么特殊挑战？（提示：增量 delta vs 完整 block）
3. modelCapabilities.ts 如何实现“最长前缀匹配”来查找模型能力？为什么用这
种策略？

11.3 面试准备
Q1：如何设计一个支持多 LLM Provider 的 Agent 系统？
回答思路：本项目的架构是“内部统一使用 Anthropic Messages 格式 + 外部通过
Proxy 适配其他 Provider”。核心组件：(1) providers.ts 通过环境变量选择
Provider；(2) proxy/handler.ts 实现双向协议转换（请求：Anthropic→OpenAI，响
应：OpenAI→Anthropic）；(3) modelCapabilities.ts 从 API 动态获取模型能力并缓
存，支持同一 Provider 下的多模型切换。这种设计的好处是Agent Loop 和 Tool 系统
完全不感知 Provider 差异，只有最底层的 API 调用层处理协议转换。

Q2：Anthropic 和 OpenAI 的 API 在 Tool Use 上有什么区别？
回答思路：三个核心差异——(1) 格式：Anthropic 用 content 数组中的 tool_use
block，OpenAI 用 tool_calls 数组；(2) 结果返回：Anthropic 用 tool_result role 的
message，OpenAI 用 tool role；(3) 图像：Anthropic 用 { type: "image", source:
{ type: "base64" } } ，OpenAI 用 { type: "image_url", image_url: { url: "data:..." } } 。

项目的 proxy/transform/ 完整处理了这些差异，包括 thinking block 的丢弃（OpenAI
不支持）。

11.4 动手练习
目标：理解协议转换层的工作方式
步骤：
1. 阅读 src/server/proxy/transform/types.ts ，理解 Anthropic 和 OpenAI 各自的类
型定义
2. 重点对比 AnthropicMessage 和 OpenAIChatMessage 的结构差异
3. 在 anthropicToOpenaiChat.ts 中追踪一个包含 tool_use 的 assistant message 的
转换过程
4. 查看测试文件 src/server/__tests__/proxy-transform.test.ts ，理解边界情况的处理
5. （可选）尝试配置一个 OpenAI 兼容的 Provider（如本地 Ollama），通过
proxy 接入项目
验证方法：你能画出 Anthropic Messages 请求 → OpenAI Chat Completions 请求的
完整字段映射图。

11.5 延伸阅读
• Anthropic Messages API：Messages API Reference — Anthropic 格式规范
• OpenAI Chat Completions API：Chat Completions — OpenAI 格式规范
• LiteLLM：litellm.ai — 开源多 Provider 适配层，与本项目 proxy 思路相似

12. Agent 产品化：多通道接入
12.1 为什么重要
一个 Agent 只能在 CLI 里用远远不够。实际产品需要接入飞书、钉钉、Telegram、
Web 等多个通道。如何设计通道适配层、管理并发会话、处理附件和流式消息，是
Agent 产品化落地的关键工程挑战。

12.2 代码阅读指引
顺
序
1

2

3

4

5

6

7

8

文件

adapters/README.md

关注点
适配层架构总览：Desktop → API → adapters
→ WS → session

adapters/common/ws-

WebSocket Bridge：自动重连、心跳、FIFO

bridge.ts

消息序列化

adapters/common/chatqueue.ts
adapters/common/
message-dedup.ts

并发会话队列管理

消息去重：防止 IM 平台重发导致重复处理

adapters/common/

跨平台附件抽象： AttachmentRef 、大小限制、

attachment/

ImageBlockWatcher

adapters/feishu/index.ts

飞书适配器：Lark WSClient → WsBridge →
Claude 会话

adapters/feishu/streaming-

流式卡片：将 Agent 流式输出映射到飞书

card.ts

CardKit

adapters/telegram/
index.ts

Telegram 适配器：grammy Bot API 封装

带着这些问题读代码：
1. WsBridge 的重连策略是什么？（指数退避、最大重试次数、心跳间隔）
2. 为什么需要 chat-queue ？如果用户在 Agent 处理上一条消息时又发了新消息怎
么办？

3. ImageBlockWatcher 如何检测 Agent 输出中的图片引用并自动上传到 IM 平台？
4. 飞书和 Telegram 适配器之间共享了哪些逻辑（ common/ ），各自定制了什
么？

12.3 面试准备
Q1：如何让 AI Agent 接入企业 IM（如飞书、钉钉）？
回答思路：分三层设计——(1) 通用适配层（ common/ ）：WsBridge（WebSocket 通
信 + 重连）、ChatQueue（并发会话管理）、MessageDedup（幂等性）、
AttachmentRef（跨平台附件抽象）；(2) 平台适配层（ feishu/ / telegram/ ）：处理
平台特有的消息格式（飞书 CardKit 卡片 vs Telegram 纯文本）、媒体上传（飞书
im.message.create vs Telegram sendPhoto）；(3) 会话桥接：IM 消息通过
WsBridge → 本地 WebSocket → Claude Code Session → Agent Loop。这种架构的
好处是 Agent 核心完全不感知通道差异，新增通道只需实现平台适配层。
Q2：IM 适配中有哪些常见的工程挑战？
回答思路：五个核心挑战——(1) 消息去重：IM 平台可能重发消息（网络抖动），需要
message-dedup 保证幂等；(2) 并发控制：用户连续发多条消息时， chat-queue 确保

按序处理，避免上下文错乱；(3) 附件处理：不同平台图片/文件格式不同，需统一为
AttachmentRef 后转成 API 的 ImageBlockParam ；(4) 流式输出映射：Agent 的流式文

本需要映射到 IM 平台的更新机制（飞书 CardKit 卡片局部更新 vs Telegram
editMessage）；(5) 连接稳定性：WebSocket 断连后自动重连，指数退避避免雪
崩。

12.4 动手练习
目标：理解 IM 适配层的架构
步骤：
1. 阅读 adapters/common/ws-bridge.ts ，画出 WsBridge 的状态机（连接 → 心跳
→ 断连 → 重连）
2. 阅读 adapters/feishu/index.ts ，追踪一条飞书消息从接收到 Agent 回复的完整链
路
3. 查看 adapters/common/attachment/image-block-watcher.ts ，理解如何检测
markdown 图片引用
4. 运行 cd adapters && bun test 查看测试覆盖情况
5. （可选）尝试为一个新的 IM 平台（如 Discord）写一个最小适配器

验证方法：你能画出飞书消息从“用户发送”到“Agent 回复展示在飞书卡片”的完
整时序图。

12.5 延伸阅读
• 飞书开放平台：飞书开发者文档 — IM Bot 开发指南
• grammy：grammy.dev — Telegram Bot 框架
• 适配器模式：Adapter Pattern (Refactoring.Guru) — 设计模式基础

13. 六阶段学习路线总览
在原四阶段基础上扩展为六阶段，增加了“多 Provider 与产品化”和“RAG 与综
合实战”两个阶段。

Phase 1：核心循环（第 1-2 周）
覆盖能力项：#1 Agent Loop、#2 Tool Calling
学习内容

核心文件

投入时间

精读 Agent Loop

src/query.ts

4-5 小时

精读 Tool 接口

src/Tool.ts , src/tools.ts

3-4 小时

精读 API 调用

src/services/api/claude.ts

3-4 小时

动手练习

添加自定义 Tool

3-4 小时

阶段检查清单：
• [ ] 能画出 Agent Loop 的流程图
• [ ] 能解释 AsyncGenerator 模式的优势
• [ ] 能描述一个 Tool 从定义到被调用的完整链路
• [ ] 成功添加了一个自定义 Tool 并被 Agent 调用

Phase 2：工具体系（第 3-5 周）
覆盖能力项：#3 MCP、#4 Context Engineering

学习内容

核心文件

投入时间

阅读 3-4 个 Tool 实现

src/tools/GlobTool/ , BashTool/ 等

4-5 小时

精读 MCP 客户端

src/services/mcp/client.ts

4-5 小时

精读 MCP 服务端

src/entrypoints/mcp.ts

2-3 小时

精读 Prompt 组装

src/constants/prompts.ts , src/context.ts

3-4 小时

动手练习

编写 MCP Server 并接入

5-6 小时

阶段检查清单：
• [ ] 能解释 MCP 的双向架构（Client + Server）
• [ ] 能描述三种 MCP Transport 的适用场景
• [ ] 能解释系统提示词的分层结构
• [ ] 成功编写并接入了自定义 MCP Server

Phase 3：上下文工程（第 6-8 周）
覆盖能力项：#5 Token 优化、#7 Memory
学习内容

核心文件

投入时间

精读 Compact 系统

src/services/compact/

5-6 小时

研究 Token 估算

src/services/tokenEstimation.ts

2-3 小时

研究 Prompt Cache

src/services/api/promptCacheBreakDetection.ts

2-3 小时

精读 Memory 系统

src/services/SessionMemory/ , src/memdir/

4-5 小时

动手练习

触发并观察 autoCompact 流程

3-4 小时

阶段检查清单：
• [ ] 能解释 autoCompact、microCompact、reactiveCompact 的区别和触发条
件
• [ ] 能计算 Prompt Cache 命中率并解释其对成本的影响
• [ ] 能描述记忆的三层架构及其生命周期
• [ ] 触发并观察了 autoCompact 流程

Phase 4：高级架构（第 9-12 周）
覆盖能力项：#6 Streaming、#8 Multi-Agent、#9 安全、#10 LLMOps
学习内容

核心文件

投入时间

精读 Streaming

src/services/api/claude.ts 流式部分

3-4 小时

精读 Multi-Agent

src/tools/AgentTool/ , src/utils/swarm/

5-6 小时

研究权限系统

src/utils/permissions/

3-4 小时

研究 LLMOps

src/cost-tracker.ts , src/services/vcr.ts

3-4 小时

动手练习

追踪子 Agent 执行流程

3-4 小时

阶段检查清单：
• [ ] 能描述三层流式架构（SSE → AsyncGenerator → WebSocket）
• [ ] 能画出父-子 Agent 的时序图
• [ ] 能解释三层安全模型
• [ ] 能解读 API usage 报告并计算成本

Phase 5：多 Provider 与产品化（第 13-15 周）
覆盖能力项：#11 多 Provider 与模型路由、#12 Agent 产品化
学习内容

核心文件

投入时间

精读协议转换层

src/server/proxy/

4-5 小时

精读模型能力系统

src/utils/model/

2-3 小时

精读 IM 适配层

adapters/common/ , adapters/feishu/

4-5 小时

研究 Feature Flag

src/services/analytics/growthbook.ts

2-3 小时

动手练习

配置 OpenAI 兼容 Provider 或写最小 IM 适配器

4-5 小时

阶段检查清单：
• [ ] 能画出 Anthropic ↔ OpenAI 的字段映射图
• [ ] 能解释模型路由和 fallback 降级策略
• [ ] 能描述 IM 适配层的三层架构

• [ ] 能解释 Feature Flag 在 Agent 灰度中的作用

Phase 6：RAG 与综合实战（第 16-18 周）
覆盖能力项：RAG（项目外补充）、综合 Portfolio
学习内容

资源

投入时间

学习 RAG 核心概念

见附录 A

5-6 小时

动手搭建 RAG 系统

LlamaIndex / LangChain 教程

8-10 小时

用 MCP 封装 RAG 服务

将 RAG 暴露为 MCP Server，接入本项目

5-6 小时

Portfolio 整理

将各阶段成果整理为 GitHub 项目

3-4 小时

阶段检查清单：
• [ ] 能设计完整的 RAG 管线（分块 → Embedding → 检索 → 重排序 → 生成）
• [ ] 成功将 RAG 服务通过 MCP 接入本项目
• [ ] GitHub 上有 2-3 个可展示的 AI Agent 相关项目
• [ ] 面试速查表中所有题目都能流利回答

14. 面试速查表
按能力项索引所有面试题，方便面试前快速翻阅。

本文位

能力项

面试题

Agent Loop

描述 AI Agent 核心循环的实现

1.3 Q1

Agent Loop

为什么用 AsyncGenerator

1.3 Q2

Agent Loop

Agent Loop 的错误恢复策略

1.3 Q3

Tool Calling

如何设计可扩展的工具系统

2.3 Q1

Tool Calling

工具过多时如何优化 Token

2.3 Q2

Tool Calling

工具执行失败的处理

2.3 Q3

置

本文位

能力项

面试题

Structured Output

如何实现可靠的结构化输出

2.6

MCP

MCP 的核心架构和通信机制

3.3 Q1

MCP

MCP 的双向性如何体现

3.3 Q2

Computer Use

AI Agent 如何安全地操控桌面 GUI

3.6

MCP vs A2A

MCP 和 A2A 的区别与适用场景

3.7

系统提示词如何结构化

4.3 Q1

Context Engineering vs Prompt Engineering

4.3 Q2

Token 优化

如何管理上下文窗口

5.3 Q1

Token 优化

Prompt Cache 原理及优化

5.3 Q2

Streaming

AI Agent 流式输出架构

6.3 Q1

Streaming

流式连接异常处理

6.3 Q2

Memory

AI Agent 记忆系统设计

7.3 Q1

Memory

非阻塞记忆预取实现

7.3 Q2

Multi-Agent

Multi-Agent 系统设计

8.3 Q1

Multi-Agent

子 Agent 上下文管理

8.3 Q2

Coordinator

如何编排多 Agent 处理复杂项目

8.6

安全

Agent 权限系统设计

9.3 Q1

安全

YOLO 模式的安全平衡

9.3 Q2

Worktree 隔离

如何让 Agent 安全地做代码实验

9.6

LLMOps

性能和成本监控

10.3 Q1

LLMOps

VCR 模式的价值

10.3 Q2

Feature Flag

如何安全地为 Agent 灰度发布

10.6

Context
Engineering
Context
Engineering

置

本文位

能力项

面试题

Agent 评估

如何评估 AI Agent 的输出质量

10.7

多 Provider

如何设计支持多 LLM Provider 的系统

11.3 Q1

协议转换

Anthropic 和 OpenAI 的 Tool Use 区别

11.3 Q2

Agent 产品化

如何让 Agent 接入企业 IM

12.3 Q1

Agent 产品化

IM 适配的工程挑战

12.3 Q2

RAG

如何设计生产级 RAG 系统

附录 A

RAG

RAG vs Fine-tuning 的选型

附录 A

Agent 评测

如何评估 AI Agent 的输出质量

10.7

Agent 评测

如何搭建 Agent 持续评测流水线

10.7

Agent 评测
Python 生态

Agent 评测框架对比（MLflow/DeepEval/
agentevals）
如何将 TypeScript Agent 概念迁移到 Python

置

10.7
15

15. Python 生态映射（面试与工程必备）
背景： 2026 年 AI Agent 工程师岗位 90%+ 要求 Python 技能。通过 cc-haha
（TypeScript）学习概念后，需要将每个概念对应到 Python 主流框架，才能在
面试和实际工作中无缝切换。

概念 → Python 框架对照表
cc-haha 概念

Python 对应框架/库

说明

Agent Loop（ query.ts /

LangGraph Agent / CrewAI Agent /

Agent 执

queryLoop() ）

AutoGen

行循环

Tool 系统（ src/Tool.ts ,

LangChain Tools / OpenAI Function

工具定义

buildTool() ）

Calling

与注册

cc-haha 概念

Python 对应框架/库

说明
官方

MCP Client/Server（ src/
services/mcp/ ）

MCP Python SDK ( mcp )

Python
MCP 客户
端

System Prompt 组装

LangChain PromptTemplate / Jinja2

提示词模

模板

板化

Compact 压缩（ src/

LangChain

对话摘要

services/compact/ ）

ConversationSummaryBufferMemory

压缩

（ src/constants/
prompts.ts ）

Context 管理（ src/
context.ts ）

Model Routing（ src/
server/proxy/ ）

上下文摄
LlamaIndex IngestionPipeline

压缩管线
LiteLLM Router / fallback 配置

Token 估算
（ src/services/

tiktoken / LiteLLM cost tracking

由与降级

量与成本
计算

Memory 记忆（ src/
LangChain Memory / Mem0 / Letta

SessionMemory/ ）

Multi-Agent（ src/tools/

多模型路

Token 计

tokenEstimation.ts ）

services/

入/组装/

分层记忆
管理

CrewAI / AutoGen / LangGraph Multi-

多 Agent

Agent

编排

Agent 间通信

Google A2A Protocol / LangGraph

Agent 间

（SendMessageTool）

Send API

通信

流式输出（ src/services/

LangChain Streaming / FastAPI + SSE

流式响应

api/claude.ts SSE）

/ OpenAI stream=True

处理

AgentTool/ , src/utils/
swarm/ ）

Prompt Caching

Anthropic cache_control / OpenAI
caching

Python
SDK 原生
支持

cc-haha 概念

Python 对应框架/库

说明

权限系统（ src/utils/

Guardrails AI / NVIDIA NeMo

Agent 安

permissions/ ）

Guardrails

全护栏

Feature Flag（ src/
services/analytics/
growthbook.ts ）

Agent 评测

GrowthBook Python SDK /
LaunchDarkly
MLflow / Ragas / DeepEval /

Agent 质

agentevals

量评测

Worktree 隔离
（ src/tools/

灰度发布

Docker SDK / E2B / Modal Sandbox

EnterWorktreeTool/ ）

代码执行
沙箱

建议的 Python 实践项目
学完 cc-haha 的每个能力项后，尝试用 Python 实现一个简化版：
1. LangGraph Agent：实现一个带工具调用和降级策略的 Agent Loop
2. MCP Server：用 Python MCP SDK 实现一个自定义工具服务器并接入
LangChain
3. RAG Pipeline：用 LlamaIndex 实现 Dense + Sparse 混合检索 + MMR 重排序
4. Multi-Agent：用 CrewAI 实现 Supervisor-Worker 模式的任务分解
5. Streaming App：用 FastAPI + SSE 实现一个支持流式输出的 Agent API
6. Agent Eval：用 DeepEval 或 MLflow 为你的 Agent 搭建持续评测流水线

附录 A：RAG 补充学习路径
本项目不包含 RAG 模块，但 RAG 是 AI Agent 工程师 JOB MODEL 中的必备能
力。以下提供独立的补充学习路径。

A.1 为什么必须补充 RAG
2026 年几乎所有 AI Agent 岗位 JD 都要求 RAG 经验。RAG 解决了 LLM 的核心限制
——训练数据截止、无法访问私有知识库。一个没有 RAG 能力的 Agent 工程师，就像
一个不会数据库的后端工程师。

A.2 RAG 核心概念速查
文档 → 分块(Chunking) → 向量化(Embedding) → 存入向量库
↓
用户查询 → 向量化 → 相似度检索 → 重排序(Re-ranking) → 注入 Prompt → LLM 生成

五个核心技术点：
技术点

说明

关键选型

Chunking

文档分块策略

固定大小 / 语义分块 / 递归字符分割

Embedding

文本向量化

Vector DB

向量存储与检索

Retrieval

检索策略

Re-ranking

结果重排序

GraphRAG

OpenAI text-embedding-3-small 、BGE、
Cohere Embed
Milvus、Pinecone、Weaviate、
Chroma
向量检索、全文检索、混合检索
（Hybrid）
Cohere Rerank、BGE Reranker、CrossEncoder

知识图谱增强检索

Neo4j + LlamaIndex / Microsoft

（2026 趋势）

GraphRAG

A.3 面试高频题
Q1：如何设计一个生产级 RAG 系统？
回答思路：分三层——(1) 离线管线：文档解析（PDF/Markdown/HTML）→ 语义分块
（控制 chunk 大小，保留上下文重叠）→ Embedding → 存入向量库（建立 HNSW
索引）；(2) 在线检索：用户 query Embedding → Top-K 向量检索 + BM25 全文检索
→ 混合排序 → Cross-Encoder 重排序 → 取 Top-N 注入 system prompt；(3) 质量保
证：Faithfulness 评估（生成是否忠于检索内容）、Relevance 评估（检索是否切
题）、Hallucination 检测。
Q2：RAG vs Fine-tuning vs GraphRAG，什么场景用哪个？
回答思路：(1) RAG 适合知识更新频繁、需要溯源的场景（如企业知识库、客服系
统）；(2) Fine-tuning 适合需要改变模型“行为风格”的场景（如特定领域术语、输

出格式）；(3) GraphRAG 适合知识之间存在复杂实体关系的场景（如法律条文引用网
络、供应链关系图谱、科研论文引用链）；(4) 三者可以结合——用 Fine-tuning 让模
型更擅长利用 RAG 检索的内容，用 GraphRAG 增强实体关系感知。

A.4 与本项目的结合练习
目标：用 MCP 将 RAG 服务接入本项目
步骤：
1. 使用 LlamaIndex 或 LangChain 搭建一个本地 RAG 服务（索引本项目的文档）
2. 将 RAG 服务封装为 MCP Server（Stdio 模式），暴露 search_docs 工具
3. 在项目的 MCP 配置中注册你的 RAG Server
4. 启动项目，让 Agent 使用你的 RAG 工具回答关于项目文档的问题
验证方法：Agent 通过 MCP 调用你的 RAG 服务，检索到相关文档并基于内容回答问
题。

A.5 推荐资源
• LlamaIndex 官方教程：llamaindex.ai — 最流行的 RAG 框架
• RAG 评估框架 RAGAS：ragas.io — RAG 质量评估标准
• Anthropic RAG 指南：Retrieval augmented generation — 官方 RAG 最佳实践
• Chunking 策略详解：Chunking Strategies for LLM Applications — Pinecone
出品的分块策略指南

附录 B：多模态处理基础
B.1 项目中的多模态支持
本项目已有的多模态处理能力：
模态

实现文件

说明
图像预处理：格式检测、尺寸约

图像输入

src/utils/imageResizer.ts

束、base64 编码大小控制
（800+ 行）

图像校验

src/utils/imageValidation.ts

API 限额校验：base64 大小 vs
API_IMAGE_MAX_BASE64_SIZE

模态

实现文件

图像格式转换

src/server/proxy/transform/

语音输入

src/services/voice.ts

Computer Use

说明
Anthropic image ↔ OpenAI
image_url 格式互转

Push-to-talk 语音采集（ audiocapture-napi ）

src/vendor/computer-usemcp/

桌面截图 + GUI 操控

B.2 面试考点
Q：如何在 Agent 中处理多模态输入？有什么工程挑战？
回答思路：核心挑战有三——(1) 成本控制：一张图片可能消耗数千 Token，需要预处
理（缩放、压缩）控制在 API 限额内；(2) 格式适配：不同 Provider 的图像格式不同
（Anthropic 用 base64 source，OpenAI 用 data URL），需要转换层；(3) 错误处
理：图像处理可能因 sharp 库缺失、内存不足、超时等原因失败，需要分类处理并优
雅降级。

附录 C：Portfolio 项目建议
每个阶段建议产出一个可展示的独立项目，用于面试和简历。

阶段

Portfolio 项目
Mini Agent CLI：从

Phase 1-2

零实现 200 行的
Agent Loop + Tool
Calling
MCP Server：对接

Phase 3-4

真实 API（GitHub
Issues / Jira / 数据
库）

技术要点

预计时间

AsyncGenerator、
Zod Schema、

1 周末

ReAct 模式

MCP SDK、Stdio
Transport、工具
设计

1 周末

阶段

Portfolio 项目

技术要点

预计时间

RAG 知识库问答
Phase 5

Agent：索引企业

Embedding、向

文档，支持多轮对

量检索、对话管理

2 周末

话
Multi-Agent 协同系
Phase 6

统：Code Review
Agent + Test Agent
联动

Coordinator 模
式、Worktree 隔

2 周末

离、结果合并

面试展示建议
• 将项目放在 GitHub，写清楚 README（架构图 + 快速开始 + 设计决策）
• 准备 5 分钟 Demo 视频或 GIF 录屏
• 在面试中用“我在这个项目中遇到了 X 问题，用了 Y 方案解决”的方式展示技术
深度
• 引用本项目（cc-haha）的设计模式作为“我研究过的生产级实现”

附录 D：Fine-tuning 入门资源
Fine-tuning 在偏工程侧的 AI Agent 岗位中优先级较低，但作为能力储备建议了
解基础概念。

D.1 核心概念
• Full Fine-tuning：更新全部参数，需要大量 GPU 资源
• LoRA (Low-Rank Adaptation)：只训练低秩适配矩阵，大幅降低资源需求
• QLoRA：量化 + LoRA，在消费级 GPU 上即可微调
• RLHF / DPO：基于人类反馈的对齐训练

D.2 推荐学习路径
1. 概念入门：Hugging Face PEFT 文档 — LoRA/QLoRA 官方教程
2. 动手实践：Unsloth — 2x 加速的 Fine-tuning 框架

3. OpenAI Fine-tuning：Fine-tuning Guide — 最简单的入门方式（API 调用即
可）
4. 评估方法：Fine-tune 后必须用 eval set 验证效果，避免过拟合

