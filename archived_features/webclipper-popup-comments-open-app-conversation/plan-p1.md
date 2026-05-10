# Plan P1 - webclipper-popup-comments-open-app-conversation

**Goal:** 完成 detail header tools 的 chatwith 彻底清理，并把 popup 评论按钮改为跳转 app.html 定位 conversation（不自动展开 comments）。

**Non-goals:**
- 不实现 app 侧“loc 就绪后自动展开 comments”的时序闸门（留到 P2）。
- 不改 i18n 文案（除非编译/测试必须）。
- 不在本阶段做 `git commit`（除非你之后明确要求）。

**Approach:**
- 将 Chat-with 的“入口语义”限定在 comments sidebar（header + comment-level），并删除/更正任何把它描述为 detail header tools 的文档与契约暗示。
- popup 评论按钮通过 background 的 `resolveOrCaptureActiveTabArticle` 获取 `conversationId` 与 canonical url，用 `encodeConversationLoc` 构造 `loc`，再用 `openOrFocusExtensionAppTab({ route })` 打开 `app.html#/?loc=...`，并关闭 popup。

**Risks / Notes:**
- `openOrFocusExtensionAppTab()` 的 `route` 形状、以及 app 侧对 `loc` 的解析必须先核对实现（HashRouter 下 `?` 会出现在 `useLocation().search`，而不是 `window.location.search`）。
- `GET_ACTIVE_TAB_CAPTURE_STATE.kind === 'article'` 只能保证“应该可用”，不保证 resolve/capture 一定成功；需要定义点击后失败时的 UI 行为（不弹窗，不改 i18n）。

**Acceptance:**
- popup 点击评论按钮会打开/聚焦 `app.html` 并定位到对应 conversation，且 popup 关闭、不再打开 inpage panel。
- 会话详情页 header tools 不显示且不可注入 `Chat with ...`。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。
- 人工冒烟：Chrome dev 模式下打开任意 article 页面，点击 popup 评论按钮能稳定跳转 app 并选中对应会话。

---

## P1-T1 彻底清理 detail header tools 的 Chat-with 入口与契约

**Files:**
- Modify (likely): `AGENTS.md`（修正“Chat with 复用 tools 槽位”的描述）
- Modify (likely): `webclipper/AGENTS.md`（如存在相同描述/导航）
- Modify: `webclipper/src/ui/AGENTS.md`
- Modify (if needed): `webclipper/src/services/integrations/detail-header-actions.ts`
- Modify (if needed): `webclipper/src/viewmodels/conversations/conversations-context.tsx`
- Modify (recommended): `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`（若该文件实际只用于 comments sidebar，则考虑重命名/迁移以消除误导）
- Modify/Add: `webclipper/tests/**`（仅在需要锁死契约时）

**Step 1: 清理契约与文档**
- 扫描并定位所有把 Chat-with 描述为 detail header tools 的说明（至少覆盖：`AGENTS.md`、`webclipper/AGENTS.md`、`webclipper/src/ui/AGENTS.md`，以及 repo 内其他可能重复描述的 docs）。
- 建议扫描关键词（示例）：`rg -n "Chat with|chatwith|detail header.*tools|tools 槽位" -S`
- 将文档表述改为：
  - detail header tools 只包含“本地工具类动作（例如 cache-images）”与 open-in actions；
  - Chat-with 仅归属 comments sidebar（header + comment-level），不属于 detail header tools。
- 约束：不改 i18n（仅修正文档）。

**Step 2: 锁死代码路径（根因）**
- 确认 `resolveDetailHeaderActions()` 永远不引入 `resolveChatWithDetailHeaderActions()`。
- 若存在任何“未来可能把 chatwith 合并进 detailHeaderActions”的代码入口（或死代码分支），删除或改为明确不可达。
- 若发现 Chat-with actions 的实现文件命名/位置暗示“detail header”（例如 `chatwith-detail-header-actions.ts`）但实际用途是 comments sidebar：在本任务内完成重命名/迁移，避免后续误接回 detail header。

**Step 3: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`

Expected:
- 通过；且 detail header tools 不出现 Chat-with。

---

## P1-T2 popup 评论按钮改为打开 app.html 并定位到对应 article conversation

**Files:**
- Modify: `webclipper/src/ui/popup/PopupShell.tsx`
- Modify/Replace: `webclipper/src/viewmodels/popup/usePopupOpenInpageCommentsSidebar.ts`
- Modify (if needed): `webclipper/src/platform/webext/extension-app.ts`
- Reference: `webclipper/src/collectors/web/article-fetch.ts`（`resolveOrCaptureActiveTabArticle`）
 - Reference: `webclipper/src/services/shared/conversation-loc`（`encodeConversationLoc` / `decodeConversationLoc`）

**Step 1: 设计跳转协议（仅 Goal 2A）**
- popup 点击评论按钮：
  - 调 `ARTICLE_MESSAGE_TYPES.RESOLVE_OR_CAPTURE_ACTIVE_TAB`（背景 handler 已存在）拿到 `conversationId` 与 canonical url。
  - 构造 loc（必须使用现有 `encodeConversationLoc`），route 使用 `/?loc=<...>`。
  - 调 `openOrFocusExtensionAppTab({ route })` 打开/聚焦 app，并 `window.close()`。
- 保留 article-only gating（沿用现有 `GET_ACTIVE_TAB_CAPTURE_STATE.kind === 'article'`）。

**Step 2: 实现与兼容**
- popup 不再发送 `UI_MESSAGE_TYPES.OPEN_CURRENT_TAB_INPAGE_COMMENTS_PANEL`。
- 当 resolve/capture 失败（例如 background 抛错/返回空）：不弹窗、不改 i18n；建议：
  - 不关闭 popup；
  - 维持按钮可再次点击；
  - 使用现有 popup 内的非阻断提示能力（若无，则先在 plan 中记录为“无提示也可接受”，避免临时 invent 文案）。

**Step 3: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`

Expected:
- compile/test 通过；
- popup 评论按钮点击会打开/聚焦 `app.html` 并定位 conversation，并且 popup 关闭。

---

## P1-T3 为 popup→app 定位跳转补齐 smoke/unit 覆盖

**Files:**
- Add/Modify: `webclipper/tests/smoke/**`
- Add/Modify: `webclipper/tests/unit/**`

**Step 1: 测试点设计**
- 覆盖 popup hook/handler：
  - 当 capture state 为 article 且 background 返回 `conversationId`，应调用 `openOrFocusExtensionAppTab({ route: "/?loc=..." })` 并关闭 popup。
- 覆盖“不会打开 inpage panel”：断言没有发送 `OPEN_CURRENT_TAB_INPAGE_COMMENTS_PANEL`。
- 覆盖失败路径：resolve/capture 失败时不关闭 popup，且不触发 open-inpage。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test`

Expected:
- 新增/调整的测试通过。

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
