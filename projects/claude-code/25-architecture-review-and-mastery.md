# 25 · 架构评价与 Mastery 验收

> **用途：** 入门/深读后的自测，以及对 v2.1.88 架构的 engineering 评价（非产品 PR）。

---

## 1. 架构优点

### 1.1 内核边界清晰

- **`query.ts`** 集中 agent loop，compact / API / tool 顺序固定
- **`query/deps.ts`** 注入便于测试，避免 1700 行不可 mock
- **`StreamingToolExecutor`** 把「流式 tool_use」与 batch 路径分开，并发语义明确

### 1.2 Generator 一等公民

从 `query()` → UI / SDK 全链路 async generator，**partial 事件**无需轮询。适合 terminal 产品低延迟反馈。

### 1.3 上下文治理前置

每轮 loop **先 compact 再 API**，且 snip / microcompact / collapse / autocompact 可组合。体现「context 是稀缺资源」的产品假设。

### 1.4 Tool 模块化

42 个 tool 各目录独立，registry 在 `tools.ts` feature-gated 加载。扩展 MCP/Skill 不改 loop 形状。

### 1.5 Headless 与 REPL 共享内核

`QueryEngine` 统一 session 语义，SDK 与 TUI 不是两套 agent 逻辑。

---

## 2. 架构代价与风险

### 2.1 体量与重复

- `print.ts` 与 REPL **大量平行逻辑**（~5500 vs 分散 components），维护成本高
- `main.tsx` ~4700 行，CLI flag 与 init 交织，新人入口难

### 2.2 状态分散

- `AppState`、`QueryEngine` 私有字段、queryLoop `State`、`sessionStorage` 多处可变状态
- Debug 需跨文件追踪 **messages 的三个视图**（REPL / API / disk）

### 2.3 Feature flag 与暴露版不一致

`bun:bundle` 裁剪导致 **公开快照含未启用分支**，读 `feature('XXX')` 时要假设「发行包可能无此代码」。

### 2.4 终止语义复杂

`needsFollowUp`、stop hooks、413 withhold、reactive compact、`maxTurns` 等多条路径交织；`transition.reason` 是读 loop 的关键。

### 2.5 暴露材料的法律与供应链风险

不适合作为依赖库；安全研究应聚焦 **为何 sourcemap 泄露**，而非鼓励未授权再分发。

---

## 3. 与「简单 Chat CLI」的本质差异

| 维度 | 简单 wrapper | Claude Code v2.1.88 |
|------|--------------|---------------------|
| 循环 | 单轮或固定 N 轮 | `while(true)` + 多 recovery |
| 上下文 | 全量 messages | 多层 compact + collapse |
| 工具 | 同步顺序 | 流式并发 + permission |
| 持久化 | 可选 log | uuid 链 + sidechain + resume |
| 扩展 | 无 | MCP / skills / plugins / agents |

---

## 4. 入门自测（路径 A 后）

1. 画出从用户回车到 `queryLoop` 的 **≥5 个函数/模块**。
2. **`shouldQuery: false`** 时会发生什么？（slash 本地命令）
3. compact 与 **`deps.callModel`** 谁在前？
4. 工具执行前必经哪个回调？
5. 为何建议首读跳过 `components/`？

<details>
<summary>参考答案要点</summary>

1. 例：`handlePromptSubmit` → `processUserInput` → `QueryEngine.submitMessage` → `query` → `queryLoop` → `deps.callModel` → `StreamingToolExecutor`
2. QueryEngine yield 命令输出后直接 return，不调 `query()`
3. compact 在前
4. `canUseTool`
5. UI 占 LOC 大半，不决定 loop 语义

</details>

---

## 5. 深读自测（路径 B 后）

1. 列举 `queryLoop` 至少 **4 种** `transition.reason` 及触发条件。
2. `StreamingToolExecutor` 与 `runTools` 选型 gate 是什么？
3. `recordTranscript` 如何在 compact 后 **截断 parent 链**？
4. `fetchSystemPromptParts` 三件套各是什么？
5. headless 权限 UI 由什么替代？
6. MCP `pending` client 如何影响 API 请求？

<details>
<summary>参考答案要点</summary>

1. 例：`collapse_drain_retry`、reactive compact retry、stop hook retry、maxTurns 等（需对照源码标注行号）
2. `config.gates.streamingToolExecution`
3. 新 compact 消息非 prefix → 不更新 parent → chain 截断
4. defaultSystemPrompt / userContext / systemContext
5. permission-prompt-tool MCP 或 allow/deny flags
6. `hasPendingMcpServers` 传入 callModel options

</details>

---

## 6. 简答（架构扫读）

1. Claude Code 的 **L0 内核** 是哪 4 个模块/目录？
2. 为何 async generator 适合 terminal agent？
3. 暴露快照用于生产集成的主要风险是什么？

---

## 7. 继续深入

| 目标 | 下一步 |
|------|--------|
| 补内核笔记 | [05](./05-query-engine.md) · [06](./06-query-agent-loop.md) · [09](./09-tools-system.md) |
| 流程专读 | [flow/](./flow/README.md) |
| 术语速查 | [99](./99-glossary-and-reading-map.md) |
| 叙事一遍 | [A1 一条 user turn](./appendix/A1-user-turn-journey.md) |

---

## 8. Mastery 标准

视为 **v2.1.88 内核 mastery** 当且仅当：

- [ ] 完成 §4、§5 全部问题并能指向源码
- [ ] 能独立写出一页「queryLoop 单轮迭代」步骤表
- [ ] 能在 15 分钟内定位：tool denied / context overflow / API fallback 三类问题
- [ ] 能解释三层多 Agent（subagent / swarm / coordinator）[21]
- [ ] 能说明 messages 三层视图（REPL / API / disk）[08]

---

## 9. 扩展专题验收（可选）

| 专题 | 验收问题 |
|------|----------|
| Memory [29] | extractMemories 何时触发？CCR 如何开 memory？ |
| Bridge [18] | REPL vs worker 形态？permission 跨进程？ |
| Remote [22] | 120s timeout 原因？activity signal 作用？ |
| Tool Search [30] | defer_loading 与 ToolSearch 分工？ |
| Loop 门控 [28] | L1 continue vs L3 permission ask 区别？ |

**关联：** [26 总图](./26-main-chain-atlas.md) · [learning-paths](./learning-paths.md) · [99 索引](./99-glossary-and-reading-map.md)
