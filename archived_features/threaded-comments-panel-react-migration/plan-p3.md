# Plan P3 - threaded-comments-panel-react-migration

**Goal:** 收尾与防回归：补齐验证清单与最小可测覆盖，做边界自检与最终冒烟，确保 React 迁移后的 comments panel 在 app + inpage 均稳定可用。

**Non-goals:** 本 phase 不引入新功能；不继续扩张 UI 设计体系；不做大规模重排文件结构。

**Approach:** 以“可验证结果”为收束：先固化手工回归脚本，再补最小单测覆盖 focus/scroll 的关键决策点（不强依赖真实 DOM layout），再做分层边界自检（避免 platform 依赖渗透到 ui），最后跑全链路 compile/test/build 并按验收标准走一遍交互冒烟。

**Acceptance:**
- 有一份可重复的手工回归清单（app + inpage）。
- 至少有一组单测覆盖 focus/scroll 目标选择与“不要抢焦点”规则。
- 边界自检通过（ui/viewmodels 不 import platform）。
- 最终 `compile && test && build` 通过。

---

## P3-T1

**Title:** 补齐回归验证清单与手工复现脚本（app + inpage）

**Files:**
- Add: `.github/features/threaded-comments-panel-react-migration/.audit/manual-smoke.md`

**Step 1: 实现功能**

- 编写手工回归清单（不入 `todo.toml` 状态）：
  - app sidebar：open -> root send -> focus+scroll to reply；reply send -> keep focus；delete confirm；locate；chatwith
  - inpage overlay：open -> 同样验证；dock/resize 基本可用
  - 边界：busy 状态下 send 禁用但 textarea 可输入；Esc 关闭菜单/清理 armed delete；click-outside 关闭菜单

**Step 2: 验证**

Run: `cat .github/features/threaded-comments-panel-react-migration/.audit/manual-smoke.md`

Expected: 清单可读且覆盖验收点。

**Step 3: 原子提交**

说明：该 feature 目录下的计划/证据文件默认不入库；本任务只要求落盘可复用，不做 `git commit`。

---

## P3-T2

**Title:** 为 focus/scroll 关键逻辑补最小可测单元（避免回归）

**Files:**
- Add: `webclipper/src/ui/comments/react/focus-rules.ts`
- Add: `webclipper/src/ui/comments/react/focus-rules.test.ts`

**Step 1: 实现功能**

- 将“目标 rootId 的选择逻辑”抽成纯函数（可测）：
  - root save result -> target rootId
  - reply send -> target rootId
  - “不要抢焦点”规则：当 `hasFocusWithinPanel === false` 时返回 null（Shadow DOM 下不要直接依赖 `document.activeElement`）
- 用 vitest 写最小单测覆盖上述规则（不依赖真实 `scrollIntoView`）。

**Step 2: 验证**

Run: `npm --prefix webclipper run test`

Expected: 测试通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/focus-rules.ts webclipper/src/ui/comments/react/focus-rules.test.ts`

Run: `git commit -m "test: P3-T2 - 为comments面板聚焦规则补最小单测"`

---

## P3-T3

**Title:** 清理类型与边界：避免 platform 依赖渗透到 ui 层（边界自检）

**Files:**
- Modify: `webclipper/src/ui/comments/react/*`（如需修复依赖边界）
- Modify: `webclipper/src/ui/comments/panel.ts`（如需修复依赖边界）

**Step 1: 实现功能**

- 运行边界自检并修复：
  - `rg -n "src/platform|/platform/" webclipper/src/ui`
  - `rg -n "src/platform|/platform/" webclipper/src/viewmodels`
  - `rg -n "@platform/" webclipper/src/ui webclipper/src/viewmodels`
- 保证 comments React 新增文件不直接 import `@platform/*`。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react webclipper/src/ui/comments/panel.ts`

Run: `git commit -m "chore: P3-T3 - 修复comments面板react迁移的分层边界"`

---

## P3-T4

**Title:** 最终全链路验证：compile/test/build + 关键交互冒烟

**Files:**
- Modify: （无；如发现问题按最小修复提交）

**Step 1: 实现功能**

- 按 `.audit/manual-smoke.md` 逐条手工验证（app + inpage）。
- 修复发现的问题（每个问题单独成 commit，commit message 以对应 task id 前缀开头，例如 `fix: P3-T4 - ...`）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。

**Step 3: 原子提交**

Run: `git status --porcelain`

Expected: 仅剩计划文件未提交（按约定计划文件不入库）；代码改动均已提交。

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
