# 15 · Provider 解析与 Transport

> **锚点：** `hermes_cli/runtime_provider.py` · `providers/` · `agent/chat_completion_helpers.py`（`try_activate_fallback`）· `agent/auxiliary_client.py`  
> **外部（A）：** [Wiki provider-transport](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/provider-transport-architecture.md) · [mudrii provider-runtime](https://github.com/mudrii/hermes-agent-docs/blob/main/developer-guide/provider-runtime.md)

Hermes 有三条易混的「换模型/换后端」路径：**runtime fallback**（主 loop 失败）、**auxiliary 链**（side task）、**credential pool 轮换**（同 provider 多 key）。本章分开讲。

---

## 1. 解析链概览

```text
(provider, model) + config + credential pool [21]
  → runtime_provider / resolve_provider_client
  → (api_mode, api_key, base_url, OpenAI-compatible client)
  → Transport preflight + API call（conversation_loop while 内）
```

**统一入口：** CLI、Gateway、Cron、delegate 子 agent、auxiliary、`ACP` 均复用 `resolve_provider_client` 族 — 避免 duplicate provider→key 映射（`try_activate_fallback` 777–798 行注释）。

---

## 2. `api_mode` 与 Transport

| api_mode | HTTP/协议形态 | 典型 provider |
|----------|---------------|---------------|
| `chat_completions` | OpenAI `/v1/chat/completions` | OpenRouter、多数 compat |
| `anthropic_messages` | Anthropic Messages API | Anthropic、MiniMax OAuth |
| `codex_responses` | OpenAI Responses（Codex 流） | openai-codex、xai |
| `codex_app_server` | Codex 子进程 app-server | 整 turn bypass [05](./05-aiagent-and-conversation-loop.md) |
| 插件扩展 | Bedrock、Responses 变体… | `ProviderProfile` |

Loop 侧（`conversation_loop` ~1051+）：

- `_build_api_kwargs` → messages + tools
- `apply_anthropic_cache_control` [13](./13-prompt-assembly-and-cache.md)
- `interruptible_api_call` — 后台线程 + stale timeout [05](./05-aiagent-and-conversation-loop.md)
- streaming vs non-streaming；tool_calls 格式互转

**Stale api_mode 防护：** 换 provider 时不沿用上一家的 `api_mode`（MiniMax OAuth 误走 `/chat/completions` 类 case — runtime_provider 注释）。

---

## 3. ProviderProfile 插件

`providers/base.py` — `ProviderProfile` + `register_provider()`：

- 内置 + `plugins/model-providers/<id>/`
- `get_provider_profile(id)` lazy 扫描
- 字段：`inference_base_url`、`env_vars`、`api_mode`、`fallback_models`、normalize 钩子

**与 PluginManager 区别：** ProviderProfile 管 **推理 endpoint**；PluginManager 管 tools/hooks/MCP — discovery 路径不同 [16](./16-plugins-mcp-and-hooks.md)。

---

## 4. Credential Pool（同 provider 多 key）

**模块：** `agent/credential_pool.py` — 详见 [21](./21-profiles-and-credential-pool.md)。

```text
401 / 429 / 402 on current key
  → mark entry exhausted + cooldown
  → pool 选下一 entry（round_robin / fill_first / …）
  → runtime_provider 重新解析 api_key
```

与 **runtime fallback** 正交：pool 换 **key**；fallback 换 **provider/model 链**。

---

## 5. Runtime Fallback（reactive，主 loop）

**入口：** `agent/chat_completion_helpers.try_activate_fallback`（720+ 行），loop 内 API retry 失败时调用（如 1026、1257+ 行）。

### 5.1 机制

```720:746:/Users/zmz/Github/hermes-agent/agent/chat_completion_helpers.py
def try_activate_fallback(agent, reason=None) -> bool:
    """Switch to the next fallback model/provider in the chain.
    Advances _fallback_index; returns False when exhausted."""
    ...
    fb = agent._fallback_chain[agent._fallback_index]
    agent._fallback_index += 1
```

| 步骤 | 行为 |
|------|------|
| 构建链 | `AIAgent.__init__` 从 config `fallback_model` / provider profile 组装 `_fallback_chain` |
| 激活 | 原地换 `provider`、`model`、`base_url`、client；`_fallback_activated = True` |
| 去重 | 跳过与当前 **相同** provider+model 或相同 base_url+model（#22548） |
| 冷却 | rate_limit/billing 时 `_rate_limited_until` +60s（仅离开 primary 时） |
| 下 turn | `_restore_primary_runtime()` 在 `run_conversation` 开头（294–297 行）试恢复主模型 |

### 5.2 典型触发

| 场景 | loop 行为 |
|------|-----------|
| Nous Portal 全局限流 | 跳过 API，直接 `_try_activate_fallback`（1009–1030 行） |
| Codex response 错误 status | 路由 fallback 而非裸抛（1176+ 行） |
| 空响应 / malformed | 「eager fallback」— 常见 rate-limit 症状（1257+ 行） |
| `classify_api_error` | `FailoverReason.rate_limit` / `billing` / … 影响 cooldown |

### 5.3 Happy path vs 失败

```text
Happy: primary 429 → retry 耗尽 → try_activate_fallback → OpenRouter 成功 → 本 turn 继续
Fail:  chain 耗尽 → return failed dict + user 可见错误（1031–1045 无 fallback 模板）
```

**与 auxiliary 区别：** fallback 改 **主 Agent 的** client/model；auxiliary 是 **独立** `call_llm`  side task。

**与 CC/OpenCode：** 类似 reactive session.error 换链；Hermes 无单独 global priority list — 链来自 **per-agent config** [02](./02-config-iteration-and-model-routing.md)。

---

## 6. Auxiliary 侧任务路由

**模块：** `agent/auxiliary_client.py` — 压缩、goal judge、curator、vision、session_search 摘要等。

**Text auto 链（docstring 1–15 行）：** 主 provider → OpenRouter → Nous Portal → custom → Anthropic → 直连 key providers → None。

**Vision 链更短；Codex OAuth 不在 auto fallback** — allow-list 漂移（25–30 行）。

**402：** `call_llm()` 沿 auto 链换下一 provider。

**Per-task：** `auxiliary.compression.*`、`auxiliary.goal_judge.*` [19](./19-goals-and-ralph-loop.md)。

| 消费者 | 笔记 |
|--------|------|
| ContextCompressor | [14](./14-context-compression.md) |
| Goal judge | fail-open [19](./19-goals-and-ralph-loop.md) |
| background_review / curator | 继承主 runtime 或 auxiliary [18](./18-multi-agent-panorama.md) |

---

## 7. 与 loop 的衔接

- `AIAgent` 持有 `provider`、`model`、`api_mode`；每 API call 前可刷新 pool entry  
- `/model` slash：**显式** 换模型，接受 cache invalidation [13](./13-prompt-assembly-and-cache.md)  
- `set_runtime_main(provider, model)` — turn 开头（conversation_loop 272–276）供 vision 等读 live override  
- Plugin hooks：`pre_api_request` 观测即将发出的 kwargs（1072+ 行）

---

## 8. 源码带读

1. `runtime_provider.py` — `_resolve_runtime_from_pool_entry`  
2. `try_activate_fallback` 720–850  
3. `restore_primary_runtime` in `agent_runtime_helpers.py`  
4. `conversation_loop` API retry 环 1003–1100  
5. `auxiliary_client.call_llm` 402 重试  

---

## 9. 自测

- [ ] pool 换 key vs fallback 换 provider 区别？  
- [ ] `_restore_primary_runtime` 何时调用？  
- [ ] fallback 为何会 skip 相同 provider/model？  
- [ ] auxiliary auto 链为何含主 provider 第一步？  
- [ ] Codex 为何不在 auxiliary auto 链？  
- [ ] Nous rate guard 如何 bypass 无意义 API 调用？  

**关联：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [14 压缩](./14-context-compression.md) · [21 Credential Pool](./21-profiles-and-credential-pool.md) · [16 Plugins](./16-plugins-mcp-and-hooks.md)
