# open-agent-research-notes

个人学习笔记仓库。跨项目沉淀 coding agent 系统的理解（中文优先）。

## 仓库定位

- **读者假设**：正在学习 agent 系统的人，不是仓库维护者
- **目标**：理解系统如何构造、如何协作、如何扩展
- **不是**：产品代码仓库、changelog 仓库

## 项目索引

| 目录 | 内容 |
|------|------|
| `projects/oh-my-openagent/` | OpenCode 插件、hook 体系、多 agent 编排 |
| `projects/opencode/` | OpenCode 内核与 runtime |
| `projects/openclaw/` | 外部系统双向集成 |
| `projects/claude-code/` | Claude Code 扩展生态 |
| `comparisons/` | 跨项目横向比较 |
| `patterns/` | 可复用模式与反模式 |
| `templates/` | 新项目学习模板 |

## 文档写作约定

- 默认中文（术语可保留英文）
- 每个结论附带证据：源码路径或文档链接
- 写「结论 + 证据 + 边界条件」，避免只写结论
- **不要写**维护向信息：已迁移、操作记录、分支名、remote 调整等
- 项目索引用「内容简介」，不用「待开始 / 已迁移」等状态词

### 代码与文档链接（oh-my-openagent 笔记）

- 源码仓库：[code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（upstream）
- 个人 fork：[zhumengzhu/oh-my-openagent](https://github.com/zhumengzhu/oh-my-openagent)（仅用于 PR/实验）
- **基准版本**：每项目 README 声明 pin commit；正文证据链接必须用 **commit SHA**，不用 `dev`/`main`
- **笔记目录内跳转**：相对路径，如 `./03-chat-params-mechanism.md`
- **指向 OmO 源码或文档**（举证链接）：
  `https://github.com/code-yeongyu/oh-my-openagent/blob/<40-char-sha>/path/to/file#L10-L20`
- **禁止** `../../src/`、`../manifesto.md` 等跨仓库相对路径
- **禁止** 证据链接使用 `blob/dev/`（分支 HEAD 会变，路径与行号会失效）
- 新增/更新笔记后：用 `git cat-file -e <sha>:<path>` 验证路径；大版本对齐时更新 baseline commit
- 代码块 `` ```12:34:src/... `` 为 Cursor 引用；GitHub 阅读请用同路径的 blob+SHA 链接

## Commit 约定

- 格式：**添加 / 补充 / 更新 + 项目 + 主题**
- 示例：
  - `添加 oh-my-openagent 学习笔记（00–16 + 99）`
  - `补充 opencode：session 生命周期主链路`
  - `更新 comparisons：hook 系统横向对比`
- **避免**运维向描述：迁移、重构目录、调整 remote（除非 commit 本身就是结构变更）
- 保持历史简洁：相关改动可 squash 为一条语义完整的 commit

## AI 助手在本仓库的工作方式

1. 新增笔记放 `projects/<name>/`，按编号或主题组织
2. 跨项目结论沉淀到 `comparisons/` 或 `patterns/`
3. 不要在本仓库写产品代码；实践代码放对应源码仓库
4. 提交前检查：文案是否面向学习者、commit message 是否符合上文约定
