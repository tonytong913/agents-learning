# 学习日志

> 记录每次学习会话的活动，append-only。

---

## [2026-05-06] query | skill discovery 注入精读

**范围**：`src/query.ts`（启动 + 消费点）、`src/utils/attachments.ts`（turn-0 阻塞路径 + skill_listing 退让 + skipSkillDiscovery guard）、`src/services/skillSearch/`（stub 机制）、`src/tools/SkillTool/SkillTool.ts`（调用链路）

**讨论内容**：

1. **skilling_listing vs skill_discovery**：两套并存机制的分工 — listing 提供全量概览（bundled+MCP），discovery 做上下文相关匹配（用户/项目/插件 200+）
2. **异步 prefetch 架构**：startSkillDiscoveryPrefetch 在迭代开始启动，与 LLM + 工具执行并行，post-tools 消费，>98% 命中率（AKI 250ms vs turn 2-30s）
3. **Turn-0 阻塞路径**：首轮无并行工作可重叠，getTurnZeroSkillDiscovery 在 userInputAttachments 中同步调用
4. **skipSkillDiscovery guard**：SKILL.md 展开内容（110KB+）误触发 ~3.3s 查询的防御（session 13a9afae）
5. **skill_listing delta 机制**：Map<agentKey, Set> 跟踪已发送技能，后续只发增量
6. **Feature gate + DCE**：条件 require + Proxy stub + import type 确保外部 build 零泄漏
7. **Subagent 差异**：subagent_spawn signal，第一轮无 discovery（visible turn 1）
8. **三层注入次序**：getAttachmentMessages → memory prefetch → skill discovery（延迟最大化）

**产出**：更新 agent-loop.md §6.5（注入点③：skill discovery 完整分析），agent-loop 待学习全部完成

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

---

## [2026-05-08] query | Tool Calling / GlobTool 生命周期

**范围**：`src/Tool.ts`, `src/tools.ts`, `src/utils/api.ts`, `src/query.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/toolExecution.ts`, `src/tools/GlobTool/GlobTool.ts`, `src/tools/GlobTool/prompt.ts`

**讨论内容**：

1. **Tool 接口分层**：模型可见 schema、执行入口、安全权限、UI/transcript、编排元信息共同组成工具契约
2. **buildTool 默认值**：`isConcurrencySafe=false`、`isReadOnly=false` 的保守默认策略，避免新工具误并发或误按只读处理
3. **API schema 转换**：`toolToAPISchema()` 将 `prompt()` 和 Zod `inputSchema` 转成 Anthropic tool schema
4. **GlobTool 生命周期**：注册 → schema 暴露 → 模型 `tool_use` → `runTools()` 分批 → `runToolUse()` 校验/权限/执行 → `tool_result` 回到下一轮上下文
5. **GlobTool 设计细节**：只读并发安全、路径语义校验、读取权限检查、结果相对路径化、截断提示
6. **与 BashTool 的差异**：Glob 风险固定较低；Bash 根据具体 command 动态判断只读和并发安全

**产出**：新增 `wiki/tool-calling.md`，更新 `wiki/index.md` 中 Tool Calling 状态为进行中

**待继续**：BashTool 高风险路径、ToolSearchTool / defer_loading、SyntheticOutputTool 结构化输出

---

## [2026-05-08] query | ToolSearchTool 渐进式加载精读

**范围**：`src/tools/ToolSearchTool/ToolSearchTool.ts`, `src/tools/ToolSearchTool/prompt.ts`, `src/utils/toolSearch.ts`, `src/services/api/claude.ts`, `src/utils/attachments.ts`, `src/utils/messages.ts`, `src/services/tools/toolExecution.ts`, `src/services/compact/compact.ts`

**讨论内容**：

1. **deferred 工具判定**：MCP tools 与 `shouldDefer=true` 工具延迟发送 schema，`alwaysLoad=true`、`ToolSearchTool` 和首轮关键工具保持直接加载
2. **ToolSearch 启用条件**：模型 `tool_reference` 支持、`ToolSearchTool` 可用性、`ENABLE_TOOL_SEARCH` 模式、auto 阈值共同决定是否启用
3. **filteredTools 策略**：非 deferred 工具、`ToolSearchTool`、已 discovered 的 deferred 工具进入 API tools；未 discovered 的 deferred 工具只公布名字
4. **工具名公告机制**：旧路径注入 `<available-deferred-tools>`，新路径使用 `deferred_tools_delta` attachment 只发送新增/移除变化
5. **搜索算法**：`select:` 精确选择；关键词搜索基于工具名拆分、MCP server/action、`searchHint`、prompt/description 加权打分
6. **tool_reference 闭环**：`ToolSearchTool` 返回 `tool_reference`，下一轮 `extractDiscoveredToolNames()` 扫描历史并触发完整 schema 加载
7. **兜底与 compact**：`buildSchemaNotSentHint()` 指导模型先加载 schema；compact boundary 保存 `preCompactDiscoveredTools` 防止压缩后丢失已加载状态

**产出**：更新 `wiki/tool-calling.md` §6（渐进式工具加载）

**待继续**：BashTool 高风险路径、SyntheticOutputTool 结构化输出、Fine-grained tool streaming

---

## [2026-05-09] query | BashTool 高风险路径精读

**范围**：`src/tools/BashTool/BashTool.tsx`, `src/tools/BashTool/bashPermissions.ts`, `src/tools/BashTool/readOnlyValidation.ts`, `src/tools/BashTool/pathValidation.ts`, `src/tools/BashTool/shouldUseSandbox.ts`, `src/tools/BashTool/sedValidation.ts`, `src/tools/BashTool/sedEditParser.ts`, `src/tools/BashTool/commandSemantics.ts`, `src/tools/BashTool/destructiveCommandWarning.ts`

**讨论内容**：

1. **只读与并发安全**：`BashTool.isConcurrencySafe(input)` 依赖具体 command 的 `isReadOnly(input)`，不同于 `GlobTool` 恒定只读并发安全
2. **AST / 安全解析**：先判断 Bash 命令能否被可靠理解；`simple` 才继续自动判断，`too-complex` 转人工确认，tree-sitter 不可用时退回 legacy 检查
3. **规则匹配不对称**：deny / ask 更难绕过，allow 更难误放行；环境变量和 wrapper 的剥离策略按规则方向不同处理
4. **sandbox auto-allow**：沙箱自动允许只覆盖未命中显式 deny/ask 的命令，显式规则仍优先
5. **路径和重定向约束**：`cd + write`、含 shell expansion 的 redirect target、process substitution 等场景保守转 `ask`
6. **sed 特例**：read-only sed、in-place sed 编辑、`_simulatedSedEdit` 预演写入闭环
7. **执行层细节**：后台任务不绕过权限；exit code 通过命令语义解释；大输出持久化并给模型 preview/path

**产出**：更新 `wiki/tool-calling.md` §6（BashTool 高风险路径），更新 `wiki/index.md` Tool Calling 摘要

**待继续**：SyntheticOutputTool 结构化输出、Fine-grained tool streaming

---

## [2026-05-09] query | SyntheticOutputTool 与 Fine-grained Tool Streaming

**范围**：`src/tools/SyntheticOutputTool/SyntheticOutputTool.ts`, `src/utils/hooks/hookHelpers.ts`, `src/main.tsx`, `src/QueryEngine.ts`, `src/utils/api.ts`, `src/services/api/claude.ts`, `src/utils/messages.ts`, `src/query.ts`, `src/query/config.ts`, `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolExecution.ts`

**讨论内容**：

1. **Tool-as-Schema 结构化输出**：`StructuredOutput` 只在 non-interactive session 中根据 `jsonSchema` 动态创建，让模型通过 `tool_use.input` 提交最终 JSON
2. **Ajv 双阶段校验**：创建工具时校验调用方 schema，执行工具时校验模型输出 input；失败转成 schema mismatch 错误
3. **structured_output side channel**：普通 `tool_result` 只返回成功文本，真实 JSON 通过 `structured_output` attachment 被 `QueryEngine` 收集进最终 result
4. **Stop hook 强制收口**：未调用 `StructuredOutput` 时通过 Stop hook 提醒模型补调用，并用最大重试次数避免无限循环
5. **Fine-grained tool streaming**：`toolToAPISchema()` 在一方 Anthropic API 且 gate/env 允许时添加 `eager_input_streaming`
6. **raw stream 累计 input_json_delta**：`claude.ts` 自己拼接 tool input 字符串，等 `content_block_stop` 后再 parse，避免 SDK partial parse 的 O(n²) 成本
7. **StreamingToolExecutor 提前执行**：完整 `tool_use` block 一到达就按并发安全策略入队执行，最后用 `getRemainingResults()` 兜底收齐结果
8. **fallback / abort 兜底**：streaming fallback 时 discard 旧 executor，用户中断或错误级联时生成 synthetic `tool_result` 保持配对完整

**产出**：更新 `wiki/tool-calling.md` §8-§10，更新 `wiki/index.md` 中 Tool Calling 状态为完成

**待继续**：进入 cc-haha 第 3 章 MCP 协议集成
