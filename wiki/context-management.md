# 上下文管理与 Token 优化

> 状态：🟡 进行中 | 最后更新：2026-05-13
> 相关源文件：`src/query.ts`, `src/constants/prompts.ts`, `src/context.ts`, `src/utils/queryContext.ts`, `src/utils/api.ts`, `src/utils/claudemd.ts`, `src/utils/hooks.ts`, `src/utils/attachments.ts`, `src/services/api/claude.ts`, `src/services/compact/`

## 1. 架构概览：多层压缩策略

cc-haha 实现了从"主动预防"到"被动恢复"的完整梯度，每次 queryLoop 迭代入口执行：

```
queryLoop 迭代入口
  ├── ① toolResultBudget — 工具结果大小预算
  ├── ② snipCompact — 片段级移除（feature gated）
  ├── ③ microCompact — 细粒度工具结果裁剪
  ├── ④ contextCollapse — 多轮搜索折叠（feature gated）
  └── ⑤ autoCompact — 阈值触发，LLM 摘要压缩
```

所有压缩在 API 调用**之前**执行，确保发给模型的 messages 已是最精简形态。

## 2. AutoCompact — 主动压缩核心

### 2.1 触发阈值

位置：`src/services/compact/autoCompact.ts`

```
有效窗口 = 模型 context window - max_output_tokens (≤ 20k)
阈值 = 有效窗口 - AUTOCOMPACT_BUFFER_TOKENS (13k)

即：当 token 使用量 ≥ (context window - max_output - 13k) 时触发
```

关键约束：
- `querySource === 'session_memory'` / `'compact'` → **拒绝**（Forked Agent 会死锁）
- `querySource === 'marble_origami'` → 拒绝（collapse 上下文代理，会破坏主线程状态）
- `feature('CONTEXT_COLLAPSE')` 启用时 → 拒绝（collapse 自己管上下文）
- `feature('REACTIVE_COMPACT')` 启用时 → 拒绝（只做被动压缩）

环境变量覆盖：
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` → 强制缩小有效窗口
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` → 百分比覆盖阈值
- `DISABLE_COMPACT` / `DISABLE_AUTO_COMPACT` → 禁用

### 2.2 断路器机制

```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

连续失败 3 次后停止尝试 —— 防止上下文不可恢复地超限时反复 hammer API。背景数据：1,279 个 session 曾有 50+ 次连续失败（最多 3,272 次），浪费 ~250K API 调用/天。

### 2.3 执行流程

```
autoCompactIfNeeded()
  ├── shouldAutoCompact() → false → return { wasCompacted: false }
  ├── ① trySessionMemoryCompaction() → 成功 → return (跳过 LLM 调用)
  └── ② compactConversation() → LLM 摘要压缩
       ├── recompactionInfo: { isRecompactionInChain, turnsSincePreviousCompact }
       └── 失败 → 递增 consecutiveFailures
```

**先尝试 Session Memory 压缩**（零 LLM 成本），失败再走传统 compact。

### 2.4 传统 Compact：compactConversation()

位置：`src/services/compact/compact.ts`

核心流程：
1. **stripImagesFromMessages**: 去掉图片（不需要给摘要模型看图片）
2. **groupMessagesByApiRound**: 按 API round 分组
3. **runForkedAgent**: 启动 compact agent（独立 Forked Agent，maxTurns=1，`querySource='compact'`）
4. **getCompactPrompt**: 调用 `getCompactUserSummaryMessage` 生成摘要提示词注入
5. **buildPostCompactMessages**: 组装 [旧摘要 + 摘要边界标记 + 保留的最近消息]

### 2.5 Session Memory Compact（替代方案）

位置：`src/services/compact/sessionMemoryCompact.ts`

不需要 LLM 调用 —— 直接在消息层面裁剪：

```
calculateMessagesToKeepIndex():
  从 lastSummarizedMessageId 开始
  → 向前扩展直到满足:
    ≥ minTokens (10k) tokens
    ≥ minTextBlockMessages (5) 条有文本的消息
  → 不超过 maxTokens (40k)
  → adjustIndexToPreserveAPIInvariants(): 不切断 tool_use/tool_result 配对
```

压缩结果：摘要消息（来自 Session Memory 文件内容）+ 保留最近消息 → 边界标记 → 完成。

### 2.6 Post-Compact 清理

位置：`src/services/compact/postCompactCleanup.ts`

```typescript
runPostCompactCleanup(querySource):
  resetMicrocompactState()         // 清空 cached MC 状态
  resetContextCollapse()           // 重置 collapse 日志
  getUserContext.cache.clear()     // 清缓存让 InstructionsLoaded hook 重触发
  resetGetMemoryFilesCache()
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()
  clearSessionMessagesCache()
```

注意：**不**清除已调用的 skill 内容 —— skill 内容必须跨 compact 存活。

## 3. MicroCompact — 细粒度工具结果裁剪

位置：`src/services/compact/microCompact.ts`

针对**工具输出**（而非对话内容）的精简，两种路径：

### 3.1 Cached MicroCompact（主路径）

利用 Anthropic Cache Editing API 删除旧工具结果，**不破坏 Prompt Cache 前缀**：

```
cachedMicrocompactPath():
  遍历 messages → 注册每个 compactable 工具的 tool_result
  → getToolResultsToDelete() → 按 triggerThreshold/keepRecent 决定删除哪些
  → createCacheEditsBlock() → 生成 cache_edits 块
  → 返回 messages 不变（cache_edits 在 API 层注入）
```

COMPACTABLE_TOOLS: Read、Bash、Grep、Glob、WebSearch、WebFetch、Edit、Write

### 3.2 Time-Based MicroCompact

当距上一条 assistant 消息超过 `gapThresholdMinutes` 时触发 —— 此时缓存已冷，前缀无论如何会重写，直接在 messages 中替换旧工具结果为占位文本。

```
evaluateTimeBasedTrigger() → gapMinutes ≥ threshold
  → 替换旧 tool_result 内容为 '[Old tool result content cleared]'
```

## 4. Tool Result Budget

在 microCompact 之前运行，对单个工具结果做大小限制。超大的 tool_result 被截断并替换为 contentReplacementState 中的引用。

与 microCompact 独立组合 —— cached MC 只看 tool_use_id，不检查 content，所以两者互不干扰。

## 5. 阶段 2-3：工具执行后的附件注入

位置：`src/query.ts:1580-1628`

工具执行完成（阶段 2）后、组装消息 continue（阶段 3）之前，有三个注入点：

### 5.1 附件收集（getAttachmentMessages）

```typescript
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null, queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults], querySource
)) {
  yield attachment
  toolResults.push(attachment)  // 以 user 消息形式追加到 toolResults
}
```

将排队的命令（prompt 模式、task-notification）转换为 attachment message，注入到 toolResults。

### 5.2 Memory Prefetch 消费

```typescript
if (pendingMemoryPrefetch?.settledAt && pendingMemoryPrefetch.consumedOnIteration === -1) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise, toolUseContext.readFileState
  )
  // 以 attachment message 形式注入 toolResults
  pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
}
```

关键设计：
- `startRelevantMemoryPrefetch()` 在 `query()` 入口触发（per user turn），在后台异步搜索相关记忆
- 每轮迭代检查 `settledAt !== null`，未完成则跳过（0-wait），下轮再试
- `readFileState` 去重：已 Read/Write/Edit 的文件不再注入
- `using` 声明确保 Generator 退出时自动清理

### 5.3 Skill Discovery 注入

```typescript
if (skillPrefetch && pendingSkillPrefetch) {
  const skillAttachments = await skillPrefetch.collectSkillDiscoveryPrefetch(pendingSkillPrefetch)
  // 以 attachment message 形式注入 toolResults
}
```

每轮迭代启动（在 API 调用前），`startSkillDiscoveryPrefetch` 在模型流式输出和工具执行期间并行运行。完成率 > 98%（AKI@250ms / Haiku@573ms vs turn 时长 2-30s）。

## 6. Snip Compact（内部功能）

位置：`src/services/compact/snipCompact.ts`（外部 build 为 stub）

片段级压缩：移除不再需要的中间结果消息。snip 后的 token 减少量通过 `snipTokensFreed` 传递给 autocCompact 的阈值检查。

## 7. Context Collapse（内部功能）

位置：`src/services/contextCollapse/`（外部 build 为 stub）

学习路线图中的描述：在 compact 之前先尝试**折叠上下文**——将多轮搜索/读取操作折叠为摘要，以更低成本解决窗口溢出。

两阶段机制（来自 query.ts 注释）：
- 90% commit：开始提交折叠
- 95% blocking-spawn：阻塞式触发折叠

当 collapse 启用时，autocompact 被禁用 —— collapse 自己就是上下文管理系统。

## 8. Reactive Compact（内部功能）

位置：`src/services/compact/reactiveCompact.ts`（外部 build 为 stub）

当 API 返回 `prompt_too_long` 错误时的应急压缩。queryLoop 中的恢复顺序：

```
API 413 prompt_too_long
  → reactiveCompact (去图 + 摘要压缩) → 重试
  → context collapse drain → 重试
  → 全部失败 → surface error
```

## 9. 各策略对 Prompt Cache 的影响

| 策略 | 缓存影响 |
|------|----------|
| Cached MicroCompact | 通过 cache_edits 删除，**不破坏缓存** |
| Time-Based MicroCompact | 直接改 content，**破坏缓存**（但此时缓存本已冷） |
| AutoCompact (compactConversation) | 重建 messages，**破坏缓存** |
| Session Memory Compact | 裁剪 messages，**破坏缓存** |
| Snip / Collapse | **破坏缓存** |

`notifyCompaction()` 在 compact 后重置缓存读取基线，防止缓存命中率下降被误报为 break。

## 10. Prompt / UserContext 构造链路

第 4 章 Context Engineering 的重点不是只看 token 压缩，而是先理解 cc-haha 如何把系统规则、用户/项目记忆和运行态上下文拼进一次模型请求。

### 10.1 三块上下文的职责

位置：`src/utils/queryContext.ts`, `src/context.ts`, `src/utils/api.ts`, `src/query.ts`

`fetchSystemPromptParts()` 并发收集三类信息：

```typescript
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  customSystemPrompt !== undefined ? Promise.resolve([]) : getSystemPrompt(...),
  getUserContext(),
  customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
])
```

职责分工：

- `getSystemPrompt()`：cc-haha 内置系统规则，包含主系统提示词、工具说明、动态 boundary 等。
- `getSystemContext()`：运行态系统上下文，例如会话开始时的 git status、cache breaker。
- `getUserContext()`：用户/项目 instruction 和日期，最后作为 meta user message 注入。

注意：`customSystemPrompt` 会跳过默认 system prompt 和 `getSystemContext()`，但不会跳过 `getUserContext()`。这说明 cc-haha 把用户/项目记忆视为工作环境的一部分，而不是默认 system prompt 的附属品。

### 10.2 getSystemPrompt 的分层结构

位置：`src/constants/prompts.ts`, `src/constants/systemPromptSections.ts`, `src/utils/api.ts`

`getSystemPrompt()` 返回的不是单个大字符串，而是 `string[]`。正常路径会把系统提示词拆成稳定前缀和动态 section：

```text
getSystemPrompt()
  → 收集 skill commands / output style / env info / enabled tools
  → 构造 dynamicSections
  → resolveSystemPromptSections()
  → return [
      static product rules,
      SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
      dynamic sections,
    ]
```

稳定前缀包含产品身份、工程行为规范、工具使用规范、风险动作规范、输出风格等：

```typescript
return [
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  getSimpleDoingTasksSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

动态 section 包括：

- `session_guidance`：根据可用工具、skill、AgentTool 等生成的会话级指导。
- `memory`：`loadMemoryPrompt()` 生成的 memdir memory prompt，不是 `CLAUDE.md`。
- `env_info_simple`：cwd、git、platform、shell、模型信息等。
- `language` / `output_style`：用户设置驱动的语言和输出风格。
- `mcp_instructions`：MCP server 提供的 instructions。
- `scratchpad` / `frc` / `summarize_tool_results` / token budget / brief 等 feature-gated section。

普通动态 section 通过 `systemPromptSection()` 缓存到会话级 section cache，直到 `/clear` 或 `/compact` 清理；只有确实会在 turn 间变化的 section 才使用 `DANGEROUS_uncachedSystemPromptSection()`。典型例子是 `mcp_instructions`，因为 MCP servers 可能在会话中途连接或断开。

### 10.3 Prompt Cache 的两层含义

位置：`src/constants/prompts.ts`, `src/utils/api.ts`, `src/services/api/claude.ts`, `src/utils/modelCost.ts`

cc-haha 里提到 Prompt Cache 时，要区分两层：

```text
工程层缓存
  → memoize / systemPromptSection cache
  → 少读文件、少重算、保持 prompt 字节稳定

API 服务端 Prompt Cache
  → 请求中带 cache_control
  → Anthropic/Claude API 返回 cache_read_input_tokens / cache_creation_input_tokens
```

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 是两层之间的桥。边界前的稳定系统提示词可以被切成 cacheable prefix，边界后的用户/会话动态内容不进入 global cache：

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

`splitSysPromptPrefix()` 会根据这个 marker 拆块：

```text
staticBlocks before boundary  → cacheScope: 'global'
dynamicBlocks after boundary  → cacheScope: null
```

随后 `buildSystemPromptBlocks()` 把可缓存 block 转成 Anthropic API 的 text block：

```typescript
{
  type: 'text',
  text: block.text,
  cache_control: getCacheControl(...)
}
```

平台上看到的命中缓存指的是服务端 Prompt Cache：

- `cache_read_input_tokens`：服务端从 prompt cache 读到的 input tokens。
- `cache_creation_input_tokens`：本次写入 prompt cache 的 input tokens。

它不是模型“记住了回答”，也不是 cc-haha 本地 memoize 命中，而是 API 服务端复用了完全相同的 prompt prefix。

### 10.4 最终 API 请求组装

位置：`src/query.ts`, `src/services/api/claude.ts`, `src/utils/api.ts`

cc-haha 最终不是把全部上下文拼成一个大字符串，而是分别走 Anthropic API 的三条主通道：

```text
system   → system prompt blocks
messages → userContext meta message + 对话历史
tools    → tool schemas
```

在 `query.ts` 中，历史消息先经过 budget / snip / microcompact / context collapse / autocompact 等处理，得到真正准备发送的 `messagesForQuery`。随后：

```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)

deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
})
```

进入 `queryModel()` 后，API 层继续做三件事：

1. `tools` 经 `toolToAPISchema()` 转成 API tool schemas。
2. `messages` 经 `normalizeMessagesForAPI()`、`ensureToolResultPairing()`、`stripExcessMediaItems()` 等处理，再由 `addCacheBreakpoints()` 添加 message 层 cache marker。
3. `systemPrompt` 加上 attribution header 和 CLI prefix，再由 `buildSystemPromptBlocks()` 切成带 `cache_control` 的 system text blocks。

最终请求形态：

```typescript
{
  model,
  messages: addCacheBreakpoints(messagesForAPI, ...),
  system,
  tools: allTools,
  tool_choice,
  betas,
  metadata,
  max_tokens,
  thinking,
  context_management,
  output_config,
}
```

这里的设计边界是：

- `systemContext` 追加进 system prompt 末尾，更像运行态系统事实。
- `userContext` 前置进 messages，更像用户/项目意图。
- `tools` 不写进 system prompt，而是作为 API tool schema 单独发送。
- system 和 messages 都可能带 `cache_control`，用于服务端 Prompt Cache。

### 10.5 getUserContext 的核心流程

位置：`src/context.ts`

```text
getUserContext()
  → 判断是否禁用 CLAUDE.md 注入
  → getMemoryFiles()
  → filterInjectedMemoryFiles()
  → getClaudeMds()
  → setCachedClaudeMdContent()
  → return { claudeMd?, currentDate }
```

关键逻辑：

```typescript
const shouldDisableClaudeMd =
  isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
  (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

const claudeMd = shouldDisableClaudeMd
  ? null
  : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

return {
  ...(claudeMd && { claudeMd }),
  currentDate: `Today's date is ${getLocalISODate()}.`,
}
```

禁用条件：

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`：硬禁用 `claudeMd`。
- bare mode 且没有显式 additional directories：跳过自动发现的 CLAUDE.md。

即使禁用了 `claudeMd`，`currentDate` 仍然会返回。

### 10.6 UserContext 不是 system prompt

位置：`src/utils/api.ts`

`getUserContext()` 只返回对象，真正的注入发生在 `prependUserContext()`：

```typescript
createUserMessage({
  content: `<system-reminder>
As you answer the user's questions, you can use the following context:
${Object.entries(context).map(([key, value]) => `# ${key}\n${value}`).join('\n')}
IMPORTANT: this context may or may not be relevant to your tasks...
</system-reminder>`,
  isMeta: true,
})
```

所以 `claudeMd` / `currentDate` 不是直接塞进 `systemPrompt`，而是变成一条排在消息最前面的 meta user message。这样做保留了语义边界：

- system prompt：产品内置行为规则。
- user context：用户和项目给出的工作偏好、项目约束、当前日期。
- 普通 user message：用户本轮真实请求。

### 10.7 getMemoryFiles 的来源与顺序

位置：`src/utils/claudemd.ts`

`getMemoryFiles()` 负责收集 instruction 文件，`getUserContext()` 不直接做文件发现。

主要来源：

1. Managed memory 和 managed `.claude/rules/*.md`
2. User memory 和 user `.claude/rules/*.md`
3. 从原始 cwd 向上走到根目录，再自根向 cwd 处理 Project / Local 文件
4. `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, `CLAUDE.local.md`
5. additional directories 下的同类文件
6. AutoMem / TeamMem 入口文件

实现细节：

- `processedPaths` 做去重。
- symlink 会同时记录原路径和 resolved path。
- nested worktree 会跳过主仓中重复的 checked-in Project 文件，但保留 `CLAUDE.local.md`。
- `@include` 有递归深度上限：`MAX_INCLUDE_DEPTH = 5`。
- 非文本扩展会跳过，避免把图片、PDF 等二进制注入 memory。

### 10.8 大文件处理策略

普通 `CLAUDE.md` / `.claude/rules/*.md` / `CLAUDE.local.md` 的策略是：完整读取、完整注入、只告警。

```typescript
const rawContent = await fs.readFile(filePath, { encoding: 'utf-8' })
```

大文件检测阈值：

```typescript
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

超过阈值后，`getLargeMemoryFiles()` 会被 status / doctor 调用，用于提示：

```text
Large CLAUDE.md will impact performance
```

但这不是截断。普通 instruction 文件即使超过 40k chars，也会进入 `claudeMd`。

例外是 AutoMem / TeamMem 的 `MEMORY.md` 入口：

```typescript
if (type === 'AutoMem' || type === 'TeamMem') {
  finalContent = truncateEntrypointContent(strippedContent).content
}
```

入口文件有独立限制：

- `MAX_ENTRYPOINT_LINES = 200`
- `MAX_ENTRYPOINT_BYTES = 25_000`

截断后会追加 warning，提示只有部分内容被加载。

### 10.9 两层 memoize 缓存

`getUserContext()` 和 `getMemoryFiles()` 是两层独立缓存：

```text
getUserContext cache
  → 缓存最终 { claudeMd?, currentDate }

getMemoryFiles cache
  → 缓存已读取和解析的 MemoryFileInfo[]
```

`getUserContext()` 没有参数，lodash memoize 实际上只有一个缓存槽。第一次调用后，后续请求直接命中外层缓存，不会重新进入 `getMemoryFiles()`。

因此只清内层不够：

```typescript
clearMemoryFileCaches()
```

只会执行：

```typescript
getMemoryFiles.cache?.clear?.()
```

如果外层 `getUserContext` 还在，下一轮仍然使用旧的 `claudeMd`。

真正要让下一轮模型重新看到新的 CLAUDE.md，需要清外层：

```typescript
getUserContext.cache.clear?.()
resetGetMemoryFilesCache(reason)
```

用户侧避坑：

- 修改 `CLAUDE.md` / rules 后，如果希望当前会话后续立刻生效，用 `/clear`、`/compact` 或开新会话。
- `/memory` 主要刷新 memory 文件列表和编辑入口，不等价于当前 `userContext` 已重建。
- 临时要求模型读取新文件可以解决当下任务，但不等价于 hidden/meta `userContext` 重新注入。

### 10.10 InstructionsLoaded hook

位置：`src/utils/hooks.ts`, `src/utils/claudemd.ts`, `src/utils/attachments.ts`

`InstructionsLoaded` 是 instruction 加载观测事件。当 `CLAUDE.md` 或 `.claude/rules/*.md` 真正进入上下文时，cc-haha 向 hook 系统发出事件。

它是 fire-and-forget：

```typescript
void executeInstructionsLoadedHooks(...)
```

用途是 audit / observability，不阻塞主流程，也不改变 prompt。

payload：

```typescript
{
  hook_event_name: 'InstructionsLoaded',
  file_path,
  memory_type,      // User | Project | Local | Managed
  load_reason,      // session_start | nested_traversal | path_glob_match | include | compact
  globs,
  trigger_file_path,
  parent_file_path,
}
```

触发路径：

1. `getMemoryFiles()` eager load：会话开始加载顶层 instruction。
2. compact 后 `resetGetMemoryFilesCache('compact')`：下一次重新加载时上报 `load_reason = compact`。
3. `memoryFilesToAttachments()` lazy load：触碰更深目录或命中 `paths` frontmatter 规则时注入 nested memory attachment，并触发 hook。

`AutoMem` / `TeamMem` 不触发该 hook，因为它们属于独立 memory system，不是 CLAUDE.md / rules 这类 instruction 文件。

### 10.11 设计取舍

`getUserContext()` 的定位不是 token-budget optimizer，而是“忠实加载用户/项目 instruction”。它对普通大文件不擅自裁剪，避免改变规则语义；性能问题通过 status / doctor 提醒用户整理。

缓存设计则偏向会话稳定性：

- 会话内 instruction 是快照，避免每轮读盘和 prompt 抖动。
- `/clear`、`/compact`、新会话代表上下文重建边界。
- `InstructionsLoaded` 让隐式加载的 instruction 对外可观测。

## 11. 相关概念

- **ForkedAgent**: compact 和 session_memory 都是用 ForkedAgent 执行的独立 Agent（`maxTurns=1`, `querySource='compact'`）
- **RecompactionInfo**: 追踪连续 compact 的元信息（是否链式重压缩、距上次 compact 轮数等）
- **promptCacheBreakDetection**: 监控缓存命中率下降，区分"合理 compact 导致"与"真正的缓存断裂"

## 12. 待学习

- [x] autoCompact 触发阈值与断路器
- [x] microCompact 两种路径（cached + time-based）
- [x] Session Memory Compact 替代方案
- [x] 阶段 2-3 附件收集、memory prefetch、skill discovery
- [x] getSystemContext / fetchSystemPromptParts / getUserContext 精读
- [x] getUserContext 大文件处理、memoize 缓存与 InstructionsLoaded hook
- [x] getSystemPrompt 分层、Prompt Cache 与最终 API 请求组装
- [ ] OpenClaw 的 compaction.ts / context-engine/ 对照

## 参考

- [cc-haha-learning-path.md §4](../raw/cc-haha-learning-path.md#4-context-engineering--prompt-engineering)
- [cc-haha-learning-path.md §5](../raw/cc-haha-learning-path.md#5-上下文窗口管理与-token-优化)
- `src/services/compact/autoCompact.ts` — 自动压缩触发与阈值
- `src/services/compact/compact.ts` — 传统 compact 核心
- `src/services/compact/microCompact.ts` — 细粒度工具结果裁剪
- `src/services/compact/sessionMemoryCompact.ts` — 零 LLM 成本压缩
- `src/services/compact/postCompactCleanup.ts` — 压缩后状态重置
- `src/query.ts:1570-1630` — 阶段 2-3 附件注入
- `src/context.ts` — `getSystemContext()` / `getUserContext()` 构造
- `src/constants/prompts.ts` — `getSystemPrompt()` / `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- `src/constants/systemPromptSections.ts` — system prompt section 缓存
- `src/utils/queryContext.ts` — `fetchSystemPromptParts()` 并发组装
- `src/utils/api.ts` — `prependUserContext()` / `appendSystemContext()` / `splitSysPromptPrefix()`
- `src/services/api/claude.ts` — `queryModel()` / `buildSystemPromptBlocks()` / `addCacheBreakpoints()`
- `src/utils/claudemd.ts` — `getMemoryFiles()` / `getClaudeMds()` / cache reset
- `src/utils/hooks.ts` — `InstructionsLoaded` hook 定义与执行
- `src/utils/attachments.ts` — nested memory attachment 与 lazy load hook
