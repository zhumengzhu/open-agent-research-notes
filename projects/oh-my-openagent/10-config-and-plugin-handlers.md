# 10 · 配置系统与 6 阶段装配流水线

> 目标：掌握 OmO 如何把“用户配置 + 项目配置 + 默认值”转成最终可运行的 agents/tools/mcps/commands。

---

## 1) 两层配置 + 统一 schema

- 根 schema 在 `src/config/schema/oh-my-opencode-config.ts`
- 校验库：Zod v4（`safeParse`）
- 自动导出 JSON Schema：`assets/oh-my-opencode.schema.json`

合并顺序（近处优先）：

1. 用户配置：`~/.config/opencode/oh-my-openagent.jsonc`
2. 项目配置：`<project>/.opencode/oh-my-openagent.jsonc`
3. Zod 默认值补齐

关键规则：

- `agents` / `categories` / `claude_code`：深合并
- `disabled_*`：并集去重
- 其他字段：后者覆盖前者

---

## 2) `config` hook 的 6 阶段

实现位于 `src/plugin-handlers/`：

1. `applyProviderConfig`：provider 能力缓存、上下文限制
2. `loadPluginComponents`：发现 Claude Code 插件组件（带超时隔离）
3. `applyAgentConfig`：装配 agents（含技能与分类）
4. `applyToolConfig`：按 agent 做 tool 权限裁剪
5. `applyMcpConfig`：合并三层 MCP
6. `applyCommandConfig`：命令与技能来源汇总

这条流水线就是 OmO 的“装配内核”。

---

## 3) 为什么这套设计是好的

- **可演化**：每阶段独立，新增配置能力不会冲垮总入口
- **可回滚**：某阶段失败容易定位（边界清晰）
- **可测试**：每阶段可独立单测
- **可观测**：可以在阶段边界打日志和诊断

潜在代价：

- 配置来源多，心智负担高
- 同一字段可能受多层影响，排障要看“来源链”

---

## 4) 必须掌握的检查点

- [ ] 说清 6 阶段各自产物
- [ ] 解释 `disabled_*` 为何是并集而不是覆盖
- [ ] 知道改 schema 后要跑 `bun run build:schema`
- [ ] 出现“配置不生效”时，能按来源链排查（用户/项目/默认）

