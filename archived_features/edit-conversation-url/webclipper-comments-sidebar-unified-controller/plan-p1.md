# Plan P1 - webclipper-comments-sidebar-unified-controller

## Goal

把 **inpage 评论侧边栏** 与 **app（app.html）右侧评论侧边栏** 的“业务流程”收敛到同一个共享 controller，保证“只入口不同”：入口不同（repo vs messaging），但 open/save/reply/delete/refresh/quote 生命周期一致。

## Non-goals

- 不把 app 改为走 messaging（坚持方案2：app 直连 repo）
- 不修改 comments 存储 schema
- 不重写 `threaded-comments-panel` 的 UI/DOM 结构（允许最小接口补强）
- 不改 i18n

## Current State（as-is，作为 P1 基线）

- UI 面板：两端已复用 `mountThreadedCommentsPanel()`（`webclipper/src/services/comments/threaded-comments-panel.ts`）
- Session：两端可复用 `CommentSidebarSession`（`webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`）
- Quote 生命周期：`CommentSidebarSession.setHandlers()` 会在 `onSave()` **返回 `true`** 时清空 quote（root comment 成功后 quote 清空的基础能力已具备）
- 现状问题：inpage 与 app 仍各自维护 refresh/save/delete 的业务流程与数据通道细节，导致“入口不同 -> 行为漂移”反复出现

## Approach（target design）

- `threaded-comments-panel`：仅 UI + 调用 `handlers`，不承载“评论业务流程”
- `CommentSidebarSession`：唯一 UI 状态真源（quote/busy/comments/openRequested/isOpen/focusSignal）
- `ArticleCommentsSidebarController`：唯一业务流程真源（open/ensureContext/refresh/saveRoot/saveReply/delete）
- `ArticleCommentsSidebarAdapter`：唯一数据通道抽象
  - app：adapter 走 `@services/comments/client/repo`
  - inpage：adapter 走 `runtime.send`（background handlers）

### Handler 契约（强约束）

- `onSave(text)`：成功 **必须返回 `true`**；失败必须 throw 或返回 `false/void`（但不要把失败伪装成成功）
- 目的：只有 root comment 保存成功才会触发 session wrapper 清 quote；并且失败不会清空 composer

## Acceptance（P1）

- root comment 保存成功后 quote 清空（inpage + app 一致）
- reply 不携带 quote（inpage + app 一致）
- 保存/删除成功后列表刷新（inpage + app 一致）
- busy/openRequested/isOpen 语义稳定：右侧栏不抖动、不反复 mount/unmount、app.html 可交互

---

## P1-T1

### 任务：定义共享 controller + adapter 接口（services/comments/sidebar），并把“上下文 state”收敛到 controller

### 修改范围（预期文件）

- Add: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Add: `webclipper/src/services/comments/sidebar/article-comments-sidebar-adapter.ts`

### 实现步骤

1. 定义 `ArticleCommentsSidebarContext`（最小必要）：
   - `canonicalUrl: string`
   - `conversationId: number | null`
2. 定义 `ArticleCommentsSidebarAdapter`（最小必要）：
   - `list({ canonicalUrl }): Promise<CommentSidebarItem[]>`
   - `addRoot({ canonicalUrl, conversationId, quoteText, commentText }): Promise<void | true>`（成功返回 `true`，失败 throw）
   - `addReply({ canonicalUrl, conversationId, parentId, commentText }): Promise<void>`
   - `delete({ id }): Promise<void>`
   - `ensureContext?(input): Promise<ArticleCommentsSidebarContext>`（inpage 需要；app 可省略）
3. 实现 `ArticleCommentsSidebarController`（仅 orchestration，不做 UI mount）：
   - 持有：`session`、`adapter`、`activeContext`
   - `open({ selectionText, focusComposer, source, ensureContext })`：
     - `session.setQuoteText(selectionText)`
     - `session.requestOpen({ focusComposer, source })`
     - 可选 `ensureContext()`（用于 inpage capture/attach-orphan）
     - `refresh()`
   - `refresh()`：`session.setBusy(true)` -> `adapter.list()` -> `session.setComments()` -> `finally setBusy(false)`
   - `installHandlers()`：把 root/reply/delete 流程收敛到 handlers，并通过 `session.setHandlers()` 注入面板
     - root：成功必须 `return true`（触发 session 清 quote）+ `await refresh()`
     - reply/delete：`await refresh()`

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `refactor: task1 - 抽取文章评论侧边栏共享 controller 与 adapter 接口`

---

## P1-T2

### 任务：实现 app adapter（repo 直连）

### 修改范围（预期文件）

- Add: `webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`

### 实现步骤

1. adapter 内部调用 `@services/comments/client/repo`：
   - `listArticleCommentsByCanonicalUrl`
   - `addArticleComment`
   - `deleteArticleCommentById`
2. adapter 层做最小 shape 映射（repo item -> `CommentSidebarItem`）
3. root comment 成功时返回 `true`（用于触发 session 清 quote）

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `refactor: task2 - 新增 app 文章评论 sidebar adapter（repo）`

---

## P1-T3

### 任务：app 右侧边栏接入共享 controller（仅入口差异），并把 UI 组件降级为“纯挂载层”

### 修改范围（预期文件）

- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`

### 实现步骤

1. `AppShell`：
   - 创建/持有 `ArticleCommentsSidebarController`（注入 app adapter + session）
   - 把“评论按钮入口”统一改为 `controller.open({ selectionText, focusComposer: true, source: 'app' })`
   - 保留右侧栏渲染门禁：必须是 `openRequested || isOpen`（否则 request 会被消费/丢失，导致抖动或无法打开）
2. `ArticleCommentsSection`：
   - 只负责 mount panel + `sidebarSession.attachPanel()` + `cleanup`
   - 移除：repo 调用、refresh、busy/quote 清理的业务逻辑（都交给 controller+session）

### 验证

- `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts`

### 提交要求

- `refactor: task3 - app 右侧评论栏接入共享 controller（入口仅 repo adapter）`

---

## P1-T4

### 任务：实现 inpage adapter（messaging + ensureContext）

### 修改范围（预期文件）

- Add: `webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`

### 实现步骤

1. `list/add/delete` 走 `runtime.send`：
   - `COMMENTS_MESSAGE_TYPES.LIST_ARTICLE_COMMENTS`
   - `COMMENTS_MESSAGE_TYPES.ADD_ARTICLE_COMMENT`
   - `COMMENTS_MESSAGE_TYPES.DELETE_ARTICLE_COMMENT`
2. `ensureContext()` 走：
   - `ARTICLE_MESSAGE_TYPES.RESOLVE_OR_CAPTURE_ACTIVE_TAB`
   - `COMMENTS_MESSAGE_TYPES.ATTACH_ORPHAN_ARTICLE_COMMENTS`
3. root comment 成功时返回 `true`（用于触发 session 清 quote）

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `refactor: task4 - 新增 inpage 文章评论 sidebar adapter（messaging）`

---

## P1-T5

### 任务：inpage 入口接入共享 controller（仅入口差异），删除重复业务流程

### 修改范围（预期文件）

- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`

### 实现步骤

1. 保留 top-frame guard、selection fallback、tabId 透传等入口行为
2. 用 `ArticleCommentsSidebarController` 替换现有 `refreshCommentsList/resolveOrCaptureArticle/bindPanelHandlers`：
   - `createInpageCommentsPanelController()` 只负责：
     - 组织 `selectionText/focusComposer/ensureArticle/tabId`
     - 调用 `controller.open({ ..., ensureContext })`
3. 确保 content script 分层约束不破坏：
   - content 侧不 import repo、不引入 app UI

### 验证

- `npm --prefix webclipper run test -- tests/smoke/background-router-open-comments-sidebar.test.ts`

### 提交要求

- `refactor: task5 - inpage 评论栏接入共享 controller（入口仅 messaging adapter）`

---

## P1-T6

### 任务：新增 unit test 覆盖共享 controller 的核心流程（open/refresh/saveRoot/saveReply/delete）

### 修改范围（预期文件）

- Add: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`

### 实现步骤

1. mock session（或使用真实 `createCommentSidebarSession()` + mock panel）
2. mock adapter，覆盖：
   - `open()` 会 requestOpen + setQuoteText + refresh
   - `saveRoot()` 成功返回 `true` 并触发 quote 清空（通过 session wrapper）
   - `saveReply/delete` 会 refresh

### 验证

- `npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-controller.test.ts`

### 提交要求

- `test: task6 - 覆盖共享 comments sidebar controller 核心流程`

---

## P1-T7

### 任务：更新 targeted tests，覆盖“仅入口不同”的一致性回归点

### 修改范围（预期文件）

- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-chrome.test.ts`

### 实现步骤

1. app smoke：新增断言
   - root save 成功后 quote 清空（UI/DOM 或 session snapshot 维度）
   - 右侧栏不会因 `openRequested` 被消费而反复 mount/unmount
2. chrome unit：如 props/挂载方式变化，更新 mock 与断言

### 验证

- `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/unit/article-comments-sidebar-chrome.test.ts`

### 提交要求

- `test: task7 - comments sidebar 回归点对齐共享 controller`

---

## P1-T8

### 任务：验证链（compile + targeted tests）

### 验证

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/unit/comment-sidebar-session.test.ts tests/unit/article-comments-sidebar-chrome.test.ts tests/unit/article-comments-sidebar-controller.test.ts`

### 提交要求

- 如本 task 无代码改动可跳过提交；如需要记录验证结果，可用：
  - `chore: task8 - verify comments sidebar controller unification`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
