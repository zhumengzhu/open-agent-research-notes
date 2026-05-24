# 14 · Context 压缩（Compressor v3）

> **锚点：** `agent/context_compressor.py` · `agent/conversation_compression.py` · `agent/conversation_loop.py`（preflight）· `agent/auxiliary_client.py`  
> **前置：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [08 Session](./08-session-and-memory.md) · [13 Prompt/Cache](./13-prompt-assembly-and-cache.md)

压缩是 Hermes **唯一 routinely 允许改写历史 messages** 的路径（AGENTS.md）。它同时触发 **session_id 旋转**、**system rebuild**、**memory provider 抽取** — 读 loop 时必须把本节与 08、13 联读。

**外部（A）：** [Wiki context-compressor](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/context-compressor-architecture.md) · [官方 context-compression-and-caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching.md)

---

## 1. 触发点（两处）

### 1.1 Preflight（进 while 之前）

`conversation_loop.py` 467–524 行 — **换小 context 模型** 或历史已膨胀时，避免 first API 直接 4xx：

```text
estimate_request_tokens_rough(messages, system_prompt, tools=agent.tools)
  → 含 tool schema token（旧估算漏掉 20–30K+）
  → >= threshold → 最多 3 pass _compress_context
  → conversation_history = None（flush 语义）
  → 重置 _empty_content_retries 等（压缩后 fresh budget）
```

### 1.2 Loop 内（iteration 级）

超阈时在 while 内再次调用 `_compress_context`（与 preflight 共用 `compress_context`）。Manual `/compress` 传 `force=True` 清 cooldown。

---

## 2. 调用栈

```text
run_conversation
  → agent._compress_context(...)          # run_agent.py 3911 转发
  → conversation_compression.compress_context
       → memory_manager.on_pre_compress
       → context_compressor.compress
       → invalidate + rebuild system
       → SessionDB end/create + parent lineage
       → context engine / memory on_session_start(boundary_reason="compression")
```

---

## 3. `ContextCompressor.compress` 算法

入口：`context_compressor.py` 1495 行 docstring + 实现。

```text
Phase 0  force=True → 清 _summary_failure_cooldown（1526–1530）
Phase 1  _prune_old_tool_results → _PRUNED_TOOL_PLACEHOLDER（cheap，无 LLM）
Phase 2  head 保护 + token-budget tail（非固定条数 protect_last_n）
Phase 3  中间 turns → auxiliary LLM 结构化摘要
Phase 4  拼回：SUMMARY_PREFIX + summary + tail messages
Phase 5  清理 orphan tool_call / tool_result 配对
```

**v3 相对 v2（模块头注释）：** Resolved/Pending 问题跟踪、「Remaining Work」替 「Next Steps」、迭代摘要（`_find_latest_context_summary`）、按比例 summary budget、tool output 预 prune。

**Multimodal 预算：** `_content_length_for_budget` — 每张图按 `_IMAGE_TOKEN_ESTIMATE=1600` tokens 计（79–109 行），防「5 张图 + 空 text」被当成零 token。

---

## 4. `SUMMARY_PREFIX` 语义

```37:51:/Users/zmz/Github/hermes-agent/agent/context_compressor.py
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted ..."
    "IMPORTANT: Your persistent memory (MEMORY.md, USER.md) in the system "
    "prompt is ALWAYS authoritative ..."
    "Respond ONLY to the latest user message that appears AFTER this summary."
)
```

**防什么：** 模型把摘要里的旧 user 请求当 **当前任务**；或忽视 MEMORY.md。Legacy 前缀 `LEGACY_SUMMARY_PREFIX` 仍识别。

---

## 5. 失败、冷却与 abort

| 状态 | 行为 |
|------|------|
| `_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600` | auto-compress 连环失败时暂停重试 |
| `force=True`（/compress） |  bypass cooldown |
| `_last_compress_aborted=True` | aux 摘要失败 → **messages 不变**，**不** 旋转 session（compress_context 327–339 行） |
| `abort_on_summary_failure=True` | 配置项 — abort 而非 fallback placeholder |
| fallback placeholder | 插入静态 marker，仍可能 drop middle window |

**Aux 模型 fallback：** 配置的 `auxiliary.compression.model` 失败但主模型救场 → `_emit_warning` 提示检查 config（354–365 行）。

**启动探测：** `check_compression_model_feasibility` — aux context < main threshold 时 auto-lower 或 hard reject；Gateway 在首 turn 经 `replay_compression_warning` 推到平台（233–248 行）。

---

## 6. `compress_context` 副作用（成功路径）

摘自 `conversation_compression.py` 251–435 行：

```text
1. on_pre_compress(messages)           # memory provider 抽取
2. compressed = compressor.compress(...)
3. todo_snapshot 注入 compressed 末尾（若有）
4. _invalidate_system_prompt()
5. new_system_prompt = _build_system_prompt(...)
6. commit_memory_session(old messages)
7. end_session(old) → create_session(new, parent=old)
8. update_system_prompt(new)
9. context_compressor.on_session_start(boundary_reason="compression")
10. memory providers on_session_start(reset=False)  # 逻辑会话继续，仅 id 变
```

**与 cache：** 新 system 字符串 → provider prefix 需 re-warm；DB 存新 `system_prompt` 行供 Gateway 下 turn 复用 [13](./13-prompt-assembly-and-cache.md)。

**与 toolset/skills：** 压缩 **不** 换 enabled toolsets / skill 索引 — 除非用户显式 `/model`、`hermes tools` 等。

---

## 7. Auxiliary 链（摘要专用）

Side task 走 `auxiliary_client.call_llm` — 与主 loop 模型解耦 [15 §6](./15-provider-and-transport.md#6-auxiliary-侧任务路由)。

配置：`auxiliary.compression.model` / `.provider`；默认 auto 链从主 provider 开始，402 时沿链降级。

**为何不用主模型摘要：** 成本 + 避免主 context 再次膨胀；curator/background_review 同理 fork auxiliary。

---

## 8. ContextEngine 插件

`ContextCompressor` 继承/协作 `ContextEngine` — 单选插件 engine（如 hermes-lcm）可在压缩 **前** MD5 去重、Smart Collapse。插件 `compress()` 签名不含 `focus_topic` 时 TypeError fallback（317–320 行）。

---

## 9. 与 CC 对照

| | Hermes | Claude Code |
|---|--------|-------------|
| 触发 | preflight + loop 内 token 估算 | 多层 compact 管道 |
| 摘要模型 | auxiliary 链 | 常为主模型或专用 compact |
| Session | SQLite lineage 旋转 | 多 session 文件策略 |
| 用户命令 | `/compress [focus]` | `/compact` |

---

## 10. 时序图（成功压缩）

```mermaid
sequenceDiagram
  participant Loop as conversation_loop
  participant CC as compress_context
  participant CMP as ContextCompressor
  participant Aux as auxiliary_client
  participant DB as SessionDB

  Loop->>Loop: estimate tokens >= threshold
  Loop->>CC: _compress_context(messages)
  CC->>CC: on_pre_compress
  CC->>CMP: compress()
  CMP->>CMP: prune tool results
  CMP->>Aux: structured summary LLM
  Aux-->>CMP: summary text
  CMP-->>CC: compressed messages
  CC->>CC: invalidate + rebuild system
  CC->>DB: end_session(old)
  CC->>DB: create_session(new, parent=old)
  CC-->>Loop: compressed, new_system_prompt
```

---

## 11. 源码带读顺序（约 2–4h）

| 顺序 | 位置 | 关注 |
|------|------|------|
| 1 | `conversation_loop` 467–524 | preflight 三 pass |
| 2 | `conversation_compression.compress_context` 全文 | abort vs success |
| 3 | `context_compressor.compress` 1495–1700+ | phase 1–5 |
| 4 | `SUMMARY_PREFIX` + `_generate_summary` | 模板语义 |
| 5 | `hermes_state` end_session / create_session | parent 链 |

---

## 12. 自测

- [ ] preflight 为何要把 `tools=` 算进 token？
- [ ] compress abort 时 session_id 变吗？
- [ ] SUMMARY_PREFIX 与 MEMORY.md 权威关系？
- [ ] 600s cooldown 谁 bypass？
- [ ] compression 后为何 `conversation_history = None`？
- [ ] `on_pre_compress` 谁调用、目的？
- [ ] iterative summary（二次压缩）如何找到旧 summary？

**关联：** [05 Loop §6](./05-aiagent-and-conversation-loop.md#6-压缩与-context) · [08 §2.4](./08-session-and-memory.md#24-session-lineagecompression) · [13](./13-prompt-assembly-and-cache.md) · [15 Auxiliary](./15-provider-and-transport.md)
