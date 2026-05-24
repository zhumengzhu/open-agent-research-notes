# 15 · 后台、PTY、快照与 Worktree

> **核心问题：** 主 loop 之外的子系统做什么？何时需要读它们？

---

## 1. 为何单独成篇

这些模块 **不在** `runLoop` 核心路径上，但影响长时间任务、终端交互、可回滚编辑、并行 git 工作区。

---

## 2. Background

[`background/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/background)

- 长时间 shell / 任务与主 session **解耦**
- 状态可查、可 cancel
- 插件可通过 **`tool` hook** 暴露查询/取消接口（内核提供底层能力）

**边界：** 后台任务仍属 **同一 OpenCode 实例**；不是自动另开 agent 进程。

---

## 3. PTY 与 Shell

| 模块 | 用途 |
|------|------|
| [`pty/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/pty) | 伪终端，交互式程序 |
| [`shell/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/shell) | shell 环境 |
| [`tool/shell.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/tool/shell.ts) | bash 工具 |

Hook：**`shell.env`** — 插件可注入环境变量（见 [06](./06-hook-system-reference.md)）。

---

## 4. Snapshot

[`snapshot/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/snapshot)

- 文件系统快照 / revert
- bootstrap 里 `snapshot.init()`
- 会话级检查点，与 git commit 互补

---

## 5. Worktree

[`worktree/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/worktree)

- Git worktree 创建与管理
- 供并行分支开发；社区插件可在此基础上做「多 agent 各用一 worktree」编排

---

## 6. 其它相邻模块（速查）

| 目录 | 一句话 |
|------|--------|
| [`file/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/file) | 读写、ripgrep、watcher |
| [`format/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/format) | 保存后格式化 |
| [`git/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/git) | git 辅助 |
| [`reference/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/reference) | @引用 git/local |
| [`share/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/share) | session 分享 |
| [`control-plane/`](https://github.com/anomalyco/opencode/tree/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/control-plane) | 远程 workspace |

写 `chat.params` 类插件时 **可跳过** 本章；做 background / 多 worktree 编排时再读。

---

## 读完后应能回答

- [ ] background 与 runLoop 关系？
- [ ] worktree 在内核里提供什么原语？
- [ ] shell.env hook 触发点？

→ **下一篇：** [16 · 插件作者指南](./16-plugin-author-guide.md)
