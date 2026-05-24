# 10 · Gateway 与平台适配

> **锚点：** `gateway/run.py` · `gateway/platforms/base.py` · `gateway/platforms/*.py`

---

## 1. Gateway 在架构中的位置

```text
Telegram / Discord / Slack / …
        ↓ webhook / long-poll / socket
PlatformAdapter (gateway/platforms/*)
        ↓ normalize → HermesMessageEvent
GatewayRunner (gateway/run.py)
        ↓ session key → AIAgent cache
AIAgent.run_conversation
        ↓
PlatformAdapter.send_* (回复、媒体、thread)
```

CLI 与 Gateway **共用** `AIAgent` + `conversation_loop`；差异在 **I/O 面** 与 **platform toolset 基线** [07]。

---

## 2. GatewayRunner 核心职责

| 职责 | 实现要点 |
|------|----------|
| 多平台生命周期 | 启动 adapter、health、shutdown notify |
| Session → Agent 映射 | OrderedDict agent cache + LRU cap |
| 消息排队 | `_pending_messages` — interrupt 期间单槽排队 |
| 双 guard | 入站/出站内容安全（见 §4） |
| Cron tick | 后台线程每 **60s** 调 `cron.scheduler.tick()` [11] |
| 审批 | `tools/approval.py`  per-session 队列（非 stdin） |

**Session key：** 通常 `platform:chat_id[:thread]` — 同 key 复用 Agent 实例与 DB session_id。

---

## 3. Agent 缓存与 expiry

`gateway/run.py` 维护 **活跃 AIAgent 缓存**：

- **OrderedDict** — `_enforce_agent_cache_cap()` pop LRU
- **Expiry watcher** — 空闲超时释放（配置相关）
- Shutdown 时 `_notify_active_sessions_of_shutdown()` 优雅通知

**意义：** 避免每条消息 new AIAgent（重建 tools/system 贵 + cache miss）；但要 cap 防内存泄漏。

---

## 4. 双 Guard（概念）

Gateway 在消息进入 loop **前** 与工具/回复 **出站前** 做策略检查（具体规则分散在 `run.py` + `platforms/base.py` + 各 adapter）：

| 方向 | 典型拦截 |
|------|----------|
| **入站** | Prompt injection 模式、非法 attachment、未授权 chat |
| **出站** | `send_message` 目标校验、敏感内容、媒体类型转换 |

Webhook 平台常用 **`_HERMES_WEBHOOK_SAFE_TOOLS`** 子集 [07] — 无 `terminal`/`execute_code` 等。

Telegram 音频/语音：`base.py` 内 `_AUDIO_EXTS` / `_TELEGRAM_VOICE_EXTS` 与 `send_message_tool` 同步。

---

## 5. PlatformAdapter 基类

`gateway/platforms/base.py`（4000+ 行）定义共享行为：

- **Thread metadata** — Telegram DM topic、`reply_to_message_id`
- **媒体路由** — 图片/音频/文档
- **Typing / reaction** 指示器
- **Proxy** — `normalize_proxy_url`
- **重试与 rate limit** 钩子

各平台文件（~32 个）：`telegram.py`, `discord.py`, `slack.py`, `feishu.py`, `matrix.py`, …

**实现模式：** 继承 base → 实现 `start/stop`、入站 parse、出站 send。

---

## 6. Interrupt 与 pending 消息

用户连发消息时：

```text
_interrupt_requested on active agent
  → 当前 turn 尽快退出（loop break；tool_executor preflight 跳过剩余 calls）
  → 新消息写入 adapter._pending_messages[session_key]（单槽「下一 turn」）
  → turn 结束后 drain → 下一 run_conversation
```

**设计：** 单槽 **不 merge** 多条为一条 — 保序逐条处理。与 `/queue` FIFO 不同（见 §7）。

**源码：** `gateway/run.py` 1616–1626（Runner 注释区分 slot vs `_queued_events` overflow）；`agent/tool_executor.py` 74–83（interrupt 时 synthetic cancelled tool results）。

---

## 7. `/queue` FIFO 与 burst

`GatewayRunner._enqueue_fifo` / `_promote_queued_event`（约 2462–2511 行）：

```text
adapter._pending_messages[session_key]  = 单槽 head（与 interrupt follow-up 共享）
self._queued_events[session_key]      = overflow 尾队列
  → drain 后 promote 下一项进 slot
  → /new、/reset 清空；/model 保留队列
```

**与 interrupt 单槽区别：** `/queue` 每条必须产生 **完整独立 turn**，不可 collapse；burst 连发则逐条 drain。

**busy 模式：** `_restart_requested` + `busy_input_mode` 为 `queue`/`steer` 时，重启过程不丢消息（2451–2449 行）。

---

## 8. SessionDB 与 PlatformRegistry

**Session key：** `build_session_key` / `session_store._generate_session_key` — 通常 `platform:chat_id[:thread]`。同 key → 同 `session_id` + `_agent_cache` 命中 [§3](#3-agent-缓存与-expiry)。

**System prompt 四态：** Gateway 每 turn 可 new `AIAgent`，但 `_restore_or_build_system_prompt` 从 SessionDB **verbatim** 复用 system 行 — 否则 prefix cache 每 turn miss [13](./13-prompt-assembly-and-cache.md)。`gateway/run.py` 1638–1641 注释写明无 cache 时 Anthropic 约 **10×** 成本。

**PlatformRegistry（v0.13+）：** Teams、LINE、Google Chat 等 adapter 插件化；22+ 通道 normalize 为 `MessageEvent` 后进 `_handle_message_with_agent`。

**PII redaction：** 日志/导出脱敏；**不**改进入模型的 prompt（Wiki gateway-session-management）。

**pre_gateway_dispatch：** 插件 hook 可 skip/rewrite 入站 [16](./16-plugins-mcp-and-hooks.md) — 顺序相对 pairing/auth 以源码为准。

---

## 9. 投递与媒体

- Telegram：`base.py` 内 `_AUDIO_EXTS` / voice vs audio 分流；与 `send_message_tool` 同步
- Cron delivery 复用 platform send [11](./11-delegation-cron-and-kanban.md)
- Native image：`_pending_native_image_paths_by_session` 与 adapter 协作

---

## 10. 与 CLI 的差异

| 维度 | CLI | Gateway |
|------|-----|---------|
| stdin/TUI | prompt_toolkit | 无 — 消息驱动
| 审批 | terminal TLS callback | session approval queue |
| cwd | 用户 shell cwd | `terminal.cwd` 配置 |
| `send_message` | 通常不可用 | 可用 |
| Background terminal | notify 配置 | 同左，platform 通知 |

**Subagent deadlock：** delegate 在 worker 线程无 CLI approval callback — 见 [11]。

---

## 11. 配置与启动

```bash
hermes gateway start          # 或 platform 子命令
hermes gateway doctor
```

`config.yaml`：`gateway.*`、各 platform token/webhook、`tools.telegram.enabled` 等。

环境变量展开走 `mcp_env` 同类 allowlist 策略（见 AGENTS.md）。

---

## 12. 调试

- 日志：模块 logger + gateway 启动 banner
- Session 问题：查 SessionDB session_id 与 gateway session_key 映射
- 工具缺失：platform tool config + `check_fn` [06]

---

## 13. 自测

- [ ] GatewayRunner 与 PlatformAdapter 分工？
- [ ] agent cache 为何要 LRU cap（128）与 1h idle TTL？
- [ ] interrupt 单槽 vs `/queue` overflow 区别？
- [ ] Gateway 为何不 expand `@file`？
- [ ] SessionDB system `present` 对费用的影响？
- [ ] webhook 平台为何 tool 子集更严？

**关联：** [03 入口](./03-cli-gateway-and-entry.md) · [07 Toolsets](./07-toolsets-and-platform-bundles.md) · [11 Cron](./11-delegation-cron-and-kanban.md) · [26 总图](./26-main-chain-atlas.md) · [22 手册 §4](./22-integrations-handbook.md#4-context-引用file--附件)

> 原 [34 Gateway 深读](./34-gateway-messaging-queues-and-sessions.md) 已并入本章 §6–§9。
