# Audit P1 - ai-chat-outline-panel

> 由 `executing-plans` 在 P1 完成后进入本文件，记录问题 -> 修复 -> 回归验证。

## Checklist

- [x] UI 与示意稿一致（把手贴右侧、无边框/无阴影、hover 仅限把手、面板覆盖把手）
- [x] 仅 `sourceType === 'chat'` 显示
- [x] 点击条目平滑跳转到正确 user 消息
- [x] `npm --prefix webclipper run compile && npm --prefix webclipper run test` 通过

## Findings

- [medium] `ConversationDetailPane.tsx` 目录锚点 wrapper 同时带 `tw-absolute` 与 `tw-relative`，会导致定位不稳定（来自 subagent 审计）。

## Fixes

- 移除 header/hidden-header 两处目录锚点 wrapper 的 `tw-relative`，保留纯 absolute 锚定。
- 修复提交：`3636b09e862781d8eb8baaf95fe8b3df955d2b15`

## Verification

- `npm --prefix webclipper run compile && npm --prefix webclipper run test` ✅
