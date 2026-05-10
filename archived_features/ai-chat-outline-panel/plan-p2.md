# Plan P2 - ai-chat-outline-panel

**Goal:** 目录面板支持滚动自动高亮：始终高亮“最接近视口中线”的 user 消息条目，并在 app + popup、宽屏/窄屏下稳定工作。

**Non-goals:**
- 不做 LLM/语义分段。
- 不做跨会话搜索/过滤。

**Approach:**
- 在 user 消息渲染处提供稳定的 DOM 锚点（wrapper element + ref map）。
- 解析“详情滚动容器 root”（`.route-scroll`）作为 `IntersectionObserver.root`，避免把 viewport 当 root 导致窄屏/内嵌滚动不准。
- 用 IO 维护“可见候选集合”，再按“中线距离最小”选 active；避免每次 scroll 全量遍历所有消息。

**Acceptance:**
- 滚动详情时目录高亮实时更新，且高亮条目对应“最接近视口中线”的 user 消息。
- 大对话滚动不卡顿（不会每次 scroll 做全量 DOM 测量）。

---

<a id="p2-t1"></a>
## P2-T1 为 user 消息渲染可定位锚点并解析滚动容器 root

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- (Optional) Add: `webclipper/src/ui/shared/dom-scroll-root.ts`

**Step 1: 实现功能**
- 在 `ConversationDetailPane` 的消息列表渲染中：
  - 只对 `role === 'user'` 的消息，在 `ChatMessageBubble` 外包一层 wrapper（若 P1 已引入 wrapper+ref，此处直接复用并扩展同一层，不要二次包裹）：
    - `data-chat-outline-index="<1..N>"`
    - `data-chat-outline-message-id="<id>"`
    - `ref` 存入 `Map<messageId, HTMLElement>`（用于点击跳转）
  - 注意：wrapper 必须 `tw-min-w-0` 且不改变现有布局/间距（保持 bubble 外观不变）。
- 解析滚动容器 root：
  - 从消息列表 root（已有 `messagesRootRef`）向上找最近的 `.route-scroll` 容器作为滚动 root。
  - 若找不到则回退 `null`（IO 以 viewport 为 root 也能工作，但精度较差；此为兜底）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 无错误。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Run: `git commit -m "feat: P2-T1 - 为AI chat目录提供user消息DOM锚点与滚动root"`

---

<a id="p2-t2"></a>
## P2-T2 实现 IntersectionObserver 驱动的中线高亮选择器

**Files:**
- Add: `webclipper/src/ui/conversations/chat-outline/useChatOutlineActiveIndex.ts`
- Add: `webclipper/src/ui/conversations/chat-outline/active-index.ts`（纯函数：中线选择逻辑，便于稳定单测）
- Add: `webclipper/tests/unit/chat-outline-active-index.test.ts`（围绕纯函数写断言；hook 层只做集成拼装）

**Step 1: 实现功能**
- 设计 hook：
  - 输入：
    - `root: Element | null`（滚动容器 root）
    - `userMessageEls: HTMLElement[]`（按 index 顺序）
    - `messagesRootEl?: HTMLElement | null`（消息列表 root，用于计算“可见消息区”中线，排除 sticky header 干扰）
  - 输出：
    - `activeIndex: number | null`
- 逻辑建议：
  1. `IntersectionObserver` 只观察 `userMessageEls`，阈值用 `[0, 0.1, 0.25, 0.5, 0.75, 1]` 或更少，维护 `visibleSet`（intersectionRatio > 0）。
  2. 在 observer 回调与 root scroll 时（`requestAnimationFrame` 节流）：
     - 计算“可见消息区”的中线 `centerY`：
       - `rootRect = (root ?? document.documentElement).getBoundingClientRect()`
       - `messagesRect = messagesRootEl?.getBoundingClientRect()`（来自 `ConversationDetailPane` 的消息列表 root ref）
       - `visibleTop = max(rootRect.top, messagesRect?.top ?? rootRect.top)`（用于自然排除 sticky header 覆盖区域）
       - `centerY = (visibleTop + rootRect.bottom) / 2`
     - 在 `visibleSet` 中选取 `abs(elCenterY - centerY)` 最小者作为 active
  3. 若 `visibleSet` 为空（例如快速滚动瞬间）：允许回退到最近一次 active；或从全部元素中选最接近（成本较高，谨慎）。
- 输出的 `activeIndex` 与目录条目 index 一致（1-based），方便直接高亮。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- chat-outline-active-index`
- Expected: 测试通过（至少覆盖“中线选择”“visibleSet 变化”“root 为空时兜底”）。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/chat-outline/useChatOutlineActiveIndex.ts`
- Run: `git add webclipper/tests/unit/chat-outline-active-index.test.ts`
- Run: `git commit -m "feat: P2-T2 - 增加AI chat目录的中线高亮选择器"`

---

<a id="p2-t3"></a>
## P2-T3 把自动高亮接回目录面板（包含滚动/点击状态一致性）

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`

**Step 1: 实现功能**
- 在 `ConversationDetailPane`：
  - 组装 `userMessageEls`（按条目 index 顺序）
  - 调用 `useChatOutlineActiveIndex({ root, userMessageEls, messagesRootEl })`
  - 将 `activeIndex` 传给 `ChatOutlinePanel`
- 点击跳转与高亮一致性：
  - 点击条目后可以立即把 `activeIndex` 置为该条目（乐观更新），避免“滚动动画期间高亮乱跳”
  - 动画结束后由 observer 收敛到真实 active（无需显式监听 scrollend，observer 会自然收敛）

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Expected: 编译与单测通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Run: `git add webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- Run: `git commit -m "feat: P2-T3 - 接入AI chat目录自动高亮"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`npm --prefix webclipper run compile && npm --prefix webclipper run test`）
