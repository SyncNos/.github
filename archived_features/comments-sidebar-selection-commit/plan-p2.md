# Plan P2 - comments-sidebar-selection-commit

**Goal:** 引入 quote 卡片右上角 `❌` 显式清空能力，并把“清空 quote”从隐式（空选区）改为显式按钮动作；同时保证清空后允许重新附加相同选区。

**Non-goals:**
- 不引入额外确认弹窗（一次点击清空）
- 不新增 i18n 字段（保持现有做法，必要 aria-label 使用硬编码英文）

**Approach:**
扩展 comments sidebar handlers：新增 “clear quote” 动作，由 controller 负责清空 session quote 并同时清空 pending locator。UI 在 quote 卡片右上角渲染 `❌`，点击调用 handler。为避免 “清空后重新选择同一文本” 被 signature dedupe 屏蔽，UI 侧需要在 clear 时重置本地 dedupe 状态。

**Acceptance:**
- quote 只会在 `❌` 点击时清空（空选区/取消选中不清空；保存评论成功也不应隐式清空 quote）
- 清空后再次选择同一文本仍可重新附加
- app + inpage 两条链路一致（共用 `ThreadedCommentsPanel`）

---

## P2-T1

**Task:** Add explicit clear-quote handler contract and controller implementation

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/ui/comments/types.ts`
- Modify: `webclipper/src/ui/comments/react/types.ts`
- Modify: `webclipper/tests/unit/comment-sidebar-session.test.ts`

**Step 1: 实现功能（含及时清理老代码）**

- 为 comments sidebar handlers 增加 `onComposerQuoteClearRequest`（命名以最终实现为准，要求语义明确且只做清空）。
- `comment-sidebar-session` 透传该 handler（保持既有“handler 包装”结构）。
- **移除隐式清空：** `comment-sidebar-session` 当前对 `onSave` 的“成功后自动 `setQuoteText('')`”包装逻辑需要在本 task 内移除（quote 清空改为显式按钮语义）。
- `article-comments-sidebar-controller` 安装 handler：
  - `session.setQuoteText('')`
  - 同步清空 pending locator（例如 `pendingRootLocator = null`）
- 同提交内删除或合并任何因新增 handler 导致的重复/旧分支（避免同时存在两条清空语义）。
- 更新/新增单测（同提交内完成）：
  - `webclipper/tests/unit/comment-sidebar-session.test.ts`：不再断言 “onSave 成功会清空 quote”，改为断言 quote 保持不变；并新增/补齐对 clear handler 的行为断言。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-sidebar-session.test.ts`

Expected: 类型检查通过、`comment-sidebar-session` 单测通过，handlers 在 app/inpage 注入链路无断裂。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts webclipper/src/services/comments/sidebar/comment-sidebar-session.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/ui/comments/types.ts webclipper/src/ui/comments/react/types.ts webclipper/tests/unit/comment-sidebar-session.test.ts`

Run: `git commit -m "feat: P2-T1 - 增加quote显式清空handler并移除onSave隐式清空"`

---

## P2-T2

**Task:** Add ❌ overlay button to clear quote; reset dedupe signature; tests

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`
- Modify: `webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

**Step 1: 实现功能（含及时清理老代码）**

- 在 quote 卡片容器右上角渲染 `❌` 按钮（仅 quote 非空时显示）。
- 交互与可访问性（必须）：
  - 使用 `<button type="button">`，确保键盘可聚焦，`Enter/Space` 可触发。
  - `aria-label` 使用硬编码英文（例如 `Clear quote`），避免引入新的 i18n key。
- 点击 `❌` 时：
  - 调用 `handlers.onComposerQuoteClearRequest?.()`
  - 重置本地 dedupe signature（确保清空后能重新附加同一 selection）
- 样式：quote 容器设为 `position: relative`，按钮为 `position: absolute; top/right`，避免改变原卡片布局。
- 清理无效 UI/事件残留：若 `ThreadedCommentsPanel` 内仍存在与“空选区抑制/隐式清空”相关的残留逻辑，应在此 task 内一并移除。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/smoke/app-shell-comments-sidebar.test.ts`

Expected: 单测覆盖 `❌` 清空与“清空后可重新附加同一选区”的回归通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/styles/inpage-comments-panel.css webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

Run: `git commit -m "feat: P2-T2 - 为quote添加❌清空按钮并补齐回归测试"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
