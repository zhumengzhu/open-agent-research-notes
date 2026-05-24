# 09 · Skills、Curator 与学习闭环

> **锚点：** `skills/` · `tools/skills_tool.py` · `tools/skill_usage.py` · `tools/skill_provenance.py` · `agent/curator.py`

Skill = **按需加载的 procedure**；Memory = **事实/偏好**。本章带读 **发现 → 索引 → 加载 → 写入 → 园丁** 全链。

---

## 1. Skill 包结构

| 来源 | 路径 | 进 git |
|------|------|--------|
| Bundled | `skills/<cat>/<name>/SKILL.md` | ✅ |
| Optional | `optional-skills/` | ✅ index；需 `hermes skills install` |
| 用户/Agent | `~/.hermes/skills/` | 用户数据 |
| 插件 | `~/.hermes/plugins/<p>/skills/` | 插件包 |

**Frontmatter：** `name`, `description`, `platforms`, `setup`（env）、`prerequisites`…

**System 只见索引** — 全文经 `skill_view` / `skills_list` + `skill` tool 按需加载 [13](./13-prompt-assembly-and-cache.md)。

---

## 2. 发现与 readiness

`tools/skills_tool.py`：

- 解析 frontmatter · `skill_matches_platform(platform)`  
- **Setup 流程** — CLI vs Gateway 不同提示；secret 持久化策略  
- `SkillReadinessStatus` — 缺 env：**可读 SKILL.md 但 tool 标记不可用**

```text
AIAgent.__init__
  → 扫描目录 + optional index
  → build_system_prompt_parts → skills 索引进 stable tier
model 调用 skill_view / skill_manage / skill tool
  → 读 body + references/ + 声明的 MCP/脚本
```

**Mid-turn 政策：** install/uninstall 默认 **deferred** 下一 session（换索引 = 换 stable prefix）；`--now` opt-in invalidate [13 §8](./13-prompt-assembly-and-cache.md#8-mid-turn-禁止-vs-允许)。

---

## 3. skill_usage 与 provenance

**`.skill_usage.json`（`tools/skill_usage.py`）：**

| 字段 | 用途 |
|------|------|
| `last_used_at` | curator stale 判定 |
| `created_by` / agent-created 标记 | curator **仅**管辖 agent 创建 |
| `pinned` | 跳过 archive/自动迁移 |

**`skill_provenance.py`：**

- `set_current_write_origin` — conversation_loop turn 开头设 foreground；review fork 独立线程 [05](./05-aiagent-and-conversation-loop.md)  
- `BACKGROUND_REVIEW` — 仅 review fork 可 autonomous 改 agent-created skills [18](./18-multi-agent-panorama.md)  
- 用户手工 skill → **默认不归** curator 自动删  

---

## 4. Curator（`agent/curator.py`）

### 4.1 定位

模块头（1–20 行）：**idle 触发** 的 auxiliary fork — **非 cron**；维护 agent-created skills 生命周期。

```text
maybe_run_curator(idle_for_seconds=...)
  → should_run_now() ?
  → idle >= min_idle_hours ?
  → run_curator_review() → fork AIAgent → skill_manage
```

### 4.2 `should_run_now` 门控（199–248 行）

| Gate | 条件 |
|------|------|
| enabled | `curator.enabled` |
| paused | `.curator_state` |
| interval | `last_run_at` 距今 > `interval_hours`（默认 **7d**） |
| **首跑** | 无 `last_run_at` → **seed 为 now，不立即跑** — 防 `hermes update` 后首轮误归档 |

用户可 **`hermes curator run --dry-run`** 绕过 interval。

### 4.3 硬不变量

- 只动 **agent-created**（`skill_usage.is_agent_created`）  
- **Never auto-delete** — 仅 **archive**（`hermes curator restore`）  
- **Pinned** 跳过一切自动状态迁移  
- **Auxiliary client** — 不碰主 session prompt cache  

**默认阈值：** `DEFAULT_STALE_AFTER_DAYS = 30` · `DEFAULT_ARCHIVE_AFTER_DAYS = 90` · `DEFAULT_MIN_IDLE_HOURS = 2`

状态文件：`get_hermes_home()/skills/.curator_state`。

---

## 5. 学习闭环（谁写谁读）

```text
session_search（FTS）[08]
  + memory tool / provider sync
  + skill_manage（foreground 或 background_review）
  → curator archive/consolidate（idle 7d 级）
  → 下 session：system 索引 + prefetch 召回
```

| 写入者 | memory | skill |
|--------|--------|-------|
| 模型 foreground | memory tool | skill_manage |
| background_review | ✅ fork | ✅ fork [18](./18-multi-agent-panorama.md) |
| curator | — | archive/patch/consolidate |
| delegate 子 agent | ❌ block | 可 create（provenance） |

---

## 6. 与 Memory 协作（原 33 篇）

### 6.1 决策树

- 用户偏好/事实 → **memory**  
- 可复用 workflow / pitfall → **skill**  
- 跨 session 语义 → **session_search** 或 provider prefetch  

### 6.2 Cache / compression

- Memory **system 快照** turn 内不刷新；`memory` tool 仍 live [13](./13-prompt-assembly-and-cache.md)  
- Skill 索引 mid-turn 不 reload；`skill_manage` 写盘成功但 index 可能下 session 才进 system  
- `on_pre_compress` — provider 抽取将丢 turn [08](./08-session-and-memory.md)  
- Compaction 不删 skills 文件；可能摘要对话中曾加载的正文 [14](./14-context-compression.md)  

---

## 7. 添加 Skill Checklist

1. `skills/<cat>/<name>/SKILL.md` + frontmatter  
2. 大依赖 → `optional-skills/` + index  
3. env → `setup` + `OPTIONAL_ENV_VARS` in config  
4. 测试：`tests/tools/test_skills*.py`  
5. 网站文档：`website/scripts/generate-skill-docs.py` — **改 SKILL.md，不改生成页**

---

## 8. 源码带读

1. `skills_tool.py` — platform filter + setup  
2. `skill_usage.py` — agent-created 判定  
3. `curator.should_run_now` + `maybe_run_curator`  
4. `background_review` spawn 条件 vs curator [05 §6](./05-aiagent-and-conversation-loop.md#6-post-turn-与-return-dict)  

---

## 9. 自测

- [ ] bundled / optional / user 路径？  
- [ ] 为何 system 只有索引？  
- [ ] curator 首跑为何 seed 不立即 archive？  
- [ ] review fork 与 curator 触发差？  
- [ ] pinned skill curator 能否动？  
- [ ] delegate 为何 block memory 不 block skill create？  

**关联：** [08 Memory](./08-session-and-memory.md) · [06 Tools](./06-tools-registry-and-model-tools.md) · [18 Multi-agent](./18-multi-agent-panorama.md) · [33 redirect](./33-skills-and-memory-interaction.md)
