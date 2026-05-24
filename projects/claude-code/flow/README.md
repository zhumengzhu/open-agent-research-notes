# Claude Code 流程索引

> 横切流程导航。模块概念见 [01 架构总览](../01-architecture-overview.md) · 文件锚点见 [26 总图](../26-main-chain-atlas.md)。

---

## 一条 User Turn（总览）

```text
handlePromptSubmit
  → processUserInput (slash? attachments? shouldQuery?)
  → QueryEngine.submitMessage
  → fetchSystemPromptParts + build ToolUseContext
  → query() → queryLoop
       → [compact pipeline]
       → callModel (stream)
       → StreamingToolExecutor / runTools
       → append messages → continue or exit
  → recordTranscript
  → yield to UI / SDK
```

详述：[A1 叙事](../appendix/A1-user-turn-journey.md)

---

## API 流

| 步骤 | 位置 |
|------|------|
| 注入 `callModel` | `query/deps.ts` |
| 流式请求 | `services/api/claude.ts` `queryModelWithStreaming` |
| Model 解析 | `utils/model/model.ts` `getRuntimeMainLoopModel` |
| Fallback | `withRetry` / `executeNonStreamingRequest` |

文档：[07 API 流](../07-api-and-model-stream.md)

---

## Tool 执行

| 步骤 | 位置 |
|------|------|
| 注册 | `tools.ts` → `tools/*` |
| 流式队列 | `StreamingToolExecutor.addTool` |
| 并发分区 | `toolOrchestration.partitionToolCalls` |
| 单 tool | `toolExecution.runToolUse` |
| 权限 | `canUseTool` → `hooks/useCanUseTool` |

文档：[09 Tool 系统](../09-tools-system.md) · [11 Permission](../11-permission-and-hooks.md)

---

## Compact 管道

| 顺序 | 机制 | 目录 |
|------|------|------|
| 1 | tool result budget | query.ts |
| 2 | snip | feature HISTORY_SNIP |
| 3 | microcompact | `services/compact/` |
| 4 | context collapse | feature CONTEXT_COLLAPSE |
| 5 | autocompact | `services/compact/compact.ts` |
| 6 | reactive compact (413) | query.ts 末尾 |

文档：[10 Compaction 专章](../10-compaction-and-context.md)

---

## Session 持久化

| 操作 | 函数 | 文件 |
|------|------|------|
| 增量写入 | `recordTranscript` | `sessionStorage.ts` |
| 子 Agent 链 | `recordSidechainTranscript` | 同上 |
| Resume | `loadConversationForResume` | `conversationRecovery.ts` |
| REPL sync | `useLogMessages` | `hooks/useLogMessages.ts` |

文档：[08 Message/session](../08-message-and-session-persistence.md)

---

## MCP 连接

| 阶段 | 位置 |
|------|------|
| 配置加载 | `services/mcp/config.ts` |
| 连接管理 | `MCPConnectionManager` |
| 合并进 tool 池 | `print.ts` / REPL init |
| 调用 | `MCPTool` |

文档：[14 MCP](../14-mcp-and-external-protocols.md)

---

## System Prompt 组装

```text
fetchSystemPromptParts
  → getSystemPrompt / getUserContext / getSystemContext
  → QueryEngine extras (agent, append, coordinator)
  → appendSystemContext → callModel
```

文档：[13 System prompt](../13-system-prompt-and-context.md)

---

## Headless / SDK

```text
main.tsx (-p) → runHeadless → runHeadlessStreaming → ask() → QueryEngine
StructuredIO → stdout (stream-json)
```

文档：[19 SDK/print](../19-sdk-headless-and-print-mode.md)

---

## Stop 与终止

| 机制 | 位置 |
|------|------|
| 无 tool_use | `needsFollowUp === false` |
| Stop hooks | `query/stopHooks.ts` |
| maxTurns | queryLoop params |
| User abort | AbortController |

文档：[06 query loop](../06-query-agent-loop.md) · [28 Loop 门控](../28-agent-loop-continuation-and-human-gates.md)

---

→ [learning-paths](../learning-paths.md) · [26 总图](../26-main-chain-atlas.md)
