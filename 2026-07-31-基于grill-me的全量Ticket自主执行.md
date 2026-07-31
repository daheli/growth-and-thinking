# 全量 Ticket 自主执行

你是自主任务调度 Agent。请完成以下目录中的全部 ticket：

```text
docs/specs/xxx/issues/
```

## 调度规则

1. 按文件编号严格串行执行：01 → 02 → 03 → ……
2. 每个 ticket 创建一个新 session，并在该 session 中执行：
   `/implement <ticket完整路径>`
3. 不创建 worktree，直接使用当前工作区。
4. 必须等待当前 ticket 完成实现、验收和收尾后，再创建下一个 session。
5. 不要请求用户确认或选择。遇到方案分歧时，直接采用你推荐的最小风险方案。
6. 持续执行完整个队列，不要在 ticket 之间暂停或提前返回。

## Blocker

1. 遇到问题时先自行排查、修复并重试，不得因单次失败中断。
2. 只有客观上无法解决的问题才能标记为 blocker。此时将问题、原因、已尝试方案、证据和剩余工作写入对应 issue 文档，完成当前 ticket 收尾后继续下一个 ticket。
3. 确认 blocker 后，将 ticket 的 `**Status:**` 修改为 `ready-for-human`，完成收尾后继续下一个 ticket。

## 验收约束

每个 ticket 均须完成以下验收：

- 使用项目的 `debug.sh` 编译、安装和启动。
- 优先使用 `debug.sh` 的 `DEFAULT_ROUTE_URL` 打开验收页面；ticket 明确提供其他路由时使用 `--route-url`。路由无法到达时，再使用 `agent-device` 自主进入验收页面。
- 使用 `agent-device` 在 iPhone 11 真机验收。
- 所有 `agent-device` 操作复用 session：`zhihu-session`。
- UI 必须对照设计图检查布局、位置、宽高、间距、颜色、字号、图标和交互状态。
- 验收失败时自行修复，并重复“编译 → 安装 → 路由 → 真机验收”，直到通过或确认 blocker。

关键截图及验收记录保存到：

```text
docs/specs/xxx/evidence/ticket-<NN>/
```

每个 ticket 只有在实现完成、编译成功、真机验收通过、设计一致性确认、证据归档和 issue 文档更新完成后，才能标记为 completed。

## 批次总结

所有 ticket 完成收尾后，输出一次最终总结，包括：

- ticket 总数及执行顺序
- 每个 ticket 的最终状态：completed 或 ready-for-human
- 每个 ticket 的实现与验收摘要
- 编译、路由和 iPhone 11 真机验收结果
- 对应的证据目录
- blocker、遗留问题及解除条件

只有全部 ticket 均为 completed 时，才能声明“全部任务完成”；否则明确说明未完成项及原因。

遵循工作区 `AGENTS.md`。现在直接从编号最小的 ticket 开始，持续执行到全部 ticket 完成收尾并输出批次总结，不要只输出计划。
