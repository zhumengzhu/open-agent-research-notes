# 14 · MCP 与外部协议

> **锚点：** `services/mcp/*` · `tools/MCPTool` · `AppState.mcp`  
> **官方文档对照：** [code.claude.com MCP](https://code.claude.com/docs/en/mcp)

---

## 1. MCP 在 Claude Code 中的角色

MCP（Model Context Protocol）让 Agent **连接外部 server**，获得：

- 额外 **tools**（动态注册到 API 请求）
- **resources**（`ListMcpResourcesTool` / `ReadMcpResourceTool`）
- **OAuth / stdio / HTTP** 等多种传输

Claude Code 不是「内置几个 MCP」，而是 **运行时连接管理 + 工具池合并**。

---

## 2. 模块地图

| 路径 | 职责 |
|------|------|
| `services/mcp/client.ts` | 客户端连接 |
| `services/mcp/config.ts` | 配置解析 |
| `services/mcp/MCPConnectionManager.tsx` | 连接生命周期（含 UI 态） |
| `services/mcp/auth.ts` / `oauthPort.ts` | OAuth 流程 |
| `services/mcp/types.ts` | `MCPServerConnection` 等类型 |
| `services/mcp/InProcessTransport.ts` | 进程内 transport |
| `services/mcp/channelNotification.ts` | Channel 通知（feature gate） |
| `tools/MCPTool/` | 调用 MCP server 暴露的 tool |
| `tools/McpAuthTool/` | MCP 认证工具 |
| `commands/` 下 `/mcp` | 用户管理 MCP server |

约 **23 文件** 在 `services/mcp/`，属 L1 扩展，但产品依赖度高。

---

## 3. 与 AppState 的集成

`AppState` 持有：

- `mcp.clients` — 连接列表（含 `pending` 状态）
- `mcp.tools` — 合并后的 MCP tool 定义

`query.ts` 调 API 时传入：

```typescript
mcpTools: appState.mcp.tools,
hasPendingMcpServers: appState.mcp.clients.some(c => c.type === 'pending'),
```

**pending server：** 模型可能在 server 未就绪时发起 MCP 调用；API 层需知悉 pending 状态。

---

## 4. 工具池合并流程（概念）

```text
assembleToolPool (tools.ts)
  → 内置 tools (~42)
  → filterToolsByDenyRules / mergeAndFilterTools
  → MCP tools 从 AppState 注入
  → 传入 QueryEngine / query loop
```

Headless：`print.ts` 在 `runHeadless` 启动时 **安装插件、连接 MCP**（`installPluginsAndApplyMcpInBackground` 等），再进入 `ask()`。

交互式：REPL 启动 + `/mcp` 命令动态变更，经 `setAppState` 触发重连。

---

## 5. 配置来源

| 来源 | 说明 |
|------|------|
| 项目 / 用户 MCP config | JSON 配置 |
| `--mcp-config` | CLI 注入 |
| `--strict-mcp-config` | 仅用 CLI 配置 |
| SDK `sdkMcpConfigs` | print 模式 SDK server |
| Plugin 同步 | 与 plugins 安装顺序相关 |

环境变量展开见 `services/mcp/envExpansion.ts`（注意 allowlist 安全模型）。

---

## 6. MCPTool 执行路径

1. 模型调用 `MCPTool`（或 dynamic MCP tool name）
2. `canUseTool` — MCP 可能有独立 permission
3. `runToolUse` → MCP client 发请求
4. 结果作为 `tool_result` user 块回到 loop

进度类型：`MCPProgress`（见 `Tool.ts` re-export）。

---

## 7. 与 System Prompt 的关系

`fetchSystemPromptParts` 接收 `mcpClients`，`getSystemPrompt` 会把 **已连接 MCP 能力**写进 system。连接变化 → prompt 变 → cache 变。

---

## 8. 与 LSP 的分工

| 机制 | 目录 | 用途 |
|------|------|------|
| MCP | `services/mcp/` | 通用外部工具协议 |
| LSP | `services/lsp/` + `LSPTool` | 语言服务器（定义、引用、诊断） |

二者都进 tool 池，但初始化与 defer 逻辑不同（API 层 `shouldDeferLspTool`）。

### 8.1 Tool Search 与 MCP defer

当 [30 Tool Search](./30-advanced-features-and-experiments.md) 启用时，MCP tool schema 常带 **`defer_loading: true`**，不占用首包 context；模型通过 `ToolSearchTool` 按需加载。compact 路径在 `services/compact/compact.ts` 重复同一逻辑。

Coordinator worker 的 MCP 列表会注入 `getCoordinatorUserContext` [21]。

### 8.2 OAuth 与认证

| 模块 | 职责 |
|------|------|
| `services/mcp/auth.ts` | OAuth 流程状态机 |
| `oauthPort.ts` | 本地 callback 端口 |
| `McpAuthTool` | 模型/用户触发登录 |

stdio MCP 用 env 展开（`envExpansion.ts`）；HTTP MCP 可能走 OAuth PKCE。headless 需 `--permission-prompt-tool` 或预配置 token [19]。

### 8.3 连接生命周期

```text
config 解析 → pending client 占位
  → 后台 connect（stdio spawn / HTTP handshake）
  → ready → tools 合并进 AppState
  → /mcp 或 settings 变更 → 重连
shutdown → client.close()
```

**pending 期间：** 模型若调 MCP tool，可能失败或排队；API 层 `hasPendingMcpServers` 告知模型 server 未就绪。

---

## 9. 自测

- [ ] MCP tools 如何进入 `callModel` 的 tools 参数？
- [ ] `pending` client 对 API 请求有何影响？
- [ ] headless 与 REPL 下 MCP 连接时机有何不同？
- [ ] `/mcp` 命令改的是 AppState 还是直接改 client？

**关联：** [13 System prompt](./13-system-prompt-and-context.md) · [19 SDK/print](./19-sdk-headless-and-print-mode.md) · [15 skills/plugins](./15-skills-and-plugins.md)
