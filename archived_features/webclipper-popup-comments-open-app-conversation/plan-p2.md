# Plan P2 - webclipper-popup-comments-open-app-conversation (Cancelled)

**Status:** Cancelled by user.

**Goal:** 不再执行（原计划：在 app 侧实现 loc 就绪后的“一次性自动展开 comments + focus”闸门，并避免重复触发）。

**Non-goals:**
- 不改变 popup 的 Goal 2A 行为（仍然只负责定位到 conversation）。
- 不修改 comments sidebar 的 Chat-with 行为归属（继续只在 sidebar）。
- 不在本阶段做 `git commit`（除非你之后明确要求）。

**Approach:**
- 使用 query（例如 `openComments=1&focus=1`）作为一次性指令（注意：HashRouter 下应读取 `useLocation().search`，不要用 `window.location.search`）。
- app 解析后等待 loc 对应 conversation 真正选中（必要时等待 detail ready / comments context ready），再触发：
  - wide/medium：展开右侧 comments sidebar 并 focus composer
  - narrow：进入 comments route 并 focus composer
- 触发完成后 `navigate(..., { replace: true })` 清理 query，避免刷新/重新激活 tab 时重复打开。

**Acceptance:**
- `#/?loc=...&openComments=1&focus=1` 可稳定打开并聚焦 comments；触发后 query 被清掉；不会出现“打开太早导致空白/闪烁”。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。

---

## P2-T1 （暂缓）app 侧实现 loc 就绪后的“一次性自动展开 comments”闸门

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify (if needed): `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Add/Modify: `webclipper/tests/smoke/**`

**Step 1: 加入一次性指令解析**
- 解析 query `openComments=1` / `focus=1`；
- 记录为一次性 pending intent（仅当前导航周期）。

**Step 2: loc 就绪闸门**
- 以“conversation 已选中 + comments context 已 setContext +（必要时）detail ready”为条件触发 open；
- narrow：进入 `comments` route；
- wide/medium：`commentsSidebarController.open({ focusComposer: true, source: 'popup', ensureContext: false })` 并展开 sidebar。

**Step 3: one-shot 清理**
- 触发后 replace 清理 query；
- 保证回到 tab / 刷新不会重复展开。

**Step 4: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
