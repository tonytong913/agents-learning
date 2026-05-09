# MCP 协议集成

> 状态：🟢 完成 | 最后更新：2026-05-09
> 相关源文件（cc-haha）：`src/services/mcp/client.ts`, `src/services/mcp/config.ts`, `src/services/mcp/auth.ts`, `src/services/mcp/useManageMCPConnections.ts`, `src/services/mcp/channelPermissions.ts`, `src/tools/MCPTool/MCPTool.ts`, `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`, `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`, `src/tools.ts`, `src/entrypoints/mcp.ts`, `src/vendor/computer-use-mcp/`

## 1. Why：为什么需要 MCP

MCP 在 cc-haha 中解决的是 Agent 外部能力接入碎片化的问题。

如果没有 MCP，Agent 要接 GitHub、Slack、数据库、浏览器、桌面 GUI、企业内部系统，每一种能力都要写一套私有集成：工具发现、参数 schema、调用、认证、权限、刷新和错误处理都不统一。系统会变成一组不可复用的 adapter，工具越多越难维护。

所以 cc-haha 把 MCP 放在工具系统的外部能力扩展层：远程 MCP server 通过标准协议暴露 `tools`、`resources`、`prompts`，cc-haha 作为 MCP client 消费这些能力；同时 cc-haha 也能反过来作为 MCP server，把自身工具暴露给其他 MCP client。

```text
外部 MCP Server
  -> connectToServer()
  -> tools/list
  -> fetchToolsForClient()
  -> MCPTool 模板适配成本地 Tool
  -> assembleToolPool()
  -> Agent Loop 正常 tool_use / tool_result
```

```text
cc-haha 作为 MCP Server
  -> startMCPServer()
  -> ListToolsRequestSchema 暴露本地工具
  -> CallToolRequestSchema 执行本地工具
  -> MCP CallToolResult 返回给外部 Client
```

## 2. What：MCP 集成解决的核心问题

MCP 集成不是简单“调用 SDK”。在 cc-haha 中，它要解决五类工程问题：

1. **配置治理**：一个 MCP server 从哪里声明，多个来源如何合并、覆盖、禁用、审批和去重。
2. **连接治理**：`stdio`、`sse`、`http`、`ws`、`sdk`、`claudeai-proxy` 等 transport 如何统一成 `MCPServerConnection`。
3. **工具适配**：远程 MCP tool 如何变成 cc-haha 内部 `Tool`，进入原有 tool calling 编排。
4. **安全与认证**：OAuth、headers、project approval、enterprise policy、channel permission 如何共同限制外部能力。
5. **动态刷新**：server 的 tools / resources / prompts 变化后，Agent 如何拿到新列表，而不是一直使用旧 cache。

核心抽象包括：

- `ScopedMcpServerConfig`：描述 server 的连接方式、来源 scope 和认证配置。
- `MCPServerConnection`：描述 server 当前是 `connected`、`failed`、`needs-auth`、`pending` 还是 `disabled`。
- `MCPTool`：远程 tool 到本地 `Tool` 接口的适配模板。
- `AppState.mcp`：保存当前 MCP clients、tools、commands 和 resources。
- notification handlers：处理 `tools/list_changed`、`resources/list_changed`、`prompts/list_changed`。

## 3. How：配置加载与去重

配置加载从 `src/services/mcp/config.ts` 开始。cc-haha 会合并多个 scope：

| Scope | 来源 | 说明 |
| --- | --- | --- |
| `enterprise` | `managed-mcp.json` | 企业托管配置；存在时可以接管全部 MCP 配置 |
| `user` | 全局用户配置 | 用户级 MCP server |
| `project` | `.mcp.json` | 从文件系统根目录到当前 CWD 逐层读取，越靠近 CWD 优先 |
| `local` | 当前项目本地配置 | 本地覆盖 |
| `plugin` | 插件声明 | 插件提供的 MCP server |
| `claudeai` | Claude.ai connector | 远程 connector 配置 |

project scope 不是天然可信。`getClaudeCodeMcpConfigs()` 只会接入 `getProjectMcpServerStatus(name) === 'approved'` 的项目 server，避免项目里的 `.mcp.json` 悄悄启动本地进程或连接外部系统。

plugin 和 claude.ai connector 还会做内容级去重。这个去重不是协议层 server ID，而是 cc-haha 自己算的 signature：

```text
stdio signature = stdio:[command, ...args]
url signature   = url:<server url>
```

例如手动配置和插件都声明：

```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"]
}
```

它们的 signature 都类似：

```text
stdio:["npx","-y","@modelcontextprotocol/server-github"]
```

即使 server name 不同，cc-haha 也认为它们会启动同一个底层 server，手动配置优先，插件配置被 suppress。

远程 server 则按 URL 去重：

```text
url:https://mcp.slack.com/mcp
```

这个 signature 只是去重启发式，不是强唯一标识。stdio signature 忽略 env，URL signature 忽略 headers；真正用于 AppState、工具命名和权限规则的 key 仍然是 server name。

## 4. How：连接建立与生命周期

`src/services/mcp/client.ts` 的 `connectToServer()` 根据 server config 选择 transport 并建立连接：

- `stdio`：启动本地 MCP 进程，传入 `command`、`args`、`env`。
- `sse`：使用 `SSEClientTransport`。
- `http`：使用 `StreamableHTTPClientTransport`。
- `ws`：使用 WebSocket transport。
- `sse-ide` / `ws-ide`：IDE 内部连接。
- `claudeai-proxy`：通过 Claude.ai MCP proxy 连接 connector。
- in-process：Claude in Chrome 和 Computer Use MCP 走进程内 transport，避免普通子进程开销和额外包装。

连接成功后，cc-haha 读取 server capabilities、server version 和 instructions。长 instructions 会截断，避免外部 server 把大量文档塞进上下文。

连接关闭时，`onclose` 不只更新状态，还会清理多层缓存：

- `connectToServer.cache`
- `fetchToolsForClient.cache`
- `fetchResourcesForClient.cache`
- `fetchCommandsForClient.cache`
- `fetchMcpSkillsForClient.cache`（开启 `MCP_SKILLS` 时）

这样下一次工具调用、资源读取或命令刷新会重新连接并拉取新状态，而不是复用旧连接上的 stale 数据。

## 5. How：远程工具适配成本地 Tool

`fetchToolsForClient()` 调用 MCP `tools/list`，把远程工具转换为 cc-haha 内部 `Tool`：

- 工具名通过 `buildMcpToolName(server, tool)` 做命名空间隔离，常见形式是 `mcp__server__tool`。
- `tool.inputSchema` 保留为 `inputJSONSchema`，因为 MCP tool 已经提供 JSON Schema。
- `annotations.readOnlyHint` 映射到 `isReadOnly()` 和 `isConcurrencySafe()`，让远程只读工具参与并发编排。
- `annotations.destructiveHint` / `openWorldHint` 进入安全判断，补足本地权限系统无法静态理解远程工具的问题。
- `checkPermissions()` 默认返回 passthrough，并建议添加本地 allow rule；真实放行仍由统一权限系统控制。
- `call()` 内部最终执行 MCP `tools/call`，再把 MCP result 转回 cc-haha 的 tool result。

`src/tools/MCPTool/MCPTool.ts` 是适配模板。它声明 `isMcp: true`、宽松输入 schema、统一结果渲染和默认权限行为。`fetchToolsForClient()` 在运行时覆盖名称、描述、schema、权限、调用逻辑和 UI 展示。

MCP `tools`、`resources`、`prompts` 的区别：

| MCP 能力 | 含义 | 例子 | cc-haha 中的落点 |
| --- | --- | --- | --- |
| `tools` | Agent 可以主动调用的函数 | `create_issue(owner, repo, title)` | 包装成内部 `Tool` |
| `resources` | 可寻址的外部数据或上下文对象 | `github://repo/x/issues/123`, `skill://team/review` | 通过资源工具读取；也可能生成 MCP skills |
| `prompts` | server 提供的可复用提示词模板或命令 | `summarize_issue(issue_number)` | 转成 MCP command |

## 6. How：Tool 池合并与 Agent Loop 刷新

`src/tools.ts` 的 `assembleToolPool()` 是本地工具 + MCP 工具的合并入口：

- 先取内置工具，再过滤被 deny rule 命中的 MCP 工具。
- 内置工具保持连续前缀，MCP 工具单独排序，避免 MCP 工具插入内置工具中间导致 prompt cache key 漂移。
- 用 `uniqBy(..., 'name')` 去重，名称冲突时内置工具优先。

`src/query.ts` 在工具执行和附件注入后会调用 `refreshTools()`。这让新连接的 MCP server，或 `tools/list_changed` 后刷新的远程工具，能在后续 turn 进入 Agent 可见工具池。

## 7. How：动态刷新与 MCP Skills

`src/services/mcp/useManageMCPConnections.ts` 会根据 server capabilities 注册三类 notification handler：

- `tools/list_changed`
- `prompts/list_changed`
- `resources/list_changed`

收到通知后，它不会重启整个 MCP 系统，而是清对应 cache、重新 fetch 对应列表，然后局部更新 `AppState.mcp`。

`resources/list_changed` 需要额外处理 MCP skills。这里的 `MCP_SKILLS` 不是 MCP 协议里的第四种能力，而是 cc-haha 在 MCP resources 上做的集成：MCP server 可以通过 `skill://...` resource 分发远程 skill，cc-haha 把这些 resource 转成 SkillTool 可发现、可调用的 skill。

假设一个 Notion MCP server 暴露：

```text
skill://notion/write-weekly-report
skill://notion/summarize-meeting-notes
docs://notion/workspace/schema
```

其中 `skill://notion/write-weekly-report` 可能包含一份类似 `SKILL.md` 的流程说明。cc-haha 会把它接入 Skill 系统。

所以当 `resources/list_changed` 到来时，可能意味着 `skill://...` 被新增、删除或更新。cc-haha 会：

- 删除 `fetchMcpSkillsForClient` 的 server cache，下一次重新拉最新 MCP skills。
- 删除 `fetchCommandsForClient` cache，避免 resources 刷新和 prompts 刷新并发时互相覆盖 commands。
- 调用 `clearSkillIndexCache()`，让下一次 skill discovery 基于最新 skills 重建索引。

如果只刷新 resource 列表但不清 skill search index，模型可能继续命中过期 skill，或者看不到新加入的 MCP skill。

## 8. How：OAuth、Headers 与不同 Server 的认证

远程 `http` / `sse` server 主要通过 `src/services/mcp/auth.ts` 的 `ClaudeAuthProvider` 接入 MCP SDK OAuth。它负责：

- **metadata discovery**：发现 OAuth 登录地址、token 地址、revocation 地址。配置了 `authServerMetadataUrl` 时优先使用配置；否则按 OAuth discovery 规则从 MCP server URL 推导。
- **PKCE callback**：为本地 CLI / desktop client 生成 `code_verifier` / `code_challenge`，避免 authorization code 被截获后可直接换 token。
- **本地 callback server**：临时监听 `127.0.0.1:<port>/callback`，浏览器登录后带 `code` 和 `state` 跳回本地。
- **secure storage**：把 access token、refresh token、expiresAt、scope、client info、discovery state 保存到系统安全存储。
- **refresh token 主动刷新**：access token 快过期时提前刷新，避免工具调用时才失败。
- **step-up auth**：遇到 `403 insufficient_scope` 时重新授权更高 scope；refresh 不能提升 scope，所以不能只刷新 token。
- **401/403 处理**：401 通常表示未登录或 token 无效；403 通常表示已登录但权限不够。

不是所有 server 都走 `ClaudeAuthProvider`：

| Server 类型 | 认证方式 |
| --- | --- |
| `stdio` | 不走 `ClaudeAuthProvider`；通常通过 `env` 或本地配置把 API token 传给子进程 |
| `http` / `sse` | `ClaudeAuthProvider` + 静态 `headers` / 动态 `headersHelper` |
| `ws` | 主要用 `headers` / `headersHelper`，不走 `ClaudeAuthProvider` |
| `sse-ide` / `ws-ide` | IDE 内部 token 或内部通道 |
| `claudeai-proxy` | 使用 Claude.ai OAuth token 连接 Anthropic MCP proxy |
| `sdk` | SDK 管理的进程内 server，不由 CLI 直接认证 |
| Computer Use / Chrome in-process | 本地权限控制，不是 OAuth |

`headersHelper` 也有安全边界。project / local scope 下执行动态 headers helper 前要确认 workspace trust，避免项目配置在用户未信任工作区前执行任意脚本。

## 9. How：Channel Permissions

channel MCP server 让 Telegram、iMessage、Discord 这类通道把消息推回 cc-haha。普通 channel 消息已经敏感，permission relay 更敏感，所以需要额外条件。

`src/services/mcp/channelPermissions.ts` 要求 permission relay server 同时满足：

- 已连接。
- 在当前 session 的 `--channels` allowlist 里。
- 声明 `capabilities.experimental['claude/channel']`。
- 声明 `capabilities.experimental['claude/channel/permission']`。

权限批准不是 Claude Code 从普通聊天文本里 regex “yes” 得到的，而是 channel server 解析用户回复后发结构化通知：

```text
notifications/claude/channel/permission
```

这减少普通消息误触发，但信任边界转移到了 channel server。如果 channel server 被攻破，它可以伪造结构化 approval；cc-haha 接受这个风险的前提是 channel server 本身已经有持续注入对话的能力。

## 10. How：Computer Use MCP 的特殊包装

Computer Use 看起来是 MCP server，但不是普通外部 server。`connectToServer()` 识别到 Computer Use MCP server 后，走 in-process transport；`fetchToolsForClient()` 再给它注入 `.call()` override。

真实执行链路在：

- `src/vendor/computer-use-mcp/mcpServer.ts`
- `src/vendor/computer-use-mcp/tools.ts`
- `src/utils/computerUse/wrapper.tsx`

它的安全边界不靠 MCP 协议本身，而靠 host wrapper 和 executor：

- `request_access` 先请求用户批准可操作 app。
- app allowlist 限制可见和可操作范围。
- macOS 原生截图过滤时，只显示已授权 app 和桌面。
- frontmost app gate 防止向未授权前台 app 点击或输入。
- session lock 避免多个 Claude session 同时控制电脑。
- Escape hotkey 允许用户全局中断。
- 坐标模式在 tool schema 构建时冻结，模型只看到一种坐标语义。

因此 Computer Use 是“用 MCP 形态暴露 GUI 操作能力”，但真正的安全模型在本地 host 包装层。

## 11. Server 侧：cc-haha 暴露自身工具

`src/entrypoints/mcp.ts` 的 `startMCPServer()` 使用 MCP SDK 创建 stdio server，并注册两个 handler：

| Handler | 作用 |
| --- | --- |
| `ListToolsRequestSchema` | 调用 `getTools()`，把本地工具的 prompt、inputSchema、可用 outputSchema 暴露为 MCP tools |
| `CallToolRequestSchema` | 查找本地工具，构造非交互 `ToolUseContext`，执行 `validateInput()` 和 `call()`，再转换为 MCP `CallToolResult` |

这条链路体现 MCP 的双向性：cc-haha 既能消费外部工具，也能把自己的工具能力提供给其他 MCP client。

当前 server 侧还有一个明确边界：`mcp.ts` 只重新暴露 cc-haha 本地工具，没有把已经接入的外部 MCP tools 再转发出去。代码里保留了 TODO，但这个限制是合理的安全默认值，可以避免 MCP 工具递归转发后权限归属不清。

## 12. How：MCP Tool 调用结果转换

MCP server 返回的是协议层 `CallToolResult`，不是 cc-haha 内部工具结果。`src/services/mcp/client.ts` 的 `callMCPTool()` 负责调用 MCP SDK 的 `client.callTool()`，然后把结果归一化。

调用链路：

```text
MCPTool.call()
  -> ensureConnectedClient()
  -> callMCPToolWithUrlElicitationRetry()
  -> callMCPTool()
  -> client.callTool({ name, arguments, _meta })
  -> processMCPResult()
  -> 本地 tool_result
```

`callMCPTool()` 做了几件关键工程处理：

- 设置独立 timeout，避免 SDK transport 异常时工具调用永久挂起。
- 把 MCP progress 转成 cc-haha 的 `mcp_progress`，供 UI 展示长任务进度。
- 遇到 `isError` 时，从 MCP content 中提取错误文本并抛出本地错误。
- 遇到 401 / session expired 时，清连接缓存并交给上层恢复或重连。
- 成功时保留 `_meta` 和 `structuredContent`，通过 `mcpMeta` 附在本地工具结果上。

结果转换的核心在 `transformResultContent()` / `processMCPResult()`。MCP content 可能包含 `text`、`image`、`audio`、`resource`、`resource_link`、`structuredContent` 等类型，cc-haha 不会无脑把它们都 stringify 到上下文：

| MCP 内容 | cc-haha 处理 |
| --- | --- |
| `text` | 直接转 text block |
| `image` | resize / downsample 后转模型可接收的 image block |
| `audio` | 解码 base64，写入 tool-results 目录，给模型返回文件路径说明 |
| `resource.text` | 加来源前缀后转 text block |
| `resource.blob` | 图片按 image 处理，其他二进制写磁盘 |
| `resource_link` | 转成可读的链接说明文本 |
| `structuredContent` | 转 JSON 文本，并推断 compact schema |

这个设计的重点是保护上下文窗口。外部 MCP server 可能返回 PDF、音频、图片或其他大 blob；如果直接把 base64 塞进消息，会快速消耗 token，甚至导致 `prompt_too_long`。所以 cc-haha 对二进制结果做落盘，只把可操作路径和必要说明交给模型。

## 13. How：MCP Resource 读取链路

MCP `resources` 是可寻址外部数据，不是函数调用。cc-haha 没有让模型直接发 `resources/read`，而是把 resource 能力包装成两个普通本地工具：

| 工具 | 作用 | 设计属性 |
| --- | --- | --- |
| `ListMcpResourcesTool` | 列出已连接 MCP servers 暴露的 resources | 只读、并发安全、`shouldDefer=true` |
| `ReadMcpResourceTool` | 按 `server + uri` 读取指定 resource | 只读、并发安全、`shouldDefer=true` |

`ListMcpResourcesTool` 使用 `fetchResourcesForClient()`，后者调用 MCP `resources/list` 并把 server name 附到每个 resource 上。结果按 server name 做 LRU cache，`resources/list_changed` 或连接关闭时清 cache。

`ReadMcpResourceTool` 的读取链路：

```text
模型调用 ReadMcpResourceTool({ server, uri })
  -> 从 mcpClients 找 server
  -> 校验 server connected 且支持 resources
  -> ensureConnectedClient()
  -> client.request({ method: "resources/read", params: { uri } })
  -> ReadResourceResultSchema 校验
  -> text 直接返回，blob 写入 tool-results 后返回路径说明
```

这让 MCP resource 进入原有 Tool Calling 编排：权限、并发、defer loading、tool_result 配对都沿用同一套机制。

MCP Skills 也是基于 resources 的上层扩展。`skill://...` resource 可以分发远程 skill，因此 `resources/list_changed` 不只是资源列表变化，也可能意味着 skill discovery 索引需要失效重建。

## 14. Skills 与 MCP 的关系

Skills 和 MCP 不是严格替代关系。更准确地说：

```text
Skill = 知识 / 流程 / 方法：模型应该怎么做
MCP   = 外部能力 / 数据 / 工具连接：模型能调用什么
```

很多早期 MCP server 如果只是包装 prompt 或流程说明，确实更适合改成 Skill。但真正需要访问 GitHub、Slack、数据库、Jira、浏览器、桌面 GUI 的场景，仍然需要 MCP 或其他工具连接层。

组合方式通常是：

```text
MCP 提供 jira.get_ticket / github.search_prs / datadog.query_logs
Skill 规定线上 bug triage 的稳定流程
```

没有 MCP，Skill 只能描述流程但拿不到外部数据；没有 Skill，MCP 只给一堆工具，模型可能乱查、漏查或顺序不稳定。

## 15. 结论：MCP 在 cc-haha 中的位置

cc-haha 把 MCP 放在“外部能力协议层”，但不让它绕过本地 Agent 架构。无论 MCP server 暴露的是 tool、resource 还是 prompt，最后都会被适配到 cc-haha 熟悉的内部形态：

```text
MCP tools
  -> MCPTool
  -> 本地 Tool 接口
  -> runTools / StreamingToolExecutor
  -> tool_result

MCP resources
  -> ListMcpResourcesTool / ReadMcpResourceTool
  -> 本地 Tool 接口
  -> tool_result

MCP prompts
  -> Command
  -> slash command / prompt command
```

所以学习 MCP 这一章，重点不是记住某个 transport API，而是理解协议标准化和产品工程之间的适配层：配置治理、连接生命周期、权限认证、动态刷新、结果归一化、二进制输出保护，最后统一回到 Agent Loop。

## 16. 面试速记

**Q：MCP tool 和本地 tool 在 Agent Loop 里有什么区别？**

MCP tool 在协议来源上是远程 server 暴露的能力，但进入 cc-haha 后会被 `MCPTool` 包装成本地 `Tool`。它仍然走同一套 schema 暴露、权限检查、并发编排、tool_result 回传。差异主要在适配层：schema 来自 MCP JSON Schema，调用通过 `tools/call`，结果需要从 MCP `CallToolResult` 转成本地消息块。

**Q：为什么 MCP result 不能直接 stringify 进上下文？**

因为 MCP content 可能包含图片、音频、resource blob 或大结构化数据。cc-haha 会按类型处理：文本直接进上下文，图片压缩成 image block，非图片二进制写入 tool-results 目录，只返回文件路径说明。这能避免 base64 污染上下文窗口。

**Q：cc-haha 如何体现 MCP 的双向性？**

Client 侧连接外部 MCP server，把远程 tools/resources/prompts 适配成本地 Tool 或 Command；Server 侧通过 `src/entrypoints/mcp.ts` 把自身本地工具暴露为 MCP `tools/list` 和 `tools/call`。两边都是协议对象和内部 Tool 抽象之间的转换。

## 17. 待后续延伸

- 在 §6 Streaming 中回看 MCP progress 如何进入实时 UI。
- 在 §7 Memory / Skills 中继续展开 MCP Skills 与 skill discovery。
- 在 §9 Permissions 中复盘 MCP permissions、channel permission relay 和 Computer Use 安全边界。

## 参考

- [cc-haha-learning-path.md §3](../raw/cc-haha-learning-path.md#3-mcp-协议-model-context-protocol)
- `src/services/mcp/config.ts` — MCP 配置加载、scope 合并、policy 和去重
- `src/services/mcp/client.ts` — MCP client 连接、工具发现、工具调用
- `src/services/mcp/auth.ts` — OAuth、token refresh、step-up auth
- `src/services/mcp/useManageMCPConnections.ts` — 连接状态管理与 list_changed 刷新
- `src/services/mcp/channelPermissions.ts` — channel permission relay
- `src/tools/MCPTool/MCPTool.ts` — MCP 工具适配模板
- `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts` — MCP resources 列表工具
- `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` — MCP resource 读取工具
- `src/utils/mcpOutputStorage.ts` — MCP 二进制输出持久化
- `src/tools.ts` — 内置工具与 MCP 工具合并
- `src/entrypoints/mcp.ts` — cc-haha 作为 MCP server 暴露工具
- `src/vendor/computer-use-mcp/` — Computer Use MCP server 与 GUI 操作能力
