# Plan P3 - webclipper-concentric-corner-system

**Goal:** 收敛剩余边缘组件圆角并完成最终验证，让全插件圆角体系可审计、可持续。

**Non-goals:** 本 phase 不引入新的视觉主题，不重写组件架构。

**Approach:**
- 先处理边缘但高可见组件（tooltip / inpage tip / locate highlight）。
- 再收敛 markdown 气泡内联块半径，避免内容区仍出现“旧体系”。
- 最后执行全量扫描与验证链，形成可交付闭环。

**Acceptance:**
- 目标路径中硬编码半径仅剩白名单（pill 或第三方库不可控场景）。
- ChatMessageBubble 与 tooltip/inpage tip 的圆角语义接入 token。
- 核心门禁 `compile + targeted tests + build` 完成；全量 `test` 结果有记录且能说明与本 feature 的相关性。

---

<a id="p3-t1"></a>
## P3-T1 统一 tooltip/inpage tip/locate highlight 边缘圆角

**Files:**
- Modify: `webclipper/src/ui/styles/tooltip.css`
- Modify: `webclipper/src/ui/styles/inpage-tip.css`
- Modify: `webclipper/src/ui/comments/locate.ts`

**Step 1: 实现**
1. 将 tooltip 与 inpage tip 的圆角替换为 token。
2. 将 locate highlight 的注入样式半径接入 token 或统一常量来源。
3. 保持提示气泡动画、边框和可读性不变。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 编译通过。

**Step 3: 原子提交**
- `refactor: task9 - 统一边缘提示组件圆角策略`

---

<a id="p3-t2"></a>
## P3-T2 统一 ChatMessageBubble markdown 内联圆角

**Files:**
- Modify: `webclipper/src/ui/shared/ChatMessageBubble.tsx`

**Step 1: 实现**
1. 将气泡容器、code/kbd/pre/img 等半径硬编码收敛到 token 映射。
2. 保持 markdown 排版与对比度不变，避免引入内容区视觉退化。
3. 兼容用户气泡与系统气泡两种 tone。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/chat-message-bubble.test.ts tests/smoke/inpage-tip-speech-bubble.test.ts`
- Expected: 编译通过；与本 task 相关的 targeted tests 通过。

**Step 3: 原子提交**
- `refactor: task10 - 统一chat markdown内容块圆角`

---

<a id="p3-t3"></a>
## P3-T3 完成圆角硬编码收敛扫描与最终验证链

**Files:**
- Modify: `webclipper/src/ui/AGENTS.md`（如需补充最终约束）
- Modify: `webclipper/AGENTS.md`（如需补充仓库级导航）

**Step 1: 实现**
1. 运行硬编码扫描并收敛剩余目标路径：
   - `rg -n "border-radius:\s*[0-9]|tw-rounded-\[" webclipper/src/ui webclipper/src/entrypoints`
2. 对无法替换的白名单项记录原因（例如 pill 或第三方组件）。
3. 完成最终验证链并写入执行备注。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/inpage-tip-speech-bubble.test.ts`
- Run: `npm --prefix webclipper run build`
- Run: `npm --prefix webclipper run test`（非阻塞；用于记录全量回归现状）
- Expected: 核心门禁通过；若全量 test 失败，需在执行备注中列出失败用例并标注是否与圆角改动相关。

**Step 3: 原子提交**
- `chore: task11 - 完成圆角统一收敛与最终验证`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
