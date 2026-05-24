# 99 · 术语表与阅读地图

> **路径：** [learning-paths.md](./learning-paths.md) · **总图：** [26 Atlas](./26-main-chain-atlas.md) · **Wiki 对照：** [12](./12-coverage-gaps-and-external-resources.md)

---

## 1. 概念 → 章节（精选）

| 概念 | 章节 |
|------|------|
| `run_conversation` / conversation_loop | [05](./05-aiagent-and-conversation-loop.md) · [26 §3](./26-main-chain-atlas.md#3-单-turn-内顺序与-05-对齐) |
| prefetch memory（每 turn 一次） | [05](./05-aiagent-and-conversation-loop.md) · [08](./08-session-and-memory.md) |
| `handle_function_call` / `invoke_tool` | [06 §6](./06-tools-registry-and-model-tools.md) |
| registry / `get_tool_definitions` / generation | [06 §5](./06-tools-registry-and-model-tools.md) |
| toolsets / webhook safe | [07](./07-toolsets-and-platform-bundles.md) |
| SessionDB / MemoryManager / compression lineage | [08](./08-session-and-memory.md) |
| skills / curator / `should_run_now` | [09](./09-skills-curator-and-learning-loop.md) |
| skills ↔ memory | [09 §10](./09-skills-curator-and-learning-loop.md#10-skills-与-memory-协作) |
| GatewayRunner / FIFO / pending | [10](./10-gateway-platforms-and-sessions.md) |
| `_AGENT_CACHE_*` | [10 §4](./10-gateway-platforms-and-sessions.md) · [03 §5.1](./03-cli-gateway-and-entry.md) |
| delegate / cron / kanban | [11](./11-delegation-cron-and-kanban.md) · [22 §10](./22-integrations-handbook.md#10-kanban-板内部) |
| config / `iteration_budget` | [02](./02-config-iteration-and-model-routing.md) |
| prompt cache / system tiers / mid-turn 政策 | [13](./13-prompt-assembly-and-cache.md) |
| ContextCompressor / preflight | [14](./14-context-compression.md) |
| `api_mode` / ProviderProfile / runtime fallback | [15](./15-provider-and-transport.md) |
| auxiliary 侧任务 | [15 §6](./15-provider-and-transport.md#6-auxiliary-侧任务路由) |
| PluginManager / `VALID_HOOKS` | [16 §4](./16-plugins-mcp-and-hooks.md#4-hook-全表与-fire-顺序单-turn) |
| `pre_tool_call` block | [16 §5](./16-plugins-mcp-and-hooks.md#5-pre_tool_call-block深读) |
| `pre_llm_call` context 注入 | [16 §6](./16-plugins-mcp-and-hooks.md#6-context-注入政策) |
| MCP / `list_changed` refresh | [16 §8](./16-plugins-mcp-and-hooks.md#8-mcp-集成) |
| `pre_gateway_dispatch` | [16 §4.2](./16-plugins-mcp-and-hooks.md) · [10](./10-gateway-platforms-and-sessions.md) |
| BaseEnvironment / `_create_environment` | [17](./17-terminal-backends.md) |
| MoA / `background_review` | [18](./18-multi-agent-panorama.md) |
| GoalManager / fail-open judge | [19](./19-goals-and-ralph-loop.md) |
| Security layers / Tirith / approval | [20](./20-security-defense-layers.md) |
| Profile / `HERMES_HOME` / `home/` 隔离 | [21 §2–§4](./21-profiles-and-credential-pool.md) |
| CredentialPool / strategy / rotate | [21 §5](./21-profiles-and-credential-pool.md#5-credentialpool-机制) |
| interrupt / steer | [05 §3.1](./05-aiagent-and-conversation-loop.md#31-interrupt-传播) · [17](./17-terminal-backends.md) |
| browser / web / PTC / 并行 / LSP / Voice… | [22 集成手册](./22-integrations-handbook.md) |
| 合格专篇标准 | [README §文档质量](./README.md#文档质量原则言之有物) · [00 §2.1](./00-philosophy-and-positioning.md#21-合格专篇标准readme-政策) |

---

## 2. 源码模块 → 章节

| 模块 | 章节 |
|------|------|
| `agent/conversation_loop.py` | [05](./05-aiagent-and-conversation-loop.md) · [26 §4](./26-main-chain-atlas.md#4-文件锚点表) |
| `run_agent.py` (`AIAgent`) | [05](./05-aiagent-and-conversation-loop.md) |
| `agent/prompt_builder.py`, `agent/system_prompt.py` | [13](./13-prompt-assembly-and-cache.md) |
| `agent/context_compressor.py` | [14](./14-context-compression.md) |
| `agent/tool_executor.py`, `agent/tool_guardrails.py` | [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails) |
| `agent/context_references.py` | [22 §4](./22-integrations-handbook.md#4-context-引用file--附件) |
| `agent/background_review.py` | [18](./18-multi-agent-panorama.md) |
| `agent/curator.py` | [09](./09-skills-curator-and-learning-loop.md) |
| `agent/lsp/` | [22 §5](./22-integrations-handbook.md#5-lsp-集成) |
| `agent/credential_pool.py` | [21 §5](./21-profiles-and-credential-pool.md#5-credentialpool-机制) |
| `hermes_cli/runtime_provider.py` | [15](./15-provider-and-transport.md) |
| `hermes_cli/plugins.py` | [16](./16-plugins-mcp-and-hooks.md) |
| `hermes_cli/goals.py` | [19](./19-goals-and-ralph-loop.md) |
| `hermes_cli/profiles.py` | [21](./21-profiles-and-credential-pool.md) |
| `hermes_cli/config.py`, `hermes_cli/commands.py` | [02](./02-config-iteration-and-model-routing.md) · [03](./03-cli-gateway-and-entry.md) |
| `tools/registry.py`, `model_tools.py` | [06](./06-tools-registry-and-model-tools.md) |
| `tools/mcp_tool.py` | [16 §8](./16-plugins-mcp-and-hooks.md#8-mcp-集成) |
| `tools/code_execution_tool.py` | [22 §2](./22-integrations-handbook.md#2-ptc-沙箱execute_code) |
| `tools/browser_*.py`, `agent/web_search_registry.py` | [22 §1](./22-integrations-handbook.md#1-browser-与-web-工具) |
| `tools/environments/`, `tools/terminal_tool.py` | [17](./17-terminal-backends.md) |
| `tools/checkpoint_manager.py` | [22 §8](./22-integrations-handbook.md#8-worktree-与-checkpoint) |
| `tools/voice_mode.py` | [22 §6](./22-integrations-handbook.md#6-voice-与-tui-皮肤) |
| `gateway/run.py` | [10](./10-gateway-platforms-and-sessions.md) · [26 §4](./26-main-chain-atlas.md#4-文件锚点表) |
| `tui_gateway/server.py`, `ui-tui/` | [03](./03-cli-gateway-and-entry.md) · [22 §6](./22-integrations-handbook.md#6-voice-与-tui-皮肤) |
| `toolsets.py` | [07](./07-toolsets-and-platform-bundles.md) |
| `hermes_state.py` | [08](./08-session-and-memory.md) |

**Redirect 编号（勿当正文读）：** 23–31、32、33–36 → 见上表对应 § 或 [22 手册](./22-integrations-handbook.md#11-阅读顺序建议)。

---

## 3. 按目标选读

| 目标 | 路径 |
|------|------|
| 第一次学 | [路径 A](./learning-paths.md#路径-a第一次学-hermes--约-46-小时)：`00 → 26 → 05 → 01 → 02` |
| 内核深读 | [路径 B](./learning-paths.md#路径-b内核深读--约-35-天) |
| 插件/工具 | [路径 C](./learning-paths.md#路径-c插件与工具--约-1-天) |
| Gateway | [路径 D](./learning-paths.md#路径-dgateway-与消息平台--约-1-天) |
| Multi-agent/运维 | [路径 E](./learning-paths.md#路径-e-multi-agent-与运维--约-05-1-天) |
| 集成子系统 | [路径 F](./learning-paths.md#路径-f集成子系统--按需) |
| 查概念落点 | [路径 G](./learning-paths.md#路径-g导航与交叉引用--约-30-min) · 本页 §1 |
| 对照 Claude Code | [../claude-code/26](../claude-code/26-main-chain-atlas.md) |

---

## 4. 脊柱篇厚度速查（2026-05）

| 编号 | 主题 | ~行 | 核心锚点 |
|------|------|-----|----------|
| 00 | 定位 | 130+ | 合格标准、推荐阅读序 |
| 01 | 架构 | 170+ | L0 地图、单 turn 数据流 |
| 05 | Loop | 210+ | prefetch、steer、post-turn |
| 06 | Tools | 250+ | 双路径、generation |
| 08–11 | 会话/编排 | 160–220 | SessionDB、Gateway FIFO |
| 13–15 | Prompt/Provider | 160–195 | cache、compress、fallback |
| 16 | Plugins | 220+ | hooks、MCP refresh |
| 17–20 | Terminal/安全 | 145–180 | approval、纵深 |
| 21 | Profile/Pool | 190+ | 多 bot、strategy |
| 22 | 集成手册 | 220+ | Browser/PTC/并行… |
| 26 | Atlas | 155+ | 12 步 turn、API 速查 |

篇幅非 KPI；上表仅供 **选读优先级**，细节以各篇为准。

---

→ [README 概念目录](./README.md#概念目录全集) · [learning-paths](./learning-paths.md)
