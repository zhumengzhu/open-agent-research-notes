# opencode 学习笔记

> **源码：** [anomalyco/opencode](https://github.com/anomalyco/opencode)  
> **基准版本：** [`7fe7b9f2`](https://github.com/anomalyco/opencode/commit/7fe7b9f258e36ad9f9acded20c5a9df201da19d5)（upstream，2026-05-24）  
> **个人 fork：** [zhumengzhu/opencode](https://github.com/zhumengzhu/opencode)

OpenCode 是 coding agent 的 **runtime 内核**（会话、LLM、工具、权限、持久化）。

## 建议阅读顺序

1. **先读边界对照**（与 OmO 的关系）：[`comparisons/opencode-vs-omo-boundary.md`](../../comparisons/opencode-vs-omo-boundary.md)
2. 插件 SDK 契约：[`packages/plugin/src/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/plugin/src/index.ts)
3. 插件加载与 trigger：[`packages/opencode/src/plugin/index.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/plugin/index.ts)
4. 主循环：[`packages/opencode/src/session/prompt.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/prompt.ts)
5. LLM 调用与 chat.params：[`packages/opencode/src/session/llm.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/session/llm.ts)
6. 工具注册：[`packages/opencode/src/tool/registry.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/tool/registry.ts)

## 概念篇（待写）

按 OmO 笔记同样风格，编号展开 session / agent / provider / compaction 等主题。
