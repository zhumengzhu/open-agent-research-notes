# 21 · Profiles 与 Credential Pool

> **锚点：** `hermes_cli/profiles.py` · `agent/credential_pool.py` · `hermes_constants.get_hermes_home()`  
> **外部（A）：** [Wiki configuration-and-profiles](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/configuration-and-profiles.md) · [Wiki credential-pool](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/credential-pool-and-isolation.md)

**Profile** = 换整颗数据宇宙；**Credential pool** = 同一 provider 多 key 轮换。正交、常组合使用。

---

## 1. 概念对照

| 概念 | 隔离边界 | 典型场景 |
|------|----------|----------|
| **Profile** | 独立 `HERMES_HOME`：config、session、memory、skills、cron、gateway pid | 工作/个人、**多 Telegram bot**、多租户 |
| **Credential pool** | 同 profile 内同 provider 多 entry | OpenRouter 多 key、429/402 轮换 |
| **Session** | SessionDB 内 `session_id` | 单聊天线程 |
| **Gateway session_key** | `platform:chat_id` | 同 profile 内多聊天 [10](./10-gateway-platforms-and-sessions.md) |

换 Profile ≠ session 内 `/model`：前者新进程/新 `HERMES_HOME`；后者同 Agent 显式换模型 [15](./15-provider-and-transport.md)。

---

## 2. Profile 目录布局

```text
default   → ~/.hermes/
named     → ~/.hermes/profiles/<name>/    # id: ^[a-z0-9][a-z0-9_-]{0,63}$
```

**解析：** `get_hermes_home()` ← `HERMES_HOME` env · `hermes -p NAME` · sticky `profile use`。

**新建时 bootstrap（`_PROFILE_DIRS` 38–52 行）：**

| 目录 | 用途 |
|------|------|
| `memories/` | MEMORY.md / USER.md |
| `sessions/` | SessionDB |
| `skills/` | 用户/agent skills + `.curator_state` |
| `cron/` | 定时任务 |
| `logs/` | agent / gateway 日志 |
| `workspace/` | 工作区 |
| **`home/`** | 子进程 `HOME` — 隔离 git/gh/ssh/npm 凭证（#4426） |

---

## 3. Profile CLI

```bash
hermes profile create work --clone       # config + .env + SOUL + MEMORY/USER
hermes profile create work --clone-all   # 全量；strip gateway.pid 等运行时文件
hermes -p work gateway start             # 该 profile 独立 gateway
hermes profile use work                  # sticky 默认 HERMES_HOME
hermes profile delete work               # 停 gateway + 删 systemd/launchd 服务
```

**clone-all 从 default 排除（94–97 行）：** `hermes-agent` 源码树、`.worktrees`、`profiles/` 兄弟目录、`node_modules`… — 防复制 3GB venv。

**删除 profile：** `_cleanup_gateway_service` → `_stop_gateway_process` — 先停服务再删目录（829+ 行 docstring）。

---

## 4. 多 Gateway / 多 Bot 模式

```text
Profile "personal"  → ~/.hermes/profiles/personal/
  → config.yaml: tools.telegram.token = BOT_A
  → gateway.pid + 独立 systemd unit

Profile "work"      → ~/.hermes/profiles/work/
  → config.yaml: tools.telegram.token = BOT_B
  → 第二个 gateway 进程
```

| 隔离项 | 说明 |
|--------|------|
| SessionDB | 各 profile 独立 `sessions/` |
| Agent cache | Gateway LRU 按 **进程** 隔离 — 不同 profile 不共享 `_agent_cache` |
| Cron tick | `~/.hermes/cron/.tick.lock` 在 **该 profile 的 HERMES_HOME** 下 |
| Kanban / goals | 均 scoped 在 profile 内 [18](./18-multi-agent-panorama.md) |

**检测运行中：** `_check_gateway_running(profile_dir)` → `gateway.pid`（474–478 行）。

模块 docstring：每个 named profile 可配 **独立 systemd/launchd gateway 服务**。

---

## 5. CredentialPool 机制

**模块头：** Persistent multi-credential pool for same-provider failover。

### 5.1 数据模型

`PooledCredential`（93+ 行）：`provider`, `id`, `label`, `auth_type`（oauth/api_key）, `priority`, `access_token`, `base_url`, `last_status`, `request_count`…

**Custom endpoint：** pool key = `custom:<normalized_name>` — 多个 OpenAI-compat 不混池（79–82 行）。

### 5.2 选择策略（`_select_unlocked` 1219–1250 行）

| strategy | 行为 |
|----------|------|
| **fill_first**（默认） | `available[0]` — 按 priority 排序后的首个 |
| **round_robin** | 取首个可用 + 旋转 `_entries` 顺序并 persist |
| **random** | `random.choice(available)` |
| **least_used** | 最小 `request_count`，选中后 +1 |

`select()` 前：`_available_entries(clear_expired=True, refresh=True)` — 清冷却 + OAuth token refresh。

### 5.3 耗尽与轮换

```text
API 401/429/402
  → mark_exhausted_and_rotate(status_code=...)
  → entry.last_status = exhausted
  → cooldown: 401 → 5min; 429/402/default → 1h（75–77 行）
  → provider reset_at 可覆盖
  → _select_unlocked() → 下一 entry
```

**跨进程 sync：** exhausted 的 anthropic claude_code / nous / codex / xai-oauth entry 可能在 `_available_entries` 时从 auth store **重新 sync**（1149–1191 行）— 用户在其他 CLI 重登后 pool 可恢复。

### 5.4 与 runtime fallback 关系

| 机制 | 换什么 |
|------|--------|
| **Pool rotate** | 同 provider **不同 key** |
| **try_activate_fallback** | **不同 provider/model 链** [15 §5](./15-provider-and-transport.md#5-runtime-fallbackreactive主-loop) |

可串联：当前 key 429 → rotate；全部 exhausted → fallback 换 OpenRouter。

### 5.5 Gateway stale 边界

`AIAgent(credential_pool=...)` 构造时注入 pool **快照**。注释：已缓存 Agent 的 routing **不**随 ambient pool refresh 自动变 — 长驻 Gateway session 可能需新 Agent 或新 turn 才 pick 新 key [03 §5.1](./03-cli-gateway-and-entry.md#51-agent-缓存1638–1647-行)。

---

## 6. 配置入口

```yaml
credential_pools:
  openrouter:
    strategy: round_robin
    entries: [...]

# Profile 无第二格式 — 仅目录 + config.yaml
```

CLI：`hermes auth` · Nous Portal · Codex OAuth → 写入 pool / auth store。

---

## 7. 与 Multi-Agent 对照

| | delegate/kanban | Profile |
|---|-----------------|---------|
| 隔离 | 同 profile 内 runtime | 整棵 HERMES_HOME |
| 用途 | 并行子 Agent | 多 bot / 多用户数据 |

Wiki「第二种 multi-agent」= Profile 维度 [18](./18-multi-agent-panorama.md)。

---

## 8. 生产场景示例

**场景 A — 双 Telegram bot：**  
`profile create bot-a` / `bot-b` → 各 `.env` 不同 token → 各 `gateway start` + systemd。

**场景 B — OpenRouter 三 key：**  
`strategy: round_robin` → 429 时 `mark_exhausted_and_rotate` → 自动 key2。

**场景 C — 安全隔离：**  
不可信 cron job → 单独 profile + 收紧 toolsets [20](./20-security-defense-layers.md)。

---

## 9. 源码带读

1. `profiles.py` module docstring + `_PROFILE_DIRS`  
2. `CredentialPool._select_unlocked`  
3. `mark_exhausted_and_rotate`  
4. `read_credential_pool` / gateway 构造 Agent 处  

---

## 10. 自测

- [ ] default vs named 路径？  
- [ ] `home/` 防什么 bleed？  
- [ ] 四种 pool strategy 差异？  
- [ ] 401 vs 429 cooldown？  
- [ ] pool rotate vs runtime fallback？  
- [ ] 两 profile 两 gateway 是否共享 SessionDB？  

**关联：** [01 架构](./01-architecture-overview.md) · [15 Provider](./15-provider-and-transport.md) · [03 入口](./03-cli-gateway-and-entry.md) · [20 Security](./20-security-defense-layers.md)
