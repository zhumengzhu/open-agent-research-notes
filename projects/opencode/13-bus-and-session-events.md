# 13 · Bus 与 Session 事件

> **核心问题：** 内核如何广播 session 生命周期？插件 `event` hook 与 `Plugin.trigger` 有何不同？

---

## 1. Bus 模型

[`bus/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/bus/index.ts)

- 进程内 **pub/sub**
- `Bus.publish(EventType, payload)`
- `Bus.subscribeAll(fn)` — plugin init 时注册

**特点：** 异步、fire-and-forget；**不** mutate 返回值（与 trigger 对比）。

---

## 2. 常见 Session 事件

| 事件（概念名） | 含义 | 典型发布点 |
|----------------|------|------------|
| session.created | 新会话 | Session.create |
| session.idle | 一轮 runLoop 结束、无 pending | prompt/runLoop 末尾 |
| session.error | 模型/运行时错误 | processor / retry |
| session.deleted | 会话删除 | API |
| message.updated | part 增量更新 | processor 流式 |
| permission.asked | 需用户批准 | permission |

精确类型见 core / session 事件 schema（[`packages/core`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/core) 与 opencode 发布点）。

---

## 3. plugin.event  wiring

[`plugin/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/plugin/index.ts) init：

```typescript
bus.subscribeAll(async (event) => {
  for (const hook of hooks) {
    await hook.event?.({ event })
  }
})
```

**每个插件** 都会收到 **全量** 事件；自行 filter `event.type`。

---

## 4. trigger vs event

| | Plugin.trigger | Bus → event |
|--|----------------|-------------|
| 同步性 | await 串行 | 异步广播 |
| 改状态 | mutate output | 只读 event |
| 用途 | 改 params/tool | idle 后续、通知、fallback |
| 失败 | 中断当前操作 | 通常 log，不阻断内核 |

---

## 5. 典型插件用法

| 事件 | 插件可做什么 |
|------|----------------|
| `session.idle` | 自动续推、通知外部系统 |
| `session.error` | 换 model 重试、告警 |
| `message.updated` | 同步 UI / 日志 |

**注意：** idle 后续逻辑在 **runLoop 之外** 触发；与 runLoop 内的 `chat.message` 不同。

---

## 6. sync 模块

[`sync/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/sync) 将部分事件 **持久化** 供多端同步（与 Bus 互补）；学 agent 主链路时可后读。

---

## 7. Instance.bind 再次强调

原生 watcher 回调里 publish 事件需 [`Instance.bind`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/instance.ts)，否则 directory 上下文丢失。

---

## 读完后应能回答

- [ ] idle 事件何时发布？
- [ ] 为何 continuation 用 event 而非 chat.message？
- [ ] permission.asked 与 permission.ask hook 关系？

→ **下一篇：** [14 · MCP / LSP / Skill / Command](./14-mcp-lsp-skill-and-command.md)
