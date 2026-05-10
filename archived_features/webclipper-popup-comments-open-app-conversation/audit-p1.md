# Audit P1 - webclipper-popup-comments-open-app-conversation

## Scope

- P1-T1 / P1-T2 / P1-T3 的实现一致性、契约收敛、以及最小可验证性（compile/test/build + 关键 smoke）。

## Findings

### ✅ Goal 1（清理）已锁死

- `resolveDetailHeaderActions()` 仍然只解析 open-in actions，不包含任何 chatwith 注入点。
- 误导性命名已消除：`chatwith-detail-header-actions` 更名为 `chatwith-comments-header-actions`，并同步更新引用与单测，降低“误接回详情头 tools”的风险。
- 文档契约已对齐：`AGENTS.md`、`webclipper/src/ui/AGENTS.md` 不再宣称“详情头 tools 包含 Chat with”。

### ✅ Goal 2A（popup 跳转 app 定位会话）已完成

- popup 顶部评论按钮不再发送 `OPEN_CURRENT_TAB_INPAGE_COMMENTS_PANEL`。
- 新链路：popup -> background `RESOLVE_OR_CAPTURE_ACTIVE_TAB` -> `openOrFocusExtensionAppTab({ route: "/?loc=..." })`，成功后 popup 关闭（由 UI 层触发）。

### ✅ P1 回归修复：popup 详情页“评论”按钮不应进入 popup 内 comments route

- 用户反馈：在 popup 的会话详情页点击“评论”会进入 popup 自己的 comments route（看起来像“评论页面”），且不跳转 app。
- 修复：为 `ConversationsScene` 增加 `onOpenCommentsExternally` 覆盖口；popup 在详情页触发评论时优先走 `openOrFocusExtensionAppTab({ route: "/?loc=..." })` 并关闭 popup，而不是 `openComments()` + sidebar controller。

### ✅ 覆盖（P1-T3）

- 补充 smoke：`tests/smoke/popup-shell-header-actions.test.ts` 覆盖“评论按钮会打开 app 并带上正确的 `loc`，且不触发 inpage open”。
  - 补充 unit：`tests/unit/conversations-scene-popup-comments-external.test.ts` 锁死“popup 详情页评论按钮优先外跳，不进入 narrow comments route”。

## Follow-ups (Not in P1)

- tooltip/文案：当前按钮仍复用 `openCommentsSidebar*` i18n key，但行为已是“跳转 app 并定位会话”。若要完全语义一致，需要引入/复用新的文案 key（本阶段按约束不改 i18n）。
- Goal 2B：已取消（用户确认 P2 不做）。

## Verification

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test`
- `npm --prefix webclipper run build`
