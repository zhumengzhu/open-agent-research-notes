# Multica 小队模式深度分析

> **基准版本：** `multica-ai/multica@v0.3.20`（2026-06-11）
> **分析日期：** 2026-06-12
> **数据来源：** 本地代码库深读（server/internal/handler/squad*.go, server/internal/daemon/prompt.go, packages/core/types/squad.ts 等）

---

## 目录

1. [概述](#1-概述)
2. [核心数据模型](#2-核心数据模型)
3. [上下文传输机制](#3-上下文传输机制)
4. [任务编排流程](#4-任务编排流程)
5. [触发机制](#5-触发机制)
6. [循环防护](#6-循环防护)
7. [状态同步](#7-状态同步)
8. [关键设计决策](#8-关键设计决策)
9. [代码索引](#9-代码索引)

---

## 1. 概述

### 什么是小队模式

小队模式（Squads）是 Multica 的多 agent 协作机制。一个 Squad 由一个 **Leader Agent** 和多个 **成员**（agent 或人类）组成。当 issue 分配给 Squad 时，Leader 负责协调，通过 `@mention` 机制将任务委派给合适的成员执行。

### 核心理念

```
Leader 负责协调，成员负责执行，通过 @mention 实现松耦合。
```

**关键特性：**
- Leader 是唯一入口：所有 Squad 任务必须经过 Leader
- @mention 委派：成员通过评论中的 mention 链接触发
- 上下文一次性注入：Briefing 在任务认领时注入，成员通过 issue 评论历史获取完整上下文
- 强制评估记录：Leader 每次触发必须记录决策（action/no_action/failed）

> *证据来源：`server/internal/handler/squad_briefing.go` 中的 `squadOperatingProtocol` 常量*

---

## 2. 核心数据模型

### 数据库表结构

```sql
-- squad 表
CREATE TABLE squad (
    id            UUID PRIMARY KEY,
    workspace_id  UUID NOT NULL,
    name          TEXT NOT NULL,
    description   TEXT,
    instructions  TEXT,          -- 用户自定义指令
    avatar_url    TEXT,
    leader_id     UUID NOT NULL, -- 必须是 agent
    creator_id    UUID NOT NULL, -- 创建者（人类）
    created_at    TIMESTAMPTZ,
    updated_at    TIMESTAMPTZ,
    archived_at   TIMESTAMPTZ,
    archived_by   UUID
);

-- squad_member 表
CREATE TABLE squad_member (
    id          UUID PRIMARY KEY,
    squad_id    UUID NOT NULL,
    member_type TEXT NOT NULL,  -- "agent" | "member"
    member_id   UUID NOT NULL,
    role        TEXT,           -- e.g. "leader", "frontend", "backend"
    created_at  TIMESTAMPTZ,
    UNIQUE(squad_id, member_type, member_id)
);

-- issue 表（相关字段）
ALTER TABLE issue ADD COLUMN assignee_type TEXT;  -- "agent" | "squad" | "member"
ALTER TABLE issue ADD COLUMN assignee_id UUID;    -- 指向 agent.id 或 squad.id
```

> *证据来源：`server/pkg/db/queries/squad.sql`、`server/migrations/084_squad.up.sql`*

### TypeScript 类型定义

```typescript
// packages/core/types/squad.ts

export interface Squad {
  id: string;
  workspace_id: string;
  name: string;
  description: string;
  instructions: string;           // 用户自定义指令
  avatar_url: string | null;
  leader_id: string;              // 必须是 agent
  creator_id: string;
  member_count?: number;
  member_preview?: SquadMemberPreview[];
}

export interface SquadMember {
  id: string;
  squad_id: string;
  member_type: "agent" | "member";
  member_id: string;
  role: string;
}

// 成员状态（5 种）
export type SquadMemberStatusValue =
  | "working"    // 有活跃任务
  | "idle"       // runtime 在线，无任务
  | "offline"    // runtime 离线
  | "unstable"   // runtime 最近 5 分钟内有活动但当前离线
  | "archived";  // 已归档
```

> *证据来源：`packages/core/types/squad.ts`*

---

## 3. 上下文传输机制

### 3.1 Squad Leader Briefing

当 Leader 认领任务时，系统自动注入三段上下文：

```go
// server/internal/handler/squad_briefing.go

func buildSquadLeaderBriefing(ctx context.Context, q *db.Queries, squad db.Squad) string {
    var sb strings.Builder
    
    // 1️⃣ Squad Operating Protocol（硬编码系统指令）
    sb.WriteString(squadOperatingProtocol)
    sb.WriteString("\n\n")
    
    // 2️⃣ Squad Roster（动态生成成员列表）
    sb.WriteString(buildSquadRoster(ctx, q, squad))
    
    // 3️⃣ Squad Instructions（用户自定义）
    if trimmed := strings.TrimSpace(squad.Instructions); trimmed != "" {
        sb.WriteString("\n\n## Squad Instructions (")
        sb.WriteString(squad.Name)
        sb.WriteString(")\n\n")
        sb.WriteString(trimmed)
    }
    
    return sb.String()
}
```

> *证据来源：`server/internal/handler/squad_briefing.go:104-117`*

### 3.2 Operating Protocol（硬编码）

```markdown
## Squad Operating Protocol

You are the LEADER of a squad. Your job is to **coordinate**, not to execute
the work yourself.

Your responsibilities, in order:

1. **Read the issue** and decide which squad member is best suited to do the work.
2. **Delegate by @mention.** Post a single comment that @mentions the chosen member(s).
   - **Be terse.** Every Multica agent already has full context.
   - Say only what cannot be inferred from the issue.
   - Use exact mention markdown: `[@Name](mention://<type>/<UUID>)`
3. **Record your evaluation.** After every trigger:
   `multica squad activity <issue-id> <outcome> --reason "<short reason>"`
   Outcome values: `action`, `no_action`, `failed`
4. **Stop after dispatching.** Once delegation is posted, end your turn.
5. **Re-evaluate on each trigger.** When you wake up again, read new activity and decide.

Hard rules:
- EVERY delegation MUST use full mention markdown syntax
- Do NOT restate the issue body
- Do NOT do the implementation yourself
- One delegation comment per turn
- ALWAYS call `multica squad activity` before ending
```

> *证据来源：`server/internal/handler/squad_briefing.go:19-89` 中的 `squadOperatingProtocol` 常量*

### 3.3 Squad Roster（动态生成）

系统自动扫描 Squad 成员，生成可直接粘贴的 mention 链接：

```markdown
## Squad Roster

Leader (you):
- Alice — agent — `[@Alice](mention://agent/550e8400-e29b-41d4-a716-446655440000)`

Members:
- Bob — agent, role: "frontend" — `[@Bob](mention://agent/6ba7b810-9dad-11d1-80b4-00c04fd430c8)`
- Carol — agent, role: "backend" — `[@Carol](mention://agent/6ba7b811-9dad-11d1-80b4-00c04fd430c8)`
- Dave — member (human), role: "reviewer" — `[@Dave](mention://member/7ca7b812-9dad-11d1-80b4-00c04fd430c8)`
```

**关键实现细节：**
- 跳过已归档的 agent 成员
- 跳过 Leader 自身（避免自委派）
- 成员列表按插入顺序排列

> *证据来源：`server/internal/handler/squad_briefing.go:119-165` 中的 `buildSquadRoster` 函数*

### 3.4 Task 结构体（完整上下文载体）

```go
// server/internal/daemon/types.go

type Task struct {
    // 基础标识
    ID          string `json:"id"`
    AgentID     string `json:"agent_id"`
    RuntimeID   string `json:"runtime_id"`
    IssueID     string `json:"issue_id"`
    WorkspaceID string `json:"workspace_id"`
    
    // 上下文注入
    WorkspaceContext    string      // 工作区级系统提示
    Agent               *AgentData  // agent 信息 + skills + model
    Repos               []RepoData  // 仓库信息
    ProjectResources    []ProjectResourceData  // 项目资源
    
    // 触发链追踪
    TriggerCommentID      string  // 触发评论 ID
    TriggerCommentContent string  // 评论内容（直接嵌入 prompt）
    TriggerAuthorType     string  // "agent" | "member"
    TriggerAuthorName     string  // 触发者名称
    
    // Squad 特有
    SquadID               string  // 小队 UUID
    SquadName             string  // 小队名称
    
    // 会话恢复
    PriorSessionID        string  // 上次 Claude session ID
    PriorWorkDir          string  // 上次工作目录
    
    // 评论增量
    NewCommentCount       int     // 自上次运行后的新评论数
    NewCommentsSince      string  // 增量锚点（RFC3339）
}
```

> *证据来源：`server/internal/daemon/types.go:36-105`*

### 3.5 Prompt 构建（按任务类型分流）

```go
// server/internal/daemon/prompt.go

func BuildPrompt(task Task, provider string) string {
    switch {
    case task.ChatSessionID != "":      return buildChatPrompt(task)
    case task.TriggerCommentID != "":   return buildCommentPrompt(task, provider)
    case task.AutopilotRunID != "":     return buildAutopilotPrompt(task)
    case task.QuickCreatePrompt != "":  return buildQuickCreatePrompt(task)
    default:                            return buildIssuePrompt(task)
    }
}
```

**Squad Leader 特殊处理：**

```go
// 检测是否是 Squad Leader
if task.Agent != nil && strings.Contains(task.Agent.Instructions, "## Squad Operating Protocol") {
    fmt.Fprintf(&b, "⚠️ **Squad leader no_action rule:** If you decide no action is needed, "+
        "call `multica squad activity %s no_action --reason \"...\"` and EXIT. "+
        "DO NOT post any comment.\n\n", task.IssueID)
}
```

> *证据来源：`server/internal/daemon/prompt.go:157-159`*

---

## 4. 任务编排流程

### 4.1 完整生命周期

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Squad 任务编排流程                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1️⃣ Issue 分配给 Squad                                              │
│     │                                                                │
│     ▼                                                                │
│  enqueueSquadLeaderTask()                                           │
│     │ 检查 isSquadLeaderReady()                                     │
│     │ 检查 HasPendingTaskForIssueAndAgent()                         │
│     ▼                                                                │
│  TaskService.EnqueueTaskForSquadLeader()                            │
│     │ 创建任务 (is_leader_task=true)                                │
│     │ 广播 EventTaskQueued                                          │
│     ▼                                                                │
│                                                                      │
│  2️⃣ Leader 认领任务                                                  │
│     │                                                                │
│     ▼                                                                │
│  buildSquadLeaderBriefing() 注入：                                  │
│     ├─ Operating Protocol                                            │
│     ├─ Squad Roster (含 mention 链接)                               │
│     └─ Squad Instructions                                            │
│                                                                      │
│  3️⃣ Leader 决策 + 委派                                               │
│     │                                                                │
│     ▼                                                                │
│  Leader 发布 comment:                                                │
│     "[@Bob](mention://agent/<UUID>) 请处理这个前端任务"              │
│     │                                                                │
│     ▼                                                                │
│  computeMentionedAgentCommentTriggers()                             │
│     │ 解析 mention://agent/<UUID>                                    │
│     ▼                                                                │
│  EnqueueTaskForMention() → 为 Bob 创建任务                          │
│                                                                      │
│  4️⃣ 成员执行                                                         │
│     │                                                                │
│     ▼                                                                │
│  Bob 执行任务，发布评论/更新状态                                     │
│     │                                                                │
│     ▼                                                                │
│  computeAssignedSquadLeaderCommentTrigger()                         │
│     │ 检测到新评论，唤醒 Leader                                      │
│     ▼                                                                │
│                                                                      │
│  5️⃣ Leader 重新评估                                                  │
│     │                                                                │
│     ▼                                                                │
│  Leader 读取新评论，决定：                                           │
│     ├─ 继续委派下一个任务                                            │
│     ├─ 记录 no_action 并退出                                         │
│     └─ 关闭 Issue                                                    │
│                                                                      │
│  6️⃣ 循环直到完成                                                     │
│     │                                                                │
│     ▼                                                                │
│  子 Issue 完成 → notifyParentOfChildDone()                          │
│     │ 系统评论 + 唤醒父 Issue 的 Squad Leader                       │
│     ▼                                                                │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键函数调用链

```
Issue 分配给 Squad
  └─→ handler.issue.go:UpdateIssue()
      └─→ handler.squad.go:shouldEnqueueSquadLeaderOnAssign()
          └─→ handler.squad.go:isSquadLeaderReady()
              └─→ service.AgentReadiness()
          └─→ handler.squad.go:enqueueSquadLeaderTask()
              └─→ service.task.go:EnqueueTaskForSquadLeader()
                  └─→ service.task.go:enqueueMentionTask(is_leader=true)
                      └─→ db.CreateAgentTask()

Leader 认领任务
  └─→ handler.daemon.go:ClaimTask()
      └─→ handler.squad_briefing.go:buildSquadLeaderBriefing()
          ├─→ buildSquadRoster()
          └─→ formatMention()
      └─→ daemon.prompt.go:BuildPrompt()

Leader 委派 @mention
  └─→ handler.comment.go:CreateComment()
      └─→ handler.comment.go:computeCommentAgentTriggers()
          └─→ handler.comment.go:computeMentionedAgentCommentTriggers()
              └─→ util.ParseMentions()
              └─→ service.task.go:EnqueueTaskForMention()

成员发布评论
  └─→ handler.comment.go:CreateComment()
      └─→ handler.comment.go:computeAssignedSquadLeaderCommentTrigger()
          └─→ service.task.go:EnqueueTaskForSquadLeader()
```

> *证据来源：`server/internal/handler/squad.go`、`server/internal/handler/comment.go`、`server/internal/service/task.go`*

---

## 5. 触发机制

### 5.1 四条触发路径

| 触发路径 | 代码位置 | 描述 |
|---------|---------|------|
| **Issue 分配** | `handler/squad.go:enqueueSquadLeaderTask` | 分配给 Squad 立即触发 Leader |
| **评论触发** | `handler/comment.go:computeAssignedSquadLeaderCommentTrigger` | Squad Issue 上的任何评论唤醒 Leader |
| **@squad mention** | `handler/comment.go:computeMentionedAgentCommentTriggers` | `[@SquadName](mention://squad/<UUID>)` 触发 Leader |
| **子 Issue 完成** | `handler/issue_child_done.go:triggerChildDoneSquad` | 子 Issue 完成时系统评论 + 唤醒 Leader |

### 5.2 Issue 分配触发

```go
// server/internal/handler/squad.go

func (h *Handler) shouldEnqueueSquadLeaderOnAssign(ctx context.Context, issue db.Issue) bool {
    // backlog 状态不触发（parking lot）
    if issue.Status == "backlog" {
        return false
    }
    return h.isSquadLeaderReady(ctx, issue)
}

func (h *Handler) isSquadLeaderReady(ctx context.Context, issue db.Issue) bool {
    // 检查 issue 是否分配给 squad
    if !issue.AssigneeType.Valid || issue.AssigneeType.String != "squad" || !issue.AssigneeID.Valid {
        return false
    }
    
    // 获取 squad 和 leader
    squad, err := h.Queries.GetSquadInWorkspace(...)
    agent, err := h.Queries.GetAgent(ctx, squad.LeaderID)
    
    // 检查 agent 就绪状态
    ready, _, err := service.AgentReadiness(ctx, h.Queries, agent)
    return ready
}
```

> *证据来源：`server/internal/handler/squad.go:946-980`*

### 5.3 评论触发

```go
// server/internal/handler/comment.go

func (h *Handler) computeAssignedSquadLeaderCommentTrigger(ctx context.Context, issue db.Issue, ...) {
    // 检查 issue 是否分配给 squad
    if !issue.AssigneeType.Valid || issue.AssigneeType.String != "squad" {
        return
    }
    
    // 获取 squad 和 leader
    squad, err := h.Queries.GetSquadInWorkspace(...)
    
    // 防止 leader 自触发
    if h.lastTaskWasLeader(ctx, issue.ID, squad.LeaderID) {
        return
    }
    
    // 入队 leader 任务
    h.enqueueSquadLeaderTask(ctx, issue, triggerCommentID, authorType, authorID)
}
```

> *证据来源：`server/internal/handler/comment.go` 中的 `computeAssignedSquadLeaderCommentTrigger` 函数*

### 5.4 @mention 触发

```go
// server/internal/handler/comment.go

func (h *Handler) computeMentionedAgentCommentTriggers(ctx context.Context, ...) {
    mentions := util.ParseMentions(content)
    
    for _, m := range mentions {
        switch m.Type {
        case "squad":
            // 获取 squad 和 leader
            squad, err := h.Queries.GetSquad(...)
            
            // 入队 leader 任务
            h.enqueueSquadLeaderTask(ctx, issue, commentID, "member", authorID)
            
        case "agent":
            // 直接入队 agent 任务
            h.TaskService.EnqueueTaskForMention(ctx, issue, agentID, commentID)
        }
    }
}
```

> *证据来源：`server/internal/handler/comment.go` 中的 `computeMentionedAgentCommentTriggers` 函数*

### 5.5 子 Issue 完成触发

```go
// server/internal/handler/issue_child_done.go

func (h *Handler) triggerChildDoneSquad(ctx context.Context, parent, child db.Issue, ...) {
    squad, err := h.Queries.GetSquadInWorkspace(...)
    
    // 循环检测：子 Issue 分配给同一个 Squad
    if childAssigneeIsSquad(child, parent.AssigneeID) {
        return
    }
    
    // 循环检测：共享 Leader
    if owner := h.effectiveChildAgentOwner(ctx, child); owner.Valid &&
        uuidToString(owner) == uuidToString(squad.LeaderID) {
        return
    }
    
    // 入队 leader 任务
    h.TaskService.EnqueueTaskForSquadLeader(ctx, parent, squad.LeaderID, triggerCommentID)
}
```

> *证据来源：`server/internal/handler/issue_child_done.go:304-351`*

---

## 6. 循环防护

### 6.1 Leader 自触发检测

```go
// server/internal/handler/squad.go

func (h *Handler) lastTaskWasLeader(ctx context.Context, issueID, agentID pgtype.UUID) bool {
    flag, err := h.Queries.GetLatestTaskIsLeaderForIssueAndAgent(ctx, ...)
    if err != nil {
        return false
    }
    return flag
}
```

**作用：** 当 Leader 发布委派评论时，不会触发自己重新执行。

> *证据来源：`server/internal/handler/squad.go:915-924`*

### 6.2 待处理任务去重

```go
hasPending, err := h.Queries.HasPendingTaskForIssueAndAgent(ctx, db.HasPendingTaskForIssueAndAgentParams{
    IssueID: issue.ID,
    AgentID: squad.LeaderID,
})
if err != nil || hasPending {
    return // 跳过入队
}
```

**作用：** 同一个 agent 在同一个 issue 上只能有一个 pending 任务。

> *证据来源：`server/internal/handler/squad.go:999-1005`*

### 6.3 子 Issue 完成循环检测

```go
// server/internal/handler/issue_child_done.go

func childAssigneeIsSquad(child db.Issue, squadID pgtype.UUID) bool {
    if !child.AssigneeType.Valid || child.AssigneeType.String != "squad" || !child.AssigneeID.Valid {
        return false
    }
    return uuidToString(child.AssigneeID) == uuidToString(squadID)
}

func (h *Handler) effectiveChildAgentOwner(ctx context.Context, child db.Issue) pgtype.UUID {
    switch child.AssigneeType.String {
    case "agent":
        return child.AssigneeID
    case "squad":
        squad, _ := h.Queries.GetSquadInWorkspace(...)
        return squad.LeaderID // 返回 leader 作为有效所有者
    }
    return pgtype.UUID{}
}
```

**作用：** 防止子 Issue 完成时触发同一个 Squad 或共享 Leader 的循环。

> *证据来源：`server/internal/handler/issue_child_done.go:367-392`*

### 6.4 Leader 角色标记

```go
// server/internal/service/task.go

func (s *TaskService) EnqueueTaskForSquadLeader(ctx context.Context, issue db.Issue, leaderID pgtype.UUID, triggerCommentID pgtype.UUID) (db.AgentTaskQueue, error) {
    return s.enqueueMentionTask(ctx, issue, leaderID, triggerCommentID, true, false)
    //                                                                       ^^^^
    //                                                                       is_leader=true
}

func (s *TaskService) enqueueMentionTask(ctx context.Context, ..., isLeader bool, ...) (db.AgentTaskQueue, error) {
    task, err := s.Queries.CreateAgentTask(ctx, db.CreateAgentTaskParams{
        // ...
        IsLeaderTask: pgtype.Bool{Bool: isLeader, Valid: isLeader},
    })
}
```

**作用：** 区分 Leader 角色和 Worker 角色（同一 agent 可以同时是两者）。

> *证据来源：`server/internal/service/task.go:504-548`*

---

## 7. 状态同步

### 7.1 WebSocket 事件

```go
// server/pkg/protocol/events.go

const (
    // 任务生命周期
    EventTaskQueued     = "task:queued"
    EventTaskDispatched = "task:dispatched"
    EventTaskRunning    = "task:running"
    EventTaskCompleted  = "task:completed"
    EventTaskCancelled  = "task:cancelled"
    EventTaskFailed     = "task:failed"
    
    // Squad 事件
    EventSquadCreated   = "squad:created"
    EventSquadUpdated   = "squad:updated"
    EventSquadDeleted   = "squad:deleted"
    
    // 评论事件
    EventCommentCreated = "comment:created"
    EventCommentUpdated = "comment:updated"
    EventCommentDeleted = "comment:deleted"
)
```

> *证据来源：`server/pkg/protocol/events.go`*

### 7.2 前端状态管理

```typescript
// React Query 拥有所有 server state
packages/core/workspace/queries.ts
  └─ squadListOptions()           // Squad 列表查询
  └─ squadMemberStatusOptions()   // 成员状态查询（30s staleTime）

// Zustand 拥有 client state
packages/core/squads/stores/view-store.ts
  └─ scope: "mine" | "all"   // 视图筛选状态

// WS → React Query 同步
packages/core/realtime/use-realtime-sync.ts
  └─ squad:created → invalidateQueries(squadKeys.all)
  └─ squad:updated → invalidateQueries(squadKeys.all)
  └─ squad:deleted → invalidateQueries(squadKeys.all) + issueKeys.all
```

> *证据来源：`packages/core/workspace/queries.ts`、`packages/core/realtime/use-realtime-sync.ts`*

### 7.3 成员状态派生

```go
// server/internal/handler/squad.go

func deriveSquadMemberStatus(
    archived bool,
    runtimeStatus pgtype.Text,
    lastSeen pgtype.Timestamptz,
    hasActiveTask bool,
    now time.Time,
) string {
    if archived {
        return "archived"
    }
    if hasActiveTask {
        return "working"
    }
    if !runtimeStatus.Valid {
        return "offline"
    }
    if runtimeStatus.String == "online" {
        return "idle"
    }
    if !lastSeen.Valid {
        return "offline"
    }
    if now.Sub(lastSeen.Time) < 5*time.Minute {
        return "unstable"
    }
    return "offline"
}
```

**状态优先级：** archived > working > offline > idle > unstable

> *证据来源：`server/internal/handler/squad.go:492-518`*

---

## 8. 关键设计决策

| 决策 | 原因 | 证据 |
|-----|------|------|
| **Leader 作为唯一入口** | 保证路由稳定性（Leader 不变，成员可变） | `squad.leader_id` 是必填字段 |
| **@mention 委派** | 松耦合，成员独立执行，无需 Leader 等待 | `squadOperatingProtocol` 中的规则 |
| **硬编码 Operating Protocol** | 保证所有 Squad 行为一致 | `squad_briefing.go:19-89` |
| **动态 Roster 生成** | 反映当前成员状态（跳过归档 agent） | `buildSquadRoster()` 函数 |
| **is_leader_task 标记** | 区分 Leader 角色和 Worker 角色 | `agent_task_queue.is_leader_task` 字段 |
| **强制 activity 记录** | 人类可追踪 Leader 决策过程 | `RecordSquadLeaderEvaluation` 端点 |
| **增量评论计数** | 避免重复读取，优化长 Issue 场景 | `NewCommentCount` 字段 |
| **循环防护** | 防止 @mention 触发无限循环 | `lastTaskWasLeader()` + `HasPendingTask` |

---

## 9. 代码索引

### 后端核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `server/internal/handler/squad.go` | 1014 | Squad CRUD + 触发逻辑 + 成员状态 |
| `server/internal/handler/squad_briefing.go` | 218 | Leader Briefing 编译器 |
| `server/internal/handler/comment.go` | ~300 | 评论触发引擎（Squad 部分） |
| `server/internal/handler/issue_child_done.go` | 392 | 子 Issue 完成通知 |
| `server/internal/service/task.go` | 1300+ | 任务入队服务 |
| `server/internal/daemon/prompt.go` | 272 | Prompt 构建器 |
| `server/internal/daemon/types.go` | 167 | Task 结构体定义 |
| `server/pkg/db/queries/squad.sql` | 151 | Squad SQL 查询 |

### 前端核心文件

| 文件 | 职责 |
|------|------|
| `packages/core/types/squad.ts` | Squad TypeScript 类型定义 |
| `packages/core/api/client.ts` | Squad API 客户端 |
| `packages/core/api/schemas.ts` | Squad Zod schemas |
| `packages/core/workspace/queries.ts` | React Query options |
| `packages/core/squads/stores/view-store.ts` | Zustand 视图状态 |
| `packages/views/squads/components/squads-page.tsx` | Squad 列表页 |
| `packages/views/squads/components/squad-detail-page.tsx` | Squad 详情页 |

### 测试文件

| 文件 | 覆盖场景 |
|------|---------|
| `server/internal/handler/squad_comment_trigger_test.go` | 评论触发场景 |
| `server/internal/handler/squad_assign_trigger_test.go` | 分配触发场景 |
| `server/internal/handler/squad_member_status_test.go` | 成员状态派生 |
| `server/internal/handler/squad_private_leader_test.go` | 私有 Leader 门控 |
| `server/internal/handler/squad_no_action_test.go` | No-action 去重 |

---

## 更新日志

| 日期 | 内容 |
|------|------|
| 2026-06-12 | 初始小队模式深度分析报告 |
