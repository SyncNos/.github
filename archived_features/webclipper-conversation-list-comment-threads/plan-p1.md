# Plan P1 - webclipper-conversation-list-comment-threads

**Goal:** 在 popup/app 的会话列表行展示 web article 的根评论数（thread 数），并保证新增/删除评论后列表可自动刷新。

**Non-goals:** 本 phase 不做 Notion / Obsidian 的元数据写入（放到 P2/P3）；不引入 conversations 表 schema 迁移（方案 1：按页计算注入）。

**Approach:** 抽一个纯函数的 “根评论数” 计算 helper（用于 UI/Notion/Obsidian 复用），然后在 `getConversationListBootstrap/getConversationListPage` 的 storage-idb 层对“当前页 items”按 article canonicalUrl 读取 `article_comments` 并注入 `commentThreadCount` 字段。最后在 `ConversationListPane` 里渲染一个轻量 chip；并补齐 delete 评论后的 `CONVERSATIONS_CHANGED` 广播，确保列表刷新能覆盖 count 变化。

**Acceptance:**
- article 会话行可展示 `💬 <n>`（n 为根评论数），无根评论时不展示。
- 新增/删除根评论后，无需手动刷新，会话列表能在既有事件刷新节奏内更新显示。
- `npm --prefix webclipper run compile` 与相关 target tests 通过。

---

## P1-T1

**Title:** 新增 comments 根评论数计算 helper（含单测）

**Files:**
- Add: `webclipper/src/services/comments/domain/comment-metrics.ts`
- Add: `webclipper/tests/unit/comment-metrics.test.ts`

**Step 1: 实现功能**

- 新增 `computeArticleCommentThreadCount(comments)`：
  - 根评论判定：`parentId == null` 或 `parentId` 指向的父评论不存在时计为 root。
  - 以 `id` 去重（避免重复行导致 count 偏大）。
  - 对异常数据（id 非正数/非有限、parentId 非有限）做容错：尽量不 throw，输出稳定数字。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/unit/comment-metrics.test.ts`

Expected: 测试通过（覆盖：无评论/仅 root/root+reply/孤儿 reply 视为 root/重复 id 去重）。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/domain/comment-metrics.ts webclipper/tests/unit/comment-metrics.test.ts`

Run: `git commit -m "feat: P1-T1 - 新增comments根评论数计算"`

---

## P1-T2

**Title:** 删除评论后广播 CONVERSATIONS_CHANGED，触发列表刷新

**Files:**
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/services/comments/background/handlers.ts`

**Step 1: 实现功能**

- 在 `storage-idb` 增加“删除前读取 comment 元信息”的能力（至少拿到 `conversationId` / `canonicalUrl`）：
  - 让 background handler 在 delete 成功后能广播 `UI_EVENT_TYPES.CONVERSATIONS_CHANGED`，并尽可能携带 `conversationId`（便于 active detail 刷新）。
- 在 `DELETE_ARTICLE_COMMENT` handler：
  - delete 成功且能解析出 `conversationId` 时：broadcast `{ reason: 'articleCommentDeleted', conversationId }`。
  - 无 `conversationId` 时：仍然 broadcast（不带 id 也能触发列表 refresh），避免 UI 停留旧 count。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/data/storage-idb.ts webclipper/src/services/comments/background/handlers.ts`

Run: `git commit -m "fix: P1-T2 - 删除评论后广播列表刷新事件"`

---

## P1-T3

**Title:** 列表查询按页注入 commentThreadCount（仅 article）

**Files:**
- Modify: `webclipper/src/services/conversations/domain/models.ts`
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Modify: `webclipper/src/platform/idb/schema.ts`（仅在需要补齐 index/store 时）
- Modify: `webclipper/tests/storage/conversations-pagination.test.ts`

**Step 1: 实现功能**

- 在 `Conversation` 增加可选字段：`commentThreadCount?: number`（仅对 `sourceType=article` 有意义）。
- 在 `readConversationListPageItems()` 内对 page items 做按页注入：
  - 仅对 `sourceType === 'article'` 且 `url` 可 canonicalize 的 item 计算 count。
  - 注意：当前 `readConversationListPage()` 的 transaction 只包含 `conversations` store；本 task 需要把 transaction 扩展为 `['conversations', 'article_comments']`，并把 `article_comments` store 传入按页注入逻辑（避免为每个 item 额外 openDb/开新 transaction）。
  - 使用 `article_comments` 的 `by_canonicalUrl_createdAt` index 读取该 url 的 comments 列表（canonicalize 口径必须与 comments 存储一致：`canonicalizeArticleUrl`）。
  - 用 `computeArticleCommentThreadCount()` 计算根评论数，并注入到该 conversation item 上。
  - 性能边界：只对当前页 items 计算；不影响 summary/facets 的统计口径。
- 缓存策略：
  - 不引入新的全局缓存（避免 stale）；让列表刷新事件自然触发重算即可。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/storage/conversations-pagination.test.ts`

Expected: 新增用例覆盖：
- article 有 `root + reply` 时 `commentThreadCount === 1`
- “孤儿 reply（parent 缺失）”计为 root
- chat items 不应带 `commentThreadCount`（或保持 `undefined`）

**Step 3: 原子提交**

Run: `git add webclipper/src/services/conversations/domain/models.ts webclipper/src/services/conversations/data/storage-idb.ts webclipper/tests/storage/conversations-pagination.test.ts webclipper/src/platform/idb/schema.ts`

Run: `git commit -m "feat: P1-T3 - 会话列表按页注入根评论数"`

---

## P1-T4

**Title:** 会话列表行展示 💬 根评论数（含 UI 单测）

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/tests/unit/conversation-list-delete-inline-confirm.test.ts`（或新增独立测试文件）

**Step 1: 实现功能**

- 在 `ConversationListPane` 每条 row 的 meta 区增加一个轻量 chip：
  - 当 `commentThreadCount > 0` 时展示：`💬 {commentThreadCount}`
  - 不引入新的 i18n key（避免本次范围扩张）；保持 aria-label 可读即可。
  - 样式与现有 `sourceTag` chip 协调：小字号、border、与 `opacity` 保持一致。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/unit/conversation-list-delete-inline-confirm.test.ts`

Expected: 增补断言：当 mock item 含 `commentThreadCount: 3` 时，渲染结果包含 `💬` 与 `3`。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ConversationListPane.tsx webclipper/tests/unit/conversation-list-delete-inline-confirm.test.ts`

Run: `git commit -m "feat: P1-T4 - 列表行展示根评论数chip"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
