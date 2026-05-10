# Plan P1 - webclipper-comments-auto-attach-selection

**Goal:** 建立“根评论输入框自动附加选区”的基础契约，并完整接通 inpage 场景。

**Non-goals:**
- 不接入 App/Popup（留到 P2）
- 不修改回复输入框行为
- 不新增设置开关

**Approach:**
先在 comments panel 的 React 层增加“根输入框交互事件 -> 选区载荷回调”契约，再把载荷下沉到 sidebar controller，复用现有 `quoteText + pending locator` 保存链路。随后在 inpage 端提供选区解析器，确保点击根输入框可覆盖引用、无选区可清空。

**Acceptance:**
- inpage 中点击根评论输入框时，有选区则覆盖引用，无选区则清空引用
- 回复输入框不触发自动附加
- `quoteText + locator` 在保存根评论时正确落库

---

## P1-T1

**Task:** Define composer selection contract

**Files:**
- Modify: `webclipper/src/ui/comments/types.ts`
- Modify: `webclipper/src/ui/comments/react/types.ts`
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`

**Step 1: 实现功能**

- 为根评论输入框新增“交互时请求选区附加”的回调契约（仅根输入框触发）。
- 契约落点：`ThreadedCommentsPanelHandlers` 与 `CommentSidebarHandlers` 同步新增 `onComposerSelectionRequest`（命名可微调，但两端必须一致）。
- 触发时机：根输入框 `pointerdown`（优先，避免 focus 抢占导致选区丢失）+ `focus`（键盘导航兜底）；同一次交互需要去重，避免重复附加。
- 明确边界：仅根输入框触发；回复输入框（`Reply…`）明确不触发。
- 在 panel bridge 与 session handlers 中打通该回调，保证不影响现有 `onSave/onReply/onDelete/onClose`。
- 文件改动分工（与 Files 一一对应）：
  - `webclipper/src/ui/comments/types.ts`：补充 panel api/setHandlers 相关类型。
  - `webclipper/src/ui/comments/react/types.ts`：补充 React handlers 契约类型。
  - `webclipper/src/ui/comments/panel.ts`：桥接回调透传。
  - `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`：在根输入框事件中触发回调并实现去重。
  - `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`：补充 sidebar handlers 契约。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过，新增契约无类型断裂。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/types.ts webclipper/src/ui/comments/react/types.ts webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`

Run: `git commit -m "feat: P1-T1 - 定义根评论输入框选区附加契约"`

---

## P1-T2

**Task:** Wire inpage selection resolver

**Files:**
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`
- Modify: `webclipper/src/services/comments/locator/index.ts`（如需新增 helper 则同目录）

**Step 1: 实现功能**

- 在 inpage 侧实现“读取当前页面选区 + 生成 locator（env=inpage）”解析器。
- 将根输入框交互回调接入 controller：每次交互执行原子更新（同步覆盖 `quoteText` + 根评论 pending locator）；无选区时两者同时清空。
- 规范化规则：
  - 选区文本为空：`quoteText = ''`，`pendingRootLocator = null`。
  - 选区文本非空但 locator 构建失败：保留 `quoteText`，`pendingRootLocator = null`（不阻塞保存）。
  - 解析异常兜底为空选区，不抛出到 UI。
- 保持双击打开与 context menu 打开行为不变。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Expected: inpage 评论侧栏可正常打开，根输入框交互后引用行为符合预期。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/sidebar/comment-sidebar-session.ts webclipper/src/services/comments/locator/index.ts`

Run: `git commit -m "feat: P1-T2 - 接通inpage根输入框自动附加选区"`

---

## P1-T3

**Task:** Add inpage auto-attach tests

**Files:**
- Modify: `webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`
- Add: `webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`（聚焦根输入框交互）
- Modify（如需）: `webclipper/tests/unit/threaded-comments-panel-focus-regression.test.ts`

**Step 1: 实现功能**

- 覆盖 inpage 根输入框点击：有选区覆盖、无选区清空。
- 覆盖“回复输入框不触发自动附加”的保护断言。
- 覆盖“pointerdown + focus 去重”断言，防止一次交互附加两次。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/article-comments-sidebar-controller.test.ts tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`

Expected: 新增断言全部通过且无回归。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts webclipper/tests/unit/article-comments-sidebar-controller.test.ts webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`

Run: `git commit -m "test: P1-T3 - 增加inpage自动附加选区测试"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
