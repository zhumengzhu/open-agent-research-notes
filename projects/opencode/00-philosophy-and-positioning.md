# 00 · 产品理念与定位

> **核心问题：** OpenCode 想解决什么？它在 agent 生态里扮演什么角色？

---

## 1. 一句话定位

OpenCode 是 **开源、终端原生、Provider 无关** 的 AI coding agent **runtime 内核**：负责会话、工具、模型调用与扩展点；TUI / Web / Desktop 与第三方插件都挂在这套内核上。

官方：[README](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/README.md)、[文档首页](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/web/src/content/docs/index.mdx)

---

## 2. 设计取向

| 取向 | 在代码里的体现 |
|------|----------------|
| **Client / Server 分离** | [`server/server.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/server/server.ts) + HTTP API |
| **Provider 无关** | [`provider/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/provider) + [`packages/llm/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/llm) |
| **扩展优先** | config / plugin / MCP / skill / command |
| **单项目实例隔离** | [`project/instance-store.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/instance-store.ts) |

---

## 3. 内置 Agent（内核默认）

文档：[agents.mdx](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/web/src/content/docs/agents.mdx)

| Agent | mode | 用途 |
|-------|------|------|
| build | primary | 全权限编码 |
| plan | primary | 只读分析 |
| general | subagent | Task 委派 |
| explore / scout | subagent | 代码库探索 |

定义：[`agent/agent.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/agent/agent.ts)。config 与 `hook.config` 可追加/覆盖 agent，**不替换** Agent 机制本身。

---

## 4. 四条扩展线

```
OpenCode 内核
  ├── Plugin (Hooks)
  ├── MCP (远程工具)
  ├── Skill (SKILL.md)
  ├── Command (slash)
  └── config.agent (人格 / 模型)
```

---

## 5. 读码时带的两个问题

1. **内核 invariant：** Session DB、`runLoop`、Permission 求值 —— 插件改不了。  
2. **Extension surface：** `chat.params`、工具列表、agent/MCP 定义 —— 插件/config 可改。

---

## 读完后应能回答

- [ ] OpenCode 与 UI 壳、与插件各是什么关系？
- [ ] 写 `chat.params` 插件属于哪条扩展线？
- [ ] 主链路从哪篇开始跟源码？

→ **下一篇：** [01 · Monorepo 与包分层](./01-monorepo-and-packages.md)  
→ **总图：** [18 · 主链路总图](./18-main-chain-atlas.md)
