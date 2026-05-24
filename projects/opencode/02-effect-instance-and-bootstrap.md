# 02 · Effect、项目实例与 Bootstrap

> **核心问题：** 打开两个项目目录时，OpenCode 如何隔离状态？服务按什么顺序启动？

---

## 1. 两个核心抽象

| 抽象 | 文件 | 作用 |
|------|------|------|
| **Instance** | [`project/instance-store.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/instance-store.ts) | 按 `directory` 缓存 runtime；HTTP 头 `x-opencode-directory` 选实例 |
| **InstanceState** | [`effect/instance-state.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/effect/instance-state.ts) | 每实例一份的可清理状态（ScopedCache） |

**边界：** 两个目录 = 两套 plugin 已加载列表、config merge 结果、LSP 进程 —— **不会串会话**。

---

## 2. Bootstrap 顺序（硬约束）

[`project/bootstrap.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/bootstrap.ts)：

```typescript
yield* config.get()
// Plugin can mutate config so it has to be initialized before anything else.
yield* plugin.init()
yield* Effect.forEach(
  [reference, lsp, shareNext, format, file, fileWatcher, vcs, snapshot, project],
  (s) => s.init(),
  { concurrency: "unbounded" },
)
```

```mermaid
flowchart LR
  C[config.get] --> P[plugin.init]
  P --> X[reference / lsp / file / vcs / ...]
```

### 为什么 plugin 必须最先

1. `hook.config` 可 **改写** agent、MCP、command、provider 列表  
2. 后续 `lsp.init` 等读的是 **已被插件修改后的 config**  
3. 大型插件可在 `server()` 内完成自有初始化，再在 `hook.config` 里注入 agents/tools

### init 的并发语义

`bootstrap` 对非 plugin 服务使用 **unbounded 并发**；且生产路径里各服务 `init()` 常被 `Effect.forkDetach` 包装（见 `packages/opencode/AGENTS.md`）—— **init 应快速返回**，重活放 InstanceState 闭包内。

---

## 3. Effect 服务模块形状

规范（[`packages/opencode/AGENTS.md`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/AGENTS.md)）：

```typescript
export class Service extends Context.Service<Service, Interface>()("@opencode/Foo") {}
export const layer = Layer.effect(Service, ...)
export * as Foo from "./foo"   // 或 index 用 export * as Foo from "."
```

**不要**用 `export namespace`（不利于 tree-shake 与 Node TS runner）。

**多文件目录**（如 `session/`）：**无** barrel `index.ts`，直接 `import { SessionPrompt } from "@/session/prompt"`，避免拖入整个目录。

---

## 4. `Instance.bind` 与原生回调

[`project/instance.ts`](https://github.com/anomalyco/opencode/blob/7fe7b9f258e36ad9f9acded20c5a9df201da19d5/packages/opencode/src/project/instance.ts) 提供 `Instance.bind(fn)`：

- 捕获当前 directory 的 ALS 上下文
- 用于 `@parcel/watcher`、`node-pty` 等 **同步** 回调里 `Bus.publish`

**不需要 bind：** `setTimeout`、`Promise.then`、Effect fiber 内部。

---

## 5. `makeRuntime` vs `InstanceState`

| API | 用途 |
|-----|------|
| `makeRuntime` | 全局/共享 Layer，dedupe memoMap |
| `InstanceState` | 每 directory 独立 + dispose 时清理 |

判断标准（AGENTS.md）：**两个目录不应共享一份状态 → 用 InstanceState**。

示例：插件 hooks 列表、LSP 子进程、文件 watcher 订阅。

---

## 6. 插件 init 在 OpenCode 眼里的两阶段

| 阶段 | OpenCode 行为 |
|------|----------------|
| `plugin.server(input)` | import 后执行工厂，返回 `Hooks` |
| `hook.config(cfg)` × N | 每个插件顺序改写内存 config |
| 之后 | 用户消息 → runLoop |

OpenCode **只感知** 上述接口；插件内部的 config 加载、hook 组装对内核不可见。

---

## 读完后应能回答

- [ ] 为何 plugin.init 在 lsp 之前？
- [ ] 两个项目同时 open 时什么会隔离？
- [ ] `Instance.bind` 解决什么问题？

→ **下一篇：** [03 · CLI、Server 与请求入口](./03-cli-server-and-entry.md)
