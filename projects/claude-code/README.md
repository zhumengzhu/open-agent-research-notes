# claude-code

> **官方产品仓库：** [anthropics/claude-code](https://github.com/anthropics/claude-code)  
> **官方文档：** [code.claude.com/docs](https://code.claude.com/docs/en/overview)  
> **社区 Rust 重写 upstream：** [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code)（Rust · 基于 oh-my-codex）  
> **个人 fork / 研究快照：** [zhumengzhu/claude-code](https://github.com/zhumengzhu/claude-code)（自 `ultraworkers/claw-code` fork；含 TS 暴露快照与架构研读材料）

Claude Code 是 Anthropic 的 terminal agent 产品。本目录用于沉淀**与 OpenCode / OmO 的概念对照**，以及对你 fork 中公开快照的** defensive 架构阅读**（非官方源码维护向）。

## 三个仓库各是什么

| 仓库 | 语言 / 形态 | 本笔记怎么用 |
|------|-------------|--------------|
| [anthropics/claude-code](https://github.com/anthropics/claude-code) | 官方发行仓（文档、issue、Actions 集成；**非完整应用源码**） | 产品行为、CLI 能力、官方扩展点的**权威参照** |
| [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) | **Rust** 社区重写 | 对照「若从零用 Rust 复刻 agent CLI」的实现选择 |
| [zhumengzhu/claude-code](https://github.com/zhumengzhu/claude-code) | TS **暴露快照**（~1900 文件）+ 研究说明 | **带读** QueryEngine、Tool、Bridge、Permission 等（pin 你 fork 的 commit） |

**注意：** upstream `claw-code` 已是 Rust 栈；你 fork 里的 TypeScript `src/` 快照来自另一套公开暴露材料，与 Rust upstream **不是同一条代码线**。读架构时先确认当前打开的是哪个仓库。

## 建议阅读顺序

1. 官方文档：安装、agents、hooks、MCP
2. 你的 fork README：暴露背景与目录地图
3. 按需对照：[opencode 内核](../opencode/README.md) · [OmO 插件](../oh-my-openagent/README.md)
4. 社区（可选）：[qqzhangyanhua claude-code 篇](https://github.com/qqzhangyanhua/learn-opencode-agent/tree/main/docs/claude-code)

## 待沉淀主题

- Tool / Command / QueryEngine 与 OpenCode `session/` 的概念映射
- Bridge、Plugin、Skill 与 OpenCode `plugin` / OmO hooks 的对照
- Permission 模型 vs OpenCode PermissionNext
- `comparisons/`：Claude Code ↔ OpenCode 横向表（待写）

## 关联

| 系统 | 本仓库笔记 |
|------|------------|
| OpenCode 内核 | [../opencode/README.md](../opencode/README.md) |
| OmO 插件 | [../oh-my-openagent/README.md](../oh-my-openagent/README.md) |
