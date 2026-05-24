# OpenCode 内核学习笔记

> **源码：** [anomalyco/opencode](https://github.com/anomalyco/opencode)  
> **基准版本：** [`7fe7b9f2`](https://github.com/anomalyco/opencode/commit/7fe7b9f258e36ad9f9acded20c5a9df201da19d5)（upstream，2026-05-24）  
> **个人 fork：** [zhumengzhu/opencode](https://github.com/zhumengzhu/opencode)

**核心目的：** 帮助读者**真正读懂** OpenCode runtime——从一条用户消息出发，跟到 `prompt.ts`、`processor.ts`、`llm/request.ts`，而不是只记住名词。

OpenCode 是 coding agent 的 **runtime 内核**（Session DB、runLoop、Permission、Plugin.trigger）。本目录围绕这条主链路组织；Monorepo 其它包（app/desktop）仅概要，深度可对照社区资料。

![OpenCode 内核主链路](./assets/main-chain-atlas.svg)

<p align="center"><sub>主链路与 Plugin 注入点（图内英文标签，避免 GitHub SVG 编码问题）· 深读见 <a href="./18-main-chain-atlas.md">18 总图</a></sub></p>

---

## 从这里开始（先选路径，再读文档）

**不要**从 00 线性读到 17——信息量大，容易失焦。

| 你是谁 | 第一步 | 完整路径 |
|--------|--------|----------|
| 第一次学 OpenCode | [18 总图](./18-main-chain-atlas.md) | [路径 A · 约 3–5h](./learning-paths.md#路径-a第一次学-opencode--约-35-小时) |
| 要改内核 / debug runLoop | [A1 叙事](./appendix/A1-full-trace-narrative.md) | [路径 B · 约 1–2 天](./learning-paths.md#路径-b读懂内核深读--约-12-天) |
| 要写插件 | [19 多模型](./19-multi-model-and-provider-system.md) | [路径 C · 约 4–8h](./learning-paths.md#路径-c写-opencode-插件--约-48-小时) |
| 评估架构 | [00 定位](./00-philosophy-and-positioning.md) | [路径 D · 约 1–2h](./learning-paths.md#路径-d架构扫读--约-12-小时) |
| 改 Web/Desktop 客户端 | [01 monorepo](./01-monorepo-and-packages.md) | [路径 E · 按需](./learning-paths.md#路径-e客户端与全栈按需--约-半天) |

**推荐起手（多数人）：** `18 → A1 → 09 → A2 → 19`

**跟源码：** 文档基于 baseline `7fe7b9f2`；本地 HEAD 可能不同 → **优先搜 `Effect.fn("…")` 名**，或 `git checkout 7fe7b9f2`。

---

## 社区资料（取长补短）

下列仓库与本笔记互补，建议对照阅读，详见 [external-resources.md](./external-resources.md)。

| 仓库 | 链接 | 补什么 |
|------|------|--------|
| ZeroZ-lab/learn-opencode | [GitHub](https://github.com/ZeroZ-lab/learn-opencode) | Monorepo 全包、flow 流程、Cookbook、FAQ |
| qqzhangyanhua/learn-opencode-agent | [GitHub](https://github.com/qqzhangyanhua/learn-opencode-agent) | 阅读地图、实践项目、**oh-flow**（OmO 插件叙事） |

**本笔记相对社区的差异化：** pin SHA 证据链 · A2 Processor 深读 · 19 多模型专题 · [OpenCode↔OmO 边界](../../comparisons/opencode-vs-omo-boundary.md)

---

## 文档分层（怎么读最高效）

| 类型 | 文档 | 读者价值 |
|------|------|----------|
| **地图** | [18 总图](./18-main-chain-atlas.md) | 5 分钟建立全局，读源码时的锚点 |
| **叙事** | [A1 Trace](./appendix/A1-full-trace-narrative.md) | 30 分钟「跟着故事走」 |
| **深读** | [A2 Processor](./appendix/A2-processor-deep-dive.md) | stream 事件状态机 |
| **专题** | [19 多模型](./19-multi-model-and-provider-system.md) | Provider / variant / small_model / subtask |
| **流程索引** | [flow/](./flow/README.md) | 权限、插件加载、工具执行等流程导航 |
| **概念** | 00–17 | 按模块查阅，不必线性 |
| **路径** | [learning-paths.md](./learning-paths.md) | 带 Checklist 的自学路线 |
| **速查** | [99 索引](./99-glossary-and-reading-map.md) | 术语 + 概念 → 章节 |

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
| **专题** | [19](./19-multi-model-and-provider-system.md) | **多模型与 Provider** |
| **附录** | [A1](./appendix/A1-full-trace-narrative.md) · [A2](./appendix/A2-processor-deep-dive.md) | 叙事 Trace · Processor 深读 |
| **导航** | [flow/](./flow/README.md) · [learning-paths](./learning-paths.md) · [99](./99-glossary-and-reading-map.md) · [external-resources](./external-resources.md) | 流程 · 路径 · 速查 · 社区对照 |

---

## Monorepo 速查

| 包 | 职责 |
|----|------|
| [`packages/opencode/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode) | CLI + HTTP + Session（**主读**） |
| [`packages/plugin/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/plugin) | 插件 SDK |
| [`packages/core/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/core) | models.dev、schema |
| [`packages/llm/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/llm) | LLM 路由 |
| [`packages/sdk/js/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/sdk/js) | HTTP Client |

app / desktop / console 等客户端包 → 见 [ZeroZ-lab packages](https://github.com/ZeroZ-lab/learn-opencode/tree/main/docs/packages) 或 [路径 E](./learning-paths.md#路径-e客户端与全栈按需--约-半天)。

模块全表见 [99](./99-glossary-and-reading-map.md)。

---

## 跨项目（可选）

使用 **oh-my-openagent** 时，先读内核再读边界：

1. 本目录路径 A 或 B  
2. [OpenCode ↔ OmO 边界](../../comparisons/opencode-vs-omo-boundary.md)  
3. **[OmO 17 · 消息 Hook 链](../oh-my-openagent/17-message-hook-journey.md)**（pin SHA + SVG）  
4. [oh-my-openagent 笔记](../oh-my-openagent/README.md)  
5. 社区对照：[qqzhangyanhua oh-flow](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/oh-flow/index.md)
