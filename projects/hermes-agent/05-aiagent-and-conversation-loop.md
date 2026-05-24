# 05 · AIAgent 与 Conversation Loop

> **锚点：** `run_agent.py`（façade）· `agent/conversation_loop.py`（**~4200 行，loop 本体**）· `agent/agent_runtime_helpers.invoke_tool` · `model_tools.py`  
> **深读配套：** [06 Tools](./06-tools-registry-and-model-tools.md) · [13 Prompt](./13-prompt-assembly-and-cache.md) · [14 压缩](./14-context-compression.md) · [08 Session/Memory](./08-session-and-memory.md)

---

## 1. 接口层 vs 实现层

| 层 | 文件 | 职责 |
|----|------|------|
| **Façade** | `run_agent.AIAgent` | 构造、字段、`run_conversation` **转发**、`_compress_context` 转发 |
| **Loop** | `conversation_loop.run_conversation` | 整 turn：前序、while、post-turn、return dict |
| **Tool** | `invoke_tool` + `handle_function_call` | 见 [06 §6](./06-tools-registry-and-model-tools.md#6-两条执行路径) |

```4053:4064:/Users/zmz/Github/hermes-agent/run_agent.py
def run_conversation(self, ...):
    """Forwarder — see ``agent.conversation_loop.run_conversation``."""
    from agent.conversation_loop import run_conversation
    return run_conversation(self, ...)
```

构造参数 ~60 个：按 **platform**（cli/gateway/cron）、**memory**（skip_memory）、**toolsets** 分组 skim 即可。

---

## 2. Turn 前序（进 while 之前）

`run_conversation` 232 行起，顺序固定（省略纯 logging）：

| 步骤 | 行级锚点 | 作用 |
|------|----------|------|
| DB session | `_ensure_db_session` | 保证 SessionDB 行存在 |
| auxiliary runtime | `set_runtime_main(provider, model)` | vision 等 side tool 见 **当前** override |
| 日志 context | `set_session_context(session_id)` | `hermes logs --session` |
| skill provenance | `set_current_write_origin` | 区分 foreground vs review fork |
| fallback 恢复 | `_restore_primary_runtime` | 上 turn fallback 后本 turn 试主模型 |
| 输入 sanitize | `_sanitize_surrogates` | 剪贴板 lone surrogate 防 JSON 崩 |
| **Turn 重置** | 319–338 | retry 计数、guardrails、`IterationBudget` **重置** |
| hydrate | 378–403 | todo store；Gateway **nudge 计数**（#22357） |
| append user | 441–445 | `messages` + `persist_user_message` 索引 |
| system | 462–463 | `_restore_or_build_system_prompt` 四态 [13](./13-prompt-assembly-and-cache.md) |
| **preflight 压缩** | 467–533 | 含 **tools schema** token；最多 3 pass [14](./14-context-compression.md) |
| pre_llm_call | 535–569 | 插件 context → **user message**，不进 system |
| on_turn_start | 607–615 | memory provider 回合计数 |
| **prefetch 一次** | 617–628 | `_ext_prefetch_cache` 整 turn 复用 |
| codex bypass | ~635–642 | `api_mode=codex_app_server` → 不进 while |

**注意：** `_turns_since_memory` / `_iters_since_skill` **不在** turn 开头 reset（359–361）— CLI 多 turn 同一 Agent 时 nudge 才累积。

### 2.1 Prefetch 缓存

```617:628:/Users/zmz/Github/hermes-agent/agent/conversation_loop.py
# prefetch once before the tool loop.
# Reuse the cached result on every iteration ...
# Use original_user_message — user_message may contain injected skill content
_ext_prefetch_cache = agent._memory_manager.prefetch_all(_query) or ""
```

10 次 tool iteration 若每轮 prefetch → 10× provider 延迟与费用。

### 2.2 Memory nudge（turn 前判定）

433–439 行：若 `_memory_nudge_interval` 命中 → `_should_review_memory=True`，**实际 spawn 在 turn 末**（4192+）。

---

## 3. 核心 while 循环

```644:669:/Users/zmz/Github/hermes-agent/agent/conversation_loop.py
while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
    if agent._interrupt_requested:
        break
    elif not agent.iteration_budget.consume():
        break
```

| 退出 | 机制 |
|------|------|
| 正常 | 无 `tool_calls` → finalize |
| `max_iterations` | 硬 cap（默认 90） |
| `iteration_budget` | 每 iteration `consume()` [02](./02-config-iteration-and-model-routing.md) |
| `_budget_grace_call` | 耗尽后再给 **一轮** API |
| interrupt | `_interrupt_requested` |
| guardrail halt | `_tool_guardrail_halt_decision` |

与 CC：以 **tool_calls 是否存在** 决定是否继续，非单一 `stop_reason`。

### 3.1 Interrupt 传播

```text
Ctrl+C / Gateway cancel / TUI interrupt
  → _interrupt_requested
  → while break
  → tool_executor preflight：synthetic cancelled results
  → terminal is_interrupted() [17](./17-terminal-backends.md)
  → Gateway pending 单槽 [10 §6](./10-gateway-platforms-and-sessions.md#6-interrupt-与-pending-消息)
```

Runtime fallback（主 API 失败）≠ auxiliary 链 [15 §6](./15-provider-and-transport.md#6-auxiliary-侧任务路由)。

### 3.2 `/steer` drain（705–753）

模型 **思考中** 用户发的 steer 必须在 **下一次 API 前** 注入：

- 优先 append 到最后一条 **tool** message 的 content
- 尚无 tool 输出 → 保持 `_pending_steer` pending
- turn 结束若仍 leftover → `result["pending_steer"]` 交 caller（4159–4164）

**为何不能随意 append user message：** 破坏 user/assistant 交替与 cache 边界。

### 3.3 Iteration 内其它步骤（skim 顺序）

1. `step_callback` — Gateway 上报上一轮 tool 摘要  
2. `_iters_since_skill += 1`；调 `skill_manage` 时 reset  
3. steer drain（上）  
4. loop 内 compression（若 token 超阈）  
5. `apply_anthropic_cache_control` + API stream  
6. `_execute_tool_calls` → 并行或串行 [06](./06-tools-registry-and-model-tools.md)  
7. guardrail `after_call` 可能 halt turn  

---

## 4. 工具调用（loop 视角）

```text
assistant.tool_calls
  → run_agent._execute_tool_calls
       → _should_parallelize_tool_batch ?
       → execute_tool_calls_concurrent | sequential
       → invoke_tool (agent-level: todo/memory/session_search/delegate/...)
       → handle_function_call (registry + hooks)
  → append role:tool messages（顺序与 tool_call_id 对齐）
```

**execute_code** iteration 可 `iteration_budget.refund()` — PTC 不占预算 [02](./02-config-iteration-and-model-routing.md)。

---

## 5. 压缩（loop 内 + preflight）

- **Preflight：** 467–533 — 换小 context 模型时 proactive  
- **Loop 内：** iteration 开头 token 估算 → `_compress_context`  
- **唯一** routine 允许改写历史 messages 的路径 [14](./14-context-compression.md)  
- 成功后：`conversation_history = None`、retry 计数清零、可能 **新 session_id**

Anthropic：`prompt_caching.apply_anthropic_cache_control` 在 API 组包时 deepcopy 注入 [13](./13-prompt-assembly-and-cache.md)。

---

## 6. Post-turn 与 return dict

Loop 结束后（4128–4227 行）：

| 阶段 | 行为 |
|------|------|
| `post_llm_call` hook | 插件观测完整 turn |
| `result` dict | `final_response`、`messages`、`api_calls`、`turn_exit_reason`、`interrupted`、token/cost 字段… |
| `_sync_external_memory_for_turn` | provider sync + queue 下 turn prefetch |
| **background_review** | `_should_review_memory \|\| _should_review_skills` → daemon 线程 [18](./18-multi-agent-panorama.md) |
| **不** shutdown memory | 多 turn session 每消息一次 `run_conversation` — 4204–4209 注释 |
| `on_session_end` hook | 插件清理 |

**Skill nudge：** 在 turn **末** 检查 `_iters_since_skill >= interval`（4177–4183），非 turn 开头。

**background_review 条件：** 有 `final_response`、未 interrupt、memory 或 skill nudge 命中。

---

## 7. 特殊 `api_mode`

| 模式 | 行为 |
|------|------|
| `chat_completions` | 默认 |
| `anthropic_messages` | 原生 Messages API |
| `codex_responses` | Codex 适配 |
| `codex_app_server` | **整 turn bypass** 主 while |

---

## 8. `delegate_task`（loop 内阻塞）

`invoke_tool` → `_dispatch_delegate_task` — 父 **while 阻塞** 至子 `run_conversation` 结束。子 agent 独立 budget（默认 50，与父 90 **独立池**）[11](./11-delegation-cron-and-kanban.md)。

---

## 9. 源码带读顺序（建议 4–6h）

| # | 区域 | 行号（约） |
|---|------|------------|
| 1 | 函数签名 + turn 重置 | 232–338 |
| 2 | hydrate + system + preflight | 378–533 |
| 3 | prefetch + codex branch | 607–642 |
| 4 | while + steer + API | 644–1200+ |
| 5 | tool 分支 + return | 3800–4227 |
| 6 | `_restore_or_build_system_prompt` | 130–227 |

---

## 10. 自测

- [ ] loop 实现在哪个文件？
- [ ] prefetch 为何用 `original_user_message`？
- [ ] memory nudge 为何 turn 末 spawn review？
- [ ] steer leftover 如何不丢？
- [ ] preflight 为何算 `tools=` token？
- [ ] 为何 post-turn 不 `shutdown_all` memory？
- [ ] grace call 与 max_iterations 区别？

**关联：** [01 架构](./01-architecture-overview.md) · [03 入口](./03-cli-gateway-and-entry.md) · [26 总图](./26-main-chain-atlas.md)
