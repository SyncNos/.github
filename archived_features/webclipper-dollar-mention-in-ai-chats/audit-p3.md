# Audit P3 - webclipper-dollar-mention-in-ai-chats

> 记录 P3 的审计发现与修复闭环（由 executing-plans 在 P3 完成后自动进入）。

## Scope

- Phase: `P3`（`P3-T1` ~ `P3-T5`）
- Baseline commits:
  - `P3-T1`: `1e4608cf`
  - `P3-T2`: `4b83c11c`
  - `P3-T3`: `1736ceed`
  - `P3-T4`: `97a2f2d3`
  - `P3-T5`: `5fb4089a`

## Read-only checks

1. NotionAI contenteditable 适配器与统一 adapter 抽象：
   - `rg -n "editor-notionai|contenteditable|editor-adapter|focus|selection" webclipper/src/services/integrations/item-mention/content`
2. 竞态与边界守护：
   - `rg -n "composition|invalidated|race|stale|focus" webclipper/src/services/integrations/item-mention/content`
3. 文档同步一致性：
   - `rg -n "\$|mention|ChatGPT|NotionAI|Tab|Enter|Esc|markdown" webclipper/AGENTS.md .github/deepwiki/modules/webclipper.md .github/deepwiki/business-context.md`
4. P3 验证链：
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test`
   - `npm --prefix webclipper run build`

## Findings

- ## 发现 F-01

  - 任务：`P3-T1 | P3-T2`
  - 严重级别：`Medium`
  - 状态：`Resolved`
  - 位置：`webclipper/src/services/integrations/item-mention/content/editor-notionai.ts:detectActiveEditor`
  - 摘要：NotionAI adapter 当前通过“遍历第一个可见 contenteditable leaf”选中编辑器，可能在存在多个 leaf 时插入到错误位置；同时缺少 NotionAI 上下文信号门控，潜在会在普通 Notion 页面误触发。
  - 风险：用户在 Notion AI 之外编辑页面时被意外弹窗/拦截键盘，或插入 Markdown 到错误 leaf；问题一旦发生属于强干扰且难定位。
  - 预期修复：优先选择 `document.activeElement` / selection 所在 leaf；增加 NotionAI 信号 gate（避免普通 Notion 页面启用）。
  - 验证：`npm --prefix webclipper run test -- tests/smoke/item-mention-notionai.test.ts tests/unit/item-mention-notionai-adapter.test.ts`
  - 解决证据：Commit `9473fe00`；上述测试通过。

- ## 发现 F-02

  - 任务：`P3-T3`
  - 严重级别：`Medium`
  - 状态：`Resolved`
  - 位置：`webclipper/src/services/integrations/item-mention/content/mention-controller.ts:pickHighlighted`
  - 摘要：pick 触发后到 background 返回 Markdown 期间，若用户继续输入导致 session/query/range 变化，插入可能会错误替换新 session 的 `$query` 片段（竞态）。
  - 风险：错误替换用户刚输入的文本，表现为“插入到了不该替换的位置”，属于数据破坏级体验问题。
  - 预期修复：pick 时捕获 session/editor 快照；await 后校验快照仍有效，否则放弃插入。
  - 验证：`npm --prefix webclipper run test -- tests/smoke/item-mention-chatgpt.test.ts tests/smoke/item-mention-notionai.test.ts`
  - 解决证据：Commit `9473fe00`；上述测试通过。
- Residual risks:
  - 宿主站点 DOM/编辑器结构改动仍可能导致 adapter 失效；当前预期修复落点是 `editor-chatgpt.ts` / `editor-notionai.ts`（保持其余状态机/候选窗单一真源不分叉）。
  - `vitest run` 存在少量 `act(...)` 警告与测试内 `stderr` 日志（未导致 fail），后续如要收敛噪音可在对应测试用例做 `act` 包裹或 mock。

## Decision

- Pass.
- Verification (2026-03-28):
  - `npm --prefix webclipper run compile` passed
  - `npm --prefix webclipper run test` passed (117 files / 467 tests)
  - `npm --prefix webclipper run build` passed
