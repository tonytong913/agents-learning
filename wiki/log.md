# 学习日志

> 记录每次学习会话的活动，append-only。

---

## [2026-05-05] wiki | 合并双项目学习

- 将 learning 文件夹移至 `~/Documents/`，重命名为 `agents-learning`
- 添加 [openclaw-learning-path.md](../raw/openclaw-learning-path.md) 作为第二个目标项目
- 重命名源文件：`LEARNING_PATH.md` → `cc-haha-learning-path.md`
- 更新 index.md 为双项目目录，交叉对照 cc-haha 与 OpenClaw
- 创建 [CLAUDE.md](../CLAUDE.md) 作为 wiki 维护 schema

## [2026-05-05] ingest | 初始安装

- 创建 wiki 目录结构（`raw/` + `wiki/`）
- 添加 `cc-haha-learning-path.md`（来自 cc-haha 项目）
- 添加 `openclaw-learning-path.md`（来自 openclaw 项目）

---

## [2026-05-05] query | needsFollowUp = true 分支精读

**范围**：`src/query.ts` 工具执行段、`src/services/tools/StreamingToolExecutor.ts`、`src/services/tools/toolOrchestration.ts`

**讨论内容**：

1. 工具执行三阶段：流式边收边执行 → 收集剩余结果 → 组装消息 continue
2. StreamingToolExecutor 内部机制：状态机、并发控制、Bash 错误级联、进度推送
3. runTools() 兜底路径：partitionToolCalls 分区策略、runToolsConcurrently/Serially
4. 两种路径对比：时机、并发策略、错误处理
5. 工具执行期间中断处理：abort、hook_stopped、interruptBehavior

**产出**：更新 agent-loop.md §6

**待继续**：阶段2-3 之间的附件收集、memory prefetch、AutoCompact 完整流程

---

## [2026-05-04] query | Agent Loop 第一轮精读

**范围**：`src/query.ts` query() / queryLoop()、`src/QueryEngine.ts` processUserInput()、`src/utils/queryHelpers.ts` isResultSuccessful()

**讨论内容**：

1. query() vs queryLoop() 的区别和分工
2. queryLoop while(true) 的所有终止条件（10+ 种 reason）
3. `blocking_limit` 的触发机制 — 调用 API 前的防御性 token 检查
4. `image_error` 的两个触发点 — 本地校验 (ImageSizeError/ImageResizeError) 和 API 拒绝后恢复失败
5. querySource 的填充方式 — 硬编码字符串，30+ 种场景
6. compact agent 和 session_memory agent 的工作流程
7. `return { reason }` 的消费情况 — 无人捕获，仅用于日志/遥测
8. isResultSuccessful() 的三层判断逻辑

**产出**：agent-loop.md — 核心笔记

**待继续**：
- needsFollowUp=true 分支（工具执行、附件收集、memory prefetch）
- StreamingToolExecutor 机制
- runTools() 并发策略
- AutoCompact 完整流程

---

## [2026-05-04] query | 初始化 Wiki

基于 cc-haha-learning-path.md 第一部分（Agent Loop 设计）创建学习笔记 wiki。
采用 Karpathy LLM Wiki 模式：`raw/` 放源文件，`wiki/` 放笔记。
