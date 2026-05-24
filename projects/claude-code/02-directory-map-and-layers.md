# 02 · 目录地图与四层分层

> **基准：** v2.1.88 · `src/` ~1900 文件  
> **框架：** [01 架构总览 §2](./01-architecture-overview.md#2-四层架构由核心到外)

---

## 1. 顶层目录（按文件数）

| 目录 | 文件数级 | 层级 | 首读 |
|------|----------|------|------|
| `utils/` | ~564 | L1/L2 混合 | 按需 |
| `components/` | ~389 | L2 | 后读 |
| `commands/` | ~189 | L1 | [12](./12-commands-and-input-preprocessing.md) |
| `tools/` | ~184 | L1 | [09](./09-tools-system.md) |
| `services/` | ~130 | L0/L1 | compact、api、mcp |
| `hooks/` | ~104 | L1 | [11](./11-permission-and-hooks.md) |
| `ink/` | ~96 | L2 | UI 渲染 |
| `bridge/` | ~31 | L3 | [18](./18-bridge-and-ide.md) |
| `cli/` | ~19 | L2 | [19](./19-sdk-headless-and-print-mode.md) |
| `skills/` | ~20 | L1 | [15](./15-skills-and-plugins.md) |
| `tasks/` | ~12 | L3 | [21](./21-tasks-team-and-coordinator.md) |
| `utils/swarm/` | ~22 | L3 | [21](./21-tasks-team-and-coordinator.md) |
| `memdir/` | ~8 | L1 | [29](./29-memory-and-auto-memory.md) |
| `coordinator/` | 少 | L3 | [21](./21-tasks-team-and-coordinator.md) |
| `proactive/` | 少 | L3 | [30](./30-advanced-features-and-experiments.md) |
| `query/` | 4 | L0 卫星 | [06](./06-query-agent-loop.md) |

**根目录单文件（L0）：** `query.ts`、`QueryEngine.ts`、`Tool.ts`、`tools.ts`、`commands.ts`、`main.tsx`。

---

## 2. L0 内核文件清单

| 路径 | 行量级 | 文档 |
|------|--------|------|
| `query.ts` | ~1729 | [06](./06-query-agent-loop.md) |
| `QueryEngine.ts` | ~1295 | [05](./05-query-engine.md) |
| `Tool.ts` | ~792 | [09](./09-tools-system.md) |
| `tools.ts` | ~389 | [09](./09-tools-system.md) |
| `services/tools/*` | 4 文件 | [09](./09-tools-system.md) |
| `services/api/claude.ts` | 大 | [07](./07-api-and-model-stream.md) |
| `services/compact/*` | 11 | [10](./10-compaction-and-context.md) |
| `query/deps.ts` 等 | 4 | [06](./06-query-agent-loop.md) |

---

## 3. `services/` 子目录

| 子目录 | 职责 | 文档 |
|--------|------|------|
| `api/` | Anthropic 客户端、streaming、cache breakpoints | 07 |
| `compact/` | 渐进式压缩 | 10 |
| `mcp/` | MCP 连接 | 14 |
| `lsp/` | LSP 管理 | 16 |
| `plugins/` | 插件加载 | 15 |
| `SessionMemory/` | session memory 实验 | 10 §8.2 |
| `extractMemories/` | turn 结束记忆提取 | 29 |
| `autoDream/` | 会话后整合 | 29、30 |
| `PromptSuggestion/` | speculation | 30 |
| `analytics/` | GrowthBook、埋点 | 24 |

---

## 4. `utils/` 怎么读

不必通读。按主题跳转：

| 主题 | 路径 |
|------|------|
| 消息 | `utils/messages.ts` |
| Session 磁盘 | `utils/sessionStorage.ts` |
| 输入 | `utils/processUserInput/` |
| Permission | `utils/permissions/`、`hooks/toolPermission/` |
| Fork/cache | `utils/forkedAgent.ts` | [10](./10-compaction-and-context.md)、[20](./20-agents-and-subagents.md)、[29](./29-memory-and-auto-memory.md) |
| Swarm | `utils/swarm/` | [21](./21-tasks-team-and-coordinator.md) |
| Bridge | `bridge/` | [18](./18-bridge-and-ide.md) |
| Remote | `remote/` | [22](./22-remote-and-server-mode.md) |
| Memory | `memdir/` | [29](./29-memory-and-auto-memory.md) |
| Tool search | `utils/toolSearch.ts` | [30](./30-advanced-features-and-experiments.md) |
| Config | `utils/config.js`、`utils/settings/` |
| Hooks | `utils/hooks/` |

---

## 5. 四层对照表

| L0 | L1 | L2 | L3 |
|----|----|----|-----|
| query、QueryEngine | tools、compact、hooks、memdir | main、print、components | bridge、remote、swarm |
| services/tools、api | commands、mcp、skills | screens、ink | voice、vim、buddy |

---

## 6. 自测

- [ ] 512k LOC 最大目录是哪两个？为何首读可跳过？
- [ ] L0 至少列 5 个路径
- [ ] `services/` 与 `tools/` 区别

**关联：** [01](./01-architecture-overview.md) · [99 索引](./99-glossary-and-reading-map.md)
