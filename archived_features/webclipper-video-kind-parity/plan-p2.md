# Plan P2 - webclipper-video-kind-parity

**Goal:** 在 IndexedDB 层把 comments 从历史 `article_comments` store 迁移到统一 store，并保证升级/兼容/回滚路径清晰（为 video/chat 同级能力打底）。

**Non-goals:**
- 不做多端同步/云端存储
- 不在本 phase 做 Notion/Obsidian 的 comments 同步扩展（P3 处理）

**Approach:**
- DB_VERSION 升级（`webclipper/src/platform/idb/schema.ts`）新增 `comments` store（或 `conversation_comments`），并在 onupgradeneeded 中从 `article_comments` 迁移数据。
- storage-idb 切换到新 store；旧 store 保持只读 fallback（短期），避免升级过程或部分数据异常导致 comments 丢失。

**Acceptance:**
- 升级后历史 comments 均可读且 UI 行为不变。
- 新增 comments 写入新 store；旧 store 不再写入（或只写一份，避免双写漂移）。

**Rules:**
- 所有命令使用 `rtk ...`
- schema migration 不可吞错（除非明确可容忍）；必须有测试覆盖
- 一个 task 一个 commit

---

## P2-T1 Add new IDB comments store + migration from article_comments

**Files:**
- Modify: `webclipper/src/platform/idb/schema.ts`

**Step 1: 实现功能**
- `DB_VERSION` +1
- 新增 `ensureCommentsStore`：
  - store: `comments`（建议）
  - indexes（建议最小集）：
    - `by_target_createdAt`（例如 `['targetKey','createdAt']`）
    - `by_conversationId_createdAt`（例如 `['conversationId','createdAt']`）
- on upgrade：
  - 遍历 `article_comments`，写入 `comments`
  - 迁移后的 `targetKey` = `url:<canonicalUrl>`（迁移时必须 normalize/canonicalize；不要把原始 URL 直接当 key）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: 通过；并新增/更新至少一个 schema migration 测试覆盖（按仓库现有 migration test 模式）。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/platform/idb/schema.ts`

Run: `rtk git commit -m "feat: P2-T1 - 新增comments store并迁移旧article_comments"`

---

## P2-T2 Switch comments storage implementation to new store (keep compatibility)

**Files:**
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/services/comments/data/storage.ts`
- Modify: `webclipper/src/services/comments/background/handlers.ts`

**Step 1: 实现功能**
- storage-idb：
  - 新增 `listCommentsByTarget/addComment/deleteComment/attachOrphan...`
  - 读取优先新 store；如新 store 为空且旧 store 有数据，可 fallback（仅过渡期）
- background handlers：
  - 通用 handler 走新 store
  - 旧 `*ARTICLE*` wrapper 仍可转调通用 handler（但不再直接访问旧 store）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: comments 相关 unit/smoke 全绿。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/comments`

Run: `rtk git commit -m "refactor: P2-T2 - comments存储切换到新store并保留兼容读"`

---

## P2-T3 Remove/lock down legacy article_comments write paths

**Files:**
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/platform/idb/schema.ts`（如需要标记 legacy store 或补充约束）

**Step 1: 实现功能**
- 禁止再写入旧 `article_comments`：
  - 要么删除旧写入函数
  - 要么显式标记为 legacy 并在运行时抛错（仅在确认无调用点后）
- 保留读取/迁移工具函数（短期）用于 debug（可选）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run compile`

Expected: 编译通过，且无遗留调用点。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/comments/data/storage-idb.ts webclipper/src/platform/idb/schema.ts`

Run: `rtk git commit -m "chore: P2-T3 - 收敛legacy article_comments写入路径"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule (gate): **P2 全部 tasks 完成后，必须先做审计再进入 P3**（不允许跳过）。
- Tooling:
  - 若使用 `$executing-plans` 执行，将在 phase 结束后**自动**进入审计闭环（使用 `$plan-task-auditor` 的流程）。
  - 若未使用 `$executing-plans`，则在 P2 结束时**手动**运行 `$plan-task-auditor`，以本目录为审计目标，输出到 `audit-p2.md`。
- Audit flow (strict):
  1. 只读审查并记录发现到 `audit-p2.md`（此阶段不改代码）
  2. 修复发现项（按 High → Medium → Low）
  3. 运行并记录验证命令（至少 `rtk npm --prefix webclipper run test`；如涉及 schema/DB_VERSION，额外补 `rtk npm --prefix webclipper run compile`）
