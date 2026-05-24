# Claude Code 学习路径

> **基准：** v2.1.88 · pin [`936e6c8`](https://github.com/zhumengzhu/claude-code/commit/936e6c8e8d7258dd1b2bc127d704f02cc23076d5)  
> **完整目录：** [README § 概念目录](./README.md#概念目录)

根据目标选 **一条** 路径。社区资料见 [external-resources.md](./external-resources.md)。

---

## 路径 A：第一次学 Claude Code ⏱ 约 4–6 小时

**适合：** 建立「它怎么跑起来」的整体感。

| 步骤 | 文档 | 时长 |
|------|------|------|
| 1 | [00 理念与边界](./00-philosophy-and-exposure.md) | 20 min |
| 2 | **[26 主链路总图](./26-main-chain-atlas.md)** | 20 min |
| 3 | [01 架构总览 §3](./01-architecture-overview.md#3-运行时脊柱l0) | 30 min |
| 4 | [A1 叙事](./appendix/A1-user-turn-journey.md) | 45 min |
| 5 | 带读 `query.ts` queryLoop | 2 h |
| 6 | [10 压缩/cache](./10-compaction-and-context.md) 浏览 | 45 min |
| 7 | [25 §自测](./25-architecture-review-and-mastery.md#4-入门自测) | 20 min |

**Checklist**

- [ ] 能画出：入口 → QueryEngine → queryLoop → callModel → tools
- [ ] 能说出 compact 与 callModel 的先后顺序
- [ ] 能解释 `needsFollowUp` 与 loop 终止的关系

---

## 路径 B：内核深读 ⏱ 约 3–5 天

**适合：** 改 agent 行为、debug tool/compact/permission。

```
26 → A1 → 05 → 06 → **28** → 07 → 08 → 09 → 10 → 11 → 15 → flow/ → 25
```

| 步骤 | 文档 | 关注点 |
|------|------|--------|
| 地图 | 26, 01 | 全局锚点 |
| 引擎 | 05 QueryEngine, 06 query loop | submitMessage、transition |
| API / 消息 | 07, 08 | 流式、fallback、JSONL |
| 工具 / 上下文 | 09, **10** | tools、[渐进式压缩与 cache](./10-compaction-and-context.md) |
| 权限 / 扩展 | 11, **15**, 13, 14 | permission、plugin、hooks、MCP |
| 流程 | [flow/](./flow/README.md) | 横切流程索引 |
| 验收 | 25 | 架构评价 + 自测 |

**Checklist**

- [ ] 读过 `query.ts` 每个 `continue` 的 `transition.reason`
- [ ] 读过 `StreamingToolExecutor` 与 `runTools` 的分工
- [ ] 读过 `recordTranscript` 与 resume 路径
- [ ] 完成 [25 §5 深读自测](./25-architecture-review-and-mastery.md#5-深读自测)

---

## 路径 C：SDK / Headless ⏱ 约 1 天

**适合：** `-p`、`stream-json`、Python/TS SDK 集成。

```
26 → 19 → 07 → 08 → 05（submitMessage SDK 映射部分）
```

**Checklist**

- [ ] 能说明 `runHeadless` → `runHeadlessStreaming` → `ask()` 调用链
- [ ] 能区分 `StructuredIO` 与 REPL 的差异
- [ ] 知道 headless 下 permission 如何实现（无 React 树）

---

## 路径 D：架构扫读 ⏱ 约 1–2 小时

**适合：** 评估是否借鉴 Claude Code 的 agent 设计。

```
00 → 01 → 26 → 25 §1–§3
```

---

## 路径 E：扩展专题（按需）

| 目标 | 文档 |
|------|------|
| IDE 集成 | [18 bridge](./18-bridge-and-ide.md) |
| 多 Agent | [20 subagent](./20-agents-and-subagents.md) → [21 team/swarm/coordinator](./21-tasks-team-and-coordinator.md) |
| Memory | [29 memory](./29-memory-and-auto-memory.md) |
| Loop 续跑 / 人机门控 | [28 agent loop gates](./28-agent-loop-continuation-and-human-gates.md) |
| Tool Search / 实验 | [30 advanced features](./30-advanced-features-and-experiments.md) |
| Remote / SDK | [22 remote](./22-remote-and-server-mode.md) · [19 SDK](./19-sdk-headless-and-print-mode.md) |
| Plan mode | [17 plan mode](./17-plan-mode-and-code-editing.md) |
| yolo / auto mode | [11 permission §4](./11-permission-and-hooks.md#4-auto-mode-与-yolo-语义) |
| 配置 | [04 config](./04-config-and-settings.md) |
| CLI 全 flag | [03 CLI §5](./03-cli-entry-and-repl.md#5-cli-flag-速查按主题) |

---

## 路径 F：部署与集成 ⏱ 按需

**适合：** IDE 插件、SDK、CCR、成本监控。

```
03 §5 flags → 19 SDK → 22 Remote → 18 Bridge → 24 Cost
```

**Checklist**

- [ ] 能说明 stream-json 事件类型与 turn 生命周期
- [ ] 能解释 CCR 120s timeout 与 activity signal
- [ ] 能配置 CCR memory 挂载路径

---

## 读完后往哪走

| 下一步 | 链接 |
|--------|------|
| 概念速查 | [99 术语表与阅读地图](./99-glossary-and-reading-map.md) |
| 流程专题 | [flow/](./flow/README.md) |
| 社区资料 | [external-resources.md](./external-resources.md) |

→ [README 入口](./README.md)
