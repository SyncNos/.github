# Plan P1 - webclipper-comments-sidebar-panel-refactor

**Goal:** 把右侧 comments sidebar 的 UI 真源从 `services/**` 归位到 `ui/**`，并建立可拆分的模块骨架（不改行为）。

**Non-goals:** 本 phase 不做功能变更、不改变外部行为、不大规模改类型（类型严控在 P3）。

**Approach:**
- 先完成“落点归位 + 单一入口导出”，把后续拆分的落脚点固定住。
- 所有引用点一次性迁移到新入口，避免 `services -> ui` 反向依赖。

**Acceptance:**
- `webclipper/src/services/comments/threaded-comments-panel.ts` 不再承载 UI 实现（迁移完成或仅保留短期过渡并在后续 phase 移除）。
- `app` 与 `inpage` 的引用点更新到 `ui/**` 新入口，行为不变。
- `npm --prefix webclipper run lint && npm --prefix webclipper run compile && npm --prefix webclipper run test` 通过。

---

<a id="p1-t1"></a>
## P1-T1 归位：将 threaded comments panel 从 services 迁移到 ui

**Files:**
- Move: `webclipper/src/services/comments/threaded-comments-panel.ts` -> `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`
- Add: `webclipper/src/ui/comments/threaded-comments-panel/index.ts`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Delete: `webclipper/src/services/comments/threaded-comments-panel.ts`（若确认无引用）

**Step 1: 实现**
1. 在 `ui/comments/threaded-comments-panel/` 下建立新入口 `index.ts`，对外导出 `mountThreadedCommentsPanel` 与必要类型。
2. 迁移原实现到 `panel.ts`，保持导出 API 与行为不变（不改函数签名/option 字段/DOM contract）。
3. 更新 `AppShell.tsx` / `ArticleCommentsSection.tsx` / `inpage-comments-panel-shadow.ts` 的 import 路径，确保 `ui` 不再从 `services/comments/threaded-comments-panel` 引入。
4. 优先用 `git mv` 执行迁移以保留历史（该文件很大，保留 blame 价值高）。
5. 用 `rg -n "from '@services/comments/threaded-comments-panel'" webclipper/src` 确认引用点已清零，再删除旧文件。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run lint`

**Step 3: 原子提交**
- `refactor: task1 - threaded comments panel 迁移到 ui`

---

<a id="p1-t2"></a>
## P1-T2 建立新模块结构与公共 types 导出

**Files:**
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/index.ts`
- Add: `webclipper/src/ui/comments/threaded-comments-panel/types.ts`

**Step 1: 实现**
1. 把对外类型（`ThreadedCommentItem`/`ThreadedCommentsPanelApi`/`ThreadedCommentsPanelChatWithConfig`/`MountOptions` 等）集中到 `types.ts`。
2. `panel.ts` 只负责实现与组装；类型从 `types.ts` import（减少循环依赖与“类型散落”）。
3. `index.ts` 作为唯一对外入口：只 re-export 必要符号，避免外部直接 import 内部实现文件。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`

**Step 3: 原子提交**
- `refactor: task2 - threaded comments panel 导出类型收敛`

---

<a id="p1-t3"></a>
## P1-T3 验证链：lint + compile + test（保持行为不变）

**Step 1: 验证**
- Run: `npm --prefix webclipper run lint`
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`

**Step 2: 原子提交**
- `chore: task3 - threaded comments panel 重构基线验证`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Focus:
  - 是否彻底消除 `services -> ui` 反向依赖
  - `ui/**` 是否仍遵守“不 import platform”约束
  - 行为是否保持一致（特别是 dockPage/resize/chatwith/locate/快捷键）
