# Plan P3 - webclipper-comments-sidebar-panel-refactor

**Goal:** 以“TS 规范 + 可测性 + 去重收敛”为目标完成最后一轮质量提升，并补齐验证链。

**Non-goals:** 不引入新功能，不改用户可见行为，不做跨 feature 的大范围 UI 重写。

**Approach:**
- 在不破坏行为的前提下，把“重复 normalize helpers”与“any 泛滥点”收敛到少数边界文件。
- 为拆分出的纯逻辑（clamp/normalize 等）补充 vitest 单测，确保重构安全。

**Acceptance:**
- 重复 helper 显著减少；`any` 使用被压缩并集中。
- `npm --prefix webclipper run lint/compile/test/build` 全通过。

---

<a id="p3-t1"></a>
## P3-T1 收敛：消除重复 normalize helpers（safeString/normalizeHttpUrl/normalizeConversationId）

**Files:**
- Add: `webclipper/src/services/url-cleaning/http-url.ts`
- Add: `webclipper/src/services/shared/numbers.ts`
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/*`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- (Optional) Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`

**Step 1: 实现**
1. 提供最小 API（避免臃肿）：
   - `normalizeHttpUrl(raw: unknown): string`（放在 `services/url-cleaning/http-url.ts`，与现有 `hostname.ts` 同目录）
   - `safeString(value: unknown): string`（仅在需要时暴露；否则保持 file-local helper）
   - `normalizePositiveInt(value: unknown): number | null`（放在 `services/shared/numbers.ts`，替代重复的 `normalizeConversationId`）
2. 将与 comments sidebar 相关的重复实现替换为 shared 引用，确保行为一致（hash 清理、http/https 校验、trim 等）。

**Verify**
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task1 - 收敛 normalize helpers`

---

<a id="p3-t2"></a>
## P3-T2 规范：减少 any，集中边界 cast，并补齐类型守卫

**Files:**
- Modify: `webclipper/src/ui/comments/threaded-comments-panel/*`

**Step 1: 实现**
1. 将 DOM/事件边界统一改用 `unknown`，用小型 type guard 处理（例如 `isHTMLElement`/`isHTMLButtonElement`）。
2. 必要的 `as any` 只能出现在“平台边界/DOM 边界”处，并集中在 1–2 个文件（例如 `dom-guards.ts`）。
3. 避免在业务逻辑中散落 `as any`（例如 chat-with action mapping、dataset/attributes 读取）。

**Verify**
- `npm --prefix webclipper run lint`
- `npm --prefix webclipper run compile`

**Commit**
- `refactor: task2 - threaded comments panel 类型与守卫收敛`

---

<a id="p3-t3"></a>
## P3-T3 测试：为拆分出的纯逻辑补 vitest 单测

**Files:**
- Add: `webclipper/src/ui/comments/threaded-comments-panel/__tests__/resize.test.ts`（示例）
- Add: `webclipper/src/services/url-cleaning/__tests__/http-url.test.ts`（示例）
- Add: `webclipper/src/services/shared/__tests__/numbers.test.ts`（示例）

**Step 1: 实现**
1. 优先测试纯函数（无需 jsdom）：
   - width clamp（min/max/viewport cap）
   - normalizeHttpUrl（非法协议、hash 清理、空值）
2. 仅在必要时引入 jsdom 来测 DOM helper（避免测试臃肿）。

**Verify**
- `npm --prefix webclipper run test`

**Commit**
- `test: task3 - 补齐 threaded comments panel 关键纯逻辑单测`

---

<a id="p3-t4"></a>
## P3-T4 最终验证：lint + compile + test + build

**Commands**
- `npm --prefix webclipper run lint`
- `npm --prefix webclipper run format:check`
- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test`
- `npm --prefix webclipper run build`

**Commit**
- `chore: task4 - threaded comments panel 重构最终验证`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Focus:
  - 是否引入了新的跨层依赖或 restricted import
  - `ui/comments/threaded-comments-panel/` 的文件体积与边界是否符合“不过度抽象 + 不臃肿”
  - 行为回归清单（dock/resize/chatwith/locate/快捷键/删除确认）是否通过最小人工冒烟
