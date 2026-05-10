# Plan P3 - comments-sidebar-selection-commit

**Goal:** 补齐 inpage 选区求值的边界保护（避免侧边栏内部选区被附加），并完成全量验证闭环。

**Non-goals:**
- 不新增站点专属适配器
- 不扩展到 comments 以外模块

**Approach:**
在 inpage `resolveInpageSelectionPayload()` 中增加 locatorRoot（`document.body`）包含性校验：若 selection 的 anchor/focus 不在 root 内，视为空选区并直接忽略，避免用户在侧边栏里选中文字导致 quote 被附加。

说明：inpage comments panel 使用 Shadow DOM（`getInpageCommentsPanelApi`），shadow 内部选区的 `anchorNode/focusNode` 不会被 `document.body.contains(...)` 命中，因此该检查可以有效排除“侧边栏内部选区”，同时不影响页面正文（light DOM）选区。

最后跑一轮全量 `compile/test/build` 并对 empty selection 相关回归做收敛。

**Acceptance:**
- inpage 侧边栏内部选中的文本不会被附加为 quote
- 全量验证命令通过

---

## P3-T1

**Task:** Ignore sidebar-internal selection in inpage selection resolver (root contains)

**Files:**
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`

**Step 1: 实现功能**

- 在读取 selection 时增加包含性判定：
  - 若 `anchorNode`/`focusNode` 不在 locatorRoot（`document.body || document.documentElement`）内，返回空 selection（`selectionText:''`）。
- 保持 locator 构建逻辑不变（文本为空时 locator 必为 null）。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Expected: inpage 编译与关键 smoke 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`

Run: `git commit -m "fix: P3-T1 - inpage忽略侧边栏内部选区避免误附加"`

---

## P3-T2

**Task:** Run full verification and tighten regressions around empty selection

**Files:**
- Modify: `webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`
- Modify: `webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`

**Step 1: 实现测试收敛**

- 补齐/收敛 “空选区不触发请求、不清空 quote” 的断言，覆盖 pointer/keyboard 两类提交。
- 确保测试不再依赖固定时间等待（避免回归到时间防抖假设）。

**Step 2: 验证（全量）**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`

Expected: 全量通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Run: `git commit -m "test: P3-T2 - 收敛空选区回归并跑全量验证"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
