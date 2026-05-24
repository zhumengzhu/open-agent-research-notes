# oh-my-openagent 学习笔记

> **官方仓库：** [code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（upstream）  
> **npm（过渡期双发）：** [`oh-my-opencode`](https://www.npmjs.com/package/oh-my-opencode) · [`oh-my-openagent`](https://www.npmjs.com/package/oh-my-openagent)  
> **基准版本：** [`20d67be4`](https://github.com/code-yeongyu/oh-my-openagent/commit/20d67be496155473f49aef3207bfe9d3737cbfa8)（2026-05-23）  
> **依赖内核：** [anomalyco/opencode](https://github.com/anomalyco/opencode) · 内核笔记见 [../opencode/README.md](../opencode/README.md)

OmO 是运行在 OpenCode 上的多 agent 编排插件。本目录按概念编号组织：从插件协议与 `chat.params`，到 tool/agent/MCP 子系统，再到系统关系与架构评估。

![OpenCode 内核与 OmO 插件层边界](../../comparisons/assets/opencode-omo-boundary.svg)

<p align="center"><sub>内核拥有 loop 与 DB · 插件通过 <code>Plugin.trigger</code> 注入策略 · 详见 <a href="../../comparisons/opencode-vs-omo-boundary.md">边界对照</a> · 跟消息走 <a href="./17-message-hook-journey.md">17 Hook 链</a></sub></p>

---

## 从这里开始

| 目标 | 第一步 |
|------|--------|
| 跟一条消息看 OmO hook 链 | **[17 消息 Hook 链](./17-message-hook-journey.md)**（配 [边界对照](../../comparisons/opencode-vs-omo-boundary.md)） |
| 写第一个 OpenCode 插件 | [01 协议](./01-opencode-plugin-protocol.md) → [03 chat.params](./03-chat-params-mechanism.md) |
| 理解 OmO 初始化 | [02 入口与生命周期](./02-plugin-entry-and-lifecycle.md) |
| 查内部 hook 全表 | [07 hook 体系概览](./07-hook-system-overview.md) |

**建议前置：** 先读 OpenCode [18 总图](../opencode/18-main-chain-atlas.md) + [A1 内核叙事](../opencode/appendix/A1-full-trace-narrative.md)，再读 [17](./17-message-hook-journey.md)。

---

## 概念目录

| 编号 | 概念 | 核心问题 |
|------|------|----------|
| [00](./00-philosophy.md) | 项目理念 | 为什么 OmO 这么多「强制」机制？ |
| [01](./01-opencode-plugin-protocol.md) | **OpenCode 插件协议** | 插件文件要满足什么形状才能被加载？ |
| [02](./02-plugin-entry-and-lifecycle.md) | **入口与初始化** | 7 步 init 在干什么？ |
| [03](./03-chat-params-mechanism.md) | **`chat.params`** | LLM 前能改哪些参数？ |
| [04](./04-anthropic-effort-case-study.md) | **anthropic-effort 范本** | 按 provider/model 改参的最小完整例 |
| [05](./05-think-mode-mechanism.md) | **think-mode** | 关键词 → variant 的完整链路 |
| [06](./06-model-capabilities-and-compat.md) | 模型能力 + clamping | graceful degradation 怎么做？ |
| [07](./07-hook-system-overview.md) | 5 层 hook 映射 | 内部 hook 与 OpenCode 12 切面的关系 |
| [08](./08-tool-registry.md) | 工具注册与门控 | 20–39 个工具如何按配置启用 |
| [09](./09-agent-system.md) | Agent 系统 | 11 个 agent 与 dynamic prompt |
| [10](./10-config-and-plugin-handlers.md) | 配置与 6 阶段装配 | JSONC → runtime |
| [11](./11-feature-modules-map.md) | Feature 模块总览 | 20 个子系统边界 |
| [12](./12-team-mode-deep-dive.md) | Team Mode | 并行多 agent 运行时 |
| [13](./13-openclaw-and-external-loop.md) | OpenClaw | 外部系统双向回路 |
| [14](./14-mcp-and-cli-operability.md) | MCP + CLI | 三层 MCP 与 doctor/run |
| [15](./15-system-relationships-and-architecture-review.md) | 架构评估 | 优缺点与技术债 |
| [16](./16-opencode-learning-map.md) | OpenCode 学习地图 | 内核还需补什么 |
| **[17](./17-message-hook-journey.md)** | **一条消息的 Hook 链** | **叙事：OmO 在每一站做了什么** |
| [99](./99-glossary-and-future-plugins.md) | 术语与插件清单 | — |

---

## 学习路径

**第一波（核心 6 篇，约 4–6h）：**

```
00 → 01 → 02 → 03 → 04 → 05
```

**第二波（OmO 子系统）：**

| 目标 | 路径 |
|------|------|
| 跟消息走 hook | **17** → 07 → [边界](../../comparisons/opencode-vs-omo-boundary.md) |
| 加新 hook | 02 + 07 + 10 |
| 加新 tool | 08 + 10 + 11 |
| 加新 agent | 09 + 10 + 15 |
| 独立 OpenCode 插件 | 01 + 03 + 04 + 05 |
| 补内核 | 16 → [opencode 笔记](../opencode/README.md) |

---

## 核心验收（01–05 + 17）

- [ ] OpenCode 插件 default export 必须是什么形状？
- [ ] `chat.params` 与 `messages.transform` 的差别与触发频率？
- [ ] OmO 内部 hook 与 OpenCode 切面的两层关系？
- [ ] 一条普通消息经过哪些 OmO handler（按顺序）？
- [ ] `resolveCompatibleModelSettings` 解决什么问题？

---

## 关联仓库

| 仓库 | 说明 |
|------|------|
| [code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | **本笔记对应源码（官方 upstream）** |
| [anomalyco/opencode](https://github.com/anomalyco/opencode) | OpenCode 内核（OmO 的运行时） |
| [opencode-thinking-toggle](https://github.com/zhumengzhu/opencode-thinking-toggle) | 独立 `chat.params` 插件实践 |

**社区对照（OmO 叙事）：** [qqzhangyanhua oh-flow](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/oh-flow/index.md)（本文 17 为 pin SHA 版）
