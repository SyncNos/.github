# Plan P2 - webclipper-discourse-topic-op-dom-capture

**Goal:** 实现 Discourse 的 OP-only DOM 抓取：优先当前页定位 OP，失败后跳 `/1`，并在超时后严格失败。

**Non-goals:**
- 本 phase 不做“跳到 `/1` 后自动跳回原楼层”
- 本 phase 不引入 Discourse API/JSON 拉取

**Approach:**
- 在 `article-fetch.ts` 中引入 Discourse 专用提取分支。
- 仅当当前页无法定位 OP 且 URL 非 `/1` 时才导航到 `/1`。
- 导航后通过“URL 到位 + DOM 稳定”双条件等待再提取；失败则抛出明确错误，禁止降级抓回复。

**Acceptance:**
- 从 `/t/slug/topic-id/20` 触发抓取时，最终正文来自 OP
- 抓取结束后停留在 `/1`
- `/1` 超时未定位 OP 时抓取失败且错误可见

---

## P2-T1 实现Discourse OP优先DOM提取器

**Files:**
- Modify: `webclipper/src/collectors/web/article-fetch.ts`

**Step 1: 实现功能**
- 在注入脚本中增加 Discourse topic 识别与 OP 节点定位（示例特征：`article[data-post-number='1']` 及等价语义节点）。
- 对 Discourse 页面优先执行 OP-only 提取：
  - title/author/publishedAt 沿用现有元信息提取
  - `contentHTML/contentMarkdown/textContent` 仅来自 OP 节点
- 非 Discourse 页面继续走现有 Readability/兜底流程。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/article-fetch-wechat-gallery.test.ts`

Expected: 编译通过，且非 Discourse 抓取路径无回归

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-fetch.ts`

Run: `git commit -m "feat: task4 - 新增Discourse OP优先DOM提取分支"`

---

## P2-T2 OP缺失时跳转到1楼并停留

**Files:**
- Modify: `webclipper/src/collectors/web/article-fetch.ts`
- Modify: `webclipper/tests/smoke/article-fetch-service.test.ts`

**Step 1: 实现功能**
- 在 Discourse topic 抓取流程中加入“当前页找不到 OP 的 fallback”逻辑：
  - 若当前路径为 `/t/slug/topic-id/N` 且 `N != 1`，导航到 `/t/slug/topic-id/1`
  - 若当前已在 `/1`，禁止重复导航（防循环）
  - 导航后等待页面路由与 DOM 稳定（带上限超时，建议显式封装 wait helper，至少校验 tab URL 与文档可读状态）
  - 重试 OP 提取
- 成功后不做“返回原楼层”动作，保持停留在 `/1`。
- 更新 `article-fetch-service` smoke 测试桩，覆盖导航 fallback 分支（包括停留 `/1` 预期）。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/article-fetch-service.test.ts`

Expected: 编译通过，导航 fallback 行为可复现且无死循环

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-fetch.ts webclipper/tests/smoke/article-fetch-service.test.ts`

Run: `git commit -m "feat: task5 - OP缺失时跳转1楼抓取并停留"`

---

## P2-T3 超时找不到OP时严格失败并提示

**Files:**
- Modify: `webclipper/src/collectors/web/article-fetch.ts`
- Modify: `webclipper/src/collectors/web/article-fetch-background-handlers.ts`
- Modify: `webclipper/src/services/bootstrap/current-page-capture.ts`

**Step 1: 实现功能**
- 当 `/1` 超时仍无法定位 OP 时，抛出明确错误（例如 `Discourse OP not found`），并中止抓取。
- 禁止回落到“抓当前可见楼层正文”路径，确保“只抓 OP”原则不被破坏。
- 错误通过 background 与 UI 现有提示链路透出，用户可理解并可重试。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts`

Expected: 编译通过，错误可经 router/UI 链路透出且类型一致

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-fetch.ts webclipper/src/collectors/web/article-fetch-background-handlers.ts webclipper/src/services/bootstrap/current-page-capture.ts`

Run: `git commit -m "feat: task6 - Discourse OP缺失时严格失败并回传错误"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
