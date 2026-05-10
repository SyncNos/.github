# Plan P1 - webclipper-video-kind-parity

**Goal:** 让 `video transcript` 成为与 `web articles` / `AI chats` 同级的一等公民：comments sidebar、popup/inpage 入口与 canonical target 推导以 “conversation kind” 为真源，不再 hardcode `article only`。

**Non-goals:**
- 不做 UI 重设计
- 不引入新的 locator 类型（第一版允许 locator 为空；复用现有 TextQuote/TextPosition selector）
- 不在本 phase 做 IDB store 改名/新建（先稳定 API 与行为；P2 再做 schema 迁移）

**Approach:**
- 引入统一的 `ConversationKindId` 与 `CommentTargetKey` 概念：由 conversation 的 `sourceType/source/url/(source,conversationKey)` 推导稳定 key，并使用正确 canonicalizer（article/video/chat 各自不同）。
- Popup/Inpage/App 三入口统一用 kind-aware target 推导与 eligibility（不再 `article only`）。
- 在 services 层逐步提供通用 comments API；旧 `Article*` 命名与 message types 作为兼容 wrapper 存续（降低一次性改动风险）。

**Acceptance:**
- App 中 `article/video/chat` 三种 conversation 都能打开右侧 comments sidebar，并完成新增/回复/删除基本链路（至少通过现有 smoke/unit 测试覆盖）。
- Popup 的 inpage comments sidebar eligibility 不再限制为 `kind=article`（对 video/chat 页面可用，或给出明确的 unsupported 原因）。

**Rules:**
- 不要提交 `.github/features/webclipper-video-kind-parity/*`
- 所有命令使用 `rtk ...`
- 一个 task 一个 commit；commit message 必须含稳定 task id（例如 `feat: P1-T3 - ...`）
- 不允许在 UI 层追加 “再加一个 if” 的补丁式修法：必须落在统一的 kind/target + API 上

---

## Open Questions (resolve during P1-T1)

- Chat 的 `convo:<source>:<conversationKey>` 如何在 “未 capture 的 inpage” 场景获取？
  - A) 允许 inpage orphan 先落到 `url:`，capture 后再 attach/迁移到 `convo:`
  - B) inpage open 时强制触发一次 capture/resolve（更重，但 key 更稳定）
- Popup “打开当前页评论侧边栏”是否要在 capture state 为 unsupported 的页面也启用？
  - 若要支持 video 页面（当前 capture state 不含 video），则必须允许 unsupported 也能打开（以 orphan `url:` 工作）。

## P1-T1 Define conversation kind + comment target

**Files:**
- Modify: `webclipper/src/services/comments/domain/models.ts`
- Add: `webclipper/src/services/comments/domain/comment-target.ts`
- Modify: `webclipper/src/services/url-cleaning/http-url.ts`（如需要补齐 `normalizeHttpUrl` 使用点）
- Modify: `webclipper/src/services/url-cleaning/video-url.ts`（仅在需要复用/补齐 canonicalization 时）

**Step 1: 实现功能**
- 新增 `ConversationKindId`（建议字面量 union）：`'article' | 'video' | 'chat' | 'unknown'`
- 新增 `pickConversationKindIdFromConversation(conversation)`：单点推导 kind（供 UI/sync/sidebars 使用）
- 新增 `CommentTargetKey`（建议 string，而非 object；便于做 IDB index key）：
  - `url:<canonicalUrl>`：用于 article/video（以及 inpage orphan 的默认写入）
  - `convo:<source>:<conversationKey>`：用于 chat（conversation 已存在时优先，避免不同 thread 合并）
- 新增 `buildCommentTargetKeyFromConversation(conversation)`：
  - `source+conversationKey` 存在时：优先返回 `convo:` key（主要给 chat 用）
  - 否则 fallback 到 `url:` key：
    - article → `url:${canonicalizeArticleUrl(url)}`
    - video → `url:${canonicalizeVideoUrl(url)}`
    - chat → `url:${normalizeHttpUrl(url)}`（仅作为 fallback；不保证全站点唯一）
- 新增 `buildOrphanCommentTargetKeyFromLocation(location.href)`：用于 inpage 打开时在无 conversation 的前提下写 orphan（默认走 `url:`）。
- 将 `ArticleComment*` 类型重命名/抽象为 `Comment*`（允许暂时保留旧 export alias 以降低改动面）。

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run compile`

Expected: `tsc` 通过，无类型错误。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/comments/domain/models.ts webclipper/src/services/comments/domain/comment-target.ts`

Run: `rtk git commit -m "refactor: P1-T1 - 引入conversation kind与comment target"`

---

## P1-T2 Make popup/inpage gating kind-aware (include video/chat)

**Files:**
- Modify: `webclipper/src/viewmodels/popup/usePopupOpenAppCommentsConversation.ts`
- Modify: `webclipper/src/platform/messaging/ui-background-handlers.ts`
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`（如需透传更多 open payload 信息）

**Step 1: 实现功能**
- Popup eligibility（关键修正）：
  - **不能继续依赖 `GET_ACTIVE_TAB_CAPTURE_STATE.kind`**：当前 capture state 只覆盖 `article/chat/unsupported`，video 页面会被判定 unsupported，导致 popup 入口永远不可用。
  - 改为：对 http(s) 页面默认允许打开 inpage comments sidebar；由 inpage 侧使用 `buildOrphanCommentTargetKeyFromLocation(location.href)` 写 orphan comments。
- UI background open handler：
  - 打开 inpage panel 时把 source/kind 信息透传给 content handler（如果需要）
- Inpage content handler：
  - 移除 `ensureArticle:true` 的 hardcode（它会触发 article-only 的 resolve/capture 路径）
  - 允许 `ensureConversation=false`：即使无法 capture conversation，也要能打开面板并写 orphan comments（url target）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: `vitest` 全绿（允许既有的 stderr 噪音；但不能新增失败用例）。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/viewmodels/popup/usePopupOpenAppCommentsConversation.ts webclipper/src/platform/messaging/ui-background-handlers.ts webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/platform/messaging/message-contracts.ts`

Run: `rtk git commit -m "feat: P1-T2 - popup/inpage comments 闸门基于kind放宽到video/chat"`

---

## P1-T3 Refactor comments sidebar to use unified target

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-session.ts`
- Prefer: Add new generic controller/adapter and migrate callsites first; rename/cleanup is deferred to P3.
  - Add: `webclipper/src/services/comments/sidebar/comments-sidebar-adapter.ts`
  - Add: `webclipper/src/services/comments/sidebar/comments-sidebar-controller.ts`
  - Add: `webclipper/src/services/comments/sidebar/comments-sidebar-app-adapter.ts`
  - Add: `webclipper/src/services/comments/sidebar/comments-sidebar-inpage-adapter.ts`
  - Modify: `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`（后续 rename）
  - Keep legacy: `webclipper/src/services/comments/sidebar/article-comments-sidebar-*.ts`（先转调新实现）

**Step 1: 实现功能**
- 目标：sidebar controller 的 context 不再叫 `ArticleCommentsSidebarContext`，而是 `CommentsSidebarContext`，核心字段为 `commentTargetKey`（字符串）。
- adapter 的 `ensureContext` 不再是 `ensureArticle` 语义，而是 `ensureConversationForTarget?`（可选）：
  - inpage 场景：当无法/不需要确保 conversation 存在时，仍允许以 `url:` 作为 orphan target 写入 comments。
- runtime/hook：
  - 继续复用 `buildArticleCommentLocatorFromRange`（后续可以 rename，但不是必需的第一步）
  - 仅替换 “article-only” 的命名与 canonicalize 逻辑

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/comments/sidebar webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`

Run: `rtk git commit -m "refactor: P1-T3 - comments sidebar 适配统一target与adapter"`

---

## P1-T4 Enable app comments sidebar for article/video/chat

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`（后续可 rename 为 `CommentsSection`，P3 再做彻底清理）
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

**Step 1: 实现功能**
- AppShell/ConversationsScene：
  - gating 由 “isArticleConversationLike + canonicalizeArticleUrl” 改为 “buildCommentTargetKeyFromConversation”
  - 对 chat/video 也传递正确 `commentTargetKey` 到 sidebar context
- 测试：
  - 扩展 smoke 覆盖：至少新增 chat case + video case，确保 `data-can-trigger` 与打开行为正确

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: app shell comments sidebar smoke 全绿，且新增 case 覆盖 chat/video。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/ui webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

Run: `rtk git commit -m "feat: P1-T4 - app comments sidebar 同级支持article/video/chat"`

---

## P1-T5 Attach orphan comments on capture flows (article/video/chat)

**Files:**
- Modify: `webclipper/src/services/comments/data/storage.ts`
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/services/bootstrap/video-transcript-capture.ts`
- Modify: `webclipper/src/services/bootstrap/current-page-capture.ts`（或对应 capture/upsert 入口，按实际代码定位）
- Modify: `webclipper/src/services/conversations/background/handlers.ts`（如需要在 upsert 后统一 attach）

**Step 1: 实现功能**
- 统一 “attach orphan comments” 的语义：
  - 在 conversation 成功 upsert/capture 后，基于该 conversation 的 `CommentTargetKey`（至少包含 `url:`）执行一次 attach，把历史 orphan comments 绑定到 `conversationId`
  - 对 chat：当 `convo:` key 可用时，需定义是否将 `url:` 下的 orphan 迁移/合并到 `convo:`（至少保证不会丢）
- video transcript capture：
  - upsert conversation 成功后调用 attach
- chat/current page capture：
  - upsert conversation 成功后调用 attach

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: 增加/更新至少一个 unit/smoke 覆盖 “attach orphan after capture” 的行为。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/bootstrap webclipper/src/services/comments webclipper/src/services/conversations/background/handlers.ts`

Run: `rtk git commit -m "feat: P1-T5 - capture后自动attach orphan comments"`

---

## P1-T6 Fix video transcript export/render heading regression

**Files:**
- Modify: `webclipper/src/services/conversations/domain/markdown.ts`
- Add/Modify: `webclipper/tests/unit/conversation-markdown-video.test.ts`（或同级覆盖）

**Step 1: 实现功能**
- `sourceType=video` 的 markdown 导出必须是 “video 专用格式”：
  - 不能按 chat 的 `## <role>` 规则渲染 `role=transcript`（避免注入 `## transcript` / `### transcript`）
  - 输出应直接包含 transcript markdown 正文（以及必要的 meta lines）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: 新增/更新单测覆盖，确保导出不包含 `## transcript`。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/conversations/domain/markdown.ts webclipper/tests/unit/conversation-markdown-video.test.ts`

Run: `rtk git commit -m "fix: P1-T6 - video transcript 导出不注入transcript标题层级"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule (gate): **P1 全部 tasks 完成后，必须先做审计再进入 P2**（不允许跳过）。
- Tooling:
  - 若使用 `$executing-plans` 执行，将在 phase 结束后**自动**进入审计闭环（使用 `$plan-task-auditor` 的流程）。
  - 若未使用 `$executing-plans`，则在 P1 结束时**手动**运行 `$plan-task-auditor`，以本目录为审计目标，输出到 `audit-p1.md`。
- Audit flow (strict):
  1. 只读审查并记录发现到 `audit-p1.md`（此阶段不改代码）
  2. 修复发现项（按 High → Medium → Low）
  3. 运行并记录验证命令（至少 `rtk npm --prefix webclipper run compile` + `rtk npm --prefix webclipper run test`）
