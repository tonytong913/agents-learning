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

## [2026-05-06] query | agent-loop 阶段2-3 注入点精读

**范围**：`src/query.ts` (阶段 2-3 注入段)、`src/utils/attachments.ts` (getAttachmentMessages / getAttachments / startRelevantMemoryPrefetch)、`src/utils/messageQueueManager.ts` (命令优先级)

**讨论内容**：

1. **getAttachmentMessages**：两阶段并行附件生成（userInputAttachments → allThreadAttachments），`maybe()` 独立容错包装，命令队列排水（queuedCommandsSnapshot 的 sleep 分支）
2. **命令优先级系统**：`now/next/later` 三级，`enqueue()`(next) vs `enqueuePendingNotification()`(later)，设计意图是防止系统通知饥饿用户输入
3. **memory prefetch**：启动在循环外 (输入不变)、消费在循环内 (非阻塞 poll + readFileState 累积去重)，`using` + `Symbol.dispose` 自动清理，两层去重 (readFileState vs alreadySurfaced，compact 为重置点)
4. **Turn vs 迭代**：Turn = 一次 query() 调用，迭代 = while(true) 每圈，一个 turn 可含多轮迭代

**产出**：更新 agent-loop.md §6.5-6.6（注入点 + 消息组装）

**待继续**：skill discovery 注入

---

## [2026-05-06] query | 上下文管理与 Token 优化

**范围**：`src/services/compact/` 全部、`src/query.ts` 阶段 2-3 附件注入段

**讨论内容**：

1. **AutoCompact 触发与断路器**：有效窗口计算（context window - max_output - 13k buffer），shouldAutoCompact 的 5 层拒绝条件，3 次连续失败断路器
2. **Session Memory Compact**：零 LLM 成本的裁剪方案，calculateMessagesToKeepIndex 的 minTokens/minTextBlockMessages/maxTokens 三级约束
3. **MicroCompact 两种路径**：Cached MC（cache_edits 不破坏缓存前缀）vs Time-Based MC（缓存已冷时直接替换）
4. **迭代入口四级压缩管道**：toolResultBudget → snipCompact → microCompact → contextCollapse → autoCompact
5. **阶段 2-3 附件注入**：getAttachmentMessages（命令 → 附件）、memory prefetch 消费（non-blocking + readFileState 去重）、skill discovery 注入（并行预取 >98% 命中率）
6. **Context Collapse / Reactive Compact**：内部功能（外部 build 为 stub），collapse 的 90% commit / 95% blocking-spawn 两阶段设计

**产出**：context-management.md — 核心笔记

**待继续**：OpenClaw compaction.ts / context-engine/ 对照

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
