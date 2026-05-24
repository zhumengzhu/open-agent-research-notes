# 99 · 术语表与阅读地图

> **路径：** [learning-paths.md](./learning-paths.md) · **社区：** [external-resources.md](./external-resources.md)

---

## 1. 概念 → 章节

| 概念 | 主章节 | 一句话 |
|------|--------|--------|
| Agent loop | [06](./06-query-agent-loop.md) · [28](./28-agent-loop-continuation-and-human-gates.md) · [26](./26-main-chain-atlas.md) | continue / 等人 / 终止 |
| QueryEngine | [05](./05-query-engine.md) | 会话引擎，submitMessage |
| 渐进式压缩 | [10](./10-compaction-and-context.md) | snip→MC→collapse→autocompact |
| Prompt cache | [10 §10](./10-compaction-and-context.md#10-prompt-cache-体系) | prefix KV 复用 |
| CacheSafeParams | [10](./10-compaction-and-context.md) · [20](./20-agents-and-subagents.md) | fork 共享前缀 |
| callModel | [07](./07-api-and-model-stream.md) | queryModelWithStreaming |
| Tool 执行 | [09](./09-tools-system.md) | StreamingToolExecutor |
| Permission | [11](./11-permission-and-hooks.md) · [28 §6](./28-agent-loop-continuation-and-human-gates.md#6-l3loop-内部的人机门控阻塞但不结束-turn) | canUseTool · yolo/auto mode |
| Plugin | [15](./15-skills-and-plugins.md) | marketplace 分包 · 可含 hooks |
| Hooks | [11 §6](./11-permission-and-hooks.md#6-hooks-体系) · [15 §4](./15-skills-and-plugins.md#4-hooks-与-plugin-的关系) | shell/HTTP 回调 · ≠ plugin |
| Message/JSONL | [08](./08-message-and-session-persistence.md) | recordTranscript |
| System prompt | [13](./13-system-prompt-and-context.md) | fetchSystemPromptParts |
| MCP | [14](./14-mcp-and-external-protocols.md) | AppState.mcp |
| Headless/SDK | [19](./19-sdk-headless-and-print-mode.md) | runHeadless → ask |
| Slash 命令 | [12](./12-commands-and-input-preprocessing.md) | shouldQuery |
| Plan mode | [17](./17-plan-mode-and-code-editing.md) | permission mode plan |
| Subagent | [20](./20-agents-and-subagents.md) | AgentTool + fork |
| Team/Swarm/Coordinator | [21](./21-tasks-team-and-coordinator.md) | Task* / Team* / coordinator mode |
| Auto-memory | [29](./29-memory-and-auto-memory.md) | memdir + extractMemories |
| Tool Search / 实验 | [30](./30-advanced-features-and-experiments.md) | defer_loading、Proactive、KAIROS |
| 模型/fallback | [27](./27-multi-model-thinking-and-fallback.md) | model.ts + query retry |

---

## 2. 核心术语

| 术语 | 含义 |
|------|------|
| **querySource** | 区分 repl/sdk/agent/compact，影响 guard |
| **ToolUseContext** | 单 turn 工具执行上下文 |
| **needsFollowUp** | 本轮是否有 tool_use 需 continue |
| **preventContinuation** | stop/post hook 终止 loop |
| **shouldQuery** | slash 后是否进 query loop |
| **bypassPermissions** | 跳过全部 permission（yolo 最强形态） |
| **auto mode** | yoloClassifier AI 判权限 |
| **messagesForQuery** | 送 API 的 message 视图 |
| **sidechain** | 子 agent transcript 文件 |
| **compact_boundary** | 压缩边界 system message |
| **cached MC** | cache_edits 删 KV 内 tool result |
| **defer_loading** | LSP/MCP tool 延迟加载 schema；常配合 ToolSearch [30] |
| **isAgentSwarmsEnabled** | Agent Teams 总开关 [21] |
| **extractMemories** | turn 结束 fork 写 auto-memory [29] |

---

## 3. 文档分层

| 类型 | 文档 |
|------|------|
| 地图 | [26](./26-main-chain-atlas.md) |
| 叙事 | [A1](./appendix/A1-user-turn-journey.md) |
| 深读 | [A2](./appendix/A2-query-loop-transitions.md) · [10](./10-compaction-and-context.md) · [28](./28-agent-loop-continuation-and-human-gates.md) |
| 流程 | [flow/](./flow/README.md) |
| 验收 | [25](./25-architecture-review-and-mastery.md) |

---

## 4. 按目标选读

| 目标 | 路径 |
|------|------|
| 第一次学 | 00 → 26 → A1 → 06 → 25 §4 |
| 内核深读 | [learning-paths B](./learning-paths.md#路径-b内核深读--约-35-天) |
| SDK 集成 | 19 → 07 → 08 |
| 压缩/cache | 10 全文 |
| Loop 续跑/等人 | **28** 全文 → 11 → 17 |
| 工具/权限/yolo | 09 → 11 → 28 §8 |
| Bridge / Remote | [18](./18-bridge-and-ide.md) · [22](./22-remote-and-server-mode.md) |
| Cost / analytics | [24](./24-cost-analytics-and-limits.md) |
| CLI flags | [03](./03-cli-entry-and-repl.md) §5 |
| 多 Agent / Team | 20 → **21** → 23 |
| Memory | **29** → 13 §memory |
| 实验特性 | **30** → 24 analytics gates |
| Bridge / CCR | **18** → **22** |
| CLI 全 flag | **03** §5 |

---

## 5. 源码模块索引

| 目录 | 文档 |
|------|------|
| query.ts / QueryEngine.ts | 05–06 |
| services/compact/ | 10 |
| services/api/ | 07、10 §10 |
| services/tools/ | 09 |
| utils/sessionStorage.ts | 08 |
| utils/forkedAgent.ts | 10、20、29 |
| memdir/ | 29 |
| utils/swarm/ | 21 |
| utils/toolSearch.ts | 30 |
| cli/print.ts | 19 |
| main.tsx / REPL | 03 |

→ [README 概念目录](./README.md#概念目录)
