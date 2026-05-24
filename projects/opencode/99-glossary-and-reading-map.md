# 99 · 术语表与阅读地图

> **用途：** 速查术语、按概念跳转章节、选学习路径。  
> **路径详情：** [learning-paths.md](./learning-paths.md) · **社区对照：** [external-resources.md](./external-resources.md)

---

## 1. 核心概念快速定位

想找某个主题时，直接跳转（借鉴 [qqzhangyanhua reading-map](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/reading-map.md) 的表格思路）：

| 核心概念 | 主章节 | 补充 | 一句话 |
|----------|--------|------|--------|
| **Agent Loop（runLoop）** | [09](./09-session-prompt-runloop.md) · [18](./18-main-chain-atlas.md) | [A1](./appendix/A1-full-trace-narrative.md) | `SessionPrompt.run` 的 while 循环 |
| **SessionProcessor / stream** | [A2](./appendix/A2-processor-deep-dive.md) | [10](./10-llm-stream-and-provider.md) | 单轮 LLM 流解析状态机 |
| **MessageV2 / Part** | [08](./08-session-message-and-storage.md) | [A1 第二幕](./appendix/A1-full-trace-narrative.md) | 消息 + 多 Part 持久化 |
| **Tool / Function Calling** | [11](./11-tool-registry-and-execution.md) | [flow/tool](./flow/README.md) | Registry 合并 → before/execute/after |
| **Permission** | [07](./07-agent-and-permission.md) | [flow/permission](./flow/README.md) | ruleset 求值 → ask/allow/deny |
| **Plugin / Hook** | [05](./05-plugin-protocol-and-loader.md) · [06](./06-hook-system-reference.md) | [16](./16-plugin-author-guide.md) | `Plugin.trigger` 同步串行 |
| **chat.params** | [10](./10-llm-stream-and-provider.md) | [19](./19-multi-model-and-provider-system.md) | 每 LLM step 前改请求参数 |
| **多模型 / Provider** | [19](./19-multi-model-and-provider-system.md) | [10](./10-llm-stream-and-provider.md) | user/agent/config/default 选模 |
| **Compaction / 上下文** | [12](./12-compaction-and-context-management.md) | [08](./08-session-message-and-storage.md) | token 压力 → 摘要压缩 |
| **Task / subagent** | [11](./11-tool-registry-and-execution.md) | [19 §subtask](./19-multi-model-and-provider-system.md) | 子会话 + 独立 model 可选 |
| **Instance / Bootstrap** | [02](./02-effect-instance-and-bootstrap.md) | [03](./03-cli-server-and-entry.md) | 按 directory 隔离 runtime |
| **Bus / SSE 事件** | [13](./13-bus-and-session-events.md) | [flow/state_sync](./flow/README.md) | Session 状态推送到客户端 |
| **MCP / LSP / Skill** | [14](./14-mcp-lsp-skill-and-command.md) | ZeroZ internals | 工具与指令扩展 |
| **OmO 插件层（可选）** | [OmO 17](../oh-my-openagent/17-message-hook-journey.md) · [边界对照](../../comparisons/opencode-vs-omo-boundary.md) | [OmO 07](../oh-my-openagent/07-hook-system-overview.md) · [oh-flow](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/oh-flow/index.md) | 内核之上的 hook 策略 |

---

## 2. 核心术语

| 术语 | 含义 | 详见 |
|------|------|------|
| **Instance** | 单个 project directory 的 runtime | [02](./02-effect-instance-and-bootstrap.md) |
| **SessionRunState** | 同 session 单 runLoop 并发控制 | [09](./09-session-prompt-runloop.md) |
| **SessionPrompt.run** | runLoop 本体 | [09](./09-session-prompt-runloop.md) |
| **SessionProcessor** | 单轮 LLM 流状态机 | [A2](./appendix/A2-processor-deep-dive.md) |
| **MessageV2 / Part** | 消息 + 多 Part 模型 | [08](./08-session-message-and-storage.md) |
| **Provider / Model / Variant** | 厂商 / 模型 / 档位 | [19](./19-multi-model-and-provider-system.md) |
| **small_model** | 标题等轻量 LLM | [19 §5](./19-multi-model-and-provider-system.md) |
| **ToolRegistry** | 工具合并与过滤 | [11](./11-tool-registry-and-execution.md) |
| **Plugin.trigger** | 同步串行 hook | [05](./05-plugin-protocol-and-loader.md) |
| **chat.params** | LLM 请求前改参 | [10](./10-llm-stream-and-provider.md) |

---

## 3. 文档分层（人类怎么读）

| 类型 | 文档 | 用途 |
|------|------|------|
| **地图** | [18](./18-main-chain-atlas.md) | 5 分钟建立全局 |
| **叙事** | [A1](./appendix/A1-full-trace-narrative.md) | 30 分钟跟故事 |
| **专题** | [19](./19-multi-model-and-provider-system.md) | 多 Provider 必读 |
| **深读** | [A2](./appendix/A2-processor-deep-dive.md) | stream 事件表 |
| **流程** | [flow/](./flow/README.md) | 权限、插件、工具等流程导航 |
| **路径** | [learning-paths](./learning-paths.md) | 带 Checklist 的自学路线 |
| **概念** | 00–17 | 按模块查阅 |
| **验收** | [17 §4](./17-architecture-review-and-mastery.md) | 自测 + 简答 |

---

## 4. `packages/opencode/src` 模块索引

| 目录 | 一句话 | 文档 |
|------|--------|------|
| session/ | 心脏 | 08–12, A1, A2 |
| provider/ | 多模型注册 | **19**, 10 |
| plugin/ | 插件引擎 | 05–06 |
| tool/ | 内置工具 | 11 |
| agent/ | Agent + model 绑定 | 07, **19** |
| config/ | 配置 | 04, **19** |
| project/ | 实例 | 02–03 |
| server/ | HTTP | 03 |
| bus/ | 事件 | 13 |
| storage/ | SQLite | 08 |

---

## 5. 按目标选读

| 目标 | 路径 |
|------|------|
| 第一次学 OpenCode | [路径 A](./learning-paths.md#路径-a第一次学-opencode--约-35-小时)：`18 → A1 → 09 → A2 → 19` |
| 读懂内核 | [路径 B](./learning-paths.md#路径-b读懂内核深读--约-12-天) |
| 写 chat.params 插件 | [路径 C](./learning-paths.md#路径-c写-opencode-插件--约-48-小时) |
| 跟 prompt.ts | 09 + A1 + A2 |
| 配多 Provider | **19** + 04 |
| 查某个流程 | [flow/](./flow/README.md) |
| 跨项目边界（OmO） | [comparisons/boundary](../../comparisons/opencode-vs-omo-boundary.md) |
| 社区资料该读哪 | [external-resources](./external-resources.md) |

---

## 6. 阅读边界与证据链接

**本笔记是什么：** 基于 pin commit 的 OpenCode **内核带读**，不是脱离源码的 Agent 通识教材。

**基线 commit：**

```
https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/<path>#L<n>
```

**跟码：** `git checkout 7fe7b9f258e36ad9f9acded20c5a9df201da19d5` 或搜 `Effect.fn` 名。

**upstream 已变时：** 先搜函数名定位，再对照本笔记结论是否仍成立；大版本对齐时更新 README 中的 baseline。

---

## 7. 与 oh-my-openagent 笔记（可选）

| OpenCode | 插件层参考 |
|----------|------------|
| [19 多模型](./19-multi-model-and-provider-system.md) | [06 model-capabilities](../oh-my-openagent/06-model-capabilities-and-compat.md) |
| [10 LLM](./10-llm-stream-and-provider.md) | [03 chat.params](../oh-my-openagent/03-chat-params-mechanism.md) |
| [A1 Trace](./appendix/A1-full-trace-narrative.md) | [02 生命周期](../oh-my-openagent/02-plugin-entry-and-lifecycle.md) |
| [06 Hook 参考](./06-hook-system-reference.md) | [07 hook 概览](../oh-my-openagent/07-hook-system-overview.md) |

→ **回到入口：** [README](./README.md)
