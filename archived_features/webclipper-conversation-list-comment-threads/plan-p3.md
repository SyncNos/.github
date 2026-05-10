# Plan P3 - webclipper-conversation-list-comment-threads

**Goal:** 将 article 的根评论数同步到 Obsidian 笔记 YAML frontmatter（`comments_root_count`）。

**Non-goals:** 本 phase 不改变 Obsidian note 路径策略、不改 comments 正文渲染格式，只增加 frontmatter 字段与测试覆盖。

**Approach:** 在 `obsidian-markdown-writer.ts` 的 frontmatter 构建时（article 分支）基于传入的 `comments` 计算根评论数，写入 `comments_root_count`（number）。更新 smoke 测试确保该字段稳定存在且值正确。

**Acceptance:**
- article note frontmatter 包含 `comments_root_count: <n>`，且与本地根评论数一致。
- 相关 smoke tests 通过。

---

## P3-T1

**Title:** Obsidian frontmatter 写入 comments_root_count

**Files:**
- Modify: `webclipper/src/services/sync/obsidian/obsidian-markdown-writer.ts`
- Modify: `webclipper/src/services/comments/domain/comment-metrics.ts`（如需对外导出更合适的 API）

**Step 1: 实现功能**

- 在 `buildFullNoteMarkdown()` 的 article 分支 frontmatter 增加：
  - `comments_root_count: <number>`
  - 取值来源：`computeArticleCommentThreadCount(comments || [])`
- 字段写入策略：
  - article notes 始终写入该字段（即使为 0），便于 Dataview 与筛选稳定。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/sync/obsidian/obsidian-markdown-writer.ts webclipper/src/services/comments/domain/comment-metrics.ts`

Run: `git commit -m "feat: P3-T1 - Obsidian笔记写入根评论数frontmatter"`

---

## P3-T2

**Title:** 修正/补齐 Obsidian markdown writer 测试

**Files:**
- Modify: `webclipper/tests/smoke/obsidian-markdown-writer.test.ts`

**Step 1: 实现功能**

- 在 article 测试用例中断言：
  - markdown 包含 `comments_root_count: 1`
  - 当 comments 为空时包含 `comments_root_count: 0`（或按实现策略调整断言）

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/smoke/obsidian-markdown-writer.test.ts`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/obsidian-markdown-writer.test.ts`

Run: `git commit -m "test: P3-T2 - Obsidian writer覆盖根评论数frontmatter"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
