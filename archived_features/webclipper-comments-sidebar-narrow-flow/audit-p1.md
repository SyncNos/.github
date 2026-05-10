# Audit P1 - webclipper-comments-sidebar-narrow-flow

> 由 `executing-plans` 在 P1 全部 tasks 完成后自动进入。

## Findings

- [x] 共享 runtime 已被 app 与 popup 复用。  
  - evidence: `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`; `webclipper/src/ui/popup/PopupShell.tsx`; `webclipper/src/ui/app/AppShell.tsx`
- [x] `ConversationsScene` 三步 route（`list -> detail -> comments`）已接入，且 state 迁移受控。  
  - evidence: `webclipper/src/ui/shared/hooks/useNarrowListDetailCommentsRoute.ts`; `webclipper/src/ui/conversations/ConversationsScene.tsx`
- [x] comments close/collapse 在窄屏语义下返回 detail。  
  - evidence: `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`（`subscribeSidebarClose`）；`webclipper/src/ui/conversations/ConversationsScene.tsx`（close 回调 `returnToDetail`）
- [x] `ConversationDetailPane` 窄屏内嵌 comments 路径已移除。  
  - evidence: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`（已无 `md:tw-hidden` 嵌入分支）
- [x] app/popup 已移除窄屏外层 `DetailNavigationHeader`，改为 detail 内联 header。  
  - evidence: `webclipper/src/ui/popup/PopupShell.tsx`; `webclipper/src/ui/app/AppShell.tsx`; `webclipper/src/ui/conversations/ConversationsScene.tsx`（`inlineNarrowDetailHeader`）
- [x] locator root 传递链路正常（detail -> runtime -> comments panel）。  
  - evidence: `ConversationDetailPane.onCommentsLocatorRootChange` -> `ConversationsScene` -> runtime `setLocatorRoot` -> `ArticleCommentsSection.getLocatorRoot`

## Findings notes

- Low: `P1-T7` 原计划文件列出会修改 `conversations-scene-popup-escape.test.ts`，最终无需改动；该测试已覆盖真实 `ConversationsScene` Escape 路由语义，保持通过。
- Medium (test strategy): `popup/app-shell` 两个 smoke 使用 mock `ConversationsScene`，适合验证 shell 层 header 行为；comments 三步真实往返集成覆盖建议在 P3 的窄屏 comments flow smoke 补齐（计划已包含）。

## Fixes

- [x] 无需新增代码修复；已根据审计结论执行 P1 验证命令并通过。

## Verification

- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- --run tests/smoke/conversations-scene-popup-escape.test.ts tests/smoke/popup-shell-header-actions.test.ts tests/smoke/app-shell-narrow-header-actions.test.ts`
- Run: `npm --prefix webclipper run test -- --run tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/app-detail-header-actions.test.ts`

Result:
- [x] compile passed
- [x] target smoke set passed（14 tests / 5 files）

## Notes

- 手工回归最小清单：
  - popup：`list -> detail -> comments`，comments close -> detail。
  - app narrow：`list -> detail -> comments`，Escape 层级回退正确。
  - article detail：comments 按钮可用且不会再出现窄屏内嵌 comments。
