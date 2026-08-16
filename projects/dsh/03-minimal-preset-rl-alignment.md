# 03 DeepSeek V4 Pro 的 harness 过拟合：minimal preset 与 reasoning_effort 注入

> 证据基线：`deepseek-ai/deepseek-harness@47f943859b` 本地源码精读 + 社区消融实验
> （xiaobright/modeltest、dsh-anchored-standard）与 DeepSeek-V4-Flash 开源权重的 encoding 源码。

## 1. 一句话定位

DeepSeek V4 Pro（正式版 0813）在官方 DeepSeek Harness 的 **minimal preset** 下能跑出灰度测试级
分数（99/96），但在通用 harness（OpenCode 等）下掉到 91–96。原因不是「加一句魔法提示词」，
而是两个机制的组合：**（1）`reasoning_effort=max` 本质是往首条消息注入一段提示词**；
**（2）minimal preset 精确复刻了模型 RL 后训练时的 prompt + 两工具 schema**。

## 2. 机制一：`reasoning_effort=max` 是「开头注入提示词」

DeepSeek-V4 开源权重里的 `encoding/encoding_dsv4.py` 定义了常量
[`REASONING_EFFORT_MAX`](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash/blob/main/encoding/encoding_dsv4.py)：

```python
REASONING_EFFORT_MAX = (
    "Reasoning Effort: Absolute maximum with no shortcuts permitted.\n"
    "You MUST be very thorough in your thinking and comprehensively decompose the problem "
    "to resolve the root cause, rigorously stress-testing your logic against all potential "
    "paths, edge cases, and adversarial scenarios.\n"
    "Explicitly write out your entire deliberation process, documenting every intermediate "
    "step, considered alternative, and rejected hypothesis to ensure absolutely no "
    "assumption is left unchecked.\n\n"
)
```

注入条件写死在 `render_message()`：

```python
# Reasoning effort prefix (only at index 0 in thinking mode with max effort)
if index == 0 and thinking_mode == "thinking" and reasoning_effort == 'max':
    prompt += REASONING_EFFORT_MAX
```

三个结论：

1. **只有 `max` 档注入这段前缀**——`high` 档走普通思考模式，不注入任何额外文本。
2. **只在 `index == 0`（首条消息）注入**——所以「开头加提示词」是字面意义成立的，
   这也是后续「首轮请求敏感」现象的根因之一。
3. 网上流传的「能让 deepseek v4 更厉害的提示词」（`Reasoning Effort: Absolute maximum...`）
   **就是这段官方常量**，不是社区发明的玄学。

## 3. 机制二：minimal preset 是「RL 训练对齐」，不是 shorter standard

dsh 的 minimal preset（`apps/cli/config/agent-presets/minimal/agent.cordis.yml`）把 system prompt
固定为唯一一句，并只暴露两个工具：

```yaml
- id: persona
  name: '@deepseek-ai/dsh-persona'
  config:
    text: You are a helpful software engineer assistant.
    complete: true            # 唯一完整提示，屏蔽身份/工具指导/后续注入
    includeRuntimeContext: false
```

只挂持久化 `bash` + `str_replace_editor`，无 compaction、无 sandbox、本地文件系统。

关键证据：官方快照测试
[`apps/web/tests/minimal-preset.snapshot.ts#L49`](https://github.com/deepseek-ai/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/apps/web/tests/minimal-preset.snapshot.ts#L49)
的测试名直接是 **`sends the exact RL prompt and schemas`**——「RL」是仓库作者明文写的，
不是从跑分反推。快照确认请求里只有 `prompt: "You are a helpful software engineer assistant."`
和 `tools: ["bash", "str_replace_editor"]`。

即：minimal preset 精确复刻了模型 RL 后训练时的 prompt + 工具 schema 分布，而不是 standard
的简单精简版（standard 有 26 个插件、25 个工具、自动注入 AGENTS.md 等）。

## 4. 消融实验：关键变量是「首轮看到什么工具」

社区 xiaobright/modeltest 做了固定微任务、单变量的首轮微探针
（[DEEPSEEK_V4_TRIGGER_MECHANISM_EXPERIMENTS_20260814.md](https://github.com/xiaobright/modeltest/blob/04255b55f16c4439e538239fb9783070c4165081/docs/v4.1/DEEPSEEK_V4_TRIGGER_MECHANISM_EXPERIMENTS_20260814.md)），
Pro 首轮工具目录替换结果：

| 首轮 API 可见工具 | 轨迹开头 / 分类 |
|---|---|
| minimal `bash + editor` | `We need` / minimal-like（基线） |
| Standard **25 工具** | `The user wants… Let me…` / 非 minimal |
| Standard 单 `bash` | `We need` / minimal-like |
| Standard `bash + read` | `We need` / minimal-like |
| Standard `bash + glob` | `The user wants… Let me…` / 非 minimal |
| Standard `bash + edit` | `We need` / minimal-like |
| PTC 单 `run_code` | `Let me` / standard-like |

三个硬结论：

1. **不是「工具数 > 1」规则**（`bash+edit` 仍触发），也**不是「单一入口」规则**（PTC `run_code` 不触发）。
2. **影响来自「可调用的 schema surface」，不是「看到工具名文本」**——把完整工具目录当 user 文本
   塞进去、API 层仍只暴露 minimal 工具，轨迹照样是 `We/Need`。
3. **Flash 和 Pro 机制不同**：Flash 的 `We need` 轨迹主要跟随 system persona，完整 25 工具目录
   不是阻断项；且 Flash 换 harness 分数稳定（90–95），Pro 才有「过拟合」现象。

### 决定性实验：动态工具晋升（anchored-standard 的原理）

两阶段微探针：turn 1 只暴露两工具 → `We need`；turn 2 恢复完整 25 工具 → 开头变 `Need…`、
无持续 `Let me`、能合法调用完整目录；turn 3 仍完整目录、不回 `Let me`。

→ 社区实现为 [xiaobright/dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard/blob/db4527a2a70a9032d3a8525ce3c0ea6ef528d6fc/README.zh-CN.md)：
**首轮 minimal 锚定、首次 tool call 后恢复全部工具**，Project2 连续 **98/99**，保高分且保完整工具。

### 复现的脆弱性（issue #6 补充，2026-08-15）

原 README 未记录、后被定位的触发条件（按影响排序）：

1. **首轮 `max_tokens`（最强）**：`1024` → 81% 出 `We need`；`256000`（DSH 适配器默认）→ 0/5。
   非单调（4096 翻转、8192 不翻转）。
2. **技能目录注入**：`dsh-tool-skill` 在有本地技能时于首步 `agent/pre-step` 注入约 9KB 技能目录，
   带技能块 → 0/9，无技能块 → 81%。
3. **DSH 身份头组合**：双头（user-id + session-id）→ 4/7，无头/单头 → 18/18（Fisher p≈0.0016，概率偏移非开关）。

## 5. 边界：已证明 vs 未证明

**已证明**（社区自己列的严格边界）：

- 正式 V4 Pro 在本题上具备接近灰测/Fable 5/Opus 5/Sol 分数带的可访问能力；
- Pro 的首轮 model-visible prompt + tool schema 显著影响后续轨迹；
- 首轮锚定后恢复完整工具不立刻丢轨迹；Flash 与 Pro 条件敏感性不同。

**未证明**：

- 用措辞（`We need`/`Good`/低阶段回复）判定后端是 Claude/Fable/泄露 checkpoint；
- 存在确定的服务端「特殊路由标识」；
- **98/99 能在其他仓库、任务长度、provider 上稳定复现**（n=2 同题复现）；
- 已证明 DeepSeek 在 7–8 月间专门对 minimal preset 过拟合（时间线）。

## 6. 其他 harness 的等价实现

| harness | 实现 | 方式 | 备注 |
|---|---|---|---|
| dsh | [dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard) | 首轮两工具 → 晋升全工具（原生 preset） | 原版，Project2 98/99 |
| opencode | [opencode-anchored-standard](https://github.com/862930522/opencode-anchored-standard/blob/750c104685a0e0f638d0a86e8c3d125901504c91/README.md) | 独立移植；**无 per-request 工具过滤 API**，改为「保留全目录 + system prompt 注入 bootstrap 规则 + 拦截非 bootstrap 工具调用返回可读错误」 | 行为对守规模型等价；原 DSH 版是「直接隐藏工具」 |
| pi | [pi-deepseek-optimized](https://github.com/jrimmer/pi-deepseek-optimized/blob/15c8f75ca7eaf4d0c6fa961a80c8b11b825fc305/README.md) | **不同路线**：缓存前缀稳定 / storm-breaker / hashline 编辑 / plan mode / rewind | 没有 anchored-standard 等价物，走的是「修 harness 失败模式」路线 |
| claude code | 无 | 封闭 harness，无法按请求改工具目录 | 只能通过 `ANTHROPIC_BASE_URL` 指向 DeepSeek 的 Anthropic 兼容端点，工具目录固定 |

关键观察：**「首轮锚定」这个技巧能否移植，取决于 harness 是否暴露「按请求控制工具目录」的钩子**。
opencode 没有这个 API，只能退化为「拦截 + 可读报错」的近似；pi 的生态里根本没人做 anchored 等价物，
而是走了更通用的 harness 优化路线（缓存、编辑、错误修复）；claude code 则完全无法移植。

## 7. 关键结论速记

- `reasoning_effort=max` = 首条消息注入 `REASONING_EFFORT_MAX` 提示词（官方 encoding 源码，非玄学）
- minimal preset = 官方 RL 训练的「单句 persona + 两工具」精确复刻（测试名 `exact RL prompt and schemas`）
- Pro 的增益定位到「首轮请求的 prompt + 工具 schema」，不是「全程只用两工具」
- 复现脆弱：首轮 `max_tokens`、技能目录注入、HTTP 头都会翻转结果；服务端行为 8-15 已疑似漂移
- 「开头加提示词」是必要非充分条件；单复制一句提示词复现不了 98/99
- 移植性取决于 harness 能否按请求改工具目录：dsh 原生可，opencode 近似可，pi 走了别的路，claude code 不可
