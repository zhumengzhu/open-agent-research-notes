# 06 · Hook 系统完整参考

> **核心问题：** 每个 hook 何时触发、input/output 是什么、能否 mutate？

契约：[`packages/plugin/src/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/plugin/src/index.ts)  
引擎：[`packages/opencode/src/plugin/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/plugin/index.ts)（`Plugin.trigger` 串行 mutate `output`）

---

## 1. 触发型 Hook

| Hook | 触发文件 | output 要点 |
|------|----------|-------------|
| `chat.message` | [`prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts) | `{ message, parts }` |
| `chat.params` | [`llm/request.ts` #L106](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/llm/request.ts#L106) | temperature, topP, topK, maxOutputTokens, **`options`** |
| `chat.headers` | [`llm/request.ts` #L126](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/llm/request.ts#L126) | `{ headers }` |
| `command.execute.before` | [`prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts) | `{ parts }` |
| `tool.execute.before` | [`prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts), [`tools.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/tools.ts) | `{ args }` |
| `tool.execute.after` | 同上 | `{ title, output, metadata }` |
| `shell.env` | [`tool/shell.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/tool/shell.ts), `pty/` | `{ env }` |
| `tool.definition` | [`tool/registry.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/tool/registry.ts) | description, parameters |
| `experimental.chat.messages.transform` | [`prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts), [`compaction.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/compaction.ts) | `{ messages }` |
| `experimental.chat.system.transform` | [`llm/request.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/llm/request.ts) | `{ system: string[] }` |
| `experimental.session.compacting` | [`compaction.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/compaction.ts) | compact 输入 |
| `experimental.compaction.autocontinue` | 同上 | `{ continue: boolean }` |
| `experimental.text.complete` | [`processor.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/processor.ts) | text part |

`chat.params` 字段详解：[10](./10-llm-stream-and-provider.md)。

---

## 2. 注册型 Hook

| Hook | 机制 |
|------|------|
| `config` | init 时 `hook.config(cfg)`，可改内存 config |
| `event` | `bus.subscribeAll` → 只读 event |
| `tool` | `{ [name]: ToolDefinition }` 并入 registry |
| `auth` | Provider 登录 UI |
| `provider` | 动态 models / Provider 工厂 |

---

## 3. 主链路时序

见 [18 · 主链路总图](./18-main-chain-atlas.md) §3。

---

## 4. 未接线（baseline）

| Hook | 状态 |
|------|------|
| `permission.ask` | SDK 有类型；内核走 [`permission/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/permission/index.ts) + Bus `permission.asked`，**无** `Plugin.trigger` |

---

## 5. 选型指南

| 目标 | Hook |
|------|------|
| 改模型参数 / provider options | `chat.params` |
| 改历史 messages | `experimental.chat.messages.transform` |
| 改 system | `experimental.chat.system.transform` |
| 拦 / 改 tool | `tool.execute.before` / `after` |
| 听 session idle | `event` |
| 加工具 | `tool` 字典 |
| 压缩前保留状态 | `experimental.session.compacting` |

---

## 读完后应能回答

- [ ] trigger 与 event 区别？
- [ ] chat.params 与 messages.transform 分工？
- [ ] 多插件同 hook 时 output 如何传递？

→ **下一篇：** [07 · Agent 与 Permission](./07-agent-and-permission.md)
