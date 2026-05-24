# 02 · 配置、Iteration Budget 与模型路由

> **锚点：** `hermes_cli/config.py` · `agent/iteration_budget.py` · `agent/model_metadata.py` · `hermes_cli/runtime_provider.py`

配置决定 **哪些 tools、哪条模型链、压缩阈值**；`IterationBudget` 决定 **单 Agent 能跑几轮 API**。本章强调与 loop 的 **硬耦合点**。

---

## 1. 配置层级

```text
代码 defaults
  ← ~/.hermes/config.yaml（或 Profile 下）
  ← 项目 .hermes/config.yaml（若存在）
  ← 环境变量（部分键；MCP env allowlist [16](./16-plugins-mcp-and-hooks.md)）
```

- **Profile：** `-p name` → `HERMES_HOME` 切换整树 [21](./21-profiles-and-credential-pool.md)
- **校验：** `hermes doctor`；JSONC 注释支持

**三套 loader** 消费同一 YAML 不同子集 — 见 [03 §5.3](./03-cli-gateway-and-entry.md#53-三套-config-loader陷阱)。

---

## 2. 与 loop 强相关的键

| 区域 | 影响 |
|------|------|
| `model.*` | provider、default model、api_mode [15](./15-provider-and-transport.md) |
| `tools.*` | platform enabled/disabled toolsets [07](./07-toolsets-and-platform-bundles.md) |
| `delegation.*` | 子 agent 并发/深度/**max_iterations** [11](./11-delegation-cron-and-kanban.md) |
| `compression.*` | threshold、abort_on_summary_failure [14](./14-context-compression.md) |
| `auxiliary.*` | 压缩/judge/vision 侧模型 [15 §6](./15-provider-and-transport.md#6-auxiliary-侧任务路由) |
| `tool_loop_guardrails.*` | [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails) |
| `memory.*` | provider 选择、nudge interval [08](./08-session-and-memory.md) |
| `goals.*` | Ralph loop [19](./19-goals-and-ralph-loop.md) |

**Mid-turn 政策：** 改 tools/skills/memory provider 默认 deferred；`/model` 与 compression 是例外 [13 §8](./13-prompt-assembly-and-cache.md#8-mid-turn-禁止-vs-允许)。

---

## 3. IterationBudget（细读）

**文件：** `agent/iteration_budget.py`

```17:29:/Users/zmz/Github/hermes-agent/agent/iteration_budget.py
"""Each agent (parent or subagent) gets its own IterationBudget.
Parent capped at max_iterations (default 90).
Each subagent gets an independent budget capped at delegation.max_iterations (default 50)
— total iterations across parent + subagents can exceed the parent's cap."""
```

| API | 行为 |
|-----|------|
| `consume()` | while 每 iteration 前；False → break（或 grace） |
| `refund()` | 如 execute_code 不占预算 |
| `remaining` | 与 `max_iterations` 并列条件 [05 §3](./05-aiagent-and-conversation-loop.md#3-核心-while-循环) |

**Turn 边界：** `run_conversation` 开头 `agent.iteration_budget = IterationBudget(agent.max_iterations)` — **每 user message 重置**（319–362 行）。

**常见误解：** 父 90 + 子 50 可 **合计 >90** — 子有 **独立** budget 对象，非从父池扣减。

---

## 4. `max_iterations` vs budget vs grace

| 概念 | 作用 |
|------|------|
| `max_iterations` | while 条件：`api_call_count < max_iterations` |
| `iteration_budget` | 细粒度 consume（与 api_call_count 并行检查） |
| `_budget_grace_call` | 耗尽后 **再允许一轮** API，然后强制退出 |

子 agent 构造时用 `delegation.max_iterations` 建 **新** IterationBudget。

---

## 5. 模型 metadata 与路由

**`agent/model_metadata.py`：**

| 函数 | 用途 |
|------|------|
| `get_model_context_length` | 压缩 threshold = ctx × percent |
| `estimate_request_tokens_rough` | preflight 含 messages + system + **tools** |
| `MINIMUM_CONTEXT_LENGTH` | auxiliary 压缩模型硬下限 |

**Smart routing / runtime fallback：** session.error 时换链 — 与 `chat.params` proactive fallback 独立 [15](./15-provider-and-transport.md) · Wiki smart-model-routing。

**`/model` slash：** 显式换 provider/model，**接受** system prefix invalidation [13](./13-prompt-assembly-and-cache.md)。

---

## 6. Nudge 相关 config（与 05 交界）

| 键（概念） | 驱动 |
|------------|------|
| `memory.nudge_interval` | `_turns_since_memory` → background_review memory 轨 |
| skill nudge interval | `_iters_since_skill` → background_review skill 轨 |

Gateway fresh Agent 从 history **hydrate** 计数 — 见 05 §2 hydrate。

---

## 7. 源码带读

1. `IterationBudget.consume/refund`  
2. `conversation_loop` 362 + while 条件  
3. `model_metadata.estimate_request_tokens_rough` 调用点（preflight）  
4. `config.yaml` 示例 + `hermes doctor` 输出对照  

---

## 8. 自测

- [ ] 父与子 iteration budget 是否共享计数？
- [ ] 每 user message budget 是否重置？
- [ ] preflight 为何要把 tools schema 算进 token？
- [ ] `/model` 与 silent 改 config 对 cache 的差别？
- [ ] execute_code 与 refund 关系？

**关联：** [01](./01-architecture-overview.md) · [05 Loop](./05-aiagent-and-conversation-loop.md) · [07 Toolsets](./07-toolsets-and-platform-bundles.md) · [11 Delegation](./11-delegation-cron-and-kanban.md)
