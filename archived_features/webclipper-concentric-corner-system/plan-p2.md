# Plan P2 - webclipper-concentric-corner-system

**Goal:** 完成主界面与 comments 关键路径的圆角统一，让 app/popup/inpage 主体视觉达到同心分级。

**Non-goals:** 本 phase 不处理 markdown 内联细节和所有边缘样式（留到 P3 收尾）。

**Approach:**
- 先改最显眼的大容器与核心控件（AppShell/Conversations/Popup/Settings）。
- 将原本过大的“conversations + inpage comments”任务拆成 4 个可原子提交的小任务。
- 全程避免行为变更，只做样式语义替换，并保持已有交互测试可回归。

**Acceptance:**
- app/popup/settings/conversations 主容器与基础控件圆角已统一到 token。
- inpage comments panel、inpage button、nav 入口圆角均接入同心层级。
- `compile + 关键 smoke tests` 通过，且每个任务都可独立提交。

---

<a id="p2-t1"></a>
## P2-T1 统一 app/popup 主容器和 settings 基础控件圆角

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/popup/PopupShell.tsx`
- Modify: `webclipper/src/ui/settings/ui.ts`

**Step 1: 实现**
1. 将主容器 `tw-rounded-3xl/2xl/xl` 收敛为 token 驱动的层级组合。
2. settings 卡片、输入框等基础控件统一到 card/control 级半径。
3. 保持 focus ring、border、hover/active 行为不变。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 编译通过，UI 组件类型不受影响。

**Step 3: 原子提交**
- `refactor: task4 - 统一app与popup主容器圆角层级`

---

<a id="p2-t2"></a>
## P2-T2 统一 conversations 外层容器、列表卡片与标签圆角

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`

**Step 1: 实现**
1. 收敛 conversations 外层容器与列表项卡片的圆角语义到 outer/card 层级。
2. 收敛 warning/source 等标签 chip 的圆角语义到 chip 层级。
3. 不改变列表交互、筛选逻辑与窄屏切换行为。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/conversations-scene-popup-escape.test.ts`
- Expected: 编译通过；场景切换 smoke 测试通过。

**Step 3: 原子提交**
- `refactor: task5 - 统一conversations列表层级圆角`

---

<a id="p2-t3"></a>
## P2-T3 统一 conversations 详情头输入与动作控件圆角

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`

**Step 1: 实现**
1. 收敛详情头输入框、切换按钮、动作按钮到 control/chip 层级 token。
2. 移除与 token 体系冲突的 `tw-rounded-*` 分叉。
3. 保持宽屏/窄屏 header 行为与动作槽位分发一致。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/detail-header-actions.test.ts`
- Expected: 编译通过；详情头动作 smoke 测试通过。

**Step 3: 原子提交**
- `refactor: task6 - 统一conversations详情头控件圆角`

---

<a id="p2-t4"></a>
## P2-T4 统一 inpage comments panel 主体块级圆角层级

**Files:**
- Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`

**Step 1: 实现**
1. 把 panel 内 `14/16/12/10px` 等硬编码迁移到 token（按同心层级分配）。
2. 重点收敛 quote、composer、thread、notice、reply textarea 的层级关系。
3. 处理 sidebar 特化样式（如 thread-quote）时保留层级关系，避免“内外反转”。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts`
- Expected: 编译通过，comments sidebar 关键交互不回归。

**Step 3: 原子提交**
- `refactor: task7 - 统一inpage comments主体圆角层级`

---

<a id="p2-t5"></a>
## P2-T5 统一 inpage button 与 nav 入口圆角语义

**Files:**
- Modify: `webclipper/src/ui/styles/inpage-button.css`
- Modify: `webclipper/src/ui/shared/nav-styles.ts`

**Step 1: 实现**
1. 将 inpage button 与 nav 入口项圆角对齐到 control/chip 层级 token。
2. 清理旧注释和旧半径语义，避免继续参考历史固定 px 值。
3. 保持 hover/active/focus 与可点击面积不变。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/inpage-button-position-ratio.test.ts`
- Expected: 编译通过；inpage button 定位 smoke 测试通过。

**Step 3: 原子提交**
- `refactor: task8 - 统一inpage按钮与nav入口圆角`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
