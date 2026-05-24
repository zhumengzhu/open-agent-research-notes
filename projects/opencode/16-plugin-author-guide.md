# 16 · 插件作者指南

> **核心问题：** 如何快速写出正确、可调试的 OpenCode 插件？

---

## 1. 最小插件

```typescript
import type { Plugin } from "@opencode-ai/plugin"

const plugin: Plugin = async () => ({
  "chat.params": async (input, output) => {
    if (input.model.providerID !== "deepseek") return
    output.options = { ...output.options, enable_thinking: false }
  },
})

export default { id: "my-plugin", server: plugin }
```

依赖：仅 `@opencode-ai/plugin`（集成测可加 `@opencode-ai/sdk`）。

---

## 2. 安装

在项目或 `~/.config/opencode/` 的 config：

```jsonc
{ "plugin": ["file:///absolute/path/to/plugin.ts"] }
```

文档：[plugins.mdx](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/web/src/content/docs/plugins.mdx)

**改 plugin 列表后需重启实例**（重新 bootstrap）。

---

## 3. Hook 选型

| 目标 | Hook | 详解 |
|------|------|------|
| 模型参数 | `chat.params` | [10](./10-llm-stream-and-provider.md) |
| HTTP 头 | `chat.headers` | [10](./10-llm-stream-and-provider.md) |
| 历史 messages | `experimental.chat.messages.transform` | [06](./06-hook-system-reference.md) |
| system | `experimental.chat.system.transform` | [06](./06-hook-system-reference.md) |
| 拦 tool | `tool.execute.before` | [11](./11-tool-registry-and-execution.md) |
| 改 tool 输出 | `tool.execute.after` | [11](./11-tool-registry-and-execution.md) |
| idle / error | `event` | [13](./13-bus-and-session-events.md) |
| 注册工具 | `tool` | [05](./05-plugin-protocol-and-loader.md) |

---

## 4. `chat.params` 要点

- **每轮 LLM 调用都触发**（含 tool 后续聊）→ 用 `sessionID` 存状态或保持无状态
- Provider 特定字段写 **`output.options`**
- 用 spread 合并，避免抹掉其它插件写的键
- 按 `input.model.providerID` / `modelID` 过滤

---

## 5. 调试

| 手段 | 说明 |
|------|------|
| `opencode run "..."` | 非交互复现 |
| `opencode serve` + SDK | 脚本调 session API |
| 减少 plugin 数量 | 隔离 hook 冲突 |
| 对照 [18 总图](./18-main-chain-atlas.md) | 确认注入点 |

---

## 6. 反模式

| 不要 | 原因 |
|------|------|
| import `packages/opencode/src/*` | 非公开 API |
| 依赖 `permission.ask` hook | baseline 未 trigger |
| 在 `event` 里长阻塞 | 拖慢 Bus |
| 假设 hook 只调一次 | chat.params 每 turn 都调 |

---

## 7. 进阶

1. 读 [09 runLoop](./09-session-prompt-runloop.md) 标出你的 hook 在链路上的位置  
2. 需要 agent / MCP 时读 [04 config](./04-config-system.md) + [05 插件](./05-plugin-protocol-and-loader.md)  
3. 验收：[17](./17-architecture-review-and-mastery.md)

---

## 读完后应能回答

- [ ] 最小 export 结构？
- [ ] thinking 开关用哪个 hook、写哪个字段？
- [ ] 多插件同 hook 冲突怎么排查？

→ **下一篇：** [17 · 架构评价与掌握验收](./17-architecture-review-and-mastery.md)
