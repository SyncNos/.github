# Plan P2 - webclipper-comments-auto-attach-selection

**Goal:** 在 App/Popup 右侧评论侧栏接入同样的根输入框自动附加选区行为，并保持现有选区语义。

**Non-goals:**
- 不修改 inpage 已完成行为（仅复用契约）
- 不改窄屏路由策略
- 不引入额外设置项

**Approach:**
复用 P1 契约，在 `ArticleCommentsSection` / `ConversationsScene` / `AppShell` 链路中注入 App/Popup 选区解析器；解析器按现有语义只接受消息区域选区。完成后补齐 app-shell 与 narrow flow 回归测试。

**Acceptance:**
- App/Popup 场景下点击根评论输入框可覆盖引用；无选区清空
- 仅根输入框触发，回复输入框不触发
- 窄屏 list/detail/comments 切换行为不回归

---

## P2-T1

**Task:** Wire app popup selection resolver

**Files:**
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/popup/PopupShell.tsx`（如需透传 runtime/context）

**Step 1: 实现功能**

- 为 App/Popup 侧栏注入根输入框选区解析器。
- 解析器沿用现有消息区选区语义：仅接受 `messagesRoot` 内部选区（`anchor/focus` 都在 root 内）。
- 消息区边界外、跨容器选区、或无可用 root（例如窄屏 comments 路由）统一视为无选区并清空引用，不抛异常。
- AppShell/PopupShell 仅在需要补传 runtime/context 时改动；若现有链路已满足，保持最小改动。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: App/Popup 侧栏链路类型与渲染通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ArticleCommentsSection.tsx webclipper/src/ui/conversations/ConversationsScene.tsx webclipper/src/ui/app/AppShell.tsx webclipper/src/ui/popup/PopupShell.tsx`

Run: `git commit -m "feat: P2-T1 - 接通app与popup根输入框自动附加选区"`

---

## P2-T2

**Task:** Connect scene runtime hooks

**Files:**
- Modify: `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`

**Step 1: 实现功能**

- 把 UI 解析到的选区载荷规范化后下沉到 sidebar controller（统一为 `{ quoteText, locator }` 语义）。
- 确保根输入框交互每次覆盖 `quoteText + pending locator`，并在空选区时同步清空两者。
- 保存后仍按现有逻辑清空引用，避免引入脏状态或重复附加。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/comment-sidebar-session.test.ts tests/unit/article-comments-sidebar-controller.test.ts`

Expected: 会话状态机与 controller 行为断言通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`

Run: `git commit -m "feat: P2-T2 - 收敛侧栏运行时选区载荷更新链路"`

---

## P2-T3

**Task:** Add app popup flow tests

**Files:**
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`
- Modify: `webclipper/tests/smoke/conversations-scene-narrow-comments-flow.test.ts`
- Modify: `webclipper/tests/smoke/app-detail-header-actions.test.ts`

**Step 1: 实现功能**

- 补充 App/Popup 根输入框自动附加选区测试。
- 补充“无选区清空引用”“回复输入框不触发”回归断言。
- 补充“无 locator root 时不崩溃并按空选区处理”的断言。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/conversations-scene-narrow-comments-flow.test.ts tests/smoke/app-detail-header-actions.test.ts`

Expected: App/Popup 与 narrow flow 测试通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/app-shell-comments-sidebar.test.ts webclipper/tests/smoke/conversations-scene-narrow-comments-flow.test.ts webclipper/tests/smoke/app-detail-header-actions.test.ts`

Run: `git commit -m "test: P2-T3 - 覆盖app与popup自动附加选区流程"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
