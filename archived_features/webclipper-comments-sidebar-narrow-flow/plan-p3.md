# Plan P3 - webclipper-comments-sidebar-narrow-flow

**Goal:** 补齐高风险测试与最终回归，确保窄屏三步流、app medium 默认关闭策略、locator 关联能力稳定，不引入回归。

**Non-goals:** 不新增产品能力；不改评论数据模型或同步边界。

**Approach:** 先补 route/state 测试，再补窄屏 comments 流 smoke，最后做边界与 locate 行为回归。将“清理残留 + compile/test/build 全量验收”作为独立任务，避免把验证混入功能任务导致漏项；所有窄屏 comments 入口断言都以详情顶部 header action 区为准。

**Acceptance:**
- `npm --prefix webclipper run compile` 通过。
- `npm --prefix webclipper run test` 通过（至少新增/受影响测试全部通过）。
- `npm --prefix webclipper run build` 通过。
- popup/app 窄屏 comments 三步流、app medium/wide 行为、locator locate 都有对应回归覆盖。
- popup/app 窄屏 comments 三步流、app medium/wide 行为、locator locate 都有对应回归覆盖，且窄屏入口只出现在详情顶部 action 区。

---

## P3-T1

**Title:** 新增窄屏三步 route 单测（state + escape）

**Files:**
- Add: `webclipper/tests/unit/use-narrow-list-detail-comments-route.test.ts`
- Modify: `webclipper/src/ui/shared/hooks/useNarrowListDetailCommentsRoute.ts`（按需）

**Step 1: 实现功能**

- 覆盖路由状态迁移：
  - `openDetail/openComments/returnToDetail/returnToList`
  - `listRestoreKey` 在 returnToList 时递增
- 覆盖 Escape 语义：
  - `comments -> detail`
  - `detail -> list`

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- --run tests/unit/use-narrow-list-detail-comments-route.test.ts`

Expected: 测试通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/unit/use-narrow-list-detail-comments-route.test.ts webclipper/src/ui/shared/hooks/useNarrowListDetailCommentsRoute.ts`

Run: `git commit -m "test: task1 - 覆盖窄屏三步route状态与escape"`

---

## P3-T2

**Title:** 新增窄屏 comments 流 smoke（list/detail/comments 往返）

**Files:**
- Add: `webclipper/tests/smoke/conversations-scene-narrow-comments-flow.test.ts`
- Modify: `webclipper/tests/smoke/conversations-scene-popup-escape.test.ts`（若复用）

**Step 1: 实现功能**

- 新增 smoke 覆盖：
  - list -> detail -> comments
  - comments close/collapse -> detail
  - detail back/Escape -> list
- 覆盖 pending-open 与 comments route 组合场景，确保不会直接落到错误路由。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- --run tests/smoke/conversations-scene-narrow-comments-flow.test.ts tests/smoke/conversations-scene-popup-escape.test.ts`

Expected: 目标 smoke 通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/conversations-scene-narrow-comments-flow.test.ts webclipper/tests/smoke/conversations-scene-popup-escape.test.ts`

Run: `git commit -m "test: task2 - 补齐窄屏comments三步流smoke"`

---

## P3-T3

**Title:** 边界回归测试（非 article 禁用、locator root 与 locate）

**Files:**
- Modify: `webclipper/tests/smoke/app-detail-header-actions.test.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-chrome.test.ts`
- Modify: `webclipper/tests/unit/threaded-comments-panel-locate.test.ts`（如受影响）

**Step 1: 实现功能**

- 补齐边界断言：
  - 非 article（或无 canonicalUrl）时不展示/不可触发 comments。
  - `onCommentsLocatorRootChange` 提供 root 后，sidebar quote click 的 locate 链路可工作。
  - app/popup comments step 中 locator env 与 root 获取路径一致。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- --run tests/smoke/app-detail-header-actions.test.ts tests/unit/article-comments-sidebar-chrome.test.ts tests/unit/threaded-comments-panel-locate.test.ts`

Expected: 目标测试通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/app-detail-header-actions.test.ts webclipper/tests/unit/article-comments-sidebar-chrome.test.ts webclipper/tests/unit/threaded-comments-panel-locate.test.ts`

Run: `git commit -m "test: task3 - 覆盖comments边界与locator回归"`

---

## P3-T4

**Title:** 清理残留路径并执行 compile/test/build 全量验收

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`（如有未清理分支）
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`（如有未清理分支）
- Modify: 其他受影响文件（按实际）

**Step 1: 实现功能**

- 清理 dead code、未使用 import、仅服务旧窄屏内嵌评论的残留分支。
- 确保行为收敛：
  - 窄屏：comments 仅通过第三步 route 展示。
  - app 非窄屏：comments 仅通过右侧 sidebar 展示。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: compile/test/build 全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx webclipper/src/ui/conversations/ConversationsScene.tsx`

Run: `git commit -m "chore: task4 - 清理comments窄屏残留并完成全量验收"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
