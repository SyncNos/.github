# Subagent Audit - P1

Source: `task/general-purpose` read-only audit on commits `2021b33f`, `17fa57a9`, `c7578a02`.

## Summary

- P1-T1 ✅ seed once + explicit empty list semantics implemented.
- P1-T2 ✅ runtime lookup migrated to platform store with failure fallback.
- P1-T3 ⚠️ regression coverage mostly complete, with two low-risk gaps.

## Findings

1. **Low** — domain matching uses plain substring and may produce false positives.
   - `webclipper/src/platform/webext/anti-hotlink-rules-store.ts` (domain includes helper)
   - `webclipper/src/collectors/web/article-fetch.ts` (forced cache decision path)
   - Suggested fix: parse markdown image URLs and match exact hostnames.

2. **Low** — missing explicit test for article flow when anti-hotlink rule loading fails.
   - `webclipper/src/collectors/web/article-fetch.ts` fallback branch
   - Suggested fix: make storage read fail and assert capture still succeeds without forced cache.
