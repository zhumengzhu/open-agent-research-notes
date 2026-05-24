# 24 · 成本、Analytics 与限额

> **锚点：** `cost-tracker.ts` · `bootstrap/state.js` · `services/api/logging.ts` · `services/analytics/` · `utils/modelCost.ts` · `policyLimits/`

---

## 1. Usage 数据流

```text
API response.usage (BetaUsage)
  → accumulateUsage / updateUsage (claude.ts)
  → bootstrap/state counters
  → calculateUSDCost (modelCost.ts)
  → cost-tracker.ts / SDK result.total_cost_usd
  → logEvent('tengu_*') (analytics)
```

---

## 2. Usage 字段与计费

### 2.1 单次 API call

| 字段 | 计费角色 |
|------|----------|
| `input_tokens` | 标准输入 |
| `output_tokens` | 标准输出 |
| `cache_read_input_tokens` | 折扣读（相对 input） |
| `cache_creation_input_tokens` | 写入 cache（常更贵） |
| `cache_deleted_input_tokens` | cached MC 删 KV（logging 专轨 [10]） |

### 2.2 价目表（`modelCost.ts`）

按 **canonical model** 映射 tier（美元 / Mtok 概念）：

| Tier | 典型模型 | input / output 量级 |
|------|----------|---------------------|
| `COST_TIER_3_15` | Sonnet 4.x | 3 / 15 |
| `COST_TIER_5_25` | Opus 4.5 | 5 / 25 |
| `COST_TIER_15_75` | Opus 4/4.1 | 15 / 75 |
| `COST_TIER_30_150` | Opus 4.6 fast mode | 30 / 150 |
| `COST_HAIKU_*` | Haiku | 更低 |

`firstPartyNameToCanonical` + `getCanonicalName` — Bedrock/Vertex id 归一化后查表。

**Fast mode：** `isFastModeEnabled()` 可能切更高 tier [27]。

### 2.3 Session 累计

`bootstrap/state.js`：

- `getTotalCostUSD` / `getTotalInputTokens` / …
- `getModelUsage` — per-model 分解
- `getTotalWebSearchRequests` — 搜索附加费

QueryEngine `totalUsage` — turn 级视图 [07]。

### 2.4 Fork 计费

`runForkedAgent` → `tengu_fork_agent_query` + usage metadata；**合并策略** 见调用方（通常计入 session 总成本）。

---

## 3. 成本展示

| 入口 | 行为 |
|------|------|
| `/cost` | 格式化 counter [12] |
| SDK `result` | `total_cost_usd`、token 明细 [19] |
| `costHook.ts` | REPL UI 刷新 |
| `cost-tracker.ts` | 主格式化、`hasUnknownModelCost` |

**Cache 经济：** 高 `cache_read` + 低 `cache_creation` = 长 session 理想；compact/cache break 后 creation spike [10][30]。

---

## 4. 限额与预算

| 机制 | 层 | 行为 |
|------|-----|------|
| `maxBudgetUsd` | QueryEngine | 累计 USD 超限 → 终止 turn |
| `maxTurns` | query loop | iteration 硬上限 |
| `taskBudget` | API | server 侧 output budget [07] |
| TOKEN_BUDGET | query.ts | 客户端 nudge continue [07][28] |
| `policyLimits/` | 组织 | 企业策略 |
| `claudeAiLimits.ts` | 订阅 | 档位配额 |
| 529 / rate limit | `withRetry.ts` | 退避 |

`maxBudgetUsd` 在 **submitMessage / loop 入口** 查 `getTotalCostUSD`，非 per-call。

---

## 5. Analytics 体系

### 5.1 事件管道

`services/analytics/index.ts` `logEvent(name, metadata)`：

- 同步路径：内存 buffer
- `firstPartyEventLogger.ts` — 1P 上报
- `datadog.ts` — shutdown flush

Metadata：**`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`** — 防把路径当 analytics 字段。

### 5.2 事件目录（节选）

| 前缀/名 | 场景 |
|---------|------|
| `tengu_fork_agent_query` | subagent/extract fork [20][29] |
| `tengu_coordinator_mode_switched` | resume mode flip [21] |
| `tengu_compact_*` | compact/cache [10] |
| `tengu_bridge_*` | bridge 连接 [18] |
| tool execute / denial | [09][11] |
| cache break | `promptCacheBreakDetection` [30] |

完整 schema skim：`services/analytics/metadata.ts`。

### 5.3 GrowthBook / Statsig

`growthbook.ts`：

| 类型 | 例 |
|------|-----|
| Gate | `tengu_amber_flint`（teams）、`tengu_passport_quail`（extract） |
| Experiment | `tengu_compact_cache_prefix`、`tengu_slate_heron` |
| Dynamic config | bridge poll 间隔等 |

**`getFeatureValue_CACHED_MAY_BE_STALE`：** 启动后可能滞后；entitlement 用 `checkGate_CACHED_OR_BLOCKING` [18]。

---

## 6. Cache break 与成本尖峰

`promptCacheBreakDetection.ts`：

- **Tracked** 变更：model、effort、tools、system 等
- **Untracked**：speculation、session_memory、prompt_suggestion [30]

Break → 下一 call `cache_creation_input_tokens` 上升 → `/cost` 可见 spike。

---

## 7. 未知模型与 advisor

- `hasUnknownModelCost` / `setHasUnknownModelCost` — 价目表无映射时 UI 警告
- `getAdvisorUsage` — advisor 子系统（若 `--advisor`）

---

## 8. 测试与 restore

- `resetCostState` / `resetStateForTests`
- `setCostStateForRestore` — resume 恢复 counter [08]

---

## 9. 调试成本问题

| 问题 | 查 |
|------|-----|
| 单 turn 暴涨 | cache_creation vs break 事件 |
| fork 重复计费 | `tengu_fork_agent_query` 次数 |
| SDK 与 UI 不一致 | print 是否 reset state |
| 模型显示 unknown | `MODEL_COSTS` 是否含 canonical |

---

## 10. 源码带读

1. `utils/modelCost.ts` — tier 与 `calculateUSDCost`
2. `services/api/logging.ts` — usage 规范化
3. `cost-tracker.ts` — 展示
4. `services/analytics/metadata.ts` — 事件字段
5. `services/api/promptCacheBreakDetection.ts`

---

## 11. 自测

- [ ] cache_read vs cache_creation 计价差异？
- [ ] `maxBudgetUsd` 检查层？
- [ ] fork 事件名与 merge 策略？
- [ ] GrowthBook stale cache 与 blocking gate 区别？
- [ ] untracked cache source 举例？

**关联：** [07 API](./07-api-and-model-stream.md) · [10 Cache](./10-compaction-and-context.md) · [27 Model](./27-multi-model-thinking-and-fallback.md) · [30 实验](./30-advanced-features-and-experiments.md)
