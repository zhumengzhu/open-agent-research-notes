# Hermes Agent 学习路径

> **基准：** pin [`889903f`](https://github.com/NousResearch/hermes-agent/commit/889903f0fa4ba125c56acf21030b5b99c99778db)  
> **目录：** [README § 概念目录](./README.md#概念目录全集) · **术语速查：** [99 索引](./99-glossary-and-reading-map.md)

---

## 路径 A：第一次学 Hermes ⏱ 约 4–6 小时

**适合：** 建立「它怎么跑起来」的整体感。

| 步骤 | 文档/动作 | 时长 |
|------|-----------|------|
| 1 | [00 定位 §3](./00-philosophy-and-positioning.md#3-架构立场) — 最小闭环 + 硬政策 | 15 min |
| 2 | **[26 总图 §1–§3](./26-main-chain-atlas.md#3-单-turn-内顺序与-05-对齐)** — 时序图 + 12 步 turn 序 | 20 min |
| 3 | [01 架构 §2–§5](./01-architecture-overview.md) | 30 min |
| 4 | [05 Loop §2–§5](./05-aiagent-and-conversation-loop.md) | 1.5 h |
| 5 | 带读 `conversation_loop.py`：`run_conversation` while + post-turn | 1.5 h |
| 6 | [03 双入口 skim §1–§5](./03-cli-gateway-and-entry.md) | 45 min |
| 7 | 跑 `hermes` + `hermes doctor` | 30 min |

**Checklist**

- [ ] 能画出：CLI/Gateway → `AIAgent` → `run_conversation` → `handle_function_call` → `registry.dispatch`
- [ ] 能说出 loop 三条主要退出条件（final text / budget / interrupt）
- [ ] 知道工具注册依赖链四层（import → register → toolset → resolve）
- [ ] 能复述 **12 步 turn 序**（prefetch → pre_llm_call → while → post-turn）— [26 §3](./26-main-chain-atlas.md#3-单-turn-内顺序与-05-对齐)
- [ ] 知道 mid-turn **禁止**换 toolset / 重建 system — [13 §2](./13-prompt-assembly-and-cache.md)

---

## 路径 B：内核深读 ⏱ 约 3–5 天

```
26 §3 turn 序 → 05 → conversation_loop 全文
  → [13 prompt/cache](./13-prompt-assembly-and-cache.md) → [14 压缩](./14-context-compression.md)
  → [06 registry](./06-tools-registry-and-model-tools.md) → [07 toolsets](./07-toolsets-and-platform-bundles.md)
  → [15 provider + pool](./15-provider-and-transport.md) → [21 §5 pool rotate](./21-profiles-and-credential-pool.md#5-credentialpool-机制)
  → [08 session/memory](./08-session-and-memory.md)
  → [10 gateway cache + FIFO](./10-gateway-platforms-and-sessions.md)
  → [16 §6 context 注入政策](./16-plugins-mcp-and-hooks.md#6-context-注入政策)（与 13 交叉）
```

| 模块 | 文件 | 笔记 |
|------|------|------|
| Loop | `agent/conversation_loop.py` | [05](./05-aiagent-and-conversation-loop.md) · [26 §3](./26-main-chain-atlas.md#3-单-turn-内顺序与-05-对齐) |
| Prompt | `agent/prompt_builder.py`, `agent/system_prompt.py` | [13](./13-prompt-assembly-and-cache.md) |
| 压缩 | `agent/context_compressor.py` | [14](./14-context-compression.md) |
| 工具 | `model_tools.py`, `tools/registry.py` | [06](./06-tools-registry-and-model-tools.md) |
| Toolsets | `toolsets.py` | [07](./07-toolsets-and-platform-bundles.md) |
| Provider | `hermes_cli/runtime_provider.py` | [15](./15-provider-and-transport.md) |
| Pool | `agent/credential_pool.py` | [21 §5](./21-profiles-and-credential-pool.md#5-credentialpool-机制) |
| 会话 | `hermes_state.py` | [08](./08-session-and-memory.md) |
| Memory | `agent/memory_manager.py` | [08](./08-session-and-memory.md) |
| Gateway | `gateway/run.py`, `gateway/platforms/base.py` | [10](./10-gateway-platforms-and-sessions.md) |
| Hooks | `hermes_cli/plugins.py` | [16 §4–§6](./16-plugins-mcp-and-hooks.md) |

**Checklist**

- [ ] 读过 `_restore_or_build_system_prompt` 四态注释
- [ ] 读过 `handle_function_call` vs `invoke_tool` + `skip_pre_tool_call_hook` — [06 §6](./06-tools-registry-and-model-tools.md)
- [ ] 读过 Gateway `_AGENT_CACHE_*` 与 interrupt 路径 — [10 §4](./10-gateway-platforms-and-sessions.md)
- [ ] 能区分 **pool rotate**（同 provider 换 key）与 **runtime fallback**（换 provider 链）— [21 §5.4](./21-profiles-and-credential-pool.md#54-与-runtime-fallback-关系)

---

## 路径 C：插件与工具 ⏱ 约 1 天

**官方：** [AGENTS.md Adding New Tools](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md#adding-new-tools) · [Plugins](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md#plugins)

```
[16 §1–§5](./16-plugins-mcp-and-hooks.md) — 发现、PluginContext、Hook 序、pre_tool_call block
  → [06 Registry §5–§6](./06-tools-registry-and-model-tools.md) — generation、双执行路径
  → [07 Toolsets](./07-toolsets-and-platform-bundles.md) — core vs webhook safe
[16 §8 MCP](./16-plugins-mcp-and-hooks.md#8-mcp-集成) — list_changed refresh、_build_safe_env
插件 ~/.hermes/plugins/ → manifest + plugins.enabled opt-in
[17 Terminal backends](./17-terminal-backends.md)（若改 terminal 行为）
```

| 深读锚点 | 文件:概念 |
|----------|-----------|
| Block tool | `plugins.py` `get_pre_tool_call_block_message` |
| Context 注入 | `invoke_hook("pre_llm_call")` → user message only |
| MCP refresh | `mcp_tool._refresh_tools` → selective `deregister` |
| Plugin tool | `PluginContext.register_tool` → `registry.register` |

**Checklist**

- [ ] 能说明 plugin tool vs core tool 路由（同一 `registry.dispatch`）
- [ ] `pre_tool_call` **block** vs `pre_approval_request` **仅观测** — [16 §5](./16-plugins-mcp-and-hooks.md#5-pre_tool_call-block深读)
- [ ] MCP refresh 为何不全量 nuke registry — [16 §8.2](./16-plugins-mcp-and-hooks.md#82-动态-refreshtoolslist_changed)
- [ ] 能添加一个 `CommandDef` 且 **Gateway handler 不遗漏** — [16 §10](./16-plugins-mcp-and-hooks.md#10-gateway-slash-扩展)
- [ ] stdio MCP env 过滤与 [20 §1](./20-security-defense-layers.md) 对应

---

## 路径 D：Gateway 与消息平台 ⏱ 约 1 天

```
03 §5 Gateway → [10 平台与会话 §4–§9](./10-gateway-platforms-and-sessions.md)
  → [16 §4.2 pre_gateway_dispatch](./16-plugins-mcp-and-hooks.md#42-valid_hooks128168-行节选)（入站 skip/rewrite）
  → gateway/platforms/ 选一平台 deep skim
```

**Checklist**

- [ ] 双 guard 各自拦截什么？（活跃 session 排队 + runner `/stop`）
- [ ] `terminal.cwd` vs CLI cwd
- [ ] FIFO vs pending session — [10 §6–§9](./10-gateway-platforms-and-sessions.md)
- [ ] Gateway per-message Agent 与 pool stale — [21 §5.5](./21-profiles-and-credential-pool.md#55-gateway-stale-边界)
- [ ] background terminal notify 配置键

---

## 路径 E：Multi-Agent 与运维 ⏱ 约 0.5–1 天

```
[11 delegate/cron/kanban](./11-delegation-cron-and-kanban.md)
  → [18 全景 MoA+review](./18-multi-agent-panorama.md)
  → [19 /goal](./19-goals-and-ralph-loop.md)
  → [20 安全纵深](./20-security-defense-layers.md)
  → [21 profile/pool §4–§8](./21-profiles-and-credential-pool.md) — 多 bot、strategy、生产场景
```

**Checklist**

- [ ] 五种 multi-agent 机制触发点与是否 block 父 loop
- [ ] background_review 与 curator 分工 — [09](./09-skills-curator-and-learning-loop.md) · [18](./18-multi-agent-panorama.md)
- [ ] Goal judge fail-open 含义 — [19](./19-goals-and-ralph-loop.md)
- [ ] Profile vs credential pool vs session_id 三维隔离 — [21 §1](./21-profiles-and-credential-pool.md#1-概念对照)
- [ ] 双 Telegram bot：两 profile + 两 gateway 进程 — [21 §4](./21-profiles-and-credential-pool.md#4-多-gateway--多-bot-模式)

---

## 路径 F：集成子系统 ⏱ 按需

**适合：** Browser、PTC、Voice 等可选能力；**一本读完** 比扫 15 篇 redirect 更高效。

```
[22 集成手册](./22-integrations-handbook.md) §1–§10（Browser → Kanban）
  需要 Gateway 队列细节 → [10 §6–§9](./10-gateway-platforms-and-sessions.md)
  需要 Skills↔Memory → [09 §10](./09-skills-curator-and-learning-loop.md#10-skills-与-memory-协作)
  需要插件并行 MCP → [16 §8](./16-plugins-mcp-and-hooks.md#8-mcp-集成) + [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails)
```

**Checklist**

- [ ] 能说明 `execute_tool_calls_concurrent` interrupt preflight — [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails)
- [ ] 能说明 PTC 与连续 tool_call 的 context 差异 — [22 §2](./22-integrations-handbook.md#2-ptc-沙箱execute_code)
- [ ] 能说明 Gateway 为何不 expand `@file` — [22 §4](./22-integrations-handbook.md#4-context-引用file--附件)

---

## 路径 G：导航与交叉引用 ⏱ 约 30 min

**适合：** 已读过脊柱，需要 **查概念落点** 或对照 Wiki。

| 步骤 | 文档 |
|------|------|
| 1 | [99 §1 概念表](./99-glossary-and-reading-map.md#1-概念--章节精选) |
| 2 | [99 §2 模块表](./99-glossary-and-reading-map.md#2-源码模块--章节) — 注意 23–36 已 redirect 到 22 |
| 3 | [12 Wiki 映射](./12-coverage-gaps-and-external-resources.md#2-wiki-概念--本笔记映射完整) |
| 4 | [26 §6 API 速查](./26-main-chain-atlas.md#6-关键-api-速查) |

---

**推荐起手：** `00 → 26 → 05 → 01 → 02` · 写插件前加 `16` · 部署多 bot 前加 `21`

---

## 对照阅读（可选）

| 主题 | Hermes | Claude Code |
|------|--------|-------------|
| Loop | [05](./05-aiagent-and-conversation-loop.md) · [26](./26-main-chain-atlas.md) | [../claude-code/06](../claude-code/06-query-agent-loop.md) |
| 网关 | [03](./03-cli-gateway-and-entry.md) · [10](./10-gateway-platforms-and-sessions.md) | [../claude-code/18](../claude-code/18-bridge-and-ide.md) |
| 子 agent | [11](./11-delegation-cron-and-kanban.md) | [../claude-code/20](../claude-code/20-agents-and-subagents.md) |
| 工具 | [06](./06-tools-registry-and-model-tools.md) | [../claude-code/07](../claude-code/07-tools-and-permissions.md) |
| Memory | [08](./08-session-and-memory.md) | [../claude-code/29](../claude-code/29-memory-and-auto-memory.md) |
| 插件/Hook | [16](./16-plugins-mcp-and-hooks.md) | [../claude-code/](../claude-code/README.md) hooks 篇 |

→ [README](./README.md) · [99 索引](./99-glossary-and-reading-map.md)
