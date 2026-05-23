# 16 · OpenCode 学习地图（为“理解 coding agent 构造”准备）

> 目标：从 OmO 反向进入 OpenCode，理解“底座如何构造 agent runtime”。
>
> 你现在的状态：OmO 主干已覆盖，下一步是补 OpenCode 内核认知。

---

## 1) 你要回答的 4 个 OpenCode 核心问题

1. 插件 hook 是怎么被注册、调度、串联执行的？
2. agent/session/model/tool 这四个 runtime 实体如何交互？
3. tool call 如何进入 LLM 请求并回填结果？
4. session 生命周期与事件模型如何驱动状态迁移？

---

## 2) 建议学习顺序（在 `~/Github/opencode`）

1. 插件协议与 hooks 类型定义（先找接口定义）
2. session 与 chat 请求主路径（从入口跟到 provider 调用）
3. tool use 执行链（tool_use -> execute -> tool_result）
4. event 系统（session.created/idle/deleted/error）
5. agent 列表/排序/展示逻辑（对照 OmO sort shim 背景）
6. provider 抽象层（OpenAI/Anthropic/Gemini 适配点）

---

## 3) 与 OmO 的映射关系（对照学习）

| OpenCode 内核能力 | OmO 对应扩展 |
|---|---|
| Hook 调度与生命周期 | 5-tier hooks + safeCreateHook |
| 基础工具协议 | 20-39 tools registry + gate |
| Agent 基础定义 | 11-agent orchestration + restrictions |
| Session/Event 基座 | continuation/boulder/runtime-fallback |
| Provider 调用 | think-mode/anthropic-effort/compat clamp |

---

## 4) 达标标准（OpenCode 侧）

- [ ] 能从 OpenCode 源码画出“一次用户请求到 LLM 返回”的主链路
- [ ] 能说明 plugin hook 在该链路中的注入点
- [ ] 能解释 OmO 为什么要 patch sort shim（并能指出这是权衡不是理想方案）
- [ ] 能给出“若重做一版 OmO，哪些能力应该下沉到 OpenCode 内核”

---

## 5) 边界对照（优先读）

→ **[OpenCode ↔ OmO 边界](../../comparisons/opencode-vs-omo-boundary.md)**（职责表、Hook 对照、主链路时序）

