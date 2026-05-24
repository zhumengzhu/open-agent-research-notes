# open-agent-research-notes

跨项目的 coding agent 学习与研究笔记。

## 仓库定位

- 语言：**中文优先**
- 属性：**个人学习仓库**
- 目标：理解 agent 系统如何被构造、如何协作、如何扩展

## 目标

- 统一沉淀 `oh-my-openagent`、`opencode`、`openclaw`、`claude code` 的学习记录
- 形成可复用的架构分析方法（系统关系、边界、不变量、优缺点评价）
- 为插件开发与内核阅读提供知识底座

## 项目索引

| 项目 | 内容简介 |
|------|----------|
| [oh-my-openagent](./projects/oh-my-openagent/) | OpenCode 插件协议、hook 体系、多 agent 编排、tool/MCP/Team Mode |
| [opencode](./projects/opencode/) | OpenCode 内核（00–19 + 附录 A1/A2）：叙事 Trace、多模型、Processor 深读 |
| [openclaw](./projects/openclaw/) | 外部系统与 agent 会话的双向集成 |
| [claude-code](./projects/claude-code/) | Claude Code 扩展生态与工具协议 |

**跨项目对照：** [OpenCode ↔ OmO 边界](./comparisons/opencode-vs-omo-boundary.md)

## 目录

- `projects/`：按项目分区的学习笔记
- `comparisons/`：跨项目横向比较
- `patterns/`：可复用设计模式与反模式
- `templates/`：新项目学习模板

## 建议使用方式

1. 从 `projects/<name>/README.md` 进入对应项目的概念目录
2. 每完成一轮学习，把可复用的结论沉淀到 `comparisons/` 和 `patterns/`
3. 每个结论附带证据来源（源码路径或文档链接）

## 写作约定

- 默认使用中文书写（术语可保留英文）
- 记录「结论 + 证据 + 边界条件」，避免只写结论
- 优先沉淀可复用的分析方法，而不是单次问题的临时笔记
