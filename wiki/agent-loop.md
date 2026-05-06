# Agent Loop 设计

> 状态：🟢 完成 | 最后更新：2026-05-06
> 相关源文件：`src/query.ts`, `src/QueryEngine.ts`, `src/utils/attachments.ts`, `src/utils/messageQueueManager.ts`, `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolOrchestration.ts`

## 1. 架构概览

### 调用链路

```
用户输入
  → QueryEngine.processUserInput()          ← 入口：解析输入、组装 context
    → query()                                ← 外层包装
      → queryLoop()                          ← 核心循环 (while true)
        → deps.callModel()                   ← 调用 LLM API (Streaming)
        ← 解析 assistant message
        if (包含 tool_use) → needsFollowUp = true
          → runTools() / StreamingToolExecutor  ← 执行工具
          ← tool_result 追加到 messages
          → continue                         ← 下一轮迭代
        else → !needsFollowUp
          → 各种恢复路径 (compact/collapse/max_tokens)
          → stop hooks
          → return { reason: ... }           ← 终止循环
```

### query() vs queryLoop()

| | `query()` | `queryLoop()` |
|---|---|---|
| 可见性 | export (公开) | 内部函数 |
| 位置 | `src/query.ts:219` | `src/query.ts:241` |
| 职责 | 外层包装，通过 `yield*` 委托给 queryLoop，在 loop 返回后收尾 | 核心 `while(true)` 循环，Agent 的"思考-行动-观察"引擎 |
| 收尾工作 | 标记已消费命令为 `completed` | 不关心 |

```typescript
// query() 的实现
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)  // 委托
  // ← 只有 queryLoop 正常 return 才会走到这里 (throw/abort 不会)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

---

## 2. queryLoop 循环终止条件

`while(true)` 通过 `return { reason: ... }` 终止，终止条件分三类：

### 2.1 正常完成

| reason | 行号 | 触发条件 |
|--------|------|----------|
| `'completed'` | ~1264 | 无 tool_use，无恢复路径，stop hooks 放行 |
| `'completed'` | ~1357 | 同上，但最后一条消息是 API 错误消息 |
| `'stop_hook_prevented'` | ~1279 | stop hook 阻止继续 |

### 2.2 异常/错误终止

| reason | 行号 | 触发条件 |
|--------|------|----------|
| `'blocking_limit'` | ~646 | 调用 API **之前**，token 总量已超过模型硬限制 |
| `'image_error'` | ~977 | 图片 base64 超限或压缩失败 |
| `'image_error'` | ~1175 | API 返回 media-size 拒绝，reactive compact 恢复失败 |
| `'prompt_too_long'` | ~1175 | API 返回 413，恢复路径耗尽 |
| `'prompt_too_long'` | ~1182 | reactiveCompact 不可用，context collapse 也无法恢复 |
| `'model_error'` | ~996 | callModel 抛出未预期的异常 |

### 2.3 外部中断/限制

| reason | 行号 | 触发条件 |
|--------|------|----------|
| `'aborted_streaming'` | ~1051 | 流式输出期间被 AbortController 中止 |
| `'aborted_tools'` | ~1515 | 工具执行期间被中止 |
| `'hook_stopped'` | ~1520 | 工具执行后 hook 发出拦截信号 |
| `'max_turns'` | ~1711 | 达到 maxTurns 限制 |

### 关键洞察：`reason` 返回值无人消费

所有调用方（QueryEngine、REPL、runAgent 等 6 处）都通过 `for await (const message of query({...}))` 消费，**只读取 yielded 的消息，不捕获 `return` 值**。

结果判断不依赖 `reason`，而是通过 `isResultSuccessful()` 分析最后一条消息：

```typescript
// QueryEngine.ts — for-await 结束后
const result = messages.findLast(m => m.type === 'assistant' || m.type === 'user')
if (!isResultSuccessful(result, lastStopReason)) {
  yield { subtype: 'error_during_execution' }
} else {
  yield { subtype: 'success' }
}
```

`reason` 本质上是**内部调试/遥测信号**，用于日志和 analytics。

---

## 3. isResultSuccessful() 判断逻辑

位置：`src/utils/queryHelpers.ts:56-94`

```typescript
function isResultSuccessful(message, stopReason):
  if (!message) → false

  if message.type === 'assistant':
    lastContent.type 是 'text' | 'thinking' | 'redacted_thinking' → true
    是 'tool_use' → false (缺 tool_result 配对)

  if message.type === 'user':
    content 全部是 'tool_result' → true  (工具执行完，等模型下一轮回复)
    否则 → false

  // 兜底：stop_reason 为 'end_turn' → true
  // (场景：API 正常结束但模型返回空内容，如 task_notification drain turn)
```

---

## 4. 关键设计模式

### 4.1 AsyncGenerator 流式管道

```typescript
async function* queryLoop(...): AsyncGenerator<StreamEvent | Message, Terminal>
```

- `yield` 逐条输出事件，上层按需拉取
- 天然支持背压（backpressure）和中断（`.return()`）
- `using` 语法自动清理资源（如 `pendingMemoryPrefetch`）

### 4.2 不可变参数 + 可变状态分离

```typescript
// 不可变参数 — 循环外解构，永不变
const { systemPrompt, canUseTool, fallbackModel, ... } = params

// 可变状态 — 每次迭代末尾整体替换
const next: State = { messages: [...], turnCount: ..., ... }
state = next
```

### 4.3 多策略错误恢复

```
prompt_too_long 413
  ├── ① context collapse drain (保留细粒度上下文，去掉已折叠的)
  ├── ② reactive compact (去图+摘要压缩)
  └── ③ 全部失败 → surface error

max_output_tokens 截断
  ├── ① 扩容重试 (8k → 64k，同请求重发)
  └── ② 注入 meta message: "Resume directly..." (最多 3 次)

media-size 拒绝 (图片/PDF 过大)
  └── reactive compact 去图重发 → 失败则 surface
```

---

## 5. 关键数据结构

### queryLoop State

```typescript
type State = {
  messages: Message[]                      // 当前对话消息
  toolUseContext: ToolUseContext            // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number      // 截断恢复计数 (≤3)
  hasAttemptedReactiveCompact: boolean      // 防死循环标记
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number                         // 当前轮次
  transition: Continue | undefined          // 上轮为什么 continue (调试用)
}
```

### Stop 条件判断

```typescript
let needsFollowUp = false
// streaming 中每收到一个 tool_use → needsFollowUp = true
const toolUseBlocks: ToolUseBlock[] = []

// 如果 needsFollowUp = true → 执行工具 → continue 下一轮
// 如果 needsFollowUp = false → 走终止逻辑
```

---

## 6. needsFollowUp = true：工具执行分支

当 LLM 返回 `tool_use` 块时，循环不会终止，而是进入**三阶段执行流程**。

### 6.1 执行时序总览

```
新一轮迭代中:
│
├── 阶段1: API 流式接收 + 边收边执行 (StreamingToolExecutor)
│   deps.callModel() → for await (message of stream)
│   每收到一个 content block:
│   ├── tool_use → toolUseBlocks.push() + needsFollowUp = true
│   ├── StreamingToolExecutor.addTool()   ← 立刻排队执行
│   └── getCompletedResults() 取出已完成结果，实时 yield
│
├── 阶段2: 流结束后收集剩余结果
│   streamingToolExecutor.getRemainingResults()
│   或 (无 streaming)
│   runTools() → runToolsConcurrently / runToolsSerially
│
├── 阶段3: 组装消息 → continue
│   next = { messages: [...prev, ...assistant, ...toolResults], ... }
│   state = next → continue 回循环顶部
```

### 6.2 阶段1：StreamingToolExecutor 边流边执行

位置：`src/services/tools/StreamingToolExecutor.ts`

**核心价值**：不等模型说完，收到 tool_use 就立刻执行。模型还在 generating 后面的 tool_use 时，前面的工具已经在跑了。

#### 工具状态机

```
addTool() → queued → canExecuteTool? → executing → completed → yielded
                                                          ↓
                                               pendingProgress 实时推送
```

#### 并发控制规则

```
canExecuteTool(isConcurrencySafe):
  ├── 当前 executing 数为 0 → ✅ 可执行
  ├── 当前全部是 isConcurrencySafe 且新工具也是 → ✅ 并行
  └── 当前有非 concurrency-safe 工具在执行 → ❌ 等待
  └── 新工具是非 concurrency-safe 且队列前面有未完成的 → ❌ 阻塞队列
```

`isConcurrencySafe` 不是工具类型决定的，而是**具体 input 决定**（通过 Zod parse 后调 `toolDefinition.isConcurrencySafe(input)` 判断）。

#### Bash 错误级联

```typescript
// 只有 Bash 错误会杀死兄弟工具。Read/WebFetch 等不影响。
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

**设计理由**：Bash 命令有隐式依赖链（`mkdir` 失败则后续命令无意义），而 Read/WebFetch 互相独立。

#### 进度消息实时推送

Progress 不走 results 队列，直接推入 `pendingProgress`，立即唤醒 `getRemainingResults`：

```typescript
if (update.message.type === 'progress') {
  tool.pendingProgress.push(update.message)
  this.progressAvailableResolve?.()  // 唤醒等待
}
```

### 6.3 阶段2：runTools() — 无 streaming 的兜底路径

位置：`src/services/tools/toolOrchestration.ts`

#### 分区策略

```typescript
partitionToolCalls(toolUseBlocks):
  相邻的 isConcurrencySafe 工具 → 合入同一 batch
  非 concurrency-safe 工具 → 单独一个 batch

  示例:
  [Read(a), Read(b), Bash(c), Read(d)]
  → [
      { isConcurrencySafe: true,  blocks: [Read(a), Read(b)] },  // 并发
      { isConcurrencySafe: false, blocks: [Bash(c)] },           // 串行
      { isConcurrencySafe: true,  blocks: [Read(d)] },           // 并发
    ]
```

#### 并发执行

```typescript
async function* runToolsConcurrently(blocks, ...):
  yield* all(
    blocks.map(async function* (toolUse) {
      yield* runToolUse(toolUse, ...)  // 每个工具独立执行
    }),
    getMaxToolUseConcurrency(),  // 默认 10，通过 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY 配置
  )
```

#### 串行执行

```typescript
async function* runToolsSerially(blocks, ...):
  for (const toolUse of blocks):
    for await (const update of runToolUse(toolUse, ...)):
      yield update
      currentContext = modifier(currentContext)  // context modifier 在串行时立即生效
```

### 6.4 两种执行路径对比

| | StreamingToolExecutor | runTools() |
|---|---|---|
| 启动时机 | **边流边执行** | 流完全结束后 |
| 并发控制 | 排队 + canExecuteTool 按接收顺序 | partitionToolCalls 分 batch |
| 进度输出 | `pendingProgress` 实时 push | 无（工具都执行完了） |
| Bash 错误级联 | `siblingAbortController.abort()` | 串行，单个失败不影响后续 batch |
| Concurrency-safe | 非 safe 工具会阻塞队列 | 独立 partition，互不阻塞 |
| 使用条件 | `config.gates.streamingToolExecution === true` | 兜底路径 |

### 6.5 阶段3：注入 + 消息组装 → continue

阶段3 不是简单的数组拼接。工具执行完毕后、组装下一轮 State 之前，有三个信息注入点——向模型"汇报"工具执行期间外部世界发生的变化。

**Why：** system prompt 在 turn 开始时已固定，但 Agent 执行工具期间外部世界在变化——用户发了新消息、后台任务完成了、文件被修改了。这些新信息如果不注入，模型就活在过时的世界观里。注入点的设计目标是：**让模型在下一轮思考前，先看到这些运行时新事件**。

**What：** 三个注入点依次执行，产生 attachment 消息 push 到 `toolResults` 数组：

```
工具执行完毕 (toolResults 已收集)
  │
  ├── 注入点① getAttachmentMessages()    ← 队列排水 + 15-20 种附件的两阶段并行生成
  ├── 注入点② pendingMemoryPrefetch       ← 异步预取的记忆文件消费
  ├── 注入点③ skillPrefetch               ← 并行预取的技能发现注入
  │
  ▼
  组装 next State { messages: [...old, ...assistant, ...toolResults] }
  → continue
```

注入的 attachment 消息被 push 到 `toolResults`，格式是 `user` 消息，下一轮模型看到时就像"用户在工具执行期间穿插告诉你的新信息"。

**How — 注入点①：getAttachmentMessages**

位置：`src/utils/attachments.ts:2933` → `getAttachments()` (第 743 行)

两阶段并行架构：

```
阶段1: userInputAttachments (仅 input !== null，即首轮用户输入)
  ├── @mentioned files    — @文件引用
  ├── MCP resources       — MCP 资源
  ├── agent mentions      — @agent 引用
  └── skill discovery     — 首轮 skill 发现（非 inter-turn）
  ⇒ await Promise.all() ← 先完成

阶段2: allThreadAttachments (每轮迭代都跑)
  ├── queued_commands      — 命令队列排水
  ├── date_change          — 日期变化提醒
  ├── deferred_tools_delta — 延迟工具 schema 增量
  ├── agent_listing_delta  — agent 列表变化
  ├── mcp_instructions_delta — MCP 指令变化
  ├── changed_files        — 文件变更检测
  ├── nested_memory        — 嵌套记忆文件
  ├── skill_listing        — 可用 skill 列表
  ├── plan_mode            — Plan 模式提示
  ├── todo_reminders       — TODO 提醒
  └── ... 更多 feature gated 项
  ⇒ await Promise.all() ← 15-20 个任务并行
```

阶段 2 必须在阶段 1 之后执行——`nested_memory` 依赖 `at_mentioned_files` 填充的 `nestedMemoryAttachmentTriggers`。

每个 `maybe()` 是独立容错包装器：成功返回结果，失败返回 `[]`，不拖垮其他附件生成。

**命令优先级系统**（注入点①的子话题）

位置：`src/utils/messageQueueManager.ts`

```
PRIORITY_ORDER = { now: 0, next: 1, later: 2 }
```

| 函数 | 默认优先级 | 调用场景 |
|------|-----------|---------|
| `enqueue()` | `next` | 用户命令（prompt、bash），不被打断 |
| `enqueuePendingNotification()` | `later` | 系统通知（tick、cron、task completion），防止饥饿用户输入 |

queryLoop 取命令时：正常迭代取 `'next'`（不含 `later`），Agent 执行了 Sleep 工具后取 `'later'`（全部清空）。Sleep 是自主模式（Proactive）的标志——Agent 声明自己空闲，趁机处理低优先级积压。

**How — 注入点②：memory prefetch**

位置：`src/utils/attachments.ts:2357` (启动), `src/query.ts:1599` (消费)

启动一次，消费一次。设计核心：用户输入在整个 turn 不变，不需要每轮重新搜索。

```typescript
// query() 入口，while 循环外 —— 启动一次
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

// 守卫条件（4 个 early return 跳过搜索）:
// 1. AutoMemory 未启用
// 2. 没有真实用户消息（只剩 isMeta 系统注入）
// 3. 单字 prompt（无空白字符，无法提取搜索词）
// 4. 会话已注入 memory 超过 MAX_SESSION_BYTES

// while 循环内，工具执行后 —— 非阻塞消费
if (pendingMemoryPrefetch?.settledAt !== null &&  // 已完成
    pendingMemoryPrefetch?.consumedOnIteration === -1) {  // 未消费
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise,
    toolUseContext.readFileState,  // 去重：已读/已注入的文件
  )
  // push 到 toolResults
  pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
}
```

两层去重：

| 机制 | 作用域 | 重置条件 |
|------|--------|----------|
| `readFileState` | 整个 session（跨 turn） | 不重置 |
| `alreadySurfaced` | 当前 messages 窗口 | compact 后自动清空 |

复用 `readFileState`（FileReadTool 也写入）实现双重目的——模型读了文件后不再注入该文件的 memory，上一轮已注入过的也不再重复。

`using` 确保 Generator 退出（return/throw/break/用户取消）时 `[Symbol.dispose]()` 自动调用，中止进行中的搜索并写遥测。

**How — 注入点③：skill discovery**

Skill discovery 是一套与 memory prefetch 同构的异步预取系统，目标是从 200+ 用户/项目/插件技能中，按上下文动态发现当前对话可能需要的技能。

技能呈现分两套机制共存：

| | skill_listing | skill_discovery |
|---|---|---|
| **触发方式** | `getAttachmentMessages` 每轮附带 | 异步 prefetch（迭代开始启动，post-tools 消费） |
| **内容** | 全量可用技能列表（initial + delta） | 上下文相关的动态发现结果 |
| **来源** | `getSkillToolCommands()` + `getMcpSkillCommands()` | AKI 模型查询 |
| **Feature gate** | 无，但 skill-search 启用后缩减为 bundled+MCP | `EXPERIMENTAL_SKILL_SEARCH` |
| **格式** | `type: 'skill_listing'`，纯文本 content | `type: 'skill_discovery'`，结构化 `{name, description, shortId}[]` |

**启动——每轮迭代开始**

位置：`query.ts:331-335`

```typescript
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(
  null,          // null = 主线程，发送 'assistant_turn' signal
  messages,      // 当前对话消息
  toolUseContext,
)
```

与 LLM API 调用、工具执行**并发运行**，不阻塞主循环。`findWritePivot` 守卫使非写操作迭代直接返回空——只在对代码有修改意图时才做发现。

**消费——工具执行后**

位置：`query.ts:1617-1628`

```typescript
if (skillPrefetch && pendingSkillPrefetch) {
  const skillAttachments =
    await skillPrefetch.collectSkillDiscoveryPrefetch(pendingSkillPrefetch)
  for (const att of skillAttachments) {
    const msg = createAttachmentMessage(att)
    yield msg
    toolResults.push(msg)
  }
}
```

`collectSkillDiscoveryPrefetch` 返回的附件格式：

```typescript
{
  type: 'skill_discovery'
  skills: { name: string; description: string; shortId?: string }[]
  signal: DiscoverySignal              // 'assistant_turn' | 'subagent_spawn' 等
  source: 'native' | 'aki' | 'both'   // 来源
}
```

性能：>98% 的情况下 prefetch 在消费点之前已解析完成（AKI 缓存命中 ~250ms vs turn 时长 2-30s）。`hidden_by_main_turn: true` 标记命中的 prefetch（用于遥测）。

**Turn-0：阻塞路径**

首轮用户输入没有并行工作可重叠，因此 turn-0 走**同步阻塞**的 `getTurnZeroSkillDiscovery`：

```typescript
// attachments.ts:801-813
...(feature('EXPERIMENTAL_SKILL_SEARCH') &&
  skillSearchModules &&
  !options?.skipSkillDiscovery    // ← 防止 SKILL.md 展开内容误触发
  ? [
      maybe('skill_discovery', () =>
        skillSearchModules.prefetch.getTurnZeroSkillDiscovery(
          input, messages, context,
        ),
      ),
    ]
  : [])
```

`skipSkillDiscovery` guard 的来源：当用户调用 `/commit` 等 skill 时，`getMessagesForPromptSlashCommand` 会将 SKILL.md 展开后作为 `input` 传入 `getAttachmentMessages`。如果不加 guard，110KB 的 SKILL.md 会被当作"用户意图"触发 discovery，每次 ~3.3s 的 AKI 查询（session 13a9afae）。

**skill_listing 的退让策略**

当 skill-search 启用时，`skill_listing` 从全量退化为只发 **bundled（内置精选）+ MCP（用户连接的服务器）**：

```typescript
// attachments.ts:2688-2693
if (skillSearchModules?.featureCheck.isSkillSearchEnabled()) {
  allCommands = filterToBundledAndMcp(allCommands)
  // 若 bundled+mcp > 30 → 回退到 only bundled
}
```

设计意图：bundled + MCP 数量少（通常是可控的）、来源可信，直接告诉模型即可。用户/项目/插件技能（长尾，200+）走 discovery 按需匹配。

**Subagent 差异**

Subagent 不走 turn-0 阻塞路径，`startSkillDiscoveryPrefetch` 用 `subagent_spawn` signal 代替 `assistant_turn`。结果是 subagent 第一轮看不到 discovery 结果（"visible turn 1"），只能靠 `skill_listing`（bundled+MCP）做初始决策。

**Feature gate 与公开仓库的 stub**

所有 skill search 真实现实在 ant-internal 仓库。公开仓库中 `services/skillSearch/*.ts` 是 Proxy 构成的万能 stub——任何属性访问和函数调用都返回另一个空 Proxy，不崩溃也不执行实际逻辑。DCE 后这些代码路径完全不存在于外部 bundle。

条件 require 确保 `'skill_discovery'` 字符串和其它 skill-search 相关字面量不会泄漏到外部 build：

```typescript
// query.ts:66-68
const skillPrefetch = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? (require('./services/skillSearch/prefetch.js') as typeof import(...))
  : null  // ← 外部 build 走这条，整个模块被 DCE
```

**三层注入点的次序**

```
注入点① getAttachmentMessages()     ← 命令排水 + 附件（含 skill_listing）
注入点② pendingMemoryPrefetch 消费    ← 相关记忆文件注入
注入点③ collectSkillDiscoveryPrefetch ← 动态发现的技能注入
```

次序无关 dependency（三个注入互不依赖彼此的输出），但 ③ 放最后是延迟最大化——给 prefetch 最多的完成时间。

### 6.6 阶段3 收尾：消息组装 → continue

```typescript
const next: State = {
  messages: [
    ...messagesForQuery,      // 本轮开始时的消息
    ...assistantMessages,     // 模型返回 (含 tool_use)
    ...toolResults,           // 工具结果 + 注入的附件消息
  ],
  toolUseContext: updatedToolUseContext,
  turnCount: nextTurnCount,
  transition: { reason: 'next_turn' },
}
state = next
continue  // → 回到 while(true) 顶部，下一轮调 API
```

组装后消息形状：

```
[...旧历史,
 assistant("我来读取文件" + tool_use: Read),
 assistant(tool_use: Bash),
 user(tool_result: Read结果),
 user(tool_result: Bash结果),
 user(attachment: relevant_memories),   ← 注入点产生的 attachment
 user(attachment: queued_command)]      ← 也混在 toolResults 里
```

### 6.7 工具执行期间的中断处理

| 条件 | 行为 | reason |
|------|------|--------|
| 工具执行中被 abort | 跳过继续，检查 maxTurns → 返回 | `'aborted_tools'` |
| hook 返回 `hook_stopped_continuation` | 收集完结果但标记阻止 | `'hook_stopped'` |
| interrupt 时 interruptBehavior='cancel' 的工具有效 | 工具被取消，用 REJECT_MESSAGE | - |
| interrupt 时 interruptBehavior='block' 的工具 | 不取消，必须执行完 | - |

---

## 7. 相关概念

- **querySource**: 标识 API 调用来源 (`'sdk'`, `'compact'`, `'session_memory'` 等 30+ 种)
- **forkedAgent**: 创建隔离上下文的子 agent，用于 compact 和 session_memory 等后台任务
- **compact agent**: 将长对话压缩为摘要，释放 token 空间，maxTurns=1
- **session_memory agent**: 从对话提取关键信息写入 MEMORY.md，实现跨会话记忆

---

## 8. 待学习

- [x] `queryLoop` 中 `needsFollowUp = true` 分支的完整流程（工具执行、附件收集、memory prefetch）
- [x] `StreamingToolExecutor` 的工作机制
- [x] `runTools()` 的并发/串行策略
- [x] 阶段2-3 之间：附件收集（getAttachmentMessages + 命令优先级系统）
- [x] 阶段2-3 之间：memory prefetch（异步预取 + using + 双层去重）
- [x] 阶段2-3 之间：skill discovery 注入
- [x] AutoCompact 的完整触发和恢复流程
- [x] Context Collapse 的实现细节

## 参考

- [cc-haha-learning-path.md §1](../raw/cc-haha-learning-path.md#1-agent-loop-设计)
- `src/query.ts` — query() 和 queryLoop()
- `src/QueryEngine.ts` — QueryEngine.processUserInput()
- `src/utils/attachments.ts` — getAttachmentMessages() / startRelevantMemoryPrefetch()
- `src/utils/messageQueueManager.ts` — enqueue() / enqueuePendingNotification() / PRIORITY_ORDER
- `src/services/tools/StreamingToolExecutor.ts` — 边流边执行工具
- `src/services/tools/toolOrchestration.ts` — runTools / runToolsConcurrently / runToolsSerially
- `src/utils/queryHelpers.ts` — isResultSuccessful()
