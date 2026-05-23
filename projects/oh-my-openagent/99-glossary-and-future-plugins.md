# 99 · 术语词典 + 未来插件清单

> **用途：** 速查（术语）+ 持续维护（插件清单）。
>
> 每次有新插件想法，往 §2 加一条；将来动手时，按"参考章节"找到入口。

---

## 1. 术语词典（按字母序）

### A

- **AGENTS.md** — 项目里给 AI / 人类阅读的"地图文件"。OmO 在几乎每个目录都放一份。
- **`AgentConfig`** — `@opencode-ai/sdk` 的 agent 定义类型。包含 description / prompt / model / temperature / tools 等。详见 [09](./09-agent-system.md)。
- **`AgentFactory`** — `(model: string) => AgentConfig` + 静态 `mode` 属性。OmO 10 个 agent 用这个模式，Prometheus 例外。
- **`AgentMode`** — `"primary" | "subagent" | "all"`。决定 agent 是否出现在 UI Tab、是否用 fallback chain。
- **Atlas** — OmO 的 todo 编排执行者。和 Sisyphus 配对：Sisyphus 决策，Atlas 推执行。
- **`AgentPromptMetadata`** — 给 dynamic-agent-prompt-builder 看的元数据（cost、triggers、useWhen），让 Sisyphus 自动知道有哪些 agent 可用。

### B

- **BackgroundManager** — 后台任务管理器。`task_create` 系列工具用它。
- **Boulder** — OmO 的"推石头"系统。todo + state file + `todoContinuationEnforcer`，逼 agent 不停推进。
- **bundled snapshot** — `dist/` 里编译进去的 model capability 快照（兜底用）。
- **Bun** — OmO 用的 JS 运行时 / 包管理器。**不能用 npm/yarn/pnpm**。

### C

- **canonical agent order** — Sisyphus → Hephaestus → Prometheus → Atlas。`installAgentSortShim()` 在数组里有 ≥2 个这些 agent 时强制排序。
- **`CategoryConfig`** — `{ model, temperature, skills, ... }`。用户用 category 给一类工作（"deep" / "ultrabrain" / "artistry"）配置默认 model。
- **`chat.headers`** — OpenCode hook，给 HTTP 请求加 header（OmO 用它注 `x-initiator` 给 GitHub Copilot）。
- **`chat.message`** — OpenCode hook，用户发完消息那一刻触发（消息已在 session）。
- **`chat.params`** — OpenCode hook，**LLM 调用前最后一刻**触发，让你改 temperature/topP/options 等。详见 [03](./03-chat-params-mechanism.md)。
- **chat-params dispatcher** — OmO 的 `chat.params` 顶层 handler，按固定顺序串 think-mode → model-fallback → anthropic-effort → compat clamp → ...
- **ContextCollector** — 跨 hook 共享上下文的单例（IntentGate 用它）。
- **ctx / PluginContext** — `Parameters<Plugin>[0]`。包含 `client`（SDK 客户端）、`directory`（项目目录）、`app` 等。

### D

- **`disabled_hooks`** — 用户配置，按名禁用 OmO 内部 hook。
- **`disabled_tools`** — 用户配置，按名禁用 tool。
- **disposable hook** — 持有外部资源（interval / listener）的 hook，有 `.dispose()` 方法。
- **DSL ladder** — `VARIANT_LADDER` / `REASONING_LADDER`，clamping 时用来"找最近的低位"。

### E

- **effort** — Anthropic Claude / 部分 OpenAI 模型的推理强度。OmO 把 `output.options.effort` 设到 `"low" | "medium" | "high"`。
- **explore** — OmO 的快速 codebase 检索 agent（read-only）。

### F

- **factory pattern** — `createXxx(deps)` 函数返回对象。OmO 全用这个模式（hook / tool / agent / manager）。
- **fallback chain** — agent 的多备选模型序列。subagent 主模型不可用时按 chain 自动切换。

### G

- **Gemini family** — `google/` / `google-vertex/` / `github-copilot/gemini-*` 都算。`isGeminiModel` 判定。
- **graceful degradation** — 模型不支持的功能（thinking / variant / topP）静默降级或丢弃，**不报错**。

### H

- **Hashline edit** — OmO 的 `LINE#ID` 哈希校验 edit 工具。Read 输出附加哈希，edit 时校验是否过时。
- **Hephaestus** — OmO 的"GPT-native autonomous worker" agent，专门给 GPT-5.5 / Codex 这种自主性强的模型用。
- **heuristic family** — 通过 regex 识别模型族（claude-opus / gpt-5 / deepseek-r1）。`detectHeuristicModelFamily()`。

### I

- **IntentGate / keyword-detector** — 识别用户消息里的 `ultrawork` / `search` / `analyze` / `team` 等关键词，注入对应 prompt。
- **internal agent** — OpenCode 的 `title` / `summary` / `compaction` 等"工具型 agent"，**OmO 应该跳过它们**（你的插件也应该）。

### K

- **Kimi K2** — Moonshot 的模型族。`isKimiK2Model` 识别。
- **keyword detector** — 见 IntentGate。

### L

- **ladder** — `VARIANT_LADDER` = `["low","medium","high","xhigh","max"]`。`REASONING_LADDER` = `["none","minimal","low","medium","high","xhigh","max"]`。clamping 时降级到 ladder 里最近的低位允许值。
- **Librarian** — OmO 的 OSS 库 / 文档检索 agent。
- **log** — `src/shared/logger.ts`。写到 `/tmp/oh-my-opencode.log`。
- **LSP tools** — OmO 6 个 LSP 工具：`lsp_goto_definition` / `lsp_find_references` / `lsp_symbols` / `lsp_diagnostics` / `lsp_prepare_rename` / `lsp_rename`。

### M

- **`ModelCapabilities`** — 模型支持什么 variant / reasoning / thinking / temperature / topP / maxOutputTokens。详见 [06](./06-model-capabilities-and-compat.md)。
- **model-fallback** — **proactive** 模型 fallback，在 `chat.params` 阶段根据配置链选模型。
- **`mode` (agent)** — primary / subagent / all。
- **Multimodal-Looker** — OmO 的图片 / PDF 看图 agent（用 `look_at` tool）。

### N

- **normalizeModelID** — `modelID.replace(/\.(\d+)/g, "-$1")`。把 `gpt-5.5` 变 `gpt-5-5`，但 `deepseek-r1` 不变（因为后面不是数字）。

### O

- **OmO** — oh-my-opencode / oh-my-openagent 简称。社区习惯叫 OmO。
- **OpenClaw** — OmO 的双向外部集成（Discord / Telegram / HTTP / shell）。
- **OverridableAgentName** — `"build" | BuiltinAgentName`。OpenCode 内置的 `build` agent 加 OmO 10 个。

### P

- **PluginContext** — 见 ctx。
- **PluginInterface** — OmO 暴露给 OpenCode 的 10 个 hook handler 集合。
- **primary** — agent mode，UI 可选、用 UI 选的 model。
- **Prometheus** — OmO 的战略规划 agent（**只能改 .md 文件**，由 `prometheus-md-only` hook 强制）。
- **provider** — OpenAI / Anthropic / Google / DeepSeek / GitHub Copilot / Kimi / Z.ai 等。

### R

- **`reasoning`** — model capability，表示这是个推理模型（thinking-by-default）。
- **`reasoningEffort`** — request 字段，控制推理强度。
- **runtime-fallback** — **reactive** 模型 fallback，在 session.error 事件触发后自动重试新模型。
- **runtime snapshot** — `build:model-capabilities` 任务从 models.dev 拉的最新缓存。

### S

- **`safeCreateHook`** — 给 hook 创建包装 try/catch，单 hook 创建失败不影响其他。
- **session** — OpenCode 的会话。每个 session 有 ID、parent ID、title。
- **Sisyphus** — OmO 主编排者（永远不直接执行，只 delegate）。
- **Sisyphus-Junior** — category 调度时的"代理" subagent。
- **Skill** — OmO 的 SKILL.md 系统。skill 可能内嵌 MCP server。
- **`skill`** tool — 让 agent 加载某个 skill 的 prompt 到上下文。
- **`skill_mcp`** tool — 让 agent 调用 skill 内嵌的 MCP server。
- **snapshot-backed / alias-backed / heuristic-backed** — `ModelCapabilities.diagnostics.resolutionMode` 的 3 种状态，告诉你"这次解析靠的是哪种数据源"。
- **subagent** — agent mode，只能通过 task tool 调，UI 不显示。
- **`supportsThinking`** — model capability bool，能不能用 thinking 模式。

### T

- **Task tool / delegate-task** — OmO 的 `task` tool，让 primary agent 调 subagent。
- **Team Mode** — OmO 多 agent 并行协调系统。默认 OFF。
- **`thinking`** — request option 字段，控制 thinking 模式（true / { type: "enabled", budget_tokens } / Anthropic 风格的对象）。

### U

- **ultrawork / ulw** — 用户关键词，IntentGate 检测后注入"深度工作"prompt 模式。

### V

- **variant** — OmO 概念。`"low" | "medium" | "high" | "xhigh" | "max"`。把多家 provider 的"推理强度"统一抽象。OmO 内部 chat-params dispatcher 把它翻译成各 provider 的具体字段（Anthropic effort、OpenAI reasoning.effort、`enable_thinking` 等）。

### Z

- **Zod v4** — OmO 的配置验证库。Schema 写完自动转 JSON Schema 给编辑器自动补全。

---

## 2. 未来想做的插件清单

> 每次有新想法就往这里加一条。**有了想法直接动手前**，按"参考章节"找到入口，再看 OmO 里有没有现成例子。

### 模板

```markdown
### N. 插件名 / 一句话目标

- **痛点：**（一句话说清现状哪里痛）
- **方案：**（一句话说清要改什么 hook、什么字段）
- **参考章节：** 03 / 04 / 06 / ...
- **OmO 内最像的例子：** `src/hooks/xxx/`
- **优先级：** 高/中/低
- **状态：** 想法 / 调研中 / 实现中 / v0.1 / v0.2 / 已发布
```

---

### 1. opencode-thinking-toggle / 强制控制 OpenAI 兼容 provider 的 thinking 开关

- **痛点：** DeepSeek / Kimi / GLM 等通过 OpenAI SDK 接入时，没有原生方式关闭 thinking。
- **方案：** `chat.params` hook，按 providerID + modelID 匹配规则，往 `output.options` 写入对应字段（`thinking` / `reasoningEffort` / `enable_thinking` / `reasoning.effort`）。
- **参考章节：** [03](./03-chat-params-mechanism.md) + [04](./04-anthropic-effort-case-study.md)
- **OmO 内最像的例子：** `src/hooks/anthropic-effort/`
- **优先级：** 高（你的第一个产出）
- **状态：** ✅ v0.1 完成（`~/Github/opencode-thinking-toggle`，23 测试全过）

**Roadmap：**

- v0.2：加入 reasoning effort 自动 clamp（按模型支持列表降级），参考 [06](./06-model-capabilities-and-compat.md)
- v0.3：支持用户在 OmO 配置里写自定义 rules（用 OmO 的 JSONC schema 集成）
- v0.4：发布到 npm

---

### 2. 占位 - 你来填

#### 灵感来源（你可以从这里挑）

| 想法 | 触发点 | 用哪个 hook |
|------|--------|------------|
| **Cost guard** | 用户用某些昂贵模型时弹警告 | `chat.params` |
| **Rate limit smoother** | 主动 sleep 避开 RPM 限制 | `chat.params`（return Promise 拖延） |
| **Token budget alarm** | session 累计 token 超阈值时通知 | `event` (session.idle) |
| **Auto-prefix prompt** | 给特定 agent / category 加固定前缀 | `chat.message` 或 `messages.transform` |
| **Custom slash command** | 自己加 `/myacme` 命令调外部 API | `chat.message` |
| **Provider 私有 API 集成** | 某 provider 有 `cache_control` 字段，OmO 没用上 | `chat.params` 加 options |
| **本地搜索 tool** | 暴露 `local_search` 给 LLM 用 | `tool` |
| **特定语言 LSP enhance** | 检测到 Rust 项目时注入 rustfmt 提示 | `event` (session.created) + `chat.message` |
| **Multi-model router** | 不同 task 路由到不同 provider | `chat.params` 改 `model.providerID` |
| **Debug overlay** | 把每次 chat.params 入出参 dump 到文件 | `chat.params`（透传 + side effect） |

→ **挑一个写一句"为什么我想要这个"**，再补痛点和方案，就能往清单里加一条。

---

### 3. 占位

---

### 4. 占位

---

## 3. 学习深化方向（按需）

如果将来某天你想再深入：

| 方向 | 入口 |
|------|------|
| 看完 OmO 60 个 hook 各自做什么 | `src/hooks/AGENTS.md` |
| 看完 20 个 feature 模块的边界 | `src/features/AGENTS.md` |
| Team Mode 怎么实现的（多 agent 并行）| `src/features/team-mode/AGENTS.md` |
| Skill MCP 怎么动态 spawn | `src/features/skill-mcp-manager/` |
| Hashline edit 哈希怎么算的 | `src/tools/hashline-edit/` |
| Boulder state 怎么持久化 | `src/features/boulder-state/` |
| 怎么写 OpenCode SDK 反向调用 | `ctx.client.*` 用法散落各 tool 里，搜 `ctx.client.` |

---

## 4. 自己写下"我学完的感受"

> 写完插件、读完所有概念后回来填这里。

| 学完日期 | 心得（你之后填） |
|----------|-----------------|
| 2026-05-23 | 文档骨架完成；待 v0.1 thinking-toggle 在 OpenCode 里真实跑过后再补 |
| | |
| | |

---

→ **回到目录：** [README](./README.md)
