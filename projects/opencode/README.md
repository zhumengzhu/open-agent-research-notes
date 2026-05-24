# opencode 学习笔记

> **源码：** [anomalyco/opencode](https://github.com/anomalyco/opencode)  
> **基准版本：** [`7fe7b9f2`](https://github.com/anomalyco/opencode/commit/7fe7b9f258e36ad9f9acded20c5a9df201da19d5)（upstream，2026-05-24）  
> **个人 fork：** [zhumengzhu/opencode](https://github.com/zhumengzhu/opencode)

OpenCode 是 coding agent 的 **runtime 内核**。本目录按 **理念 → 架构 → 核心模块 → 扩展 → 验收** 组织。

---

## 人类阅读建议（重要）

1. **不要** 从 00 线性读到 17 —— 容易失焦。  
2. **推荐起手：** [18 总图](./18-main-chain-atlas.md) → [附录 A1 叙事](./appendix/A1-full-trace-narrative.md) → 按需跳读。  
3. **跟源码：** 文档行号基于 baseline `7fe7b9f2`；本地 HEAD 可能不同 → **优先搜 `Effect.fn("…")` 名**，或 `git checkout 7fe7b9f2`。  
4. **写插件：** 18 → [19 多模型](./19-multi-model-and-provider-system.md) → 10 → 16。  
5. **跨项目（可选）：** [增强插件边界](../../comparisons/opencode-vs-omo-boundary.md)

---

## 概念目录

| 层级 | 编号 | 主题 |
|------|------|------|
| **A 定位** | [00](./00-philosophy-and-positioning.md) | 产品理念与定位 |
| **B 架构** | [01](./01-monorepo-and-packages.md) | Monorepo 与包分层 |
| | [02](./02-effect-instance-and-bootstrap.md) | Effect、实例、bootstrap |
| | [03](./03-cli-server-and-entry.md) | CLI / Server / 请求入口 |
| | [04](./04-config-system.md) | 配置系统 |
| **C 扩展面** | [05](./05-plugin-protocol-and-loader.md) | 插件协议与加载 |
| | [06](./06-hook-system-reference.md) | Hook 完整参考 |
| **D 核心运行时** | [07](./07-agent-and-permission.md) | Agent 与 Permission |
| | [08](./08-session-message-and-storage.md) | Session / Message / 存储 |
| | [09](./09-session-prompt-runloop.md) | SessionPrompt 主循环 |
| | [10](./10-llm-stream-and-provider.md) | LLM 流与 chat.params |
| | [11](./11-tool-registry-and-execution.md) | ToolRegistry、Task/subagent |
| | [12](./12-compaction-and-context-management.md) | Compaction 与上下文 |
| **E 横切** | [13](./13-bus-and-session-events.md) | Bus 与 Session 事件 |
| | [14](./14-mcp-lsp-skill-and-command.md) | MCP / LSP / Skill / Command |
| | [15](./15-background-pty-and-worktree.md) | 后台 / PTY / 快照 / worktree |
| **F 实践** | [16](./16-plugin-author-guide.md) | 插件作者指南 |
| | [17](./17-architecture-review-and-mastery.md) | 架构评价与验收 |
| **Atlas** | [18](./18-main-chain-atlas.md) | **主链路总图** |
| **专题** | [19](./19-multi-model-and-provider-system.md) | **多模型与 Provider（必读）** |
| **附录** | [A1 叙事 Trace](./appendix/A1-full-trace-narrative.md) | 一条用户消息的故事 |
| | [A2 Processor 深读](./appendix/A2-processor-deep-dive.md) | LLM 流状态机 |
| **索引** | [99](./99-glossary-and-reading-map.md) | 术语表与阅读地图 |

---

## 推荐阅读顺序

### 路径 1：写插件（含多模型）

```
18 → A1（浏览）→ 19 → 10 → 06 → 16
```

### 路径 2：读懂内核（叙事 + 深度）

```
18 → A1 → 08 → 09 → A2 → 10 → 19 → 11 → 12 → 13 → 17
```

### 路径 3：架构扫读

```
00 → 01 → 02 → 04 → 18 → 99
```

---

## Monorepo 速查

| 包 | 职责 |
|----|------|
| [`packages/opencode/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode) | CLI + HTTP + Session（**主读**） |
| [`packages/plugin/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/plugin) | 插件 SDK |
| [`packages/core/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/core) | models.dev、schema |
| [`packages/llm/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/llm) | LLM 路由 |
| [`packages/sdk/js/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/sdk/js) | HTTP Client |

模块全表见 [99](./99-glossary-and-reading-map.md)。
