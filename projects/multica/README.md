# Multica 项目研究笔记

> **源码：** [multica-ai/multica](https://github.com/multica-ai/multica)
> **官网：** [multica.ai](https://multica.ai)
> **分析基准：** v0.3.20（2026-06-11），36,272 Stars

**核心目的：** 理解 Multica 如何将「AI agent 作为一等公民」的理念落地为一个完整的 Managed Agents 平台——从多态行动者模型、本地 daemon 执行架构、到 Squads/Skills/Autopilots 等差异化功能。

Multica 不是另一个 coding agent，也不是另一个任务管理工具。它是**两者之间的缺失层**——让人类和 AI agent 在同一套任务管理流程中对等协作。名字致敬 1960 年代的 Multics 操作系统（首创分时复用），隐喻「人类与 AI agent 共享同一套任务系统」。

---

## 从这里开始

| 你是谁 | 第一步 | 完整路径 |
|--------|--------|----------|
| 想快速了解 Multica 是什么、做什么 | [01 深度分析](./01-deep-analysis.md) 的「项目概述」和「核心功能」章节 | `01 全文` |
| 想理解 agent 如何作为一等公民运作 | [01 深度分析](./01-deep-analysis.md) 的「核心理念」和「项目架构」章节 | `01 → 竞品分析` |
| 想对比 Multica vs 其他 agent 管理平台 | [01 深度分析](./01-deep-analysis.md) 的「竞品分析」章节 | `01 → 差异化价值` |
| 想了解团队、融资、社区 | [01 深度分析](./01-deep-analysis.md) 的「创始团队」和「社区情况」章节 | `01 全文` |

**推荐起手（多数人）：** 通读 `01-deep-analysis.md` 全文，重点看「核心理念」→「项目架构」→「竞品分析」。

---

## 文档索引

| 编号 | 主题 | 层级 | 核心问题 |
|------|------|------|----------|
| [01](./01-deep-analysis.md) | 深度全景分析 | 全面 | Multica 是什么、谁做的、怎么做、为什么能赢、对手是谁？ |
| [02](./02-squad-mode-analysis.md) | 小队模式深度分析 | 深度 | Squad 如何实现多 agent 协作？上下文如何传输？任务如何编排？ |

---

## 项目特征速查

| 维度 | 特征 |
|------|------|
| 定位 | AI Managed Agents 平台（agent 是一等公民的任务管理工具） |
| 技术栈 | Go 后端 + Next.js 16 前端 + Electron 桌面端 + Expo iOS |
| 数据库 | PostgreSQL 17 + pgvector（65 个表，含历史迁移） |
| 状态管理 | TanStack Query 5（服务器状态）+ Zustand 5（客户端状态） |
| 包管理 | pnpm workspaces + Turborepo monorepo |
| Agent 支持 | 12 种 CLI（Claude Code、Codex、Copilot、OpenClaw、OpenCode、Hermes、Gemini、Pi、Cursor、Kimi、Kiro CLI、Antigravity） |
| 差异化功能 | Squads（agent 团队路由）、Skills（可复用知识包）、Autopilots（定时自动化）、运行时自动检测 |
| 团队 | 4 人工程师 + 十几名 AI agent |
| 融资 | 两轮早期融资（"顶级美元基金"，非 YC） |
| 许可证 | Modified Apache 2.0（内部使用免费，SaaS 需商业许可） |
| GitHub Stars | 36,272（2026-06-11） |
| 发布节奏 | 几乎每日发布（v0.3.0 → v0.3.20 用了 29 天） |

---

## Multica 在 agent 生态中的位置

```
层 1：Agent CLI/IDE       Claude Code, Codex, Cursor, Pi, OpenCode
                          ↑ Multica 在这些之上运行
层 2：自主编码代理         Devin, OpenHands, SWE-agent
                          （单一 agent 端到端，替代路径）
层 3：代理管理平台         Multica, Statica, Agent Kanban
                          （Multica 的品类）
层 4：代理编排框架         CrewAI, LangGraph, AutoGen
                          （代码优先 vs 平台优先）
```

Multica 处于**层 3**——它是编排层，不是执行层。它不调用 LLM API，而是在用户本地机器上启动 agent CLI 子进程。

---

## 核心设计决策

| 决策 | 做了什么 | 为什么 |
|------|---------|--------|
| **多态行动者** | `assignee_type` + `assignee_id`（member/agent） | agent 和人在数据模型层面完全对等 |
| **本地执行** | agent 在用户机器上跑，server 只做调度 | 代码不经过 Multica 服务器，安全+隐私 |
| **厂商中立** | 支持 12 种 agent CLI，不绑定任何 LLM | 避免 vendor lock-in |
| **Squads** | agent 分组到 leader 下，分配给 squad | 随团队增长保持路由稳定 |
| **Skills** | Markdown + 文件 → 注入 agent 工作目录 | 团队知识可复用、可沉淀 |
| **Session Resumption** | 同一 (agent, issue) 对复用上次 session | 保持对话上下文连续 |

---

## 外部资料

### 官方/一手来源

| 来源 | 链接 | 补什么 |
|------|------|--------|
| GitHub 仓库 | [multica-ai/multica](https://github.com/multica-ai/multica) | 完整源码 |
| 官网 | [multica.ai](https://multica.ai) | 产品定位、功能介绍 |
| X/Twitter | [@MulticaAI](https://x.com/MulticaAI) | 官方动态 |
| 创始人 X | [@jiayuan_jy](https://x.com/jiayuan_jy) | Building in public |
| 自部署指南 | [SELF_HOSTING.md](https://github.com/multica-ai/multica/blob/main/SELF_HOSTING.md) | Docker Compose / K8s 部署 |
| 产品全景文档 | [docs/product-overview.md](https://github.com/multica-ai/multica/blob/main/docs/product-overview.md) | 983 行完整产品文档 |

### 媒体报道与评测

| 来源 | 链接 | 补什么 |
|------|------|--------|
| 品玩 PingWest | [报道](https://www.pingwest.com/a/313471) | 创始人背景、团队配置、融资信息 |
| 腾讯新闻 | [报道](https://news.qq.com/rain/a/20260506A035P500) | 商业化计划 |
| NewsGlobeNow | [报道](https://www.newsglobenow.com/new326973.html) | 融资详情、商业模式 |

### 视频资料

| 来源 | 链接 | 播放量 | 补什么 |
|------|------|--------|--------|
| Better Stack | [Multica: The Open Source Tool That Makes Claude Code 10x Better](https://www.youtube.com/watch?v=WdGSXQPwwmo) | 57.3K | 最受欢迎的介绍视频 |
| Fru Dev | [Add 15 AI Agents to Multica in 60 Seconds](https://www.youtube.com/watch?v=9WLqcxetvAg) | 667 | 快速上手 |

### 社区博客

| 来源 | 链接 | 补什么 |
|------|------|--------|
| DEV Community | [Forget Solo Coding](https://dev.to/) | Multica 作为 agent 团队管理工具的介绍 |
| toolchew | [Running a 16-Agent Team on a €4.49/mo Server](https://toolchew.com/) | 低成本自托管实战 |
| Stork.AI | [Multica: The Ultimate Open-Source Manager](https://stork.ai/) | Claude Code agent 管理场景 |

---

## 更新日志

| 日期 | 内容 |
|------|------|
| 2026-06-11 | 初始深度分析报告，基于 v0.3.20 版本 |
