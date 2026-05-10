# Audit P1 - webclipper-ai-chat-autosave-backfill

## Scope

- Phase: `plan-p1.md`
- Reviewed commits:
  - `ecdbe948` (P1-T1)
  - `ecd9ef95` (P1-T2)
  - `1cabadfa` (P1-T3)
  - `dc8275ff` (P1-T4)
  - `dc92d850` (P1-T5)
  - `98b95cd9` (P1-T6)
- Subagent evidence: `.audit/subagent-audit-p1.md`

## Findings

### A1 (high) - backfill success can swallow same-tick incremental deltas

- Location: `webclipper/src/services/bootstrap/content-controller.ts` (`handleTick`, `backfill.skipIncrementalSave`)
- Symptom:
  - backfill 写入成功后，`computeIncremental(snapshot)` 仍会执行并推进 baseline；
  - 但该 tick 直接 `return`，导致 incremental diff 未落库，存在丢增量风险。
- Status: fixed

## Fix

- Fix commit: `0dc83094`
- 调整 `handleTick`：当 backfill 成功且 incremental 也有变化时，不再丢弃 incremental；追加一轮 incremental append 写入，避免 same-tick 增量被 baseline 吞掉。
- 对 incremental 写入增加 key 过滤：若 key 已在 backfill 写入集合中，则本 tick 不重复提交该 key。
- 保持 `computeIncremental(snapshot)` 仍执行，用于刷新 session baseline。
- 补充 smoke 测试覆盖“同一 tick backfill + incremental”不丢增量。

## Verification

- `npm --prefix webclipper run test -- tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
