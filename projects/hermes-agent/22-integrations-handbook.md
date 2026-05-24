# 22 · 集成子系统手册（Browser / PTC / 并行 / LSP / Voice …）

> **定位：** 原 22–31、36 等 **薄专题** 合并为一本 **带源码锚点** 的手册；单节可独立跳读。  
> **锚点：** `agent/tool_executor.py` · `tools/code_execution_tool.py` · `tools/browser_*` · `agent/lsp/` · `tools/voice_mode.py` · `agent/kanban/`  
> **Wiki 索引：** [12 §G 子系统](./12-coverage-gaps-and-external-resources.md)

各节结构固定：**入口函数 → 数据流 → 与 loop 边界 → 失败/限制**。避免只列概念表。

---

## 1. Browser 与 Web 工具

**入口：** `tools/browser_*`（navigate、snapshot、click…）+ `web_search` / `web_extract`（auxiliary 链 [15](./15-provider-and-transport.md)）

```text
model tool_call browser_*
  → registry.dispatch → Playwright/Chromium 会话（backend 相关）
  → snapshot/DOM 摘要回 JSON string → messages
```

| 要点 | 说明 |
|------|------|
| Mutating | `browser_click/type/...` 在 `MUTATING_TOOL_NAMES` [§3](#3-并行-tool-与-guardrails) |
| Gateway | webhook 平台常禁用 terminal/browser 全量 — `_HERMES_WEBHOOK_SAFE_TOOLS` [10](./10-gateway-platforms-and-sessions.md) |
| Vision | 截图分析可走 auxiliary vision 链，非主 loop 必走 |

**与 CC 对照：** CC 有 MCP browser；Hermes 内置 Playwright 工具集，走同一 `handle_function_call` 管道 [06](./06-tools-registry-and-model-tools.md)。

---

## 2. PTC 沙箱（execute_code）

**入口：** `tools/code_execution_tool.py`

Programmatic Tool Calling：子进程跑 Python，经 RPC 调父进程 `registry.dispatch`，**中间 tool 结果不进 context**。

```text
generate hermes_tools.py stub（动态 schema = 当前 enabled tools 子集）
  → 本地：UDS RPC listener
  → 远程 Docker/SSH：file-based request/response poll
  → 仅 stdout 摘要回模型
```

| 约束 | 源码/政策 |
|------|-----------|
| 子 agent | `delegate_task` **禁止** execute_code — 强制逐步 tool [11](./11-delegation-cron-and-kanban.md) |
| 白名单 | stub 只暴露 `_last_resolved_tool_names` [06](./06-tools-registry-and-model-tools.md) |
| 平台 | 本地 UDS：Linux/macOS；Windows 禁用 |
| 安全 | 继承 terminal backend 隔离；`env_passthrough` 不 bypass 凭证 scrub [20](./20-security-defense-layers.md) |

---

## 3. 并行 Tool 与 Guardrails

### 3.1 并行调度

**入口：** `agent/tool_executor.py` — `execute_tool_calls_concurrent`（约 65 行）

```65:83:/Users/zmz/Github/hermes-agent/agent/tool_executor.py
def execute_tool_calls_concurrent(agent, assistant_message, messages: list, ...):
    """Execute multiple tool calls concurrently using a thread pool.
    Results are collected in the original tool-call order ..."""
    if agent._interrupt_requested:
        ... synthetic cancelled tool results ...
        return
```

**Preflight 顺序（同函数内）：** interrupt → 解析 args → checkpoint（write_file/patch/破坏性 terminal）→ plugin `pre_tool_call` block → `_tool_guardrails.before_call` → 再进线程池。

**Mutating 集（不可与只读乱并行）：**

```41:59:/Users/zmz/Github/hermes-agent/agent/tool_guardrails.py
MUTATING_TOOL_NAMES = frozenset({
    "terminal", "execute_code", "write_file", "patch", ...
    "delegate_task", "send_message", ...
})
```

只读类（`read_file`, `grep`, `web_search`, `browser_snapshot`…）可并行；MCP server 需 `supports_parallel_tool_calls: true` [16](./16-plugins-mcp-and-hooks.md)。

**Gateway：** worker 线程装 terminal approval TLS，审批走 session 队列而非 stdin [10](./10-gateway-platforms-and-sessions.md)。

### 3.2 Loop guardrails

**入口：** `agent/tool_guardrails.py` — 纯函数 `ToolCallGuardrailConfig`

| 维度 | 防什么 |
|------|--------|
| exact failure repeat | 同 tool+args 连续失败 |
| same-tool streak | 同工具无进展重复 |
| idempotent no-progress | 只读工具连续相同结果 |

默认 **warning 不 block**；`hard_stop_enabled` 才熔断（config `tool_loop_guardrails`）。

### 3.3 大结果溢出

三层（与 [06](./06-tools-registry-and-model-tools.md) dispatch 后处理联动）：

1. per-tool `max_result_size_chars` 截断  
2. 超大写盘，模型见路径摘要  
3. 单 turn 多 tool 总 char 预算（`budget_config`）

---

## 4. Context 引用（@file / 附件）

**入口：** CLI/TUI 展开 `@path`；Gateway **不** expand（防路径注入与跨用户泄露 [10](./10-gateway-platforms-and-sessions.md)）。

展开后进入 user message；与 system tier 的 AGENTS.md 扫描不同层 [13](./13-prompt-assembly-and-cache.md)。Native image：Gateway 走 `_pending_native_image_paths_by_session` 与 adapter 媒体路径。

---

## 5. LSP 集成

**入口：** `agent/lsp/` + LSP 类 tools（goto、references、diagnostics…）

与 `read_file`/`patch` 并列注册 [06](./06-tools-registry-and-model-tools.md)；LSP 会话按 workspace 懒启动。Mutating rename 类工具走 mutating 路径。

**学习建议：** 先读 [06 §registry](./06-tools-registry-and-model-tools.md)，再打开 `tools/lsp_*.py` 对应 handler。

---

## 6. Voice 与 TUI 皮肤

**Voice 入口：** `tools/voice_mode.py` + `tui_gateway/server.py`

```text
/voice 或 TUI toggle → check_voice_requirements()
  → STT → 文本作为 user message → run_conversation
  → 可选 TTS 回平台
```

不改变 system/toolset [13](./13-prompt-assembly-and-cache.md)。Gateway voice memo 由 platform adapter 转写后进同一 loop。

**TUI/i18n：** `ui-tui/` TypeScript 壳；Python `hermes_cli` 为逻辑源。皮肤/i18n 改 TS 层，loop 无感 [03](./03-cli-gateway-and-entry.md)。

---

## 7. Trajectory 与研究管线

**入口：** `agent/trajectory.py` · `AIAgent._save_trajectory`（`save_trajectories=True` 时 turn 末）· `batch_runner.py` · 离线 `trajectory_compressor.py`

```text
run_conversation 结束
  → _save_trajectory(messages, ...)   # 可选 JSONL
  → 格式见官方 trajectory-format.md
离线：
  trajectory_compressor.py --input=data/...  # 采样/压 token 供训练
```

| 要点 | 说明 |
|------|------|
| 与 loop | **失败不 block** 用户回复；best-effort 写盘 |
| 与 SessionDB | transcript 主路径仍是 SQLite；trajectory 为 **研究/导出** 副本 |
| Atropos 等 | 外部训练栈消费 JSONL — 非运行时依赖 |

**带读：** 仓库 `website/docs/developer-guide/trajectory-format.md` + `agent/trajectory.py`。

---

## 8. Worktree 与 Checkpoint

### 8.1 CheckpointManager

**入口：** `tools/checkpoint_manager.py` — `agent._checkpoint_mgr`

```575:581:/Users/zmz/Github/hermes-agent/tools/checkpoint_manager.py
"""Call new_turn() at start of each conversation turn and
ensure_checkpoint(dir, reason) before any file-mutating tool call.
Deduplicates: at most one snapshot per directory per turn."""
```

**触发点（tool_executor preflight）：** `write_file` / `patch` 前；破坏性 `terminal` 命令前（见 [22 §3.1](./22-integrations-handbook.md#31-并行调度)）。

| Config（概念） | 作用 |
|----------------|------|
| `checkpoints.enabled` | 主开关 |
| `max_snapshots` | 每目录保留份数 |
| `max_total_size_mb` | 全局上限 |

TUI：`tui_gateway` 暴露 checkpoint 相关 RPC（`session["agent"]._checkpoint_mgr`）。

### 8.2 Git worktree

多分支并行开发（与 OpenCode team-mode worktree **不同产品**）。配置见 `config.yaml` worktree 段；工具在 `tools/` 下与 git 集成 — 读 AGENTS.md worktree 节 + 对应 test。

---

## 9. Hermes Proxy 与 ACP

**Proxy：** 独立进程转发 API；主 loop 仍用 `runtime_provider` 解析。

**ACP（Agent Client Protocol）：** `write_file`/`patch` 可走 editor 侧 approval；`handle_function_call` 内 ACP 分支 [06](./06-tools-registry-and-model-tools.md)。与 Gateway session approval 队列并行存在 [10](./10-gateway-platforms-and-sessions.md)。

Provider 解析与 auxiliary 链见 [15](./15-provider-and-transport.md)。

---

## 10. Kanban 板内部

**入口：** `agent/kanban/` + `todo` tool；Gateway/CLI 共享 SessionDB 中的 todo 状态。

与 [11 §Kanban](./11-delegation-cron-and-kanban.md) 编排章互补：11 讲 cron/delegate 边界，本节强调 **todo 持久化与 loop hydrate**（Gateway 新 Agent 从 history 恢复 nudge 计数，见 [05](./05-aiagent-and-conversation-loop.md)）。

---

## 11. 阅读顺序建议

| 若你关心 | 先读 |
|----------|------|
| 性能 / 多 tool | §3 + `tool_executor.py` 65–250 |
| 安全 / 沙箱 | §2 + [17 Terminal](./17-terminal-backends.md) |
| 消息平台能力边界 | §1 + [10 Gateway](./10-gateway-platforms-and-sessions.md) |
| 插件 MCP 并行 | §3 + [16 Plugins](./16-plugins-mcp-and-hooks.md) |
| 查概念 / redirect 落点 | [99 §2](./99-glossary-and-reading-map.md#2-源码模块--章节) · [路径 G](./learning-paths.md#路径-g导航与交叉引用--约-30-min) |

---

## 12. 自测

- [ ] `execute_tool_calls_concurrent` interrupt preflight 做什么？
- [ ] `MUTATING_TOOL_NAMES` 为何含 `browser_click` 不含 `browser_snapshot`？
- [ ] PTC 与连续 tool_call 的 context 成本差？
- [ ] Gateway 为何不 expand `@file`？
- [ ] auxiliary vision 与主 loop 分工？

**旧编号 redirect：** [23](./23-code-execution-sandbox.md) · [24](./24-tool-parallel-guardrails-and-overflow.md) · [25](./25-context-references.md) · [27–32](./27-lsp-integration.md) · [36](./36-kanban-board-internals.md)
