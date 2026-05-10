# Subagent Audit P2 (code-review)

## Findings summary

1 high-signal defect found.

## Issue: Backfill can be permanently marked complete before data is actually persisted

- **Severity:** High
- **File:line:**
  - `webclipper/src/services/bootstrap/content-controller.ts:472`
  - `webclipper/src/services/bootstrap/content-controller.ts:665`
- **Why it matters:**
  - `maybeRunBackfill()` 在 append 写入前就把 `state.completed = true`。
  - 如果后续 `saveSnapshot(...)` 失败，外层仅记录错误，但不会回滚 `completed`。
  - 后续 tick 会永久跳过 backfill，导致缺口历史无法再补齐。
- **Suggested fix:**
  - 仅在 backfill append 写入成功后将 `completed` 置为 `true`；
  - 或在写入失败路径显式回滚 `completed = false`。

## Suggested verification commands

```bash
cd /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper
npm run test -- tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts tests/unit/autosave-backfill-reconciler.test.ts
```

```bash
# add a regression test:
# first append write fails, second tick retries and succeeds
cd /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper
npm run test -- -t "retries backfill after transient append failure"
```
