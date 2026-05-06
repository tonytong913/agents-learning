# Agents Learning Wiki — 维护指引

## 架构

```
learning/
├── CLAUDE.md          ← 本文件：wiki 结构、约定、工作流
├── raw/               ← 不可变源文件（只读，不修改）
│   ├── cc-haha-learning-path.md
│   └── openclaw-learning-path.md
└── wiki/              ← LLM 生成的 markdown 笔记
    ├── index.md       ← 内容目录（所有页面 + 摘要 + 状态）
    ├── log.md         ← 按时间追加的操作日志
    └── *.md           ← 各能力项的实体/概念笔记
```

## 核心操作

### 1. Ingest（摄入新源）
- 新源文件放入 `raw/`，命名语义化（如 `xxx-learning-path.md`）
- 在 `wiki/index.md` 源文件索引中注册
- 讨论关键要点后，产出笔记页并更新 index 目录
- 在 `wiki/log.md` 末尾追加 ingest 记录

### 2. Query（查询与学习）
- 对照源文件按能力项逐项深入学习
- 笔记页保持交叉引用：两个项目的同类实现互相链接对照
- 每个笔记页顶部标注：状态 (🟡/🟢/⬜)、最后更新日期、相关源文件路径
- 有价值的问答产出应归档为新笔记页

### 3. Lint（健康检查）
- 定期检查：是否有矛盾论述、过时内容、孤立页面、缺失交叉引用
- 检查 index.md 目录与实际页面是否一致
- 检查所有链接是否有效

## 笔记页约定

- 每个页面聚焦一个能力项或概念
- 顶部 frontmatter 区域：状态、最后更新日期、相关源文件
- 交叉引用两个项目的实现差异时使用 `> 💡 对比：` 块引用
- 状态流转：⬜ 待开始 → 🟡 进行中 → 🟢 完成

## 命名约定

- 页面名使用 kebab-case，如 `agent-loop.md`、`tool-calling.md`
- 源文件名包含项目标识，如 `cc-haha-learning-path.md`
