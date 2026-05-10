# Plan P1 - webclipper-comments-sidebar-narrow-flow

**Goal:** 在不重写 comments 面板的前提下，让 popup 与 app 窄屏都支持 `list → detail → comments` 三步流，并移除窄屏 detail 顶部内嵌评论。

**Non-goals:** 本 phase 不做 app 中屏断点与默认关闭策略（放到 P2）；不改 app 宽屏三栏 comments 行为。

**Approach:** 先抽共享的 comments sidebar runtime（session/controller/snapshot + locator root ref），再在 `ConversationsScene` 引入三步 route。窄屏详情统一切换为 `ConversationDetailPane` 内联 header（`onBack`），避免外层 `DetailNavigationHeader` 与内层 header 双轨并存；`comments` 入口固定放在详情顶部 header action 区。`comments` step 复用 `ArticleCommentsSection` sidebar 模式，close/collapse 返回 detail。

**Acceptance:**
- popup 与 app 窄屏均可从 detail 顶部 header action 区进入 comments step，并可创建/回复/删除评论。
- `ConversationDetailPane` 不再在窄屏渲染内嵌评论（`md:tw-hidden` 路径移除）。
- comments 面板 collapse/close 在窄屏语义下返回 detail，不会停留空页面，且入口不出现在正文区域。
- 受影响 smoke：`conversations-scene-popup-escape`、`popup-shell-header-actions`、`app-shell-narrow-header-actions` 通过。

---

## P1-T1

**Title:** 抽共享 comments sidebar runtime（session/controller/snapshot/rootRef）

**Files:**
- Add: `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**

- 新建 `useArticleCommentsSidebarRuntime`（置于 `viewmodels/comments`），统一封装：
  - `createCommentSidebarSession`
  - `createArticleCommentsSidebarController(createArticleCommentsSidebarAppAdapter())`
  - `useSyncExternalStore` snapshot
  - `commentsLocatorRootRef`
- hook 输出最小能力集合（controller/session/snapshot/rootRef setter/getter），不耦合布局策略（narrow/medium/wide 由调用方决定）。
- `AppShell` 使用该 runtime 替换当前内联 session/controller/ref 逻辑，宽屏行为保持不变。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: `tsc --noEmit` 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "refactor: task1 - 抽共享comments侧栏runtime"`

---

## P1-T2

**Title:** 实现窄屏三步导航容器（list/detail/comments）

**Files:**
- Add: `webclipper/src/ui/shared/hooks/useNarrowListDetailCommentsRoute.ts`

**Step 1: 实现功能**

- 新增三步 route hook（不修改旧 `useNarrowListDetailRoute`，避免影响既有用例）：
  - route: `list | detail | comments`
  - `openDetail`：`list -> detail`
  - `openComments`：`detail -> comments`（其他状态 no-op）
  - `returnToDetail`：`comments -> detail`
  - `returnToList`：`detail/comments -> list` 且 `listRestoreKey++`
- Escape 行为：
  - `comments` 下 Escape -> detail
  - `detail` 下 Escape -> list

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/shared/hooks/useNarrowListDetailCommentsRoute.ts`

Run: `git commit -m "feat: task2 - 新增窄屏三步导航hook"`

---

## P1-T3

**Title:** ConversationsScene 接入 comments step（open/close/escape/back）

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`（仅在必要时）

**Step 1: 实现功能**

- `ConversationsScene` 引入 comments route，并在窄屏分三段渲染：
  - `list` -> `ConversationListPane`
  - `detail` -> `ConversationDetailPane onBack={returnToList}`
  - `comments` -> `ArticleCommentsSection sidebarSession=...`
- `ConversationsScene` props 扩展（向后兼容）：
  - 接收 comments runtime（session/controller/snapshot/rootRef）
  - 接收 `commentChatWith` 相关 resolver（与 app 宽屏同源）
- detail -> comments 打开链路：
  - `onTriggerCommentsSidebar` 中先 `openComments()`
  - 再 `controller.open({ selectionText, locator, focusComposer: true, ensureContext: false, source: 'popup'|'app' })`
- comments close/collapse:
  - 通过 controller `onClose` 或 session close 信号触发 `returnToDetail()`
- `ConversationDetailPane` 暂不删除内嵌 comments（留到 P1-T6），本任务只保证可由外部注入 `onTriggerCommentsSidebar` 与 `onCommentsLocatorRootChange`。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ConversationsScene.tsx webclipper/src/ui/conversations/ConversationDetailPane.tsx webclipper/src/ui/conversations/ArticleCommentsSection.tsx`

Run: `git commit -m "feat: task3 - conversationsscene接入comments第三步"`

---

## P1-T4

**Title:** popup 接入 comments step，并切换为 detail 内联 header

**Files:**
- Modify: `webclipper/src/ui/popup/PopupShell.tsx`

**Step 1: 实现功能**

- popup 新建并持有 comments runtime（复用 P1-T1 hook）。
- 将 runtime 透传给 `ConversationsScene`，启用 comments step。
- popup 在 narrow/detail 时不再渲染外部 `DetailNavigationHeader`，避免与 `ConversationDetailPane` header 双轨冲突。
- `ConversationDetailPane` 的 comments 入口仍保留在顶部 header action 区，不放进正文内容。
- 保留 popup 列表态顶部操作区（Fetch/Settings）原语义。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/popup/PopupShell.tsx`

Run: `git commit -m "feat: task4 - popup接入comments三步流并切换内联header"`

---

## P1-T5

**Title:** app 窄屏接入 comments step，并切换为 detail 内联 header

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**

- app 窄屏路由下将 comments runtime 传入 `ConversationsScene`。
- app 在 narrow/detail 时移除外层 `DetailNavigationHeader`，交由 `ConversationDetailPane` 渲染统一 header。
- `ConversationDetailPane` 顶部 header action 区承载 comments 入口，不再在正文中渲染入口。
- 保障 app 窄屏 comments close -> detail，detail back/Escape -> list。
- 注意不影响 app 宽屏 comments sidebar 现有行为（包含默认自动打开逻辑）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "feat: task5 - app窄屏接入comments三步流并切换内联header"`

---

## P1-T6

**Title:** 移除 ConversationDetailPane 窄屏内嵌评论

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`

**Step 1: 实现功能**

- 删除窄屏内嵌 comments（`md:tw-hidden` + `ArticleCommentsSection embedded`）路径。
- 保持 comments 按钮可用，但仅作为 “打开 sidebar/comments step” 入口（由上层注入回调决定）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx`

Run: `git commit -m "refactor: task6 - 移除窄屏内嵌评论路径"`

---

## P1-T7

**Title:** 修正受影响 smoke 测试（popup/app 窄屏头部与 escape）

**Files:**
- Modify: `webclipper/tests/smoke/conversations-scene-popup-escape.test.ts`
- Modify: `webclipper/tests/smoke/popup-shell-header-actions.test.ts`
- Modify: `webclipper/tests/smoke/app-shell-narrow-header-actions.test.ts`

**Step 1: 实现功能**

- 更新 mock/断言以适配窄屏 header 由 `ConversationDetailPane` 内联承载后的结构变化。
- 覆盖 comments route 存在时 Escape 的层级回退（comments -> detail -> list）。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- --run tests/smoke/conversations-scene-popup-escape.test.ts tests/smoke/popup-shell-header-actions.test.ts tests/smoke/app-shell-narrow-header-actions.test.ts`

Expected: 目标 smoke 全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/conversations-scene-popup-escape.test.ts webclipper/tests/smoke/popup-shell-header-actions.test.ts webclipper/tests/smoke/app-shell-narrow-header-actions.test.ts`

Run: `git commit -m "test: task7 - 更新窄屏header与escape smoke覆盖"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
