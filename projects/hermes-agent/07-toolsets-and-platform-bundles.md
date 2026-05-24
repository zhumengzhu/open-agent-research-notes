# 07 · Toolsets 与平台工具包

> **锚点：** `toolsets.py` · `model_tools._compute_tool_definitions` · `config.yaml` `tools.*` · `hermes_cli/tools_config.py`

Toolset 是 **配置层编组**；模型 API 只见扁平 tool 名列表。本章带读 **core 列表、平台差异、解析算法、gate**。

---

## 1. 两层：Toolset vs Tool

```text
config: tools.telegram.enabled = ["web", "file", ...]
  → resolve_toolset 展开 includes
  → 并集 tool 名 set
  → registry.get_definitions(names) + check_fn 过滤
  → OpenAI tools[] 进 run_conversation
```

**发现 ≠ 暴露：** `tools/*.py` register 了但不在 resolved set → 模型看不见 [06](./06-tools-registry-and-model-tools.md)。

---

## 2. `_HERMES_CORE_TOOLS`（31–73 行）

CLI 与多数 messaging 平台的 **共享 superset**（改一处，各 `hermes-*` platform 同步）：

| 类 | 工具（节选） |
|----|----------------|
| Web | `web_search`, `web_extract` |
| 终端 | `terminal`, `process` |
| 文件 | `read_file`, `write_file`, `patch`, `search_files` |
| 视觉/图 | `vision_analyze`, `image_generate` |
| Skills | `skills_list`, `skill_view`, `skill_manage` |
| Browser | `browser_navigate` … `browser_cdp`（13 个） |
| 规划 | `todo`, `memory`, `session_search`, `clarify` |
| 编排 | `execute_code`, `delegate_task`, `cronjob` |
| 消息 | `send_message`（gateway check_fn） |
| HA | `ha_*`（`HASS_TOKEN`） |
| Kanban | `kanban_*`（worker / 显式 toolset） |
| macOS | `computer_use`（cua-driver） |

各 `hermes-telegram`、`hermes-discord` 等条目在 `TOOLSETS` 里 **`"tools": _HERMES_CORE_TOOLS`**（395+ 行），再叠 platform 专属工具（如 Yuanbao `yb_*`）。

---

## 3. Webhook 安全子集

不可信入站（公开 PR webhook 等）**禁止** shell/文件/browser：

```78:83:/Users/zmz/Github/hermes-agent/toolsets.py
_HERMES_WEBHOOK_SAFE_TOOLS = [
    "web_search", "web_extract", "vision_analyze", "clarify",
]
```

`hermes-webhook` toolset（534–537 行）**仅**此四工具 — 与 `_HERMES_CORE_TOOLS` 对比即知 attack surface 差异 [20 Security](./20-security-defense-layers.md)。

---

## 4. `TOOLSETS` 字典结构

```python
"web": {
    "description": "...",
    "tools": ["web_search", "web_extract"],
    "includes": [],  # 可引用其它 toolset 名
}
```

- **原子 toolset：** `web`, `terminal`, `file`, `browser`, `messaging`, `research`, `development`, `memory`, `safe`, `moa`, `kanban`…
- **Platform toolset：** `hermes-telegram`, `hermes-discord`, … → 大多 `_HERMES_CORE_TOOLS` + 扩展
- **聚合：** `hermes-gateway` → `includes` 列出全部 platform toolset（540–543 行）

`get_toolset(name)` 还会 **合并** `registry.get_tool_names_for_toolset(name)` — 插件/MCP 动态注册进同名 toolset 时自动并上（567–572 行）。

---

## 5. `resolve_toolset` 算法

```600:631:/Users/zmz/Github/hermes-agent/toolsets.py
def resolve_toolset(name, visited=None):
    if name in {"all", "*"}:  # 递归全部 toolset
    if name in visited: return []  # 环 / 菱形依赖
    visited.add(name)
    toolset = get_toolset(name)
    # hermes-<platform> 未定义时：_HERMES_CORE_TOOLS + 插件注册工具
    # 合并 tools + 递归 includes
```

**菱形依赖：** 已访问节点返回 `[]`，工具由另一分支已收集 — 非 bug。

**Plugin platform：** `hermes-foo` 且 `platform_registry.is_registered("foo")` → 自动 `_HERMES_CORE_TOOLS` + 插件 tools（638–646 行）。

---

## 6. `get_tool_definitions` 消费路径

```338:359:/Users/zmz/Github/hermes-agent/model_tools.py
if enabled_toolsets is not None:
    for toolset_name in effective_enabled_toolsets:
        resolved = resolve_toolset(toolset_name)
        tools_to_include.update(resolved)
else:
    # 全部 toolset − disabled_toolsets
```

| 模式 | 行为 |
|------|------|
| CLI `-t web,file` | 只展开指定 toolsets |
| 默认 | 所有 toolset 减 disabled |
| Kanban worker | `HERMES_KANBAN_TASK` env → 强制 append `kanban` toolset [06](./06-tools-registry-and-model-tools.md) |

`AIAgent(platform="telegram")` → `load_*` 读 `tools.telegram.enabled/disabled` [03](./03-cli-gateway-and-entry.md)。

---

## 7. 配置与 UI

```yaml
tools:
  cli:
    enabled: [web, terminal, file, browser]
    disabled: [moa]
  telegram:
    enabled: [...]
```

- **`hermes tools`** — curses UI（`tools_config.py`）；勿用 deprecated `simple_term_menu`
- **动态 schema：** config mtime 进入 `get_tool_definitions` quiet 缓存键 [06 §5](./06-tools-registry-and-model-tools.md#5-get_tool_definitions-与-gateway-缓存陷阱)

---

## 8. `check_fn` gate（运行时）

Toolset 静态列表 ⊃ 实际 schema。常见 gate：

| 工具 | gate |
|------|------|
| `send_message` | gateway 进程在跑 |
| `terminal` | backend 可用（local/docker/ssh…） |
| `browser_*` | Playwright/Chromium |
| `kanban_*` | `HERMES_KANBAN_TASK` 或 profile 开 kanban |
| `computer_use` | macOS + cua-driver |
| `ha_*` | `HASS_TOKEN` |
| MCP tools | server 连接 + `supports_parallel_tool_calls` 等 |

`check_fn` False → **不出现在 schema**；模型无法调用不存在的工具名。

---

## 9. Prompt cache 政策

Mid-turn 换 toolset = 换 API `tools[]` 前缀 → prefix cache miss。Slash/config 变更默认 **deferred 下一 session**；`--now` opt-in [13 §8](./13-prompt-assembly-and-cache.md#8-mid-turn-禁止-vs-允许)。

---

## 10. 与 Claude Code 对照

| | Hermes | Claude Code |
|---|--------|-------------|
| 编组 | `toolsets.py` 显式 + includes | tools assemble + deny |
| 平台差异 | per-platform YAML | 主 CLI 为主 |
| 条件暴露 | `check_fn` 30s TTL | defer_loading / ToolSearch |
| 不可信面 | `_HERMES_WEBHOOK_SAFE_TOOLS` | 类似 MCP 沙箱策略 |

---

## 11. 源码带读

1. `_HERMES_CORE_TOOLS` + `_HERMES_WEBHOOK_SAFE_TOOLS`  
2. 任选一 `hermes-telegram` 与 `hermes-webhook` 条目对比  
3. `resolve_toolset("hermes-gateway")` 展开深度  
4. `model_tools._compute_tool_definitions` enabled 分支  

---

## 12. 自测

- [ ] core 与 webhook safe 差哪些工具？
- [ ] `includes` 与 `tools` 如何合并？
- [ ] 插件 platform 如何自动获得 core？
- [ ] check_fn 失败时模型能否「猜」工具名调用？
- [ ] Kanban worker 如何强制 kanban toolset？

**关联：** [06 Registry](./06-tools-registry-and-model-tools.md) · [10 Gateway](./10-gateway-platforms-and-sessions.md) · [05 Loop](./05-aiagent-and-conversation-loop.md) · [11 Delegation](./11-delegation-cron-and-kanban.md)
