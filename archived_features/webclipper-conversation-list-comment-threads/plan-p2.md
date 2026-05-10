# Plan P2 - webclipper-conversation-list-comment-threads

**Goal:** 将 article 的根评论数同步到 Notion Web Articles 数据库的 Number 属性（`Comment Threads`）。

**Non-goals:** 本 phase 不修改 Notion 页面 comments section 的渲染内容；不引入新的 Notion DB（只补齐 Web Articles schema）。

**Approach:** 在 `conversation-kinds.ts` 的 article kind dbSpec 增加 `Comment Threads`（number）并在 page properties 中写入 `commentThreadCount`。在 Notion sync orchestrator 的 article create/update 路径里：
- **在 buildCreateProperties/buildUpdateProperties 之前**读取本地 article comments 并计算根评论数（注意：`buildNotionCommentsBlocks().threads` 不是“根评论数”，它只统计带 quote 的 thread），再把结果注入到 `convo.commentThreadCount`。
- 尽量复用 sync 本来就需要读取的 comments，避免重复读取。
- 为避免每次 sync 都无条件触发 `updatePageProperties`，需要让 `pagePropertiesNeedUpdate` 能正确比较 `number` 属性（至少覆盖 `{ number: <n> }` vs page.properties 返回的 number 结构）。

**Acceptance:**
- Notion `SyncNos-Web Articles` DB schema 包含 `Comment Threads`（Number）。
- 同步 article 后页面属性值与本地根评论数一致；评论变化后二次同步会更新该值。

---

## P2-T1

**Title:** Notion Web Articles DB 增加 Comment Threads 数字属性

**Files:**
- Modify: `webclipper/src/services/protocols/conversation-kinds.ts`

**Step 1: 实现功能**

- 在 article kind `notion.dbSpec.properties` 与 `ensureSchemaPatch` 增加：
  - `Comment Threads: { number: {} }`
- 在 article kind `pageSpec.buildCreateProperties/buildUpdateProperties`：
  - 写入 `Comment Threads`，取值来源：`Number.isFinite(Number((conversation as any).commentThreadCount)) ? Number((conversation as any).commentThreadCount) : 0`

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/protocols/conversation-kinds.ts`

Run: `git commit -m "feat: P2-T1 - Notion文章库增加Comment Threads属性"`

---

## P2-T2

**Title:** Notion sync 写入 Comment Threads（create/update）

**Files:**
- Modify: `webclipper/src/services/sync/notion/notion-sync-orchestrator.ts`
- Modify: `webclipper/tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`（按需补齐覆盖）

**Step 1: 实现功能**

- 在 article sync 的 create/update 路径中（`pageSpec.buildCreateProperties/buildUpdateProperties` 调用之前）：
  - 读取本地 article comments 并计算根评论数（root count）：
    - count = `computeArticleCommentThreadCount(articleComments)`
    - 注入：`(convo as any).commentThreadCount = count`
  - 让 `pagePropertiesNeedUpdate()` 能正确比较 Notion `number` 属性，避免每次都因为 JSON stringify 结构差异而误判需要更新（至少补齐 `normalizePagePropertyValue()` 对 `prop.number` 的处理）。
- 测试策略（最小覆盖）：
  - 在 `notion-sync-orchestrator-kind-routing` 的 article 场景 stub storage 增加 `getArticleCommentsByConversationId` 返回含 1 个 root 的 comments；
  - 断言 `createPageInDatabase` 的 properties 包含 `Comment Threads` 且值为 `1`。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/sync/notion/notion-sync-orchestrator.ts webclipper/tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`

Run: `git commit -m "feat: P2-T2 - Notion同步写入文章根评论数"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
