# ClawSweeper 深度调研

> **官方仓库：** [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)  
> **官网 / 文档：** [clawsweeper.bot](https://clawsweeper.bot) · [clawsweeper.openclaw.ai](https://clawsweeper.openclaw.ai)（实时仪表盘）  
> **状态仓库：** [openclaw/clawsweeper-state](https://github.com/openclaw/clawsweeper-state)  
> **基准版本：** 笔记撰写时以 `main` 为准；正文举证请 pin 到具体 commit SHA

## 项目定位

ClawSweeper 是 OpenClaw 生态中的 **AI 驱动维护机器人**，用于自动审查 GitHub Issues 和 Pull Requests。与传统的时间规则 stale bot（如 `actions/stale`）不同，它使用 Codex (GPT-5.5) 对每个 issue/PR 进行**语义级审查**，以 0.1% 的极低关闭率确保不误杀有效 issue。

**核心设计哲学**：
- **保守主义**：评审只提议，从不直接关闭；执行前重新验证所有状态
- **零信任安全**：Codex 从未获得写令牌；所有 GitHub mutation 由确定性执行器完成
- **可审计性**：所有决策以纯文本 Markdown 存储在 Git 仓库中，完全可追溯

## 四车道架构 (Four Operational Lanes)

ClawSweeper 的架构被清晰地划分为**四条独立的操作车道**，每条车道负责不同的职责：

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Review Lane │    │ Apply Lane  │    │ Repair Lane │    │ Commit Lane │
│   (审查)    │───▶│   (执行)    │    │   (修复)    │    │  (提交审查) │
│  提案-only  │    │  守护式执行  │    │  自动修复   │    │  代码审查   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

*图：ClawSweeper 四车道架构示意*

### 1. 审查车道 (Review Lane)

**职责**：对 open issues/PRs 进行 Codex AI 评审，生成结构化报告。

**关键实现**（源码：[`src/clawsweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/clawsweeper.ts)）：

1. **规划阶段 (Plan)**：扫描目标仓库的所有 open issues/PRs，根据调度器优先级选择待评审项
2. **分片 (Shard)**：将待评审项分配到多个并行分片中（最多 70 个并发 shard）
3. **代码检出**：每个分片 checkout 目标仓库的 `main` 分支
4. **Codex 评审**：调用 OpenAI Codex（内部模型，高推理力度），10 分钟超时
5. **报告生成**：每个评审项生成 `records/<repo-slug>/items/<number>.md`
6. **评论同步**：在目标 Issue/PR 上同步一条 marker-backed 的评审评论

**关键常量**（[`src/clawsweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/clawsweeper.ts)）：
```typescript
const DEFAULT_CODEX_MODEL = "internal";
const DEFAULT_REASONING_EFFORT = "high";
const DEFAULT_SERVICE_TIER = "";
const REVIEW_ITEM_PROMPT_PATH = "prompts/review-item.md";
const CLAWSWEEPER_DECISION_SCHEMA_PATH = "schema/clawsweeper-decision.schema.json";
```

**边界条件**：
- 审查是**提案性质的 (proposal-only)**，从不直接关闭项目
- 每个 Issue/PR 只有一个 marker-backed 评审评论，就地编辑更新

### 2. 执行车道 (Apply Lane)

**职责**：读取已有的评审报告，执行关闭操作，但只在评审仍然有效时才执行。

**关键实现**（源码：[`src/clawsweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/clawsweeper.ts) 的 `apply-decisions` 命令）：

1. 读取已有的 `records/` 中 `proposed_close` 状态的报告
2. 重新获取 GitHub 最新状态，检查 snapshot 是否变化
3. 检查维护者身份、保护标签、关联 PR 状态等安全条件
4. 如果一切有效 → 关闭 Issue/PR
5. 将已关闭的报告移到 `records/<repo-slug>/closed/<number>.md`

**安全检查流程**：
```
1. 重新获取 GitHub 最新状态
2. 检查 snapshot hash → 如果有非 bot 变更则跳过
3. 检查维护者标签/身份
4. 检查受保护标签
5. 检查关联的 open PR (Fixes #123)
6. 检查同一作者的 Issue/PR 配对
7. 检查仓库 profile 的 applyCloseRules
8. 所有检查通过 → 执行关闭
```

**边界条件**：
- 维护者创建的项目**永不自动关闭**（除非验证为 `implemented_on_main`）
- 受保护标签阻止关闭提案
- 含 `Fixes #123` 的开放 PR 会阻止对应 Issue 关闭
- 同一作者的开放 Issue/PR 配对不会被单方面关闭

### 3. 修复车道 (Repair Lane)

**职责**：处理维护者命令，执行自动修复和自动合并。

**关键实现**：
- 评论路由：[`src/repair/comment-router.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/repair/comment-router.ts)（120KB）
- 执行修复产物：[`src/repair/execute-fix-artifact.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/repair/execute-fix-artifact.ts)（141KB）

**维护者命令**（通过 Issue/PR 评论触发）：
```
@clawsweeper review       # 重新评审
@clawsweeper autofix      # 自动修复（不合并）
@clawsweeper automerge    # 自动合并
@clawsweeper status       # 查看状态
@clawsweeper stop         # 停止自动流程
```

**自动合并状态机**：
1. 维护者发送 `@clawsweeper automerge`
2. 路由采纳 PR 分支，运行 exact-head 评审
3. 如果发现可操作问题 → Codex 修复 → 重新评审
4. 等所需 CI 检查和可合并性通过
5. 通过安全/策略门控后合并

**边界条件**：
- `automerge` 最多进行 10 轮修复循环
- Draft PRs 在 GitHub 标记为 ready for review 之前只修复不合并
- 安全敏感的修复需要显式 `autofix`/`automerge` opt-in

### 4. 提交评审车道 (Commit Review Lane)

**职责**：评审 `main` 分支上的代码提交。

**关键实现**：
- 主入口：[`src/commit-sweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/commit-sweeper.ts)
- 提交分类：[`src/commit-classifier.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/commit-classifier.ts)

**流程**：
1. 接收 `push` 事件或手动触发
2. 等待 60 秒让 push 稳定
3. 分类提交（纯文档/Changelog → 跳过）
4. 代码类提交 → 每个提交一个 Codex worker
5. 生成 `records/<repo-slug>/commits/<sha>.md`
6. 可选：在目标提交上创建 Check Run

**边界条件**：
- 纯文档、Changelog、README/license、资产类提交自动跳过，不消耗 Codex 时间
- Commit Review 不会关闭任何项目、写评论或修复代码

## AI 集成机制

### Codex 调用路径

```
clawsweeper.ts → codexEnv() 清理环境变量 →
  构造 ReviewPromptBuild → spawnSync 调用 Codex CLI →
  解析 Codex 输出的 JSON → 验证符合 JSON Schema
```

**环境变量安全**（[`src/codex-env.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/codex-env.ts)）：

```typescript
export function codexEnv(options: CodexEnvOptions = {}): NodeJS.ProcessEnv {
  const env = { ...process.env };
  // 删除所有 GitHub 写令牌
  delete env.GH_TOKEN;
  delete env.GITHUB_TOKEN;
  delete env.CLAWSWEEPER_APP_ID;
  delete env.CLAWSWEEPER_APP_PRIVATE_KEY;
  // 删除所有 AI 密钥（只传只读 GH_TOKEN）
  delete env.OPENAI_API_KEY;
  delete env.CODEX_API_KEY;
  env.GIT_OPTIONAL_LOCKS = "0";
  return env;
}
```

**结论**：Codex 运行期间**没有任何 GitHub 写令牌**，所有 GitHub mutation 由确定性 TypeScript 执行器完成。

### 评审提示系统 (Prompts)

| 提示文件 | 大小 | 用途 |
|---------|------|------|
| [`prompts/review-item.md`](https://github.com/openclaw/clawsweeper/blob/main/prompts/review-item.md) | 64KB | 核心评审提示：Issue/PR 分类、决策证据、安全检查 |
| [`prompts/review-commit.md`](https://github.com/openclaw/clawsweeper/blob/main/prompts/review-commit.md) | 8KB | 提交评审：diff 分析、安全审查 |
| [`prompts/pr-close-coverage-proof.md`](https://github.com/openclaw/clawsweeper/blob/main/prompts/pr-close-coverage-proof.md) | 2.5KB | PR 关闭覆盖率证明 |

### 决策 Schema

[`schema/clawsweeper-decision.schema.json`](https://github.com/openclaw/clawsweeper/blob/main/schema/clawsweeper-decision.schema.json)（33KB）定义了 Codex 必须返回的**严格结构化 JSON**，包含约 50 个字段：

- `decision`: `"close"` | `"keep_open"` — 核心决策
- `closeReason`: 9 种关闭原因
- `confidence`: `"high"` | `"medium"` | `"low"`
- `evidence[]`: 带有 `file`, `line`, `sha`, `command` 的结构化证据
- `securityReview`: 独立的安全审查结论
- `prRating`: S/A/B/C/D/F/NA 七级 PR 评分

## 关闭决策分类

只有以下情况才建议关闭（源码：[`src/clawsweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/clawsweeper.ts)）：

| 理由 | 含义 | 适用对象 |
|------|------|---------|
| `implemented_on_main` | 当前 main 已实现 | Issue, PR |
| `mostly_implemented_on_main` | 大部分已实现 | PR（需 60 天） |
| `cannot_reproduce` | 无法复现 | Issue |
| `clawhub` | 更适合 ClawHub 插件 | Issue |
| `duplicate_or_superseded` | 重复或已被取代 | Issue, PR |
| `low_signal_unmergeable_pr` | 低信号/不可合并 | PR |
| `not_actionable_in_repo` | 在此仓库无法操作 | Issue |
| `incoherent` | 无法理解的描述 | Issue |
| `stale_insufficient_info` | 超过 60 天且信息不足 | Issue（需 60 天） |

**极其保守**：截至 2026 年 4 月 27 日，过去 7 天 3,478 个审查的 issue 中**仅 4 个建议关闭**（0.1%）。

## 调度机制

### 多时间粒度调度

定义在 [`.github/workflows/sweep.yml`](https://github.com/openclaw/clawsweeper/blob/main/.github/workflows/sweep.yml)（129KB），通过 GitHub Actions 的 `schedule` 事件触发：

| Cron 表达式 | 频率 | 用途 |
|------------|------|------|
| `*/5 * * * *` | 每 5 分钟 | Hot intake（新的活跃项） |
| `3,18,33,48 * * * *` | 每 15 分钟 | Apply closures（应用关闭） |
| `6,21,36,51 * * * *` | 每 15 分钟 | 评论同步 |
| `41 * * * *` | 每小时 | 普通评审 |
| `7 */6 * * *` | 每 6 小时 | 审计检查 |
| `13 8,20 * * *` | 每天两次 | 重试失败的 Codex 评审 |

### 调度器优先级算法

在 [`src/clawsweeper.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/clawsweeper.ts) 中实现，调度器将项目分为多个**桶 (Buckets)**：

```typescript
type SchedulerBucket =
  | "hot_issue"       // 新的活跃 Issue（每 5 分钟）
  | "hot_pull_request" // 新的活跃 PR（每 5 分钟）
  | "activity"        // 近期有活动的
  | "daily_pull_request" // PR 每天
  | "recent_issue"    // 30 天以内的 Issue（每天）
  | "weekly_issue";   // 30 天以上的 Issue（每周）
```

**关键常量**：
```typescript
const HOT_REVIEW_DAYS = 7;        // 热窗口：7 天
const RECENT_ISSUE_DAYS = 30;     // 近期 Issue：30 天
const HOURLY_REVIEW_MS = 60 * 60 * 1000;  // 小时检查
const FRESH_DAYS = 7;             // 新鲜有效期限
const STALE_INSUFFICIENT_INFO_MIN_AGE_DAYS = 60; // 不充分信息的最短年龄
```

### 并发控制

[`config/automation-limits.json`](https://github.com/openclaw/clawsweeper/blob/main/config/automation-limits.json) + [`src/limits.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/limits.ts) 实现动态 Worker 容量管理：

```
workers.max: 从 config 读取
  → 按车道按比例分配:
    normal_review: max 的 70%
    hot_intake: max 的 35%
    commit_review: max 的 5%
    repair: max 的 40%
    exact_item: 固定 1
```

## 数据存储

### Git 作为数据库

ClawSweeper 使用**Git 仓库**作为持久化存储：

- **状态仓库**：`openclaw/clawsweeper-state` 的 `state` 分支
- **存储结构**：
```
records/
  <repo-slug>/
    items/
      <number>.md        # 开放项的评审报告
    closed/
      <number>.md        # 已关闭项的归档报告
    commits/
      <sha>.md           # 提交评审报告
```

### 报告格式

每个 `.md` 报告包含：
- 决策和关闭原因
- 证据链（文件/行号/SHA/命令）
- 建议的维护者评论
- Codex 运行时元数据（模型、耗时、token 数）
- GitHub snapshot hash（用于检测变更）

### 评论同步机制

**Marker-backed 评论系统**：
```markdown
<!-- clawsweeper-review v3 -->
[评审内容...]
<!-- /clawsweeper-review -->
```

- 每个 Issue/PR **只有一个** ClawSweeper 评审评论
- 评论通过 marker 前缀识别，就地编辑更新
- 隐藏的 `verdict/action` markers 供修复管道读取

## 维护者指令

在 issue/PR 评论中可用的指令（源码：[`src/repair/comment-router.ts`](https://github.com/openclaw/clawsweeper/blob/main/src/repair/comment-router.ts)）：

| 指令 | 功能 |
|------|------|
| `@clawsweeper status` | 显示项目摘要 |
| `@clawsweeper re-review` | 重新审查（不触发修复） |
| `@clawsweeper autofix` | 进入有限的审查/修复但**不合并**循环 |
| `@clawsweeper automerge` | 进入有限审查/修复/合并循环 |
| `@clawsweeper fix ci` | 修复 CI 问题 |
| `@clawsweeper address review` | 处理 PR 审查评论 |
| `@clawsweeper rebase` | 变基 |
| `@clawsweeper approve` | 维护者批准通过人类审查暂停 |
| `@clawsweeper stop` | 停止自动修复/合并循环 |
| `@clawsweeper explain` | 解释当前状态 |

**权限检查**：仅仓库 `admin`/`maintain`/`write` 权限或 `author_association` 验证通过者可执行写操作。

## 性能数据

| 指标 | 数据 |
|------|------|
| 7 天内新审查的 issues | ~3,700 |
| 建议关闭的 issues | ~4（0.1%） |
| 7 天内新审查的 PRs | ~3,450 |
| 建议关闭的 PRs | ~4（0.1%） |
| 自动关闭总数（自启动） | 10,000+ |
| 需审查的打开项总数 | ~7,000 |

## 与类似项目对比

| 项目 | 类型 | 与 ClawSweeper 的区别 |
|------|------|----------------------|
| **[probot/stale](https://github.com/probot/stale)** | 时间规则自动关闭 | 仅基于时间规则，无 AI 审查，无法理解内容 |
| **[actions/stale](https://github.com/actions/stale)** | GitHub Action 自动关闭 | 官方 Stale Action，同样纯时间触发，无语义理解 |
| **[better-stale-bot](https://dosu.dev/blog/an-ai-stale-bot-that-you-can-trust)** | AI 驱动的 Stale 机器人 | Dosu 的开源替代，会阅读 issue 内容，但无修复/自动合能力 |

**ClawSweeper 的关键差异点**：
1. **AI 内容审查**——不是简单查时间戳，而是用 Codex 理解每个 issue/PR 的内容
2. **提案优先**——从不直接关闭，先写报告再执行
3. **修复+自动合并**——不仅仅是关闭，还能自动修复和合并 PR
4. **0.1% 关闭率**——极其保守，不误杀有效 issue
5. **维护者指令系统**——丰富的注释指令生态

## 部署与自托管

**所需 Secrets**：
- `OPENAI_API_KEY`——Codex Responses 代理密钥
- `CLAWSWEEPER_APP_CLIENT_ID`——GitHub App 客户端 ID
- `CLAWSWEEPER_APP_PRIVATE_KEY`——GitHub App 私钥

**所需 GitHub App 权限**：
- Contents: read/write
- Issues: read/write
- Pull requests: read/write
- Workflows: write
- Actions: read/write
- Checks: write（可选）

**部署步骤**：
1. Fork 此仓库
2. 配置 [`config/target-repositories.json`](https://github.com/openclaw/clawsweeper/blob/main/config/target-repositories.json) 添加目标仓库
3. 设置 GitHub App 和 OpenAI API Key
4. 部署 GitHub Actions workflow
5. 可选：配置仪表盘 (Cloudflare Pages) 和状态仓库

## 文档索引

| 文档 | 链接 |
|------|------|
| 调度器文档 | [`docs/scheduler.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/scheduler.md) |
| 工作车道 | [`docs/work-lane.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/work-lane.md) |
| 提交审查 | [`docs/commit-sweeper.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/commit-sweeper.md) |
| 目标分发器 | [`docs/target-dispatcher.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/target-dispatcher.md) |
| PR 审查评论 | [`docs/pr-review-comments.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/pr-review-comments.md) |
| 修复车道 | [`docs/repair/README.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/repair/README.md) |
| 自动合并流程 | [`docs/repair/automerge-flow.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/repair/automerge-flow.md) |
| 关联 issue 发现 | [`docs/related-issue-discovery.md`](https://github.com/openclaw/clawsweeper/blob/main/docs/related-issue-discovery.md) |

## 相关文章

| 文章 | 来源 | 日期 |
|------|------|------|
| [ClawSweeper: How OpenClaw's Codex Bot Triages 7,000 Issues](https://apidog.com/blog/clawsweeper-openclaw-github-triage-bot/) | Apidog Blog | 2026-04-28 |
| [ClawSweeper – AI Maintenance Bot for GitHub Repositories](https://runany.dev/blog/clawsweeper/) | RunAny Dev | 2026-06-02 |

## 待深入研究

- 调度器优先级算法的完整实现细节（`docs/scheduler.md`）
- 修复车道的 Codex 提示构建逻辑（`src/repair/fix-prompt-builder.ts`）
- PR 质量评级系统（S/A/B/C/D/F/NA）的评分标准
- 真实行为证明 (Real Behavior Proof) 的验证机制
- 仪表盘的 Cloudflare Workers 实现
