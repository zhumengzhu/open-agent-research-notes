# 附录 A1 · 一条用户消息的完整叙事（Trace）

> **读法：** 跟着故事走一遍，再对照 [18 总图](../18-main-chain-atlas.md) 和源码。  
> **跟码建议：** checkout baseline `7fe7b9f2`，或 **搜 Effect.fn 名**（行号可能随版本变）。

**场景：** 你在 TUI 里用 `build` agent、模型 `anthropic/claude-sonnet-4`，输入：「读 README 并总结」，模型决定调用 `read` 工具，然后给出文字总结。

---

## 第一幕：请求进入内核

1. **TUI** 把消息 POST 到本地 HTTP（或 in-process SDK），header 带 `x-opencode-directory: /your/project`。
2. **[`handlers/session.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts)** 收到 prompt 请求。
3. **[`InstanceStore`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/instance-store.ts)** 按 directory 取实例；若首次访问，跑 **[`bootstrap`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/bootstrap.ts)**：`plugin.init()` → 其它服务 init。
4. 调用 **`SessionPrompt.prompt()`**。

此时还没有 LLM 调用；只是实例就绪。

---

## 第二幕：User Message 落库

5. **`SessionPrompt.createUserMessage`**（搜此 Effect.fn）：
   - 解析 **agent**：请求里的 `build`，或 default agent。
   - 解析 **model**（优先级见 [19 多模型](../19-multi-model-and-provider-system.md)）：
     - 请求显式 `model` → 用之；
     - 否则 agent 绑定的 `agent.model`；
     - 否则 session 表 / 上一条 user 的 model；
     - 否则 `Provider.defaultModel()`（config `model` → recent → 首个 provider 排序）。
   - 解析 **variant**（如 reasoning effort）：请求 variant → agent.variant（若 model 匹配）。
6. 组装 **`MessageV2.User`**：`agent`、`model: { providerID, modelID, variant }`、text parts。
7. **`Plugin.trigger("chat.message")`** — 插件可改 message/parts（首条变体、关键词等）。
8. 写入 SQLite **`message` + `part` 表**（见 [08](../08-session-message-and-storage.md)）。
9. 若 agent/model 相对 session 有变化，发 **AgentSwitched / ModelSwitched** 同步事件。

**此时 DB 里多了一条 user 消息，但 assistant 还没有。**

---

## 第三幕：runLoop 启动

10. **`SessionRunState.ensureRunning(sessionID, …, runLoop)`** — 保证本 session 只有一个 loop 在跑。
11. **`SessionPrompt.run`**（`while true`）开始 **step 1**：
    - `MessageV2.filterCompactedEffect` 读 history。
    - 扫描 `lastUser`、`lastFinished`、pending 的 subtask/compaction part → 本例均无。
    - **退出判定**不满足 → 继续。
12. **fork 异步 `ensureTitle`**：用 **small model**（`getSmallModel` 或 title agent 的 model）生成会话标题 —— **这是同 session 内第二次潜在的 LLM 调用，与用户主对话并行**（见 [19 §5](../19-multi-model-and-provider-system.md)）。

---

## 第四幕：第一轮 LLM（决定读 README）

13. **`agents.get(lastUser.agent)`** → `build` 的 permission、steps、prompt。
14. **新建空 assistant message** 行（DB 先占位）。
15. **`SessionProcessor.create`** + **`resolveTools`**：
    - 合并 builtin + MCP + plugin tools；
    - 按 agent permission 过滤 → `read` 在列表里。
16. **`experimental.chat.messages.transform`** — 插件可改整段 history。
17. **`MessageV2.toModelMessagesEffect`** + system / skills / instructions。
18. **`handle.process(streamInput)`** 进入 processor（详见 [A2 Processor 深读](./A2-processor-deep-dive.md)）。

### 在 llm/request 里（第一次 chat.params）

19. 拼 system → **`experimental.chat.system.transform`**。
20. merge **ProviderTransform** 默认 options + agent.options + **variant options**。
21. **`Plugin.trigger("chat.params")`** — 你写的 thinking-toggle / effort 插件在这里改 `output.options`。
22. **`Plugin.trigger("chat.headers")`**。
23. **`streamText`** → Anthropic API（经 `@ai-sdk/anthropic` 等）。

### Processor 消费流

24. `start-step` → 写 step-start part，记 snapshot。
25. `reasoning-*`（若模型支持）→ reasoning parts。
26. `text-delta` / `text-end` → 可见文本 part；`text-end` 时 **`experimental.text.complete`**。
27. `tool-input-start` → 创建 **pending tool part**（`read`）。
28. **`tool-call`** → part 变 **running**，input = `{ file_path: "README.md" }`。
29. `finish-step` → 写 finishReason（可能是 `tool-calls` 或 `stop`）、token 用量、step-finish part。
30. **`process()` 返回 `"continue"`**（还有 tool 要执行）。

---

## 第五幕：执行 read 工具

31. runLoop 看到 **hasToolCalls** 或 finish 为 tool-calls → **不 break**。
32. **Processor / tools 路径** 执行 `read`：
    - **`tool.execute.before`**（可拦/改 args）；
    - Permission check；
    - **`ReadTool.execute`** → 读磁盘；
    - **`tool.execute.after`**（truncator 等改 output）；
    - tool part → **completed**，结果写入 part.state.output。
33. **`runLoop` 回到 while 顶部** — step 2。

---

## 第六幕：第二轮 LLM（读结果后总结）

34. 再次读 history —— 现在包含：**user → assistant(tool-call) → tool result 已合并进 messages**。
35. 再建 **新的 assistant message**（新一轮 assistant 行）。
36. 再次 transform → toModelMessages → **`handle.process`**。

### 第二次 chat.params（关键）

37. **`chat.params` 再次触发** — 同一 session、同一 provider/model，但是 **step 2**。
38. 插件若用 `sessionID` 存状态，这里会再跑一遍；若只改 `options.enable_thinking`，每轮都生效。

39. 模型这次多半返回 **纯 text**（或 reasoning + text），无 tool-call。
40. `finish-step` → finishReason 常为 **`stop`**。
41. **`process()` 返回 `"continue"`** 或触发 compact（若 overflow）。

---

## 第七幕：runLoop 退出

42. runLoop 扫描：assistant **finish** 且 **无 pending tool-call**，且 `lastUser.id < lastAssistant.id`。
43. **`break`** 跳出 while。
44. **`SessionRunState` onIdle** → status **idle** → **Bus** → 插件 **`event`** 收到 `session.idle`。
45. HTTP 响应 / TUI 展示完整 assistant 消息。

**全链路结束。**

---

## 分支对照（本故事未走的路径）

| 若发生… | runLoop 走… |
|---------|-------------|
| user 带 subtask part | `handleSubtask` → 可能 **另一 agent + 另一 model** → TaskTool |
| token 超限 | `compaction.create` → compaction LLM（可能再用专门 model） |
| 用户 slash command | `command()` → template → 回到 `prompt()` |
| Provider 报错 | Processor **retry**（SessionRetry）→ 仍失败则 `session.error` |
| 模型 hallucinate 工具名 | Invalid tool 兜底 |

---

## 与多模型的关系（本故事里的 3 次潜在 LLM）

| 次序 | 用途 | 模型来源 |
|------|------|----------|
| 1 | 标题生成 | `small_model` / title agent / getSmallModel |
| 2 | 主对话 step1（tool-call） | user.message.model |
| 3 | 主对话 step2（总结） | **同上**（跟 lastUser.model） |

主循环内 **同一 user turn 的多 step 共用 lastUser 上的 model**；换模型需要 **新 user 消息** 或 session/agent 切换。

---

## 读完后应能回答

- [ ] chat.params 在本故事里被调了几次？
- [ ] tool 结果如何回到第二轮 LLM？
- [ ] idle 事件在 runLoop 内还是外触发？
- [ ] 标题生成会不会影响主对话 model？

→ **Processor 细节：** [A2](./A2-processor-deep-dive.md)  
→ **多模型规则：** [19](../19-multi-model-and-provider-system.md)  
→ **选下一步路径：** [learning-paths](../learning-paths.md) · **OmO 插件层：** [OmO 17](../../oh-my-openagent/17-message-hook-journey.md)
