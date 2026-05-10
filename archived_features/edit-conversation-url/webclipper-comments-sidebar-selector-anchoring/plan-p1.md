# Plan P1 - webclipper-comments-sidebar-selector-anchoring

**Goal:** 让右侧 comments sidebar 的“根评论”支持基于 selector anchoring 的点击定位（inpage + app），并具备失败轻提示。

**Non-goals:** 不做跨环境定位、不做 embedded/narrow inline 评论区定位、不做定位高亮。

**Approach:** 复用成熟实现：使用 `dom-anchor-text-quote` + `dom-anchor-text-position` 将“选区 Range”序列化为 locator（quote selector + position selector + env），写入 `article_comments` 的可选字段。渲染侧在根评论点击时还原 `Range`，用 `scrollIntoView({ behavior: 'smooth' })` 触发滚动；失败则展示轻提示。

**Acceptance:**
- inpage sidebar 与 app 右侧 sidebar：对带 locator 的根评论点击可 smooth 定位；reply 不触发定位。
- 历史评论（无 locator）点击不滚动，但有轻提示。
- 不改变 open-only / close-only-by-header 的既有语义。

---

## P1-T1 引入 selector anchoring 依赖与类型定义

**Files:**
- Modify: `webclipper/package.json`
- Add/Modify: `webclipper/src/services/comments/locator/*`（新建目录）
- Modify: `webclipper/src/services/comments/domain/models.ts`

**Step 1: 实现功能**

- 添加依赖：
  - `dom-anchor-text-quote`
  - `dom-anchor-text-position`
- 新建 locator 领域类型（不与 UI 强耦合）：
  - `ArticleCommentLocator`（包含 `env: 'inpage'|'app'`、`quote` selector、`position` selector、版本字段）
  - 仅用于“根评论 + quoteText 非空”的写入路径

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

Run: `git add webclipper/package.json webclipper/src/services/comments/locator webclipper/src/services/comments/domain/models.ts`

Run: `git commit -m "feat: task1 - 引入评论定位的selector-anchoring类型与依赖"`

---

## P1-T2 为 article_comments 增加可选 locator 存储字段（无破坏升级）

**Files:**
- Modify: `webclipper/src/services/comments/data/storage-idb.ts`
- Modify: `webclipper/src/services/comments/domain/models.ts`
- Modify: `webclipper/src/services/comments/client/repo.ts`

**Step 1: 实现功能**

- `ArticleComment` / `ArticleCommentDto` 增加可选字段（例如 `locator?: ArticleCommentLocator | null`）。
- `storage-idb.ts`：
  - `addArticleComment` 写入 `locator`（可选）
  - `toComment` 读出 `locator`（可选）
- 不新增 index，不修改 store schema（避免强制 DB version bump）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；旧数据读写不报错。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/data/storage-idb.ts webclipper/src/services/comments/domain/models.ts webclipper/src/services/comments/client/repo.ts`

Run: `git commit -m "feat: task2 - 为文章评论存储增加可选locator字段"`

---

## P1-T3 inpage：打开 sidebar 时捕获选区 selector，并在保存根评论时写入 locator

**Files:**
- Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`（若需扩展 payload）
- Modify: `webclipper/src/services/comments/background/handlers.ts`

**Step 1: 实现功能**

- 在 inpage 打开 sidebar 时（仍保留现有 “selectionText/quoteText” 逻辑）：
  - 捕获当前 selection `Range`（在 top frame）
  - 生成 locator：`TextQuoteSelector` + `TextPositionSelector`（root 选用页面内容 root）
  - 将 locator 暂存到 comments sidebar session（或 controller 内部）作为“下一次根评论保存时使用”
- 保存根评论（onSave）时：
  - 仅当 `quoteText` 非空且 locator 存在时，随 addRoot 请求一起发送到 background 存储
  - reply 不携带 locator

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；inpage 打开与保存评论不报错。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/platform/messaging/message-contracts.ts webclipper/src/services/comments/background/handlers.ts`

Run: `git commit -m "feat: task3 - inpage保存根评论时写入selector locator"`

---

## P1-T4 app：打开 comments sidebar 时捕获选区 selector，并在保存根评论时写入 locator

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`（若需要在 app 侧 session/controller 之间传递 locator）
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`

**Step 1: 实现功能**

- 在 app 详情页点击“评论按钮”打开右侧 sidebar 前：
  - 从 `messagesRootRef` 内的 selection 读取 `Range`
  - 生成 locator 并注入到 comments sidebar session/controller 的“下一次根评论保存”上下文
- 保存根评论（onSave）时：
  - 仅当 `quoteText` 非空且 locator 存在时写入
- 保持既有语义：点击评论按钮不负责关闭 sidebar。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；app 打开 sidebar + 保存评论不报错。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx webclipper/src/ui/app/AppShell.tsx webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`

Run: `git commit -m "feat: task4 - app保存根评论时写入selector locator"`

---

## P1-T5 sidebar：根评论点击触发 selector 定位 + smooth scroll + 失败轻提示

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`（为 app sidebar 提供 anchor root/scroll root）

**Step 1: 实现功能**

- `threaded-comments-panel.ts`：
  - 对 root comment 的可点击区域绑定 click（避免影响 delete/reply/composer）
  - 当 item 含 locator 且为 root 时：还原 Range 并 `scrollIntoView({ behavior: 'smooth', block: 'center' })`
  - 失败时展示轻提示（panel 内部的轻量提示，不用 alert）
- 为不同环境提供 anchor root：
  - inpage：页面内容 root（避免 shadow DOM panel 自己成为搜索范围）
  - app：详情内容区域 root（避免在 sidebar 内误匹配）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；点击 root comment 会尝试定位；失败会提示。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/threaded-comments-panel.ts webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/src/ui/conversations/ArticleCommentsSection.tsx webclipper/src/ui/app/AppShell.tsx`

Run: `git commit -m "feat: task5 - 评论侧边栏根评论点击定位（selector anchoring）"`

---

## P1-T6 单测：locator 序列化/还原与点击定位的最小覆盖

**Files:**
- Add/Modify: `webclipper/src/services/comments/locator/*.test.ts`
- Add/Modify: `webclipper/src/services/comments/threaded-comments-panel.test.ts`（若已有类似测试则就近添加）

**Step 1: 实现功能**

- 覆盖最小行为：
  - 给定一个简单 DOM + Range，生成 locator 并还原为 Range（exact match）
  - `threaded-comments-panel`：当 root comment 含 locator 时，点击会调用 `scrollIntoView`（用 stub/spy）
  - reply item 不触发定位

**Step 2: 验证**

Run: `npm --prefix webclipper run test`

Expected: 新增测试通过；若存在已知阻塞测试需在 note 中标注与本任务无关。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/locator webclipper/src/services/comments/threaded-comments-panel.test.ts`

Run: `git commit -m "test: task6 - 为评论selector定位补最小单测覆盖"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令

