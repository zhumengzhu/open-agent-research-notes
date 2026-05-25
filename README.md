# open-agent-research-notes

跨项目的 coding agent 学习与研究笔记。

**核心目的：** 帮助读者真正理解 agent 系统如何构造、协作与扩展——每条结论尽量附带可核对的源码证据，而不是只给结论。

## 仓库定位

- 语言：**中文优先**
- 属性：**个人学习仓库**（不是产品代码、不是 changelog）
- 首要读者：正在读 **OpenCode 内核** 或在其上写插件的开发者

## 项目索引

| 项目 | 官方仓库 | 内容简介 | 入口 |
|------|----------|----------|------|
| **[opencode](./projects/opencode/)** | [anomalyco/opencode](https://github.com/anomalyco/opencode) | OpenCode **runtime 内核**：Atlas、叙事 Trace、Processor 深读、多模型 | [开始学习](./projects/opencode/README.md) |
| [oh-my-openagent](./projects/oh-my-openagent/) | [code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | 插件协议、hook 体系、多 agent 编排、tool/MCP/Team Mode | [目录](./projects/oh-my-openagent/README.md) |
| [openclaw](./projects/openclaw/) | [openclaw/openclaw](https://github.com/openclaw/openclaw) | 个人 AI 助手 Gateway、多通道 inbox（独立产品） | [目录](./projects/openclaw/README.md) |
| [claude-code](./projects/claude-code/) | [anthropics/claude-code](https://github.com/anthropics/claude-code) · [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) · [zhumengzhu/claude-code](https://github.com/zhumengzhu/claude-code) | 官方产品 + Rust 社区重写 + 个人 TS 研究快照 | [目录](./projects/claude-code/README.md) |
| **[pi](./projects/pi/)** | [earendil-works/pi](https://github.com/earendil-works/pi) | 极简主义 coding agent：4 工具 + 游戏引擎架构 + 激进扩展性 | [开始学习](./projects/pi/README.md) |

**跨项目对照：** [OpenCode ↔ OmO 边界](./comparisons/opencode-vs-omo-boundary.md)

---

## 社区 OpenCode 学习资源（取长补短）

本仓库专注 **内核深读 + pin commit 证据链**；下列社区资料在广度、叙事或实战上各有长处，建议对照阅读，不必重复造轮子。

| 仓库 | 链接 | 擅长补什么 |
|------|------|------------|
| **ZeroZ-lab/learn-opencode** | [github.com/ZeroZ-lab/learn-opencode](https://github.com/ZeroZ-lab/learn-opencode) | Monorepo 全景（app/desktop/console 等包）、`flow/` 流程专题、Cookbook 实战、FAQ、带源码子模块 |
| **qqzhangyanhua/learn-opencode-agent** | [github.com/qqzhangyanhua/learn-opencode-agent](https://github.com/qqzhangyanhua/learn-opencode-agent) | 24 章体系 + 实践项目、阅读地图与概念速查、**oh-flow「一条消息旅程」**（OmO 插件视角）、动画实验室 |

**对照阅读指南（详细）：** [projects/opencode/external-resources.md](./projects/opencode/external-resources.md)

**本笔记相对社区的差异化：**

- 内核主链路：**18 Atlas → A1 叙事 → A2 Processor 状态机 → 19 多模型**（社区资料较少同等深度）
- 证据链：正文链接 **pin commit SHA**，不用 `blob/dev/`
- 跨项目：**comparisons/** 明确 OpenCode 内核与 OmO 插件层的职责边界

---

## 目录结构

| 目录 | 用途 |
|------|------|
| `projects/` | 按项目分区的学习笔记 |
| `comparisons/` | 跨项目横向比较 |
| `patterns/` | 可复用设计模式与反模式 |
| `templates/` | 新项目学习模板 |

## 建议使用方式

1. **学 OpenCode 内核：** 从 [projects/opencode/README.md](./projects/opencode/README.md) 选一条学习路径，不要从 00 线性读到 17
2. **学插件层：** 先读 [边界对照](./comparisons/opencode-vs-omo-boundary.md) → [OmO 17 消息 Hook 链](./projects/oh-my-openagent/17-message-hook-journey.md)
3. 每完成一轮学习，把可复用的结论沉淀到 `comparisons/` 和 `patterns/`
4. 每个结论附带证据来源（源码路径或文档链接）

## 写作约定

- 默认使用中文书写（术语可保留英文）
- 记录「结论 + 证据 + 边界条件」，避免只写结论
- 优先沉淀可复用的分析方法，而不是单次问题的临时笔记

详细约定见 [AGENTS.md](./AGENTS.md)。
