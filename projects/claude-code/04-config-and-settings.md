# 04 · 配置与 Settings

> **锚点：** `utils/config.ts` · `utils/settings/` · `schemas/` · `migrations/` · `bootstrap/state.js`

---

## 1. 配置来源与合并

Walked + merge（**closer 优先**），典型层级：

| 层级 | 路径（概念） | 备注 |
|------|--------------|------|
| 项目 | `.claude/settings.json`、`.claude/settings.local.json` | local 不 commit |
| 用户 | `~/.claude/settings.json` | 全局默认 |
| CLI | `--settings` JSON 文件 | 一次性 override |
| 环境变量 | `ANTHROPIC_*`、`CLAUDE_CODE_*` | 见 §3 |
| Managed | 企业下发 settings | headless 订阅 [19] |

`getGlobalConfig()` / `getCurrentProjectConfig()` — `utils/config.ts`  
**AppState** 持有 denormalized 快照 — runtime **真相** [01]。

### 1.1 安全边界

- **project settings 不可** 设某些危险路径（如 `autoMemoryDirectory` — 防恶意 repo [29]）
- MCP env 展开有 allowlist（`mcp_env_allowlist` 概念，对照 OmO [14]）

---

## 2. Settings 与 AppState 同步

| 模式 | 机制 |
|------|------|
| REPL | `useSettingsChange` hook → `setAppState` |
| Headless | `settingsChangeDetector.subscribe` → `applySettingsChange` [19] |

同步字段示例：`fastMode`、`permissionMode`、MCP 连接态、effort、model override。

**Hooks snapshot：** `hooksConfigSnapshot.ts` — 启动 capture，变更 update [11]。

---

## 3. 环境变量速查（与 settings 并列）

| 变量 | 作用 | 文档 |
|------|------|------|
| `CLAUDE_CODE_SIMPLE` | bare：关 hooks/LSP/memory/plugin | [03] |
| `CLAUDE_CODE_REMOTE` | remote worker | [22] |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | CCR memory 挂载 | [29] |
| `CLAUDE_CODE_COORDINATOR_MODE` | coordinator | [21] |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 外部开 swarm | [21] |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 关 auto-memory | [29] |
| `CLAUDE_CODE_PROACTIVE` | proactive 行为 | [30] |
| `ENABLE_TOOL_SEARCH` | tool search 手动/auto:N | [30] |
| `ANTHROPIC_MODEL` | 默认模型 override | [27] |

Env 与 settings **冲突时**以各模块文档为准（memory 有独立优先级链 [29]）。

---

## 4. Agent 行为相关键

| 配置 | 影响 |
|------|------|
| `autoCompactEnabled` | proactive autocompact [10] |
| `model` | 主循环默认 [27] |
| `autoMemoryEnabled` | memdir / extractMemories [29] |
| `autoMemoryDirectory` | 仅 user/local 覆盖路径 |
| `enabledPlugins` | 插件加载 [15] |
| permission / allow rules | [11] |
| `outputStyle` | system prompt 动态段 [13] |
| `fastMode` | API fast mode [27] |
| hooks 配置 | [11] |

具体 schema：`schemas/` + `migrations/`（版本演进、idempotent migrate）。

---

## 5. Feature flags 双层门控

```text
bun:bundle feature('NAME')     → 编译期：代码是否存在
GrowthBook / Statsig gate      → 运行时 killswitch / 实验
env / CLI opt-in               → 外部用户显式开启
```

示例：

- Agent Teams：`isAgentSwarmsEnabled()` [21]
- extractMemories：`tengu_passport_quail` [29]
- TOKEN_BUDGET：`feature('TOKEN_BUDGET')` [07]

外部 snapshot 可能 **整段 DCE** — 读源码时看 `feature()` 守卫。

---

## 6. Migrations

`migrations/` — settings 字段重命名、默认值填充；启动时 apply，带 `_migrations` 跟踪（概念同 OmO migrate）。

---

## 7. 与 QueryEngine 初始化

`setup.ts` / `print.ts` 启动链读取 settings → 构造：

- Tool pool deny/allow
- MCP config paths
- LSP 是否 init（非 bare）
- Worktree 默认行为 [23]
- Model / thinking 默认值

---

## 8. 自测

- [ ] settings 与 AppState 谁是 runtime 真相？
- [ ] headless 如何监听 settings 变更？
- [ ] 关 autocompact 的 settings 键 vs env？
- [ ] project settings 为何不能设 autoMemoryDirectory？
- [ ] feature() 与 GrowthBook 双层门控举例？

**关联：** [11 Permission](./11-permission-and-hooks.md) · [15 Plugins](./15-skills-and-plugins.md) · [29 Memory](./29-memory-and-auto-memory.md) · [30 实验门控](./30-advanced-features-and-experiments.md)
