# 13 · Prompt 组装与 Prefix Cache

> **锚点：** `agent/prompt_builder.py` · `agent/system_prompt.py` · `agent/prompt_caching.py` · `conversation_loop._restore_or_build_system_prompt`  
> **外部（A）：** [Wiki prompt-builder](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/prompt-builder-architecture.md) · [Wiki prompt-caching](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/prompt-caching-optimization.md)

AGENTS.md **硬政策：** mid-turn 禁止换 toolset、重建 system、reload skills/memory（**压缩边界**与显式 `/model`、`--now` 除外）。本章解释 **为何** 以及 **代码如何实现**。

---

## 1. 模块分工

| 模块 | 职责 |
|------|------|
| `prompt_builder.py` | 无状态片段：identity、platform hints、skills **索引**、context 文件扫描、注入防护 |
| `system_prompt.py` | 三层 `build_system_prompt_parts` → `build_system_prompt` |
| `prompt_caching.py` | Anthropic `cache_control` 注入（`system_and_3`） |
| `conversation_loop.py` | SessionDB 恢复/持久化 system 行 |

```text
prompt_builder（片段）
       ↓
build_system_prompt_parts → stable | context | volatile
       ↓
build_system_prompt → agent._cached_system_prompt
       ↓
SessionDB.update_system_prompt（首 build / 压缩后 rebuild）
       ↓
后续 turn：get_session → verbatim 复用（Gateway 关键）
```

---

## 2. 三层 tier（详解）

`system_prompt.py` docstring（1–21 行）定义语义：

### 2.1 stable（会话内尽量不变）

典型内容（`build_system_prompt_parts` 内组装）：

- `SOUL.md` 或 `DEFAULT_AGENT_IDENTITY`
- Tool use enforcement、OpenAI/Google 模型族 operational guidance
- `SKILLS_GUIDANCE` + **skills 索引**（非 SKILL.md 全文 [09](./09-skills-curator-and-learning-loop.md)）
- `PLATFORM_HINTS`（cli / telegram / cron…）
- Kanban、computer-use、session_search 等 **条件 guidance**（platform/tool 门控）
- Nous subscription block（若适用）

**不进 stable：** 每 API 变的 ephemeral 段（见下）。

### 2.2 context（cwd 相关，session 内通常稳定）

- 调用方传入的 `system_message`（若有）
- `build_context_files_prompt`：在 `TERMINAL_CWD` 或 `os.getcwd()` 下发现：
  - `.hermes.md` / `HERMES.md`、向上至 git root
  - `AGENTS.md`、`.cursorrules`、`SOUL.md`（若未在 stable 加载）
- 经 `_scan_context_content` 扫描（见 §4）

**Gateway：** 必须用 `TERMINAL_CWD` 指向 **用户项目**，否则会把 hermes 安装目录里的 AGENTS.md 塞进 prompt（`system_prompt.py` 264–267 行注释，~10k token 浪费）。

### 2.3 volatile（冻结快照）

- `MEMORY.md` / `USER.md` 块（`MemoryStore.format_for_system_prompt`）
- External memory provider 的 `build_system_prompt()` 块
- **日期级** timestamp + session_id + model + provider 行

```299:305:/Users/zmz/Github/hermes-agent/agent/system_prompt.py
    # Date-only (not minute-precision) so the system prompt is byte-stable
    # for the full day.  Minute-precision changes invalidate prefix-cache KV
    timestamp_line = f"Conversation started: {now.strftime('%A, %B %d, %Y')}"
```

**Frozen snapshot：** turn 内 `memory` tool 写盘 **不** 刷新 volatile 块 [08](./08-session-and-memory.md)；tool 结果仍显示 live 状态。下 session 或压缩后 `invalidate_system_prompt` 才重建。

### 2.4  deliberately 不在 system 里

- `ephemeral_system_prompt` — **仅 API 调用时** 临时注入，不进 DB/cache 行
- Skills 全文 — 仅 `skill` tool 按需加载
- Memory prefetch 块 — 走 user/context fence，非 system [08]

---

## 3. `build_system_prompt` 与 invalidate

```321:337:/Users/zmz/Github/hermes-agent/agent/system_prompt.py
def build_system_prompt(agent, system_message=None):
    """Called once per session (cached on agent._cached_system_prompt) and
    only rebuilt after context compression events."""
    parts = build_system_prompt_parts(...)
    return "\n\n".join(p for p in (parts["stable"], parts["context"], parts["volatile"]) if p)
```

```340:348:/Users/zmz/Github/hermes-agent/agent/system_prompt.py
def invalidate_system_prompt(agent):
    agent._cached_system_prompt = None
    if agent._memory_store:
        agent._memory_store.load_from_disk()  # 重建时读最新 MEMORY/USER
```

**调用 invalidate 的典型时机：** context compression 成功后；**不**在普通 memory tool 写入后。

---

## 3.1 `build_system_prompt` 与 invalidate

```python
# system_prompt.py — 语义摘要
def build_system_prompt(agent, system_message=None):
    # 每 session 缓存；compression 等边界才 rebuild
    parts = build_system_prompt_parts(...)
    return join(stable, context, volatile)

def invalidate_system_prompt(agent):
    agent._cached_system_prompt = None
    agent._memory_store.load_from_disk()  # rebuild 时读最新 MEMORY/USER
```

**调用 invalidate 的典型时机：** [14 压缩成功](./14-context-compression.md) 后；**不**在普通 `memory` tool 写入后。

---

## 4. Context 文件安全（prompt_builder）

`_CONTEXT_THREAT_PATTERNS` + invisible Unicode 检测（`prompt_builder.py` 36–71 行）：

- ignore previous instructions、hidden div、exfil curl、读 `.env` 等
- 命中 → `[BLOCKED: filename …]` 占位，**不加载原文**

与 [20 Security](./20-security-defense-layers.md) 第 5 层一致；SOUL.md 同样扫描。

---

## 5. SessionDB 四态（与 API cache 叠加）

见 [05 §6](./05-aiagent-and-conversation-loop.md#6-system-prompt-恢复四态)。

- **Provider 侧：** Anthropic/OpenRouter/Nous 的 prefix cache（1h TTL 等）
- **应用侧：** DB 存 **整段 system 字符串**，Gateway 每 turn 新 `AIAgent` 仍复用同一 prefix

两层叠加：DB miss → rebuild → provider 侧也要重新 warm cache。

---

## 6. `apply_anthropic_cache_control`

`prompt_caching.py` — **`system_and_3` 策略：**

```1:7:/Users/zmz/Github/hermes-agent/agent/prompt_caching.py
"""4 cache_control breakpoints — system prompt + last 3 non-system messages,
all at the same TTL (5m or 1h). Reduces input token costs by ~75% ..."""
```

```49:77:/Users/zmz/Github/hermes-agent/agent/prompt_caching.py
def apply_anthropic_cache_control(api_messages, cache_ttl="5m", ...):
    # breakpoints: system + last 3 non-system messages
```

- `copy.deepcopy` 后注入，不 mutate 内存中的 `conversation_history`
- OpenRouter / native Anthropic / Nous Portal 路径启用（官方 configuration 文档）
- **最多 4 个 breakpoint** — Anthropic 限制；skill 块可占其中 slot

---

## 7. Mid-turn 禁止事项（与代码对应）

| 用户操作 | 默认行为 | 原因 |
|----------|----------|------|
| `/skills install` | deferred 下一 session | 换 skills 索引 = 换 stable 前缀 |
| `hermes tools` 改 toolset | deferred | 换 API tools 列表 |
| memory 写入 | 立即写盘，**不** rebuild system | frozen snapshot |
| `/model` | 允许，接受 cache miss | 显式用户意图 |
| compression | 允许 rebuild + 改 history | 唯一 routine 大改 context 路径 |

插件 context 注入走 **user message**，不进 system [16](./16-plugins-mcp-and-hooks.md) — 官方 hooks 文档。

---

## 8. Skills 索引缓存

`clear_skills_system_prompt_cache()` — skill 安装/卸载后清索引；配合 deferred apply 避免 mid-turn 换 stable 块。

Curator / background_review **fork** 继承父 cached system — 故意 hit 同一 prefix [18](./18-multi-agent-panorama.md)。

---

## 9. 与 Claude Code 对照

| | Hermes | Claude Code |
|---|--------|-------------|
| 分层 | stable/context/volatile 显式 | 多文件 static/dynamic prompt |
| System 稳定性 | DB 行 + `_cached_system_prompt` | 类似 session 级组装 |
| Cache | `cache_control` system_and_3 | provider cache breakpoints |
| 项目 context | AGENTS.md in context tier | CLAUDE.md / rules |

---

## 10. 自测

- [ ] stable/context/volatile 各举 2 个真实字段？
- [ ] 为何 timestamp 只精确到日？
- [ ] invalidate 是否会 reload MEMORY.md？
- [ ] Gateway 为何依赖 DB system 行？
- [ ] ephemeral 与 volatile 区别？
- [ ] system_and_3 几个 breakpoint？

**关联：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [08 Memory](./08-session-and-memory.md) · [09 Skills](./09-skills-curator-and-learning-loop.md) · [14 压缩](./14-context-compression.md)
