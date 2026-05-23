# 04 · 范本精读：`anthropic-effort` hook

> **核心问题：** "基于 provider/model 修改 LLM 入参"的最小完整例子怎么写？
>
> 这是你的 `opencode-thinking-toggle` 最该照抄的范本。113 行代码包含了你需要的所有模式。

---

## 1. 这个 hook 干什么

Anthropic API 接受 `output_config.effort: "low" | "medium" | "high"`，部分模型还接受 `"max"`。但：

- **OpenCode 不知道哪些模型支持 `max`**（只有 Opus 支持）
- **github-copilot 走的是代理 Anthropic，不支持 `max`**
- **Anthropic OAuth（Claude Pro/Max）走第三方客户端协议，也不支持 `max`**
- **Haiku 模型完全不支持 effort 参数**

`anthropic-effort` 在 `chat.params` 阶段：
1. 识别请求是 Claude provider
2. 看用户/agent 是否要 `variant: "max"`
3. 根据模型族和 provider 限制，clamp 到合适的 effort 值
4. 写入 `output.options.effort`，同时把 `message.variant` 也改成 clamped 值

→ **完全一样的模式**：你的 DeepSeek thinking 控制就是"识别 DeepSeek → 根据模型族决定开/关 → 写入 `output.options.thinking` / `output.options.enable_thinking`"。

## 2. 完整代码走读

文件：[`src/hooks/anthropic-effort/hook.ts`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/src/hooks/anthropic-effort/hook.ts)（113 行）。

### 2.1 模型/provider 识别

```1:21:src/hooks/anthropic-effort/hook.ts
import { isProviderUsingOAuth, log, normalizeModelID } from "../../shared"

const OPUS_PATTERN = /claude-.*opus/i
const EFFORT_UNSUPPORTED_PATTERN = /claude-.*haiku/i
const INTERNAL_SKIP_AGENTS = new Set(["title", "summary", "compaction"])

function isClaudeProvider(providerID: string, modelID: string): boolean {
  if (["anthropic", "google-vertex-anthropic", "opencode"].includes(providerID)) return true
  if (providerID === "github-copilot" && modelID.toLowerCase().includes("claude")) return true
  return false
}

function isOpusModel(modelID: string): boolean {
  const normalized = normalizeModelID(modelID)
  return OPUS_PATTERN.test(normalized)
}

function isEffortUnsupportedModel(modelID: string): boolean {
  const normalized = normalizeModelID(modelID)
  return EFFORT_UNSUPPORTED_PATTERN.test(normalized)
}
```

**学习要点：**
- 用 **regex pattern** 识别模型族（haiku/opus 是关键词）—— 你可以套用到 `deepseek-r1` / `deepseek-reasoner` / `deepseek-chat` / `deepseek-v3`
- 用 **provider 白名单** 识别 provider 族 —— 同一族 Claude 在多个 provider 下都要识别（anthropic / vertex / opencode / copilot）
- `normalizeModelID()` 把 `deepseek.r1.0613` 这种点号归一化成 `deepseek-r1-0613` —— 你的 DeepSeek 检测应该也用它

### 2.2 内部 agent 跳过

```22:26:src/hooks/anthropic-effort/hook.ts
function shouldSkipForInternalAgent(agentName: string | undefined): boolean {
  if (!agentName) return false
  return INTERNAL_SKIP_AGENTS.has(agentName.trim().toLowerCase())
}
```

**学习要点：** OpenCode 用 `title` / `summary` / `compaction` 这种内部 agent 做后台任务（给 session 起标题、压缩上下文）。这些调用不应该被业务 hook 影响 —— **你的插件也应该跳过它们**。

### 2.3 provider 限制识别

```28:38:src/hooks/anthropic-effort/hook.ts
/**
 * Providers that expose constrained APIs rejecting `output_config.effort: "max"`
 * (supported values: low | medium | high). Includes:
 * - Anthropic OAuth (Claude Pro/Max via third-party clients)
 * - GitHub Copilot (proxied Anthropic, doesn't support "max")
 */
function isConstrainedProvider(providerID: string): boolean {
  if (providerID === "github-copilot") return true
  if (providerID === "anthropic") return isProviderUsingOAuth(providerID)
  return false
}
```

**学习要点：** 同一个 provider，**通过 OAuth 接入 vs 通过 API key 接入，能力可能不一样**。`isProviderUsingOAuth()` 从 OpenCode 当前会话状态读取这个信号。

### 2.4 阶梯 clamping

```55:68:src/hooks/anthropic-effort/hook.ts
/**
 * Valid thinking budget levels per model tier.
 * Opus supports "max"; all other Claude models cap at "high".
 */
const MAX_VARIANT_BY_TIER: Record<string, string> = {
  opus: "max",
  default: "high",
}

function clampVariant(variant: string, isOpus: boolean, isConstrained: boolean): string {
  if (variant !== "max") return variant
  if (isConstrained) return MAX_VARIANT_BY_TIER.default
  return isOpus ? MAX_VARIANT_BY_TIER.opus : MAX_VARIANT_BY_TIER.default
}
```

**学习要点：** 把"模型族 → 最高 variant"做成查表，clamping 逻辑清晰可维护。

### 2.5 hook 主体（factory + 闭包）

```70:112:src/hooks/anthropic-effort/hook.ts
export function createAnthropicEffortHook() {
  return {
    "chat.params": async (input, output): Promise<void> => {
      const { agent, model, message } = input

      // 早返条件层层把关
      if (!model?.modelID || !model?.providerID) return
      if (isEffortUnsupportedModel(model.modelID)) return
      if (message.variant !== "max") return
      if (!isClaudeProvider(model.providerID, model.modelID)) return
      if (shouldSkipForInternalAgent(agent?.name)) return
      if (output.options.effort !== undefined) return  // 不覆盖别人已设的

      const opus = isOpusModel(model.modelID)
      const constrained = isConstrainedProvider(model.providerID)
      const clamped = clampVariant(message.variant, opus, constrained)
      output.options.effort = clamped

      const shouldOverrideMessageVariant = !opus || constrained
      if (shouldOverrideMessageVariant) {
        (message as { variant?: string }).variant = clamped
        log("anthropic-effort: clamped variant max→high", { ... })
      } else {
        log("anthropic-effort: injected effort=max", { ... })
      }
    },
  }
}
```

**学习要点（按价值排序）：**

1. **早返链式** —— 7 个 `if (...) return` 一字排开。**任何一个不满足直接跳过**，避免嵌套。
2. **不覆盖原则** —— `if (output.options.effort !== undefined) return` 表示"如果上游已经设置了，我不抢"。当多个 hook/插件都可能改同一个字段时，这个习惯很重要。
3. **同时改 `output.options` 和 `message.variant`** —— `message.variant` 是给 OpenCode 后续步骤看的，`output.options.effort` 是真正传给 LLM 的。两者要保持一致。
4. **`log()` 包含完整上下文** —— `{ sessionID, provider, model, reason }`，调试时直接 grep `/tmp/oh-my-opencode.log` 就能定位。

## 3. 注册到 OmO 5 层

```258:260:src/plugin/hooks/create-session-hooks.ts
const anthropicEffort = isHookEnabled("anthropic-effort")
  ? safeHook("anthropic-effort", () => createAnthropicEffortHook())
  : null
```

→ 然后这个 `anthropicEffort` 引用被传到 [`src/plugin/chat-params.ts`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/src/plugin/chat-params.ts)，dispatcher 在最后一步调它：

```193:194:src/plugin/chat-params.ts
await args.anthropicEffort?.["chat.params"]?.(normalizedInput, output)
```

**但你写独立插件时不需要这套**。你的 12 切面里直接挂 `chat.params` 就行，OpenCode 会自动调你。

## 4. 你照抄的版本（DeepSeek 极简版）

```typescript
import type { Plugin } from "@opencode-ai/plugin"

const DEEPSEEK_REASONING_PATTERN = /deepseek-(r1|reasoner)/i
const DEEPSEEK_CHAT_PATTERN = /deepseek-(chat|v3)/i
const INTERNAL_SKIP_AGENTS = new Set(["title", "summary", "compaction"])

function isDeepSeekProvider(providerID: string, modelID: string): boolean {
  if (providerID === "deepseek") return true
  // 用户用 OpenAI 兼容协议接入 DeepSeek 时，providerID 可能是 "openai" / "openai-compat"
  // 但 modelID 一定带 deepseek 关键词
  if (modelID.toLowerCase().includes("deepseek")) return true
  return false
}

const serverPlugin: Plugin = async () => ({
  "chat.params": async (input, output) => {
    const { agent, model } = input
    if (!model?.modelID) return
    if (!isDeepSeekProvider(model.providerID, model.modelID)) return
    if (agent?.name && INTERNAL_SKIP_AGENTS.has(agent.name.toLowerCase())) return

    output.options = output.options ?? {}

    if (DEEPSEEK_REASONING_PATTERN.test(model.modelID)) {
      // R1/reasoner: 强制开 thinking
      output.options.thinking = { type: "enabled" }
      output.options.reasoningEffort = "high"
      return
    }

    if (DEEPSEEK_CHAT_PATTERN.test(model.modelID)) {
      // chat/V3: 强制关 thinking
      delete output.options.thinking
      delete output.options.reasoningEffort
      output.options.enable_thinking = false  // DeepSeek 自定义字段
      return
    }
  },
})

export default { id: "opencode-thinking-toggle", server: serverPlugin }
```

→ **30 行就完事**。配置驱动、单测、多 provider 支持都是锦上添花。

## 5. anthropic-effort 没用到但你可能需要的

- **状态保持**：think-mode 用 `Map<sessionID, ThinkModeState>` 跨多轮记忆"用户曾经触发过 think 关键词"。如果你的 DeepSeek 控制要根据用户输入动态切，可能也需要 —— 但**强制开/关方案不需要**，纯 stateless。
- **session.deleted 清理**：用 Map 的话就需要 `event` hook 监听 `session.deleted` 清理，参考 `think-mode/hook.ts:68-75`。

---

## 读完后应该能回答

- [ ] anthropic-effort 用了哪几种 pattern 识别 Claude provider？
- [ ] 为什么要跳过 `title` / `summary` / `compaction` agent？
- [ ] OAuth 接入和 API key 接入的 provider 怎么区分？
- [ ] 如果上游已经设置了 `output.options.effort`，这个 hook 怎么处理？
- [ ] 为什么要同时改 `output.options.effort` 和 `message.variant`？

---

→ **下一篇：** [05 · 关键词→变体切换：think-mode](./05-think-mode-mechanism.md)
