# 社区学习资源对照

> **用途：** 知道「什么时候读社区资料、什么时候读本笔记」——取长补短，避免重复劳动。  
> **本笔记基准：** [`7fe7b9f2`](https://github.com/anomalyco/opencode/commit/7fe7b9f258e36ad9f9acded20c5a9df201da19d5)

---

## 1. 两个值得对照的仓库

### [ZeroZ-lab/learn-opencode](https://github.com/ZeroZ-lab/learn-opencode)

| 维度 | 说明 |
|------|------|
| 形态 | `docs/` 笔记 + `source/opencode` 子模块 + `examples/` |
| 强项 | Monorepo **包级**分析（app / desktop / console / sdk…）、`flow/` 流程文档、Cookbook、FAQ、三条学习路径 + Checklist |
| 弱项 | 部分路径引用可能随 upstream 过时；`session/` 深读不如本笔记 A2 |
| 建议读 | `learning_paths.md` → `flow/agent_lifecycle.md` → `packages/opencode/README.md` Module 0 → `internals/session.md`（与 A2 对照） |

### [qqzhangyanhua/learn-opencode-agent](https://github.com/qqzhangyanhua/learn-opencode-agent)

| 维度 | 说明 |
|------|------|
| 形态 | VitePress 站点：OpenCode 24 章 + 实践篇 + 中级篇 + OmO 专章 |
| 强项 | **阅读地图**、理论↔实践映射表、`oh-flow` 插件层叙事、OmO hook 分层（16/18 章） |
| 弱项 | 源码基线较旧（`f8475649`）；部分 OmO 链接用 `dev` 分支；Hook 数量等数字可能滞后 |
| 建议读 | `reading-map.md` → 第 4 章会话管理 → **`oh-flow`**（插件视角）→ `16-plugin-overview`（若用 OmO） |

---

## 2. 三方对照：谁解决什么问题

| 你想搞懂… | 优先读本笔记 | 可补读社区 |
|-----------|-------------|------------|
| 5 分钟建立全局 | [18 总图](./18-main-chain-atlas.md) | ZeroZ `flow/agent_lifecycle.md` |
| 跟一条消息走完整内核 | [A1 叙事](./appendix/A1-full-trace-narrative.md) | qqzhangyanhua 第 4 章（概念层） |
| 跟一条消息走 **OmO 插件 hook 链** | **[OmO 17](../oh-my-openagent/17-message-hook-journey.md)** + [边界对照](../../comparisons/opencode-vs-omo-boundary.md) | qqzhangyanhua [oh-flow](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/oh-flow/index.md) |
| Processor / stream 事件 | [A2 深读](./appendix/A2-processor-deep-dive.md) | （社区基本无同等深度） |
| 多模型 / variant / small_model | [19 专题](./19-multi-model-and-provider-system.md) | qqzhangyanhua 第 5、17 章 |
| Monorepo 全包地图 | [01 包分层](./01-monorepo-and-packages.md) | ZeroZ `architecture/` + 各 `packages/*/README` |
| UI / Desktop / Console 客户端 | [01 §客户端包](./01-monorepo-and-packages.md)（概要） | ZeroZ `packages/app`、`desktop`、`console` |
| 写第一个插件 | [16 作者指南](./16-plugin-author-guide.md) | ZeroZ Cookbook `01-create-custom-agent` |
| 权限 / 工具执行流程 | [07](./07-agent-and-permission.md) + [11](./11-tool-registry-and-execution.md) | ZeroZ `flow/permission_flow`、`flow/tool_execution` |
| 从零搭 Agent（练手） | （本笔记不做教程平台） | qqzhangyanhua **实践篇 P1–P4** |

---

## 3. 从社区借鉴、在本笔记已落实的设计

| 借鉴来源 | 做法 | 本笔记落点 |
|----------|------|------------|
| ZeroZ | 多条路径 + 时间估算 + Checklist | [learning-paths.md](./learning-paths.md) |
| ZeroZ | `flow/` 流程专题独立索引 | [flow/README.md](./flow/README.md) |
| qqzhangyanhua | 核心概念 → 章节速查 | [99 §2](./99-glossary-and-reading-map.md) |
| qqzhangyanhua | 「不要线性读全书」的入口警告 | 本 README「从这里开始」 |
| qqzhangyanhua | 源码快照 / 阅读边界说明 | 各文顶部 baseline + [99 §6](./99-glossary-and-reading-map.md) |
| qqzhangyanhua | OmO「一条消息旅程」叙事 | [oh-my-openagent/17](../oh-my-openagent/17-message-hook-journey.md) |
| 本笔记 | 重要节点 SVG 插图 | [18 assets](../opencode/assets/) · [boundary assets](../comparisons/assets/) · [OmO 17 assets](../oh-my-openagent/assets/) |
| 本笔记独有 | pin SHA 证据链 | [AGENTS.md](../../AGENTS.md) |
| 本笔记独有 | 内核 vs 插件边界 | [comparisons/boundary](../../comparisons/opencode-vs-omo-boundary.md) |

---

## 4. 不建议照搬的部分

- qqzhangyanhua 的 `blob/dev/` 链接习惯（分支 HEAD 会变）
- ZeroZ 文档里未经验证的文件路径（读时用本笔记 baseline 交叉核对）
- 整站 VitePress + 28 个实践项目（本仓库定位是**研究笔记**，不是课程平台）

---

→ **回到入口：** [README](./README.md) · **选路径：** [learning-paths.md](./learning-paths.md)
