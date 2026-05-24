# 关键流程索引

> **用途：** 把分散在 00–17 里的「流程型」知识集中导航——借鉴 [ZeroZ-lab flow/](https://github.com/ZeroZ-lab/learn-opencode/tree/main/docs/flow) 的组织方式，内容仍在本笔记各章。

每条流程：**一句话 → 本笔记章节 → 可选社区补充**。

---

## 主链路

| 流程 | 一句话 | 本笔记 | 社区补充 |
|------|--------|--------|----------|
| **Agent 生命周期** | 用户消息 → runLoop → LLM → Tool → 再 loop | [18](../18-main-chain-atlas.md) · [A1](../appendix/A1-full-trace-narrative.md) · [09](../09-session-prompt-runloop.md) | [ZeroZ agent_lifecycle](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/agent_lifecycle.md) |
| **单轮 LLM 数据流** | messages → transform → chat.params → stream → Part 落库 | [18 §4](../18-main-chain-atlas.md) · [A2](../appendix/A2-processor-deep-dive.md) · [10](../10-llm-stream-and-provider.md) | — |
| **Bootstrap 启动** | directory → Instance → plugin.init → config hook | [02](../02-effect-instance-and-bootstrap.md) · [03](../03-cli-server-and-entry.md) | ZeroZ `internals/project.md` |

---

## 扩展与治理

| 流程 | 一句话 | 本笔记 | 社区补充 |
|------|--------|--------|----------|
| **插件加载** | 扫描配置 → 工厂函数 → PluginInterface 注册 | [05](../05-plugin-protocol-and-loader.md) · [06](../06-hook-system-reference.md) | [ZeroZ plugin_loading](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/plugin_loading.md) |
| **工具执行** | registry 合并 → before → execute → after | [11](../11-tool-registry-and-execution.md) | [ZeroZ tool_execution](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/tool_execution.md) |
| **权限求值** | ruleset → ask / allow / deny | [07](../07-agent-and-permission.md) | [ZeroZ permission_flow](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/permission_flow.md) |
| **上下文压缩** | token 压力 → compacting hook → 摘要替换 | [12](../12-compaction-and-context-management.md) | qqzhangyanhua 第 4 章 §压缩 |
| **快照 / 回滚** | Git snapshot → revert | [15](../15-background-pty-and-worktree.md) | [ZeroZ snapshot_rollback](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/snapshot_rollback.md) |

---

## 横切与客户端

| 流程 | 一句话 | 本笔记 | 社区补充 |
|------|--------|--------|----------|
| **Bus 事件** | Session 状态 → SSE → 客户端 | [13](../13-bus-and-session-events.md) | — |
| **状态同步（UI）** | Server-Driven，前端哑终端 | [03](../03-cli-server-and-entry.md) · [01](../01-monorepo-and-packages.md) | [ZeroZ state_sync](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/state_sync.md) |
| **MCP 连接** | 配置 → 客户端 → tool 暴露 | [14](../14-mcp-lsp-skill-and-command.md) | ZeroZ `internals/mcp-implementation.md` |
| **错误 / 重试** | session.error → retry / fallback | [09](../09-session-prompt-runloop.md) · [10](../10-llm-stream-and-provider.md) | [ZeroZ error_handling](https://github.com/ZeroZ-lab/learn-opencode/blob/main/docs/flow/error_handling.md) |

---

## 插件层（OmO，可选）

若安装了 oh-my-openagent，在**内核流程之上**还有插件内部 hook 链：

| 流程 | 本笔记 | 社区补充 |
|------|--------|----------|
| 一条消息的 OmO hook 链 | [OmO 17](../oh-my-openagent/17-message-hook-journey.md) · [边界 §4](../../comparisons/opencode-vs-omo-boundary.md) | [qqzhangyanhua oh-flow](https://github.com/qqzhangyanhua/learn-opencode-agent/blob/main/docs/oh-flow/index.md) |

→ **选学习路径：** [learning-paths.md](../learning-paths.md)
