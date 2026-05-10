# Plan P1 - webclipper-concentric-corner-system

**Goal:** 建立 WebClipper 圆角单一真源与同心分级规则，并完成按钮/菜单基础层收敛。

**Non-goals:** 本 phase 不做全量页面替换，不处理所有历史组件，仅先搭建全局基线。

**Approach:**
- 先在 `tokens.css` 增加可复用 `--radius-*` 语义层级，作为后续唯一来源。
- 优先收敛共享按钮与菜单，因为它们被 app/popup/inpage 高频复用。
- 同步补充 UI 规范，避免后续改动重新引入硬编码半径。

**Acceptance:**
- 已定义并落地 `--radius-*` tokens（含 host/shadow 生效路径）。
- `buttons.css`、`button-styles.ts`、`MenuPopover.tsx` 不再使用散落硬编码半径（除 pill）。
- 文档明确“同心分级圆角”与“禁止裸半径”约束。

---

<a id="p1-t1"></a>
## P1-T1 建立圆角 token 真源并定义同心分级

**Files:**
- Modify: `webclipper/src/ui/styles/tokens.css`
- Modify: `webclipper/src/ui/comments/shadow-styles.ts`（仅确认 token 透传行为，无额外主题分叉）

**Step 1: 实现**
1. 在 `tokens.css` 新增圆角 token（建议：`--radius-outer`、`--radius-card`、`--radius-control`、`--radius-chip`、`--radius-inline`、`--radius-pill`）。
2. 明确分级关系用于同心视觉：外层 > 卡片 > 控件 > chip > 内联；圆形元素使用 pill。
3. 保持现有 app/popup/shadow 样式注入链路不变，确保 token 在 shadow DOM 同样可用。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 编译通过。

**Step 3: 原子提交**
- `feat: task1 - 建立webclipper同心圆角tokens真源`

---

<a id="p1-t2"></a>
## P1-T2 收敛按钮与菜单到 radius tokens

**Files:**
- Modify: `webclipper/src/ui/styles/buttons.css`
- Modify: `webclipper/src/ui/shared/button-styles.ts`
- Modify: `webclipper/src/ui/shared/MenuPopover.tsx`

**Step 1: 实现**
1. 将 `buttons.css` 中 `12px/11px/999px` 替换为对应 radius token。
2. `button-styles.ts` 的菜单 panel 圆角类（如 `tw-rounded-[14px]`）改为 token 驱动策略（可通过 CSS class 或变量桥接）。
3. `MenuPopover.tsx` 背板圆角（当前 `tw-rounded-[18px]`）改为统一层级，避免与 panel 不同心。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts`
- Expected: 编译通过，评论侧栏相关测试不回归。

**Step 3: 原子提交**
- `refactor: task2 - 按钮与菜单圆角收敛到tokens`

---

<a id="p1-t3"></a>
## P1-T3 补充圆角规范文档与扫描约束

**Files:**
- Modify: `webclipper/src/ui/AGENTS.md`
- Modify: `webclipper/AGENTS.md`（仅补充导航/约束，避免与 UI 规范冲突）

**Step 1: 实现**
1. 在 UI 规范中新增“同心圆角分级”章节，写明 token 名称、适用层级和禁用项。
2. 增加审计命令示例：扫描 `border-radius: <px>` / `tw-rounded-[...]` 的剩余硬编码。
3. 强调新增组件默认使用 token，不允许新增裸半径。

**Step 2: 验证**
- Run: `rg -n "border-radius:\s*[0-9]|tw-rounded-\[" webclipper/src/ui webclipper/src/entrypoints`
- Expected: 输出结果符合“可解释且可继续收敛”的基线，不出现新增无主来源半径。

**Step 3: 原子提交**
- `docs: task3 - 补充webclipper同心圆角工程规范`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
