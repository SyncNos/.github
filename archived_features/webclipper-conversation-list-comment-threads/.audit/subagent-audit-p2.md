## Summary
P2 implementation is functionally complete. No blocking findings.

## Findings

### Medium (non-blocking) - number null/zero comparison edge case
- **File:** `webclipper/src/services/sync/notion/notion-sync-orchestrator.ts`
- `normalizePagePropertyValue` can treat `{ number: null }` as equivalent to `{ number: 0 }` if implemented via `Number(prop.number)` directly.
- This may skip property update when a page has null but desired is explicit 0.

## Suggested fix
- Normalize null/undefined number to empty string before numeric conversion.

## Pass checks
- `Comment Threads` is present in article dbSpec + ensureSchemaPatch.
- create/update paths include `Comment Threads`.
- smoke test checks create properties include `Comment Threads`.
