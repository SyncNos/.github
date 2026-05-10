# Plan P1 - webclipper-markdown-reading-profiles

**Goal:** 以“协议先行 + 可读性硬指标”落地 `Medium-like`，并把当前散落在 `ChatMessageBubble` 的排版逻辑收敛成可扩展底座。

**Non-goals:** 本 phase 不实现 Notion-like / Book-like，不接入设置页切换 UI。

**Approach:**
- 先定义 profile 协议（id、回退、排版 token 契约），让风格能力可验证。
- 再把 `ChatMessageBubble` 从单一硬编码 class 串改为“结构样式 + profile token”组合。
- 最后补可读性守护测试（measure/reflow/text-spacing/contrast 代理检查），保证 P2/P3 扩展不回退体验。

**UX Guardrails（本 phase 必达）:**
- 正文行长目标 `55-75ch`，硬上限 `<=80ch`；CJK 目标 `<=40` 字符。
- 行高不低于 `1.5`，正文推荐 `1.6+`。
- `320px` 宽度下正文不出现双向滚动（`pre/table` 允许局部横滚）。
- 长链接与长 token 不撑破容器（`overflow-wrap` 生效）。

**Acceptance:**
- 存在稳定协议：`medium | notion | book` + `unknown -> medium`。
- `Medium-like` 已在详情 markdown 生效，并覆盖正文、标题、列表、引用、代码、图片说明、表格。
- `ChatMessageBubble` 不再依赖单一大段硬编码 profile class。
- `compile + targeted tests` 通过。

---

<a id="p1-t1"></a>
## P1-T1 定义阅读风格协议契约与 resolver 回退

**Files:**
- Add: `webclipper/src/services/protocols/markdown-reading-profiles.ts`
- Add: `webclipper/src/ui/shared/markdown-reading-profile-presets.ts`
- Add: `webclipper/tests/unit/markdown-reading-profiles.test.ts`

**Step 1: 实现**
1. 定义稳定协议：`MarkdownReadingProfileId = 'medium' | 'notion' | 'book'`。
2. 定义 `MarkdownReadingProfileSpec`（最少包含：`id`、`labelKey`、`font stack`、`font size`、`line-height`、`paragraph gap`、`measure`、`heading scale`、`code scale`）。
3. 实现 resolver：输入任意字符串，输出合法 id；未知值统一回退 `medium`。
4. 协议层只做“值契约 + 回退”，preset 层做“class/token 映射”。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts`
- Expected: 协议类型、回退逻辑、preset 完整性断言通过。

**Step 3: 原子提交**
- `refactor: task1 - 建立markdown阅读风格协议契约与回退`

---

<a id="p1-t2"></a>
## P1-T2 落地 Medium-like 排版并替换硬编码 profile 样式

**Files:**
- Modify: `webclipper/src/ui/shared/ChatMessageBubble.tsx`
- Modify: `webclipper/src/ui/shared/markdown-reading-profile-presets.ts`
- Modify: `webclipper/src/ui/styles/tokens.css`（如需补充阅读字体栈/token）
- Modify: `webclipper/tests/unit/chat-message-bubble.test.ts`

**Step 1: 实现**
1. 将 markdown 样式拆为两层：
   - 固定结构层（语义节点：`p/h1-6/ul/ol/blockquote/pre/table/img`）
   - profile 排版层（字号、行高、段距、行长、链接与引用对比）
2. 在 preset 层实现 `medium`，并显式满足：正文可读节奏、合理 measure、长 token 断行。
3. 保持 markdown 解析语义不变，不改 `syncnos-md-image*` 与 renderer 输出约定。
4. user/assistant 两类气泡都验证链接与正文可读，不引入“只在一种气泡可读”的回归。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/chat-message-bubble.test.ts tests/unit/markdown-renderer.test.ts`
- Expected: 组件与 renderer 相关测试通过；无语义回归。

**Step 3: 原子提交**
- `feat: task2 - 落地medium-like阅读排版并移除硬编码profile样式`

---

<a id="p1-t3"></a>
## P1-T3 增加可读性守护测试与 P1 验证闭环

**Files:**
- Add: `webclipper/tests/smoke/markdown-reading-profiles-smoke.test.ts`
- Modify: `webclipper/tests/unit/markdown-reading-profiles.test.ts`
- Modify: `webclipper/tests/unit/chat-message-bubble.test.ts`
- Modify: `.github/features/webclipper-markdown-reading-profiles/audit-p1.md`（执行期回填）

**Step 1: 实现**
1. 增加“守护断言”而不追求视觉快照：
   - unknown profile 回退
   - profile token 完整字段存在
   - 关键节点（标题、blockquote、pre、table、image-link）均由统一 profile 入口驱动
2. 增加 reflow / long-token 防回归检查（通过 class/token 断言验证策略已挂载）。
3. 在 audit 草稿记录 P1 体验风险与证据位置。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/markdown-reading-profiles-smoke.test.ts`
- Expected: P1 目标测试全部通过。

**Step 3: 原子提交**
- `test: task3 - 增加markdown可读性守护测试基线`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
