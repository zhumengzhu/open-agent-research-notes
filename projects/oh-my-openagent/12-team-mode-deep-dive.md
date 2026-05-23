# 12 · Team Mode 深潜：并行多代理协作系统

> 目标：掌握 Team Mode 的运行机制、数据结构、不变量，以及它和普通 `task` 委派的区别。

---

## 1) Team Mode 不是什么

- 不是 `task` 工具的“语法糖”
- 不是临时并发调用
- 不是任意 agent 都可入队

它是独立的“团队运行时”（共享 mailbox/tasklist/worktree + 生命周期管理）。

---

## 2) 核心组成

目录：`src/features/team-mode/`

- `team-runtime/`：创建/状态/关闭
- `team-mailbox/`：异步消息收发
- `team-tasklist/`：任务共享与抢占
- `team-worktree/`：成员独立 git worktree
- `team-state-store/`：运行状态持久化与锁
- `team-layout-tmux/`：可选 tmux 视觉布局
- `tools/`：12 个 `team_*` 工具

---

## 3) 生命周期

1. `team_create`：加载 TeamSpec、校验成员、初始化状态
2. `team_send_message` / `team_task_*`：团队协作执行
3. `team_shutdown_request` + approve/reject：优雅关闭
4. `team_delete`：清理状态、任务、邮箱、worktree、tmux

---

## 4) 关键不变量（必须背）

- **Spawn race-safe**：先查 `team-session-registry` 再查 runtime state
- **任务抢占锁**：`team-tasklist` 必须原子锁，避免双 claim
- **原子写**：状态文件写临时文件后 rename
- **成员资格前置校验**：不在 parse 通过后再运行时拒绝
- **禁止嵌套 team**：成员不能再 `team_create`

---

## 5) 与普通 `task` 的分工

- `task`：一次性子任务委派，轻量
- `team_*`：长期协作运行时，重型

经验法则：

- 单问题拆两三步：`task`
- 多角色持续协作、需要共享任务池与消息队列：`team-mode`

---

## 6) 架构评价（这一层）

优点：

- 状态模型明确（state/mailbox/task/worktree 分离）
- 工具面完整（生命周期、消息、任务、查询全覆盖）
- 不变量写得很清楚，利于长期演进

不足：

- 系统复杂度高，学习曲线陡
- IO 与锁密集，出问题时排障成本高
- 对 tmux / git 环境依赖明显（可移植性受限）

---

## 7) 必须掌握的检查点

- [ ] 说清为何 `team_*` 不能被 `task` 替代
- [ ] 能解释 12 个 `team_*` 工具分哪三类（生命周期/消息/任务/查询）
- [ ] 能复述 5 条关键不变量

