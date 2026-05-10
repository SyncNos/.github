# Subagent Audit P2 (read-only)

Source: code-review subagent (`gpt-5.4-mini`)  
Scope: `threaded-comments-panel-react-migration` phase P2 (`P2-T1` ~ `P2-T11`)

## Verdict

PASS WITH RISKS

## Findings

### P2-1

- Severity: Medium
- Evidence: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx:113-120,145-168,222-234,291-305`
- Why it matters: save/reply/delete async flow lacked synchronous re-entry lock; rapid interaction could trigger duplicate operations before disabled state committed.
- Recommendation: add synchronous mutex/ref lock around async action path and clear in `finally`.

### P2-2

- Severity: Medium
- Evidence: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx:325-366`
- Why it matters: comment-level chat-with flow lacked requestId/loading guard; concurrent clicks could produce stale response races and menu state overwrite.
- Recommendation: add loading lock + request id guard/cancellation semantics for comment-level chat-with resolution.
