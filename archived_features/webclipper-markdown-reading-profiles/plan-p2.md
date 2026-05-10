# Plan P2 - webclipper-markdown-reading-profiles

**Goal:** 在同一协议下扩展 `Notion-like` 与 `Book-like`，保证“风格差异明显 + 可读性底线一致”。

**Non-goals:** 本 phase 不接入设置页，不改 storage 键，不新增第四种 profile。

**Approach:**
- `notion` 与 `book` 只通过协议 preset 扩展，不增加第二套渲染入口。
- 每个 profile 都必须通过同一可读性守护断言（measure、line-height、断行、节点完整性）。
- 在 light/dark + user/assistant 双维度下做回归，避免“只在一种背景下可读”。

**Acceptance:**
- `medium/notion/book` 三套 profile 都可解析并可渲染。
- 三套 profile 均覆盖正文、标题、列表、引用、代码、表格、图片说明。
- `ChatMessageBubble` 在 profile 变化时只切样式，不改 markdown 语义输出。
- 三套 profile 的关键契约有测试覆盖。

---

<a id="p2-t1"></a>
## P2-T1 在协议下实现 Notion-like profile（紧凑但不牺牲可读性）

**Files:**
- Modify: `webclipper/src/ui/shared/markdown-reading-profile-presets.ts`
- Modify: `webclipper/src/ui/shared/ChatMessageBubble.tsx`
- Modify: `webclipper/tests/unit/markdown-reading-profiles.test.ts`

**Step 1: 实现**
1. 新增 `notion` preset（紧凑层级、信息密度更高），但保持可读性底线：
   - 正文行高不低于 `1.5`
   - measure 不超出约束
   - 长链接可断行
2. 保持与现有 renderer 输出 class 兼容（`syncnos-md-image*`、katex、table）。
3. 不改 resolver 默认值（仍为 `medium`）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts`
- Expected: `notion` 的契约与渲染测试通过。

**Step 3: 原子提交**
- `feat: task4 - 新增notion-like阅读风格并保持可读性底线`

---

<a id="p2-t2"></a>
## P2-T2 在协议下实现 Book-like profile（长文沉浸）

**Files:**
- Modify: `webclipper/src/ui/shared/markdown-reading-profile-presets.ts`
- Modify: `webclipper/src/ui/styles/tokens.css`（如需补 serif / CJK fallback token）
- Modify: `webclipper/tests/unit/markdown-reading-profiles.test.ts`

**Step 1: 实现**
1. 新增 `book` preset（衬线正文、更宽松节奏、长文优先）。
2. 明确 CJK fallback 字体栈，避免中英文混排风格冲突。
3. 保持代码块、链接、引用在 dark/light 双模式都可读，不出现低对比回归。
4. 不增加新渲染分支，继续复用统一协议入口。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts tests/unit/markdown-renderer.test.ts`
- Expected: `book` 契约通过，bubble/renderer 不回归。

**Step 3: 原子提交**
- `feat: task5 - 新增book-like阅读风格并完善长文排版节奏`

---

<a id="p2-t3"></a>
## P2-T3 建立三风格回归矩阵（light/dark + user/assistant）

**Files:**
- Add: `webclipper/tests/smoke/markdown-reading-profiles-matrix.test.ts`
- Modify: `webclipper/tests/unit/markdown-reading-profiles.test.ts`
- Modify: `.github/features/webclipper-markdown-reading-profiles/audit-p2.md`（执行期回填）

**Step 1: 实现**
1. 建立三风格统一断言：id 解析、unknown 回退、preset 字段完整。
2. 增加矩阵检查：
   - `profile x bubbleRole`（3 x 2）
   - light/dark 两套 token 是否存在可读性兜底
3. 把高风险点写入 audit：标题层级、blockquote、pre/table 横滚边界、图片说明换行。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/markdown-reading-profiles-matrix.test.ts`
- Expected: P2 目标测试通过，三风格矩阵稳定。

**Step 3: 原子提交**
- `test: task6 - 建立三风格可读性回归矩阵`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
