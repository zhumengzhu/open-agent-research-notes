# 30 · 高级特性与实验门控

> **锚点：** `utils/toolSearch.ts` · `tools/ToolSearchTool/` · `services/PromptSuggestion/` · `proactive/` · `services/autoDream/` · feature flags in `bun:bundle`

---

## 1. 为何单独成篇

下列能力 **不在 L0 主链路** 上，但影响 token 经济、UX 与 multi-agent；且大量依赖 **GrowthBook + env + bun feature()** 三重门控。本篇作 **实验特性索引 + 深读入口**。

---

## 2. Tool Search 与 defer_loading

### 2.1 问题

MCP + 内置 tool schema 过大时，首包 token 爆炸。解法：**deferred tools** — API 请求里带 `defer_loading: true`，模型通过 **ToolSearchTool** 按需 discover。

### 2.2 启用逻辑

`utils/toolSearch.ts`：

- `isToolSearchEnabledOptimistic()` — 注册阶段粗判（`tools.ts` 是否含 ToolSearchTool）
- `isToolSearchEnabled(model, tools, messages…)` — 精确判定
- 自动阈值：`ENABLE_TOOL_SEARCH=auto:N`，默认 **context 的 10%** 被 tool 描述占满则开启
- 手动：`ENABLE_TOOL_SEARCH=1/0`

`services/api/claude.ts`：

- MCP tools、部分 LSP、`shouldDefer` 工具 → defer
- 不支持 tool_reference 的模型 **过滤掉 ToolSearchTool**

`services/compact/compact.ts` — compact 路径同样计算 `useToolSearch`。

### 2.3 执行路径

`services/tools/toolExecution.ts` — ToolSearch 命中后动态加载 deferred tool schema 进后续 turn。

**关联：** [09 Tools](./09-tools-system.md) · [16 LSP defer](./16-lsp-and-code-intelligence.md)

### 2.3 执行路径细节

`ToolSearchTool` 输入 keywords → 匹配 `searchHint` + tool name → 返回 **tool_reference** blocks → 后续 turn API 加载对应 deferred schema。

`services/tools/toolExecution.ts`：

- `isToolSearchEnabledOptimistic()` 快速路径
- 实际 execute 前再验 `isToolSearchToolAvailable(tools)`

**不支持 tool_reference 的模型：** `claude.ts` 整包过滤 ToolSearchTool，防 API 400。

---

## 3. PromptSuggestion 与 Speculation

`services/PromptSuggestion/speculation.ts`：

- Turn 结束后 **推测** 用户下一句输入
- REPL：`handleSpeculationAccept`、`ActiveSpeculationState`
- 使用 `getLastCacheSafeParams()` 共享 cache [20]
- **Untracked cache source** — `promptCacheBreakDetection.ts` 列为可能 break prefix [24]

UX：用户 Tab/Accept 采纳建议，减少打字；拒绝则丢弃。

---

## 4. Proactive 与 KAIROS

### 4.1 Proactive

`proactive/` + `useProactive.ts`（REPL hook）：

- 条件：`feature('PROACTIVE')` 或 `feature('KAIROS')`
- env：`CLAUDE_CODE_PROACTIVE`
- **与 coordinator 互斥** — `main.tsx` 在 coordinator mode 下不启 proactive

行为：空闲时主动分析仓库、建议下一步（具体 prompt 见 `proactive/index`）。

### 4.2 KAIROS（ant 构建为主）

`feature('KAIROS')`：

- `kairosActive` analytics metadata
- 专用 assistant history、viewerOnly 模式
- 替代 autoDream 的 disk-skill dream 路径 [29]
- REPL 大量 `#if KAIROS` 分支（build-time DCE）

外部 snapshot 可能 **整 feature 不存在** — 读源码时注意 `feature()` 守卫。

---

## 5. Context Collapse

历史模块/实验：在超长 context 下 **折叠** 旧 tool 输出或消息块（具体 gate 以 snapshot 为准）。

- [10 Compaction](./10-compaction-and-context.md) 已述主路径 microcompact/autocompact
- context collapse 若为 **独立 experiment**，常与 GrowthBook 绑定；部分 build 可能 DCE

学习时：`grep contextCollapse` 确认当前 pin 是否仍活跃。

---

## 6. Jobs 与 Templates

| 概念 | 锚点 |
|------|------|
| `TEMPLATES` | 会话历史/模板目录常量 |
| `CLAUDE_JOB_DIR` | job 定义持久化 |
| Workflow scripts | `WORKFLOW_SCRIPTS` env — 自动化流水线 hook |

偏 **企业/CI** 场景；与 [19 print/SDK](./19-sdk-headless-and-print-mode.md) 结合跑 batch job。

---

## 7. 其它 feature-gated 能力（速查）

| 能力 | 门控提示 | 目录/文件 |
|------|----------|-----------|
| Web browser / computer use | model + permission | 浏览器类 tools |
| UDS inbox / ListPeers | ant/experimental | inbox 相关 |
| Transcript classifier / AFK | analytics + classifier | 会话分类、离开检测 |
| Scratchpad | `tengu_scratch` | coordinator [21] |
| Behavioral classifiers | GrowthBook | `yoloClassifier` 等 [11] |
| Context collapse | experiment / DCE | grep `contextCollapse` 确认 pin |
| Workflow scripts | `WORKFLOW_SCRIPTS` env | CI 流水线 hook |

---

## 8. 门控三层模型

```text
1. bun:bundle feature('NAME')     → 编译期：代码是否存在
2. env / CLI flag                 → 运行时 opt-in
3. GrowthBook / Statsig gate      → 远程 killswitch / 实验
```

**范例 — Agent Teams [21]：**

```text
feature 存在 → isAgentSwarmsEnabled()
  → ant: true
  → external: CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS | --agent-teams
  → AND tengu_amber_flint
```

**范例 — extractMemories [29]：**

```text
feature('EXTRACT_MEMORIES') → isExtractModeActive() → tengu_passport_quail
```

---

## 9. 与主链路交互

| 特性 | 触达主链位置 |
|------|--------------|
| ToolSearch | `claude.ts` normalize tools · toolExecution |
| Speculation | stopHooks 后 · REPL input |
| Proactive | REPL idle · 非 coordinator |
| extractMemories | stopHooks [29] |
| autoDream | stopHooks / DreamTask [29] |

共同主题：多数在 **turn 边界** 或 **API 编组层** 介入，不改变 `queryLoop` 核心 state machine [06]。

---

## 10. 源码带读

1. `utils/toolSearch.ts` — 阈值与 token 计数
2. `tools/ToolSearchTool/ToolSearchTool.ts`
3. `services/PromptSuggestion/speculation.ts`
4. `proactive/index.js` + `screens/REPL.tsx` proactive 分支
5. `services/api/promptCacheBreakDetection.ts`

---

## 11. 自测

- [ ] defer_loading 与 ToolSearchTool 的分工？
- [ ] 自动启用 tool search 的默认 token 比例？
- [ ] speculation 为何可能影响 prompt cache？
- [ ] proactive 为何与 coordinator 互斥？
- [ ] 三层门控各解决什么问题？

**关联：** [09 Tools](./09-tools-system.md) · [10 Context](./10-compaction-and-context.md) · [20 Fork/CacheSafeParams](./20-agents-and-subagents.md) · [24 Analytics](./24-cost-analytics-and-limits.md) · [29 Memory](./29-memory-and-auto-memory.md)
