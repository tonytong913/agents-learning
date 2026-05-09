# MCP 协议集成

> 状态：🟡 进行中 | 最后更新：2026-05-09
> 相关源文件（cc-haha）：`src/services/mcp/client.ts`, `src/services/mcp/config.ts`, `src/tools/MCPTool/MCPTool.ts`, `src/tools.ts`, `src/query.ts`, `src/entrypoints/mcp.ts`

## 1. 学习范围

MCP 在 cc-haha 中不是旁路功能，而是工具系统的外部能力扩展层。学习这一章要同时看两条链路：

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

## 2. Client 侧：远程工具进入本地 Tool 池

`src/services/mcp/client.ts` 的 `connectToServer()` 根据 server config 选择 transport 并建立连接。连接成功后，`fetchToolsForClient()` 发起 `tools/list`，把外部 MCP 工具转换为 cc-haha 内部 `Tool`：

- 工具名通过 `buildMcpToolName(server, tool)` 做命名空间隔离，避免和本地工具冲突。
- `tool.inputSchema` 被保留为 `inputJSONSchema`，因为 MCP 工具本身已经提供 JSON Schema。
- `annotations.readOnlyHint` 同时影响 `isReadOnly()` 和 `isConcurrencySafe()`，让远程只读工具可以参与并发编排。
- `annotations.destructiveHint` / `openWorldHint` 进入安全判断，补足本地权限系统无法静态理解远程工具的问题。
- `checkPermissions()` 默认返回 passthrough，并建议添加本地 allow rule；真实放行仍由统一权限系统控制。

`src/tools/MCPTool/MCPTool.ts` 是适配模板：它声明 `isMcp: true`、宽松输入 schema、统一结果渲染和默认权限行为。`fetchToolsForClient()` 在运行时覆盖名称、描述、schema、权限、调用逻辑和 UI 展示。

## 3. Tool 池合并与 Agent Loop 刷新

`src/tools.ts` 的 `assembleToolPool()` 是本地工具 + MCP 工具的合并入口：

- 先取内置工具，再过滤被 deny rule 命中的 MCP 工具。
- 内置工具保持连续前缀，MCP 工具单独排序，避免 MCP 工具插入内置工具中间导致 prompt cache key 漂移。
- 用 `uniqBy(..., 'name')` 去重，名称冲突时内置工具优先。

`src/query.ts` 在工具执行和附件注入后会调用 `refreshTools()`。这让新连接的 MCP Server 或 `tools/list_changed` 后刷新的远程工具，能在后续 turn 进入 Agent 可见工具池。

## 4. Server 侧：cc-haha 暴露自身工具

`src/entrypoints/mcp.ts` 的 `startMCPServer()` 使用 MCP SDK 创建 stdio server，并注册两个 handler：

| Handler | 作用 |
| --- | --- |
| `ListToolsRequestSchema` | 调用 `getTools()`，把本地工具的 prompt、inputSchema、可用 outputSchema 暴露为 MCP tools |
| `CallToolRequestSchema` | 查找本地工具，构造非交互 `ToolUseContext`，执行 `validateInput()` 和 `call()`，再转换为 MCP `CallToolResult` |

这条链路体现了 MCP 的双向性：cc-haha 既能消费外部工具，也能把自己的工具能力提供给其他 MCP Client。

## 5. 待继续

- [ ] 配置加载链路：全局 / 项目 / managed / plugin MCP server 如何合并与去重。
- [ ] OAuth 与远程 Server：`auth.ts`、header 注入、step-up auth、token refresh。
- [ ] `tools/list_changed`、`resources/list_changed`、`prompts/list_changed` 的刷新机制。
- [ ] MCP 权限与 channel permissions 的细节。
- [ ] Computer Use MCP 的特殊包装与安全边界。

## 参考

- [cc-haha-learning-path.md §3](../raw/cc-haha-learning-path.md#3-mcp-协议-model-context-protocol)
- `src/services/mcp/client.ts` — MCP client 连接、工具发现、工具调用
- `src/tools/MCPTool/MCPTool.ts` — MCP 工具适配模板
- `src/tools.ts` — 内置工具与 MCP 工具合并
- `src/entrypoints/mcp.ts` — cc-haha 作为 MCP server 暴露工具
