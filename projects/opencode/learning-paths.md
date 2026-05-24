# OpenCode 学习路径

> **核心目的：** 让读者在有限时间内获得最大理解，而不是读完所有文件。  
> **基准版本：** [`7fe7b9f2`](https://github.com/anomalyco/opencode/commit/7fe7b9f258e36ad9f9acded20c5a9df201da19d5)

根据你的目标选 **一条** 路径。社区资料可作为补充，见 [external-resources.md](./external-resources.md)。

---

## 路径 A：第一次学 OpenCode ⏱ 约 3–5 小时

**适合：** 第一次接触 OpenCode，需要建立「它到底怎么跑起来」的整体感。

| 步骤 | 文档 | 时长 | 社区补充（可选） |
|------|------|------|------------------|
| 1 | [18 主链路总图](./18-main-chain-atlas.md) | 15 min | [ZeroZ flow/agent_lifecycle](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/agent_lifecycle.md) |
| 2 | [A1 叙事 Trace](./appendix/A1-full-trace-narrative.md) | 45 min | — |
| 3 | [09 runLoop](./09-session-prompt-runloop.md) | 40 min | — |
| 4 | [A2 Processor](./appendix/A2-processor-deep-dive.md) | 30 min | — |
| 5 | [19 多模型](./19-multi-model-and-provider-system.md) | 45 min | [qqzhangyanhua 第 5 章](https://github.com/qqzhangyanhua/learn-opencode-agent/tree/main/docs/05-provider-system) |
| 6 | [17 §4 自测](./17-architecture-review-and-mastery.md) | 20 min | — |

**进度 Checklist**

- [ ] 能画出：HTTP → SessionPrompt → Processor → llm/request → Tool 的主链路
- [ ] 能说出：`chat.message` 与 `chat.params` 分别在什么阶段触发
- [ ] 能解释：model 选择的优先级（user / agent / config / default）
- [ ] 能回答 [18 文末三问](./18-main-chain-atlas.md#读完后应能回答)

---

## 路径 B：读懂内核（深读） ⏱ 约 1–2 天

**适合：** 要改 `packages/opencode` 或 debug runLoop / stream / compaction。

**顺序：**

```
18 → A1 → 08 → 09 → A2 → 10 → 19 → 11 → 12 → 13 → flow/ → 17
```

| 阶段 | 文档 | 关注点 |
|------|------|--------|
| 地图 + 叙事 | 18, A1 | 全局锚点 |
| 存储 + 循环 | 08, 09 | MessageV2、runLoop、SessionRunState |
| 流 + 模型 | A2, 10, 19 | stream 事件、chat.params、Provider |
| 工具 + 上下文 | 11, 12 | ToolRegistry、Task、Compaction |
| 横切 | 13, [flow/](./flow/README.md) | Bus、权限流、插件加载 |
| 验收 | 17 | 架构评价 + 自测 |

**进度 Checklist**

- [ ] 读过 `session/prompt.ts` 的 runLoop 分支（subtask / compaction / 正常 LLM）
- [ ] 读过 `session/processor.ts` 并对照 A2 事件表
- [ ] 读过 `session/llm/request.ts` 的 hook 注入顺序
- [ ] 能独立定位：「工具没执行」 vs 「LLM 没返回 tool-call」 分别该看哪几个文件
- [ ] 完成 [17 §4](./17-architecture-review-and-mastery.md) 全部自测

**社区补充：** ZeroZ [`internals/session.md`](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/internals/session.md)（模块百科，路径以本笔记 baseline 为准核对）

---

## 路径 C：写 OpenCode 插件 ⏱ 约 4–8 小时

**适合：** 要写 `chat.params`、注册 tool、或理解 hook 触发时机。

```
18 → A1（浏览）→ 19 → 10 → 06 → 05 → 16
```

| 步骤 | 文档 | 产出 |
|------|------|------|
| 1 | 18 + A1 浏览 | 知道 hook 注入点 |
| 2 | 19 + 10 | 理解 model / variant / 每 step 的 chat.params |
| 3 | 06 + 05 | Hook 全表 + 插件加载 |
| 4 | 16 | 动手写最小插件 |

**若同时使用 oh-my-openagent：** 读完 16 后读 [边界对照](../../comparisons/opencode-vs-omo-boundary.md) → [OmO 17 消息 Hook 链](../oh-my-openagent/17-message-hook-journey.md)。

**进度 Checklist**

- [ ] 能列出一次 LLM 轮次中必触发的 hook（见 [18 §3](./18-main-chain-atlas.md)）
- [ ] 能说明 `config` hook 与 `chat.params` hook 的区别
- [ ] 写出一个只改 `chat.params` 的最小插件并验证

**社区补充：** ZeroZ [Cookbook 01](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/cookbook/01-create-custom-agent.md)

---

## 路径 D：架构扫读 ⏱ 约 1–2 小时

**适合：** 技术负责人、评估是否采用 OpenCode 作为 agent runtime。

```
00 → 01 → 02 → 04 → 18 → 99
```

**社区补充：** ZeroZ [architecture/README](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/architecture/README.md)（含系统架构图）

**进度 Checklist**

- [ ] 能说明 OpenCode 与「纯 CLI 包装 LLM」的架构差异（Instance、Session DB、Plugin）
- [ ] 能列出 monorepo 中你实际会改的包（通常只有 `packages/opencode` + `packages/plugin`）
- [ ] 能判断：你的需求该改内核还是写插件

---

## 路径 E：客户端与全栈（按需） ⏱ 半天+

**适合：** 要改 Web UI、Desktop 或理解 Server-Driven 同步。

本笔记 [01 monorepo](./01-monorepo-and-packages.md) 仅作概要；**深度包分析请读 ZeroZ：**

- [`packages/app`](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/packages/app/README.md)
- [`packages/desktop`](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/packages/desktop/README.md)
- [`flow/state_sync`](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/state_sync.md)

---

## 读完后往哪走

| 下一步 | 链接 |
|--------|------|
| 概念速查 | [99 术语表与阅读地图](./99-glossary-and-reading-map.md) |
| 流程专题索引 | [flow/](./flow/README.md) |
| 学 OmO 插件层 | [oh-my-openagent](../oh-my-openagent/README.md) |
| 社区资料对照 | [external-resources.md](./external-resources.md) |

→ **回到入口：** [README](./README.md)
