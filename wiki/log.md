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

---

## [2026-05-09] maintenance | cc-haha 学习路径与 wiki 目录对齐

**范围**：`raw/cc-haha-learning-path.md`, `wiki/index.md`, `wiki/mcp-integration.md`, `README.md`

**内容**：

1. 明确 cc-haha 路线定位：1-12 章是源码精读主线，13-15 和附录是阶段规划、面试、Python 迁移与项目外补充能力。
2. 按学习路线重排 `wiki/index.md` 的 cc-haha 目录，合并原先分散的 prompt/context、desktop/IM 等条目，新增多 Provider 与产品化通道的路线占位。
3. 增量创建 `mcp-integration.md` 作为第 3 章起点，记录 MCP client/server 双向链路、`MCPTool` 适配和 Tool 池合并机制。
4. 对照 cc-haha 源码校准 stale path：`adapters/common/chatqueue.ts` → `adapters/common/chat-queue.ts`。

**待继续**：展开 MCP 配置加载、OAuth、list_changed 刷新、channel permissions 与 Computer Use MCP 特殊包装。

---

## [2026-05-09] query | MCP 协议集成 why/what/how 重构

**范围**：`src/services/mcp/config.ts`, `src/services/mcp/client.ts`, `src/services/mcp/auth.ts`, `src/services/mcp/useManageMCPConnections.ts`, `src/services/mcp/channelPermissions.ts`, `src/tools/MCPTool/MCPTool.ts`, `src/entrypoints/mcp.ts`, `src/vendor/computer-use-mcp/`, `src/utils/computerUse/wrapper.tsx`

**讨论内容**：

1. **why/what/how 结构**：MCP 解决外部能力接入碎片化；核心问题拆成配置治理、连接治理、工具适配、安全认证和动态刷新。
2. **配置合并与去重**：enterprise / user / project / local / plugin / claude.ai connector 多 scope 合并；stdio command signature 与 URL signature 是内容级去重指纹，不是协议层唯一 ID。
3. **tools / resources / prompts**：tools 是可调用动作，resources 是可寻址外部数据，prompts 是可复用模板/命令。
4. **MCP Skills**：通过 `skill://` resources 分发远程 skill；`resources/list_changed` 需要清 MCP skill cache 与 skill search index。
5. **OAuth 与认证差异**：`http` / `sse` 主要走 `ClaudeAuthProvider`；metadata discovery、PKCE callback、本地 callback server、secure storage、refresh token、step-up auth、401/403 分工；其他 server 类型分别用 env、headers、IDE token、Claude.ai proxy 或本地权限控制。
6. **动态刷新与权限**：`tools/list_changed`、`prompts/list_changed`、`resources/list_changed` 局部刷新 AppState；channel permission relay 依赖 allowlist 和 experimental capability。
7. **Computer Use MCP**：走 in-process transport 和 `.call()` override，安全边界在 app allowlist、截图过滤、frontmost gate、session lock 和 Esc 中断。

**产出**：重构 `wiki/mcp-integration.md`，更新 `wiki/index.md` MCP 摘要。

**待继续**：`callMCPTool()` 结果转换、MCP resource 读取链路、server 侧 `CallToolResult` 格式。

---

## [2026-05-09] query | MCP 结果转换、Resources 与 Server 侧收尾

**范围**：`src/services/mcp/client.ts`, `src/tools/MCPTool/MCPTool.ts`, `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`, `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`, `src/utils/mcpOutputStorage.ts`, `src/entrypoints/mcp.ts`

**讨论内容**：

1. **MCP tool 调用闭环**：`MCPTool.call()` 通过 `callMCPToolWithUrlElicitationRetry()` / `callMCPTool()` 调远程 `tools/call`，再转成本地 tool result。
2. **结果归一化**：`processMCPResult()` / `transformResultContent()` 按 text、image、audio、resource、resource_link、structuredContent 分流处理。
3. **二进制输出保护**：音频和非图片 blob 不直接进入上下文，而是通过 `persistBinaryContent()` 写入 tool-results 目录，只返回路径说明。
4. **Resource 读取链路**：`ListMcpResourcesTool` 封装 `resources/list`，`ReadMcpResourceTool` 封装 `resources/read`，两者都是只读、并发安全、deferred 工具。
5. **Resources 与 MCP Skills**：`skill://...` resource 是 MCP Skills 的基础，`resources/list_changed` 需要触发 resource cache 与 skill search index 失效。
6. **Server 侧暴露工具**：`src/entrypoints/mcp.ts` 通过 `ListToolsRequestSchema` 和 `CallToolRequestSchema` 把 cc-haha 本地工具暴露给外部 MCP client。
7. **章节收束**：MCP 在 cc-haha 中是外部能力协议层，但所有能力最终都适配回本地 Tool / Command / Agent Loop。

**产出**：补齐 `wiki/mcp-integration.md` §12-§17，更新状态为完成；同步 `wiki/index.md` MCP 状态为完成。

**待继续**：进入 cc-haha 第 4 章 Context Engineering / Prompt Engineering。

---

## [2026-05-13] query | Context Engineering：getUserContext、缓存与 InstructionsLoaded

**范围**：`src/context.ts`, `src/utils/queryContext.ts`, `src/utils/api.ts`, `src/utils/claudemd.ts`, `src/utils/hooks.ts`, `src/utils/attachments.ts`, `src/services/compact/postCompactCleanup.ts`, `src/commands/clear/caches.ts`, `src/commands/memory/memory.tsx`

**讨论内容**：

1. **getUserContext 定位**：收集用户/项目 instruction 与当前日期，返回 `{ claudeMd?, currentDate }`，再由 `prependUserContext()` 包装成 meta user message。
2. **fetchSystemPromptParts 分工**：`customSystemPrompt` 可跳过默认 system prompt 和 `getSystemContext()`，但不会跳过 `getUserContext()`。
3. **Memory 文件来源**：`getMemoryFiles()` 处理 Managed/User/Project/Local、additional dirs、AutoMem/TeamMem、`@include`、rules 与 nested worktree 去重。
4. **大文件策略**：普通 `CLAUDE.md` / rules 完整读取并注入，只通过 status / doctor 告警；AutoMem/TeamMem 的 `MEMORY.md` 入口有 200 行 / 25KB 截断。
5. **两层 memoize**：`getUserContext` 缓存最终 user context，`getMemoryFiles` 缓存底层文件读取结果；只清内层不会让下一轮模型看到新的 `claudeMd`。
6. **缓存清除边界**：`/clear`、`/compact`、post-compact cleanup 会清外层并重置 memory cache；`/memory`、settings sync、worktree 切换多用于内层正确性刷新。
7. **InstructionsLoaded hook**：当 CLAUDE.md / rules 真正进入上下文时发出 audit / observability 事件，payload 包含 `file_path`、`memory_type`、`load_reason`、`globs`、`trigger_file_path`、`parent_file_path`。
8. **使用者避坑**：修改 instruction 后若要当前会话立刻生效，应使用 `/clear`、`/compact` 或新会话；要求模型临时读取文件不等价于重建 hidden/meta `userContext`。

**产出**：更新 `wiki/context-management.md` §10，补齐 Prompt / UserContext 构造、memory 文件加载、大文件策略、memoize 缓存、InstructionsLoaded hook 与使用者建议；更新 `wiki/index.md` Context Engineering 摘要。

**待继续**：完成 Context Engineering 章节剩余的 system prompt 细节回看后，再决定是否进入 Streaming。

---

## [2026-05-14] query | Context Engineering：getSystemPrompt、Prompt Cache 与最终 API 请求

**范围**：`src/constants/prompts.ts`, `src/constants/systemPromptSections.ts`, `src/utils/queryContext.ts`, `src/utils/api.ts`, `src/query.ts`, `src/services/api/claude.ts`, `src/utils/modelCost.ts`

**讨论内容**：

1. **getSystemPrompt 分层**：返回 `string[]` 而非单个字符串；稳定产品规则放在 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 前，session/env/tool/MCP 等动态 section 放在边界后。
2. **systemPromptSection 缓存**：普通 section 会话内缓存，直到 `/clear` 或 `/compact`；`DANGEROUS_uncachedSystemPromptSection` 用于确实会在 turn 间变化的内容，例如 MCP instructions。
3. **Prompt Cache 两层含义**：cc-haha 本地 memoize / section cache 是工程层缓存；平台上的 `cache_read_input_tokens` / `cache_creation_input_tokens` 是 Anthropic API 服务端 Prompt Cache。
4. **cache_control 桥接**：`splitSysPromptPrefix()` 根据 dynamic boundary 拆 system prompt block，`buildSystemPromptBlocks()` 给可缓存 block 加 `cache_control`。
5. **最终请求组装**：`query.ts` 先压缩整理 `messagesForQuery`，再 `appendSystemContext()` 到 system prompt，`prependUserContext()` 到 messages，最后 `queryModel()` 组装 `system + messages + tools`。
6. **API 三通道**：system prompt、messages、tools 分别发送；工具不塞进 system prompt，而是由 `toolToAPISchema()` 转成 API tool schema。
7. **message 层缓存**：`addCacheBreakpoints()` 在 messages 层添加 cache marker，system 和 messages 都可能参与服务端 Prompt Cache。

**产出**：重排并扩展 `wiki/context-management.md` §10，使 Prompt / UserContext 构造链路覆盖 `getSystemPrompt`、Prompt Cache、最终 API 请求组装；更新 `wiki/index.md` Context Engineering 摘要。

**待继续**：快速精读 `AgentTool/prompt.ts` 与 `utils/effort.ts`，收口 Context Engineering 第 4 章剩余补充项。

---
