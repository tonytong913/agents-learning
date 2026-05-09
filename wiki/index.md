# Agents Learning Wiki

> 基于两个 AI Agent 项目的源码学习笔记。按 AI Agent 工程师能力项组织，交叉对照两个项目的设计决策与实现方式。
> 遵循 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式：增量更新、交叉引用、持续维护。

## 目标项目

| 项目 | 学习路线 | 语言 |
|------|----------|------|
| cc-haha | [cc-haha-learning-path.md](../raw/cc-haha-learning-path.md) | TypeScript |
| OpenClaw | [openclaw-learning-path.md](../raw/openclaw-learning-path.md) | TypeScript |

## 目录

### cc-haha

| 页面 | 状态 | 说明 |
|------|------|------|
| [agent-loop.md](agent-loop.md) | 🟢 完成 | 路线 §1 Agent Loop 设计 — query()/queryLoop()、循环终止条件、工具执行三阶段、附件注入与记忆预取 |
| [tool-calling.md](tool-calling.md) | 🟢 完成 | 路线 §2 Tool Calling / Function Calling — Tool 接口、Zod / JSON Schema、runTools 编排、GlobTool 生命周期、BashTool 高风险路径、ToolSearch 渐进式加载、StructuredOutput、Fine-grained tool streaming |
| [mcp-integration.md](mcp-integration.md) | 🟡 进行中 | 路线 §3 MCP 协议集成 — why/what/how、配置合并与去重、Transport、MCPTool 适配、OAuth、list_changed、MCP Skills、Channel Permissions、Computer Use |
| [context-management.md](context-management.md) | 🟡 进行中 | 路线 §4-§5 Context Engineering / 上下文窗口管理与 Token 优化 — System Prompt 注入、AutoCompact、MicroCompact、Session Memory Compact、附件注入 |
| streaming.md | ⬜ 待开始 | 路线 §6 Streaming 与实时交互 — SSE、AsyncGenerator、StreamingToolExecutor、WebSocket 转发 |
| memory-system.md | ⬜ 待开始 | 路线 §7 Memory 系统 — SessionMemory、extractMemories、memdir、非阻塞记忆预取 |
| multi-agent.md | ⬜ 待开始 | 路线 §8 Multi-Agent — AgentTool、forkSubagent、Swarm、Coordinator |
| permissions.md | ⬜ 待开始 | 路线 §9 安全、权限与沙箱 — Permission Mode、Bash 分类、YOLO、Worktree 隔离 |
| observability.md | ⬜ 待开始 | 路线 §10 LLMOps / 可观测性 — 成本追踪、API 日志、VCR、Feature Flag、Agent 评估 |
| model-routing.md | ⬜ 待开始 | 路线 §11 多 Provider 与模型路由 — Anthropic/OpenAI 协议转换、模型能力、fallback 降级 |
| productization-channels.md | ⬜ 待开始 | 路线 §12 Agent 产品化：多通道接入 — Desktop、WebSocket、飞书/Telegram 适配、附件和流式消息映射 |

### cc-haha 补充能力

| 页面 | 状态 | 说明 |
|------|------|------|
| python-ecosystem.md | ⬜ 待开始 | 路线 §15 Python 生态映射 — 将 cc-haha TypeScript 概念迁移到 LangGraph、MCP Python SDK、LiteLLM、DeepEval 等 |
| rag.md | ⬜ 待开始 | 附录 A RAG 补充学习路径 — 项目外能力，通过 MCP 接入 cc-haha 做综合实战 |
| multimodal.md | ⬜ 待开始 | 附录 B 多模态处理基础 — 图像输入、Provider 格式转换、语音输入、Computer Use |
| portfolio.md | ⬜ 待开始 | 附录 C-D Portfolio 与 Fine-tuning 储备 — 阶段项目、展示材料、LoRA/QLoRA 基础 |

### OpenClaw

| 页面 | 状态 | 说明 |
|------|------|------|
| openclaw-overview.md | ⬜ 待开始 | OpenClaw 架构概览 — 项目结构、与 cc-haha 的异同 |

## 图例

- 🟡 进行中 — 正在学习，笔记不完整
- 🟢 完成 — 核心内容已覆盖
- ⬜ 待开始 — 尚未开始

## 源文件索引

| 源文件 | 路径 | 说明 |
|--------|------|------|
| cc-haha-learning-path.md | [raw/cc-haha-learning-path.md](../raw/cc-haha-learning-path.md) | cc-haha 学习路线图，14 大能力项和阅读指引 |
| openclaw-learning-path.md | [raw/openclaw-learning-path.md](../raw/openclaw-learning-path.md) | OpenClaw 学习路线图，完整源码阅读指引 |
