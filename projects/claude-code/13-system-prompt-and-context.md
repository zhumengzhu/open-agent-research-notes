# 13 · System Prompt 与 Context 组装

> **锚点：** `utils/queryContext.ts` · `constants/prompts.ts` · `context.ts` · `memdir/memdir.ts` · `QueryEngine.submitMessage`

---

## 1. 问题定义

发给 Anthropic API 的不只是 `messages[]`，还有 **system prompt 多块** 与 **user/system context 字典**。它们：

- 在 **cache key** 中占前缀（同 turn 多轮 loop 应稳定）
- 由 **tools、MCP、cwd、CLAUDE.md、memory、agent** 等共同决定
- 在 QueryEngine 与 compact **fork 子 Agent** 时必须一致（`CacheSafeParams` [20]）

---

## 2. 三件套：`fetchSystemPromptParts`

```44:73:/Users/zmz/Github/claude-code/src/utils/queryContext.ts
export async function fetchSystemPromptParts({ ... }): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}> {
  const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
    customSystemPrompt !== undefined
      ? Promise.resolve([])
      : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
    getUserContext(),
    customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
  ])
  return { defaultSystemPrompt, userContext, systemContext }
}
```

| 字段 | 来源 | 进入 API 方式 |
|------|------|---------------|
| `defaultSystemPrompt` | `getSystemPrompt()` | `asSystemPrompt()` → system blocks |
| `userContext` | `getUserContext()` | `prependUserContext(messages, …)` |
| `systemContext` | `getSystemContext()` | `appendSystemContext(systemPrompt, …)` |

**customSystemPrompt：**- 若 CLI/SDK 传入 `--system-prompt`，跳过默认 build，`systemContext` 默认 build 也跳过，避免 append 到空 default。

---

## 3. QueryEngine 内的二次组装

`submitMessage` 在 `fetchSystemPromptParts` 之后还会：

1. 合并 **agent** 的 `getSystemPrompt()`（非 built-in agent）[20]
2. 注入 **coordinator** `getCoordinatorUserContext()` → 常进 `userContext.workerToolsContext` [21]
3. 注入 memory-mechanics、output style 等 turn 级 extras
4. 应用 **`appendSystemPrompt`**（CLI `--append-system-prompt`）
5. 转为 `asSystemPrompt(...)` 供 API 使用

Loop 内每轮：

```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

`userContext` 通过 `prependUserContext(messagesForQuery, userContext)` 进入 messages **前缀**（影响 cache）。

---

## 4. `getSystemPrompt` 结构与依赖

`constants/prompts.ts` — 按 **section** 组装（`systemPromptSection` 模式）：

| Section（概念） | 内容 |
|-----------------|------|
| 行为准则 |  reversible vs destructive 行动、确认策略 |
| Tools | 当前 tool 池摘要（与 registry 同步） |
| MCP | 已连接 server 能力 |
| Memory | `loadMemoryPrompt()` — 见 §5 |
| Model 提示 | 最新 Claude 4.5/4.6 id 等 |
| Environment | cwd、git、日期等 |

**输入依赖：**

- **tools** — 换 tool 集 → system 变 → cache 变 [09]
- **mainLoopModel** — 模型特定说明 [27]
- **additionalWorkingDirectories** — 多目录
- **mcpClients** — 连接变化即时反映

**bare mode（`CLAUDE_CODE_SIMPLE`）：** early return，drop memory section 等；与 `isAutoMemoryEnabled()` 双关 [29]。

---

## 5. Memory 段（与 29 的关系）

### 5.1 `loadMemoryPrompt()`

`memdir/memdir.ts`：

- 读 `MEMORY.md` entrypoint（`getAutoMemPath()`）
- **截断：** max 200 行 + 25KB（`truncateEntrypointContent`）
- 含 WHEN_TO_ACCESS、WHAT_NOT_TO_SAVE、类型说明（`memoryTypes.ts`）
- KAIROS 分支可能改 recall 策略 [30]

进入 system 的 `memory` section；与 **磁盘 auto-memory 文件** 内容同步（模型 `/remember` 或 extractMemories 写入后下 turn 可见）。

### 5.2 运行时检索

`findRelevantMemories` — 按当前 query 选片段，可能 **追加** 到 userContext 或 attachment（非全量进 static system）。详见 [29](./29-memory-and-auto-memory.md)。

### 5.3 Team memory

feature `TEAMMEM` — `teamMemPaths` / `teamMemPrompts` 注入 team 级记忆 [21]。

---

## 6. Context 文件：`context.ts`

与 **磁盘 instruction** 相关（非 auto-memory）：

| 来源 | 机制 |
|------|------|
| `CLAUDE.md` | 项目根向上 walk |
| `.claude/rules/` | 规则片段 |
| User CLAUDE | `~/.claude/CLAUDE.md` 等 |
| `@` 引用 | `parseReferences`、paste 展开 |

与 `history.ts` 配合处理大 paste、`expandPastedTextRefs`。

**bare mode：** 跳过 CLAUDE.md 自动发现 → 需 `--system-prompt-file` 或显式 `--system-prompt`。

---

## 7. Attachments 与 turn 级注入

`processUserInput` / `getAttachmentMessages` 在 **user message 入库前** 附加：

- IDE selection、diagnostics [18]
- 图片（resize → base64）
- `@agent` mention
- Skill discovery（turn 0 可能阻塞；后续 prefetch [15]）

这些进入 **messages**，一般不进入 static system prompt（除非 skill 改 tool 集间接影响 `getSystemPrompt`）。

---

## 8. Coordinator / Agent 叠加

| 来源 | 注入位置 | 内容 |
|------|----------|------|
| Custom agent | system 追加 | agent 专用 system prompt |
| Coordinator | userContext | worker tool 列表、MCP 名、scratchpad [21] |
| Plan mode | permission + runtime model | 不直接改 system 结构 [17] |

Coordinator system：`getCoordinatorSystemPrompt()` — 编排者角色、禁止亲自改代码等（`coordinatorMode.ts`）。

---

## 9. Compact / fork 一致性

- **Autocompact** 把 `systemPrompt, userContext, systemContext, toolUseContext` 打包为 `cacheSafeParams` 传给 compact agent [10]
- **Side question**（SDK 控制请求）用 `buildSideQuestionFallbackParams` 重建 cache 前缀
- **extractMemories / AgentTool fork** 必须复用 parent `CacheSafeParams` [20][29]
- **Untracked sources**（speculation、session_memory）可能 break cache — [30][24]

---

## 10. 组装时序（单 turn）

```text
submitMessage 开始
  → fetchSystemPromptParts (parallel getSystemPrompt + contexts)
  → merge agent / coordinator / appendSystemPrompt
  → 固定为 turn 级 systemPrompt + userContext + systemContext
queryLoop iteration
  → prependUserContext(messagesForQuery)
  → appendSystemContext → fullSystemPrompt
  → normalizeMessagesForAPI + addCacheBreakpoints [07]
  → callModel
```

Turn 内 **systemPrompt 字符串通常不变**；变的是 messages 与 cache_edits。

---

## 11. 自测

- [ ] 三件套各自从哪个函数来？如何进入 API？
- [ ] `customSystemPrompt` 为何清空默认 `systemContext` build？
- [ ] tools/MCP 变化如何影响 system？对 cache 的影响？
- [ ] `loadMemoryPrompt` 与 extractMemories 的分工？
- [ ] coordinator 信息进 userContext 还是 systemContext？
- [ ] loop 每轮 `fullSystemPrompt` 与 turn 开始时组装的 prompt 关系？

**关联：** [12 Commands](./12-commands-and-input-preprocessing.md) · [29 Memory](./29-memory-and-auto-memory.md) · [21 Coordinator](./21-tasks-team-and-coordinator.md) · [10 Compact/Cache](./10-compaction-and-context.md)
