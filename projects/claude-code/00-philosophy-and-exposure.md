# 00 · 理念、暴露背景与阅读边界

> **基准：** v2.1.88 · pin [`936e6c8`](https://github.com/zhumengzhu/claude-code/commit/936e6c8e8d7258dd1b2bc127d704f02cc23076d5)

---

## 1. 这份笔记在学什么

Claude Code 是 Anthropic 的 **terminal coding agent 产品**：在终端里读代码、改文件、跑命令、调 MCP、spawn 子 Agent。

本目录研究的是 **2026-03-31 从 npm sourcemap 暴露的 TypeScript 快照（v2.1.88）**，目的：

- 理解 **Agent CLI 的产品级架构**（不是 toy demo）
- 学习 **query loop + tool orchestration + context 治理** 的工程做法
- 建立 **defensive 源码阅读** 习惯（暴露材料 ≠ 可依赖的发行物）

**不目的：**

- 不当作官方 SDK 或二次开发依赖
- 不用于规避许可或服务条款
- 不在未 pin commit 的前提下与当前 npm 产品行为硬对齐

---

## 2. 暴露事件与 baseline

| 项 | 说明 |
|----|------|
| 暴露时间 | 2026-03-31 |
| 来源 | `@anthropic-ai/claude-code` npm 包内 `cli.js.map` 指向 R2 上的 TS 源 |
| 公开讨论 | [@Fried_rice 推文](https://x.com/Fried_rice/status/2038894956459290963) |
| 本笔记 pin | [zhumengzhu/claude-code @ 936e6c8](https://github.com/zhumengzhu/claude-code/commit/936e6c8e8d7258dd1b2bc127d704f02cc23076d5) |
| 规模 | ~1900 文件 · ~512k LOC · Bun · React/Ink |

**阅读纪律：** 文档结论以 pin commit 为准；CLI 已迭代的行为以 [官方文档](https://code.claude.com/docs/en/overview) 为准，二者可能不一致。

---

## 3. 三个仓库，别混读

| 仓库 | 是什么 | 本笔记 |
|------|--------|--------|
| [anthropics/claude-code](https://github.com/anthropics/claude-code) | 官方产品仓（文档、issue、集成） | 产品行为参照 |
| [zhumengzhu/claude-code](https://github.com/zhumengzhu/claude-code) | TS 暴露快照 + 研究说明 | **带读主线** |
| [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) | Rust 社区重写 | 仅作「另一种实现」对照，**非** v2.1.88 源码 |

---

## 4. 架构立场（读源码前先建立）

### 4.1 Agent 最小闭环

Claude Code 的「Agent 性」来自 **可循环的决策–行动–反馈**：

```text
用户意图 → LLM 决策 → tool_use → 执行 → tool_result → 再决策 → … → 终止
```

内核在 `query.ts` 的 `queryLoop`；不是「一问一答」的 chat wrapper。

### 4.2 四层优先级

| 层 | 含义 | 首读 |
|----|------|------|
| L0 | query loop、API、tool 执行 | 必读 |
| L1 | tools、compact、permission、prompt | 第二遍 |
| L2 | CLI、REPL、print/SDK | 后读 |
| L3 | bridge、team、remote、voice | 按需 |

512k LOC 中 **~389 文件在 `components/`**（Ink UI）。Agent 行为问题先查 L0/L1，不要从 UI 倒推。

### 4.3 设计取向（从代码可见）

- **Async generator 贯穿**：`query()`、`submitMessage()`、`queryModelWithStreaming()` 均可 yield 中间事件
- **Feature flag 裁剪**：`bun:bundle` 的 `feature()` 在构建时剔除内部分支
- **上下文优先于模型**：每轮 loop 开头先做 snip / microcompact / autocompact，再 `callModel`
- **权限即回调**：`canUseTool` 注入 tool 执行前，交互与 headless 各有一套实现

---

## 5. 阅读边界与伦理

- 快照来自 **供应链暴露**，适合安全研究与架构教育，**不适合**当作「合法获得的官方源码」直接商用
- fork README 中的 [companion essay](https://writings.hongminhee.org/2026/03/legal-vs-legitimate/) 讨论再实现与 copyleft，可作为伦理背景
- 本笔记 **只记录架构理解**，不提供「如何复刻闭源产品」的操作指南

---

## 6. 与其他笔记的关系

本目录 **34 篇正文** + 附录 + 导航，分层：

| 层级 | 编号 | 读法 |
|------|------|------|
| 地图 | 26、A1 | 先建立全局 |
| 内核 | 05–11、28 | 深读必过 |
| 扩展 | 12–17、09 | 工具/命令/MCP |
| 高级 | 18–24、20–21 | 按需专题 |
| 专题 | 27–30 | Memory/实验/模型 |

后期横向对照见仓库 `comparisons/`（OpenCode、OmO）。

---

## 7. 读完本章应能回答

- [ ] 本笔记的 pin commit 与官方发行仓各是什么？
- [ ] Agent loop 的内核文件是哪两个？
- [ ] 为什么建议先跳过 `components/`？
- [ ] 暴露快照适合做什么、不适合做什么？

**下一步：** [26 主链路总图](./26-main-chain-atlas.md) · [learning-paths.md](./learning-paths.md)
