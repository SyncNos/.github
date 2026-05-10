# Plan P2 - webclipper-comments-selection-recovery

**Goal:** 把选区恢复能力接入 inpage 评论侧栏主链路，并完成关键回归覆盖。

**Non-goals:**
- 不扩展到 comments 之外模块
- 不修改已有 root/reply 触发语义
- 不引入 page-world 站点适配

**Approach:**
在 inpage comments content handler 中接入恢复解析器：优先原生选区，失败时按 host gate 走坐标反推。保持 `ThreadedCommentsPanel` 的自动触发策略不变，仅替换选区求值器。随后补齐 smoke + unit 回归，并显式覆盖 app/popup 不回归，再加轻量原因码与能力守卫辅助后续审计。

**Acceptance:**
- 白名单站点下，原生为空时可附加引用文本
- 非白名单站点行为保持不变
- reply 输入框行为、删除/保存/定位链路无回归
- app/popup 评论侧栏引用行为与既有版本一致（不因 inpage 兜底改动被破坏）

---

## P2-T1

**Task:** Wire inpage comments selection fallback

**Files:**
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/selection/selection-recovery-resolver.ts`

**Step 1: 实现功能**

- 在 inpage `resolveComposerSelection` 链路接入恢复解析器。
- 当 `trigger=auto` 时使用最近坐标轨迹；当 `trigger=button` 时允许复用同一解析器。
- 保持保存时 `quoteText + locator` 的既有约束：文本可用优先，locator 可空。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: inpage 侧类型与调用链编译通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/selection/selection-recovery-resolver.ts`

Run: `git commit -m "feat: P2-T1 - 接通inpage评论选区恢复兜底链路"`

---

## P2-T2

**Task:** Add inpage and app regressions

**Files:**
- Modify: `webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`
- Modify: `webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`
- Add: `webclipper/tests/unit/inpage-selection-recovery-fallback.test.ts`

**Step 1: 实现功能**

- 补充“原生选区为空 -> 走反推 -> 引用附加成功”场景。
- 补充“非白名单 -> 不触发反推”场景。
- 补充“reply 交互不污染引用”的回归断言。
- 补充 app/popup 侧 `selectionchange` 引用附加行为不回归断言。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/smoke/app-shell-comments-sidebar.test.ts tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/unit/inpage-selection-recovery-fallback.test.ts`

Expected: inpage 与 app/popup 的自动附加和恢复兜底场景均通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts webclipper/tests/smoke/app-shell-comments-sidebar.test.ts webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts webclipper/tests/unit/inpage-selection-recovery-fallback.test.ts`

Run: `git commit -m "test: P2-T2 - 增加inpage选区恢复回归测试"`

---

## P2-T3

**Task:** Add diagnostics and capability guards

**Files:**
- Modify: `webclipper/src/services/comments/selection/selection-recovery-resolver.ts`
- Modify: `webclipper/src/services/comments/selection/recovery-capabilities.ts`
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/tests/unit/inpage-selection-recovery-fallback.test.ts`

**Step 1: 实现功能**

- 增加轻量原因码（如 `native-empty`、`point-failed`、`outside-root`、`locator-null`）用于 debug 追踪。
- 增加能力守卫：明确 `caretRangeFromPoint` 与 `caretPositionFromPoint` 的可用性判定与优先级，API 不可用时快速降级。
- 增加保护：轨迹过期、坐标无效、根节点不匹配时快速降级为空。
- 保证日志默认低噪，不影响生产主流程。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/inpage-selection-recovery-fallback.test.ts`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: 原因码覆盖路径正确，保护逻辑不回归。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/selection/selection-recovery-resolver.ts webclipper/src/services/comments/selection/recovery-capabilities.ts webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/tests/unit/inpage-selection-recovery-fallback.test.ts`

Run: `git commit -m "chore: P2-T3 - 增加选区恢复诊断原因码与守卫"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
