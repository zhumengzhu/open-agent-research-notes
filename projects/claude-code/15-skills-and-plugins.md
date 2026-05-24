# 15 · Skills 与 Plugins

> **锚点：** `skills/` · `tools/SkillTool/` · `services/plugins/` · `utils/plugins/` · `types/plugin.ts` · `plugins/builtinPlugins.ts`

---

## 1. 三者边界（先建立地图）

Claude Code 扩展生态里最容易混的是 **Skill / Plugin / MCP**：

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Skill     │     │    Plugin    │     │     MCP      │
│ 能力说明包   │     │  分发安装包  │     │ 外部工具协议 │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ SKILL.md     │     │ plugin.json  │     │ server config│
│ 模型按名调用 │     │ marketplace  │     │ 动态 tool    │
│ SkillTool    │     │ 可含 skill   │     │ MCPTool      │
│ 可 embed MCP │     │ + hooks + MCP│     │ AppState.mcp │
└──────────────┘     └──────────────┘     └──────────────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                         assembleToolPool
```

| 概念 | 回答的问题 | 配置入口 |
|------|------------|----------|
| **Skill** | 模型这次任务该遵循什么工作流？ | `.claude/skills/` · plugin 内 skills |
| **Plugin** | 如何 **安装/启用** 一捆扩展？ | `enabledPlugins` · `/plugin` · marketplace |
| **MCP** | 如何连外部 server 拿 tool？ | plugin manifest · `.mcp.json` · settings |

**Hooks 不是第四种扩展**：它是 **回调机制**，可来自 settings **或** plugin manifest。详见 [11 §6](./11-permission-and-hooks.md#6-hooks-体系)。

---

## 2. Skill 系统

### 2.1 是什么

Skill = **带 YAML frontmatter 的 Markdown 能力包**（`SKILL.md`），描述何时用、怎么做、可选脚本/MCP 声明。

发现来源：

- 项目 `.claude/skills/`
- 用户 skills 目录
- Plugin 安装的 `skillsPath` / `skillsPaths`
- Bundled skills（`skills/bundled/`）

### 2.2 生命周期

```text
Turn 0 / prefetch
  → skill discovery (feature EXPERIMENTAL_SKILL_SEARCH)
  → attachment: available skills list

模型调用 SkillTool(name)
  → 加载 SKILL.md body + frontmatter
  → 可能启用 embedded MCP / 子 prompt
  → tool_result 回 loop

Compact 后
  → createSkillAttachmentIfNeeded (compact.ts)
  → 避免摘要后丢失 skill 上下文
```

`QueryEngine.discoveredSkillNames`：turn 内 telemetry（`was_discovered`），每 turn clear（[05 §3](./05-query-engine.md#3-实例状态持久跨-turn)）。

### 2.3 SkillTool 路径

1. 模型调用 `Skill`（name 参数）
2. 解析 skill 定义
3. 注入能力说明；可能 spawn 子 workflow
4. 结果作为 user `tool_result` → **L1 loop continue**

---

## 3. Plugin 系统

### 3.1 是什么

Plugin = **可安装的分发包**，从 marketplace 或 git 拉取，启用后向运行时注入多种 **组件**。

**不是** OpenCode 的 npm plugin（TS 模块热加载）；Claude Code plugin 是 **目录 + manifest + 组件扫描**。

### 3.2 目录结构（官方约定）

`utils/plugins/pluginLoader.ts` 文件头注释：

```text
my-plugin/
├── plugin.json          # manifest（metadata、组件路径）
├── commands/            # slash commands (*.md)
├── agents/              # 自定义 agent 定义
├── skills/              # skill 包
├── hooks/               # hooks.json
└── (可选 MCP / LSP 声明于 manifest)
```

### 3.3 `LoadedPlugin` 类型

`types/plugin.ts`：

| 字段 | 含义 |
|------|------|
| `name` / `source` / `path` | 标识与磁盘路径 |
| `manifest` | 完整 PluginManifest |
| `commandsPath(s)` | slash 命令 |
| `agentsPath(s)` | AgentTool 可用 agent |
| `skillsPath(s)` | skill 目录 |
| `hooksConfig` | HooksSettings |
| `mcpServers` | MCP server 配置 |
| `lspServers` | LSP 配置 |
| `outputStylesPath(s)` | 输出样式 |
| `isBuiltin` | 内置 plugin（`@builtin`） |

`PluginComponent` 枚举：`commands` | `agents` | `skills` | `hooks` | `output-styles`。

### 3.4 发现与加载顺序

`pluginLoader.ts`：

1. Marketplace plugins（`plugin@marketplace` in settings）
2. Session-only（`--plugin-dir` / SDK `plugins` option）
3. Builtin plugins（`plugins/builtinPlugins.ts`）

流程：

```text
loadAllPluginsCacheOnly
  → validate manifest (utils/plugins/schemas.ts)
  → 解析各组件路径
  → enabledPlugins 过滤
  → loadPluginHooks / loadPluginCommands / MCP integrate / ...
```

Headless：`print.ts` → `installPluginsAndApplyMcpInBackground` **在首次 ask() 前**。

REPL：启动时 + `/plugin` / `/reload-plugins` 热更新。

### 3.5 Marketplace

- 官方 marketplace 名：`claude-plugins-official` 等（`officialMarketplace.ts`）
- `/plugin install frontend-design@claude-plugins-official`
- 安全：`BLOCKED_OFFICIAL_NAME_PATTERN` 防 impersonation
- autoupdate：官方 marketplace 默认开启（部分除外）

---

## 4. Hooks 与 Plugin 的关系

**Plugin 可以携带 Hooks，但 Hooks 不必来自 Plugin。**

```text
用户 settings.json
  hooks.PreToolUse: [{ type: command, command: "..." }]

Plugin manifest
  hooks: { Stop: [{ matcher: "...", hooks: [...] }] }

loadPluginHooks.ts
  → convertPluginHooksToMatchers(plugin)
  → registerHookCallbacks (bootstrap/state)
  → 与用户 hooks 合并进同一 registry
```

Plugin hook 额外带 `pluginRoot` / `pluginName` — 便于解析相对路径与 telemetry。

执行引擎统一：`utils/hooks/`（command / prompt / HTTP）。

---

## 5. Plugin 还能扩展什么

| 组件 | 扩展能力 | 进入运行时 |
|------|----------|------------|
| **commands/** | 新 slash 命令 | `processUserInput` |
| **agents/** | 新 subagent 类型 | `AgentTool` |
| **skills/** | 领域 workflow | `SkillTool` + discovery |
| **hooks** | 生命周期策略 | [11](./11-permission-and-hooks.md) |
| **mcpServers** | 外部 API/tool | [14 MCP](./14-mcp-and-external-protocols.md) |
| **lspServers** | 语言服务器 | [16 LSP](./16-lsp-and-code-intelligence.md) |
| **output-styles** | 输出格式 | system prompt 动态段 |

Builtin 示例：`plugins/bundled/` — 随 CLI 分发，用户可在 `/plugin` UI 开关。

---

## 6. 与 MCP 的关系

三条路径最终都汇入 `AppState.mcp` → `assembleToolPool`：

1. 用户 `.mcp.json` / settings
2. Plugin manifest `mcpServers`
3. Skill frontmatter embedded MCP（tier-3 概念，官方 docs）

Late connect 影响 prompt cache 与 `hasPendingMcpServers`（[14 §3](./14-mcp-and-external-protocols.md#3-与-appstate-的集成)）。

---

## 7. 与 OpenCode / OmO 对照（后期）

| | Claude Code Plugin | OmO / OpenCode |
|--|-------------------|----------------|
| 分发 | marketplace zip/git | npm package |
| 运行时 | 组件扫描 + manifest | TS plugin module |
| MCP | manifest 声明 | .mcp.json + skill-embedded |
| Hooks | shell/HTTP in manifest | TS hook handlers |

---

## 8. 读源码顺序

1. `types/plugin.ts` — 数据模型
2. `utils/plugins/pluginLoader.ts` — 发现/加载
3. `utils/plugins/loadPluginHooks.ts` — hooks 合并
4. `services/plugins/PluginInstallationManager.ts` — 安装 UI/CLI
5. `tools/SkillTool/` — skill 调用
6. `cli/print.ts` headless plugin 安装时机

---

## 9. 自测

- [ ] Skill vs Plugin vs MCP 各解决什么问题？
- [ ] Plugin Manifest 可声明哪几类组件？
- [ ] Plugin hooks 与用户 settings hooks 如何合并？
- [ ] headless 何时安装 plugin？
- [ ] compact 后 skill 上下文如何保留？
- [ ] 与 OpenCode npm plugin 的本质区别？

**关联：** [11 Permission/Hooks](./11-permission-and-hooks.md) · [14 MCP](./14-mcp-and-external-protocols.md) · [04 Config](./04-config-and-settings.md) · [12 Commands](./12-commands-and-input-preprocessing.md)
