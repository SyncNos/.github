# Threaded Comments Panel Manual Smoke (P3-T1)

## Scope

- Product surfaces: `app` (sidebar), `inpage` (overlay/sidebar)
- Rendering path: React-only threaded comments panel (legacy DOM renderer removed)
- Verify behavior parity and regression-sensitive interactions

## Preconditions

1. Install and build extension from current branch.
2. Ensure at least one article conversation exists for app/inpage verification.
3. Prepare one page where comment locate can succeed, and one where locate can fail.

## App Sidebar Flow

1. Open app conversation detail and trigger comments panel.
2. Confirm panel opens and comment list/composer render normally.
3. Add a root comment from composer.
4. Confirm after send:
   - root appears in list
   - focus moves to corresponding reply textarea
   - target reply textarea is visible (scroll into view)
5. Send a reply under that root.
6. Confirm reply is added and focus remains at the same thread reply textarea.
7. Delete flow:
   - first click delete button -> enters confirm state
   - second click same button -> comment/reply deleted
   - click outside -> confirm state clears
   - press `Esc` -> confirm state clears
8. Locate flow (sidebar variant):
   - click quote/comment body -> navigates/highlights source location when resolvable
   - in failure case -> notice appears with failure message and auto-hides
9. Chat with AI:
   - header chat-with menu opens/closes correctly
   - comment-level chat-with: single-action direct trigger works
   - multi-action menu opens and closes on outside click / `Esc`

## Inpage Overlay/Sidebar Flow

1. Trigger inpage comments panel from page action.
2. Confirm open/close works repeatedly and collapse button closes panel.
3. Validate docked/sidebar behavior:
   - panel is anchored as sidebar (not floating wrong position)
   - resize handle appears only in resizable mode
4. Repeat critical interaction checks:
   - root send -> focus+scroll to reply textarea
   - reply send -> focus remains in thread
   - delete two-step confirm + outside/Esc clear
   - chat-with menu behavior (header + comment-level)
   - locate success/failure notice behavior

## Boundary and Keyboard Semantics

1. Busy semantics:
   - send buttons disabled during active operation
   - textarea remains editable while busy
2. Shortcut semantics:
   - `Cmd/Ctrl+Enter` sends composer text
   - `Cmd/Ctrl+Enter` sends reply text for active thread
3. Escape semantics:
   - closes open chat-with menu first
   - then clears delete armed state
4. Click-outside semantics:
   - clears delete armed state
   - closes comment-level chat-with menu

## Regression Watchlist

- No duplicate send/reply/delete requests from rapid repeated clicks.
- No stale chat-with request result overwriting newer menu state.
- No lingering UI updates after panel teardown (no post-close side effects).
