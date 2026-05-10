# Plan P3 - ai-chat-outline-panel

**Goal:** 把目录面板打磨到“长期可用”：hover 不闪退、键盘可达、长对话性能稳定，并补齐文档与验收清单。

**Non-goals:**
- 不引入新设置项（不加开关）。
- 不改变现有消息渲染结构/markdown renderer（仅在外层加 wrapper/refs）。

**Approach:**
- 交互层集中处理 open/close：仅把手触发；面板与把手之间切换用关闭延迟吸收鼠标轨迹抖动。
- 可访问性用语义元素 + 视觉清零：允许 `<button>` 但完全去按钮外观；补齐 `aria-label` 与 `Esc` 关闭。
- 性能上避免每次渲染重建大型数组/Map；IO 观察对象只限 user 消息锚点。

**Acceptance:**
- hover 把手/面板时不会闪开闪关；移出后稳定收回。
- 键盘可聚焦把手并能打开/关闭面板（至少支持 `Tab` 聚焦、`Esc` 关闭）。
- 在超长对话中滚动与高亮保持流畅。
- 文档与手工验收清单可直接复用。

---

<a id="p3-t1"></a>
## P3-T1 交互打磨：仅把手触发、关闭延迟、防闪与面板覆盖把手

**Files:**
- Modify: `webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`（仅当需要调整定位/容器）

**Step 1: 实现功能**
- open/close 策略：
  - 仅把手 `mouseenter` 打开（不做额外热区）
  - 鼠标离开把手/面板时启动 `closeDelay`（例如 140–220ms），进入面板则 cancel，避免“穿过空隙”导致闪退
  - 打开时面板覆盖把手（把手 `opacity:0` + `pointer-events:none`）
- 定位策略：
  - 把手固定在内容区右上角，`right: 0`，不与 header actions 抢位（header 下方）
  - 面板从右侧滑出（overlay），不改变消息布局宽度

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 无错误。
- 手工：在 app + popup，hover 把手->面板，快速移动鼠标不闪退。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- Run: `git commit -m "feat: P3-T1 - 打磨AI chat目录hover交互并防闪"`

---

<a id="p3-t2"></a>
## P3-T2 可访问性：键盘可达与 Esc 关闭（不改变视觉为按钮）

**Files:**
- Modify: `webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`

**Step 1: 实现功能**
- 把手语义：
  - 优先使用 `<button type="button">`（避免自造 role/button）
  - 视觉上必须“非按钮”：无边框、无阴影、无背景（或透明），仅两条横线
  - `aria-label="Outline"`（不改 i18n；仅供读屏）
  - 保留可见的 `:focus-visible` 轮廓（可很轻），避免“完全无焦点提示”
- 键盘行为：
  - `Enter`/`Space` 打开面板（可选；至少要能通过 hover+focus 触发）
  - `Esc` 关闭面板并把 focus 还给把手
  - 面板内条目可点击即可（不强制做 roving tabindex；先保证基础可用）

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- 手工：仅键盘操作能打开/关闭面板；关闭后 focus 不丢失。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- Run: `git commit -m "feat: P3-T2 - 为AI chat目录把手补齐键盘与Esc关闭"`

---

<a id="p3-t3"></a>
## P3-T3 性能与一致性：大对话渲染稳定、状态 memo、必要的降级策略

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/chat-outline/useChatOutlineActiveIndex.ts`

**Step 1: 实现功能**
- 避免无谓重算：
  - `entries` 用 `useMemo` 基于 `detail.messages` 计算
  - `messageId -> element` 的 Map 用 ref 持久化，不在 render 里 new
  - IO 观察对象列表变化时，先 `disconnect` 再重新 observe，避免泄漏
- 降级策略（只在必要时）：
  - 若 `IntersectionObserver` 不可用（极少），回退到 `scroll` + rAF + “只扫描可见集合/最近 N 个”策略

**Step 2: 验证**
- Run: `npm --prefix webclipper run test`
- 手工：超长对话滚动时高亮仍可用且无明显卡顿。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Run: `git add webclipper/src/ui/conversations/chat-outline/useChatOutlineActiveIndex.ts`
- Run: `git commit -m "perf: P3-T3 - 优化AI chat目录高亮性能与稳定性"`

---

<a id="p3-t4"></a>
## P3-T4 文档与验收：更新 deepwiki/AGENTS 要点并补齐手工验收清单

**Files:**
- Modify: `.github/deepwiki/modules/webclipper.md`（新增“会话详情目录面板”说明与边界）
- (Optional) Modify: `webclipper/src/ui/AGENTS.md`（若需要补充“目录面板属于详情页 overlay UI”的协作约束）

**Step 1: 实现功能**
- 文档只写不易漂移的原则与入口：
  - 真源判定：仅 `sourceType === 'chat'` 显示
  - 位置与交互：把手常驻内容区右上角，hover 展开 overlay 面板，点击条目滚动居中，高亮按视口中线
- 补齐手工验收清单（可以写在 deepwiki 或另起 doc，但要确保可复用）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
- Expected: 全通过。

**Step 3: 原子提交**
- Run: `git add .github/deepwiki/modules/webclipper.md`
- Run: `git add webclipper/src/ui/AGENTS.md`
- Run: `git commit -m "docs: P3-T4 - 记录AI chat目录面板能力与验收清单"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`）
