# Pi Coding Agent 学习笔记

> **源码：** [earendil-works/pi](https://github.com/earendil-works/pi)
> **基准版本：** [`fc8a1559`](https://github.com/earendil-works/pi/commit/fc8a1559017f1e581cfa971aa3cef11a507a4975)（upstream，2026-05）
> **npm：** `@mariozechner/pi-coding-agent` / `@mariozechner/pi-agent-core` / `@mariozechner/pi-ai` / `@mariozechner/pi-tui`
> **作者：** Mario Zechner（[libGDX](https://github.com/libgdx/libgdx) 创始人）

**核心目的：** 理解 Pi 的「激进极简主义」设计哲学——只用 4 个工具（read/write/edit/bash）、~150 词系统提示词，如何构建一个 ~45K stars 的 coding agent。

Pi 是 Mario Zechner 对 Claude Code 日益复杂化的反抗。他用游戏引擎的架构思维（渲染器抽象 → 游戏循环 → 场景图 → 内容），构建了一个**核心极简、扩展激进**的 agent 框架。

---

## 从这里开始

| 你是谁 | 第一步 | 完整路径 |
|--------|--------|----------|
| 想快速了解 Pi 的设计哲学 | [01 分包架构](./01-package-architecture.md) | `01 → 03 → 08` |
| 想理解 agent 循环怎么工作 | [03 Agent 循环](./03-agent-loop.md) | `03 → 05 → 07` |
| 想给 Pi 写扩展 | [06 扩展系统](./06-extension-system.md) | `06 → 02（provider 部分）` |
| 想对比 Pi vs 其他 coding agent | [09 生态与对比](./09-ecosystem-comparison.md) | `09 → 01 → 08` |
| 想深入 LLM provider 抽象 | [02 LLM 抽象层](./02-llm-abstraction-layer.md) | `02 → 08` |

**推荐起手（多数人）：** `01 → 03 → 05 → 06`

---

## 文档索引

| 编号 | 主题 | 层级 | 核心问题 |
|------|------|------|----------|
| [01](./01-package-architecture.md) | 分包架构与依赖图 | 架构 | 4 个包如何分层？tui 为什么零依赖 LLM？ |
| [02](./02-llm-abstraction-layer.md) | LLM 抽象层（pi-ai） | 基础 | 如何用 4 种协议统一 10+ provider？ |
| [03](./03-agent-loop.md) | Agent 循环（pi-agent-core） | 核心 | LLM → tool → result → LLM 循环怎么转？ |
| [04](./04-terminal-ui.md) | TUI 框架（pi-tui） | 基础 | 微分渲染、保留模式是怎么做的？ |
| [05](./05-agent-session.md) | AgentSession 运行时 | 核心 | 怎么把 agent 循环变成可用的 CLI？ |
| [06](./06-extension-system.md) | 扩展系统 | 核心 | 26+ 事件钩子怎么组织？扩展怎么写？ |
| [07](./07-session-compaction.md) | 会话持久化与压缩 | 横切 | JSONL 树形会话 + 自动压缩怎么工作？ |
| [08](./08-stealth-mode.md) | Stealth Mode | 专题 | 为什么伪装成 Claude Code？怎么实现的？ |
| [09](./09-ecosystem-comparison.md) | 生态与对比分析 | 广度 | Pi 的扩展生态、SDK 用法、vs 竞品 |

---

## 项目特征速查

| 维度 | 特征 |
|------|------|
| 语言 | TypeScript（ESM，strict mode），Node.js ≥ 22.19 + Bun |
| 代码规模 | ~147K LOC，583 个 .ts 文件 |
| 包 | 4 个（ai / agent / coding-agent / tui） |
| 默认工具 | read、write、edit、bash（仅 4 个） |
| 系统提示词 | ~150 词，< 1000 token |
| Provider | 10+（Anthropic、OpenAI、Google、Bedrock、Mistral、xAI 等） |
| 扩展模型 | TypeScript 扩展，26+ 生命周期钩子 |
| 会话格式 | JSONL 树形（id/parentId），支持分支 |
| 许可协议 | MIT |

---

## 外部资料（取长补短）

| 来源 | 链接 | 补什么 |
|------|------|--------|
| awesome-ai-anatomy | [NeuZhou/awesome-ai-anatomy/pi-mono](https://github.com/NeuZhou/awesome-ai-anatomy/tree/main/pi-mono) | 深度源码拆解（147K LOC 逐行分析） |
| 作者博客 | [mariozechner.at](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) | 第一手设计动机与权衡 |
| Pragmatic Engineer | [采访 Mario](https://newsletter.pragmaticengineer.com/p/building-pi-and-what-makes-self-modifying) | 哲学讨论 |
| Agent Architectures | [agentarchitectures.com/framework/pi-dev](https://agentarchitectures.com/framework/pi-dev) | 架构分析 |
| Pi 文档 | [pi.dev/docs](https://pi.dev/docs/latest) | 官方文档 |

---

## Pi 的独特设计决策

Pi 之所以值得研究，不在于它做了什么，而在于它**刻意不做什么**：

| Pi 不做 | 为什么 | 替代方案 |
|---------|--------|---------|
| 无 Sub-agent（内置） | 不可观察，调试困难 | 通过 bash 调用自己，或装扩展 |
| 无 Plan Mode | 写计划到文件更可见 | 写 `PLAN.md`，agent 读写 |
| 无 MCP 支持 | 上下文开销大（18K token） | CLI 工具 + README（渐进式披露） |
| 无 Background Bash | 复杂度高，不可观察 | 用 tmux |
| 无内置 Todo | 模型容易迷糊 | 写 `TODO.md`（文件即状态） |
| 无权限弹窗 | 安全剧场（prompt injection 无法根本防御） | 在容器里跑 |

**统一哲学：文件即状态、可见即真理、simple > clever。**
