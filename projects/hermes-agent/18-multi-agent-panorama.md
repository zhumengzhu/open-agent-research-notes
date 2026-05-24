# 18 · Multi-Agent 全景

> **锚点：** `tools/delegate_tool.py` · `tools/mixture_of_agents_tool.py` · `agent/background_review.py` · `agent/curator.py`  
> **Wiki：** [multi-agent-architecture](https://github.com/cclank/Hermes-Wiki/blob/master/concepts/multi-agent-architecture.md)

[11](./11-delegation-cron-and-kanban.md) 覆盖 delegate/cron/kanban **进程级** 子 Agent；本篇覆盖 **同 turn / turn 后** 协作机制及 **谁改主 session**。

---

## 1. 五种机制总表

| 机制 | 触发 | 改主 session messages? | 改 MEMORY/skills? | 阻塞父 loop? |
|------|------|------------------------|-------------------|--------------|
| **delegate_task** | model tool | 仅 +1 tool result | 子 agent 禁 memory | **是** |
| **mixture_of_agents** | model tool | +MoA call/result | 否 | **是** |
| **background_review** | turn 末 nudge | **否** | **是**（fork 写盘） | 否（daemon） |
| **send_message** | model tool | +tool result | 否 | 否（单次 tool） |
| **kanban worker** | 任务板 | 主 board 会话 | 按 worker | 否（独立 session） |

**Profile** [21](./21-profiles-and-credential-pool.md) = 部署隔离，非单 session fork。

---

## 2. delegate_task（复习）

```text
invoke_tool("delegate_task")  # 非 registry 直 dispatch
  → ThreadPoolExecutor + subagent approval TLS
  → 子 AIAgent.run_conversation（空 history）
  → DELEGATE_BLOCKED_TOOLS 剥离 [11 §2.2](./11-delegation-cron-and-kanban.md#22-delegate_blocked_tools)
```

与 CC：Hermes **默认同步 block** 父 while。

---

## 3. mixture_of_agents（MoA）

**文件：** `tools/mixture_of_agents_tool.py` · toolset **`moa`**（默认常 disabled [07](./07-toolsets-and-platform-bundles.md)）

```text
mixture_of_agents(prompt, ...)
  → 并行 reference models（多 frontier via OpenRouter 等）
  → aggregator 模型合成
  → 单条 JSON tool result 回父 loop
```

| 对比 delegate | MoA |
|---------------|-----|
| 路由 | `registry.dispatch` | `invoke_tool` **无** 特殊分支 |
| 中间步骤 | 父不可见 | 父 **保留** MoA 调用与结果 |
| 成本 | 单模型子 loop | 多模型并行 |

适合极难推理；非日常默认。

---

## 4. background_review（学习闭环核心）

**模块头（1–16 行）：**

```1:16:/Users/zmz/Github/hermes-agent/agent/background_review.py
"""After every turn, run_conversation may call spawn_background_review ...
Writes go straight to memory + skill stores. Main conversation and prompt cache are never touched.
The fork inherits ... cached system prompt ... same prefix cache."""
```

### 4.1 触发（conversation_loop 4177–4202）

```text
turn 末：
  memory nudge 命中（_turns_since_memory >= interval）→ review_memory
  skill nudge 命中（_iters_since_skill >= interval）→ review_skills
  且 final_response 存在、未 interrupt
  → agent._spawn_background_review(messages_snapshot, ...)
```

**与 curator 区别：** review = **每 turn 可能** 的轻量 fork；curator = idle 7d 级 archive [09 §5](./09-skills-curator-and-learning-loop.md)。

### 4.2 Fork 行为

- **继承：** provider、model、base_url、**`_cached_system_prompt` verbatim** — prefix cache 命中  
- **工具白名单：** memory、skill_manage 等；其它 deny  
- **Prompt 分轨：** `_MEMORY_REVIEW_PROMPT` vs `_SKILL_REVIEW_PROMPT`（34–80 行）— skill 侧 **主动** 更新 library  
- **Provenance：** `skill_provenance.BACKGROUND_REVIEW` — 仅 review fork 可 autonomous 改 agent-created skills  

### 4.3 硬不变量

- **不** append 主 session messages  
- **不** invalidate 主 Agent system cache  
- 失败 best-effort — 不影响用户已收到的回复  

---

## 5. send_message 与 Kanban

- **send_message：** Gateway `check_fn`；跨平台副作用；delegate **禁止** [11](./11-delegation-cron-and-kanban.md)  
- **Kanban：** 持久队列 + worker Agent；见 [11 §4](./11-delegation-cron-and-kanban.md#4-kanban) · [22 §10](./22-integrations-handbook.md#10-kanban-板内部)

---

## 6. 调度关系图

```text
用户 turn ──主 AIAgent─┬─ delegate_task（同步子 loop，隐藏中间步）
                      ├─ mixture_of_agents（多模型 tool，保留 MoA 结果）
                      ├─ send_message / 普通 tools
                      │
                      └─ turn 结束 ──→ background_review（异步 fork → MEMORY/skills）
                                    └─→ curator（idle，auxiliary）[09]
Cron / Kanban ── 不在用户 turn 内 ── [11]
```

---

## 7. 与 Claude Code / OpenCode 对照

| | Hermes | Claude Code | OpenCode |
|---|--------|-------------|----------|
| 子 agent | delegate 同步 | AgentTool / bg | task delegate |
| 多模型 | MoA tool | 较少内置 | provider 切换 |
| 自改进 | background_review | auto-memory 形态不同 | skills 模块 |
| 团队 | kanban 内置 | — | team-mode mailbox |

---

## 8. 源码带读

1. `conversation_loop` 4192–4202 spawn 条件  
2. `background_review.spawn_background_review_thread`  
3. `mixture_of_agents_tool.py` 入口  
4. `delegate_tool` vs MoA 注册方式对比  

---

## 9. 自测

- [ ] 五种机制哪些改主 session transcript？
- [ ] review 为何继承 cached system？
- [ ] MoA 与 delegate 在 dispatch 路径上的区别？
- [ ] memory nudge vs skill nudge 计数器差异？
- [ ] review 失败是否影响已发送回复？
- [ ] curator 与 review 触发频率差？

**关联：** [11 Delegation](./11-delegation-cron-and-kanban.md) · [09 Skills](./09-skills-curator-and-learning-loop.md) · [13 Cache](./13-prompt-assembly-and-cache.md) · [05 §6 Post-turn](./05-aiagent-and-conversation-loop.md#6-post-turn-与-return-dict)
