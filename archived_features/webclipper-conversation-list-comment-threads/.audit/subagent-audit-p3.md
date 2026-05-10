## Summary
P3 implementation is correct. No blocking findings.

## Findings
- Frontmatter includes `comments_root_count` for article notes.
- Value derives from `computeArticleCommentThreadCount(comments || [])`.
- Empty comments behavior is explicit (`comments_root_count: 0`).
- smoke tests cover both non-empty and empty comments scenarios.

## Suggested fixes
- None.

## Pass checks
- goal checks passed
- target smoke test passed
