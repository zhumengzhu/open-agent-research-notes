# 14 · MCP 三层体系与 CLI 可运维面

> 目标：掌握 OmO 的外部能力接入（MCP）和日常运维入口（CLI/doctor/run）。

---

## 1) MCP 三层模型

1. Tier-1 内置 MCP：`src/mcp/`（全局远程服务）
2. Tier-2 Claude Code MCP：来自 `.mcp.json`
3. Tier-3 Skill-embedded MCP：`src/features/skill-mcp-manager/`

你要重点掌握 Tier-3，因为它最复杂也最关键。

---

## 2) Skill MCP Manager 的关键设计

目录：`src/features/skill-mcp-manager/`

- client key：`${sessionID}:${skillName}:${serverName}`（会话隔离）
- 双传输：stdio + HTTP
- 连接竞态防护：`pendingConnections`
- 会话删除清理：`plugin/event.ts` 的 `session.deleted`
- 空闲回收：定时清理 + TTL
- OAuth step-up：401/403 自动处理重试链

这是 OmO “插件内动态扩展外部能力”的核心。

---

## 3) CLI 子系统（运维入口）

目录：`src/cli/`

核心命令：

- `install`：安装与配置
- `run`：非交互执行
- `doctor`：系统诊断（System/Config/Tools/Models）
- `mcp-oauth`：MCP OAuth 登录/状态
- `refresh-model-capabilities`：刷新模型能力缓存
- `boulder`：状态查看

---

## 4) 为什么 CLI 很关键

- CLI 是“工程可运营性”入口，不只是开发辅助
- `doctor` 是快速定位环境问题的第一工具
- `run` 支持 CI/自动化，保证 OmO 不局限在交互式前端

---

## 5) 架构评价（这一层）

优点：

- MCP 分层清楚，便于扩展
- Tier-3 关注会话隔离和安全，设计成熟
- CLI 命令面完整，具备产品化运维能力

不足：

- MCP 体系概念多，学习成本偏高
- OAuth + 多传输 + 回收策略组合复杂，调试门槛高

---

## 6) 必须掌握的检查点

- [ ] 解释三层 MCP 的边界和来源
- [ ] 解释 Tier-3 为什么必须按 session 隔离
- [ ] 出现“技能 MCP 失效”时，知道先看连接、鉴权、清理哪一层
- [ ] `doctor` 四类检查能说清

