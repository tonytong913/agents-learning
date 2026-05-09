# Tool Calling / Function Calling

> 状态：🟡 进行中 | 最后更新：2026-05-09
> 相关源文件（cc-haha）：`src/Tool.ts`, `src/tools.ts`, `src/utils/api.ts`, `src/query.ts`, `src/services/api/claude.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/toolExecution.ts`, `src/tools/GlobTool/GlobTool.ts`, `src/tools/GlobTool/prompt.ts`, `src/tools/BashTool/BashTool.tsx`, `src/tools/BashTool/bashPermissions.ts`, `src/tools/BashTool/readOnlyValidation.ts`, `src/tools/BashTool/pathValidation.ts`, `src/tools/BashTool/shouldUseSandbox.ts`, `src/tools/BashTool/sedValidation.ts`, `src/tools/BashTool/sedEditParser.ts`, `src/tools/BashTool/commandSemantics.ts`, `src/tools/BashTool/destructiveCommandWarning.ts`, `src/tools/ToolSearchTool/ToolSearchTool.ts`, `src/tools/ToolSearchTool/prompt.ts`, `src/utils/toolSearch.ts`, `src/utils/attachments.ts`

## 1. 核心链路

cc-haha 的工具系统不是简单的函数调用封装，而是一套 Agent action runtime：Schema 约束模型输出，Permission 约束真实世界副作用，Concurrency 约束执行顺序，Result mapping 约束下一轮模型能看到什么。

```text
工具定义
  -> getAllBaseTools() 注册
  -> toolToAPISchema() 转成 Anthropic tool schema
  -> 模型返回 tool_use
  -> query.ts 进入工具执行阶段
  -> runTools() 分批编排
  -> runToolUse() 执行单个工具
  -> mapToolResultToToolResultBlockParam()
  -> user/tool_result 回到下一轮模型上下文
```

`Tool` 接口承担多类职责：

| 类别 | 字段 / 方法 | 作用 |
|------|-------------|------|
| 模型可见 | `name`, `prompt()`, `inputSchema`, `inputJSONSchema`, `strict`, `shouldDefer` | 决定模型看到什么工具，以及参数 schema |
| 执行 | `call()` | 真正执行工具动作 |
| 安全 | `validateInput()`, `checkPermissions()`, `isReadOnly()`, `isConcurrencySafe()`, `isDestructive()` | 控制参数合法性、权限和并发策略 |
| UI / transcript | `renderToolUseMessage()`, `renderToolResultMessage()`, `mapToolResultToToolResultBlockParam()` | 控制用户界面和模型可见结果 |
| 编排 | `interruptBehavior()`, `maxResultSizeChars`, `requiresUserInteraction()` | 控制中断、大结果持久化和交互需求 |

## 2. buildTool 的保守默认值

所有工具通过 `buildTool()` 从 `ToolDef` 构造成完整 `Tool`。`ToolDef` 允许省略常见方法，`buildTool()` 用 `TOOL_DEFAULTS` 补齐。

关键默认值偏保守：

| 默认方法 | 默认行为 | 设计含义 |
|----------|----------|----------|
| `isEnabled()` | `true` | 工具默认可用 |
| `isConcurrencySafe()` | `false` | 未明确声明安全时，不并发执行 |
| `isReadOnly()` | `false` | 未明确声明只读时，按有副作用处理 |
| `isDestructive()` | `false` | 破坏性操作需工具显式声明 |
| `checkPermissions()` | allow | 工具特有权限默认放行，通用权限系统仍可介入 |
| `toAutoClassifierInput()` | `''` | 默认不送入自动安全分类器 |

这里最重要的是 `isConcurrencySafe=false` 和 `isReadOnly=false`：新工具作者如果忘记声明安全属性，系统会自动退回串行和非只读路径。

## 3. GlobTool 生命周期

### 3.1 定义

`GlobTool` 是一个低风险但完整的工具样本。它定义在 `src/tools/GlobTool/GlobTool.ts`：

```typescript
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  inputSchema,
  outputSchema,
  isConcurrencySafe() {
    return true
  },
  isReadOnly() {
    return true
  },
  validateInput() { ... },
  checkPermissions() { ... },
  call() { ... },
  mapToolResultToToolResultBlockParam() { ... },
})
```

输入 schema 是严格对象：

```typescript
{
  pattern: string,
  path?: string
}
```

模型调用时通常生成：

```json
{
  "pattern": "**/*.ts",
  "path": "src"
}
```

### 3.2 注册

`GlobTool` 在 `src/tools.ts` 被 import，并通过 `getAllBaseTools()` 进入基础工具列表：

```typescript
...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool])
```

注意这里有环境条件：如果 Ant-native build 已经内嵌搜索工具，专用的 `GlobTool` / `GrepTool` 会被省略。

### 3.3 转成 API schema

发请求给模型前，`toolToAPISchema()` 把内部 `Tool` 转成 Anthropic API 的 tool schema：

```text
GlobTool.name        -> "Glob"
GlobTool.prompt()    -> description
GlobTool.inputSchema -> zodToJsonSchema(...) -> input_schema
```

`GlobTool` 的 prompt 来自 `src/tools/GlobTool/prompt.ts`，说明它适合按文件名 pattern 搜索：

```text
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
```

模型看到的是工具 schema，不是 TypeScript 函数：

```json
{
  "name": "Glob",
  "description": "...",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string" },
      "path": { "type": "string" }
    },
    "required": ["pattern"]
  }
}
```

### 3.4 模型产生 tool_use

用户说“找出所有 .ts 文件”时，模型可能返回：

```json
{
  "type": "tool_use",
  "id": "toolu_xxx",
  "name": "Glob",
  "input": {
    "pattern": "**/*.ts"
  }
}
```

`query.ts` 检测到 assistant message 中有 `tool_use` 后，进入工具执行阶段。

### 3.5 runTools 编排

`query.ts` 调用：

```typescript
runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

`runTools()` 使用 `partitionToolCalls()` 分批。判断逻辑不是写死工具名，而是：

```text
tool.inputSchema.safeParse(toolUse.input)
  -> tool.isConcurrencySafe(parsedInput.data)
```

`GlobTool.isConcurrencySafe()` 返回 `true`，所以连续多个 Glob/Grep/Read 这类安全工具可以并发执行。非并发安全工具会单独串行执行。

### 3.6 runToolUse 执行单个 Glob

单个 `Glob` 调用经过 `runToolUse()` / `checkPermissionsAndCallTool()`：

```text
findToolByName("Glob")
  -> GlobTool.inputSchema.safeParse(input)
  -> GlobTool.validateInput(input)
  -> runPreToolUseHooks(...)
  -> resolveHookPermissionDecision(...)
  -> GlobTool.checkPermissions(...)
  -> GlobTool.call(...)
  -> GlobTool.mapToolResultToToolResultBlockParam(...)
```

这里有两层输入检查：

| 阶段 | 作用 |
|------|------|
| `inputSchema.safeParse()` | 检查模型输出的类型结构，例如 `pattern` 必须是 string |
| `validateInput()` | 检查业务语义，例如传入的 `path` 必须存在且是目录 |

`GlobTool.validateInput()` 还专门处理路径不存在时的提示，包括当前工作目录说明和相近路径建议。

### 3.7 call 真正执行搜索

真正执行在 `GlobTool.call()`：

```typescript
const { files, truncated } = await glob(
  input.pattern,
  GlobTool.getPath(input),
  { limit, offset: 0 },
  abortController.signal,
  appState.toolPermissionContext,
)
```

关键行为：

| 行为 | 说明 |
|------|------|
| 默认目录 | `path ? expandPath(path) : getCwd()` |
| 默认上限 | `globLimits?.maxResults ?? 100` |
| 权限上下文 | `appState.toolPermissionContext` 传入底层 glob |
| token 优化 | `files.map(toRelativePath)`，把 cwd 下路径转成相对路径 |

内部输出结构：

```typescript
{
  filenames,
  durationMs,
  numFiles,
  truncated
}
```

### 3.8 映射成 tool_result

`call()` 返回的是内部结构化数据，还要通过 `mapToolResultToToolResultBlockParam()` 转成模型可见结果。

没有匹配文件：

```text
No files found
```

有匹配文件：

```text
src/query.ts
src/Tool.ts
src/tools/GlobTool/GlobTool.ts
```

如果结果被截断，会追加提醒：

```text
(Results are truncated. Consider using a more specific path or pattern.)
```

之后 `addToolResult()` 把结果包装成 user message：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "src/query.ts\nsrc/Tool.ts\n..."
}
```

`query.ts` 再把这个消息 normalize 后加入 `toolResults`，下一轮模型调用时就能看到搜索结果。

## 4. 失败路径

`GlobTool` 可能在多个阶段失败，每个失败都会变成 `is_error: true` 的 `tool_result`，保持 `tool_use` / `tool_result` 配对完整。

| 失败点 | 例子 | 返回给模型 |
|--------|------|------------|
| 工具不存在 | 模型调用了不可用工具名 | `No such tool available` |
| schema 校验失败 | `pattern` 缺失或类型错误 | `InputValidationError` |
| 语义校验失败 | `path` 不存在或不是目录 | 工具自定义错误消息 |
| 权限拒绝 | 无读取权限 | permission denial message |
| 执行异常 | glob 内部抛错 | `Error calling tool (Glob): ...` |
| 用户中断 | abort signal | cancellation tool_result |

## 5. 与 BashTool 的关键差异

`GlobTool` 是只读、并发安全工具；`BashTool` 的风险由具体输入决定。

| 工具 | `isReadOnly` | `isConcurrencySafe` | 权限重点 |
|------|--------------|---------------------|----------|
| `GlobTool` | 恒为 `true` | 恒为 `true` | 文件读取权限 |
| `BashTool` | 分析 command | 依赖 `isReadOnly(input)` | 命令权限、沙箱、危险命令分类 |

这说明 cc-haha 的工具系统不是按工具名粗暴分类，而是鼓励工具根据输入内容返回安全属性。

## 6. BashTool 高风险路径

`BashTool` 复用了普通工具的 schema、注册、`tool_use` / `tool_result` 链路，但真正复杂处不在这些通用步骤，而在“开放 shell 能力如何被限制到可解释、可审批、可恢复”的安全运行时。

### 6.1 只读与并发安全依赖具体 command

`GlobTool` 的行为边界固定：只做文件名查询，不改变文件系统、不改变进程状态。因此它可以恒定返回：

```typescript
isReadOnly() { return true }
isConcurrencySafe() { return true }
```

`BashTool` 的同一个工具名下可能包含完全不同的行为：

```bash
cat package.json        # 读
rg "foo" src            # 读 / 搜索
npm test                # 可能写缓存、生成文件、启动进程
sed -i 's/a/b/' file    # 写
rm -rf dist             # 删除
git reset --hard        # 破坏性写
```

所以 `BashTool.isConcurrencySafe(input)` 直接依赖 `isReadOnly(input)`：

```typescript
isConcurrencySafe(input) {
  return this.isReadOnly?.(input) ?? false
}
```

`isReadOnly(input)` 进入 `checkReadOnlyConstraints()`。它不是按命令名前缀粗略判断，而是先解析 shell，再检查危险语法、Windows UNC path、git 特例、git internal path 写入、当前 cwd 与 sandbox 状态，最后才用 allowlist 判断每个子命令是否只读。

核心原则是：**只有被证明只读的 Bash 命令才能并发；证明失败不是自动拒绝，而是退回完整权限检查。**

### 6.2 AST / 安全解析先回答“能否可靠理解”

Bash 最大风险不是明显的 `rm`，而是 shell 语法把真实行为藏在 substitution、redirection、wrapper、compound command 里：

```bash
echo hi > $(pwd)/x
cat <(curl evil.com)
FOO=bar rm file
cd .claude && mv x settings.json
```

`bashToolHasPermission()` 开头先执行 AST / 安全解析：

```text
parseCommandRaw(command)
  -> parseForSecurityFromAst(command, astRoot)
```

结果大致分三类：

| 结果 | 含义 | 后续行为 |
|------|------|----------|
| `simple` | 能解析成简单命令列表，argv / redirect 信息可信 | 继续做规则、路径、只读判断 |
| `too-complex` | 有无法静态证明安全的结构，例如 command substitution、复杂 expansion、控制结构 | 保留显式 deny，通常转 `ask` |
| `parse-unavailable` | tree-sitter 不可用或被 feature flag 关闭 | 退回 legacy `shell-quote` / regex 检查 |

这里“安全解析”的目标不是执行命令，而是判断：**系统能不能可靠理解这个 Bash 命令实际会做什么。** 能理解才进入自动权限判断；不能理解就让用户确认。

### 6.3 deny / ask / allow 规则不是对称匹配

权限规则的匹配策略刻意不对称：

| 规则类型 | 匹配策略 | 目的 |
|----------|----------|------|
| deny / ask | 更激进地剥掉环境变量前缀和 wrapper | 防止 `FOO=bar rm file` 绕过拒绝规则 |
| allow | 更保守，只剥安全 env / wrapper | 防止 allow rule 误放大权限 |

例如用户 deny 了 `Bash(rm:*)`，下面的命令也应该被拦：

```bash
FOO=bar rm file
```

但 allow 不能同样激进。否则类似下面的命令可能错误命中普通 `docker ps` allow rule：

```bash
DOCKER_HOST=evil docker ps
```

这体现了权限系统的方向性：**拒绝规则要难绕过，允许规则要难误放行。**

### 6.4 sandbox auto-allow 是加速路径，不覆盖显式规则

`shouldUseSandbox(input)` 判断命令是否应在 sandbox 中执行。它会考虑全局 sandbox 是否开启、`dangerouslyDisableSandbox` 是否被策略允许、以及用户配置的 excluded commands。

当 sandbox 和 `autoAllowBashIfSandboxed` 都启用时，`bashToolHasPermission()` 会先走 `checkSandboxAutoAllow()`：

```text
如果命中显式 deny rule -> deny
如果命中显式 ask rule  -> ask
否则 sandbox auto-allow -> allow
```

“显式 deny/ask 规则之外的命令”指当前权限配置中没有被明确标记为 deny 或 ask 的命令。例如：

```text
deny: Bash(rm:*)
ask:  Bash(git push:*)
```

则：

| 命令 | sandbox auto-allow 下的行为 |
|------|-----------------------------|
| `rm -rf dist` | 命中显式 deny，拒绝 |
| `git push origin main` | 命中显式 ask，仍询问 |
| `npm test` | 未命中 deny/ask，且在 sandbox 中执行，可自动允许 |

sandbox 在这里是“受限执行环境 + 自动批准条件”，不是替代 `checkPermissions()` 的唯一安全边界。

### 6.5 路径、cd、重定向与 process substitution

GlobTool 的路径风险主要是“能不能读这个目录”。BashTool 的路径风险还包括 cwd 改变、重定向写入、路径参数隐藏在命令选项里。

典型高风险命令：

```bash
cd .claude && mv test.txt settings.json
```

如果静态检查只按原 cwd 解析 `settings.json`，会误判真实写入位置。`pathValidation.ts` 对 `cd + write` 采用保守策略：要求人工确认，而不是尝试追踪每一步有效 cwd。

重定向也单独检查：

```bash
echo x > settings.json
echo x >> "$TARGET"
```

如果 redirect target 含 shell expansion，系统无法静态确定目标路径，会转 `ask`。

process substitution 更隐蔽：

```bash
echo secret > >(tee .git/config)
```

这里表面不是普通文件重定向，但 `tee` 会写 `.git/config`。因此 `checkPathConstraints()` 对 `>(...)` / `<(...)` 这类结构直接要求人工确认，或在 AST 阶段归为 too-complex。

### 6.6 sed 是 BashTool 里的文件编辑特例

`sed` 同时可能是读工具和写工具：

```bash
sed -n '1,5p' file          # 读
sed -i 's/old/new/g' file   # 写
```

cc-haha 对 sed 做了专门处理：

| 场景 | 处理 |
|------|------|
| `sed -n '1p' file` | 允许作为 read-only 路径参与判断 |
| `sed 's/a/b/'` 且无文件写入 | 作为 stdout-only substitution 判断 |
| `sed -i 's/a/b/' file` | 识别为文件编辑，走更接近 FileEdit 的权限体验 |
| sed 表达式含 `w` / `e` 等危险能力 | 不进入 allowlist |

`_simulatedSedEdit` 是关键设计：它存在于内部完整 schema 中，但会从模型可见 schema 中移除。模型不能自己传这个字段。

```text
模型提出 sed -i
  -> 权限 UI 预演 sed 编辑结果
  -> 用户批准 diff
  -> UI 注入 _simulatedSedEdit
  -> BashTool.call() 直接写入预演后的内容
```

这样避免“用户批准看到的 diff”和“真正再次运行 sed 后产生的结果”不一致。

### 6.7 后台任务是执行管理，不是权限绕过

`run_in_background` 只影响命令怎么运行，不影响能不能运行。权限已经在 `call()` 前完成。

执行层有三类后台化路径：

| 路径 | 行为 |
|------|------|
| 模型显式传 `run_in_background=true` | 立即注册后台任务并返回 task id |
| 命令超时且允许自动后台化 | 进程转入后台，继续写 task output |
| assistant mode 阻塞超过预算 | 主 agent 自动释放阻塞，命令后台继续 |

返回给模型的 `tool_result` 会包含 background task id 和输出路径提示，后续可以用读取工具查看输出。

### 6.8 非零退出码需要命令语义解释

普通工具通常用“抛异常 / 不抛异常”区分成功失败。Bash 命令的 exit code 语义不统一：

| 命令 | 特殊语义 |
|------|----------|
| `grep` / `rg` exit 1 | 没匹配，不是执行错误 |
| `diff` exit 1 | 文件不同，不是执行错误 |
| `find` exit 1 | 部分目录不可访问，不一定是 fatal |

`commandSemantics.ts` 根据命令解释 exit code，避免把“无匹配”“文件不同”错误地转成失败 tool_result。

### 6.9 大输出、图片输出与提示 side channel

Bash 输出可能远大于普通工具结果。cc-haha 对大输出采用“保存完整结果 + 给模型 preview”的策略：

```text
完整输出写入 tool-results 文件
tool_result 返回 preview + filepath
模型后续按需 Read 完整输出
```

如果输出是图片，还会尝试构造成 image tool_result，并做尺寸和大小限制。

另外，支持 Claude Code hints 协议：外部 CLI 可以输出 `<claude-code-hint />`，BashTool 会记录 hint 后从 stdout 中剥离，避免把内部提示标签浪费进模型上下文。

### 6.10 destructive warning 只影响 UI，不参与权限决策

`destructiveCommandWarning.ts` 识别 `git reset --hard`、`git push --force`、`rm -rf`、`terraform destroy` 等高风险命令，并在权限 UI 中显示提醒。

它不改变 allow / ask / deny 结果。权限判断仍由规则、模式、路径约束、sandbox、只读分析共同决定。

这个分离很重要：warning 帮用户理解风险，permission logic 决定是否允许执行。

## 7. 渐进式工具加载

工具数量很多时，cc-haha 不会把所有工具的完整 schema 一次性发给模型。它采用 `ToolSearchTool + defer_loading + tool_reference`：核心工具直接发送完整 schema，延迟工具先只公布名称，模型需要时再通过 `ToolSearchTool` 取回 schema。

```text
系统决定哪些工具可 defer
  -> 模型看到核心工具 schema + deferred 工具名列表
  -> 模型调用 ToolSearchTool(select:xxx 或关键词)
  -> ToolSearchTool 返回 tool_reference
  -> 下一轮 API 扫描历史中的 tool_reference
  -> filteredTools 包含已发现的 deferred tool
  -> toolToAPISchema() 发送完整 schema
```

### 7.1 哪些工具会 defer

入口是 `src/tools/ToolSearchTool/prompt.ts` 的 `isDeferredTool(tool)`：

| 条件 | 行为 | 原因 |
|------|------|------|
| `tool.alwaysLoad === true` | 不 defer | 工具必须首轮可见 |
| `tool.isMcp === true` | defer | MCP 工具动态且数量可能很大 |
| `tool.name === ToolSearchTool` | 不 defer | 搜索工具必须先可用 |
| 特定核心通信/Agent 工具 | 不 defer | 模型首轮需要看到完整行为契约 |
| `tool.shouldDefer === true` | defer | 工具作者显式声明可延迟 |

注意：defer 的不是本地工具实现，而是“是否把完整参数 schema 发给模型”。工具代码始终在本地工具集合里。

### 7.2 ToolSearch 是否启用

`isToolSearchEnabled()` 是请求级最终判断，包含：

| 检查 | 说明 |
|------|------|
| 模型支持 | 不支持 `tool_reference` 的模型禁用，例如 Haiku 类模式 |
| 工具可用 | `ToolSearchTool` 必须在当前 tools 列表里，不能被 disallowedTools 移除 |
| 模式 | `ENABLE_TOOL_SEARCH` 控制 `tst` / `tst-auto` / `standard` |
| auto 阈值 | `auto` 模式下，deferred tool schema 超过上下文窗口一定比例才启用 |

`ENABLE_TOOL_SEARCH` 模式：

| 配置 | 模式 | 行为 |
|------|------|------|
| unset | `tst` | 默认启用 |
| `true` | `tst` | 强制启用 |
| `auto` / `auto:N` | `tst-auto` | deferred schema 超过上下文 N% 才启用 |
| `false` / `auto:100` | `standard` | 禁用渐进式加载 |

auto 模式先用 token counting 估算 deferred tool schema 成本；失败时退回字符数启发式。默认阈值是上下文窗口的 10%。

### 7.3 第一轮模型看到什么

`src/services/api/claude.ts` 构造 `filteredTools`：

```typescript
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true
  return discoveredToolNames.has(tool.name)
})
```

含义：

| 工具 | 是否发送完整 schema |
|------|----------------------|
| 非 deferred tool | 是 |
| `ToolSearchTool` | 是 |
| 已发现的 deferred tool | 是 |
| 未发现的 deferred tool | 否 |

未发现的 deferred tool 仍会以“名称列表”形式告诉模型。旧路径在请求前插入：

```xml
<available-deferred-tools>
mcp__github__create_issue
mcp__slack__send_message
</available-deferred-tools>
```

新路径使用 `deferred_tools_delta` attachment，渲染为 system reminder：

```text
The following deferred tools are now available via ToolSearch:
mcp__github__create_issue
mcp__slack__send_message
```

delta 机制会扫描之前已经宣布过的 deferred tools，只发送新增/移除变化，避免工具池变化频繁打破缓存。

### 7.4 怎么知道加载什么

加载什么由模型发起，但搜索由本地确定性算法完成。模型先看到 deferred 工具名和 `ToolSearchTool` 的 prompt，然后根据任务选择：

```json
{ "query": "select:mcp__slack__send_message" }
```

或：

```json
{ "query": "slack send", "max_results": 5 }
```

`ToolSearchTool` 支持两种查询：

| 查询形式 | 行为 |
|----------|------|
| `select:A,B,C` | 精确选择一个或多个工具名 |
| 关键词 | 对 deferred 工具名、`searchHint`、prompt/description 打分 |

关键词打分逻辑：

| 信号 | 分值 |
|------|------|
| 工具名 part 精确匹配 | MCP 12 / 普通 10 |
| 工具名 part 包含匹配 | MCP 6 / 普通 5 |
| `searchHint` 匹配 | 4 |
| full name fallback | 3 |
| prompt/description 匹配 | 2 |

MCP 工具名会按 server/action 拆分：

```text
mcp__slack__send_message
  -> slack, send, message
```

普通工具名按 CamelCase 和下划线拆分：

```text
NotebookEdit
  -> notebook, edit
```

因此 query `"slack send"` 会自然匹配 `mcp__slack__send_message`。

### 7.5 tool_reference 如何让下一轮加载 schema

`ToolSearchTool.mapToolResultToToolResultBlockParam()` 找到匹配时返回 `tool_reference`：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": [
    {
      "type": "tool_reference",
      "tool_name": "mcp__slack__send_message"
    }
  ]
}
```

下一轮 API 请求前，`extractDiscoveredToolNames(messages)` 扫描历史 user message 的 `tool_result.content[]`，收集所有 `tool_reference.tool_name`。`claude.ts` 再把这些名字视为 `discoveredToolNames`，允许它们进入 `filteredTools`，于是 `toolToAPISchema()` 才会发送完整 schema。

```text
Turn N:
  ToolSearchTool("slack send")
  -> tool_result: tool_reference(mcp__slack__send_message)

Turn N+1:
  extractDiscoveredToolNames()
  -> filteredTools includes mcp__slack__send_message
  -> API tools includes full schema
  -> 模型正式调用 mcp__slack__send_message
```

### 7.6 兜底与 compact

如果模型知道 deferred 工具名却没有先加载 schema，typed 参数很容易生成错误。`buildSchemaNotSentHint()` 会在 Zod 校验失败时追加提示：

```text
Load the tool first: call ToolSearch with query "select:<tool.name>", then retry this call.
```

Compact 会删除含 `tool_reference` 的原始消息。为了不丢失“已经加载过哪些 deferred tools”，compact boundary 会保存：

```typescript
boundaryMarker.compactMetadata.preCompactDiscoveredTools = [...]
```

`extractDiscoveredToolNames()` 后续会从 compact boundary 读回这些工具名，避免 compact 后重复丢 schema。

## 8. 待继续

- `SyntheticOutputTool`：用 Tool-as-Schema 实现结构化输出
- Fine-grained tool streaming：工具参数流式传输与提前执行

## 9. 相关章节

- [Agent Loop 设计](agent-loop.md)
- [Context Management](context-management.md)
- [cc-haha learning path §2](../raw/cc-haha-learning-path.md#2-tool-calling--function-calling)
