# 17 · Terminal Backends（执行环境）

> **锚点：** `tools/environments/base.py` · `tools/terminal_tool.py`（`_create_environment` ~1106）· `tools/interrupt.py`

**外部（A）：** [Wiki terminal-backends](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/terminal-backends.md) · [官方 Tools Runtime § Terminal](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime)

同一 **`terminal` / `process` tool schema**，背后可切 7+ 执行后端 — Hermes 相对「本机 bash only」agent 的显著差异。

---

## 1. 统一模型：spawn-per-call

`BaseEnvironment` 模块头（1–6 行）：

```1:6:/Users/zmz/Github/hermes-agent/tools/environments/base.py
"""Unified spawn-per-call model: every command spawns a fresh ``bash -c`` process.
Session snapshot (env, functions, aliases) captured once at init, re-sourced before each command.
CWD persists via stdout markers (remote) or temp file (local)."""
```

| 对比 | spawn-per-call | 持久交互 shell |
|------|----------------|----------------|
| 进程 | 每条命令新 `bash -c` | 单 PTY 长连接 |
| 状态 | snapshot re-source | shell 内存态 |
| Hermes | **默认 terminal** | tmux/interactive 另路径 |

**`task_id`：** 隔离并发 delegate / 多 session 的环境实例与 snapshot key [11](./11-delegation-cron-and-kanban.md)。

---

## 2. 后端清单与选型

`_create_environment(env_type, ...)`（1106–1179+ 行）分支：

| env_type | 实现 | 典型场景 |
|----------|------|----------|
| **local** | `_LocalEnvironment` | 默认；本机 cwd |
| **docker** | `_DockerEnvironment` | 容器隔离；可选 mount cwd、forward env |
| **ssh** | `_SSHEnvironment` | 远程 dev 机 |
| **singularity** | `_SingularityEnvironment` | HPC |
| **modal** | `_ModalEnvironment` / `_ManagedModalEnvironment` | serverless；见 §3 |
| **daytona** | `_DaytonaEnvironment` | 云 dev env，可休眠 |
| **vercel_sandbox** | Vercel 沙箱 | 短时云命令 |

**Gate：** `check_terminal_requirements()` — backend 不可用则 **terminal 不进 schema** [07](./07-toolsets-and-platform-bundles.md)。

**配置：**

```yaml
terminal:
  env: docker          # → env_type
  cwd: /path/to/project
  image: ...
  container_cpu/memory/disk: ...
```

环境 override：`TERMINAL_ENV`、`TERMINAL_CWD`（Gateway **必须** 指向用户项目 [13](./13-prompt-assembly-and-cache.md)）。

---

## 3. Modal：direct vs managed

`terminal_tool._get_modal_backend_state(modal_mode)`：

- **direct** — 用户 Modal 凭证，本地选 backend  
- **managed** — Nous managed tool gateway（`is_managed_tool_gateway_ready("modal")`）

选错 → `_create_environment` 抛清晰错误或 fallback 提示 — 读 config `terminal.container_config.modal_mode`。

---

## 4. 工厂调用链

```text
model 调用 terminal tool
  → terminal_tool(..., task_id=effective_task_id)
  → _get_or_create_env(task_id)  # 同 task 复用环境实例
  → _create_environment(env_type, image, cwd, timeout, ssh_config, container_config, ...)
  → BaseEnvironment.run_command / execute
  → stdout/stderr JSON 回 model
```

**远程同步：** `environments/file_sync.py` — delegate / 云 backend 与本地 workspace 同步。

**持久 env：** `is_persistent_env()` — 部分 backend 跨命令保留容器（与 spawn-per-call 语义共存：仍是新 bash -c，但 filesystem 持久）。

---

## 5. 中断与 activity

### 5.1 Interrupt

```text
用户 interrupt [05 §3.1]
  → tools/interrupt.is_interrupted()
  → BaseEnvironment._wait_for_process 退出 poll
  → 终止 subprocess；返回 partial/错误 JSON
```

`HERMES_DEBUG_INTERRUPT=1` — 打 poll 循环诊断（base.py 28–39 行）。

**与 loop：** `agent._execution_thread_id` 绑定 interrupt 作用域（conversation_loop 591–605 行）— 只杀 **当前 turn** 的 terminal，不误伤其它 session。

### 5.2 Activity callback

```41:48:/Users/zmz/Github/hermes-agent/tools/environments/base.py
# Thread-local activity callback — long _wait_for_process loops report liveness to gateway.
set_activity_callback(cb)
```

Gateway 长命令（>10s 默认间隔）上报「terminal running 120s」— 用户知 agent 未挂死 [10](./10-gateway-platforms-and-sessions.md)。

---

## 6. 审批与安全

```text
terminal 命令
  → dangerous pattern 匹配
  → approvals.mode: manual | smart | off
  → CLI: prompt_toolkit TLS callback
  → Gateway: tools/approval.py per-session 队列 [10]
  → Subagent: auto-deny（无 TLS）[11 §2.4](./11-delegation-cron-and-kanban.md)
  → [可选] Tirith 语义扫描 [20](./20-security-defense-layers.md)
```

**Checkpoint：** mutating 命令前 `CheckpointManager.ensure_checkpoint` [22 §8](./22-integrations-handbook.md#8-worktree-与-checkpoint)。

---

## 7. 与 execute_code

| | terminal | execute_code |
|---|----------|--------------|
| 模型可见 | 单次命令输出 | 脚本 stdout 摘要 |
| 隔离 | BaseEnvironment | UDS / file RPC [22 §2](./22-integrations-handbook.md#2-ptc-沙箱execute_code) |
| 子 agent | 可用（TLS 限制） | **blocked** [11](./11-delegation-cron-and-kanban.md) |

---

## 8. 失败路径示例

| 失败 | 表现 |
|------|------|
| Docker 未装 | check_fn false → 无 terminal tool |
| SSH 断连 | handler 异常 → `{"error":...}` |
| 超时 | timeout 杀进程；interrupt 优先于 hang |
| 审批 deny | JSON error；子 agent 默认 deny |
| cwd 不存在 | backend 创建或报错 — 依 env_type |

---

## 9. 与 Claude Code 对照

| | Hermes | Claude Code |
|---|--------|-------------|
| 抽象 | BaseEnvironment 多 backend | 主 local + sandbox 策略 |
| 云 | Modal/Daytona/Vercel 内置 | 较少 |
| 隔离维度 | task_id + backend + profile HOME [21](./21-profiles-and-credential-pool.md) | seatbelt / permission |

---

## 10. 源码带读

1. `base.py` `_wait_for_process` + interrupt  
2. `terminal_tool._create_environment` 全分支  
3. `terminal_tool.terminal_tool` approval 钩子  
4. `delegate_tool` subagent approval initializer  

---

## 11. 自测

- [ ] spawn-per-call 与持久 shell 区别？  
- [ ] Gateway TERMINAL_CWD 为何必须？  
- [ ] task_id 隔离什么？  
- [ ] modal managed vs direct 配置键？  
- [ ] activity callback 解决什么 UX 问题？  
- [ ] interrupt 如何传到 Docker 内进程？  

**关联：** [06 Tools](./06-tools-registry-and-model-tools.md) · [07 Toolsets](./07-toolsets-and-platform-bundles.md) · [11 Delegate](./11-delegation-cron-and-kanban.md) · [20 Security](./20-security-defense-layers.md)
