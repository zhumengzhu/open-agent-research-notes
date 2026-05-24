# 00 · 理念、定位与阅读边界

> **基准：** pin [`889903f`](https://github.com/NousResearch/hermes-agent/commit/889903f0fa4ba125c56acf21030b5b99c99778db)  
> **本笔记状态：** 内核脊柱篇已按源码加厚（05–06、08–11、13–15、17–20 等）；专篇合并见 [README](./README.md#专篇合格标准与合并策略)。

---

## 1. Hermes 是什么

**Hermes Agent**（Nous Research, MIT）是面向 **长期运行、自进化** 的 coding/general agent：

| 能力 | 源码锚点 |
|------|----------|
| 双入口 CLI + Gateway | `hermes_cli/main.py` · `gateway/run.py` [03](./03-cli-gateway-and-entry.md) |
| 开放模型 + pool + fallback | `credential_pool.py` · `runtime_provider.py` [15](./15-provider-and-transport.md) |
| 学习闭环 Skills/Memory | `skills_tool.py` · `memory_manager.py` [09](./09-skills-curator-and-learning-loop.md) |
| 七类 terminal + delegate | `tools/environments/` · `delegate_tool.py` [17](./17-terminal-backends.md) · [11](./11-delegation-cron-and-kanban.md) |
| 插件 + MCP | `hermes_cli/plugins.py` · `mcp_tool.py` [16](./16-plugins-mcp-and-hooks.md) |
| 多 Profile 隔离 | `profiles.py` · `get_hermes_home()` [21](./21-profiles-and-credential-pool.md) |

**与 Claude Code：** 开源 Python 单体、Gateway 一等公民、无 Bridge SDK。  
**与 OpenCode：** 非 Effect plugin host — **自带完整产品与消息网关**；对照见 [Claude Code](../claude-code/README.md) / [OpenCode](../opencode/README.md)。

---

## 2. 本笔记读什么

### 2.1 合格专篇标准（README 政策）

一篇合格需满足 **4 项中 3 项**：

1. **源码锚点** — `file:line` 或函数名  
2. **Happy path** — 正常时序  
3. **边界/失败** — interrupt、429、block、fail-open…  
4. **模块分界** — 与相邻篇的接口  

篇幅不是 KPI；**言之有物** 优先于篇数。

### 2.2 读 / 不读

| 读 | 不读（另有文档） |
|----|------------------|
| `conversation_loop.py` · `run_agent.py` | 每个 skill 的 SKILL.md 正文 |
| `model_tools.py` · `registry.py` · `toolsets.py` | website 用户教程全文 |
| `gateway/run.py` 生命周期 | 各平台 Bot 注册截图步骤 |
| `hermes_cli/` 命令与 config | install.sh 细节 |

官方 **[AGENTS.md](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md)** = 贡献者手册；本笔记 = **架构学习地图 + 带读**。

---

## 3. 架构立场

### 3.1 Agent 最小闭环（加厚版）

```text
用户/slash/Gateway MessageEvent
  → AIAgent.run_conversation
  → conversation_loop.run_conversation
      prefetch memory（一次/turn）
      pre_llm_call hooks → user context
      while iteration:
        interrupt / budget / compression preflight
        messages + get_tool_definitions
        chat.completions (stream)
        tool_calls → handle_function_call
          → pre_tool_call block?
          → invoke_tool → registry.dispatch
        无 tools → finalize + post-turn（curator / goals / background_review）
  → SessionDB + MemoryManager 持久化
```

同步 Python loop；Gateway asyncio **包一层**，单 turn 内 loop **同步** [05](./05-aiagent-and-conversation-loop.md)。

### 3.2 四层优先级

| 层 | 目录 | 首读专篇 |
|----|------|----------|
| **L0 内核** | `agent/conversation_loop.py` | [05](./05-aiagent-and-conversation-loop.md) · [26](./26-main-chain-atlas.md) |
| **L1 工具/状态** | `model_tools.py` · `tools/` · `hermes_state.py` | [06](./06-tools-registry-and-model-tools.md) · [08](./08-session-and-memory.md) |
| **L2 产品壳** | `cli.py` · `hermes_cli/` · `ui-tui/` | [03](./03-cli-gateway-and-entry.md) |
| **L3 网关/调度** | `gateway/` · `cron/` · `plugins/kanban/` | [10](./10-gateway-platforms-and-sessions.md) · [11](./11-delegation-cron-and-kanban.md) |

**横切：** Prompt [13](./13-prompt-assembly-and-cache.md) · 压缩 [14](./14-context-compression.md) · Provider [15](./15-provider-and-transport.md) · 安全 [20](./20-security-defense-layers.md)。

**集成深读：** 浏览器/PTC/并行/LSP 等 → [22 手册](./22-integrations-handbook.md)（原 22–31、36 合并）。

### 3.3 硬政策（读代码时会反复遇到）

1. **Prompt cache 不可 mid-turn 破坏** — 不换 toolset、不重建 system、不 reload memory（compression 例外）[13](./13-prompt-assembly-and-cache.md)  
2. **Profile 安全** — 一律 `get_hermes_home()`，禁硬编码 `~/.hermes` [21](./21-profiles-and-credential-pool.md)  
3. **插件不改 core** — `PluginManager` hook / `register_tool` [16](./16-plugins-mcp-and-hooks.md)  
4. **Gateway 双 guard** — 活跃 session 排队 + runner 拦截 `/stop` 等（AGENTS.md Known Pitfalls）[10](./10-gateway-platforms-and-sessions.md)  
5. **Context 注入走 user** — 插件 `pre_llm_call` 不进 system [16 §6](./16-plugins-mcp-and-hooks.md#6-context-注入政策)

---

## 4. 推荐阅读顺序

| 阶段 | 篇章 | 时间 |
|------|------|------|
| 地图 | **00** → **26** → **01** | ~30 min |
| 内核 | **05** → **06** → **08** → **13** → **14** | ~3 h |
| 入口与网关 | **03** → **10** → **02** → **15** | ~2 h |
| 扩展 | **16** → **21** → **09** → **11** | ~2 h |
| 按需 | **17–20** · **22 手册** | 随用随读 |

完整路径：[learning-paths.md](./learning-paths.md)

---

## 5. 与相邻项目

| 项目 | 关系 |
|------|------|
| [Claude Code](../claude-code/README.md) | 闭源 TS agent loop（query.ts、compact） |
| [OpenCode](../opencode/README.md) | Effect runtime + Plugin.trigger |
| [oh-my-openagent](../oh-my-openagent/README.md) | OpenCode 插件层（非 Hermes 代码） |
| agentskills.io | Hermes skills 格式兼容 |

---

## 6. 读完应能回答

- [ ] 两个主入口及汇合点（`run_conversation`）？  
- [ ] Loop 本体在哪个文件（非 `run_agent` 大函数体）？  
- [ ] 工具 dispatch 公共 API（`handle_function_call` vs `invoke_tool`）？  
- [ ] Gateway 为何常 per-message 新建 `AIAgent`？  
- [ ] Profile 与 credential pool 区别？  
- [ ] 插件 context 为何不进 system prompt？  

**下一步：** [learning-paths 路径 G](./learning-paths.md#路径-g导航与交叉引用--约-30-min) · [99 索引](./99-glossary-and-reading-map.md)
