# Audit P2 - webclipper-ai-chat-autosave-backfill

## Scope

- Phase: `plan-p2.md`
- Reviewed commits:
  - `5d946535` (P2-T1)
  - `b222e16c` (P2-T2)
  - `ce3f8a21` (P2-T3)
  - `404267d7` (P2-T4)
- Subagent evidence: `.audit/subagent-audit-p2.md`

## Findings

### B1 (high) - backfill `completed` state may be set before write success

- Location: `webclipper/src/services/bootstrap/content-controller.ts`
- Symptom:
  - `maybeRunBackfill` 在返回 backfill payload 时已把 `state.completed = true`；
  - 若后续 append 保存失败，状态不会回滚，后续 tick 永久跳过 backfill。
- Status: fixed

## Fix

- Fix commit: `736d0ed7`
- 将 `state.completed = true` 的时机后移到 append 写入成功之后。
- append 失败时不再误标记完成，后续 tick 仍可按节流/签名条件继续重试。
- 补充 smoke 回归：首轮 append 失败后，下一 tick 能再次尝试并成功补齐。

## Verification

- `npm --prefix webclipper run test -- tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
