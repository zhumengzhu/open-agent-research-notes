# 08 · 工具注册与门控

> **核心问题：** 20–39 个 OmO 工具是怎么按配置启用/禁用的？想给独立插件暴露自定义 tool 给 LLM 调用，怎么做？
>
> 想写"暴露新 tool 给 LLM"类的插件时必读。

---

## 1. OpenCode 工具的本质

来自 `@opencode-ai/plugin` 的 `ToolDefinition` 类型。**对 LLM 来说，工具就是一个带名字、JSON Schema 参数、和异步执行函数的对象**。

简化结构：

```typescript
type ToolDefinition = {
  description: string             // 给 LLM 看的，告诉它什么时候用
  args: ZodSchema | JsonSchema    // 参数验证
  execute: (args, ctx) => Promise<string | object>
}
```

OmO 插件通过实现 `tool` 这个 OpenCode hook 返回 `Record<string, ToolDefinition>`：

```typescript
return {
  tool: {
    my_search: {
      description: "Search the web",
      args: z.object({ query: z.string() }),
      execute: async ({ query }) => `Results for ${query}: ...`,
    },
    // ...
  },
  // 其他 11 个 hook
}
```

→ OpenCode 拿到这个 record 后，把每个 tool 加进 LLM 的 tools 列表，LLM 想用就用 tool_use 块调用。

## 2. OmO 工具清单：20 always-on + 最多 19 conditional

详见 [`src/tools/AGENTS.md`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/src/tools/AGENTS.md) 的完整表。配置门控 5 个开关：

| 门控 | 配置 | 加几个工具 |
|------|------|----------|
| `look_at` | 不在 `disabled_agents` 里有 `multimodal-looker` | +1 |
| `interactive_bash` | `isInteractiveBashEnabled(config)`（tmux 启用） | +1 |
| `task_create/get/list/update` | `experimental.task_system` | +4 |
| `edit` (hashline-edit) | `hashline_edit: true` | +1 |
| 12 个 `team_*` | `team_mode.enabled: true` | +12 |

总最大：20 + 1 + 1 + 4 + 1 + 12 = **39 个**。

## 3. 注册主流程

完整代码在 [`src/plugin/tool-registry.ts`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/src/plugin/tool-registry.ts) 的 `createToolRegistry()`：

```mermaid
flowchart TB
  S[Start] --> A[准备 factory 表<br/>defaultToolRegistryFactories]
  A --> B[构造 always-on 工具<br/>builtinTools + grep + glob + ast-grep + session + background + call_omo + skill + task]
  B --> C{逐个门控判断<br/>(5 个 conditional gate)}
  C --> D[合并 allTools<br/>spread + spread + spread]
  D --> E[normalizeToolArgSchemas<br/>统一 arg schema 形状]
  E --> F[filterDisabledTools<br/>剔除 disabled_tools 里的]
  F --> G{max_tools 配置?}
  G -->|是| H[trimToolsToCap<br/>按 LOW_PRIORITY 顺序裁]
  G -->|否| I[返回 ToolRegistryResult]
  H --> I
```

### 关键代码片段

#### 3.1 条件门控（5 个）

```294:339:src/plugin/tool-registry.ts
const taskSystemEnabled = isTaskSystemEnabled(pluginConfig)
const taskToolsRecord: Record<string, ToolDefinition> = taskSystemEnabled
  ? {
      task_create: factories.createTaskCreateTool(pluginConfig, ctx),
      task_get: factories.createTaskGetTool(pluginConfig),
      task_list: factories.createTaskList(pluginConfig),
      task_update: factories.createTaskUpdateTool(pluginConfig, ctx),
    }
  : {}

const hashlineEnabled = pluginConfig.hashline_edit ?? false
const hashlineToolsRecord: Record<string, ToolDefinition> = hashlineEnabled
  ? { edit: factories.createHashlineEditTool(ctx) }
  : {}

const teamModeToolsRecord: Record<string, ToolDefinition> = pluginConfig.team_mode?.enabled
  ? {
      team_create: factories.createTeamCreateTool(...),
      team_delete: factories.createTeamDeleteTool(...),
      // ... 12 个 team_* 工具
    }
  : {}
```

→ **门控只在装配阶段判断一次**，运行时不再 if-else。LLM 看到的就是已经过滤好的工具列表。

#### 3.2 合并 + 过滤

```341:368:src/plugin/tool-registry.ts
const allTools: Record<string, ToolDefinition> = {
  ...factories.builtinTools,             // 6 LSP
  ...factories.createGrepTools(ctx),
  ...factories.createGlobTools(ctx),
  ...factories.createAstGrepTools(ctx),
  ...factories.createSessionManagerTools(ctx),
  ...backgroundTools,
  call_omo_agent: callOmoAgent,
  ...(lookAt ? { look_at: lookAt } : {}),
  task: delegateTask,
  skill_mcp: skillMcpTool,
  skill: skillTool,
  ...(interactiveBashEnabled ? { interactive_bash: factories.interactive_bash } : {}),
  ...teamModeToolsRecord,
  ...taskToolsRecord,
  ...hashlineToolsRecord,
}

for (const toolDefinition of Object.values(allTools)) {
  normalizeToolArgSchemas(toolDefinition)
}

const filteredTools: ToolsRecord = filterDisabledTools(allTools, pluginConfig.disabled_tools)

const maxTools = pluginConfig.experimental?.max_tools
if (maxTools) {
  trimToolsToCap(filteredTools, maxTools)
}
```

### 3.3 `max_tools` 裁剪策略

```126:154:src/plugin/tool-registry.ts
const LOW_PRIORITY_TOOL_ORDER = [
  "session_list",
  "session_read",
  "session_search",
  "session_info",
  "interactive_bash",
  "look_at",
  "call_omo_agent",
  "task_create", "task_get", "task_list", "task_update",
  "background_output", "background_cancel",
  "edit",
  "ast_grep_replace", "ast_grep_search",
  "glob", "grep",
  "skill_mcp", "skill", "task",
  "lsp_rename", "lsp_prepare_rename", "lsp_find_references",
  "lsp_goto_definition", "lsp_symbols", "lsp_diagnostics",
] as const
```

→ **被裁掉的工具按"低优先级在前"的顺序删除**。某些 provider（如旧 OpenAI API）对 tools 数量有上限，OmO 配置 `experimental.max_tools` 后会先裁 session_* 这类辅助工具，保住 LSP / edit 这类核心。

## 4. 三种工具的代码组织模式

### 4.1 Direct ToolDefinition（最简单）

LSP 6 个工具和 `interactive_bash` 是直接导出的 `ToolDefinition` 对象，没有工厂：

```typescript
// src/tools/lsp/...
export const builtinTools = {
  lsp_goto_definition: { description, args, execute },
  lsp_find_references: { ... },
  // ...
}
```

适用：**完全无状态、不需要 `ctx` / config 注入**。

### 4.2 Factory 单工具

大多数工具：

```typescript
// src/tools/hashline-edit/index.ts
export function createHashlineEditTool(ctx: PluginContext): ToolDefinition {
  return {
    description: "...",
    args: z.object({ ... }),
    execute: async (args) => {
      // 用 ctx.client 反向调 OpenCode API
    },
  }
}
```

适用：**需要 `ctx` 注入，但每个工具独立**。

### 4.3 Factory 工具集合

某些"工具家族"一起导出：

```typescript
// src/tools/grep/index.ts
export function createGrepTools(ctx: PluginContext): Record<string, ToolDefinition> {
  return {
    grep: { ... },
  }
}
```

适用：**多个 tool 共享内部 helper 或状态**。

## 5. `disabled_tools` 配置

用户在配置里：

```jsonc
{
  "disabled_tools": ["interactive_bash", "background_output"]
}
```

`filterDisabledTools` 会跑：

```typescript
function filterDisabledTools(allTools, disabled) {
  if (!disabled) return allTools
  const result = { ...allTools }
  for (const name of disabled) delete result[name]
  return result
}
```

→ **OmO 是允许任意 tool 被禁用的**，包括 LSP 这种核心。但裁太多 LLM 会变笨，所以 doctor check 会警告。

## 6. `normalizeToolArgSchemas` 干什么

OmO 的 tool args 有时用 Zod schema，有时是 JSON schema 对象，有时缺字段。`normalizeToolArgSchemas` 把它们统一成 OpenCode 期待的形状：

```typescript
// src/plugin/normalize-tool-arg-schemas.ts (你没读，但思路如下)
function normalizeToolArgSchemas(tool: ToolDefinition): void {
  // 1. 如果 args 是 Zod，留着
  // 2. 如果是 plain JSON schema，补全 $schema/type/properties
  // 3. 如果缺 description，从 tool.description 拷贝
}
```

→ **这层防御是因为 tool 工厂可能由不同人写、风格不一**。你独立插件如果只写一两个工具不需要这层。

## 7. 你独立插件暴露自定义工具的最小例子

```typescript
import type { Plugin } from "@opencode-ai/plugin"
import { z } from "zod"

const serverPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      // 工具名（snake_case），LLM 用这个名字调
      my_calculator: {
        description: "Evaluate a mathematical expression and return the result.",
        args: z.object({
          expression: z.string().describe("e.g. '2 + 3 * 4'"),
        }),
        execute: async ({ expression }) => {
          try {
            // 注意：实际生产代码不能用 eval，这只是演示
            const result = Function(`"use strict"; return (${expression})`)()
            return String(result)
          } catch (err) {
            return `Error: ${err}`
          }
        },
      },
    },
  }
}

export default { id: "my-calculator-plugin", server: serverPlugin }
```

→ **就这么简单**。LLM 看到 `my_calculator` 出现在工具列表里，需要计算时就会调它。

## 8. 关键 do / don't

| Do | Don't |
|----|-------|
| 用 snake_case 工具名 | camelCase 或 kebab-case |
| description 写清"什么时候用 / 不该什么时候用" | description 只写"do X" |
| args 用 Zod 配 `.describe()` | 纯 TypeScript 类型（LLM 看不到） |
| execute 返回字符串或可序列化对象 | 返回 Promise、function、Symbol |
| 异常用 try/catch 包，return 错误信息 | throw 出去（OpenCode 会显示丑陋 stack） |
| 工具间命名冲突时，给自己加 namespace 前缀（`zmz_calculator`） | 用通用名（如 `search`）— 容易和别人冲突 |

## 9. 高级：暴露 tool 给特定 agent

OmO 用 `AGENT_TOOL_RESTRICTIONS` 限制特定 agent 能用哪些 tool（例如 `oracle` 不能用 `write`/`edit`）：

```typescript
// src/shared/agent-tool-restrictions.ts (思路)
export const AGENT_TOOL_RESTRICTIONS = {
  oracle: { denied: ["write", "edit", "task", "call_omo_agent"] },
  multimodal_looker: { allowed: ["read"] },  // allowlist
  // ...
}
```

→ **OpenCode 自己没有"per-agent tool 白名单"机制**，这是 OmO 自己加的层（通过运行时 chat.params 阶段动态注入 disabled tools）。你独立插件想做类似的事需要在 `chat.message` 或 `chat.params` hook 里读 `input.agent.name` 决定要不要让某 tool 出现 —— 但这需要黑魔法（OpenCode 没暴露 "运行时禁用某 tool" 的 API）。

简单做法：**你的工具自己在 execute 内判断 agent 名**：

```typescript
execute: async (args, ctx) => {
  // OpenCode SDK 可以读当前 agent，但需要看 ctx 文档
  // 简化做法：在 LLM prompt 里告诉它"oracle 不要用此工具"
  return "..."
}
```

---

## 读完后应该能回答

- [ ] 一个 OpenCode 工具的最小定义是什么？
- [ ] OmO 5 个 conditional 工具门控分别是什么配置？
- [ ] `max_tools` 配置怎么决定先裁哪些工具？
- [ ] direct ToolDefinition、factory 单工具、factory 工具集合三种模式分别什么时候用？
- [ ] LLM 看到一个新工具，靠什么知道"什么时候该用"？

---

→ **下一篇：** [09 · Agent 系统](./09-agent-system.md)
