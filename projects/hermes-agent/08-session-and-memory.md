# 08 · Session 持久化与 Memory

> **锚点：** `hermes_state.py`（SessionDB）· `agent/memory_manager.py` · `agent/memory_provider.py` · `agent/agent_runtime_helpers.invoke_tool`（memory 分支）  
> **前置：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [13 Prompt/Cache](./13-prompt-assembly-and-cache.md)

Hermes 有两条易混的「记忆」线：**会话 transcript**（SQLite + FTS）与 **跨会话 memory**（MEMORY.md + 可选 external provider）。再加第三条 **`conversation_history`** — 单 turn 内存中的 message 列表。本章按 **读写时机** 带读，并说明 compression 如何 **旋转 session_id**。

---

## 1. 三条存储，三种生命周期

| 名称 | 存储 | 生命周期 | 典型读者 |
|------|------|----------|----------|
| `conversation_history` | 内存 list | 单 turn 拷贝/变异 | `run_conversation` while |
| **SessionDB** | `~/.hermes/sessions.db`（路径因 install 而异） | 跨 turn、跨 Gateway 消息 | CLI resume、Gateway 同 session_key |
| **Memory** | MEMORY.md / USER.md + provider | 跨 session | system volatile + prefetch 块 |

```text
Turn 开始
  SessionDB → 加载 history + system_prompt 行
  MemoryManager.prefetch_all（整 turn 一次，见 [05](./05-aiagent-and-conversation-loop.md)）
Turn 中
  append 到 messages；周期性 flush 到 SessionDB
Turn 结束
  sync_turn；可选 background_review
Compression
  end_session(old) → create_session(new, parent=old) → 新 system 行
```

---

## 2. SessionDB（`hermes_state.py`）

### 2.1 能力与 schema

- **SCHEMA_VERSION = 13**（迁移链 v10 trigram FTS、v11 重索引 tool_name/tool_calls）
- **双 FTS5 表：** 默认 `unicode61` + **trigram** 表 — 后者支持 CJK 子串搜索（255–284 行注释）
- WAL + fallback（`apply_wal_with_fallback`）— 并发 cron/gateway/delegate 时减少锁失败

### 2.2 关键 API

| 方法 | 用途 |
|------|------|
| `create_session` / `end_session` | 生命周期；compression 时 `end_session(..., "compression")` |
| `get_session(session_id)` | 含 **`system_prompt`** 列 — Gateway prefix cache 复用 [13](./13-prompt-assembly-and-cache.md) |
| `update_system_prompt` | 首 build 或 compression 后写入完整 system 快照（759–766 行） |
| `append_message` | 持久化单条 message；multimodal content JSON 编码（1492–1494 行） |
| FTS `search_messages` | `session_search` tool 底层；MATCH 查询 sanitize（2037+ 行） |

**`append_message` 额外字段：** `tool_calls`、`reasoning_*`、`platform_message_id`（Telegram 等平台 recall 用）、`observed` 等 — 压缩/回放时保留 tool 配对信息。

### 2.3 与 loop 的衔接

| 时机 | 行为 |
|------|------|
| Turn 开始 | 从 DB 加载 history；`_restore_or_build_system_prompt` 读 `system_prompt` 四态 |
| Turn 中 | `_flush_messages_to_session_db` 增量写入 |
| Preflight 压缩后 | `conversation_history = None` — 强制 flush **全部** 压缩后 messages 到新 session（loop 510–515 行） |
| Gateway 新 Agent | 同 `session_id` 读 DB；hydrate todo/nudge [05](./05-aiagent-and-conversation-loop.md) |

### 2.4 Session lineage（compression）

`conversation_compression.compress_context`（375–410 行）：

```text
commit_memory_session(old messages)
end_session(old_id, "compression")
new_id = timestamp_uuid
create_session(new_id, parent_session_id=old_id)
update_system_prompt(new_id, new_system_prompt)
agent.session_id = new_id
```

**意义：** SQLite 里保留 **父子链**；`session_search` 可跨 lineage 召回；title 自动编号（`get_next_title_in_lineage`）。

**失败路径：** DB split 失败 → 警告；逻辑 session 可能已变但索引不完整 — 运维看 log `Session DB compression split failed`。

---

## 3. `session_search` 工具

**入口：** `invoke_tool` → `session_search` 分支（1550–1566 行），**非** registry。

```text
query + role_filter + limit
  → SessionDB FTS（+ trigram 子串）
  → 可选 around_message_id 窗口
  → 可选 auxiliary LLM 摘要（长结果）
```

与 **memory prefetch** 不同：session_search 是 **模型主动** 跨 session 检索；prefetch 是 turn 前 provider 推送。

Built-in **learning loop** 一环 [09 §6](./09-skills-curator-and-learning-loop.md)。

---

## 4. MemoryManager

`agent/memory_manager.py` — docstring 明确：**run_agent 唯一集成点**。

### 4.1 单 external provider 规则

```258:280:/Users/zmz/Github/hermes-agent/agent/memory_manager.py
if not is_builtin:
    if self._has_external:
        logger.warning("Rejected memory provider '%s' — ... Only one external ...")
        return
```

- **builtin**（MEMORY.md）始终存在
- Honcho / Mem0 / … **最多一个** — 防 tool schema 膨胀与写冲突

### 4.2 Turn 生命周期

```text
build_system_prompt()           → system volatile 块（frozen snapshot）[13]
on_turn_start(turn, message)    → provider 钩子
prefetch_all(user_message)      → turn 前 recall（conversation_loop 只调一次）
build_memory_context_block()    → 注入 user 侧 fence，非 system
sync_all(user, assistant)       → turn 后写入 provider
queue_prefetch_all()            → 异步预热下一 turn
on_pre_compress(messages)       → 压缩丢弃前抽取 insight [14]
on_session_end / shutdown       → 边界清理
```

**Prefetch 只一次：** `conversation_loop` 注释 — 每 iteration 重调 `prefetch_all` 会 10× 延迟/费用；query 用 **original_user_message** 避免 skill 注入污染 recall。

### 4.3 Context fence 与 UI  scrub

```227:241:/Users/zmz/Github/hermes-agent/agent/memory_manager.py
def build_memory_context_block(raw_context: str) -> str:
    return (
        "<memory-context>\n"
        "[System note: ... NOT new user input ...]\n\n"
        f"{clean}\n"
        "</memory-context>"
    )
```

- `sanitize_context` — 去重 fence / 防 provider  double-wrap
- `StreamingContextScrubber` — 流式 delta **跨 chunk** 剥掉 `<memory-context>`，防 TUI 泄露

### 4.4 Memory 工具路由

| 调用 | 路径 |
|------|------|
| `memory` tool（builtin） | `invoke_tool` → `memory_tool` + `on_memory_write` bridge |
| provider 自定义 tool | `MemoryManager.handle_tool_call` |
| `handle_function_call` | `_AGENT_LOOP_TOOLS` 拒绝 — 必须走 invoke_tool |

**Delegate 子 agent：** `memory` 在 block 列表 — 防 fork 写父级 MEMORY [11](./11-delegation-cron-and-kanban.md)。

---

## 5. MemoryProvider ABC

| 方法 | 时机 |
|------|------|
| `system_prompt_block()` | 组装 system volatile |
| `prefetch(query)` | turn 前 |
| `sync_turn(user, asst)` | turn 后 |
| `get_tool_schemas()` / `handle_tool_call()` | 可选 |
| `on_pre_compress(messages)` | 压缩前 — 抽取将丢失的 turn |
| `on_session_start(..., boundary_reason="compression")` | session_id 旋转后刷新 provider 缓存 (#6672) |
| `shutdown()` | 退出 |

**Frozen snapshot 政策：** turn 内 `memory` tool 写盘 **不** 刷新 system volatile；模型仍从 tool result 看到 live 状态；下 session 或 compression 后 `invalidate_system_prompt` 才重建 [13](./13-prompt-assembly-and-cache.md)。

---

## 6. 内置 MEMORY.md

- 路径：`get_hermes_home()` 下，与 skills 并列
- `MemoryStore` — load/save；compression 时 `invalidate_system_prompt` 会 `load_from_disk()`
- Cron：`skip_memory=True` — 定时任务默认不写 external provider（防污染）

---

## 7. 插件 memory（`plugins/memory/`）

内置多种 provider；**2026-05 政策：** 新 backend 应 **独立 repo** → `~/.hermes/plugins/`，不再 in-tree。

激活：`memory.provider` + `hermes memory setup`。

---

## 8. 与 Prompt Cache / Compression 的交界

| 事件 | system 快照 | SessionDB |
|------|-------------|-----------|
| 普通 memory tool 写入 | 不变 | 不变（仅 MEMORY.md） |
| compression 成功 | **rebuild** + 新 session 行 | 新 `session_id`，parent 链 |
| Gateway 续聊 | 复用 **旧** system 字符串 | 同 session_id |

---

## 9. 源码带读顺序（约 2–3h）

| 顺序 | 文件 | 关注 |
|------|------|------|
| 1 | `hermes_state.py` SessionDB 类 + FTS 迁移 | schema v13 |
| 2 | `conversation_loop` prefetch + flush | 一次 prefetch |
| 3 | `memory_manager.py` add_provider + prefetch_all | 单 external |
| 4 | `conversation_compression.compress_context` 375–435 | session 旋转 |
| 5 | `invoke_tool` memory/session_search 分支 | agent-level |

---

## 10. 自测

- [ ] SessionDB vs `conversation_history` vs MEMORY.md 各存什么？
- [ ] compression 后 `session_id` 为何变、`parent_session_id` 用途？
- [ ] 为何 prefetch 不能每 iteration 调用？
- [ ] turn 内改 MEMORY 为何 system 不变？
- [ ] 为何只允许一个 external provider？
- [ ] `session_search` 与 prefetch 边界？
- [ ] delegate 为何 block memory？

**关联：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [09 Skills §10](./09-skills-curator-and-learning-loop.md#10-skills-与-memory-协作) · [14 压缩](./14-context-compression.md) · [06 Tools](./06-tools-registry-and-model-tools.md)
