# P1 Audit Input (local fallback, no subagent)

## Scope
- webclipper/src/services/comments/threaded-comments-panel.ts
- webclipper/src/ui/styles/inpage-comments-panel.css
- webclipper/src/ui/conversations/ArticleCommentsSection.tsx
- webclipper/src/ui/app/AppShell.tsx
- webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts
- webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts
- webclipper/src/services/integrations/detail-header-actions.ts
- webclipper/src/ui/conversations/ConversationDetailPane.tsx
- webclipper/src/ui/conversations/DetailNavigationHeader.tsx

## Findings
1. Fixed: failure paths were previously collapsed to empty actions in app/inpage resolver, causing ambiguous menu state.
   - Fix commit: 9e25db70
   - Result: app/inpage now surface explicit error messages for missing detail/runtime failures.

## Residual risk
- Manual smoke on real pages is still required for clipboard write and new-tab open behavior.
