# 17 · 架构评价与掌握验收

> **核心问题：** 内核设计权衡是什么？如何自测已掌握构造？

---

## 1. 设计优点

| 点 | 说明 |
|----|------|
| 清晰插件边界 | 公开 SDK + `Plugin.trigger` |
| Instance 隔离 | 多 directory 不串 session |
| MessageV2 + Part | reasoning / tool 分 part |
| Config 多级 merge | 用户 / 项目 / 插件分层 |
| 扩展线多元 | Plugin / MCP / Skill / Command |
| Effect 模块化 | InstanceState 生命周期明确 |

---

## 2. 设计代价

| 点 | 说明 |
|----|------|
| `prompt.ts` 过大 | 需按 [09 跳读地图](./09-session-prompt-runloop.md) 切面阅读 |
| experimental hook | 签名可能变；插件应 pin OpenCode 版本 |
| trigger 串行 | 多插件同 hook 可能互相覆盖 output |
| `permission.ask` 未接线 | SDK 与运行时有 gap（见 [06](./06-hook-system-reference.md)） |

---

## 3. 内核栈（复习）

```
Client → HTTP → Instance(bootstrap) → SessionPrompt.runLoop
  → Processor → llm/request(chat.params) → Provider
  → Tool execute → DB
Bus → event hook（idle / error）
```

详图：[18](./18-main-chain-atlas.md)

---

## 4. 掌握验收

### 4.1 画图（闭卷）

画时序图：用户输入 → runLoop → LLM → tool → idle，标 **≥8 个** hook 或 Bus 事件。

### 4.2 指文件（闭卷）

| 问题 | 路径 |
|------|------|
| 插件加载 | `plugin/loader.ts`, `plugin/index.ts` |
| config merge | `config/config.ts` |
| 主循环 | `session/prompt.ts`（`SessionPrompt.run`） |
| chat.params | `session/llm/request.ts` |
| 并发 | `session/run-state.ts` |
| 工具 | `tool/registry.ts`, `session/tools.ts` |
| 压缩 | `session/compaction.ts` |

### 4.3 概念

- [ ] `Plugin.trigger` 与 `event` 区别？
- [ ] runLoop 在什么条件下 break？
- [ ] `subtask` part 与 Task tool 关系？
- [ ] chat.params 每 turn 是否触发？

<details>
<summary>简答要点（先自测再展开）</summary>

- **trigger vs event：** trigger 同步、mutate output、在请求/tool 路径上；event 异步广播、只读，idle/error 等在 runLoop 外。
- **runLoop break：** assistant finish 且非 tool-calls finish、无未完成的非 providerExecuted tool、user 已被回复（`lastUser.id < lastAssistant.id`）。
- **subtask vs task tool：** subtask 是 user part，runLoop 优先 `handleSubtask`；LLM 发起的 task 走同一 TaskTool.execute，结果回填 history。
- **chat.params：** 每个 LLM step 都触发（含 tool 后续聊）。

</details>

### 4.4 多模型

- [ ] 主 model、small_model、variant 各是什么？
- [ ] subagent 如何换 model？

<details>
<summary>简答要点</summary>

见 [19](./19-multi-model-and-provider-system.md)：user/agent/config/default 决定主 model；small 用于标题等；variant 是 options 档位；subtask.part.model 或 agent.model 可独立。

</details>

### 4.5 实践（选做）

- [ ] 实现仅含 `chat.params` 的最小插件（见 [16](./16-plugin-author-guide.md)）
- [ ] 用 `opencode run` 对比 hook 开/关时的 `options` diff

---

## 5. 与增强插件的边界（可选阅读）

若同时使用社区增强插件（如 oh-my-openagent），职责划分见 [边界对照](../../comparisons/opencode-vs-omo-boundary.md)。**学 OpenCode 内核本身不依赖该文档。**

---

## 读完后应能回答

- [ ] 说出 3 个内核 invariant 与 3 个 extension surface
- [ ] 能否在 5 分钟内从 18 总图定位到 chat.params 源码行？

→ **索引：** [99 · 术语表](./99-glossary-and-reading-map.md)
