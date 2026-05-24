# Hermes Agent 学习笔记

> **官方仓库：** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
> **基准：** pin [`889903f`](https://github.com/NousResearch/hermes-agent/commit/889903f0fa4ba125c56acf21030b5b99c99778db)（2026-05-24）  
> **文档站：** [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)  
> **开发指南：** 仓库内 [`AGENTS.md`](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md)

Hermes Agent 是 Nous Research 的 **自进化 Python Agent**：CLI + Gateway 双入口、OpenAI 格式 tool loop、Skills/Memory 闭环、多平台消息（Telegram/Discord/…）、七类终端后端。

本笔记从 **源码** 读架构，与官方 user-guide 互补（user-guide 讲用法，本笔记讲 `run_agent` / `conversation_loop` / `gateway` 脊柱）。

---

## 从这里开始

| 你是谁 | 第一步 | 完整路径 |
|--------|--------|----------|
| 第一次学 Hermes | **[26 主链路总图](./26-main-chain-atlas.md)** | [路径 A · 约 4–6h](./learning-paths.md#路径-a第一次学-hermes--约-46-小时) |
| 内核深读 | [05 AIAgent loop](./05-aiagent-and-conversation-loop.md) | [路径 B · 3–5 天](./learning-paths.md#路径-b内核深读--约-35-天) |
| 写插件/工具 | [16 Plugins/MCP](./16-plugins-mcp-and-hooks.md) | [路径 C](./learning-paths.md#路径-c插件与工具--约-1-天) |
| 多 Profile / 多 bot | [21 Profile/Pool](./21-profiles-and-credential-pool.md) | [路径 E](./learning-paths.md#路径-e-multi-agent-与运维--约-05-1-天) |
| Gateway 集成 | [03 CLI/Gateway](./03-cli-gateway-and-entry.md) | [路径 D](./learning-paths.md#路径-dgateway-与消息平台--约-1-天) |

**推荐起手：** `00 → 26 → 05 → 01 → 02`

---

## 文档分层

| 类型 | 文档 | 用途 |
|------|------|------|
| **地图** | [26 总图](./26-main-chain-atlas.md) | 5 分钟全局 |
| **架构** | [01 总览](./01-architecture-overview.md) | 四层 + 依赖链 |
| **内核** | [05 Loop](./05-aiagent-and-conversation-loop.md) | `run_conversation` 深读 |
| **入口** | [03 CLI/Gateway](./03-cli-gateway-and-entry.md) | 双入口与配置加载 |
| **路径** | [learning-paths](./learning-paths.md) | Checklist |
| **速查** | [99 索引](./99-glossary-and-reading-map.md) | 术语 → 章节 |

---

## 概念目录（全集）

| 层级 | 编号 | 主题 | 状态 |
|------|------|------|------|
| **A 定位** | [00](./00-philosophy-and-positioning.md) | 理念、与 CC/OpenCode 边界 | ✅ |
| **B 骨架** | [01](./01-architecture-overview.md) | 四层架构、模块依赖 | ✅ |
| | [02](./02-config-iteration-and-model-routing.md) | 配置、iteration budget | ✅ |
| | [03](./03-cli-gateway-and-entry.md) | CLI、Gateway、TUI 入口 | ✅ |
| **C 内核** | [05](./05-aiagent-and-conversation-loop.md) | AIAgent、`conversation_loop` | ✅ |
| **Atlas** | [26](./26-main-chain-atlas.md) | 主链路总图 | ✅ |
| **D 工具与会话** | [06](./06-tools-registry-and-model-tools.md)–[11](./11-delegation-cron-and-kanban.md) | Registry、Session、Skills、Gateway、编排 | ✅ |
| **E 内核加深** | [13](./13-prompt-assembly-and-cache.md)–[17](./17-terminal-backends.md) | Prompt、压缩、Provider、插件、Terminal | ✅ |
| **F 多 Agent+运维** | [18](./18-multi-agent-panorama.md)–[21](./21-profiles-and-credential-pool.md) | MoA、Goals、Security、Profile | ✅ |
| **G 子系统** | [22](./22-browser-and-web-tools.md)–[36](./36-kanban-board-internals.md) | Web、PTC、护栏、LSP、Voice、Trajectory… | ✅ |
| **导航** | [learning-paths](./learning-paths.md)（A–G 七路径） · [99](./99-glossary-and-reading-map.md) · [12](./12-coverage-gaps-and-external-resources.md) | 路径 · 术语/模块索引 · Wiki 对照 | ✅ |

**进度：** 核心专篇 + **集成手册 [22](./22-integrations-handbook.md)**；重复薄篇已并入父章（33→09，34→10，35→05/15）。

### 文档质量原则（言之有物）

一篇合格专篇 **至少满足下列 4 项中的 3 项**：

| 要求 | 反例（泛泛而谈） |
|------|------------------|
| **源码锚点** — `file:line` 或具名函数 + 行为 | 只有「Gateway 负责消息路由」 |
| **一条 happy path** — 从入口到写回 `messages` | 只有组件框图 |
| **一条边界/失败** — interrupt、cache miss、block 条件 | 只列功能表 |
| **与相邻模块分界** — 谁调用谁、谁不改什么 | 重复 Wiki 概念名 |

**篇幅不是 KPI。** 80 行但每段可带读源码 = 合格；150 行纯表格 = 不合格。

### 结构策略（合并 vs 独立）

| 策略 | 适用 |
|------|------|
| **独立加厚** | 内核脊柱：05、06、08、13、14… — 学习路径单独一步 |
| **并入父章** | 与父篇 50%+ 交叉、单独读 <15min：如 33→09、34→10 |
| **集成手册** | 可选子系统（Browser、Voice、LSP…）— 合并为 [22 手册](./22-integrations-handbook.md)，旧编号留 redirect |

**不再扩写** 40 行以下的独立小篇；新内容进父章或 22 手册对应节。

| 层级 | 文档 | 状态 |
|------|------|------|
| **脊柱** | 05、13、06、08、14、10 | 05/13/10 已加厚；06/08/14 待续 |
| **扩展** | 07–11、15–21 | 中等篇幅，按 Wiki 补案例 |
| **集成** | [22 手册](./22-integrations-handbook.md) | 替代 22–31、36 薄篇 |
| **已合并** | 33→[09](./09-skills-curator-and-learning-loop.md#10-skills-与-memory-协作) · 34→[10](./10-gateway-platforms-and-sessions.md#11-会话-db-与-platformregistry) · 35→[05](./05-aiagent-and-conversation-loop.md#5-loop-内关键机制iteration-级) + [15](./15-provider-and-transport.md#6-auxiliary-侧任务路由) |

---

## 外部学习资源（经筛选）

| 等级 | 资源 | 用途 |
|------|------|------|
| **A** | [Hermes-Wiki](https://github.com/cclank/Hermes-Wiki) | 45 概念页架构百科；缺专题时首选深读 |
| **A** | [官方 Developer Guide](https://hermes-agent.nousresearch.com/docs/) | 权威 developer-guide（tools-runtime、gateway-internals 等） |
| **A** | [AGENTS.md](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md) | 贡献政策与硬约束 |
| **B+** | [mudrii/hermes-agent-docs](https://github.com/mudrii/hermes-agent-docs) | 社区文档索引（developer/user/reference） |
| **B** | [hermes-principle](https://github.com/relunctance/hermes-principle) | 中文 Gateway/Skills/Cron 补充；需与源码核对 |

完整对照表与 P0–P2 补写建议 → **[12 覆盖缺口与外链](./12-coverage-gaps-and-external-resources.md)**。

---

## 与 Claude Code / OpenCode 对照（后期）

| 概念 | Hermes | Claude Code [../claude-code/](../claude-code/README.md) | OpenCode [../opencode/](../opencode/README.md) |
|------|--------|------------------------------------------------------|-----------------------------------------------|
| 语言 | Python | TypeScript/Bun | TypeScript/Bun |
| 主 loop | `agent/conversation_loop.py` | `query.ts` | `SessionPrompt.loop` |
| 工具注册 | `tools/registry.py` | `tools.ts` + `Tool.ts` | `ToolRegistry` |
| 双入口 | CLI + Gateway | REPL + print/SDK | CLI + Server |
| 会话 | SQLite `SessionDB` | JSONL uuid 链 | SQL + Bus |

---

## 本地源码

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent && git checkout 889903f0fa4ba125c56acf21030b5b99c99778db
source .venv/bin/activate   # 或 venv
scripts/run_tests.sh tests/agent/ -q   # 勿直接 pytest
```

本地路径：`~/Github/hermes-agent/`
