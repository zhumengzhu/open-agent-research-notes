# openclaw

> **官方仓库：** [openclaw/openclaw](https://github.com/openclaw/openclaw)  
> **官网 / 文档：** [openclaw.ai](https://openclaw.ai) · [docs.openclaw.ai](https://docs.openclaw.ai)  
> **基准版本：** 笔记撰写时以 `main` 为准；正文举证请 pin 到具体 commit SHA

[OpenClaw](https://github.com/openclaw/openclaw) 是**独立产品**：本地优先的个人 AI 助手 Gateway，连接 WhatsApp / Telegram / Discord / Slack 等多通道，管理 session、skills 与工具。

与本仓库其它项目的关系：

| 系统 | 关系 |
|------|------|
| **OpenClaw（本目录主题）** | 独立开源项目，自有 Gateway 与多通道 inbox |
| **[oh-my-openagent](../oh-my-openagent/)** | OpenCode 插件；[`src/openclaw/`](https://github.com/code-yeongyu/oh-my-openagent/tree/20d67be496155473f49aef3207bfe9d3737cbfa8/src/openclaw) 提供与 OpenClaw **双向集成**（事件出站 + 回复入站） |
| **[opencode](../opencode/)** | OmO 所依赖的 agent runtime 内核 |

## 建议阅读顺序

1. OpenClaw 官方：[Getting started](https://docs.openclaw.ai) · [Architecture](https://docs.openclaw.ai)（文档站内）
2. 本仓库（待写）：Gateway、session 模型、多通道路由
3. 若你同时用 OmO：[13 · OpenClaw 双向回路](../oh-my-openagent/13-openclaw-and-external-loop.md)

## 待沉淀主题

- Gateway 控制面与 session 模型
- 多通道 inbox 与 agent 路由
- Skills / workspace 注入
- 与 OmO `openclaw` feature 的集成边界（产品 vs 插件桥接）
