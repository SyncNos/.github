# Plan P1 - webclipper-comments-chat-with-ai

**Goal:** 在 root comment 上提供 comment 级 `Chat with AI` 按钮，并完成“复制 payload 到剪贴板”的最小闭环（不自动跳转），同时覆盖 app(including embedded) 与 inpage 两端。

**Non-goals:**
- 本 phase 不打开/聚焦 AI 平台 tab（Phase 2 做）
- 本 phase 不引入 tab group（Phase 4 做）

**Approach:**
- 新增 **comment 级** ChatWith 配置（与 header 的 `chatWith` 配置解耦），避免语义混淆：header 仍是文章级，comment 是 root comment 级。
- UI 不直接读取 storage / 不直接调用 tabs API：通过 `MountOptions.commentChatWith` 注入 resolver。
- payload 先用 v1：以现有 Chat with AI 的 `maxChars` 为限，至少包含 `commentText + articleTitle + canonicalUrl`，`quoteText` 在无值时允许降级为空（Phase 3 再完整精炼模板）。
- 验证不只跑 compile：P1 每个 task 都补最小单测/烟测，避免改动 comments shared renderer 后出现隐藏回归。

**Acceptance:**
- inpage 评论侧边栏与 app 的 `ArticleCommentsSection`（sidebar + embedded）中：root comment 可见 `Chat with AI`，reply 不可见。
- 点击 action 后 payload 被复制（成功提示 / 失败提示）。
- 通过：`npm --prefix webclipper run compile` + 相关新增测试。

---

## P1-T1 为 root comment 渲染 Chat with AI 入口

**Files:**
- Modify: `webclipper/src/ui/comments/types.ts`
- Modify: `webclipper/src/ui/comments/render.ts`
- Modify: `webclipper/src/ui/comments/panel.ts`
- (Maybe) Modify: `webclipper/src/ui/comments/shadow-styles.ts`
- Add: `webclipper/tests/unit/threaded-comments-panel-comment-chatwith.test.ts`

**Step 1: 实现功能**
- 在 `webclipper/src/ui/comments/types.ts` 中新增 comment 级配置（建议独立类型，而不是复用 `ThreadedCommentsPanelChatWithConfig`）：
  - `ThreadedCommentsPanelCommentChatWithConfig.resolveActions(rootComment, articleContext) => actions[]`
  - `MountOptions.commentChatWith?: ...`
- 在 `renderThreadedComments()` 中仅对 root comment 渲染 trigger（reply 不渲染）：
  - `actions.length === 1` 直接执行；`>1` 展示 menu。
  - menu 开关由 panel 级 click/escape 收口，避免每条 comment 绑定全局监听。
- 交互语义：
  - busy 时禁用；`commentText` 为空时禁用（避免空 payload）。
  - 不引入 `@platform/**` 依赖。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/threaded-comments-panel-comment-chatwith.test.ts tests/unit/threaded-comments-panel-delete-confirm.test.ts`

Expected: root comment 按钮渲染与交互通过；删除二次确认能力无回归。

**Step 3: 原子提交**
Run: `git add webclipper/src/ui/comments/types.ts webclipper/src/ui/comments/render.ts webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/shadow-styles.ts webclipper/tests/unit/threaded-comments-panel-comment-chatwith.test.ts`

Run: `git commit -m "feat: P1-T1 - root评论增加comment级Chat with入口"`

---

## P1-T2 接入 comment 级 Chat with actions 解析与复制

**Files:**
- Add: `webclipper/src/services/integrations/chatwith/chatwith-clipboard.ts`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`
- Add: `webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts`
- Add: `webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts`
- Modify: `webclipper/src/ui/comments/panel.ts`
- Add: `webclipper/tests/unit/chatwith-comment-actions.test.ts`

**Step 1: 实现功能**
- 抽取共享剪贴板写入逻辑到 `chatwith-clipboard.ts`（保持与现有行为一致：`navigator.clipboard` + `execCommand` fallback）。
- 新增 `buildChatWithCommentPayloadV1()`：
  - 输入：`quoteText?`, `commentText`, `articleTitle`, `canonicalUrl`
  - 输出：`string`（末尾保留 `\n`）
  - 截断：复用 `truncateForChatWith(maxChars)`（`maxChars` 来自 `loadChatWithSettings()`）
- 新增 `resolveChatWithCommentActions()`：
  - Phase 1 的 `onTrigger` 仅做“复制 + 成功提示”（不跳转，Phase 2 做）
  - 单平台 label 可复用 `resolveSingleEnabledChatWithActionLabel()`
- `mountThreadedCommentsPanel()` 透传 `commentChatWith`，并通过 panel notice 回显成功/失败。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/chatwith-comment-actions.test.ts`

Expected: 编译通过；comment action 在“空平台/单平台/多平台”下行为正确。

**Step 3: 原子提交**
Run: `git add webclipper/src/services/integrations/chatwith/chatwith-clipboard.ts webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts webclipper/src/ui/comments/panel.ts webclipper/tests/unit/chatwith-comment-actions.test.ts`

Run: `git commit -m "feat: P1-T2 - comment级chatwith支持复制payload"`

---

## P1-T3 app 与 inpage 两端接入 comment 级 Chat with

**Files:**
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-chrome.test.ts`
- (Optional) Modify: `webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`

**Step 1: 实现功能**
- app/popup（`ArticleCommentsSection`）：
  - sidebar 模式与 embedded 模式都接入 `commentChatWith`（不能只在 sidebar 生效）。
  - embedded 模式补齐 `articleTitle` 来源（由调用方传入；拿不到时降级但不抛异常）。
- inpage：
  - 在 `ensurePanel()` mount options 中注入 comment 级 resolver。
  - 复用 `RESOLVE_OR_CAPTURE_ACTIVE_TAB` 获取当前文章 `title/url`，不新增 message contract。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-chrome.test.ts tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Expected: app 与 inpage 两个入口都可见 comment 级 Chat with 入口，且不影响现有评论流程。

**Step 3: 原子提交**
Run: `git add webclipper/src/ui/conversations/ArticleCommentsSection.tsx webclipper/src/ui/conversations/ConversationDetailPane.tsx webclipper/src/ui/app/AppShell.tsx webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/tests/unit/article-comments-sidebar-chrome.test.ts webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Run: `git commit -m "feat: P1-T3 - app与inpage接入comment级chatwith复制"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（至少 `compile + 关键单测`）
