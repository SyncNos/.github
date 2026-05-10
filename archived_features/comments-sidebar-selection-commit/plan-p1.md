# Plan P1 - comments-sidebar-selection-commit

**Goal:** 把“附加选中区（quote）”从时间防抖改为“操作完成事件提交”，并在同一 phase 里移除被替代的旧逻辑与残留引用。

**Non-goals:**
- 不做拖拽过程实时更新（只在完成时更新一次）
- 不引入新设置项
- 不引入“手动附加选区”按钮

**Approach:**
在 `ThreadedCommentsPanel` 内把 selection 监听拆分为两段：`selectionchange` 只标记 dirty；`pointerup/keyup` 作为“操作完成事件”触发一次性 commit。commit 时读取最终 selection signature、做 dedupe，并在空选区时直接忽略（不触发请求、不改变 quote）。同时删除原有 `setTimeout(300)` debounce 与 320ms 空选区抑制等老代码，避免两套机制共存。

**Acceptance:**
- `pointerup` 后触发一次 attach 请求；拖拽过程中不触发
- `keyup` 后触发一次 attach 请求；键盘选区可用（尤其是 `shift + arrow`：在松开 `Shift` 后提交，而不是每次方向键抬起就提交）
- 不再存在 300ms debounce 与 320ms 抑制窗口相关逻辑残留

---

## P1-T1

**Task:** Switch auto-attach trigger to pointerup/keyup commit (remove time debounce)

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

**Step 1: 实现功能（含及时清理老代码）**

- 将 `document.selectionchange` 的职责收敛为：标记 “selection dirty”，不直接触发 attach。
- 新增 commit 触发点：
  - `document.pointerup`（包含 mouse/touch/pen，建议以 capture 监听避免被 stopPropagation 影响）
  - `document.keyup`（覆盖键盘选区，建议以 capture 监听避免被 stopPropagation 影响）
- commit 规则：
  - commit 触发后使用 `requestAnimationFrame`（或等价的 next-frame）读取最终 selection signature，避免读取到尚未稳定的 selection（仍然是“事件驱动”，不是时间 debounce）。
  - 若 selection 为空：直接忽略（不触发 `onComposerSelectionRequest`，不影响 quote）
  - 若 signature 与上次相同：忽略（dedupe）
  - 否则触发 `onComposerSelectionRequest({ trigger: 'auto' })`
- `keyup` 特殊规则（避免“选区仍在进行中”时误提交）：
  - 若事件上仍处于 modifier 按下状态（`event.shiftKey || event.ctrlKey || event.metaKey || event.altKey` 为 true），视为仍在操作中：本次不 commit，等待用户松开 modifier 再提交。
  - 这可以保证 `shift + arrow` 只在用户松开 `Shift` 后提交一次，而不是每次方向键抬起都提交。
- 若 handlers 尚未安装：保持“pending request”语义（等 handlers 可用再 commit 一次）。
- **删除老旧代码（同提交完成）**：
  - 移除 `setTimeout(300)` debounce 与相关 timer/flag/ref
  - 移除 320ms 空选区抑制与其调用点（因为空选区不再触发请求/清空）
  - 移除与上述逻辑绑定的无效分支与测试依赖点（仅限本 task 触及范围）

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: 类型检查通过，comments panel 可编译。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

Run: `git commit -m "feat: P1-T1 - 选区附加改为pointerup/keyup提交并移除时间防抖"`

---

## P1-T2

**Task:** Update unit/smoke tests for commit-based selection attach

**Files:**
- Modify: `webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`
- Modify: `webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts`
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

**Step 1: 实现测试改造（同步删除老断言/等待方式）**

- 把依赖 `advanceTimersByTime(300)` / `setTimeout(350)` 的等待方式替换为：
  - `selectionchange` 仅标记 dirty
  - 通过 dispatch `pointerup` 或 `keyup` 触发一次性 commit
- 更新测试描述文案：不再称为 “debounced”。
- 保持原有回归覆盖：重复 selectionchange dedupe；composer/reply 交互不应触发 quote 变化。
- **删除老旧测试逻辑（同提交完成）**：移除对 300ms debounce 的假设与相关 fake timer 依赖点（若不再需要）。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/smoke/app-shell-comments-sidebar.test.ts`

Expected: 相关 unit/smoke 通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/unit/threaded-comments-panel-auto-attach-selection.test.ts webclipper/tests/smoke/inpage-comments-sidebar-toggle.test.ts webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`

Run: `git commit -m "test: P1-T2 - 更新选区附加测试为操作完成事件提交"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
