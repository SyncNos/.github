# Plan P1 - webclipper-native-sidepanel-comments (Shared sidebar: Comments + AI Web)

## Goal

把现有“文章评论右侧侧边栏”升级为**共享侧边栏**：

- 同一个右侧侧边栏支持两种内容：`Comments` / `AI chats`（iframe AI web）。
- 入口改为侧边栏 header 下拉：用户切到 `AI chats` 时，在侧边栏内尝试加载 AI 网页（iframe）。
- `AI chats` 初次进入时：自动加载 Settings 中配置的“第一个 AI”。
- 在 `app`（扩展页面）与 `inpage`（页面注入面板）两处行为保持一致（同样的 header 下拉切换）。
- iframe 不可用时：提供明确失败提示 + `Open in new tab`。

## Non-goals

- 不做原生 Side Panel。
- 不新增设置项，不新增/修改 i18n。
- P1 不承诺所有 AI 站点可 iframe；重点是把“尝试 + 失败检测 + CTA”跑通。
- 暂不实现 per-tab AI tab 映射（已归档见 `archive-p2.md`）。

## Approach

1. 复用现有右侧评论面板的容器：`mountThreadedCommentsPanel(...)`（app/inpage 都在用）。
2. 在面板内部新增 `AI chats` 视图：
   - body 内保留 comments UI，同时新增一个 iframe host 区域；
   - 通过 `mode` 切换显示 `comments` / `ai`。
3. header 升级为下拉菜单（替换原来的 `Comments` 文本）：
   - options：`Comments` / `AI chats`；
   - 切换时只切 view，不改变 open/close 语义。
4. 默认 AI 加载：
   - 用户切到 `AI chats` 时，如果尚未设置 URL，则读取 ChatWith Settings 的平台列表，取“第一个 AI”，并加载其 URL。
5. iframe 失败检测（best-effort）：
   - 加载超时或 load 事件异常后进入 error 态；
   - 提供 `Open in new tab` CTA。

## 核心障碍（必须预期会遇到）

1. **X-Frame-Options / CSP frame-ancestors**：ChatGPT/Notion AI 常见 `SAMEORIGIN`，会直接拦截 iframe。
2. **第三方 Cookie**：跨域 iframe cookie 视为第三方，`SameSite=Lax` 常阻止发送。
3. **Safari / Firefox / Brave 默认屏蔽第三方 cookie**（Chrome 也在逐步收紧）。
4. **扩展上下文豁免不适用**：只有 `chrome-extension://` 顶层页面才是第一方；DOM 注入的 iframe 不算。

## Acceptance（P1）

- 用户从侧边栏 header 下拉切到 `AI chats`：尝试加载 Settings 中第一个 AI 的 iframe。
- header 下拉可在 `Comments` / `AI chats` 间切换（app/inpage 一致）。
- AI 网页不可 iframe 时：能在 sidebar 内看到明确错误提示，并能一键 `Open in new tab`。
- 验证链：`npm --prefix webclipper run compile`。

---

<a id="p1-t1"></a>
## P1-T1 扩展 comments 面板为“共享侧边栏”（mode + ai iframe）

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`
- Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`

**Step 1: 定义 UI mode 与最小 API**

- 新增 panel 内部 `mode: 'comments' | 'ai'`，默认 `comments`。
- 新增最小 API（向后兼容）：
  - `setMode(mode)`
  - `setAiUrl(url)`

**Step 2: header 下拉切换**

- 用“下拉菜单样式”替换 header title：
  - `Comments`
  - `AI chats`

**Step 3: 失败检测 + CTA**

- iframe 加载超时/异常后进入 error 态，并提供 `Open in new tab`。

**Step 4: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

---

<a id="p1-t2"></a>
## P1-T2 接线：Chat with AI -> 共享侧边栏（app）

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`

**Step 1: 接入 sidebar**

- `Chat with AI` 点击后：
  - 保留“复制 payload”逻辑；
  - 打开右侧共享侧边栏；
  - 切到 `AI chats` 并设置 URL（`setAiUrl`）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

---

<a id="p1-t3"></a>
## P1-T3 inpage comments panel 同步支持 `AI chats`

**Files:**
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- (Optional) Modify: `webclipper/src/services/bootstrap/content-controller.ts`（若提供 inpage 触发入口）

**Step 1: 启用共享侧边栏能力**

- inpage 面板启用 mode dropdown + ai iframe。
- 保持 open-only 语义。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

---

<a id="p1-t4"></a>
## P1-T4 人工冒烟 checklist

1. 在 app 中点击 `Chat with AI`：侧边栏切到 `AI chats`，能看到 iframe loading/失败提示。
2. header 下拉切换：`Comments` / `AI chats` 可互切。
3. ChatGPT/Notion 等不可 iframe 的站点：侧边栏内会显示拒绝连接/空白等失败表现（无 “Open in new tab” 按钮）。

---

<a id="p1-t5"></a>
## P1-T5 移除 `Chat with AI` 按钮入口（仅保留 sidebar header 下拉）

**Files:**
- Modify: `webclipper/src/services/integrations/detail-header-actions.ts`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
- (Optional) Modify: `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`（如不再需要入口逻辑）

**Step:**
- 不再生成/渲染 `slot = 'chat-with'` 的 actions，使 `Chat with AI` 按钮从 UI 消失。

**Verify:**
- Run: `npm --prefix webclipper run compile`

---

<a id="p1-t6"></a>
## P1-T6 `AI chats` 选中时自动加载 Settings 里第一个 AI

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`

**Step:**
- 当面板切到 `AI chats` 且尚未设置 URL 时：从 ChatWith Settings 读取 platforms，选取“第一个 AI”，并 `setAiUrl(...)` + 触发加载。
- 无可用 AI 时：显示提示，引导用户去 Settings 配置。

**Verify:**
- Run: `npm --prefix webclipper run compile`

---

<a id="p1-t7"></a>
## P1-T7 移除 AI chats 的 `Open in new tab` CTA

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`
- (Optional) Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`

**Step:**
- `AI chats` 失败时不提供一键打开新 tab 的按钮（仅保留状态提示 + Retry）。

**Verify:**
- Run: `npm --prefix webclipper run compile`

---

<a id="p1-t8"></a>
## P1-T8 实验：DNR 剥 iframe 安全 header（XFO/CSP）

**⚠️ 风险声明（必须接受）**
- 这是“hacker / 实验性”方案：通过 `declarativeNetRequest` 移除目标站点 response headers（`X-Frame-Options` / `Content-Security-Policy` 等）来绕过 iframe 限制。
- 可能导致 Chrome Web Store 审核拒绝，且 cookie / 登录态问题仍可能存在。

**Files:**
- Modify: `webclipper/wxt.config.ts`
- Modify: `webclipper/src/entrypoints/background.ts`
- Add: `webclipper/src/services/integrations/iframe-bypass/iframe-bypass-dnr.ts`

**Step:**
- 增加 `declarativeNetRequest` 权限，并在 background 启动时按 ChatWith Settings 的 platforms 列表注册 dynamic rules：
  - 仅对 `resourceTypes: ['sub_frame']` 生效；
  - 移除 `X-Frame-Options`、`Content-Security-Policy` 等响应头。
- Settings 变更时自动重刷规则。

**Verify:**
- Run: `npm --prefix webclipper run compile`

---

<a id="p1-t9"></a>
## P1-T9 修复：DNR 规则匹配与安装稳定性

**背景**
- 即使实现了 DNR 剥 header，仍可能因为规则未命中/未及时安装而看起来“完全没效果”。

**Files:**
- Modify: `webclipper/src/services/integrations/iframe-bypass/iframe-bypass-dnr.ts`
- Modify: `webclipper/src/entrypoints/background.ts`

**Step:**
- `urlFilter` 改为官方推荐的 `||domain/`（避免 `||domain`/`^` 造成意外匹配或不匹配）。
- 同时写入 `session rules`（即时生效）+ `dynamic rules`（持久化），降低 service worker 初始化时序问题。
- 增加 `onStartup` 触发重刷，确保浏览器重启后规则仍然存在且正确。

**Verify:**
- Run: `npm --prefix webclipper run compile`
