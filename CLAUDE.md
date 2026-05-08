# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

# Agents Learning Wiki — 维护指引

这是一个基于 AI Agent 项目源码的学习笔记 wiki。按 AI Agent 工程师能力项组织，交叉对照 cc-haha 与 OpenClaw 两个项目的设计决策与实现方式。遵循 [Karpathy LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式：增量更新、交叉引用、持续维护。

## 重要：本仓库不包含目标项目源码

本仓库只存放学习笔记。wiki 笔记中引用的源码路径（如 `src/query.ts`、`src/agents/tool-catalog.ts`）**均指目标项目（cc-haha / OpenClaw）仓库内的文件**，不在本仓库中。本仓库只有 `raw/`（学习路线图源文件）和 `wiki/`（笔记输出）。

## 架构

```
agents-learning/
├── CLAUDE.md          ← 本文件：wiki 结构、约定、工作流
├── raw/               ← 不可变源文件（只读，不修改）
│   ├── cc-haha-learning-path.md   ← cc-haha 学习路线图，14 大能力项
│   └── openclaw-learning-path.md  ← OpenClaw 学习路线图，11 大能力项
└── wiki/              ← LLM 生成的 markdown 笔记
    ├── index.md       ← 内容目录（所有页面 + 摘要 + 状态）— 权威状态源
    ├── log.md         ← 按时间追加的操作日志（append-only）
    ├── agent-loop.md  ← 示例笔记页
    └── *.md           ← 各能力项的实体/概念笔记
```

## 目标项目

| 项目 | 语言 | 源文件 | 本地仓库 | GitHub |
|------|------|--------|----------|--------|
| cc-haha | TypeScript | `raw/cc-haha-learning-path.md` | `~/repos/cc-haha` | https://github.com/NanmiCoder/cc-haha |
| OpenClaw | TypeScript | `raw/openclaw-learning-path.md` | `~/repos/openclaw` | https://github.com/openclaw/openclaw |

## 核心操作

### 1. 学习会话（Query）

用户请求学习某个能力项时的标准流程：

1. **读取源文件**：从 `raw/` 中读取对应学习路线图的相关章节，了解该能力项在两个项目中的模块映射
2. **讨论关键设计**：对照源文件中的核心模块和设计思想，展开讨论
3. **产出/更新笔记**：将讨论内容归档到 `wiki/<topic>.md`，遵循笔记页约定
4. **更新目录**：在 `wiki/index.md` 中更新对应页面的状态和说明
5. **追加日志**：在 `wiki/log.md` 末尾追加本次学习记录

笔记页应保持交叉引用：两个项目对同一能力项的实现差异使用 `> 💡 对比：` 块引用互相对照。

### 2. Ingest（摄入新源）

- 新源文件放入 `raw/`，命名语义化（如 `xxx-learning-path.md`）
- 在 `wiki/index.md` 的源文件索引中注册
- 讨论关键要点后，产出笔记页并更新 index 目录
- 在 `wiki/log.md` 末尾追加 ingest 记录

### 3. Lint（健康检查）

- 定期检查：是否有矛盾论述、过时内容、孤立页面、缺失交叉引用
- 检查 `wiki/index.md` 目录与实际页面是否一致
- 检查所有链接是否有效

## 笔记页约定

- 每个页面聚焦一个能力项或概念
- 顶部元信息行（非 YAML frontmatter）：状态、最后更新日期、相关源文件
- 交叉引用两个项目的实现差异时使用 `> 💡 对比：` 块引用
- 状态流转：⬜ 待开始 → 🟡 进行中 → 🟢 完成
- 页面名使用 kebab-case，如 `agent-loop.md`、`tool-calling.md`
- 源文件名包含项目标识，如 `cc-haha-learning-path.md`

## 全局状态

- **当前进度和页面目录以 `wiki/index.md` 为准**，该文件是权威状态源。本文件不重复维护状态快照。
- **操作历史见 `wiki/log.md`**（append-only）。

## 图例

- 🟡 进行中 — 正在学习，笔记不完整
- 🟢 完成 — 核心内容已覆盖
- 待开始 — 尚未开始
