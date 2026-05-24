# 03 · CLI 入口、main 与 REPL

> **锚点：** `main.tsx` · `setup.ts` · `replLauncher.tsx` · `screens/REPL.tsx` · Commander program

---

## 1. 启动分叉

```text
main.tsx (Commander argv)
  ├─ bridge worker 子命令（若启用 BRIDGE_MODE）
  ├─ -p / --print  → cli/print.ts runHeadless     [19]
  └─ 默认           → setup() → launchRepl → REPL
```

`main.tsx` ~4700 行：**所有 CLI flag 定义与 action 集中在此**，然后分支到 REPL 或 print。读内核前只需 skim flag 表 + action 入口，不必通读。

---

## 2. `setup.ts`（交互式初始化）

| 步骤 | 作用 |
|------|------|
| Node ≥18 | 运行时检查 |
| `setCwd` / `setProjectRoot` | git root、项目边界 |
| Worktree | 可选 `--worktree` [23] |
| Session id | `initSessionMemory` |
| Hooks | `hooksConfigSnapshot` capture [11] |
| LSP | `reinitializeLspServerManager`（非 bare） [16] |
| Plugins/MCP | 后台连接 [14][15] |

**bare mode（`--bare` / `CLAUDE_CODE_SIMPLE=1`）：** 跳过 hooks、LSP、plugin sync、CLAUDE.md、auto-memory、keychain OAuth 等；仅 `--system-prompt[-file]`、`--append-system-prompt[-file]`、`--add-dir` 显式提供上下文 [04]。

---

## 3. REPL 栈

```text
replLauncher.tsx → App → REPL (screens/REPL.tsx)
```

REPL **不实现** agent loop，只负责：

| 职责 | 关联 |
|------|------|
| Ink 输入/消息/Spinner | L2 UI |
| `handlePromptSubmit` | [12] → QueryEngine |
| `useLogMessages` ↔ `recordTranscript` | [08] |
| Permission 对话框 | [11] |
| `initReplBridge` | [18] |
| Swarm UI / teammate banner | [21] |
| Coordinator context | [21] |
| Proactive/KAIROS idle | [30] |

### 3.1 交互式启动顺序

```text
argv 解析 → workspace trust（print 跳过）
  → bootstrap state（cwd、session、remote）
  → teammate argv 恢复（agent-id/team-name…）[21]
  → setup()
  → swarm backend detect（若 teams 启用）
  → launchRepl
```

Teammate worker：`storedTeammateOpts` 存在时跳过 TeamCreate，直接 member loop（`main.tsx` ~1187+）。

---

## 4. 与 print 模式对比

| 维度 | REPL | print (`-p`) |
|------|------|--------------|
| 入口 | `launchRepl` | `runHeadless` [19] |
| UI | Ink ~389 components | 无 |
| Permission | React | flags / MCP prompt tool |
| 输出 | 终端渲染 | text/json/stream-json |
| Trust | 有 dialog | **跳过**（help 警告） |
| QueryEngine | 长生命周期 | 每 `ask()` 可新建 |
| Settings | `useSettingsChange` | `settingsChangeDetector` |

---

## 5. CLI Flag 速查（按主题）

### 5.1 会话与模型

| Flag | 作用 |
|------|------|
| `-p, --print` | Headless，走 print.ts |
| `--continue` | 恢复最近 session [08] |
| `--resume <id\|jsonl>` | 指定 session |
| `--fork-session` | 分支新 session |
| `--no-session-persistence` | 不落盘（仅 print） |
| `--model <alias\|id>` | 主循环模型 [27] |
| `--fallback-model` | 过载 fallback（仅 print） |
| `--effort low\|medium\|high\|max` | API effort [27] |
| `--agent <name>` | 指定 agent [20] |
| `--task-budget <tokens>` | API task_budget [07] |
| `--betas <...>` | Beta headers（API key 用户） |

### 5.2 工具与权限

| Flag | 作用 |
|------|------|
| `--allowed-tools` | Tool allowlist |
| `--disallowed-tools` | Tool denylist |
| `--tools` | 内置 tool 子集（`""` 禁用全部） |
| `--permission-mode` | default/plan/bypass/auto 等 [11] |
| `--permission-prompt-tool` | MCP 代问权限 [19] |
| `--mcp-config` | MCP JSON |
| `--strict-mcp-config` | 仅用 CLI MCP 配置 |

### 5.3 上下文与配置

| Flag | 作用 |
|------|------|
| `--system-prompt` / `--system-prompt-file` | 覆盖 default system |
| `--append-system-prompt` / `-file` | 追加 system [13] |
| `--settings <file\|json>` | 额外 settings [04] |
| `--add-dir` | 额外可访问目录 |
| `--bare` | 最小模式 |
| `--plugin-dir` | 临时加载插件 [15] |
| `--disable-slash-commands` | 禁用 skills |

### 5.4 集成与高级

| Flag | 作用 |
|------|------|
| `--ide` | 启动连 IDE [18] |
| `-w, --worktree [name]` | Session 级 worktree [23] |
| `--tmux` | worktree + tmux pane |
| `--sdk-url` | Remote control plane [22] |
| `--output-format stream-json` | SDK NDJSON [19] |
| `--replay-user-messages` | stream-json ack user |
| `--rewind-files <uuid>` | 按 snapshot 恢复文件 [08] |
| `--from-pr` | 链 PR resume |
| `--agent-teams` | 强制 multi-agent（ant 可见；外部需 env）[21] |
| `--proactive` | 主动模式 [30] |
| `--chrome` / `--no-chrome` | 浏览器集成 |

### 5.5 调试

| Flag | 作用 |
|------|------|
| `-d, --debug [filter]` | 分类 debug log |
| `--debug-file <path>` | 写文件（隐含 -d） |
| `--verbose` | 覆盖 config verbose |

---

## 6. main.tsx action 内关键分支（skim）

`.action(async (prompt, options) => { ... })` 内大致：

1. 解析 options → bootstrap flags
2. Remote / sdk-url → 可能 bypass REPL [22]
3. `--print` → `runHeadless`
4. 否则 trust → setup → `launchRepl`
5. 可选 positional `prompt` → 首条 user message 入队

Coordinator：`CLAUDE_CODE_COORDINATOR_MODE` 或 resume `matchSessionMode` [21]。

---

## 7. 自测

- [ ] main 三条路径：REPL / print / bridge worker？
- [ ] `--bare` 关掉了哪些子系统？
- [ ] setup 与 runHeadless 各负责什么？
- [ ] REPL 是否实现 query loop？
- [ ] print 为何跳过 trust dialog？

**关联：** [19 SDK/print](./19-sdk-headless-and-print-mode.md) · [04 Config](./04-config-and-settings.md) · [18 Bridge](./18-bridge-and-ide.md)
