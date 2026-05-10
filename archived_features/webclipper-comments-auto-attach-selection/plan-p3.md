# Plan P3 - webclipper-comments-auto-attach-selection

**Goal:** 完成边界加固、兼容性回归与文档对齐，确保新入口可稳定长期维护。

**Non-goals:**
- 不新增产品功能
- 不重构 comments 模块整体架构

**Approach:**
先把“仅根输入框触发”与“无选区清空”做成显式守卫，再执行全链路回归（双击入口、popup/context menu 入口、保存/回复/删除主流程）。最后补齐 deepwiki 事实描述，避免后续认知漂移。

**Acceptance:**
- 仅根输入框触发自动附加，回复输入框始终不触发
- 既有打开入口与评论主流程无回归
- deepwiki 行为说明与实际实现一致

---

## P3-T1

**Task:** Enforce root-only trigger guard

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/react/focus-rules.ts`（如需）

**Step 1: 实现功能**

- 显式限制自动附加触发点仅为根评论输入框。
- 为“无选区清空”增加防重入/防误触守卫，避免 reply 输入行为污染引用区。
- 对 `pointerdown + focus` 同一交互做去重，避免一次点击触发两次附加。
- 仅根输入框交互允许触发“空选区清空”；reply 输入框 focus/input 不得触发任何清空逻辑。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/unit/threaded-comments-panel-focus-regression.test.ts tests/unit/threaded-comments-panel-shortcuts.test.ts`

Expected: 根输入框与回复输入框行为边界稳定。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/react/focus-rules.ts`

Run: `git commit -m "fix: P3-T1 - 固化根输入框自动附加触发边界"`

---

## P3-T2

**Task:** Run compatibility regressions

**Files:**
- Modify: `webclipper/tests/smoke/content-controller-inpage-combo.test.ts`
- Modify: `webclipper/tests/smoke/background-router-open-comments-sidebar.test.ts`
- Modify: `webclipper/tests/smoke/inpage-button-click-combo.test.ts`

**Step 1: 实现功能**

- 补齐兼容性断言：双击 inpage 按钮链路、background relay、inpage click combo 规则不回归。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- tests/smoke/content-controller-inpage-combo.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/smoke/inpage-button-click-combo.test.ts`

Expected: 现有入口兼容测试全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/content-controller-inpage-combo.test.ts webclipper/tests/smoke/background-router-open-comments-sidebar.test.ts webclipper/tests/smoke/inpage-button-click-combo.test.ts`

Run: `git commit -m "test: P3-T2 - 回归评论侧栏既有入口兼容性"`

---

## P3-T3

**Task:** Update behavior documentation

**Files:**
- Modify: `webclipper/AGENTS.md`
- Modify: `.github/deepwiki/modules/webclipper.md`
- Modify: `.github/deepwiki/business-context.md`
- Modify: `README.md`
- Modify: `README.zh-CN.md`

**Step 1: 实现功能**

- 更新“评论侧栏触发方式”事实描述：补充根输入框自动附加选区行为与边界。
- 保持单一事实源：deepwiki 记录完整行为细节，README 仅保留摘要与入口说明，避免多处冲突。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 代码检查通过，文档与实现行为一致。

**Step 3: 原子提交**

Run: `git add webclipper/AGENTS.md .github/deepwiki/modules/webclipper.md .github/deepwiki/business-context.md README.md README.zh-CN.md`

Run: `git commit -m "docs: P3-T3 - 同步评论自动附加选区行为说明"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
