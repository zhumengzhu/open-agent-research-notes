# 16 · LSP 与代码智能

> **锚点：** `services/lsp/` · `tools/LSPTool/` · `services/api/claude.ts` `shouldDeferLspTool` · `services/lsp/manager.ts`

---

## 1. 角色与边界

LSP 层给 CLI 提供 **IDE 级代码智能**（定义、引用、诊断、rename、symbols），封装为内置 **LSPTool**，与 MCP 并列但走独立 **LSP manager** 进程通信。

| 能力 | LSPTool | FileRead/Grep | MCP LSP 类 server |
|------|---------|---------------|-------------------|
| 语义导航 | ✅ | ❌ | 取决于 server |
| 文本搜索 | ❌ | ✅ | — |
| 进程管理 | CC 内置 manager | 无 | 用户配置 |
| schema 进 API | 可 defer | 始终加载 | 可 defer（tool search 时） |

与 [18 Bridge](./18-bridge-and-ide.md) 分工：Bridge 传 IDE **宿主** selection/diagnostics；LSP 是 CLI 侧 **language server** 子进程。

---

## 2. 架构

```text
startup: reinitializeLspServerManager() [async]
           ↓
manager.ts singleton → createLSPServerManager()
           ↓
LSPServerManager.initialize() → getAllLspServers() [config.ts]
           ↓
per-language LSPServerInstance (stdio/socket)
           ↓
LSPTool tool_use → sendRequest(filePath, method, params)
           ↓
passiveFeedback.ts — publishDiagnostics 等通知
```

### 2.1 核心文件

| 文件 | 职责 |
|------|------|
| `manager.ts` | 全局 singleton、init 状态机（not-started/pending/success/failed） |
| `LSPServerManager.ts` | 多 server 路由、extension → server 映射 |
| `LSPServerInstance.ts` | 单 language server 生命周期 |
| `LSPClient.ts` | JSON-RPC 通信 diagram |
| `config.ts` | 读取用户/项目 LSP server 配置 |
| `LSPDiagnosticRegistry.ts` | 诊断缓存，供 passive 注入 |
| `passiveFeedback.ts` | 注册 notification handler（如不阻塞主 loop 的 diagnostic 更新） |

### 2.2 初始化状态机

`getInitializationStatus()` 返回：

- `not-started` — 尚未调用 init
- `pending` — `reinitializeLspServerManager()` 进行中
- `success` / `failed` — 终态

**调用约定：** `getLspServerManager()` 在 pending 或未 init 时返回 `undefined`；LSPTool 与 API 层必须 **graceful degrade**，不能假设 LSP 已就绪。

Bare mode（`CLAUDE_CODE_SIMPLE`）：通常跳过 LSP init，减依赖。

---

## 3. defer_loading 与 API 层

`services/api/claude.ts` 中 `shouldDeferLspTool()`：

- LSP manager **未 success** 时，LSPTool schema 带 **`defer_loading: true`**
- 避免 server 未就绪时把大 schema 打进首包；就绪后后续 turn 正常加载
- 与 [30 Tool Search](./30-advanced-features-and-experiments.md) 的 MCP defer 策略同族，但 LSP 走 **就绪状态** 而非 token 阈值

Tool search 启用时，deferred LSP + deferred MCP 由 `ToolSearchTool` 按需 discover。

---

## 4. LSPTool 行为

模型传入 operation（goto definition、find references、diagnostics、prepare rename…）：

1. `ensureServerStarted(filePath)` — 按扩展名选 server
2. `openFile` / `changeFile` — 同步 buffer（若未 open）
3. `sendRequest` — LSP method + params
4. 结果 **文本化** 进 `tool_result`（非 raw LSP JSON 给模型）

与 FileEdit 联动：编辑后应 `changeFile`/`saveFile` 保持 LSP 视图一致（工具链内部处理）。

---

## 5. 被动诊断与上下文注入

`passiveFeedback.ts` + `LSPDiagnosticRegistry`：

- Language server push `textDocument/publishDiagnostics`
- 可在不额外 tool call 的情况下，让后续 turn 感知项目诊断
- 与 Bridge 注入的 IDE diagnostics **可能重复**；Bridge 偏「用户当前文件」，LSP 偏「已 open 的 workspace 文件」

---

## 6. 故障与降级

| 场景 | 行为 |
|------|------|
| init failed | LSPTool defer 或返回明确错误；Grep/Read 仍可用 |
| server crash | manager 可 per-server 重启（视实现） |
| 无 server 配置某扩展名 | `getServerForFile` undefined → tool 提示不支持 |

日志：`logForDebugging('[LSP SERVER MANAGER] ...')`；生产问题查 debug 配置。

---

## 7. LSPTool 操作一览

| Operation（概念） | LSP method | 典型用途 |
|-------------------|------------|----------|
| goto definition | textDocument/definition | 跳定义 |
| find references | textDocument/references | 找引用 |
| diagnostics | textDocument/publishDiagnostics | 错误列表 |
| document symbols | textDocument/documentSymbol | 大纲 |
| prepare rename | textDocument/prepareRename | 重命名前检查 |
| rename | textDocument/rename | 批量改名 |

模型通过 LSPTool schema 选 operation + filePath + position；manager 路由到对应 language server。

---

## 8. 配置来源

`services/lsp/config.ts` `getAllLspServers()` — 读用户/项目 JSON 配置 language server 命令行（typescript-language-server、rust-analyzer 等）。

无配置则对应扩展名 **无 server** — tool 返回明确错误，不 crash loop。

---

## 9. 源码带读

1. `services/lsp/manager.ts` — init 入口
2. `services/lsp/LSPServerManager.ts` — `sendRequest` 路由
3. `services/api/claude.ts` — `shouldDeferLspTool`
4. `tools/LSPTool/` — 模型可见 operations

---

## 10. 自测

- [ ] LSP defer 解决什么问题？与 tool search defer 有何不同？
- [ ] LSPTool vs MCPTool 的进程与配置来源差异？
- [ ] init pending 时 LSPTool 调用会怎样？
- [ ] Bridge diagnostics 与 LSP diagnostics 数据源区别？

**关联：** [09 Tools](./09-tools-system.md) · [18 Bridge](./18-bridge-and-ide.md) · [30 Tool Search](./30-advanced-features-and-experiments.md)
