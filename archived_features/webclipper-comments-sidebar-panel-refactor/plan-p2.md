# Plan P2 - webclipper-comments-sidebar-panel-refactor

**Goal:** 在 `ui/comments/threaded-comments-panel/` 内完成职责拆分，把 `panel.ts` 从“巨型实现”拆成多个小模块（仍保持行为不变）。

**Non-goals:** 本 phase 不做跨模块大范围去重（放到 P3），不做 UI 文案/i18n 调整。

**Approach:**
- 以“单向依赖”的模块结构拆分：`panel.ts` 只做组装；其余模块按单一职责提供小 API。
- 每个 task 拆一类职责，并保持可独立编译通过。

**Acceptance:**
- `panel.ts` 体积显著下降；复杂逻辑被下沉到清晰文件（建议每个文件 < 300 行）。
- 行为保持一致；`lint/compile/test` 通过。

---

<a id="p2-t1"></a>
## P2-T1 拆分：抽离 Shadow CSS 构建与注入

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/shadow-styles.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 把 `toHostTokensCss()` 与 `PANEL_SHADOW_CSS` 构建移入 `shadow-styles.ts`，导出 `buildThreadedCommentsPanelShadowCss()`（或同等命名）。
2. `panel.ts` 仅负责把 style element append 到 shadow root。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task1 - 抽离 threaded comments panel shadow styles`

---

<a id="p2-t2"></a>
## P2-T2 拆分：抽离 dockPage 顶开页面逻辑

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/dock.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 把 `ensureDockStyle/readDockWidthPx/dockResize/setDockOpen` 等 dock 相关逻辑封装为 `createDockController(...)`。
2. `panel.ts` 只持有 controller，并在 open/close/cleanup 时调用。
3. 保持现有 DOM contract 不变：`html[data-webclipper-comments-dock='1']` + `--webclipper-comments-dock-width`。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task2 - 抽离 dockPage 控制器`

---

<a id="p2-t3"></a>
## P2-T3 拆分：抽离 sidebar resize + width persistence

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/resize.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 把 width state、`COMMENTS_SIDEBAR_WIDTH_*`、`readPersistedSidebarWidthPx/persistSidebarWidthPx/clampSidebarWidthPx` 与 pointer drag 事件封装为 `installSidebarResize(...)`。
2. `panel.ts` 只负责：当 `variant=sidebar` 时创建 handle，并调用 install 返回 cleanup。
3. 保持 storage key 与 CSS var contract 不变：`webclipper_comments_sidebar_width_v1` + `--webclipper-comments-panel-width`。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task3 - 抽离 sidebar resize 与持久化`

---

<a id="p2-t4"></a>
## P2-T4 拆分：抽离 Chat-with 菜单状态机与渲染

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/chatwith.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 将 `chatWithState`、action normalize、trigger label 选择、menu open/close、render menu items 等封装为 `createChatWithMenuController(...)`。
2. `panel.ts` 只负责：在 header 中创建 root 节点，并把 root 交给 controller 绑定。
3. 保持行为：单 action 时直接触发；多 action 时展开 menu；错误/空态仍通过 notice 展示。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task4 - 抽离 chat-with 菜单控制器`

---

<a id="p2-t5"></a>
## P2-T5 拆分：抽离 locate/scroll/highlight 能力

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/locate.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 把高亮 style 注入、flash/clear、scrollIntoView、inpage retry+nudge 等 locate 逻辑封装为 `createThreadLocateController(...)`。
2. `panel.ts` 只在 click handler 中调用 locate controller，避免 locate 细节散落在 render 代码里。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task5 - 抽离 locate/highlight 控制器`

---

<a id="p2-t6"></a>
## P2-T6 拆分：抽离 threads/composer 渲染与事件绑定

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/render.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/panel.ts`

**Step 1: 实现**
1. 将 DOM create 与事件绑定拆到 `render.ts`：
   - header/body/composer/threads 的节点创建
   - render comments/replies
   - refreshButtons/autosizeTextarea 等 UI 细节
2. `panel.ts` 只保留“状态 + API 实现 + 组装各 controller”，render 由 `render.ts` 以小函数形式暴露（避免引入 class）。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task6 - 抽离 threaded comments panel 渲染层`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Focus:
  - 模块边界是否清晰、依赖是否单向
  - `panel.ts` 是否只做组装（不再成为“第二个巨型文件”）
  - 是否引入了不必要的通用工具导致臃肿

