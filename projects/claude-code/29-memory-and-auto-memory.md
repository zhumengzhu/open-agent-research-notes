# 29 · Memory 与 Auto-Memory

> **锚点：** `memdir/` · `services/extractMemories/` · `query/stopHooks.ts` · `memdir/paths.ts` · `services/autoDream/`

---

## 1. 两种「记忆」

Claude Code 里记忆分 **两条线**，勿混淆：

| 类型 | 载体 | 谁写入 | 谁读取 |
|------|------|--------|--------|
| **项目记忆** | `CLAUDE.md`、rules、skills | 用户/模型显式编辑 | system prompt 组装 [13] |
| **Auto-memory** | `~/.claude/projects/<key>/memory/` | 模型 + extractMemories 后台 | system prompt memory 段、RAG 检索 |

本篇聚焦 **Auto-memory**（memdir + extractMemories + autoDream）。

---

## 2. 开关与路径

### 2.1 `isAutoMemoryEnabled()`

`memdir/paths.ts` 优先级链（首条命中即返回）：

1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` → **关**
2. `=0` → **开**
3. `CLAUDE_CODE_SIMPLE`（--bare）→ **关**
4. `CLAUDE_CODE_REMOTE` 且无 `CLAUDE_CODE_REMOTE_MEMORY_DIR` → **关**
5. `settings.json` 的 `autoMemoryEnabled`
6. 默认 → **开**

### 2.2 目录布局

```text
{getMemoryBaseDir()}/projects/{sanitized-project-root}/memory/
  MEMORY.md          # ENTRYPOINT（memdir/memdir.ts）
  *.md               # 其它记忆文件
```

覆盖路径：

- `CLAUDE_CODE_REMOTE_MEMORY_DIR` — CCR 挂载卷 [22]
- `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` — Cowork 空间级路径
- settings `autoMemoryDirectory`（**仅 user/local settings**，防恶意 repo 写 `~/.ssh`）

`isAutoMemPath()` — filesystem permission **写 carve-out**：在 auto-mem 目录内写文件可免 dangerous write prompt（仍受路径校验约束）。

---

## 3. extractMemories（turn 结束后台提取）

### 3.1 触发点

`query/stopHooks.ts` → `handleStopHooks`：

- 模型产出 **final response（无 tool calls）** 时
- `feature('EXTRACT_MEMORIES')` 且 `isExtractModeActive()`（GrowthBook `tengu_passport_quail` 等）
- **void** 异步 `executeExtractMemories`，不 block 主 loop 返回

### 3.2 实现模式

`services/extractMemories/extractMemories.ts`：

- **Perfect fork**：`runForkedAgent` + `createCacheSafeParams`，共享 parent prompt cache [20]
- 工具子集：Read/Write/Edit/Grep/Glob/Bash/REPL 等（无全量 tool）
- 读现有 manifest：`scanMemoryFiles` + `formatMemoryManifest`
- 若 "主 agent 已写记忆" → `hasMemoryWritesSince` 跳过重复区间

### 3.3 与主 agent 分工

主 agent system prompt **始终** 含 save 指引；后台 agent 是 **safety net**——主 agent 漏提及时补写。

---

## 4. 检索与注入

| 模块 | 作用 |
|------|------|
| `findRelevantMemories.ts` | 按当前 query 选相关 memory 片段 |
| `memoryScan.ts` | 扫描目录、manifest |
| `memoryAge.ts` | 时效/衰减（若启用） |
| `teamMemPaths.ts` / `teamMemPrompts.ts` | feature `TEAMMEM` — team 级记忆同步 [21] |

注入时机：system prompt 组装 [13]、或 turn 前 attachment（视 feature gate）。

---

## 5. autoDream

`services/autoDream/autoDream.ts` — 实验性 **会话后整合**（类似「做梦」 consolidate）：

- 与 KAIROS 互斥：`getKairosActive()` 时 autoDream 关，KAIROS 用 disk-skill dream [30]
- 依赖 `DreamTask` runtime [21]
- prompt：`consolidationPrompt.ts`

用户命令：`/dream`（若未 SIMPLE/disabled）。

---

## 6. 相关命令与 hooks

- `/remember` — 显式写入 memory（非 extract 路径）
- Stop hooks 其它消费者：task completed、confidence rating
- Remote：必须配 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 才持久 [22]

---

## 7. 数据流总图

```mermaid
flowchart TB
  subgraph turn["单 turn 结束"]
    SH[handleStopHooks]
    SH --> EM[executeExtractMemories]
    EM --> FA[runForkedAgent]
    FA --> MD[memory/*.md]
  end
  subgraph next["下一 turn"]
    FR[findRelevantMemories]
    MD --> FR
    FR --> SP[system prompt memory 段]
  end
  subgraph alt["用户显式"]
    R[/remember] --> MD
  end
```

---

## 8. 安全注意

- `validateMemoryPath` — 拒绝 `/`、`C:\`、UNC、`..` 类危险根
- project settings **不可** 设 `autoMemoryDirectory`（仅 user/local）
- extract fork **skipTranscript** 策略避免 sidechain 膨胀（见源码 flags）

---

## 9. MEMORY.md 与 system prompt 衔接

### 9.1 Entrypoint 约束

`memdir/memdir.ts`：

- 文件名固定 `MEMORY.md`（`ENTRYPOINT_NAME`）
- **200 行 + 25KB** 双 cap — 超长 index 截断并 warning
- 内容模板：`memoryTypes.ts`（WHEN_TO_ACCESS、WHAT_NOT_TO_SAVE、类型 frontmatter 示例）

### 9.2 `buildMemoryPrompt` vs `loadMemoryPrompt`

| 函数 | 调用方 | 输出 |
|------|--------|------|
| `loadMemoryPrompt` | `getSystemPrompt` memory section [13] | 进 API system |
| `buildMemoryPrompt` | 内部/CLI 展示 | 格式化 manifest |

模型读到的 memory 指令 + entrypoint 摘要 **每 turn 在 system 前缀** — 改 MEMORY.md 后下一 turn cache 可能 shift（[10] cache break）。

### 9.3 `/remember` 与主 agent 写入

用户或模型显式 `/remember` → 直接写 memdir；extractMemories 检测 `hasMemoryWritesSince` **跳过已覆盖区间**，避免 duplicate fork 写。

---

## 10. 源码带读

1. `memdir/paths.ts` — 开关与路径
2. `services/extractMemories/extractMemories.ts` — fork 提取
3. `query/stopHooks.ts` — 触发
4. `memdir/findRelevantMemories.ts` — 检索
5. `services/autoDream/autoDream.ts` — dream 门控

---

## 11. 自测
- [ ] CCR 下 auto-memory 默认为何关闭？如何开启？
- [ ] extractMemories 与主 agent 写 memory 如何 dedupe？
- [ ] CacheSafeParams 对 extract fork 的意义？
- [ ] autoDream 与 KAIROS dream 互斥原因？
- [ ] `isAutoMemPath` 如何影响 write permission？

**关联：** [08 Session](./08-message-and-session-persistence.md) · [13 System Prompt](./13-system-prompt-and-context.md) · [20 Fork](./20-agents-and-subagents.md) · [22 Remote](./22-remote-and-server-mode.md) · [21 Team memory](./21-tasks-team-and-coordinator.md)
