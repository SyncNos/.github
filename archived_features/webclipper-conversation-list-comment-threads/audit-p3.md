# Audit P3 - webclipper-conversation-list-comment-threads

> 由 `executing-plans` 在 P3 全部 tasks 完成后自动进入。

## Findings

- [x] article note frontmatter 稳定包含 `comments_root_count`，值与根评论数一致。
- [x] `obsidian-markdown-writer` smoke 测试覆盖该字段，并在 comments 为空时行为明确（当前策略：始终写入 `0`）。

## Fixes

- [x] 本轮无审计阻塞问题，无需额外修复。

## Verification

- Run: `npm --prefix webclipper run test -- tests/smoke/obsidian-markdown-writer.test.ts`

Result:
- [x] target smoke passed
