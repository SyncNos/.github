# Audit P1

## Scope

- Tasks: `P1-T1`, `P1-T2`, `P1-T3`
- Commits: `2021b33f`, `17fa57a9`, `c7578a02`
- Evidence: `.audit/subagent-audit-p1.md`

## Findings

1. **LOW (fixed)** — anti-hotlink domain detection in article markdown uses substring matching and can false-positive.
   - Affected files:
     - `webclipper/src/platform/webext/anti-hotlink-rules-store.ts`
     - `webclipper/src/collectors/web/article-fetch.ts`
   - Required fix: switch to URL hostname extraction + exact hostname match.

2. **LOW (fixed)** — no explicit regression test for article-flow fallback when anti-hotlink rule loading fails.
   - Affected files:
     - `webclipper/tests/smoke/article-fetch-service.test.ts`
   - Required fix: mock anti-hotlink rule read failure and assert capture path remains successful without forced cache.

## Remediation status

- [x] Finding 1 fixed
- [x] Finding 2 fixed

## Verification

- `npm --prefix webclipper run compile` ✅
- `npm --prefix webclipper run test` ✅
