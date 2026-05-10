# Plan P1 - webclipper-discourse-topic-op-dom-capture

**Goal:** 建立 Discourse topic 级 canonical URL 规则，并打通文章会话与评论上下文的统一键，先消除“同帖多会话”根因。

**Non-goals:**
- 本 phase 不实现 `/1` 跳转抓取流程（P2 做）
- 本 phase 不改变非 Discourse 页面抓取行为

**Approach:**
- 在 URL 清洗层新增 article 场景 canonical 函数（保留现有 `normalizeHttpUrl` 向后兼容）。
- 让文章会话 key、评论 canonicalUrl、侧栏 context key（含 app 与 inpage）统一落到 topic 级 URL。
- 对历史 `/N` URL 数据采用惰性兼容：读写时统一 canonical，而不是一次性全量迁移。

**Acceptance:**
- `/t/slug/topic-id/20` 与 `/t/slug/topic-id/1` 会生成同一 canonical URL
- 同一 topic 不再创建重复 article conversation
- 评论侧栏在同一 topic 不同楼层间切换时保持同一会话上下文

---

## P1-T1 新增Discourse topic canonical规则与单测

**Files:**
- Modify: `webclipper/src/services/url-cleaning/http-url.ts`
- Modify: `webclipper/tests/unit/http-url.test.ts`

**Step 1: 实现功能**
- 在 `http-url.ts` 新增 article 场景 canonical 方法（建议命名：`canonicalizeArticleUrl`）：
  - 仅处理 `http/https`
  - 去 hash
  - 若命中 Discourse topic 路径 `^/t/<slug>/<topicId>(/<postNumber>)?/?$`，则移除楼层后缀 `/postNumber` 与尾部斜杠
  - 对 Discourse topic URL 忽略 query（避免 `?u=` 等导致同帖多 canonical）
- 保持 `normalizeHttpUrl` 的兼容语义不破坏现有调用方。
- 新增/更新单测覆盖：
  - `/t/x/123`、`/t/x/123/1`、`/t/x/123/20` 归一一致
  - 非 Discourse URL 不受影响

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- tests/unit/http-url.test.ts`

Expected: URL 归一化单测通过

**Step 3: 原子提交**
Run: `git add webclipper/src/services/url-cleaning/http-url.ts webclipper/tests/unit/http-url.test.ts`

Run: `git commit -m "feat: task1 - 新增Discourse主题URL canonical规则"`

---

## P1-T2 打通文章会话与评论canonical统一键

**Files:**
- Modify: `webclipper/src/collectors/web/article-fetch.ts`
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/services/comments/background/handlers.ts`
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现功能**
- 文章抓取链路统一使用 article canonical URL：
  - conversationKey 使用 canonical URL 生成
  - upsertConversation 的 `url` 使用 canonical URL
  - resolveOrCapture 返回的 `url` 使用 canonical URL
- 会话存储层 `normalizeArticleUrl` 对齐 canonical 规则，确保历史 `/N` 会话可命中同一 topic。
- 评论存储层与 background handler 的 canonicalUrl 归一化对齐 canonical 规则，避免 `/1` 与 `/20` 形成两套评论线程。
- 会话编辑/更新链路（`conversations-context.tsx`）在 article 场景改用同一 canonical 方法，避免手工更新 URL 时重新分叉会话键。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/storage/article-comments-idb.test.ts`

Expected: 编译通过，消息与存储调用链无类型回归，评论存储归一化测试通过

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-fetch.ts webclipper/src/services/conversations/data/storage-idb.ts webclipper/src/services/comments/data/storage-idb.ts webclipper/src/services/comments/background/handlers.ts webclipper/src/viewmodels/conversations/conversations-context.tsx`

Run: `git commit -m "refactor: task2 - 统一文章会话与评论canonical键"`

---

## P1-T3 稳定评论侧栏上下文并兼容历史URL

**Files:**
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现功能**
- 侧栏 `contextKey` 构造统一使用 article canonical URL，确保同 topic 楼层切换不触发会话重置。
- app 侧边栏适配器（非 inpage）也统一 canonical，避免旧会话 URL（含 `/N`）触发评论线程分叉。
- inpage adapter 的 fallback/ensureContext 返回 canonical URL。
- 在 inpage 打开评论面板时，`canonicalUrlFallback` 也走 canonical 规则。
- App Shell 与 ArticleCommentsSection 传入评论模块的 canonicalUrl 同步改用 article canonical，确保 popup 与 inpage 行为一致。
- 如检测到同会话下旧 canonicalUrl（带 `/N`）与新 canonicalUrl（topic 级）不一致，触发现有 comments canonical 迁移路径进行惰性收敛。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-controller.test.ts tests/smoke/app-shell-comments-sidebar.test.ts`

Expected: app/inpage 两侧评论上下文与切换逻辑测试通过

**Step 3: 原子提交**
Run: `git add webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/ui/conversations/ArticleCommentsSection.tsx webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "feat: task3 - 稳定Discourse同主题评论侧栏会话"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
