# 03 质量评估与风险清单

> 基于源码精读 + `yarn test`（19 文件 / 163 用例全绿）+ `yarn lint`（干净）+ CI 配置审查。
> 证据基线：`cordis@8cc9e33`。

## 1. 数据快照

| 维度 | 数值 |
|------|------|
| 源码规模 | 9 包，~4000 行 TS（core 约 2500 行） |
| 测试规模 | 19 个 spec 文件，163 用例，~4300 行 |
| 测试状态 | 全部通过（Node 26.7 本地；CI 跑 Node 24/26） |
| lint | 通过（`@cordisjs/eslint-config`） |
| 版本状态 | core `4.0.0-rc.8`，loader `1.0.0-rc.5`（README 明示 API 不稳定） |
| 提交结构 | 537 commits，作者 Shigma 占绝对多数 |

## 2. 设计优点

1. **深模块封装**：对外 API 只有 `ctx.plugin() / ctx.on() / ctx.provide()` 几个入口，复杂度全部藏在 Fiber 状态机与 Proxy 追踪里——用户代码几乎不需要理解内部机制
2. **错误处理极其彻底**：`composeError` 做长栈拼接（[utils.ts#L260](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/core/src/utils.ts#L260)）、`buildOuterStack` 在 effect 注册点捕获调用栈、所有异步 dispose 失败都被 `ctx.logger.error` 兜底、`await fiber` 会等待 inertia 收敛
3. **状态机严谨**：Fiber 6 态 + inertia 锁 + epoch 幂等，测试用 fake timers 精确验证交错场景（"inertia lock 1/2/3"）
4. **可观测性先行**：`getTraceable` 让服务方法自动绑定调用方上下文；`internal/*` 事件体系让 loader/include/hmr 无需侵入 core
5. **文档即测试**：艰深处（如 inertia 拒绝传播的注释 [fiber.ts#L189-L195](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/core/src/fiber.ts#L189-L195)）有高质量解释性注释

## 3. 风险与技术债

| # | 风险 | 证据 | 严重度 |
|---|------|------|--------|
| 1 | **强耦合 Node 内部 API**：ModuleLoader v1/v2、loadCache 语义差异 | [internal.ts](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/loader/src/internal.ts)、hmr 的 `Map.prototype.delete` 绕过 | 高 |
| 2 | **配置求值使用 `new Function + with + eval`** | [config/utils.ts#L3-L12](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/loader/src/config/utils.ts#L3-L12) | 中（注入面，无文档边界） |
| 3 | **代码密度过高**：`createTraceable` 百行 Proxy 嵌套，符号协议 ~20 个全局 `Symbol.for` | utils.ts、symbols 表 | 中（维护门槛） |
| 4 | 异步配置校验被显式禁止（StandardSchema async 路径抛 TypeError） | [fiber.ts#L32-L40](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/core/src/fiber.ts#L32-L40) | 中 |
| 5 | 未处理的 TODO/FIXME | fiber.ts `FIXME internal/fiber-info`、loader `FIXME merge config`、`TODO async validation` | 低 |
| 6 | 死代码小瑕疵：`_refresh()` 中 `let epoch = false; epoch = ''` | [fiber.ts#L386-L387](https://github.com/cordiverse/cordis/blob/8cc9e33/packages/core/src/fiber.ts#L386-L387) | 低 |
| 7 | `noImplicitAny: false` 放宽了类型严格度 | tsconfig.base.json | 低 |

## 4. 测试覆盖的盲区

- `composeError` 长栈拼接逻辑（handleError）**没有专项测试**——这是错误定位质量的关键路径
- include 的 `!js` 插值求值与 `interpolate` 无测试
- Node 22/23 的内部 loader 回退路径无 CI（矩阵只有 24/26）
- HMR 测试 800 行是最大单文件，但也说明复杂度集中；rollback 路径覆盖有限

## 5. 如果重做，最先改哪里

1. **抽出 Node 适配层**：把 `internal.ts` 的 ModuleLoader 访问与 HMR 的 loadCache 操作封装成独立"node-internals"适配包（含版本探测 + 单测 mock），避免核心逻辑与 Node 内部结构互相纠缠
2. **替换 `new Function` 求值**：`!js` 插值改为受限表达式求值器（或显式沙箱 + 文档声明信任边界）
3. **给 `createTraceable` 写行为规格**：当前语义靠 shadow.spec 反推，值得固化成文档化的不变量

## 6. 值得复用 vs 不值得

**值得**：
- Fiber 的 epoch + inertia 锁（幂等重载模式，任何"依赖驱动的服务生命周期"都适用）
- realm 符号隔离（比字符串作用域更安全的命名空间方案）
- `composeError` 长栈增强（异步框架的调试利器）
- internal/* 事件作为扩展点（core 零侵入扩展模式）

**不值得原样照搬**：
- Proxy 全量拦截 `get/set`（性能与调试代价高，仅在需要"调用方追踪"的框架核心使用）
- 直接操作 Node 内部模块加载器（属于版本敏感的实现细节，应隔离）
- `with + eval` 配置求值（安全边界模糊）
