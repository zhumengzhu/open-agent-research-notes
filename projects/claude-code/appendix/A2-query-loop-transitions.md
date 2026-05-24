# A2 · queryLoop Transition 与 Terminal 深读

> **锚点：** `query.ts` · `query/transitions.ts`（类型）  
> **前置：** [06 Agent Loop](./06-query-agent-loop.md)

---

## 1. 两类出口

| 类型 | 语法 | 含义 |
|------|------|------|
| **Terminal** | `return { reason: '...' }` | 结束 `queryLoop`，回到 `query()` 包装 |
| **Continue** | `state = { ... transition: { reason } }; continue` | 下一轮 iteration |

`State.transition` 记录 **上一轮为何 continue**，供测试与 413/collapse 互斥逻辑使用。

---

## 2. Terminal reason 全表

| reason | 典型触发 |
|--------|----------|
| `blocking_limit` | autocompact 关 + token ≥ blocking limit |
| `image_error` | media size 错误且 recovery 失败 |
| `model_error` | API 异常不可重试 |
| `aborted_streaming` | 流式阶段 abort |
| `prompt_too_long` | reactive/collapse 仍 413 |
| `completed` | 正常结束或 API error message 早退 |
| `stop_hook_prevented` | stop hook 禁止继续 |
| `aborted_tools` | 工具阶段 abort |
| `hook_stopped` | hook 停止 turn |
| `max_turns` | 达到 maxTurns |

---

## 3. Continue transition 全表

| reason | 触发 |
|--------|------|
| `collapse_drain_retry` | 413 前 drain staged collapse |
| `reactive_compact_retry` | reactive compact 成功 |
| `max_output_tokens_escalate` | 8k→64k 单次提升 |
| `max_output_tokens_recovery` | 注入 meta recovery user message |
| `stop_hook_blocking` | stop hook 返回 blocking messages |
| `token_budget_continuation` | TOKEN_BUDGET feature |
| `next_turn` | 正常 tool follow-up |

---

## 4. 关键互斥（读代码时易晕）

1. **PTL withhold 后** 不能 fall through stop hooks（death spiral）
2. **collapse_drain_retry** 与 **reactive_compact_retry** 顺序：先 drain 再 reactive
3. **compaction 刚发生** 时 skip blocking limit check（usage stale）
4. **stop_hook_active** 与 preventContinuation 组合决定是否 early return

---

## 5. StreamingToolExecutor 生命周期

| 事件 | 动作 |
|------|------|
| 新 iteration | 新建 executor（或 reuse 配置） |
| model fallback | `discard()` + 新建，防 orphan tool_result |
| tool error (Bash) | sibling abort |

---

## 6. 调试建议

- 搜 `transition:` / `return { reason` in `query.ts`
- 开 debug log：`/tmp` 或 `CLAUDE_CODE_DEBUG`
- Analytics：`tengu_auto_compact_succeeded`、`tengu_model_fallback_triggered`

---

## 7. 自测

- [ ] 区分 Terminal 与 Continue 各 3 例
- [ ] 为何 reactive 失败不跑 stop hooks？
- [ ] fallback 为何要 discard executor？

**关联：** [06](./06-query-agent-loop.md) · [10 413 恢复](./10-compaction-and-context.md#9-l5reactive-compact-与-413-withhold)
