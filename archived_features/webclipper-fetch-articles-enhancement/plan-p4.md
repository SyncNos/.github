# Plan P4 - webclipper-fetch-articles-enhancement

**Goal:** 删除 fetch articles 链路中“旧的手写 HTML→Markdown 转换器”与其兼容兜底，确保转换主干唯一（`contentHTML -> cleanHtmlFragment -> Turndown -> Markdown`），并保持最小可靠兜底（`textContent`）。

**Non-goals:**
- 不触碰 AI 对话 collectors（例如 `zai/kimi/doubao/...` 里同名的 `htmlToMarkdown` 函数）。
- 不删除 fetch articles 链路中“仍有兼容价值”的逻辑：Discourse `/1` 回退、content message “no receiver” 重试、Readability 按需注入重试、以及微信/小红书 hydration 等待（这些仅做保留，不在本 phase 清理范围内）。
- 本 phase 仅针对 **web article fetch** 链路中“可确定无用/重复”的 legacy 与 re-export 清理。

**Acceptance:**
- `webclipper/src/collectors/web/article-extract/markdown.ts` 被删除，且 `webclipper/src/collectors/web/article-extract/**` 范围内不再引用 `htmlToMarkdown()`。
- Markdown 生成兜底统一为：`htmlToMarkdownTurndown(...) || normalizeText(textContent)`（不再出现 “Turndown || legacy converter || text” 三段链路）。
- `extractBySiteSpec()` 不再返回 `contentMarkdown: ''` 这类占位字段；site spec 仅负责产出 `contentHTML/textContent/meta`，markdown 由 engine 统一生成。
- `ARTICLE_FETCH_SITE_SPECS` 的导出链路收敛：`article-fetch-sites/index.ts` 为唯一真源；`article-extract/sites` 仅做一跳 re-export，不再“转发再转发”。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。

---

<a id="p4-t1"></a>
## P4-T1 删除旧手写 HTML→Markdown 转换器（字节级清理）

**Files:**
- Delete: `webclipper/src/collectors/web/article-extract/markdown.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/src/collectors/web/article-extract/sites/discourse.ts`
- Modify: `.github/deepwiki/modules/webclipper.md`

**Step 1: 实现功能**
- 在 `engine.ts` 与 `sites/discourse.ts` 中移除 `htmlToMarkdown()`：
  - 删除对应 import
  - 将所有 `htmlToMarkdownTurndown(...) || htmlToMarkdown(...) || normalizeText(text)` 改为 `htmlToMarkdownTurndown(...) || normalizeText(text)`
- 删除文件：`webclipper/src/collectors/web/article-extract/markdown.ts`
- 确保“字节级清理”（无残留引用）：
  - `rg -n "htmlToMarkdown\\(" webclipper/src/collectors/web/article-extract`
  - `rg -n "htmlToMarkdown\\(" webclipper/src/collectors/web/article-extract/engine.ts` → 0 命中
  - `rg -n "htmlToMarkdown\\(" webclipper/src/collectors/web/article-extract/sites/discourse.ts` → 0 命中
  - `rg -n "@collectors/web/article-extract/markdown" webclipper/src/collectors/web`
  - `rg -n "article-extract/markdown" webclipper/src/collectors/web`（防止存在非 alias import 形式）
  - 预期：均为 0 命中（注意：其它 collectors 目录的同名函数不在本 task 范围内）
- 更新 deepwiki（避免文档漂移）：
  - 将 “兼容兜底：回退旧 htmlToMarkdown()” 改为 “兜底：回退 textContent”
  - 将 “下一步：减少旧 htmlToMarkdown 兜底命中率” 改为 “已移除旧转换器，兜底仅为 textContent”
  - 同步修正文档中关于 Readability 注入的旧表述：统一为“按需注入重试一次”（与当前实现一致；尤其是表格里仍写“手动抓取时注入 readability.js”的那行）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected:
- Discourse / WeChat / 小红书 / Bilibili 的既有回归用例仍通过。
- `contentMarkdown` 仍能稳定产出；若 Turndown 产出空，`textContent` 兜底可用。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/engine.ts webclipper/src/collectors/web/article-extract/sites/discourse.ts .github/deepwiki/modules/webclipper.md`

Run: `git rm webclipper/src/collectors/web/article-extract/markdown.ts`

Run: `git commit -m "refactor: P4-T1 - 删除fetch articles旧htmlToMarkdown转换器兜底"`

---

<a id="p4-t2"></a>
## P4-T2 移除 site spec 返回结构中的 `contentMarkdown` 占位字段（收敛职责）

**Why:**
- 当前 `extractBySiteSpec()` 返回 `contentMarkdown: ''`，这是历史遗留占位字段。
- 现状已经是：site spec 只负责“结构化 HTML 抽取”，markdown 由 `engine.ts` 统一生成；因此应从 site spec 的返回结构中移除该字段，避免误解/重复职责。

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/site-spec-extractor.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`（仅在类型/解构需要时调整）
- Modify: `webclipper/tests/unit/article-extract-xiaohongshu.test.ts`

**Step 1: 实现功能**
- 调整 `extractBySiteSpec()` 的返回对象：
  - 移除 `contentMarkdown` 字段
  - 保留：`title/author/publishedAt/excerpt/contentHTML/textContent`
- 在 `engine.ts` 的 site spec 分支保持不变语义：
  - `contentMarkdown` 仍由 `htmlToMarkdownTurndown(sitePayload.contentHTML, baseHref) || normalizeText(sitePayload.textContent)` 生成
- “字节级清理”检查：
  - `rg -n "contentMarkdown: ''" webclipper/src/collectors/web/article-extract/site-spec-extractor.ts` → 0 命中
- 更新单测：`article-extract-xiaohongshu` 不再断言 `res.contentMarkdown === ''`（因为字段已移除）；改为只断言 `textContent/contentHTML/author` 等抽取结果。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 既有 site spec 回归用例不变（小红书/Bilibili）。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/site-spec-extractor.ts webclipper/src/collectors/web/article-extract/engine.ts webclipper/tests/unit/article-extract-xiaohongshu.test.ts`

Run: `git commit -m "refactor: P4-T2 - site spec移除contentMarkdown占位字段"`

---

<a id="p4-t3"></a>
## P4-T3 收敛 `ARTICLE_FETCH_SITE_SPECS` 的导出链路（去掉多层 re-export）

**Why:**
- 目前存在 `article-fetch-sites/index.ts` → `article-extract/sites/specs.ts` → `article-extract/sites/index.ts` 的多层转发。
- 目标是：`article-fetch-sites/index.ts` 作为 specs 唯一真源；`article-extract/sites/index.ts` 只做一跳 re-export。

**Files:**
- Delete: `webclipper/src/collectors/web/article-extract/sites/specs.ts`
- Modify: `webclipper/src/collectors/web/article-extract/sites/index.ts`

**Step 1: 实现功能**
- 删除 `sites/specs.ts`
- 调整 `sites/index.ts`：
  - 将 `export { ARTICLE_FETCH_SITE_SPECS } from '@collectors/web/article-extract/sites/specs';`
  - 改为 `export { ARTICLE_FETCH_SITE_SPECS } from '@collectors/web/article-fetch-sites';`
- “字节级清理”检查：
  - `rg -n "article-extract/sites/specs" webclipper/src` → 0 命中

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: engine 仍能拿到 `ARTICLE_FETCH_SITE_SPECS`，现有用例不变。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/sites/index.ts`

Run: `git rm webclipper/src/collectors/web/article-extract/sites/specs.ts`

Run: `git commit -m "refactor: P4-T3 - 收敛ARTICLE_FETCH_SITE_SPECS导出链路"`

---

<a id="p4-t4"></a>
## P4-T4 缩小 `article-extract/sites` 的导出 API 面（移除未被外部使用的 re-export）

**Why:**
- `sites/index.ts` 当前导出 `findDiscourseOpNode`、`isXiaohongshuNotePage`，但它们只在各自模块内部使用。
- 缩小导出面可以降低误用与耦合，让对外 API 更稳定。

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/sites/index.ts`
- (Optional) Modify: `webclipper/src/collectors/web/article-extract/engine.ts`（如有 import 变化）

**Step 1: 实现功能**
- 从 `sites/index.ts` 的 export 列表中移除：
  - `findDiscourseOpNode`
  - `isXiaohongshuNotePage`
- “字节级清理”检查：
  - `rg -n "\\bfindDiscourseOpNode\\b" webclipper/src` 只允许命中 `sites/discourse.ts` 本文件内部
  - `rg -n "\\bisXiaohongshuNotePage\\b" webclipper/src` 只允许命中 `sites/xiaohongshu.ts` 本文件内部

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 无外部依赖这些导出；编译测试通过。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/sites/index.ts`

Run: `git commit -m "refactor: P4-T4 - 缩小article-extract sites导出API面"`

---

## Phase Audit

- Audit file: `audit-p4.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（至少 `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`）
