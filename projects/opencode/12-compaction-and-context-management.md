# 12 · Compaction 与上下文管理

> **核心问题：** 上下文满了怎么办？压缩前后插件能保留什么？

---

## 1. 为什么需要 Compaction

长会话的 messages 会超过 model context window。**runLoop** 检测到 overflow 后进入 compaction 分支，用 **摘要替代旧历史**，然后继续对话。

文件：[`session/compaction.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/compaction.ts)

---

## 2. 流程

```mermaid
flowchart LR
  O[overflow 检测] --> H[experimental.session.compacting]
  H --> L[Compaction LLM 摘要]
  L --> R[替换/裁剪 messages]
  R --> A[experimental.compaction.autocontinue]
  A -->|true| LOOP[回到 runLoop]
```

**Compaction agent prompt：** [`agent/prompt/compaction.txt`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/agent/prompt/compaction.txt)

压缩本身是一次 **额外的 LLM 调用**（通常用较快/便宜 model，受 config 影响）。

---

## 3. 相关 Hook

| Hook | 时机 | 典型用途 |
|------|------|----------|
| `experimental.session.compacting` | 压缩开始前 | 注入须保留的状态（todo、外部任务 ID 等） |
| `experimental.compaction.autocontinue` | 压缩成功后 | 是否自动 continue |
| `experimental.chat.messages.transform` | 压缩路径也会走 | 统一改 messages |

插件典型用法：在 **compacting** 把不可丢失的上下文写进压缩输入，而不是仅依赖 transform。

---

## 4. Instruction 与其它上下文源

除 messages 外，上下文还包括：

| 来源 | 模块 |
|------|------|
| 项目 instructions | [`session/instruction.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/instruction.ts) |
| Agent system | [`agent/agent.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/agent/agent.ts) |
| Skill 内容 | 按需经 skill tool 加载 |
| config `instructions` | merge 进 instruction |

Compaction **主要动 messages**；instructions 通常保留。

---

## 5. config 域

[`config/config.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/config/config.ts) 内 `compaction` 相关字段控制：

- 是否启用 / 阈值
- 用于摘要的 model
- autocontinue 默认

具体键以 schema 为准（读 config Info struct）。

---

## 6. 插件边界

- 不能关闭 overflow 检测
- 可通过 compacting / autocontinue hook **影响压缩内容与是否继续**
- 不能替换 compaction 摘要算法本身（在内核 `compaction.ts`）

---

## 读完后应能回答

- [ ] compaction 在 runLoop 哪条分支？
- [ ] 两个 experimental compaction hook 区别？
- [ ] 为何 compacting 比单独 messages.transform 更适合留状态？

→ **下一篇：** [13 · Bus 与 Session 事件](./13-bus-and-session-events.md)
