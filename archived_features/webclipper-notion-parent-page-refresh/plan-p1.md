# Plan P1 - webclipper-notion-parent-page-refresh

**Goal:** 把 Notion parent pages discovery 收敛为 services/background 单一真源，并让 Settings UI 具备可见 loading + 跨分页找到可用 pages 的能力。

**Non-goals:**
- 不新增“手动输入 page id”设置项
- 不改 Notion 同步编排器与 DB 创建逻辑
- 不改 i18n 翻译表（仅移除引用）

**Approach:** 先从根因修复 Notion connect/disconnect 状态不同步（避免 `runTask` 队列自我等待，并让 token/pending/error 的变化能驱动 Settings 自动 refresh），确保用户点击后无需刷新插件页面。随后在 `webclipper/src/services/sync/notion/` 引入独立的 parent pages discovery 单一真源模块（分页/过滤/resolve saved page），通过 `registerNotionSettingsHandlers` 新增 message 暴露给 UI。最后迁移 `useSettingsSceneController` 使用该 message 并在同提交删除旧直连实现，同时重构 `NotionOAuthSection` 的 loading/empty 表达以移除误导占位。

**Acceptance:**
- connect/disconnect 点击后无需刷新插件页面，状态能立即更新
- 点击刷新/自动加载时可见 loading（例如 refresh 按钮 spinner），不再“看起来没反应”
- 能跨分页找到可用 pages（避免第一页过滤后为空导致长期空列表）
- Settings 不再直连 `api.notion.com`；旧直连实现被及时删除

---

<a id="p1-t1"></a>
## P1-T1 在 services 层新增 Notion parent pages 单一真源模块

**Files:**
- Add: `webclipper/src/services/sync/notion/notion-parent-pages.ts`
- (Optional) Modify: `webclipper/src/services/sync/notion/notion-api.ts`（仅在需要复用/抽取通用逻辑时）

**Step 1: 实现功能**

- 定义服务层 API（建议）：`listNotionParentPages({ accessToken, savedPageId?, pageSize?, maxPages? })`
- 行为要求：
  - `POST /v1/search` 支持 `start_cursor` 分页
  - 过滤规则与现有保持一致：`object=page`、非 `archived/in_trash`、排除 `parent.database_id` 的页
  - 结果去重（按 id）
  - 若 `savedPageId` 不在搜索结果中，提供 resolve 能力（`GET /v1/pages/:id`）并在可用时注入到列表头部
  - 尽量复用现有 `notionFetch` / `getPageTitle`（避免重复实现与口径漂移）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

Manual:
- 打开扩展 `Settings -> Notion`，点击 `Disconnect` 后无需刷新页面即可立刻变为 `Connect` 状态。
- 点击 `Connect` 发起 OAuth 后，完成授权回到 Settings 页面时，连接状态能自动更新（不要求手动刷新插件页面）。

**Step 3: 原子提交**

- `refactor: task1 - 收敛 Notion parent pages discovery 到 services 单一真源`

---

<a id="p1-t2"></a>
## P1-T2 新增 background handler + message type：列出/刷新 Notion parent pages

**Files:**
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/services/sync/notion/settings-background-handlers.ts`
- Modify: `webclipper/src/entrypoints/background.ts`（如果 handler 注册依赖需要调整）

**Step 1: 实现功能**

- 新增 message type（例如）：`NOTION_MESSAGE_TYPES.LIST_PARENT_PAGES`
- background handler 语义：
  - 从 token store 获取 access token（UI 不传 token）
  - 调用 `notion-parent-pages.ts` 获取 `pages + resolvedSavedPage`
  - 返回 payload 给 UI：`{ pages: {id,title}[], resolvedSaved?: {id,title} | null }`
  - 失败时返回结构化 error（遵循现有 router `ok/err` 形态）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

- `refactor: task2 - 通过 background message 暴露 Notion parent pages 列表`

---

<a id="p1-t3"></a>
## P1-T3 迁移 Settings ViewModel 到新 message，并在同提交删除旧直连实现

**Files:**
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- Modify: `webclipper/src/viewmodels/settings/utils.ts`（删除 Notion 直连 fetch 相关函数）

**Step 1: 实现功能**

- `onLoadNotionPages()` 改为调用 `send(NOTION_MESSAGE_TYPES.LIST_PARENT_PAGES, ...)` 获取 options（不再直接 fetch Notion API）
- 同一提交内删除旧代码：
  - `searchNotionParentPages()`
  - `retrieveNotionParentPage()`
  - 以及所有引用与 import
- 增加残留扫描（作为本 task 的自检步骤，确保“及时删老代码”）
  - Run: `rg -n "searchNotionParentPages|retrieveNotionParentPage|https://api\\.notion\\.com/v1/(search|pages)" webclipper/src/viewmodels`

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

- `refactor: task3 - Settings 改走 background 列表并删除 Notion 直连实现`

---

<a id="p1-t4"></a>
## P1-T4 重构 NotionOAuthSection 的 loading/empty 表达，移除 clickRefresh 占位

**Files:**
- Modify: `webclipper/src/ui/settings/sections/NotionOAuthSection.tsx`

**Step 1: 实现功能**

- 使用 `loadingNotionPages` 做可见 loading：
  - refresh 按钮在 loading 时显示 spinner（或替换图标）
  - Select 在 loading 时禁用（避免用户误判）
- 移除对 `t('clickRefresh')` 的引用：
  - 空 options 时 placeholder 使用更中性的 label（例如 `t('parentPage')`），不要提示“点击刷新”
  - 不改 i18n 翻译表

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

- `refactor: task4 - Notion Parent Page 刷新增加可见 loading 并移除 clickRefresh 占位`

---

<a id="p1-t5"></a>
## P1-T5 从根本修复 Notion connect/disconnect 状态不同步（无需刷新插件页面）

**Files:**
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- Modify: `webclipper/src/services/sync/notion/settings-background-handlers.ts`

**Step 1: 实现功能**

- 修复“断开后状态不更新/需要刷新插件页面”的根因：
  - 禁止在 `runTask()` 的 task 内 `await refresh()`（避免任务队列自我等待导致 UI 状态卡死）
  - disconnect 后应在当前页面立即更新 `notionConnected/pollingNotion/notionPages/notionParentPage*` 等状态（并在后台清理存储后由 refresh 二次校准）
- 让“连接完成/断开完成”的状态变化能动态推送到 Settings：
  - 扩展 `storageOnChanged` 监听：当 `notion_oauth_token_v1` / `notion_oauth_pending_state` / `notion_oauth_last_error` 等键变更时，触发一次 `void refresh()`（不 await）
- 统一 `GET_AUTH_STATUS` 的 shape：
  - background 返回 `workspaceName` 时，UI 不需要依赖刷新页面才能看到一致的状态文本（若现有 UI 展示该字段）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

- `fix: task5 - 修复 Notion connect/disconnect 状态无需刷新即可更新`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
  4. 自检分层边界：`rg -n \"@platform/\" webclipper/src/ui webclipper/src/viewmodels`
