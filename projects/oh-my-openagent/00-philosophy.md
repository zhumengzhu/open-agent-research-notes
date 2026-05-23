# 00 · 项目理念（30 秒消化）

> 用户已有 2 个月使用经验，此篇极简。

---

## 4 条核心信条

1. **Human Intervention is a Failure Signal** —— 类比无人驾驶：每一次人类接管方向盘都是系统失败的信号。
2. **Indistinguishable Code** —— AI 写的代码必须和资深工程师写的不可区分。
3. **Token Cost vs Productivity** —— 多花 token 换 10× 生产力是值得的。
4. **Predictable / Continuous / Delegatable** —— Agent = 编译器：markdown 进，可工作代码出。

## 这跟你学插件机制有什么关系？

OmO 的每个 hook、每个 tool guard、每个强制机制（hashline 哈希校验、todo 强制续推、prometheus-md-only），都是在为这 4 条服务。

→ **看到任何"奇怪的强制行为"，先用这 4 条解释一下，再继续往下挖代码。**

---

## 延伸

- 完整 manifesto：[`docs/manifesto.md`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/docs/manifesto.md)
- 入门：[`docs/guide/overview.md`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/docs/guide/overview.md)
- 编排细节：[`docs/guide/orchestration.md`](https://github.com/code-yeongyu/oh-my-openagent/blob/20d67be496155473f49aef3207bfe9d3737cbfa8/docs/guide/orchestration.md)

→ **下一篇：** [01 · OpenCode 插件协议本质](./01-opencode-plugin-protocol.md)
