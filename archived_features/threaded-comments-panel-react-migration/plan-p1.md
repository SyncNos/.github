# Plan P1 - threaded-comments-panel-react-migration

**Goal:** 为“root创建后聚焦到对应Reply并滚动可见”建立可依赖的返回值契约与类型链路（`createdRootId`），为后续 React 迁移提供确定性 focus 目标。

**Non-goals:** 本 phase 不改 UI 渲染实现（暂不引入 React 面板），不改变现有 DOM 面板行为；仅做契约升级与类型/冒烟兜底。

**Approach:** 先在 comments panel handlers / sidebar session contract 上引入结构化 `{ ok: true }` 结果类型，保证 quote 清空语义不被破坏；再把 `addRoot` 的创建结果（至少 comment `id`）从 background → adapter → controller 一路透传出来，最终让 `onSave` 返回 `createdRootId`。UI 迁移与 focus/scroll 在 P2 完成。

**Acceptance:**
- `controller.onSave` 可返回 `{ ok: true, createdRootId: number }` 且不破坏 quote 清空语义。
- app adapter 与 inpage adapter 都能返回创建出的 comment（至少包含 `id`）。
- 最小验证：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。

---

## P1-T1

**Title:** 为 comment save 返回值引入结构化 ok 结果类型（UI + session contract）

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`
- Modify: `webclipper/src/ui/comments/types.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`（如需要）

**Step 1: 实现功能**

- 在 `comment-sidebar-contract.ts` 的 `CommentSidebarHandlers.onSave` 返回值类型中允许结构化结果：
  - 兼容：`void | boolean`
  - 新增：`{ ok: boolean; createdRootId?: number | null }`
- 在 `ui/comments/types.ts` 的 `ThreadedCommentsPanelApi.setHandlers().onSave` 同步升级类型（保持两端一致）。
- 建议同时引入一个“统一的保存结果类型别名”（两端各自定义同形状别名即可）：
  - 例如 `type CommentSaveResult = void | boolean | { ok: boolean; createdRootId?: number | null };`
  - 避免在多处重复手写 union，降低后续漂移风险
- 确认 `comment-sidebar-session.ts` 的 `isOkResult()` 能识别结构化 `{ ok: true }`（当前已有识别逻辑；如不满足则补齐，但不要改变既有语义）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: `tsc --noEmit` 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts webclipper/src/ui/comments/types.ts webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`

Run: `git commit -m "refactor: P1-T1 - 为comment保存结果引入结构化ok返回值类型"`

---

## P1-T2

**Title:** 升级 ArticleCommentsSidebarAdapter.addRoot 返回创建出的 comment（至少包含 id）

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-adapter.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`

**Step 1: 实现功能**

- 将 `ArticleCommentsSidebarAdapter.addRoot(...)` 的返回值从 `Promise<void | true>` 升级为：
  - `Promise<{ id: number }>`（或 `Promise<ArticleCommentDto>`，以现有 repo 返回 dto 为准）
- 在 app adapter 中：
  - 复用 `@services/comments/client/repo.addArticleComment()` 的返回值（它已经返回 `ArticleCommentDto`）
  - 将该 dto 映射为 `{ id }`（或直接透传 dto），用于后续 controller `createdRootId`
- 不更改 addReply/delete/list/migrate 的语义与错误处理。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/sidebar/article-comments-sidebar-adapter.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`

Run: `git commit -m "refactor: P1-T2 - addRoot返回创建comment的id用于后续聚焦"`

---

## P1-T3

**Title:** 升级 inpage adapter addRoot/addReply 返回值并保持消息协议兼容

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`

**Step 1: 实现功能**

- `createArticleCommentsSidebarInpageAdapter().addRoot(...)`：
  - 从 runtime `COMMENTS_MESSAGE_TYPES.ADD_ARTICLE_COMMENT` 的响应中读取 comment（background 已返回 comment）
  - 返回 `{ id: number }`（或 dto），与 app adapter 对齐
- 具体实现要写死“从响应 `res.data` 取 id”的规则（避免实现时误读 wrapper）：
  - 只有当 `res?.ok === true` 且 `Number(res.data?.id) > 0` 时返回 `{ id }`
  - 否则抛错（不要返回 `true` 让上层误以为有 createdRootId）
- `addReply(...)` 保持 `void` 返回即可（reply focus 目标是 thread rootId，UI 侧已知）。
- 错误处理保持现有语义：`res?.ok` 不为 true 时抛错，避免“看似成功但实际失败”。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`

Run: `git commit -m "refactor: P1-T3 - inpage addRoot返回创建comment的id"`

---

## P1-T4

**Title:** controller.onSave 返回 createdRootId 并保持 quote 清空语义

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`（如需要）

**Step 1: 实现功能**

- 在 `article-comments-sidebar-controller.ts` 的 `handlers.onSave` 中：
  - 调用 `adapter.addRoot(...)` 拿到 `{ id }`
  - 返回 `{ ok: true, createdRootId: id }`
- 确保 quote 清空语义仍由 session wrapper 触发：
  - `comment-sidebar-session.ts` 目前在 `onSave` 返回 `true` 或 `{ ok: true }` 时 `setQuoteText('')`
  - 不要在 controller 内额外清 quoteText（避免双写导致状态漂移）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`

Run: `git commit -m "refactor: P1-T4 - onSave返回createdRootId以支持后续聚焦"`

---

## P1-T5

**Title:** 补齐类型链路与最小冒烟：compile/test/build 不回退

**Files:**
- Modify: `webclipper/src/ui/comments/panel.ts`（如因类型变化需要最小适配）
- Modify: 任何因 P1-T1~T4 类型升级导致的编译报错文件（最小修复）

**Step 1: 实现功能**

- 修复所有 TS 类型链路（以最小改动为原则），确保：
  - legacy panel 继续能 `await handler(text)` 并工作
  - 不依赖新增 UI 能力
- 在进入最终验证前，做一次“调用点扫描”，确保没有遗漏的返回值假设（尤其是把返回值当 `boolean` 用的地方）：
  - `rg -n "onSave\\b" webclipper/src -S`
  - `rg -n "await\\s+.*onSave\\b|=\\s*.*onSave\\b" webclipper/src -S`

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 三条命令均通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/panel.ts`

Run: `git commit -m "chore: P1-T5 - 对齐类型链路并完成compile/test/build冒烟"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
