# Audit P1 - webclipper-dollar-mention-in-ai-chats

> 记录 P1 的审计发现与修复闭环（由 executing-plans 在 P1 完成后自动进入）。

## Scope

- Phase: `P1`（`P1-T1` ~ `P1-T5`）
- Baseline commits:
  - `a9e5fe0b` (P1-T1)
  - `14c22355` (P1-T2)
  - `b369c637` (P1-T3)
  - `8e6926b2` (P1-T4)
  - `ab8a2eab` (P1-T5)
  - `bd4b3640` (audit fix)

## Read-only checks

1. 协议契约与匹配模型：
   - `rg -n "mention-contract|Mention(Query|Candidate|SearchResult)|title|source|domain|lastCapturedAt" webclipper/src/services/integrations/item-mention`
2. 消息协议与背景注册：
   - `rg -n "SEARCH_MENTION_CANDIDATES|BUILD_MENTION_INSERT_TEXT|register.*mention" webclipper/src/platform/messaging/message-contracts.ts webclipper/src/entrypoints/background.ts webclipper/src/services/integrations/item-mention/background-handlers.ts`
3. 插入文本同源与 no-truncation：
   - `rg -n "formatConversationMarkdownForExternalOutput|truncateForChatWith" webclipper/src/services/integrations/item-mention webclipper/src/services/integrations/chatwith`
4. P1 验证链：
   - `npm --prefix webclipper run test -- tests/smoke/background-router-item-mention.test.ts tests/unit/item-mention-search.test.ts`
   - `npm --prefix webclipper run compile`

## Findings

1. 候选预取数量与 limit 约束不一致：
   - 问题：background handler 期望预取 200 条用于排序，但存储侧 limit 上限是 50，导致预取参数无意义且容易误导后续维护。
   - 修复：将预取固定为 50（与 contract 上限一致），由 scanLimit/timeLimit 控制成本边界。
   - Commit: `bd4b3640`

## Decision

- P1 审计通过：协议/路由/存储查询/插入文本真源已打通，关键验证已跑通（unit/storage/smoke + compile）。下一步进入 P2 实现 content 交互闭环。
