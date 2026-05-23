# oh-my-openagent 学习笔记

> **源码：** [code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)  
> **基准版本：** [`20d67be4`](https://github.com/code-yeongyu/oh-my-openagent/commit/20d67be496155473f49aef3207bfe9d3737cbfa8)（2026-05-23，upstream）  
> **外链：** 正文举证链接均 pin 到该 commit（`blob/<sha>/...`），不用 `dev` 分支。

OmO 是运行在 OpenCode 上的多 agent 编排插件。本目录按概念编号组织：从插件协议与 `chat.params`，到 tool/agent/MCP 子系统，再到系统关系与架构评估。

---

## 概念目录

按"先吃懂插件机制，再扩展到 OmO 自己的抽象"顺序排列。**01–05 是核心 5 篇**，剩下的按需展开。

| 编号 | 概念 | 核心问题 | 状态 |
|------|------|----------|------|
| [00](./00-philosophy.md) | 项目理念 | 为什么 OmO 这么多"强制"机制？ | 待读 |
| [01](./01-opencode-plugin-protocol.md) | **OpenCode 插件协议本质** | 一个 `.js` 文件要长什么样才能被 OpenCode 当插件加载？ | 待读 |
| [02](./02-plugin-entry-and-lifecycle.md) | **OmO 插件入口与初始化** | OmO 是怎么挂到 OpenCode 上的？7 步初始化在干什么？ | 待读 |
| [03](./03-chat-params-mechanism.md) | **`chat.params` 钩子机制** | 调用 LLM 之前我能改哪些参数？怎么改？ | 待读 |
| [04](./04-anthropic-effort-case-study.md) | **范本精读：anthropic-effort hook** | "基于 provider/model 修改 LLM 入参"的最小完整例子怎么写？ | 待读 |
| [05](./05-think-mode-mechanism.md) | **关键词→变体切换：think-mode** | 用户输入关键词触发推理变体的完整链路是怎么实现的？ | 待读 |
| [06](./06-model-capabilities-and-compat.md) | 模型能力 + 兼容性 clamping | OmO 怎么做 graceful degradation？variant ladder 是什么？ | 待读 |
| [07](./07-hook-system-overview.md) | OmO 5 层 hook 与 12 切面的映射 | 内部 hook 为什么分 5 层？跟 OpenCode 的 12 切面是什么关系？ | 待读 |
| [08](./08-tool-registry.md) | 工具注册与门控 | 20–39 个工具是怎么按配置启用/禁用的？ | 待读 |
| [09](./09-agent-system.md) | Agent 系统 | 11 个 agent 是工厂模式还是？dynamic-agent-prompt-builder 在干什么？ | 待读 |
| [10](./10-config-and-plugin-handlers.md) | 配置系统与 6 阶段装配 | 用户配置如何变成最终 agents/tools/mcps/commands？ | 待读 |
| [11](./11-feature-modules-map.md) | Feature 模块总览 | 20 个子系统的边界、复杂度、依赖关系是什么？ | 待读 |
| [12](./12-team-mode-deep-dive.md) | Team Mode 深潜 | 并行多代理协作为何是独立运行时？ | 待读 |
| [13](./13-openclaw-and-external-loop.md) | OpenClaw 双向回路 | 外部系统通知与回注会话如何闭环？ | 待读 |
| [14](./14-mcp-and-cli-operability.md) | MCP + CLI 可运维面 | 三层 MCP 与 doctor/run 如何形成运维能力？ | 待读 |
| [15](./15-system-relationships-and-architecture-review.md) | 系统关系与架构评估 | 哪些设计好？哪些是技术债？ | 待读 |
| [16](./16-opencode-learning-map.md) | OpenCode 学习地图 | 为理解 coding agent 内核还需补什么？ | 待读 |
| [99](./99-glossary-and-future-plugins.md) | 术语词典 + 我未来想做的插件清单 | — | 待读 |

---

## 学习路径建议

**第一波（核心 5 篇，估 4–6h）：**

```
00 → 01 → 02 → 03 → 04 → 05
```

读完这 6 篇就能：
- 独立写一个 OpenCode 插件
- 看懂 OmO 任何一个 hook 的实现思路
- 立即上手 `opencode-thinking-toggle` 的开发

**第二波（补齐 OmO 必要子系统）：**

| 目标 | 读哪几篇 |
|----------|----------|
| 100%掌握 OmO 的必要子系统 | 06 → 10 → 11 → 12 → 13 → 14 → 15 |
| 给 OmO 加一个新 hook | 02 + 07 + 10 |
| 给 OmO 加一个新 tool | 08 + 10 + 11 |
| 给 OmO 加一个新 agent | 09 + 10 + 15 |
| 写完全独立的 OpenCode 插件 | 01 + 03 + 04 + 05（已足够） |
| 进入 OpenCode 底层学习 | 16（再去 `~/Github/opencode` 按图索骥） |

---

## 核心问题验收（学完 01–05 应该能回答）

- [ ] OpenCode 插件 default export 必须是什么形状？
- [ ] 12 个生命周期 hook 各自的触发时机和能改的东西？
- [ ] `chat.params` 和 `experimental.chat.messages.transform` 的差别？
- [ ] OmO 为什么要把内部 hook 分 5 层？和 OpenCode 的 12 切面是什么关系？
- [ ] 想做"基于 provider/model 修改 LLM 调用参数"的事，应该挂哪个 hook？
- [ ] OmO 的 `resolveCompatibleModelSettings` 解决了什么问题？
- [ ] DeepSeek 关 thinking 应该改 `output` 的哪个字段？

---

## 关联仓库

| 仓库 | 说明 |
|------|------|
| [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | 本笔记对应的源码 |
| [opencode-thinking-toggle](https://github.com/zhumengzhu/opencode-thinking-toggle) | 基于 `chat.params` 的独立 OpenCode 插件实践 |
