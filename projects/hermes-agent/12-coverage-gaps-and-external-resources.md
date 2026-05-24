# 12 · 覆盖缺口与外部学习资源

> **文档策略（2026-05）：** 可选子系统合并为 [22 集成手册](./22-integrations-handbook.md)；33/34/35 并入 09/10/05+15。旧编号保留 redirect。质量原则见 [README §文档质量](./README.md#文档质量原则言之有物)。
> **基准：** pin `889903f`；Wiki 跟踪 HEAD — 读差异时以源码为准。

---

## 1. 外部资源（质量分级）

| 等级 | 资源 | 链接 |
|------|------|------|
| **A** | Hermes-Wiki | [cclank/Hermes-Wiki](https://github.com/cclank/Hermes-Wiki) |
| **A** | 官方 Developer Guide | [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/) |
| **A** | AGENTS.md | [NousResearch/hermes-agent/AGENTS.md](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md) |
| **B+** | hermes-agent-docs | [mudrii/hermes-agent-docs](https://github.com/mudrii/hermes-agent-docs) |
| **B** | hermes-principle | [relunctance/hermes-principle](https://github.com/relunctance/hermes-principle) |

---

## 2. Wiki 概念 → 本笔记映射（完整）

| Wiki 概念 | 本笔记 |
|-----------|--------|
| agent-loop-and-prompt-assembly | [05](./05-aiagent-and-conversation-loop.md) + [13](./13-prompt-assembly-and-cache.md) |
| prompt-builder / prompt-caching | [13](./13-prompt-assembly-and-cache.md) |
| context-compressor | [14](./14-context-compression.md) |
| parallel-tool-execution / tool-loop-guardrails / large-tool-result | [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails) |
| tool-registry / model-tools / toolsets | [06](./06-tools-registry-and-model-tools.md) · [07](./07-toolsets-and-platform-bundles.md) |
| memory-system / session-search-and-sessiondb | [08](./08-session-and-memory.md) |
| skills-system / skills-and-memory-interaction | [09 §10](./09-skills-curator-and-learning-loop.md#10-skills-与-memory-协作) |
| messaging-gateway / gateway-session / MESSAGE-QUEUE | [10 §6–§9](./10-gateway-platforms-and-sessions.md) |
| cli-architecture / skin / i18n | [03](./03-cli-gateway-and-entry.md) · [22 §6](./22-integrations-handbook.md#6-voice-与-tui-皮肤) |
| multi-agent / MoA / kanban | [18](./18-multi-agent-panorama.md) · [11](./11-delegation-cron-and-kanban.md) · [22 §10](./22-integrations-handbook.md#10-kanban-板内部) |
| goal-and-ralph-loop | [19](./19-goals-and-ralph-loop.md) |
| cron-scheduling | [11](./11-delegation-cron-and-kanban.md) |
| provider-transport / provider-plugin / auxiliary-client | [15](./15-provider-and-transport.md) |
| configuration-and-profiles / credential-pool | [02](./02-config-iteration-and-model-routing.md) · [21](./21-profiles-and-credential-pool.md) |
| mcp-and-plugins / hook-system | [16](./16-plugins-mcp-and-hooks.md) |
| terminal-backends | [17](./17-terminal-backends.md) |
| browser / web / PTC / LSP / voice / trajectory / proxy / @refs | [22 集成手册](./22-integrations-handbook.md) |
| security-defense-system | [20](./20-security-defense-layers.md) |
| interrupt-and-fault-tolerance | [05 §3.1](./05-aiagent-and-conversation-loop.md#31-interrupt-传播) |
| checkpoints / worktree-isolation | [22 §8](./22-integrations-handbook.md#8-worktree-与-checkpoint) |
| smart-model-routing / iteration-budget | [02](./02-config-iteration-and-model-routing.md) |
| fuzzy-matching-engine | [24 redirect](./24-tool-parallel-guardrails-and-overflow.md) → [22 §3](./22-integrations-handbook.md#3-并行-tool-与-guardrails) |

Wiki **changelog** 页不逐条镜像 — 跟进 upstream 时用 Wiki changelog + `git log`。

---

## 3. 编号规划（无 04 / 无 12 正文重复）

| 段 | 编号 | 说明 |
|----|------|------|
| 定位+骨架 | 00–03 | 无 04（预留） |
| 内核 | 05 | 无 04 |
| Meta | **12** | 本页（缺口/外链） |
| 扩展 | 06–11 | 工具与会话 |
| 加深 | 13–17 | Prompt/Provider/插件 |
| 运维 | 18–21 | Multi-agent/安全/Profile |
| 子系统 | [22 集成手册](./22-integrations-handbook.md) + 23–36 redirect | 不再维护 15 篇独立薄文 |
| Atlas | 26 | 总图 |
| 导航 | 99, learning-paths | |

---

## 4. 用法

1. **本笔记建立顺序** → [learning-paths](./learning-paths.md)  
2. **单点深读** → 上表 Wiki 链 + 官方 developer-guide  
3. **版本漂移** → pin `889903f` 验证后再信 Wiki HEAD 叙述  

→ [README 概念目录](./README.md#概念目录全集) · [99 索引](./99-glossary-and-reading-map.md)
