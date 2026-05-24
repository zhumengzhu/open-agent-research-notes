# 06 · 工具注册与 model_tools

> **锚点：** `tools/registry.py` · `model_tools.py` · `agent/agent_runtime_helpers.py`（`invoke_tool`）· `hermes_cli/plugins.py`  
> **前置：** [05 Loop](./05-aiagent-and-conversation-loop.md) · [07 Toolsets](./07-toolsets-and-platform-bundles.md)

工具系统是 Hermes 的 **执行平面**：模型只见 OpenAI-format schema；运行时一切落到 `registry.dispatch`。本章按 **发现 → 编组 → 暴露 → 执行** 带读，并区分 **两条 dispatch 入口**（`invoke_tool` vs `handle_function_call`）。

---

## 1. 四步模型（不要混成一步）

| 步骤 | 模块 | 输出 |
|------|------|------|
| **发现** | `discover_builtin_tools()` | 模块 import 时 `registry.register()` |
| **编组** | `toolsets.py` | tool 名集合（见 [07](./07-toolsets-and-platform-bundles.md)） |
| **暴露** | `get_tool_definitions()` | 进 API 的 schema 列表 + `_last_resolved_tool_names` |
| **执行** | `invoke_tool` / `handle_function_call` | JSON string 回 `messages` |

**发现 ≠ 暴露：** `tools/foo.py` 注册了但不在任何 enabled toolset → 模型 **看不见**，但 MCP refresh 或测试仍可能 dispatch。

---

## 2. 依赖链（不可绕）

```text
tools/registry.py          # 零 upstream；singleton `registry`
       ↑
tools/*.py                 # 顶层 registry.register()
       ↑
model_tools.py             # import 时 discover_builtin_tools()（约 180 行）
       ↑
run_agent · gateway · cron · delegate 子 agent
```

**插件工具：** `PluginManager` → `ctx.register_tool()` → 同样 bump `registry._generation`，**不必**改 `tools/`。

**MCP 动态工具：** `deregister` + 重 register；generation 变化使 `get_tool_definitions` quiet 缓存失效。

---

## 3. `ToolRegistry` 核心机制

### 3.1 注册与 shadow 拒绝

```234:305:/Users/zmz/Github/hermes-agent/tools/registry.py
def register(self, name, toolset, schema, handler, check_fn=None, ...):
    """override=True 才允许插件故意替换 built-in。"""
    ...
    if existing and existing.toolset != toolset:
        if both_mcp: ...  # MCP 同名可覆盖
        elif not override:
            logger.error("Tool registration REJECTED: '%s' would shadow ...")
            return
    self._generation += 1
```

| 字段 | 运行时作用 |
|------|------------|
| `check_fn` | False → **不进 schema**（`get_definitions` 过滤） |
| `is_async` | `dispatch` 走 `_run_async` |
| `max_result_size_chars` | dispatch 后截断（默认见 `budget_config`） |
| `dynamic_schema_overrides` | 如 `execute_code` 动态工具列表 |

### 3.2 `check_fn` 双层缓存

```110:121:/Users/zmz/Github/hermes-agent/tools/registry.py
# check_fn TTL cache — ~30 s
_CHECK_FN_TTL_SECONDS = 30.0
```

- **30s TTL：** `check_terminal_requirements`、Playwright 探测等不必每 API call 重跑
- **单次 `get_definitions` 内** 再缓存同一 `check_fn` 结果

`hermes tools enable foo` 改 env 后，30s 内 schema 可能仍显示旧可用性 — 设计权衡。

### 3.3 `dispatch`

```390:416:/Users/zmz/Github/hermes-agent/tools/registry.py
def dispatch(self, name, args, **kwargs) -> str:
    entry = self.get_entry(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        ...
        return json.dumps({"error": sanitized})
```

**契约：** 永远返回 JSON 字符串；异常不抛出到 loop。

---

## 4. 自动发现

`discover_builtin_tools()`（`model_tools.py`）：

- 扫描 `tools/*.py`（排除 `registry.py`、`mcp_tool.py`）
- **AST** 检测模块 **顶层** `registry.register(` — 函数体内的 register **不会** 被发现
- `importlib.import_module` 触发 side-effect 注册

进程启动时 `discover_builtin_tools()` 在 `model_tools` import 链上执行一次（约 180 行）。

---

## 5. `get_tool_definitions` 与 Gateway 缓存陷阱

```290:325:/Users/zmz/Github/hermes-agent/model_tools.py
if quiet_mode:
    cache_key = (
        frozenset(enabled_toolsets), ...,
        registry._generation,
        cfg_fp,  # config mtime — 动态 schema 变更
        bool(os.environ.get("HERMES_KANBAN_TASK")),
    )
    cached = _tool_defs_cache.get(cache_key)
    if cached is not None:
        _last_resolved_tool_names = [t["function"]["name"] for t in cached]
        return list(cached)  # 浅拷贝 list，schema dict 只读共享
```

**为何浅拷贝：** Gateway 长驻进程里，若 caller 往 cached list **append** memory/LCM schema，会污染缓存 → 下一 session 出现 **重复 tool 名** → DeepSeek/Kimi 等 HTTP 400（issue **#17335**）。

**Kanban worker：** `HERMES_KANBAN_TASK`  env 强制把 `kanban` toolset 并入 enabled 列表（340–346 行），即使用户 profile 禁用了 kanban chat tools。

**全局 `_last_resolved_tool_names`：** `execute_code` stub 与 `handle_function_call` 的 `enabled_tools` 回退源；子 agent 应传 **caller 提供的** `enabled_tools`，避免覆盖父 session 工具集（832–839 行）。

---

## 6. 两条执行路径

Loop 里 tool 执行 **不是** 只有 `handle_function_call`：

```mermaid
flowchart TD
  TC[model tool_calls] --> PAR{并行?}
  PAR -->|否| SEQ[sequential path]
  PAR -->|是| CONC[execute_tool_calls_concurrent]
  CONC --> INV[invoke_tool]
  SEQ --> INV
  INV --> AGENT{agent-level?}
  AGENT -->|todo/memory/session_search/delegate| 专用 handler
  AGENT -->|memory provider tool| MemoryManager.handle_tool_call
  AGENT -->|其它| HFC[handle_function_call skip_pre_tool_call_hook=True]
  HFC --> REG[registry.dispatch]
```

### 6.1 `invoke_tool`（`agent_runtime_helpers.py` ~1521）

**Agent 级工具** 在此处理，**不**进 registry：

| 工具 | 行为 |
|------|------|
| `todo` | `agent._todo_store` |
| `memory` | `memory_tool` + 可选 `on_memory_write` 通知 external provider |
| `session_search` | SessionDB FTS + 可选 LLM 摘要 [08](./08-session-and-memory.md) |
| `delegate_task` | `_dispatch_delegate_task` [11](./11-delegation-cron-and-kanban.md) |
| external memory tools | `MemoryManager.handle_tool_call` |

其余 → `handle_function_call(..., skip_pre_tool_call_hook=True)`，因 `invoke_tool` 已做过 block 检查。

### 6.2 `handle_function_call`（`model_tools.py` ~741）

```767:846:/Users/zmz/Github/hermes-agent/model_tools.py
function_args = coerce_tool_args(function_name, function_args)
if function_name in _AGENT_LOOP_TOOLS:
    return json.dumps({"error": f"{function_name} must be handled by the agent loop"})
# pre_tool_call block（除非 skip）
# ACP edit approval：write_file / patch
# notify_other_tool_call — 重置 read loop 计数
result = registry.dispatch(...)  # execute_code 传 enabled_tools
# post_tool_call + transform_tool_result hooks
```

**`_AGENT_LOOP_TOOLS`：** `todo`, `memory`, `session_search`, `delegate_task` — 若误路由到此，返回 error 而非 silent 失败。

**Hook 顺序：**

1. `pre_tool_call` — 可 **block**（返回 error JSON）
2. `maybe_require_edit_approval` — ACP/Zed 路径
3. `registry.dispatch`
4. `post_tool_call` — 观测（含 `duration_ms`）
5. `transform_tool_result` — 可 **替换** result 字符串

**双 fire 防护：** `run_agent._invoke_tool` → `invoke_tool` 已检查 block 时，`handle_function_call` 设 `skip_pre_tool_call_hook=True`。

---

## 7. 并行 vs 串行（与 06 的交界）

`run_agent._execute_tool_calls`（3964 行）按 `_should_parallelize_tool_batch` 分流：

- 只读 batch → `execute_tool_calls_concurrent` → 每 worker 调 `invoke_tool`
- mutating / 路径冲突 → sequential

详见 [22 手册 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails)。Guardrail `before_call` 在 **并发 preflight** 与 sequential 路径均有。

---

## 8. 结果截断与错误净化

- per-tool `max_result_size_chars` → `registry.get_max_result_size`
- 异常字符串经 `_sanitize_tool_error` — 防 framing token / CDATA 污染 model context
- 大结果溢出三层见 [22 §3.3](./22-integrations-handbook.md#33-大结果溢出)

---

## 9. 异步与 event loop

| 场景 | Loop |
|------|------|
| 主 CLI 线程 | `_get_tool_loop()` 持久 loop |
| delegate worker | `_get_worker_loop()` thread-local |
| `registry.dispatch` + `is_async` | `_run_async` 桥接 |

**禁止** 热路径 `asyncio.run()` — 会关闭 loop，httpx 客户端 GC 报错（AGENTS.md）。

---

## 10. 添加 core tool Checklist

1. `tools/foo.py` — **模块顶层** `registry.register(...)`
2. **`toolsets.py`** — 加入 `_HERMES_CORE_TOOLS` 或新 toolset
3. `requires_env` → `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`
4. 测试：`scripts/run_tests.sh tests/tools/test_foo.py -q`

**优先插件：** `~/.hermes/plugins/<name>/` + `ctx.register_tool` — 不改 core；需 `override=True` 才能替换 built-in。

---

## 11. 源码带读顺序（约 2–3h）

| 顺序 | 文件 | 关注 |
|------|------|------|
| 1 | `registry.py` register + dispatch + get_definitions | generation、shadow、check_fn TTL |
| 2 | `model_tools.py` get_tool_definitions[definitions] + handle_function_call | 缓存 #17335、hook 链 |
| 3 | `agent_runtime_helpers.py` invoke_tool | agent-level 分叉 |
| 4 | `tool_executor.py` execute_tool_calls_concurrent preflight | 与 invoke_tool 关系 |
| 5 | 任选一 tool 如 `tools/grep_tool.py` | register 样板 |

---

## 12. 自测

- [ ] 发现与暴露的两步区别？
- [ ] 为何 `_AGENT_LOOP_TOOLS` 在 handle_function_call 里直接 error？
- [ ] quiet_mode 缓存为何要 `list(cached)` 浅拷贝？
- [ ] `skip_pre_tool_call_hook` 谁设、为何？
- [ ] execute_code 为何传 `enabled_tools` 而非只用全局 `_last_resolved_tool_names`？
- [ ] MCP refresh 如何使 tool defs 缓存失效？

**关联：** [07 Toolsets](./07-toolsets-and-platform-bundles.md) · [05 Loop](./05-aiagent-and-conversation-loop.md) · [16 Plugins](./16-plugins-mcp-and-hooks.md) · [22 并行 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails)
