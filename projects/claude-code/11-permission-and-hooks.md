# 11 · Permission 与 Hooks

> **锚点：** `hooks/useCanUseTool.tsx` · `hooks/toolPermission/PermissionContext.ts` · `utils/hooks/` · `utils/permissions/yoloClassifier.ts` · `utils/plugins/loadPluginHooks.ts`  
> **衔接：** [09 Tool 执行](./09-tools-system.md) · [28 Loop 续跑与人机门控](./28-agent-loop-continuation-and-human-gates.md)

---

## 1. 架构位置

Permission 与 Hooks 是 **L3 人机门控** 的核心（见 [28 §6](./28-agent-loop-continuation-and-human-gates.md#6-l3loop-内部的人机门控阻塞但不结束-turn)）：

```text
runToolUse
  → canUseTool (permission mode + rules + classifier + hooks)
  → executePreToolUseHooks
  → tool.call
  → executePostToolUseHooks / executePermissionDeniedHooks

query loop 无 tool_use 时
  → handleStopHooks → executeStopHooks
```

**Hooks ≠ Plugins**：Hooks 是 **事件回调机制**；Plugin 是 **能力打包**，可 **附带** hooks 配置。详见 [15 §4](./15-skills-and-plugins.md#4-hooks-与-plugin-的关系)。

---

## 2. Permission 模型

### 2.1 `canUseTool` 返回值

| behavior | 含义 | Loop 行为 |
|----------|------|-----------|
| `allow` | 执行 tool | 正常 continue |
| `deny` | 拒绝 | 合成 error tool_result，模型可见 |
| `ask` | 需用户确认 | **阻塞** UI / elicitation |

QueryEngine 用 `wrappedCanUseTool` 记录 denial → SDK `permission_denials`（[05 §4.1](./05-query-engine.md#41-阶段-1setup)）。

### 2.2 Permission mode 五态

SDK schema（`entrypoints/sdk/coreSchemas.ts`）：

| mode | 行为 | 典型场景 |
|------|------|----------|
| `default` | 标准：规则 + 弹窗 | 日常开发 |
| `acceptEdits` | **自动 allow** 文件编辑类 tool | 少打断写文件 |
| `dontAsk` | **不弹窗**；未预批则 deny | CI / 保守自动化 |
| `bypassPermissions` | **跳过全部** permission 检查 | 需 `--dangerously-skip-permissions` |
| `plan` | 分析为主，限制非 md 编辑 | 先计划后执行（[17](./17-plan-mode-and-code-editing.md)） |

CLI / settings：

- `--permission-mode bypassPermissions` + `--dangerously-skip-permissions`
- `interactiveHelpers.tsx`：bypass 被 settings 禁用时拒绝切换（`isBypassPermissionsModeDisabled`）

### 2.3 Rules 两层

| 层 | 位置 | 效果 |
|----|------|------|
| **Registry** | `tools.ts` `filterToolsByDenyRules` | 被 deny 的 tool **不出现在 schema** |
| **Execute** | `canUseTool` + Bash 命令解析 | input 级二次检查（如 `Bash(git:*)`） |

来源：CLI `--allowedTools` / `--disallowedTools`、walked settings、`alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules`。

---

## 3. `canUseTool` 决策链（REPL 路径）

`PermissionContext.ts` `createPermissionContext` 概念顺序：

```text
1. tool.checkPermissions (tool 级，默认 allow defer 到 general system)
2. alwaysAllow / alwaysDeny / alwaysAsk rules  pattern match
3. permission mode 短路
     · bypassPermissions → allow
     · dontAsk → deny if not pre-approved
     · acceptEdits → allow edit tools
     · plan → 限制非 plan 路径（见 17）
4. auto mode classifier (TRANSCRIPT_CLASSIFIER feature)
     · yoloClassifier side query → allow / soft_deny
5. Bash speculative classifier (bashPermissions.ts)
6. executePermissionRequestHooks → hook 可 override behavior
7. ask → PermissionQueue → UI 对话框
8. user allow / deny / always allow
```

**Approval 来源**（telemetry）：`user` | `hook` | `classifier`（含 `auto-mode`、`bash_allow`）。

---

## 4. Auto mode 与「yolo」语义

官方 **没有** 名为 `yolo` 的用户开关；源码内部用 `yoloClassifier.ts`：

| 概念 | 是什么 |
|------|--------|
| **Auto mode** | 用 **side query 小模型** 读 transcript + 规则，自动 allow/deny tool |
| **Settings** | `autoMode.allow` / `soft_deny` / `environment` 三组字符串规则 |
| **CLI** | `claude auto-mode defaults` · `critique` · `config` |
| **Feature** | `TRANSCRIPT_CLASSIFIER` |
| **与 bypass 区别** | classifier **仍可能 deny**；bypass 是 **零检查** |

社区口语「yolo」通常指：

1. `permissionMode: bypassPermissions`（最激进）
2. auto mode 规则开满 + `acceptEdits`
3. `--allowedTools` 白名单很大

`useCanUseTool.tsx` 在 classifier allow 时调 `setYoloClassifierApproval`；deny 时 `recordAutoModeDenial`。

---

## 5. REPL vs Headless

| 维度 | REPL | Headless / SDK |
|------|------|----------------|
| UI 许可 | `useCanUseTool.tsx` + React 队列 | `print.ts` `createCanUseToolWithPermissionPrompt` |
| MCP 代问 | — | MCP elicitation tool |
| Classifier | 完整 auto mode | 部分路径 no-op（remote session 注释） |
| `shouldAvoidPermissionPrompts` | false | background agent 可为 true → 自动 deny |

---

## 6. Hooks 体系

### 6.1 配置来源（两路合并）

```text
settings.json hooks.{Event}[]     ─┐
                                   ├→ bootstrap/state hook registry
plugin manifest hooksConfig       ─┘   (loadPluginHooks.ts)
```

Plugin hooks 转成 `PluginHookMatcher`（带 `pluginRoot` / `pluginName`），与用户 hooks **同一执行引擎**。

### 6.2 Hook 类型（command / prompt / HTTP）

`schemas/hooks.ts`：

- **`command`** — shell 命令；可设 `if` 条件（permission rule 语法过滤）
- **`prompt`** — LLM 评估（execPromptHook）
- **HTTP** — 远程 webhook（execHttpHook）

### 6.3 Hook 事件全集（节选）

来自 `loadPluginHooks.ts` 注册的 `HookEvent`：

| 类别 | 事件 |
|------|------|
| Tool | `PreToolUse` · `PostToolUse` · `PostToolUseFailure` · `PermissionDenied` · `PermissionRequest` |
| Turn | `Stop` · `StopFailure` · `UserPromptSubmit` |
| Session | `SessionStart` · `SessionEnd` · `PreCompact` · `PostCompact` |
| Agent | `SubagentStart` · `SubagentStop` · `TeammateIdle` |
| Task | `TaskCreated` · `TaskCompleted` |
| 其它 | `Notification` · `Elicitation` · `ConfigChange` · `WorktreeCreate` · `FileChanged` · … |

### 6.4 `HookResult` 能做什么

`types/hooks.ts`：

| 字段 | 效果 |
|------|------|
| `blockingError` | 注入错误 message → **促使 loop continue**（stop_hook_blocking） |
| `preventContinuation` | **终止 loop** |
| `permissionBehavior` | `allow` / `deny` / `ask` / `passthrough` 覆盖 permission |
| `updatedInput` | 改 tool 输入 |
| `additionalContext` | 追加 context |
| `initialUserMessage` | 相当于替用户发 message |
| `retry` | 请求重试 tool |

Stop hook 聚合为 `AggregatedHookResult` → `query.ts` 读 `blockingErrors` / `preventContinuation`。

### 6.5 典型插入点

```text
Compact:  executePreCompactHooks → compactConversation → processSessionStartHooks('compact')
Tool:     executePreToolUseHooks → call → executePostToolUseHooks
Stop:     handleStopHooks → executeStopHooks (+ extractMemories / promptSuggestion 等 side work)
Submit:   UserPromptSubmit hooks (processUserInput 路径)
```

---

## 7. Stop hooks 与 loop 续跑

`query/stopHooks.ts`：

- 主 thread / SDK 结束时 **`saveCacheSafeParams`**（[10 §10.2](./10-compaction-and-context.md#102-cachesafeparamsfork-与-compact-的共享前缀契约)）
- `preventContinuation: true` → `return { reason: 'stop_hook_prevented' }`
- `blockingErrors` → 注入 messages → `transition: stop_hook_blocking` → **continue**

**PTL / API 错误路径故意跳过 stop hooks**（防 death spiral，[28 §4](./28-agent-loop-continuation-and-human-gates.md#4-l1loop-何时-return结束本轮-agent-运行)）。

PostToolUse 还可 yield `hook_stopped_continuation` attachment → `shouldPreventContinuation`（[28 §4.1](./28-agent-loop-continuation-and-human-gates.md#41-tool-执行层的-preventcontinuation)）。

---

## 8. 基于 Hooks / Permission 的扩展能力

| 需求 | 机制 |
|------|------|
| 危险 Bash 拦截 | PreToolUse deny + `if: "Bash(rm *)"` |
| 改完跑 lint | PostToolUse command hook |
| 测试不过不许停 | Stop hook → blockingError |
| 自动批权限 | PermissionRequest hook → allow |
| 审计日志 | PostToolUse HTTP hook |
| 自定义 stop 条件 | Stop → preventContinuation |
| 注入团队规范 | SessionStart / UserPromptSubmit |
| 无 UI 自动化 | bypassPermissions 或 auto mode rules + dontAsk |

Plugin 打包分发：见 [15](./15-skills-and-plugins.md)。

---

## 9. Bash / Edit 特例

- `bashPermissions.ts` — 命令级 allowlist + **speculative classifier**（与 auto mode 并行）
- `permissionLogging.ts` — 代码编辑 telemetry
- Plan mode：非 md 编辑受限；`EnterPlanMode` 改 `AppState.toolPermissionContext.mode`

---

## 10. 与 OpenCode / OmO 对照（后期）

| | Claude Code | OmO (OpenCode plugin) |
|--|-------------|------------------------|
| Hook 形态 | shell / HTTP / prompt command | TypeScript lifecycle handlers |
| 配置 | settings + plugin manifest | plugin JSONC |
| Permission | canUseTool + mode + classifier | category / agent permission |

首读 Claude Code 自身即可；对照非必须。

---

## 11. 自测

- [ ] 五种 permission mode 各一句话？
- [ ] bypassPermissions 与 auto mode classifier 区别？
- [ ] Hook blockingError 与 preventContinuation 对 loop 的影响？
- [ ] Plugin hooks 与用户 settings hooks 如何合并？
- [ ] REPL vs headless 谁弹 permission 框？
- [ ] PTL 后为何不跑 stop hooks？
- [ ] 「yolo」在源码里对应什么文件名？

**关联：** [28 Loop 门控](./28-agent-loop-continuation-and-human-gates.md) · [15 Plugins](./15-skills-and-plugins.md) · [17 Plan mode](./17-plan-mode-and-code-editing.md) · [flow/](../flow/README.md)
