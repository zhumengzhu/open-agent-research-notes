# A1 · 一条 User Turn 的完整旅程（叙事）

> **用途：** 30–45 分钟跟着故事走一遍主路径。  
> **地图：** [26 总图](./26-main-chain-atlas.md) · **细节：** [06](./06-query-agent-loop.md) · [10](./10-compaction-and-context.md)

---

## 场景

用户在项目目录启动 `claude`，REPL 已跑完 `setup()`。输入：

```text
帮我在 src/foo.ts 里加一个导出函数并跑测试
```

---

## 第一幕：输入进入产品壳

1. **REPL** `PromptInput` 提交 → `handlePromptSubmit`（`utils/handlePromptSubmit.ts`）
2. 非 slash → `processUserInput`：
   - 解析 @ 引用、粘贴
   - UserPromptSubmit hooks（若有）
   - 生成 `UserMessage` + 附件
   - **`shouldQuery: true`**
3. `onQuery` 调用 **`QueryEngine.submitMessage`**

*若用户输入 `/doctor`，则 `shouldQuery: false`，本地命令输出后直接 return，**以下不发生**。*

---

## 第二幕：QueryEngine 组装 turn

4. `fetchSystemPromptParts` — system / userContext / systemContext（[13](./13-system-prompt-and-context.md)）
5. 合并 agent prompt、append、coordinator context
6. 构建 **ToolUseContext**：`assembleToolPool` 的 tools、mainLoopModel、thinking、abortController
7. 调用 **`query({ messages, systemPrompt, toolUseContext, canUseTool, querySource: 'repl_main_thread', ... })`**

---

## 第三幕：queryLoop 第一轮 iteration

8. **`stream_request_start`** yield 给 UI（Spinner）
9. **压缩管道**（[10](./10-compaction-and-context.md)）：
   - tool result budget
   - snip（若开）
   - microcompact（time-based 或 cached MC）
   - context collapse（若开）
   - autocompact（若 token 超阈值）
10. 假设未 compact：`messagesForQuery` = 当前历史 + 新 user message
11. **`deps.callModel`** 流式请求（[07](./07-api-and-model-stream.md)）

---

## 第四幕：模型流式回复 + 工具

12. UI 收到 text delta、`assistant`  partial
13. 模型发出 **tool_use**：`Read` path=`src/foo.ts`
14. **StreamingToolExecutor** 立即 `addTool` → `runToolUse`：
    - `canUseTool` → allow
    - PreToolUse hooks
    - FileRead 执行 → tool_result
15. yield progress / tool_result message
16. 流结束，`needsFollowUp = true`

---

## 第五幕：queryLoop 第二轮 iteration

17. **再次**跑压缩管道（通常轻量；cached MC 可能删旧 Read result 的 cache）
18. `messagesForQuery` 含 assistant + tool_result
19. 第二次 **callModel**
20. 模型 tool_use：`Edit` 或 `FileEdit` 修改文件 → 同样执行链
21. 再 follow-up：tool_use `Bash` `npm test` …

---

## 第六幕：结束 turn

22. 某轮流结束 **无 tool_use** → `needsFollowUp = false`
23. **handleStopHooks**：
    - `saveCacheSafeParams`（供下轮 side query / fork）
    - prompt suggestion、extract memories（feature，async）
24. 若无 blocking stop hook → **`return { reason: 'completed' }`**
25. QueryEngine 更新 `mutableMessages`，**`recordTranscript`** 增量写 JSONL（[08](./08-message-and-session-persistence.md)）
26. yield SDK/内部 message；REPL 渲染最终 assistant 文本

---

## 支线（本场景未触发）

| 事件 | 路径 |
|------|------|
| Permission deny | synthetic tool_result → 模型可见拒绝 |
| Token 满 | autocompact → 摘要 → 同 turn 继续 API |
| API 413 | reactive compact → continue |
| User Ctrl+C | AbortSignal → `aborted_streaming` |
| maxTurns | `return max_turns` |

---

## 读完后应能指出的文件

`handlePromptSubmit` → `QueryEngine.submitMessage` → `query` → `queryLoop` → `microcompact`/`autocompact` → `queryModelWithStreaming` → `StreamingToolExecutor` → `runToolUse` → `handleStopHooks` → `recordTranscript`

**下一步：** [A2 transition 表](./A2-query-loop-transitions.md) · [25 自测](./25-architecture-review-and-mastery.md)
