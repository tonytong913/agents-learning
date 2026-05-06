# 上下文管理与 Token 优化

> 状态：🟡 进行中 | 最后更新：2026-05-06
> 相关源文件：`src/query.ts`, `src/services/compact/`

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

## 10. 相关概念

- **ForkedAgent**: compact 和 session_memory 都是用 ForkedAgent 执行的独立 Agent（`maxTurns=1`, `querySource='compact'`）
- **RecompactionInfo**: 追踪连续 compact 的元信息（是否链式重压缩、距上次 compact 轮数等）
- **promptCacheBreakDetection**: 监控缓存命中率下降，区分"合理 compact 导致"与"真正的缓存断裂"

## 11. 待学习

- [x] autoCompact 触发阈值与断路器
- [x] microCompact 两种路径（cached + time-based）
- [x] Session Memory Compact 替代方案
- [x] 阶段 2-3 附件收集、memory prefetch、skill discovery
- [ ] OpenClaw 的 compaction.ts / context-engine/ 对照

## 参考

- [cc-haha-learning-path.md §5](../raw/cc-haha-learning-path.md#5-上下文窗口管理与-token-优化)
- `src/services/compact/autoCompact.ts` — 自动压缩触发与阈值
- `src/services/compact/compact.ts` — 传统 compact 核心
- `src/services/compact/microCompact.ts` — 细粒度工具结果裁剪
- `src/services/compact/sessionMemoryCompact.ts` — 零 LLM 成本压缩
- `src/services/compact/postCompactCleanup.ts` — 压缩后状态重置
- `src/query.ts:1570-1630` — 阶段 2-3 附件注入
