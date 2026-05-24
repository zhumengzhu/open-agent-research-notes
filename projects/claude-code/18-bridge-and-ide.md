# 18 · Bridge 与 IDE 集成

> **锚点：** `bridge/`（~33 文件）· `commands/bridge/` · `bridge/bridgeEnabled.ts` · README Architecture §4

---

## 1. 定位

**Bridge** 是 Claude Code CLI 与 IDE 扩展（VS Code、JetBrains 等）及 **Remote Control (CCR)** 的 **双向集成层**：

| 方向 | 典型 payload |
|------|----------------|
| IDE/Web → CLI | selection、open files、diagnostics、file_uuid 上传 |
| CLI → IDE | permission 请求、打开文件、diff、session 状态 |

三种连接形态：

| 形态 | 场景 |
|------|------|
| **REPL 内嵌** | `--ide`，本地 IDE 插件 ↔ 当前 CLI 进程 |
| **Bridge worker** | `bridgeMain.ts` 长期跑，control plane spawn 子 session |
| **CCR Web composer** | OAuth + file upload，经 bridge API 进 CLI |

与 [22 Remote](./22-remote-and-server-mode.md)：**Bridge 是协议与 transport**；Remote 是 worker 运行时语义（timeout、memory、settings sync）。

---

## 2. 启用门控

`bridge/bridgeEnabled.ts`：

```text
feature('BRIDGE_MODE')
  AND isClaudeAISubscriber()     // 需 claude.ai OAuth，非 Bedrock/Vertex/API-key-only
  AND GrowthBook tengu_ccr_bridge
```

- `isBridgeEnabled()` — UI 可见性，cache 可能 stale
- `isBridgeEnabledBlocking()` —  entitlement 门，最多 ~5s 等 GrowthBook
- `getBridgeDisabledReason()` — 用户可读诊断

外部 snapshot 可能 **无 BRIDGE_MODE**（整段 DCE）。

---

## 3. 模块地图

```text
bridge/
├── bridgeMain.ts              # worker 主循环、poll、spawn、shutdown
├── bridgeApi.ts               # HTTP client、BridgeFatalError
├── bridgeMessaging.ts         # 消息 envelope 类型
├── replBridge.ts              # REPL 会话桥
├── initReplBridge.ts          # REPL 启动 hook
├── replBridgeTransport.ts     # transport 抽象
├── sessionRunner.ts           # createSessionSpawner
├── bridgePermissionCallbacks.ts
├── inboundMessages.ts         # IDE → CLI 解析
├── inboundAttachments.ts      # selection / file_uuid → @path
├── jwtUtils.ts                # token 刷新调度
├── trustedDevice.ts
├── workSecret.ts              # CCR v2 worker 注册
├── bridgeConfig.ts            # base URL、access token
├── capacityWake.ts            # 容量唤醒
├── sessionIdCompat.ts         # infra id ↔ compat id
├── flushGate.ts               # 输出 flush 协调
└── pollConfig.ts              # poll 间隔默认值
```

---

## 4. 生命周期

### 4.1 REPL 内嵌 Bridge

```text
launchRepl → initReplBridge
  → 检测 IDE / bridge config
  → replBridgeTransport 连接
  → inbound 消息 → inboundAttachments → processUserInput [12]
  → permission → bridgePermissionCallbacks → IDE UI
```

### 4.2 Bridge Worker

`bridgeMain.ts`：

1. `createBridgeApiClient` + `createTokenRefreshScheduler`
2. Poll work queue（`getPollIntervalConfig`）
3. `createSessionSpawner` 启动子 Claude Code（`SpawnMode`：local/container）
4. `BackoffConfig`：conn 2s→120s cap，10min give-up；general 500ms→30s
5. Session 完成 → `SessionDoneStatus` 上报
6. Worktree cleanup：`createAgentWorktree` / `removeAgentWorktree` [23]
7. Shutdown：SIGTERM → grace 30s → SIGKILL

CCR v2：`buildCCRv2SdkUrl`、`registerWorker`、`decodeWorkSecret` [22]。

---

## 5. 消息与附件

### 5.1 inboundAttachments

Web composer 上传 `file_uuid` → bridge `GET /api/oauth/files/{uuid}/content` → 写入 `~/.claude/uploads/{sessionId}/` →  prepend `@path` 给模型 Read。

**Best-effort：** 失败则跳过该附件，message 仍送达（无 @path）。

### 5.2 Selection / diagnostics

IDE 宿主 API → `inboundMessages` → attachment 块进 user message（不进 static system [13]）。

---

## 6. Permission 跨进程

```mermaid
sequenceDiagram
  participant Loop as query loop
  participant CUT as useCanUseTool
  participant BPC as bridgePermissionCallbacks
  participant IDE as IDE / Web UI

  Loop->>CUT: canUseTool(tool, input)
  CUT->>BPC: 需用户决策
  BPC->>IDE: permission_request
  IDE->>BPC: allow/deny
  BPC->>CUT: 结果
  CUT->>Loop: 继续或 deny
```

- **Fail closed：** 超时无响应 → deny [11]
- Bridge 可 `set_permission_mode` 改 AppState
- Headless remote：同类语义经 `remotePermissionBridge` [22]

---

## 7. Session activity（compact keep-alive）

长 compact / 长 tool 执行无 stdout 时：

- `sendSessionActivitySignal` → control plane 知 worker 仍 alive
- 防 CCR idle kill [22]

---

## 8. 与 LSP / MCP 分工

| | Bridge | LSP [16] | MCP [14] |
|---|--------|----------|----------|
| 数据源 | IDE 宿主 | language server | 外部 server |
| 典型数据 | 选区、IDE 诊断、上传文件 | 符号级导航 | 任意 tool |
| 进程 | IDE + 可选 worker | CC spawn LSP | 用户配置 |

三者可同时启用；模型需分辨 **IDE 诊断 vs LSP 诊断** 来源。

---

## 9. 安全

| 机制 | 作用 |
|------|------|
| `jwtUtils` | access token 生命周期 |
| `trustedDevice` | 设备信任 |
| `validateBridgeId` | bridge id 格式 |
| `workSecret` | worker 注册密钥（勿 log） |
| OAuth subscriber gate | 无 claude.ai token 不可用 CCR bridge |

---

## 10. 调试

- `bridgeDebug.ts` / `debugUtils.ts` — axios 错误
- `bridgeStatusUtil.ts` — 连接状态、`formatDuration`
- Analytics：`tengu_bridge_*` 类事件 [24]
- Debug filter：`-d bridge`（`-d` 分类过滤 [03]）

---

## 11. 源码带读

1. `bridge/bridgeEnabled.ts` — 何时可用
2. `bridge/initReplBridge.ts` — REPL 挂载
3. `bridge/inboundAttachments.ts` — file_uuid 流程
4. `bridge/bridgePermissionCallbacks.ts` — permission 往返
5. `bridge/bridgeMain.ts` — worker poll/spawn（前 300 行 + shutdown）

---

## 12. 自测

- [ ] Bridge 三种形态区别？
- [ ] 为何需 claude.ai subscriber？
- [ ] permission 跨进程超时行为？
- [ ] file_uuid 如何变成模型可读 @path？
- [ ] compact 时 session activity 作用？
- [ ] Bridge vs remote/ 分工？

**关联：** [03 CLI](./03-cli-entry-and-repl.md) · [11 Permission](./11-permission-and-hooks.md) · [16 LSP](./16-lsp-and-code-intelligence.md) · [22 Remote](./22-remote-and-server-mode.md)
