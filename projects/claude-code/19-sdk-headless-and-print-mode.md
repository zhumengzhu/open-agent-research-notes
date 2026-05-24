# 19 · SDK、Headless 与 print 模式

> **锚点：** `cli/print.ts` · `QueryEngine.ask` · `StructuredIO` / `RemoteIO` · `entrypoints/agentSdkTypes.ts`

---

## 1. 三种运行形态

| 形态 | 入口 | UI | 典型用户 |
|------|------|-----|----------|
| 交互式 REPL | `main.tsx` → `launchRepl` | Ink React | 终端开发者 |
| Headless print | `main.tsx` → `runHeadless` | 无 | 脚本、CI、管道 |
| SDK 嵌入 | print + `sdkUrl` / stream-json | 无 | Python/TS SDK |

**内核统一：** 都到 **`QueryEngine.ask()`** 或 REPL **`submitMessage` → `query()`**。

---

## 2. `runHeadless` 职责

`cli/print.ts` ~5500 行 — headless **总控**：

1. Managed settings 订阅（无 React 的替代）
2. Remote settings sync [22]
3. Coordinator `matchSessionMode` [21]
4. Session resume — `loadInitialMessages` [08]
5. MCP / plugins 后台安装 [14]
6. Tool pool + permission 回调
7. **`runHeadlessStreaming`** 消费 generator
8. Background task 等待 + shutdown dump [23]
9. SIGINT → abort + graceful shutdown

**读法：** 扫 export/函数目录 + 跟一条 `ask()`，不必线性通读。

---

## 3. Streaming 管道

```text
runHeadlessStreaming
  → ask({ prompt, canUseTool, tools, ... })
       → new QueryEngine(...)
       → yield* submitMessage(prompt)
  → map Message → SDKMessage
  → StructuredIO.outbound (stdout NDJSON)
```

`ask()` 是 **便捷 wrapper** — 每次可新建 engine；REPL 用 **长生命周期** engine。

---

## 4. 输出格式

| format | 行为 |
|--------|------|
| `text` | 最终 assistant 文本 |
| `json` | 单 JSON 结果对象 |
| `stream-json` | 逐条 **SDKMessage** NDJSON |

### 4.1 SDKMessage 类型（概念）

| type | 时机 |
|------|------|
| `system` | init、compact boundary |
| `user` | user 输入（`--replay-user-messages` 可选回显） |
| `assistant` | 流式 partial / complete |
| `result` | turn 结束汇总（cost、duration） |
| `error` | 致命/可恢复错误 |

定义见 `entrypoints/agentSdkTypes.ts`；与内部 `Message` 差一层 mapping [08]。

### 4.2 Guard

`installStreamJsonStdoutGuard` — 防止 debug `console.log` 污染 stdout JSONL。

---

## 5. Permission（无 UI）

| 机制 | 用途 |
|------|------|
| `--permission-prompt-tool` | MCP tool 代问 allow/deny |
| `--allowedTools` / `--disallowedTools` | 静态 allowlist |
| `permissionMode` settings | plan / bypass 等 [11] |
| `createCanUseToolWithPermissionPrompt` | 组装回调链 |

Headless **无 React** — 不能 pop Ink dialog；必须 pre-approve 或 MCP prompt tool。

Coordinator / background：`shouldAvoidPermissionPrompts` 路径 [21][23]。

---

## 6. Resume / Continue / Fork

| Flag | 行为 |
|------|------|
| `--continue` | 最近 session JSONL |
| `--resume <id\|path>` | 指定 session |
| `--fork-session` | 分支新 session id |
| `--no-session-persistence` | 不落盘 |
| `--rewind-files <uuid>` | 按 file history 恢复 [08] |

---

## 7. Remote / SDK URL

`--sdk-url` / `CLAUDE_CODE_REMOTE`：

- `RemoteIO` 替代 stdin/stdout 本地绑定
- API timeout 120s、session activity keep-alive [22]
- Memory 需 `CLAUDE_CODE_REMOTE_MEMORY_DIR` [29]
- `buildCCRv2SdkUrl` / workSecret [18]

---

## 8. 与 REPL 对比

| 维度 | REPL | Headless |
|------|------|----------|
| QueryEngine 生命周期 | 长 | 每 ask 可新建 |
| Permission | React UI | flags / MCP tool |
| 输出 | Ink 渲染 | StructuredIO |
| Trust dialog | 有 | **跳过**（help 警告） |
| Bridge | `initReplBridge` [18] | 通常无 |
| Background 等待 | UI 指示 | print 显式 wait [23] |

---

## 9. Turn 中断与 orphan

print 维护 `TurnInterruptionState`：

- SIGINT mid-turn → 记录中断点
- Orphan permission request → 下次 resume 处理
- stream-json 消费者应处理 **partial turn** 事件

---

## 10. Turn 生命周期（stream-json）

```text
init → user (optional replay) → assistant (partial→complete)
  → tool 循环 … → result { total_cost_usd, duration } → exit
```

Remote/CCR：`RemoteIO` 替代本地 stdin/stdout，事件语义同 StructuredIO [22]。

---

## 11. SDK 集成 Checklist

- [ ] 使用 `stream-json` + guard stdout
- [ ] 处理 `result.total_cost_usd` [24]
- [ ] permission：MCP prompt tool 或 allowlist
- [ ] resume：`--continue` 或显式 session id
- [ ] remote：settings sync + memory dir

---

## 12. 自测

- [ ] `runHeadless` 与 `launchRepl` 共享哪些 init？
- [ ] stream-json 一条 assistant 事件从哪 yield？
- [ ] headless 谁替代 `useCanUseTool` UI？
- [ ] `ask()` vs REPL QueryEngine 生命周期？
- [ ] 为何 print 跳过 workspace trust？

**关联：** [08 Session](./08-message-and-session-persistence.md) · [14 MCP](./14-mcp-and-external-protocols.md) · [03 入口](./03-cli-entry-and-repl.md) · [22 Remote](./22-remote-and-server-mode.md)
