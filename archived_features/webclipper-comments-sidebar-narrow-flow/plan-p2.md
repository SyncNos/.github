# Plan P2 - webclipper-comments-sidebar-narrow-flow

**Goal:** 仅在 `app.html` 引入中屏（medium）布局：comments 关闭时两栏、打开时三栏，并且 medium 默认关闭（不继承 wide 已打开状态）。

**Non-goals:** popup 不做 medium；不改 popup 的 narrow 流；不改变 app wide 的既有持久化键语义。

**Approach:** 引入 `narrow/medium/wide` 档位 hook，并在 `AppShell` 中拆分 comments 开关语义：wide 继续使用 `COMMENTS_SIDEBAR_COLLAPSED_KEY`；medium 使用会话态且默认关闭；narrow 交给 P1 的三步 flow。明确跨档位切换状态机，避免 medium 与 wide 互相污染。

**Acceptance:**
- app medium：刷新/首次进入默认两栏；点击 comments 后三栏；关闭后回两栏。
- app wide：`COMMENTS_SIDEBAR_COLLAPSED_KEY` 与自动打开策略不回归。
- `wide -> medium` 时即使 wide 先前为打开，medium 也必须默认关闭。

---

## P2-T1

**Title:** 引入 app 响应式档位（narrow/medium/wide）hook

**Files:**
- Add: `webclipper/src/ui/shared/hooks/useResponsiveTier.ts`
- Modify: `webclipper/src/ui/shared/hooks/useIsNarrowScreen.ts`（仅在需要复用底层 matchMedia 逻辑时）

**Step 1: 实现功能**

- `useResponsiveTier` 输出：`'narrow' | 'medium' | 'wide'`
  - `narrow < md(768)`
  - `wide >= xl(1280)`
  - 其余 `medium`
- 提供可选参数覆盖断点，默认值与 Tailwind 断点一致。
- `useIsNarrowScreen` 可选择复用新 hook，但需保证现有调用方行为不变。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/shared/hooks/useResponsiveTier.ts webclipper/src/ui/shared/hooks/useIsNarrowScreen.ts`

Run: `git commit -m "feat: task1 - 引入responsive tier判定hook"`

---

## P2-T2

**Title:** AppShell 中屏默认关闭 comments，点击后三栏

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**

- `AppShell` 采用 `tier` 分支：
  - narrow：继续走 P1 三步流。
  - medium：保留两/三栏容器，但默认 comments 关闭。
  - wide：暂按现有逻辑（详细在 P2-T3）。
- medium comments 开关不使用 `COMMENTS_SIDEBAR_COLLAPSED_KEY`，仅会话态控制。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "feat: task2 - app中屏comments默认关闭"`

---

## P2-T3

**Title:** AppShell 宽屏行为保持不变（存储键/自动打开不回归）

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**

- wide 下保持：
  - `COMMENTS_SIDEBAR_COLLAPSED_KEY` 读写语义不变。
  - article 默认自动打开 comments sidebar 的 effect 不变（`source: 'app-default'`）。
- medium 分支与 wide 分支状态隔离，避免 medium 的 close 写回 wide 持久化键。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "refactor: task3 - 保持wide comments行为不变"`

---

## P2-T4

**Title:** 中屏与窄屏切换时状态重置策略（medium 始终默认关闭）

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**

- 定义跨档位重置策略：
  - `wide -> medium`: medium 强制回默认关闭。
  - `narrow -> medium`: medium 强制回默认关闭。
  - `medium -> wide`: 恢复 wide 持久化键驱动状态。
- 同时保证 comments runtime 不会残留错误 `openRequested` 导致 medium 初始误开。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "feat: task4 - 中屏状态重置策略"`

---

## P2-T5

**Title:** 修正受影响 AppShell smoke 测试（medium/wide 状态隔离）

**Files:**
- Modify: `webclipper/tests/smoke/app-shell-comments-sidebar.test.ts`
- Modify: `webclipper/tests/smoke/app-shell-sidebar-collapse.test.ts`（如受影响）
- Modify: `webclipper/tests/smoke/app-shell-narrow-header-actions.test.ts`（如受影响）

**Step 1: 实现功能**

- 为 medium/wide 分支补齐或更新断言：
  - medium 默认关闭。
  - wide 继续受 `COMMENTS_SIDEBAR_COLLAPSED_KEY` 影响。
  - 档位切换时 medium 不污染 wide。

**Step 2: 验证**

Run: `npm --prefix webclipper run test -- --run tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/app-shell-sidebar-collapse.test.ts tests/smoke/app-shell-narrow-header-actions.test.ts`

Expected: 目标 smoke 全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/app-shell-comments-sidebar.test.ts webclipper/tests/smoke/app-shell-sidebar-collapse.test.ts webclipper/tests/smoke/app-shell-narrow-header-actions.test.ts`

Run: `git commit -m "test: task5 - 覆盖medium与wide comments状态隔离"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
