## Coverage check

- **P4-T1 — Pass**
  - Legacy converter file `webclipper/src/collectors/web/article-extract/markdown.ts` is removed.
  - `engine.ts` and `sites/discourse.ts` no longer reference the legacy converter fallback path.

- **P4-T2 — Pass**
  - `extractBySiteSpec` return payload no longer includes placeholder `contentMarkdown`.
  - Unit test `tests/unit/article-extract-xiaohongshu.test.ts` now asserts the field is absent.

- **P4-T3 — Pass**
  - `webclipper/src/collectors/web/article-extract/sites/specs.ts` bridge is deleted.
  - `sites/index.ts` now exports `ARTICLE_FETCH_SITE_SPECS` directly from `@collectors/web/article-fetch-sites`.

- **P4-T4 — Pass**
  - `sites/index.ts` no longer re-exports `findDiscourseOpNode` and `isXiaohongshuNotePage`.
  - Remaining usage of both symbols is internal to their own modules.

## Findings

No blocking findings.

## Suggested fixes

- No fixes required for blocking/high-signal issues.

## Verification notes

- Reviewed implementation commits:
  - `155cc5bd`
  - `0fb87489`
  - `37f0e089`
  - `c036c6cc`
- Phase validation command:
  - `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` → exited with code `0`
