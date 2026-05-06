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
| [agent-loop.md](agent-loop.md) | 🟡 进行中 | Agent Loop 设计 — query()/queryLoop()、循环终止条件、工具执行三阶段、附件注入与记忆预取 |
| tool-calling.md | ⬜ 待开始 | Tool Calling / Function Calling — Tool 系统架构、Zod Schema、权限控制 |
| prompt-engineering.md | ⬜ 待开始 | Prompt Engineering — System Prompt、Memory、自定义 Prompt |
| context-management.md | 🟡 进行中 | 上下文管理与 Token 优化 — AutoCompact、MicroCompact、Session Memory Compact、附件注入 |
| mcp-integration.md | ⬜ 待开始 | MCP 协议集成 — Client/Server、Tool 代理、连接管理 |
| multi-agent.md | ⬜ 待开始 | Multi-Agent — AgentTool、Coordinator、forkedAgent |
| permissions.md | ⬜ 待开始 | 权限系统 — Permission Mode、Tool Permission、hook 拦截 |
| streaming.md | ⬜ 待开始 | Streaming — SSE、AsyncGenerator、StreamingToolExecutor |
| observability.md | ⬜ 待开始 | 可观测性 — 成本追踪、日志、VCR 录制回放、Feature Flag |
| desktop-app.md | ⬜ 待开始 | 桌面应用 — Tauri、WebSocket、Zustand 状态管理 |
| hooks.md | ⬜ 待开始 | Hooks 系统 — 生命周期钩子、Stop Hooks、Post-Sampling Hooks |
| memory-system.md | ⬜ 待开始 | Memory 系统 — SessionMemory、AutoMemory、Memory Prompt |
| im-adapters.md | ⬜ 待开始 | IM 适配器 — 飞书、Telegram 集成 |
| testing.md | ⬜ 待开始 | 测试策略 — VCR、单元测试、集成测试 |

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
