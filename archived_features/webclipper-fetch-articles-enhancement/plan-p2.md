# Plan P2 - webclipper-fetch-articles-enhancement

**Goal:** 参考 obsidian-clipper，把“通用抽取（Defuddle）+ 通用转换（Turndown 规则）”串成主干链路，并补齐 HTML 清洗/URL 绝对化，提升跨站稳定性。

**Non-goals:** 本 phase 不做“阅读模式 UI”；不引入 Obsidian 那套模板变量体系；不做 Discourse `.json` 兜底。

**Approach:** 在 P1 的 Turndown 基础上，引入 Defuddle 做通用正文抽取（对齐 Obsidian 的 extract 策略），再补 HTML 清洗与 URL 绝对化，最后调整抽取顺序与稳定等待策略，使其更“通用”而非“站点特判”。

**Clean-as-you-go rule:**
- P2 以“主干唯一”为目标：逐步删除手拼 markdown 的旁路实现，让 `contentHTML -> (clean) -> Turndown` 成为唯一出口。

**Acceptance:**
- Discourse cooked 中的列表、引用、代码块、details/summary 等在 Markdown 中保持结构可读。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。
- 站点特化能力不回归：微信公众号图集、小红书 note、Bilibili opus 相关回归用例均通过。

---

<a id="p2-t1"></a>
## P2-T1 引入Defuddle通用抽取（对齐obsidian-clipper的extract策略）

**Files:**
- Modify: `webclipper/package.json`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/src/collectors/web/article-extract/sites/index.ts`
- (Optional) Add: `webclipper/src/collectors/web/article-extract/defuddle.ts`

**Step 1: 实现功能**
- 引入依赖：`defuddle`（优先仅使用其“抽取正文 HTML + 元信息”的部分）。
- 在 content 侧为 `extractWebArticleFromCurrentPage()` 增加 `extractByDefuddle()` 分支：
  - 输入：当前 `document`（或 clone）与 `location.href`
  - 输出：`title/author/publishedAt/contentHTML/textContent`（`contentMarkdown` 仍由 Turndown 转换器生成）
- 抽取顺序建议：site spec（保留）→ Discourse OP（保留，尽量轻量）→ Defuddle → Readability → fallbackExtract。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Expected: 依赖安装后编译通过；Defuddle 分支不会导致现有 smoke tests 变脆弱。

**Step 3: 原子提交**
Run: `git add webclipper/package.json webclipper/src/collectors/web/article-extract/engine.ts webclipper/src/collectors/web/article-extract/defuddle.ts`

Run: `git commit -m "feat: P2-T1 - 引入Defuddle作为通用文章抽取器"`

---

<a id="p2-t2"></a>
## P2-T2 加入HTML清洗与URL绝对化步骤（脚本/样式/内联style等）

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/markdown-turndown.ts`
- (Optional) Add: `webclipper/src/collectors/web/article-extract/html-clean.ts`

**Step 1: 实现功能**
- 参考 `obsidian-clipper-main/src/content.ts`：在转换前做通用清洗：
  - 移除 `script/style`
  - 移除所有元素的 `style` 属性
  - 将相对 `href/src/srcset` 绝对化（基于 `baseHref`）
- 清洗策略必须作用于“fragment HTML”（例如 Discourse `.cooked`），不要依赖 `document.documentElement.outerHTML` 才能工作。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-discourse-op-onebox-regression`

Expected: onebox/图片/链接都能产出可用 Markdown；相对链接不再丢失或变成空。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/markdown-turndown.ts webclipper/src/collectors/web/article-extract/html-clean.ts`

Run: `git commit -m "refactor: P2-T2 - 增加HTML清洗与URL绝对化提升Markdown质量"`

---

<a id="p2-t3"></a>
## P2-T3 调整extract顺序与稳定等待策略（SPA/hydration通用）

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Add: `webclipper/tests/smoke/article-extract-markdown-complex.test.ts`

**Step 1: 实现功能**
- 重排 `engine.ts` 的抽取顺序（以通用优先）并收敛等待策略：
  - 先做“DOM 稳定等待”（有上限、可早停）
  - 再 site spec（少量必要保留：小红书 note、Bilibili opus）
  - 再 Discourse OP（首贴 `.cooked`）
  - 再 Defuddle
  - 再 Readability
  - 最后 fallbackExtract
- 新增 complex smoke test：覆盖 onebox、details、blockquote、code、table，锁住 Turndown 规则集的回归风险。

**Step 2: 验证**
Run: `npm --prefix webclipper run test`

Expected: 新增测试通过，并能覆盖关键规则回归。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/engine.ts webclipper/tests/smoke/article-extract-markdown-complex.test.ts`

Run: `git commit -m "refactor: P2-T3 - 重排extract顺序并新增复杂Markdown回归用例"`

---

<a id="p2-t4"></a>
## P2-T4 让site spec输出走“统一HTML→Markdown转换器”（保留适配但消除多条markdown生成路径）

**Why:**
- 当前 `extractBySiteSpec()` 自己拼 `contentMarkdown`（图片 blocks + 纯文本），这会让“通用转换器（Turndown）”变成旁路，难以做到真正的“完全重构/主干唯一”。
- 本 task 的目标是：**site spec 仍然存在且优先级不变**，但 markdown 生成也统一走 Turndown 规则，便于后续维护与回归。

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/site-spec-extractor.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/tests/smoke/article-extract-bilibili-opus.test.ts`
- Modify: `webclipper/tests/unit/article-extract-xiaohongshu.test.ts`

**Step 1: 实现功能**
- 将 `extractBySiteSpec()` 改为产出“结构化 HTML + 元信息”：
  - `contentHTML`：由图片 blocks + 文本段落拼接而成（保持现有行为）
  - `textContent`：仍保留纯文本（用于空内容兜底）
  - `contentMarkdown`：交由 engine 的统一转换器生成（不要在 site-spec-extractor 里再手写 markdown）
- 在 `engine.ts` 中将 site spec 分支改为：
  - 先拿到 `contentHTML`
  - 再用 Turndown 转成 `contentMarkdown`
  - 必要时保留“空 markdown → 退回 textContent”的安全兜底

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-bilibili-opus`

Run: `npm --prefix webclipper run test -- article-extract-xiaohongshu`

Expected: 现有用例仍通过（允许 markdown 细节变化，但必须保留图片与正文，不得丢内容）。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/site-spec-extractor.ts webclipper/src/collectors/web/article-extract/engine.ts webclipper/tests/smoke/article-extract-bilibili-opus.test.ts webclipper/tests/unit/article-extract-xiaohongshu.test.ts`

Run: `git commit -m "refactor: P2-T4 - 统一site spec到Turndown转换器"`

---

<a id="p2-t5"></a>
## P2-T5 让微信公众号share media图集走“HTML追加 + Turndown统一转换”（消除markdown手拼旁路）

**Why:**
- 当前 Readability 分支会把 `wechatGalleryMarkdown` 直接拼到 markdown 结果里，这是另一条旁路。
- 本 task 的目标是：保留“图集追加”的适配能力，但让 markdown 由 **同一条 Turndown 主干**产出。

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/src/collectors/web/article-extract/sites/wechat.ts`
- Modify: `webclipper/tests/smoke/article-extract-wechat-gallery.test.ts`

**Step 1: 实现功能**
- 只保留 `buildWechatShareMediaGalleryHtml()`（HTML 片段）：
  - engine 在生成 `contentHTML` 时继续把 gallery HTML 追加到正文 HTML 后
  - markdown 完全由 Turndown 从 `contentHTML` 统一转换，不再额外拼接 `wechatGalleryMarkdown`
- 同步删除旁路 API：`buildWechatShareMediaGalleryMarkdown()` 及其所有引用（避免旧代码残留）。
- 保持图片 URL sanitize 行为不变（回归用例覆盖）。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-wechat-gallery`

Expected: 输出依然是 image blocks（不是 table），且图片 URL 被正确清洗。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/engine.ts webclipper/src/collectors/web/article-extract/sites/wechat.ts webclipper/tests/smoke/article-extract-wechat-gallery.test.ts`

Run: `git commit -m "refactor: P2-T5 - 微信图集并入Turndown统一转换"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（至少 `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`）
