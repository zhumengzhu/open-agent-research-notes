# opencode-visual-cache v1.2.11 字段计算详解

> 基于 [`src/index.tsx`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx) 源码分析。版本 `1.2.11`（[`b65529b`](https://github.com/Hotakus/opencode-visual-cache/commit/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156)）。

---

## 数据采集链路

插件通过 `createEffect` 响应式追踪 `api.state.session.messages(sid)`，每次 session 状态变化时全量扫描所有 assistant 消息并累加各字段；同时监听 `message.updated` 和 `message.part.updated` 事件，通过递增计数器 `setRefreshTick` 强制 createEffect 重算。分布计算还用了 100ms debounce 避免流式更新时过于频繁的重算。

```typescript
createEffect(() => {
  const msgs = props.api.state.session.messages(sid) as Message[]
  for (const msg of msgs) {
    if (msg.role !== "assistant") continue
    const t = (msg as AssistantMessage).tokens; if (!t) continue
    // 累加各字段...
  }
})
```

（[`L407-L497`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L407-L497)）

**会话范围**：`sid` 来自 `sidebar_content` slot 的 `{ session_id }` prop，始终是当前主 session。插件只扫描当前 session 的 assistant 消息，**不包含子 agent session**（如 `oh-my-openagent` 通过 Task 工具 spawn 的子 agent）。这意味着在使用了多 agent 编排的场景下，费用和 token 统计仅反映主 session 的消耗，子 agent 的 token 和缓存命中不在统计范围内。

---

## 速查：精度分级

| 等级 | 含义 | 字段 | 说明 |
|---|---|---|---|
| 🟢 | API 精确值 | Read, Miss, Out, Cost（Detail）, Total Hit, Dist Total | `msg.tokens.*` 和 `msg.cost` 直接来自 OpenCode，无计算误差 |
| 🟡 | 依赖配置 | Rate, Total Saved | 依赖 `opencode.json` 中的模型定价配置，配置错误则数值错误 |
| 🟠 | 含义需注意 | Hit（取最后一条消息的命中率）, Trend（取最后两条消息的差值） | 数值本身精确，但不代表 session 全貌 |
| 🔴 | 估算值 | System, User, Agent Instr, Tool Call, Tool Result | 字符级 BPE 近似，非 API 精确值；受 `api.state.part()` 同步状态影响 |

---

## 字段逐项分析

### Hit — 当前消息命中率

```
Hit [████████████] 99.8% ↑0.2%
```

**公式**（[`L419-L433`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L419-L433)）：

```typescript
const mit = num(t.input) + num(t.cache?.read)    // 这条消息的总输入（含缓存）
const mrt = num(t.cache?.read)                     // 缓存命中部分
lastMsgHitRate = (mrt / mit) × 100                // 这条消息的命中率
hitRate = lastMsgHitRate                           // 取最后一条有 cache 数据的消息
```

🟠 这里显示的是**最后一条 assistant 消息的命中率**，不是 session 整体累计。前序消息的缓存表现不会被计入。session 整体累计见 Total Hit。

### Trend — 命中率变化方向

```
↑0.2%
```

**公式**（[`L416-L437`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L416-L437)）：

```typescript
prevMsgHitRate = lastMsgHitRate
lastMsgHitRate = ...
trend = lastMsgHitRate - prevMsgHitRate   // 最新两条消息的命中率差值
```

🟠 比较范围限于最新两条消息。实践中的一个用途：Anthropic 模型的 cache 在 5 分钟 TTL 过期后命中率骤降，此时 ↓ 箭头可以直观提示缓存已失效。

### Total Hit — Session 累计命中率

```
Total Hit: 96.4%
```

**公式**（[`L434`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L434)）：

```typescript
sessionHitRate = (sumRead / (sumRead + sumFreshInput)) × 100
```

`sumRead` = 所有消息的 `tokens.cache.read` 累计。`sumFreshInput` = 所有消息的 `tokens.input` 累计。

🟢 反映整个 session 的缓存效率，分子分母均来自 API 精确值。

### Read / Miss / Out — Token 明细

```
Read: 18.3M tok       Miss: 922.2K tok       Out: 20.8K tok
```

**公式**（[`L421`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L421)）：

```typescript
read += num(t.cache?.read)   // Read — 缓存命中的 token
input += num(t.input)         // Miss — 本次新计算的输入 token
output += num(t.output)       // Out  — 模型输出 token
```

三项均直接读取 `msg.tokens`，🟢 API 精确值。

**说明**：

- `Miss` 标签对应代码中的 `input` 变量，表示未缓存命中的输入 token 量。`tokens.input` 在不同 provider 上的具体含义有细微差异（Anthropic 中为"未命中缓存的新 token"，DeepSeek/OpenAI 有不同归一化方式）
- `Out` 不包含 reasoning tokens（`tokens.reasoning` 是独立字段）。DeepSeek 开启 `thinking` 模式时，`tokens.output` 只计可见输出，思维链 token 不在此列

### Total Saved — 缓存节省金额

```
Total Saved: ~¥52.82
```

**公式**（[`L425-L431`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L425-L431)）：

```typescript
for (const provider of api.state.provider) {
  if (provider.id !== pid) continue
  const model = provider.models[mid]
  inputRate = num(model.cost.input)
  cacheReadRate = num(model.cost.cache?.read)
  if (inputRate > cacheReadRate) {
    saved = (read × (inputRate - cacheReadRate)) / 1_000_000
  }
  break
}
```

🟡 计算逻辑：将已缓存的 token 数 × 缓存读与全价输入的差价，得出按输入价格重新计算本应支付的费用。注意点：

- 未计入 cache write 成本。对于有显式 cache write 的 provider（Anthropic），write 通常比 input 贵约 25%，不应视为零成本
- 遍历 `api.state.provider` 时取第一个匹配项，存在多 provider 时可能取到非预期定价
- 依赖 `opencode.json` 中的模型定价配置，费用数字含插件汇率换算，见 Cost 章节

### Cost — 累计费用

```
Cost: ¥7.97
```

**公式**（[`L422`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L422)）：`cost += num(msg.cost)`。

🟢 `msg.cost` 是 OpenCode 根据 `opencode.json` 中的模型定价计算出的 USD 费用，插件再乘以用户设定的汇率显示。

**关于汇率**：插件默认假设 provider 定价为 USD。如果 `opencode.json` 中已配置为人民币定价（如 `cost.input: 3.13`），且汇率不是 1，会产生双重换算。此时用 `/cache-rate 1` 可关闭汇率转换，Rate 和 Total Saved 一并适用。

### Rate — 模型单价

```
Rate: ¥3.13/M in / ¥0.26/M cache
```

**公式**（[`L429`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L429)）：

```typescript
inputRate = num(model.cost.input) × exchangeRate
cacheReadRate = num(model.cost.cache?.read) × exchangeRate
```

🟡 从 `api.state.provider` 读取当前 session 的模型定价。若 `opencode.json` 未配置 `cost` 字段，此行不显示。

### Estimated Token Dist. — Token 分布估算

```
System:       9,287 tok       Agent Instr: 26,100 tok
User:         2,915 tok       Tool Call:   11,700 tok
                              Tool Result: 35,000 tok
                              Total:      316,000 tok
```

**公式**（[`L268-L296`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L268-L296)、[`L440-L488`](https://github.com/Hotakus/opencode-visual-cache/blob/b65529b6587c6abb1d9cc9f73a4df3dbf61bf156/src/index.tsx#L440-L488)）：每项通过 `estimateTokens()` 做字符级 BPE 估算——ASCII 文本约 3.5-4 字符/token，JSON/代码约 3.5，CJK 约 1 字符/token。

| 字段 | 数据来源 |
|---|---|
| System | `agentCfg.prompt` + 各消息 `user.system` |
| User | `api.state.part(msg.id)` 中 `text` 和 `file` 类型 |
| Agent Instr | `reasoning` 文本 + `subtask` prompt |
| Tool Call | tool `state.input` JSON |
| Tool Result | tool `state.output` 或 `state.error` |
| **Total** | 最后一条消息的 `tokens.input + tokens.cache.read`（🟢 API 精确值） |

🔴 注意点：

- Total ≠ 分项之和是预期行为。差值来自 OpenCode 运行时注入的系统内容（环境信息、工具 schema 定义、skill 目录等），这些不在消息可读字段中
- 字符级 BPE 近似，与真实 tokenizer 有偏差（JSON/代码可达 30-50%）
- 实测在同一 session 内分项数值出现过逆向缩小（Agent Instr 26.5K→26.1K, Tool Call 12.6K→11.7K, Tool Result 51.8K→35.0K），而 Total（315.2K→316.0K）正常增长。根因是 `api.state.part(msg.id)` 在状态同步间隙可能返回空数组

---

## 实现特点

### 全量扫描

每次 session 状态变化时遍历全部消息；增量更新（仅累加新消息）可降低重复计算。当前用 100ms debounce 控制重算频率。

### createEffect 与事件双轨

`createEffect` 已响应式追踪 `api.state.session.messages()` 状态变化，同时监听 `message.updated` 和 `message.part.updated` 事件触发额外刷新。这种双轨设计可能用于覆盖 createEffect 不触发的边界时序。

### untrack 与分布计算

分布计算中 `api.state.part()` 调用被 `untrack()` 包裹以阻断响应式依赖（防止死循环）。结果是分布数据不随 part 更新自动刷新，依赖事件驱动的 100ms debounce 手动触发。

### 子 agent session 支持

插件只统计当前主 session，不包含 oh-my-openagent 等工具 spawn 的子 agent session。底层数据是支持扩展的：`api.state.session.get()` 可查询 `parentID`，`api.event.on("session.created")` 可捕获新 session 创建，`api.state.session.messages(childSid)` 可读子 session 消息。

但当前基于 `createEffect` 的架构不擅长动态增删追踪目标——effect 在编译时确定了对 `api.state.session.messages(sid)` 的响应式依赖，运行时加子 session 需要动态增减 effect，SolidJS 没有简洁的范式支持。可行的改造方向是放弃 `createEffect` 全量扫描，改为事件驱动累加——但这意味着数据获取方式发生根本性变化，不是小改动。
