# Claude Code 社区学习资源

> 针对 **2026-03-31 泄露的 TS 快照（v2.1.88）**，不是 [anthropics/claude-code](https://github.com/anthropics/claude-code) 官方发行仓，也不是 [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) Rust 重写。  
> **用法：** 预习 → 带读源码 → 以 [pin commit](https://github.com/zhumengzhu/claude-code/commit/936e6c8e8d7258dd1b2bc127d704f02cc23076d5) 为准核对。

---

##  Tier 1：优先配合本笔记

| 仓库 | Stars 量级 | 形态 | 最适合阶段 | 说明 |
|------|------------|------|------------|------|
| [Windy3f3f3f3f/how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works) | ~2.4k | 中英双语文档站 | **路径 A 步骤 1** | 15 篇专题：`02-agent-loop`、`03-context-engineering`、`04-tool-system` 等与本地模块名高度对齐 |
| [cablate/claude-code-research](https://github.com/cablate/claude-code-research) | — | 10 phase 分析报告 | **路径 B 各 phase** | `source-code-analysis/phase-03-agent-architecture` 等；偏逆向报告，适合深读前扫盲 |
| [qqzhangyanhua/learn-opencode-agent/docs/claude-code](https://github.com/qqzhangyanhua/learn-opencode-agent/tree/main/docs/claude-code) | — | 20 章 + reading-guide | **路径 A 步骤 5 / 路径 C** | 概念层架构思维，与 OpenCode 笔记同作者，便于横向对照 |

### Windy 文档目录（en/docs）

| 文件 | 对应本笔记 |
|------|------------|
| `01-overview.md` | 01–02 |
| `02-agent-loop.md` | 04–05, A1 |
| `03-context-engineering.md` | 07 |
| `04-tool-system.md` | 06 |
| `05-code-editing-strategy.md` | 06（FileEdit 等） |
| `06-hooks-extensibility.md` | 08–09 |
| `07-multi-agent.md` | 11 |
| `08-memory-system.md` | 07 |
| `09-skills-system.md` | 10 |
| `10-plan-mode.md` | 05, 11 |
| `11-permission-security.md` | 08 |
| `12-user-experience.md` | 03 |
| `13-minimal-components.md` | 02 |
| `14-system-prompt-design.md` | 05, cablate phase-01 |
| `15-task-system.md` | 11 |

### cablate phase 目录

| Phase | 主题 |
|-------|------|
| phase-01-system-prompt | System prompt 构建 |
| phase-02-tool-definitions | Tool schema / 注册 |
| phase-03-agent-architecture | QueryEngine / query loop |
| phase-04-skills-system | Skills |
| phase-05-memory-context | Memory / compact |
| phase-06-security-permissions | Permission |
| phase-07-api-model-architecture | API / model 路由 |
| phase-08-special-features | Plan mode 等 |
| phase-09-harness-engineering | Harness 工程化 |
| phase-10-cost-quota | 成本与配额 |

---

## Tier 2：补充阅读

| 仓库 | 特点 | 注意 |
|------|------|------|
| [soufianebouaddis/claude-code](https://github.com/soufianebouaddis/claude-code) | 泄露背景 + 长 README 架构深读 | 文档为主 |
| [Kuberwastaken/claude-code](https://github.com/Kuberwastaken/claude-code) | 早期备份 + 拆解文章 | 版本可能早于 v2.1.88，核对文件名 |
| [InlitX/claude-code-source](https://github.com/InlitX/claude-code-source) | 事件时间线与架构要点 | 短文档 |
| [l3tchupkt/claude-code](https://github.com/l3tchupkt/claude-code) | sourcemap 重建 + 流程图（print → QueryEngine → query → tools） | 图 helpful |
| [davccavalcante/claude-code-leaked](https://github.com/davccavalcante/claude-code-leaked) | 43 tools / 39 services 结构化清单 | 当索引用 |
| [zhumengzhu/claude-code](https://github.com/zhumengzhu/claude-code) | **本计划 baseline 源码 + 伦理说明** | 主线 |

---

## Tier 3：相关但非 TS 泄露主线

| 仓库 | 关系 |
|------|------|
| [anthropics/claude-code](https://github.com/anthropics/claude-code) | 官方产品、文档、Actions；**非**完整 TS 应用源码 |
| [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) | Rust 社区重写；学习「复刻 agent CLI」时可对照，模块名不一致 |
| [Yeachan-Heo/oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) | zhumengzhu fork README 提到的 OmX 工作流（文档辅助，非 CC 内核） |

---

## 推荐使用顺序

```
1. Windy quick-start + 02-agent-loop     （1h）
2. 本地 query.ts / QueryEngine.ts 带读    （3h）
3. cablate phase-03 当检查清单            （30min）
4. qqzhangyanhua reading-guide 查漏      （30min）
5. 按需深读 Windy 03–11 / cablate 其他 phase
```

**反模式：**

- 只读社区文档不打开 `query.ts`（会漏掉 generator / 边界条件）
- 把 claw-code Rust 模块名硬映射到 TS 快照
- 用 npm 最新版 CLI 行为反推 v2.1.88 源码（产品已迭代）

---

[← 返回 learning-paths](./learning-paths.md)
