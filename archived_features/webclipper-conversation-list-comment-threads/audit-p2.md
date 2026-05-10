# Audit P2 - webclipper-conversation-list-comment-threads

> 由 `executing-plans` 在 P2 全部 tasks 完成后自动进入。

## Findings

- [x] Notion `SyncNos-Web Articles` DB schema 包含 `Comment Threads`（Number）且 `ensureSchemaPatch` 覆盖。
- [x] Notion sync 的 article create/update 都能写入/更新 `Comment Threads`，仅该字段变化时也能触发 property update。
- [x] smoke 覆盖：article 场景 `createPageInDatabase` properties 含 `Comment Threads`。

## Fixes

- [x] 修复了审计发现的 number 属性比较边界（`null` 与 `0` 误判相等）：
  - `normalizePagePropertyValue()` 对 `prop.number == null` 显式返回空字符串，避免 `Number(null) === 0` 的误判。

## Verification

- Run: `npm --prefix webclipper run test -- tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`

Result:
- [x] target smoke passed
