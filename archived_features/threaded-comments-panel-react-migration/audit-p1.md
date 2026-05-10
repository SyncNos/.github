# Audit P1 - threaded-comments-panel-react-migration

> Phase 审计闭环：先记录发现，再修复问题，再跑本 phase 验证命令。

## Findings

### 任务看板（来自 `todo.toml`）

- `P1-T1` 为 comment save 返回值引入结构化 ok 结果类型
- `P1-T2` 升级 `ArticleCommentsSidebarAdapter.addRoot` 返回创建出的 comment id
- `P1-T3` 升级 inpage adapter addRoot 返回值并保持协议兼容
- `P1-T4` controller.onSave 返回 `createdRootId` 并保持 quote 清空语义
- `P1-T5` 类型链路冒烟（compile/test/build）

### 发现 F-01

- 任务：`P1-T5`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`webclipper/tests/unit/article-comments-sidebar-controller.test.ts:93`
- 摘要：`P1-T4` 将 `onSave` 返回值升级为结构化 `{ ok, createdRootId }` 后，controller 单测仍断言 `true`，导致 phase 回归验证失败。
- 风险：类型契约升级后测试未同步，`npm --prefix webclipper run test` 失败会阻断 phase 验证闭环。
- 预期修复：同步测试 mock 和断言，改为验证结构化返回值，并确保 quote 清空语义仍由 session wrapper 保持。
- 验证：`npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-controller.test.ts`
- 解决证据：`2452156d6186d36bdfccae2392d33cdff6db1733`

### 发现 F-02

- 任务：`P1-T2 | P1-T3 | P1-T4`
- 严重级别：`Low`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/.audit/subagent-audit-p1.md`
- 摘要：只读 subagent 审查未发现额外功能问题。
- 风险：无新增风险。
- 预期修复：无需修复。
- 验证：subagent 审查记录已落盘。
- 解决证据：`.github/features/threaded-comments-panel-react-migration/.audit/subagent-audit-p1.md`

## Fixes

- 已在 `P1-T5` 提交中修复 controller 单测，确保新的结构化保存结果契约与测试一致。
- subagent 审查结果已归档到 `.audit/subagent-audit-p1.md`，未发现额外功能回归。

## Verification

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。
