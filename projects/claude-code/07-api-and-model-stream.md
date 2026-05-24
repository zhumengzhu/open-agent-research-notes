# 07 · API 流与 Model 路由

> **锚点文件：** `services/api/claude.ts` · `query/deps.ts` · `utils/model/model.ts`  
> **上游：** [06 query loop](./06-query-agent-loop.md) · **下游：** tool 执行

---

## 1. 在架构中的位置

每轮 `queryLoop` 在 **上下文压缩之后** 调用 API：

```text
compact 段 → deps.callModel(...) → 流式 assistant 块 → tool_use → StreamingToolExecutor
```

`callModel` 不是裸 HTTP，而是 **`queryModelWithStreaming`** 的注入别名（见 `query/deps.ts`）。

---

## 2. 依赖注入：`query/deps.ts`

```typescript
// productionDeps() 核心映射
callModel: queryModelWithStreaming
microcompact: ...
autocompact: ...
```

测试里可替换 `callModel`，无需 mock 整个 `query.ts`。读 loop 时看到 `deps.callModel` 即 **`services/api/claude.ts`**。

---

## 3. `queryModelWithStreaming` 入口

```752:779:/Users/zmz/Github/claude-code/src/services/api/claude.ts
export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  thinkingConfig,
  tools,
  signal,
  options,
}: { ... }): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage,
  void
> {
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}
```

**输入：**

| 参数 | 来源 |
|------|------|
| `messages` | loop 内 `messagesForQuery`（已 compact） |
| `systemPrompt` | `appendSystemContext(systemPrompt, systemContext)` |
| `thinkingConfig` | `ToolUseContext.options.thinkingConfig` |
| `tools` | 当前 turn 可用工具集 |
| `options.model` | `getRuntimeMainLoopModel(...)` — loop 内可随 permission mode 变化 |
| `options.fallbackModel` | QueryEngine / CLI 传入 |
| `options.querySource` | 区分 repl / agent / compact 等 |

**输出（generator yield）：**

- `StreamEvent` — 流式 delta（UI / SDK partial messages）
- `AssistantMessage` — 完整 assistant 消息（含 tool_use blocks）
- `SystemAPIErrorMessage` — API 级错误（可触发 fallback / reactive compact）

---

## 4. Model 选择优先级

主循环模型由 `utils/model/model.ts` 等模块解析，典型优先级：

1. Session 内 `/model` 覆盖（`getMainLoopModelOverride`）
2. 启动时 `--model` / settings
3. 环境变量 `ANTHROPIC_MODEL`
4. 默认模型（订阅档位、allowlist 过滤后）

Loop 内 **`getRuntimeMainLoopModel`** 还会根据 **permission mode**（如 `plan`）和 **上下文 token 量**（如 >200k 切换）动态调整。

**fallback：** `options.fallbackModel` + `onStreamingFallback` — 流式失败时可切非流式或备用模型（见 `executeNonStreamingRequest`）。

---

## 5. Thinking / Effort / Fast mode

在 `query.ts` 调用 `callModel` 时传入：

- `thinkingConfig` — adaptive / disabled 等（QueryEngine 初始化）
- `effortValue` — 来自 `AppState.effortValue`
- `fastMode` — feature gate + settings

这些 **不改变 loop 形状**，只改变 API 请求参数。

---

## 6. 流式与 Tool 的衔接

Loop 内（约 652–900 行）：

1. `for await (const message of deps.callModel(...))` 消费流
2. 见到 `tool_use` block → `needsFollowUp = true`，交给 `StreamingToolExecutor.addTool`
3. 流结束 → `streamingToolExecutor.getRemainingResults()` 收集 tool results
4. 若无 follow-up → 检查 stop hooks、413 恢复等

**注意：** 注释明确 `stop_reason === 'tool_use'` **不可靠**；以流中是否出现 tool_use 为准。

---

## 7. 错误与恢复

| 错误类型 | 典型处理 |
|----------|----------|
| Prompt too long (413) | withhold → collapse drain → reactive compact |
| 流式失败 | `onStreamingFallback` → 非流式 retry |
| 529 / 过载 | `withRetry` 指数退避 |
| User abort | `AbortSignal` 贯穿 callModel 与 tool |

Compact 子系统也会 **直接调用** `queryModelWithStreaming`（生成摘要），与主 loop 共用 API 层。

---

## 8. 相关 services/api 文件

| 文件 | 职责 |
|------|------|
| `claude.ts` | 主入口 `queryModelWithStreaming`、`queryModel`、`addCacheBreakpoints`（prompt cache + cache_edits 注入） |
| `client.ts` | Anthropic client 构造 |
| `withRetry.ts` | 重试与 fallback 包装 |
| `errors.ts` / `errorUtils.ts` | 错误分类 |
| `logging.ts` / `usage.ts` | token 用量（含 `cache_deleted_input_tokens`） |
| `promptCacheBreakDetection.ts` | prefix cache miss 检测；`notifyCacheDeletion` 排除 cached MC 预期掉量 |
| `bootstrap.ts` | API bootstrap |
| `sessionIngress.ts` | 远程 session 入口 |

首读 **claude.ts** 即可；其余按需跳转。

> **关于 prompt cache / cache_edits：** 见 [10 §6.4–§6.6](./10-compaction-and-context.md#64-cached-microcompactcached_microcompact) 与 [10 §10](./10-compaction-and-context.md#10-prompt-cache-体系)。

---

## 9. Message 归一化与 API 编组

### 9.1 `normalizeMessagesForAPI`

`services/api/messageNormalization.ts` — loop 内 `messagesForQuery` → API wire format：

- 剔除 progress 等非 API 类型
- **tool_use ↔ tool_result 配对** — 修复 orphan blocks
- Thinking blocks 格式校验
- Tool search：处理 `tool_reference` 块（需模型支持 [30]）

`claude.ts` 在 normalize **之后** 还可能二次过滤 deferred tools（精确 `isToolSearchEnabled` vs normalize 内的 optimistic check）。

### 9.2 `addCacheBreakpoints`

`claude.ts` `addCacheBreakpoints`：

- 在 system + messages 前缀插入 `cache_control: { type: 'ephemeral' }`
- 注入 **cache_edits**（cached microcompact 删 KV 内 tool result）[10]
- 与 `notifyCacheDeletion` 联动 usage 统计 [24]

**Invariant：** 同一 turn 多轮 loop 应 **复用相同 prefix**（system + tools + messages 前缀不变），否则 cache_creation spike。

### 9.3 Betas 合并

`getMergedBetas()` — task_budget、prompt cache、extended context 等 beta header 合并进请求（[27] provider 变体可能影响可用 betas）。

---

## 10. Token budget 与 task_budget

Claude Code 有 **两套独立的「预算续跑」机制**，都可能导致 loop **continue**（见 [28 §3.3](./28-agent-loop-continuation-and-human-gates.md#33-token-budget-自动续轮feature-token_budget)）。

### 10.1 Token budget（feature `TOKEN_BUDGET`）

**客户端侧**：`query/tokenBudget.ts` `checkTokenBudget`

| 条件 | 行为 |
|------|------|
| 非 subagent + budget > 0 + turn 输出 < 90% budget | `action: 'continue'` → 注入 meta nudge message |
| 连续 3+ 次 continuation 且增量 < 500 tokens | diminishing returns → stop |
| subagent 或有 agentId | 不续（`action: 'stop'`） |

注入内容来自 `getBudgetContinuationMessage(pct, turnTokens, budget)` — 告诉模型「你还有预算，请继续完成任务」。

**与 API 无关**：纯客户端在 stop hooks 之后、return completed 之前判定。

### 10.2 task_budget（API beta）

**服务端侧**：`QueryParams.taskBudget: { total: number }` → `configureTaskBudgetParams` → `output_config.task_budget`

```193:197:/Users/zmz/Github/claude-code/src/query.ts
  // API task_budget (output_config.task_budget, beta task-budgets-2026-03-13).
  // Distinct from the tokenBudget +500k auto-continue feature.
```

| 维度 | task_budget | TOKEN_BUDGET feature |
|------|-------------|----------------------|
| 配置方 | QueryEngine / SDK 传入 | feature gate + bootstrap state |
| 执行方 | **Anthropic API** 计数 | **query.ts** 客户端 nudge |
| compact 后 | `taskBudgetRemaining` 跨 compact 累积（`query.ts:282-291` 注释） | 独立 tracker |

compact 后 API 只见摘要，server 会 under-count spend；客户端用 `taskBudgetRemaining` 告诉 server 被摘要掉的 spend（见 `query.ts` loop-local 变量注释）。

### 10.3 maxTurns / maxBudgetUsd（QueryEngine 层）

| 限制 | 检查位置 | 效果 |
|------|----------|------|
| `maxTurns` | query loop 每 iteration 末 | Terminal `max_turns` |
| `maxBudgetUsd` | QueryEngine submitMessage 末 | yield error result |

与 token budget continuation **独立**——maxTurns 硬截断 iteration 次数；USD 预算截断整个 turn。

---

## 11. 自测

- [ ] `deps.callModel` 在 production 指向哪个函数？
- [ ] model 在 loop 的哪一步确定？会不会 mid-turn 变化？
- [ ] 流式 tool_use 与 batch `runTools` 如何选型？
- [ ] `normalizeMessagesForAPI` 与 tool search 的两阶段检查区别？
- [ ] token budget continuation 与 task_budget 区别？
- [ ] compact 后 taskBudgetRemaining 为何需要？

**关联：** [13 System prompt](./13-system-prompt-and-context.md) · [28 Loop 门控](./28-agent-loop-continuation-and-human-gates.md) · [26 总图](./26-main-chain-atlas.md) · [flow/api-stream](./flow/README.md#api-流)
