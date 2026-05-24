# 13 · OpenClaw：外部系统双向回路

> 目标：理解 OmO 如何把 OpenCode 会话和外部系统（Discord/Telegram/Webhook/Shell）打通成闭环。

---

## 1) 双向而不是单向通知

OpenClaw（[openclaw/openclaw](https://github.com/openclaw/openclaw)）是独立的多通道 Gateway 产品。OmO 的 `src/openclaw/` 是在 OpenCode 插件内与其**双向集成**的桥接层，不是 OpenClaw 本体。

OpenClaw 集成有两条通道：

- 出站：OpenCode 事件 -> 外部系统
- 入站：外部回复 -> 回注 tmux pane -> 回到会话

这让 OmO 具备“会话不在前台也能继续协作”的能力。

---

## 2) 关键链路

出站：

`plugin event` -> `runtime-dispatch.ts` -> `dispatcher.ts` -> webhook/shell

入站：

外部 API 轮询 -> `reply-listener-*` -> `session-registry` -> `reply-listener-injection.ts` -> `tmux send-keys`

---

## 3) 核心文件

- `src/openclaw/index.ts`：初始化与唤醒
- `runtime-dispatch.ts`：事件映射
- `dispatcher.ts`：执行网关
- `session-registry.ts`：消息与会话/窗格关联
- `reply-listener.ts` + `daemon.ts`：后台轮询进程

---

## 4) 安全与稳定性设计

- URL 校验（HTTPS，localhost 例外）
- 授权用户过滤（入站）
- Token 脱敏日志
- pane 注入限流
- daemon 独立进程，避免阻塞主会话

---

## 5) 架构评价（这一层）

优点：

- 真正闭环（outbound + inbound）
- 与主流程解耦（daemon）
- 安全措施比较完整

不足：

- 对 tmux 与外部 API 强依赖，环境门槛高
- 涉及多进程与状态文件，故障面更大

---

## 6) 必须掌握的检查点

- [ ] 画出 OpenClaw 双向数据流
- [ ] 说清 `session-registry` 为什么是关键中介
- [ ] 明确 daemon 模式的收益与风险

