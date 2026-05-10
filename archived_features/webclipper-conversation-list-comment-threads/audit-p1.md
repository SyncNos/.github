# Audit P1 - webclipper-conversation-list-comment-threads

> 由 `executing-plans` 在 P1 全部 tasks 完成后自动进入。

## Findings

- [x] 会话列表按页注入 `commentThreadCount`，且仅对 article 生效。
- [x] UI 行内 chip 在 `commentThreadCount > 0` 时展示且样式与现有 meta 一致。
- [x] 删除评论后会触发 `UI_EVENT_TYPES.CONVERSATIONS_CHANGED`，列表无需手动刷新即可更新 count。
- [x] 单测覆盖：`computeArticleCommentThreadCount`、storage pagination 注入、`ConversationListPane` 渲染。
- [x] 低优先级观察：`getArticleCommentDeleteContextById` 当前使用 `getAll()` 扫描，功能正确；大数据量下可后续优化为定向查询。

## Fixes

- [x] 本轮无阻塞问题，无需额外修复。

## Verification

- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/comment-metrics.test.ts tests/storage/conversations-pagination.test.ts tests/unit/conversation-list-delete-inline-confirm.test.ts`

Result:
- [x] compile passed
- [x] target tests passed
