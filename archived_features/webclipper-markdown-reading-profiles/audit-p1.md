# Audit P1 - webclipper-markdown-reading-profiles

> 记录 P1 的审计发现与修复闭环（由 executing-plans 在 P1 完成后自动进入）。

## Scope

- Phase: `P1`（`P1-T1` ~ `P1-T3`）
- Baseline commits:
  - `add28427`（P1-T1）
  - `18f3c736`（P1-T2）
  - `f8cd3b72`（P1-T3）

## Read-only checks

1. 协议与回退策略检查：
   - `rg -n "MarkdownReadingProfile(Id|Spec)|resolve.*ReadingProfile|unknown|medium" webclipper/src/services/protocols webclipper/src/ui/shared`
2. Medium 接入与硬编码收敛检查：
   - `rg -n "medium|reading-profile|ChatMessageBubble|presets" webclipper/src/ui/shared`
3. 可读性守护策略挂载检查：
   - `rg -n "overflow-wrap|max-w-|line-height|measure|blockquote|pre|table" webclipper/src/ui/shared webclipper/src/ui/styles`
4. P1 验证链：
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/markdown-reading-profiles-smoke.test.ts`

## Findings

1. Read-only checks 结果：未发现阻塞缺陷。
2. 关键风险与证据：
   - 脏值 profile 回退：`tests/unit/markdown-reading-profiles.test.ts`（`unknown -> medium`）覆盖。
   - reflow/长 token：`tests/unit/chat-message-bubble.test.ts` 与 `tests/smoke/markdown-reading-profiles-smoke.test.ts` 覆盖 `[overflow-wrap:anywhere]`、`[&_pre]:tw-overflow-auto`、`[&_table]:tw-overflow-x-auto`。
3. 审计命令结果：
   - `npm --prefix webclipper run compile` 通过。
   - `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/markdown-reading-profiles-smoke.test.ts` 通过。
4. 残余检查项（留给后续 phase 的最终 UX 验收）：
   - `320px` 实机手工重排抽查（非快照）在 P3 总验收执行。

## Decision

- Pass（可进入 P2）。
