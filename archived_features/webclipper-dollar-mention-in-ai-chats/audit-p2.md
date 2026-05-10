# Audit P2 - webclipper-dollar-mention-in-ai-chats

> 记录 P2 的审计发现与修复闭环（由 executing-plans 在 P2 完成后自动进入）。

## Scope

- Phase: `P2`（`P2-T1` ~ `P2-T5`）
- Baseline commits:
  - `7a1e7947` (P2-T1)
  - `2f2d8607` (P2-T2)
  - `487f9c3b` (P2-T3)
  - `bd62d8ca` (P2-T4)
  - `07263d99` (P2-T5)

## Read-only checks

1. 触发状态机与片段替换边界：
   - `rg -n "mention-session|triggerStart|triggerEnd|Esc|Tab|Enter|Arrow" webclipper/src/services/integrations/item-mention/content`
2. 候选窗渲染与键盘高亮：
   - `rg -n "item-mention-shadow|highlight|keyboard|aria|shadow" webclipper/src/ui/inpage webclipper/src/ui/styles`
3. ChatGPT textarea 适配器：
   - `rg -n "editor-chatgpt|textarea|replaceRange|selection" webclipper/src/services/integrations/item-mention/content`
4. P2 验证链：
   - `npm --prefix webclipper run test -- tests/smoke/item-mention-chatgpt.test.ts tests/unit/item-mention-session.test.ts tests/unit/item-mention-chatgpt-adapter.test.ts`
   - `npm --prefix webclipper run compile`

## Findings

- No blocking findings.
- Residual risks:
  - 候选窗定位当前基于 textarea rect（非 caret rect）；后续若需要更贴近光标位置，可在 adapter 层引入 caret 坐标计算。
  - 目前 click-outside 不会主动关闭；若产品希望“点空白关闭”，应在 controller 增加全局 pointerdown gate（不影响 Esc 契约）。

## Decision

- P2 审计通过：ChatGPT 端 `$` 触发、候选窗、键盘导航、替换插入、Esc 保留文本的行为链路与测试已落地；进入 P3 做 NotionAI `contenteditable` 适配与跨站点收口。
