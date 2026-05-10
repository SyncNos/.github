# Plan P1 - webclipper-fetch-articles-enhancement

**Goal:** 把“抓取不完整（尤其 linux.do onebox 截断）”先用可回归的测试锁住，再用最小改动修复当前转换器问题，并落地 Turndown 风格转换器作为重构基础。

**Non-goals:** 本 phase 不引入新的 UI 设置；不做全站 extractor 大重写。

**Approach:** 先把现有“站点适配/特化逻辑”收敛到单一目录（便于后续完全重构与回归矩阵维护），再补回归用例并修 root 误选 bug；随后引入 Turndown 风格转换器（最小规则集），并把 Discourse cooked/fallback 切换到新转换器但保持兼容。

**Note (plan drift):** P1 里曾保留旧 `htmlToMarkdown()` 作为短期兜底；在后续 P4 会将其彻底移除并改为仅回退 `textContent`（避免“新旧两套 HTML→Markdown”长期共存）。

**Clean-as-you-go rule:**
- 本 feature 不把“清理旧代码”拖到最后统一做；每个迁移 task 结束时应尽量删除对应的旧实现/旁路 API。
- 允许极短期 shim/兼容层（用于平滑迁移 import 或兜底），但必须在同一 phase 内被回收，并用 `rg`/`tsc`/tests/build 验证无残留引用。

**Acceptance:**
- `https://linux.do/t/topic/1635410` 抓取结果不再仅 onebox，首贴正文可见且 Markdown 结构正常。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。
- 现有站点特化不回归：`article-extract-wechat-gallery`、`article-extract-bilibili-opus`、`article-extract-xiaohongshu`（以及本 phase 新增的 xhs engine smoke）均通过。

---

<a id="p1-t0"></a>
## P1-T0 整理站点适配代码目录（为完全重构做结构铺垫）

**Goal:**
- 把“站点特化/适配”的逻辑集中到一个明确的目录，减少 `engine.ts` 的站点 if/else，降低后续重构的耦合面。
- 本 task **不改变行为**（只移动与收敛组织结构）。

**Target layout (content-side):**
- Add: `webclipper/src/collectors/web/article-extract/sites/`
  - Add: `webclipper/src/collectors/web/article-extract/sites/discourse.ts`
  - Add: `webclipper/src/collectors/web/article-extract/sites/wechat.ts`
  - Add: `webclipper/src/collectors/web/article-extract/sites/xiaohongshu.ts`
  - Add: `webclipper/src/collectors/web/article-extract/sites/specs.ts`
  - Add: `webclipper/src/collectors/web/article-extract/sites/index.ts`

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/src/collectors/web/article-extract/discourse-op.ts`（降级为 re-export shim，后续 P1-T6 清理）
- Modify: `webclipper/src/collectors/web/article-extract/wechat-share-media.ts`（降级为 re-export shim，后续 P1-T6 清理）
- Add: `webclipper/src/collectors/web/article-extract/sites/*`

**Step 1: 实现功能**
- 把下列逻辑迁移到 `article-extract/sites/`，并在 `engine.ts` 中改为从该目录引入：
  - Discourse：`parseDiscourseTopicPathOnPage()` / `findDiscourseOpNode()` / `extractDiscourseOpOnly()`
  - WeChat：`isWechatShareMediaPage()` / `extractWechatShareMediaImageUrls()` / `buildWechatShareMediaGalleryHtml()`（以及后续可能要删除的 `buildWechatShareMediaGalleryMarkdown()` 先留在 shim 层）
  - 小红书：把 `engine.ts` 内的 `isXiaohongshuNotePage()` / `waitForXiaohongshuNoteHydrated()` 迁移到 `sites/xiaohongshu.ts`
- `discourse-op.ts` / `wechat-share-media.ts` 保留为 re-export shim，避免一次性改动过多 import；后续在 P1-T6 再移除这些 shim。
- 把现有 site spec 的入口也收敛到 `sites/specs.ts`：
  - `sites/specs.ts` 负责从 `webclipper/src/collectors/web/article-fetch-sites/` re-export `ARTICLE_FETCH_SITE_SPECS`
  - `engine.ts` 不再直接 import `@collectors/web/article-fetch-sites`，统一从 `@collectors/web/article-extract/sites` 引入（实现“适配入口单一”）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test`

Expected: 编译与测试通过，且行为无变化（纯结构重排）。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/engine.ts webclipper/src/collectors/web/article-extract/discourse-op.ts webclipper/src/collectors/web/article-extract/wechat-share-media.ts webclipper/src/collectors/web/article-extract/sites`

Run: `git commit -m "refactor: P1-T0 - 收敛文章抓取站点适配目录"`

---

<a id="p1-t1"></a>
## P1-T1 补齐fetch articles重构前的回归用例（linux.do onebox 截断）

**Files:**
- Add: `webclipper/tests/smoke/article-extract-discourse-op-onebox-regression.test.ts`

**Step 1: 实现功能**
- 新增 smoke test：构造一段模拟 Discourse OP `.cooked` HTML（包含 onebox 区块 + 后续正文列表/段落），通过 `extractWebArticleFromCurrentPage()` 断言 Markdown **包含 onebox 后的正文关键字**（避免只输出 onebox）。
- 断言策略使用 `toContain()`，避免对格式细节过敏。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-discourse-op-onebox-regression`

Expected:
- 在当前实现下应能稳定复现“只输出 onebox”的问题（红灯）。
- 如果意外变绿：说明用例的 HTML 夹具未覆盖到真实的“onebox 内嵌 `<article>` 被误选 root”形态，需要补齐夹具结构，直到能稳定复现再继续后续修复。

**Step 3: 原子提交**
Run: `git add webclipper/tests/smoke/article-extract-discourse-op-onebox-regression.test.ts`

Run: `git commit -m "test: P1-T1 - 增加linux.do onebox 截断回归用例"`

---

<a id="p1-t2"></a>
## P1-T2 修复现有htmlToMarkdown root误选导致只输出onebox的问题

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/markdown.ts`

**Step 1: 实现功能**
- 修复 `htmlToMarkdown()` 的 root 选择策略：
  - 仅当 wrapper 的 **直接子节点** 是单个 `ARTICLE` 时才使用该 `ARTICLE` 作为 root；
  - 否则使用 wrapper 自身作为 root（避免 onebox 内嵌 `<article>` 导致误选）。
- 确保 `details` 归一化仍可工作（不要因为 root 变化导致 details 丢失）。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-discourse-op-onebox-regression`

Expected: P1-T1 回归用例由红转绿，并且不会引入其它 Markdown 用例回归。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/markdown.ts`

Run: `git commit -m "fix: P1-T2 - 修复htmlToMarkdown误选root导致onebox截断"`

---

<a id="p1-t3"></a>
## P1-T3 引入Turndown风格HTML→Markdown转换器（最小规则集）

**Files:**
- Modify: `webclipper/package.json`
- Add: `webclipper/src/collectors/web/article-extract/markdown-turndown.ts`

**Step 1: 实现功能**
- 引入 `turndown`（以及按需评估 `turndown-plugin-gfm`），并封装 `htmlToMarkdownTurndown(html, baseHref)`：
  - 最小规则集覆盖：`a/img/pre>code/blockquote/ul/ol/li/details/summary`；
  - URL 处理：保证相对 `href/src` 能正确基于 `baseHref` 转绝对（实现可在 P2 做增强，但 P1 需要最小可用）。
- 对齐 obsidian-clipper 的方向：可扩展规则集，但 P1 先保证“能用 + 不截断”。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过，且新增模块可被测试引用（不破坏现有构建）。

**Step 3: 原子提交**
Run: `git add webclipper/package.json webclipper/src/collectors/web/article-extract/markdown-turndown.ts`

Run: `git commit -m "feat: P1-T3 - 引入Turndown风格HTML转Markdown转换器"`

---

<a id="p1-t4"></a>
## P1-T4 切换Discourse cooked 与 fallback 到新转换器并保持兼容

**Files:**
- Modify: `webclipper/src/collectors/web/article-extract/sites/discourse.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- (Optional) Add: `webclipper/tests/smoke/article-extract-discourse-op-markdown.test.ts`

**Step 1: 实现功能**
- Discourse OP（`.cooked`）转换优先走 `htmlToMarkdownTurndown()`；P1 阶段短期保留旧 `htmlToMarkdown()` 作为兜底（避免新规则不完备导致空内容），并在后续 P4 移除旧转换器（兜底改为仅回退 `textContent`）。
- 对 Readability/fallback 产物：同样优先走新转换器，必要时按场景开关（例如仅对 fragment 走新转换器）。
- 继续保留并验证 hydration 等待（如果当前仍有该问题），但不把它做成 Discourse 专用大适配。

**Step 2: 验证**
Run: `npm --prefix webclipper run test`

Expected: 全量测试通过；P1-T1 onebox 回归用例通过，且 Discourse OP `.cooked` Markdown 结构不再被 onebox 截断。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/sites/discourse.ts webclipper/src/collectors/web/article-extract/engine.ts webclipper/tests/smoke/article-extract-discourse-op-markdown.test.ts`

Run: `git commit -m "refactor: P1-T4 - 切换Discourse与fallback到新Markdown转换器"`

---

<a id="p1-t5"></a>
## P1-T5 增加小红书note的engine级回归用例（确保站点适配不被通用重构吞掉）

**Files:**
- Add: `webclipper/tests/smoke/article-extract-xiaohongshu-note.test.ts`

**Step 1: 实现功能**
- 新增 smoke test：用 JSDOM 构造一个最小的 `#noteContainer` hydrated DOM（文本 + 图片），再调用 `extractWebArticleFromCurrentPage()`：
  - 断言 `contentMarkdown` 包含正文关键字
  - 断言包含至少 1 条 xhs 图片 URL（并且已被 `sanitizeSiteImageUrl()` 正常绝对化）
- 目标：锁住“site spec 优先级 + hydration wait + 转换器升级”的组合不会导致 xhs 回退到 Readability/fallback。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- article-extract-xiaohongshu-note`

Expected: 测试稳定通过。

**Step 3: 原子提交**
Run: `git add webclipper/tests/smoke/article-extract-xiaohongshu-note.test.ts`

Run: `git commit -m "test: P1-T5 - 增加小红书note的engine级回归用例"`

---

<a id="p1-t6"></a>
## P1-T6 回收并删除临时shim（避免旧文件长期共存）

**Why:**
- P1-T0 允许 `discourse-op.ts` / `wechat-share-media.ts` 作为短期 shim 以减少迁移爆炸面。
- 但为了满足“及时清理旧代码”，我们在 P1 内就回收这些 shim，避免拖到 P3。

**Files:**
- Delete: `webclipper/src/collectors/web/article-extract/discourse-op.ts`
- Delete: `webclipper/src/collectors/web/article-extract/wechat-share-media.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- Modify: `webclipper/src/collectors/web/article-extract/sites/index.ts`

**Step 1: 实现功能**
- 用 `rg` 找出全仓库对两个 shim 的引用并改为从 `article-extract/sites/*` 引入（或从 `sites/index.ts` 的统一出口引入）。
- 删除 shim 文件本体，确保迁移后的入口唯一。

**Step 2: 验证**
Run: `rg -n "article-extract/(discourse-op|wechat-share-media)" webclipper/src -S`

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: `rg` 无命中；编译/测试/构建通过。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-extract/engine.ts webclipper/src/collectors/web/article-extract/sites/index.ts`

Run: `git rm webclipper/src/collectors/web/article-extract/discourse-op.ts webclipper/src/collectors/web/article-extract/wechat-share-media.ts`

Run: `git commit -m "refactor: P1-T6 - 删除fetch articles站点适配shim文件"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（至少 `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`）
