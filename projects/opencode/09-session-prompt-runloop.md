# 09 · SessionPrompt 主循环

> **核心问题：** `runLoop` 如何调度 LLM 与 tool？何时退出？2100 行 `prompt.ts` 怎么跳读？

基准文件：[`session/prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts)

---

## 1. 入口与并发

| API | 作用 |
|-----|------|
| `prompt()` | 创建 user message → 进入 runLoop |
| `loop()` | 无新 user 时继续（tool 结果已写入后） |
| `command()` / `shell()` | 旁路入口，最终仍可能进 runLoop |

**单 session 单 loop：** [`SessionRunState.ensureRunning`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/run-state.ts#L84-L90) 保证同一 `sessionID` 不会并发两个 runLoop；busy 时新请求会失败或等待。

[`run-state.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/run-state.ts) 要点：

- 每 session 一个 `Runner`（Map 缓存）
- `onIdle` → 发布 status `idle`（Bus → 插件 `event`）
- `onInterrupt` → 取消时恢复到最后 assistant message

`loop()` 调用链（baseline [#L1487](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts#L1487)）：

```typescript
return yield* state.ensureRunning(sessionID, lastAssistant(sessionID), runLoop(sessionID))
```

---

## 2. `prompt.ts` 跳读地图

不要通读全文；按 **Effect.fn 名** 搜索：

| 函数名 | 大致职责 |
|--------|----------|
| `SessionPrompt.createUserMessage` | user 入库、`chat.message` |
| `SessionPrompt.resolveTools` | 合并工具列表 |
| `SessionPrompt.handleSubtask` | subtask 分支（见 [11](./11-tool-registry-and-execution.md)） |
| `SessionPrompt.run` | **runLoop 本体** |
| `SessionPrompt.loop` | ensureRunning 包装 |
| `SessionPrompt.command` | slash command |
| `SessionPrompt.shellImpl` | 交互 shell |

runLoop 起点：baseline [`#L1239`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts#L1239)。

---

## 3. runLoop 骨架（while true）

```mermaid
flowchart TB
  START[while true] --> SCAN[扫描 lastUser / lastFinished / tasks]
  SCAN --> EXIT{应退出?}
  EXIT -->|是| DONE[break]
  EXIT -->|否| POP[task = compaction|subtask]
  POP -->|subtask| ST[handleSubtask → continue]
  POP -->|compaction| CP[compaction.process → continue/break]
  POP -->|无| OV{token overflow?}
  OV -->|是| CC[compaction.create → continue]
  OV -->|否| NORM[新建 assistant message]
  NORM --> PROC[Processor + resolveTools + transform + LLM]
  PROC --> OUT{outcome}
  OUT -->|break| DONE
  OUT -->|continue| START
```

---

## 4. 退出条件（必读）

baseline [`#L1256–1274`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts#L1256-L1274) 逻辑摘要：

```typescript
// 满足以下全部 → break
lastAssistant?.finish &&
  !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls &&   // 非 providerExecuted 的 tool part
  lastUser.id < lastAssistant.id
```

**易错点：**

- 部分 Provider 在仍有 tool-call 时返回 `finish: "stop"`；内核用 `hasToolCalls` **强制继续 loop**，把 tool 结果送回模型。
- `providerExecuted` 的 tool 在 Provider 流内已处理，**不再** re-loop。

正常轮次结束：`processor` 返回 `result === "stop"` → outcome `"break"`（见 baseline ~#L1450 一带）。

---

## 5. 一轮正常路径（逐步）

1. `MessageV2.filterCompactedEffect(sessionID)` — 读 history（含压缩后视图）
2. 从后往前找 `lastUser`、`lastFinished`；收集未完成的 `compaction` / `subtask` part
3. 若队列有 **subtask** → `handleSubtask` → `continue`
4. 若队列有 **compaction** → `compaction.process` → `continue` 或 `break`
5. 若 `lastFinished` 且 **token overflow** → `compaction.create` → `continue`
6. `agents.get` + `agent.steps` 限制步数
7. 新建空 **assistant message** 行
8. `processor.create` → `resolveTools` →（可选 structured output tool）
9. **`experimental.chat.messages.transform`**
10. `MessageV2.toModelMessagesEffect` + system/skills/instructions
11. `handle.process` → 内部调 [`LLM.stream`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/llm.ts) / request 路径
12. 根据 processor 结果：`continue`（还有 tool）或 `break`

Hook 注入点总图见 [18 · 主链路总图](./18-main-chain-atlas.md)。

---

## 6. Processor 分工

[`session/processor.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/processor.ts) 消费 **单轮** LLM stream，返回 `compact | stop | continue`。

| 职责 | 说明 |
|------|------|
| 消费 LLM 流 | text / reasoning / tool-call 等事件 |
| 增量持久化 | `updatePart` / `updatePartDelta` |
| 调度 tool | 与 tools.ts 配合；providerExecuted 例外 |
| `experimental.text.complete` | text-end 时 |

**边界：** prompt = 循环与分支；processor = 单轮 stream 状态机。

**完整事件表与返回值：** [附录 A2 · Processor 深读](./appendix/A2-processor-deep-dive.md)

---

## 7. 调试

| 手段 | 说明 |
|------|------|
| 日志 tag | `SessionPrompt.run`、`step` 字段 |
| `opencode run` | 最小复现 |
| 禁用其它插件 | 隔离 hook 影响 |
| Effect trace | `Effect.fn("SessionPrompt.run")` 名在 trace 中可见 |

---

## 读完后应能回答

- [ ] 为何 `finish: stop` 仍可能继续 loop？
- [ ] `SessionRunState` 与 runLoop 关系？
- [ ] subtask 与 compaction 谁先处理？

→ **下一篇：** [10 · LLM 流与 Provider](./10-llm-stream-and-provider.md)

**深入阅读：**

- 完整故事线：[附录 A1 · 用户消息 Trace](./appendix/A1-full-trace-narrative.md)
- Processor 状态机：[附录 A2](./appendix/A2-processor-deep-dive.md)
