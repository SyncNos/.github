# Plan P2 - threaded-comments-panel-react-migration

**Goal:** 将 threaded comments panel 迁移为 React 实现（shadow 内渲染 + keyed 列表），并在发送 comment 后稳定 focus + scroll 到目标 `Reply…` 输入框；同时保持现有交互能力不回退，并删除 legacy DOM 实现。

**Non-goals:** 本 phase 不引入虚拟列表、不做性能极限优化；不重做 i18n 与文案体系；不改变消息协议与存储 schema。

**Approach:** 在 `mountThreadedCommentsPanel()` 内保留“面板外壳/结构能力”（shadow root、open/close、dock/resize、stopPropagation），把 UI 交互（header/body/composer/threads）迁移为 React 组件。通过一个 panel store（`useSyncExternalStore`）把 `api.set*` 的 imperative 调用桥接到 React state；React 内用 rootId/commentId 作为 key，确保 `Reply…` textarea 节点稳定存在。发送成功后使用 `createdRootId` / `rootId` 精确 focus 并 `scrollIntoView`。

**Migration strategy (important):**
- P2 的目标是“最终只剩 React 实现”，但为了降低中途回归与可验证性风险：
  - 在 P2-T1~T9 期间允许保留 legacy DOM 渲染代码（不删除、不在用户面前双显）
  - 保证同一时刻只存在一套“可见 UI 渲染器”（legacy 或 React），避免重复 UI/事件冲突
  - P2-T10 一次性把默认渲染器切到 React（此时 React 应已具备 feature parity）
  - P2-T11 删除 legacy DOM 渲染与残留

**Acceptance:**
- React 面板在 app sidebar 与 inpage overlay 两处生效，并满足 focus + scroll 需求。
- delete 二次确认、locate、Chat with AI 菜单（header + comment-level）等能力可用且不回退。
- legacy DOM 实现被删除（`render.ts` 旧路径不再存在）。
- 最小验证：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。

---

## P2-T1

**Title:** 新增 React 版 ThreadedCommentsPanel 组件骨架（shadow 内渲染）

**Files:**
- Add: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Add: `webclipper/src/ui/comments/react/types.ts`（如需要）

**Step 1: 实现功能**

- 新增 React 组件骨架，职责：
  - 纯 UI：header/body/quote/composer/threads 容器（先不接入复杂行为）
  - 包含 notice 区域（用于 locate/chatwith 的提示），先以 props 驱动显示（计时隐藏逻辑放在 panel.ts/store）
  - 只依赖 `@ui/*` 与 `@services/shared/*`（禁止 import `@platform/*`）
- 组件接受最小 props：
  - `variant`、`fullWidth`、`surfaceBg`（如需要）
  - `snapshot`（busy/open/quote/comments）
  - `handlers`（onSave/onReply/onDelete/onClose）
  - `onRequestClose`（collapse/close）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/comments/react/types.ts`

Run: `git commit -m "feat: P2-T1 - 新增react版comments panel组件骨架"`

---

## P2-T2

**Title:** 引入 panel store 并用 useSyncExternalStore 把 api.set* 桥接到 React

**Files:**
- Add: `webclipper/src/ui/comments/react/panel-store.ts`
- Modify: `webclipper/src/ui/comments/panel.ts`

**Step 1: 实现功能**

- 新增 `panel-store.ts`：
  - `getSnapshot()`/`subscribe()`/一组 setter（busy/quote/comments/handlers/openState/notice）
  - snapshot 结构与现有 `ThreadedCommentsPanelApi` 能力对齐
- 在 `panel.ts`：
  - 创建 shadow root 后，新增一个 React mount container（例如 `div.webclipper-inpage-comments-panel__react-root`）
  - 用 `react-dom/client` 挂载 `<ThreadedCommentsPanel ... />`
  - `apiRef.setBusy/setQuoteText/setComments/setHandlers/open/close` 更新 store，触发 React 重渲染
- “只显示一个渲染器”的落地方式（二选一，执行时选其一并在 P2-T11 清理掉中间态）：
  - 方式 A（推荐，最少中途回归）：P2-T2 只做 store + React mount，但默认仍显示 legacy UI；在 P2-T10 切换为 React 可见并移除 legacy 可见路径
  - 方式 B（更激进）：P2-T2 起直接让 React 变为可见渲染器（允许中途分支暂时功能不全，但不得在 P2 完成前合并）
- 保留既有结构能力：
  - dock/resize 的安装逻辑不变
  - stopShortcutKeyPropagation 仍挂在 shadow root
- 明确约束：不允许“legacy 与 React 同时可见”（避免重复 UI/事件冲突）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/panel-store.ts webclipper/src/ui/comments/panel.ts`

Run: `git commit -m "refactor: P2-T2 - 引入panel store并桥接到react渲染"`

---

## P2-T3

**Title:** 迁移 threads/replies 渲染到 React keyed 列表（不再全量销毁重建）

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

**Step 1: 实现功能**

- 在 React 组件中实现 comments 渲染：
  - roots：`parentId == null`，排序与现有一致（createdAt desc + id desc）
  - replies：按 rootId 分组，排序与现有一致（createdAt asc + id asc）
  - 使用 `key={root.id}` / `key={reply.id}`，确保节点稳定
- 渲染 thread 内的 `Reply…` textarea 作为稳定节点（为后续 focus/scroll 准备 ref）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

Run: `git commit -m "feat: P2-T3 - react keyed渲染threads与replies"`

---

## P2-T4

**Title:** 迁移 root composer 到 React 并保持快捷键/禁用态/清空逻辑

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

**Step 1: 实现功能**

- 在 React 组件中实现顶部 composer：
  - textarea `Write a comment…`（沿用现有 placeholder，避免 i18n 扩散）
  - send 按钮禁用规则：`busy || !trim(text)`
  - `Cmd/Ctrl+Enter` 发送（与现有一致）
  - 发送成功后清空 textarea + autosize（可用 `scrollHeight` 方案或维持简单实现）
- 保持“busy 时 textarea 仍可编辑，但 send 被禁用”的语义。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

Run: `git commit -m "feat: P2-T4 - react迁移顶部composer与发送逻辑"`

---

## P2-T5

**Title:** 实现 focus + scroll 闭环：root 创建后跳到对应 Reply，reply 发送后停留

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/ui/comments/react/panel-store.ts`（如需要透传 createdRootId）

**Step 1: 实现功能**

- root 发送成功后：
  - 读取 `handlers.onSave(text)` 的返回值 `{ ok: true, createdRootId }`
  - 将其作为“目标 rootId”，在 React state 中设置 `pendingFocusRootId`
  - 在 comments 更新并渲染完成后（`useLayoutEffect` 或 `requestAnimationFrame`）：
    - focus 对应 thread 的 `Reply…` textarea
    - `scrollIntoView({ block: 'nearest' })`
    - caret 移到末尾
- reply 发送成功后：
  - 目标 rootId = 当前 thread rootId
  - 清空 reply textarea 后立即 focus（同节点稳定存在，focus 不应丢）
  - 同步 `scrollIntoView({ block: 'nearest' })` 保证可见
- 约束：
  - 若用户在发送过程中主动把焦点移出 panel（例如点击页面其他位置），不强行抢回。
  - Shadow DOM 注意点：不能只用 `document.activeElement` 判断“焦点是否在 panel 内”（在 shadow 场景下它通常退化为 host）。
    - 推荐实现：在 panel.ts/shadowRoot 上监听 `focusin/focusout`（或读取 `shadowRoot.activeElement`）维护一个 `hasFocusWithinPanel` 信号，focus-rules 以此信号决定是否执行聚焦。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/comments/react/panel-store.ts`

Run: `git commit -m "feat: P2-T5 - 发送后聚焦并滚动到目标reply输入框"`

---

## P2-T6

**Title:** 迁移 delete 二次确认（按钮内联确认）并实现 click-outside / escape 清理

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

**Step 1: 实现功能**

- 在 React 组件中实现 delete 行为：
  - 第一次点击：进入 armed 状态（该按钮变为确认态文案 + tooltip）
  - 第二次点击：调用 `handlers.onDelete(id)`，busy 期间禁用
  - 点击 panel 内其他区域：清除 armed 状态
  - `Esc`：清除 armed 状态（不影响输入内容）
- 保持“按钮内联二次确认”语义，不使用 `alert/confirm`。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

Run: `git commit -m "feat: P2-T6 - react迁移评论删除二次确认交互"`

---

## P2-T7

**Title:** 迁移 sidebar locate：点击 quote 定位 root comment 并保留失败提示

**Files:**
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/ui/comments/locate.ts`（如需要暴露更适配 React 的方法）

**Step 1: 实现功能**

- 保持现有 locate controller 能力（`createThreadLocateController`）：
  - `panel.ts` 创建 locate controller，并把 `locateThreadRoot(root)` 与 `onLocateFailed()` 通过 props 传给 React 组件
- 在 React 中：
  - 仅在 `variant === 'sidebar'` 时让 quote 可点击
  - 点击 quote 时，调用 `locateThreadRoot(root)`；失败则触发 notice（复用现有“无法定位”提示语义）
  - 点击目标过滤：可复用现有 `shouldIgnoreLocateClick` 逻辑（避免输入控件触发 locate）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/comments/locate.ts`

Run: `git commit -m "feat: P2-T7 - react迁移sidebar定位locate交互"`

---

## P2-T8

**Title:** 迁移 header Chat with AI 菜单（如启用）到 React 并复用现有 controller

**Files:**
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

**Step 1: 实现功能**

- 复用 `createChatWithMenuController`：
  - `panel.ts` 继续持有 controller 实例
  - React header 提供一个容器 ref（`div`），`panel.ts` 在 ref attach 时调用 `controller.attachRoot(el)`
- 保持现有行为：
  - 点击 shadow 内其他地方关闭菜单
  - `Esc` 关闭菜单

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`

Run: `git commit -m "feat: P2-T8 - react迁移header chatwith菜单并复用controller"`

---

## P2-T9

**Title:** 迁移 comment-level Chat with AI 菜单（如启用）并保持 Esc/点击关闭语义

**Files:**
- Modify: `webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- Modify: `webclipper/src/ui/comments/comment-chatwith-config.ts`（如需要）

**Step 1: 实现功能**

- 在 React thread/comment 渲染中实现 commentChatWith：
  - trigger：每个 root comment（以及必要时 replies）渲染菜单入口
  - open/close：与现有语义一致（点击切换，点击外部关闭，`Esc` 关闭）
  - actions：复用 `ThreadedCommentsPanelCommentChatWithConfig.resolveActions(...)`
  - disabled：busy 时禁用；action 级 disabled 保持
- 关闭逻辑要支持：
  - shadow click 关闭所有打开的 comment chatwith 菜单
  - `Esc` 关闭（优先级不干扰输入）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx webclipper/src/ui/comments/comment-chatwith-config.ts`

Run: `git commit -m "feat: P2-T9 - react迁移comment级chatwith菜单"`

---

## P2-T10

**Title:** 切换 mountThreadedCommentsPanel 默认使用 React 实现并验证 app + inpage

**Files:**
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/index.ts`（如需要）

**Step 1: 实现功能**

- `mountThreadedCommentsPanel()` 不再渲染 legacy DOM UI，仅渲染 React 面板。
- 确认两个宿主调用点无需改动：
  - app：`webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
  - inpage：`webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/index.ts`

Run: `git commit -m "refactor: P2-T10 - mountThreadedCommentsPanel切换为react实现"`

---

## P2-T11

**Title:** 删除 legacy DOM 实现（panel.ts/render.ts 旧路径）并清理未使用代码

**Files:**
- Delete: `webclipper/src/ui/comments/render.ts`
- Modify: `webclipper/src/ui/comments/panel.ts`
- Modify: `webclipper/src/ui/comments/index.ts`
- Modify: `webclipper/src/ui/comments/types.ts`（如需抽取更明确的结果类型）

**Step 1: 实现功能**

- 删除 `render.ts` 及其旧导出/调用链路（不保留双实现）。
- 清理 `panel.ts` 中仅用于旧 DOM 渲染的辅助函数与状态。
- 确保：
  - `stopShortcutKeyPropagation` 仍保留
  - resize/dock/open/close 结构能力仍保留

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/comments/panel.ts webclipper/src/ui/comments/index.ts webclipper/src/ui/comments/types.ts`

Run: `git add -u`

Run: `git commit -m "refactor: P2-T11 - 删除legacy DOM渲染实现并清理残留"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
