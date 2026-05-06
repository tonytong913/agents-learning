# Agents Learning

> AI Agent 工程学习笔记 — 基于 [cc-haha](https://github.com/NanmiCoder/cc-haha) 和 [OpenClaw](https://github.com/openclaw/openclaw) 两个开源项目的源码阅读，按 AI Agent 工程师能力项组织，交叉对照设计决策与实现方式。

## 这是什么

一个以 LLM Wiki 模式维护的学习仓库。笔记按能力项（Agent Loop、Tool Calling、Context Management 等）组织，每个能力项对照 cc-haha 和 OpenClaw 两个项目的实现差异进行讨论。

不包含目标项目源码，只存放学习笔记。

## 结构

```
agents-learning/
├── raw/                    # 不可变源文件：学习路线图
│   ├── cc-haha-learning-path.md
│   └── openclaw-learning-path.md
├── wiki/                   # LLM 生成的 markdown 笔记
│   ├── index.md            # 内容目录 + 状态追踪（权威状态源）
│   ├── log.md              # 操作日志（append-only）
│   └── *.md                # 各能力项笔记
├── CLAUDE.md               # Claude Code 工作指引
└── README.md
```

## 能力项覆盖

| 领域 | 状态 |
|------|------|
| Agent Loop 设计 | 🟡 进行中 |
| 上下文管理与 Token 优化 | 🟡 进行中 |
| Tool Calling / Function Calling | ⬜ 待开始 |
| Prompt Engineering | ⬜ 待开始 |
| MCP 协议集成 | ⬜ 待开始 |
| Multi-Agent | ⬜ 待开始 |
| 权限系统 | ⬜ 待开始 |
| Streaming | ⬜ 待开始 |
| 可观测性 | ⬜ 待开始 |
| Hooks 系统 | ⬜ 待开始 |
| Memory 系统 | ⬜ 待开始 |
| 测试策略 | ⬜ 待开始 |
| OpenClaw 架构概览 | ⬜ 待开始 |

详见 [`wiki/index.md`](wiki/index.md)。

## 目标项目

| 项目 | 语言 | 仓库 |
|------|------|------|
| cc-haha | TypeScript | [NanmiCoder/cc-haha](https://github.com/NanmiCoder/cc-haha) |
| OpenClaw | TypeScript | [openclaw/openclaw](https://github.com/openclaw/openclaw) |

## 用法

配合 [Claude Code](https://claude.ai/code) 使用。`CLAUDE.md` 定义了标准学习会话流程：

1. 选取一个能力项，从 `raw/` 学习路线图定位相关模块
2. 阅读源码，讨论关键设计
3. 产出/更新 `wiki/<topic>.md` 笔记
4. 更新 `wiki/index.md` 目录状态
5. 追加 `wiki/log.md` 操作日志

## 模式

遵循 [Karpathy LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式：增量更新、交叉引用、持续维护。

## License

CC0 — 学习笔记，自由使用。
