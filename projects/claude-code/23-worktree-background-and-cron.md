# 23 · Worktree、Background 与 Cron

> **锚点：** `utils/worktree.ts` · `setup.ts` · `ScheduleCronTool` · `tasks/` · `SleepTool` · `bridge/bridgeMain.ts`

---

## 1. Git Worktree

### 1.1 Tools

| Tool | 行为 |
|------|------|
| `EnterWorktreeTool` | 新建 worktree、切 cwd、`saveWorktreeState` |
| `ExitWorktreeTool` | 合并/丢弃、恢复主 cwd |

### 1.2 Session 级 setup

`setup.ts` 可选：

- `createWorktreeForSession` — 每 session 隔离分支目录
- `createTmuxSessionForWorktree` — tmux 绑定 [21]

`saveWorktreeState` → session storage → resume 恢复 [08]。

Bridge worker spawn 也可能 `createAgentWorktree` [18]。

### 1.3 与 Plan mode

Plan 在主仓读代码、写 plan；execute 可在 worktree 实施 [17] — 减少主分支污染。

Hooks：`WorktreeCreate` · `WorktreeRemove` [11]。

---

## 2. Background 任务架构

```text
主 thread (querySource: repl_main_thread)
  ├─ 用户继续输入 → 新 submitMessage（并行）
  └─ background task
       ├─ LocalShellTask — bash 输出流
       ├─ LocalAgentTask — 独立 query loop
       ├─ RemoteAgentTask — CCR worker [22]
       ├─ InProcessTeammateTask — swarm [21]
       └─ DreamTask — autoDream [29]
```

### 2.1 隔离维度

| 维度 | 主 thread | background |
|------|-----------|------------|
| `querySource` | `repl_main_thread` | `agent:*` / `bg:*` / teammate |
| Permission UI | REPL 对话框 | `shouldAvoidPermissionPrompts` → 常 auto-deny |
| mutableMessages | 主历史 | 独立 / sidechain |
| cached microcompact | 可用 | **禁用**（防污染）[10] |
| CacheSafeParams | stopHooks 写入 | fork 可读，不 write 主槽 [20] |

### 2.2 并发与等待

- Task manager 层 concurrency 限制（按实现 `tasks/`）
- `TaskOutputTool` / `TaskStopTool` — 主 thread 读/停后台 [21]
- `print.ts` shutdown：`getRunningTasks` dump；`runHeadlessStreaming` 可 **wait for agents** 再 exit

---

## 3. 与 Loop 门控 [28]

Background agent loop **与主 thread 并行**：

- 主 thread **不阻塞**等子 agent（除非 TaskOutput/wait）
- 子 agent 无 REPL permission → 未批 tool 可能 deny
- Sleep 结束 → **新 turn**，非 continue 当前主 loop iteration

---

## 4. Cron / 定时触发

Feature `AGENT_TRIGGERS`：

| Tool | 作用 |
|------|------|
| `ScheduleCronTool` | 注册 cron |
| `CronCreate` / `CronDelete` / `CronList` | CRUD |
| `RemoteTriggerTool` | 远程触发 |

```text
Cron _fire
  → synthetic user message 或 dispatch
  → submitMessage / 专用 entry
  → 完整 query loop（独立 turn）
```

与 KAIROS/proactive 产品线交汇 [30]。

---

## 5. Sleep 与 Proactive 唤醒

Feature `PROACTIVE` / `KAIROS` — **`SleepTool`**：

| 维度 | 行为 |
|------|------|
| 模型侧 | 声明等待 N 分钟 / 条件 |
| Loop | 结束 **当前 iteration**；session 仍 alive |
| Wake | cron / 外部事件 / 用户 message → **新 turn** |
| Cache | wake 间隔 vs prompt cache ~5min TTL 权衡（prompts.ts 注释） |

Sleep ≠ permission ask — **主动暂停 agent**，非等人点 allow [28]。

Proactive 流程：

```text
模型完成 → Sleep(duration)
  → session Idle
  → (时间到) trigger
  → prefetch memory [29] + 续任务
```

Coordinator mode 下 proactive 通常 **关闭** [30]。

---

## 7. Concurrency 与 TaskOutput

主 thread 与 background **并行** submitMessage：

- Task manager 限制同 provider/model 并发（概念同 OmO background_task）
- `TaskOutputTool` 阻塞读某一 task 输出（主 thread 等待 **该 task**，非全局）
- `TaskStopTool` 取消

Headless exit：`runHeadlessStreaming` 可选 wait all background → 防 CI 丢结果。

---

## 8. Worktree 与 Team

- Session `--worktree`：setup 时创建，非必须 EnterWorktree tool
- Swarm teammate：per-member worktree [21]
- Bridge worker spawn：也可能 `createAgentWorktree` [18]

---

## 9. AgentTool vs Background Task

| | AgentTool [20] | Background Task |
|---|----------------|-----------------|
| 触发 | 主 loop tool_use | TaskCreate / spawn API |
| 结果回传 | tool_result 到 **父 loop** | TaskOutput / 独立 transcript |
| 寿命 | 单次 delegate 常见 | 可长期运行 |
| Cache | 常 share CacheSafeParams | 视 querySource |

Subagent 完成 → 父 `needsFollowUp` → continue [28]。

Swarm spawn 可能 per-teammate worktree（`utils/worktree.ts` + `utils/swarm/`）；leader 与 member cwd 隔离 [21]。

---

## 10. 自测

- [ ] worktree 在 setup 还是 EnterWorktree tool？
- [ ] background 与主 thread permission 差异？
- [ ] headless 如何等 background 结束？
- [ ] Sleep vs permission ask vs L3 门控区别？
- [ ] cron 触发新 turn 还是 continue 当前 loop？
- [ ] 为何 background 不走 cached microcompact？

**关联：** [21 Tasks/Team](./21-tasks-team-and-coordinator.md) · [20 Agents](./20-agents-and-subagents.md) · [28 Loop 门控](./28-agent-loop-continuation-and-human-gates.md) · [29 Dream](./29-memory-and-auto-memory.md)
