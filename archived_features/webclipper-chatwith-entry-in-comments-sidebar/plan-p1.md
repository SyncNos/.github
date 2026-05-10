# Plan P1 - webclipper-chatwith-entry-in-comments-sidebar

## Goal

把 `Chat with...` 的主入口从 `app/popup` 详情页 header actions 迁移到 **右侧评论侧边栏（comments sidebar）的 header** 中，并同时支持：

- `app`：扩展页面右侧 comments sidebar
- `inpage`：网页注入的右侧 comments sidebar

同时 **全局移除**原先 `app/popup` 详情页 header actions 里的 `Chat with...` 入口（popup 不再提供入口可接受）。

## Non-goals

- 不做 iframe embed（已证伪且高风险）。
- 不做 per-tab AI companion tab。
- 不新增设置项（复用 `chat_with` settings）。

## Approach

1. 以 `mountThreadedCommentsPanel(...)` 为 UI 真源：在其 header 右侧新增 `Chat with...` 下拉按钮（列出 enabled 平台）。
2. 下拉点击后执行现有语义（保持一致）：
   - 生成 payload（复用 `buildChatWithPayload` / settings）
   - 写入剪贴板
   - 打开外链新 tab
3. `app` / `inpage` 通过注入不同的 “resolve actions” 回调来提供上下文：
   - app：用 `selectedConversation.id -> getConversationDetail` 获取 messages
   - inpage：用 `ARTICLE_MESSAGE_TYPES.RESOLVE_OR_CAPTURE_ACTIVE_TAB -> CORE_MESSAGE_TYPES.GET_CONVERSATION_DETAIL`
4. 移除旧入口：不再在详情页 header actions 中渲染/生成 chat-with slot。

## Acceptance

- app/inpage 的右侧 comments sidebar header 都出现 `Chat with...` 下拉，并能对 enabled 平台执行“复制 + 打开新 tab”。
- app/popup 详情页 header actions 不再出现 `Chat with...`。
- `npm --prefix webclipper run compile` 通过。

---

<a id="p1-t1"></a>
## P1-T1 Panel：为 comments sidebar header 增加 Chat with 下拉入口（UI + plumbing）

**Files**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`
- Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`

**Steps**
1. 给 `mountThreadedCommentsPanel(..., options)` 增加可选参数（建议：`chatWith?: { resolveActions: () => Promise<{ id; label; onTrigger; disabled? }[]> }`）。
2. 在 header 中渲染一个 `Chat with...` 按钮（右侧，与 collapse 按钮同一行），点击后打开下拉菜单。
3. 下拉菜单内容：
   - 打开时异步调用 `resolveActions`
   - 显示 loading / empty / error 的最小态
   - 点击 menu item 时：关闭菜单并执行 `onTrigger()`
4. 保证不会影响 comments sidebar 的 open/close 语义（不触发 close）。

**Verify**
- `npm --prefix webclipper run compile`

---

<a id="p1-t2"></a>
## P1-T2 App：将 Chat with 下拉接入右侧 comments sidebar（复用现有 payload）

**Files**
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- (Optional) Modify: `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`（复用 `resolveChatWithDetailHeaderActions`）

**Steps**
1. 在 `AppShell.tsx` 里基于 `selectedConversation` 提供一个 `resolveChatWithActions` 回调：
   - 读取 ChatWith Settings（platforms + template + max chars）
   - `getConversationDetail(selectedConversation.id)` 拉取 detail.messages
   - 生成 actions（优先复用 `resolveChatWithDetailHeaderActions`）
2. 将该回调透传到 `ArticleCommentsSection.tsx -> mountThreadedCommentsPanel` 的 `chatWith.resolveActions`。
3. 仅在 `article` 且右侧 comments sidebar 可见时启用入口；否则隐藏或 disabled。

**Verify**
- `npm --prefix webclipper run compile`

---

<a id="p1-t3"></a>
## P1-T3 Inpage：将 Chat with 下拉接入注入 comments sidebar（resolve/capture + detail）

**Files**
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- (Optional) Modify: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`（若需要共享 helper）

**Steps**
1. 在 inpage panel 创建时，为 `mountThreadedCommentsPanel` 注入 `chatWith.resolveActions`：
   - `runtime.send(ARTICLE_MESSAGE_TYPES.RESOLVE_OR_CAPTURE_ACTIVE_TAB, { tabId? })`
   - `runtime.send(CORE_MESSAGE_TYPES.GET_CONVERSATION_DETAIL, { conversationId })`
2. 将 resolve 返回的 `title/url/author/publishedAt` 组装为最小 `Conversation`（`sourceType: 'article'`）以复用既有 payload 生成逻辑。
3. 同 P1-T1：菜单点击执行 `onTrigger()`（复制 + 打开新 tab）。

**Verify**
- `npm --prefix webclipper run compile`

---

<a id="p1-t4"></a>
## P1-T4 移除：全局删除 app/popup 详情页 header 的 Chat with 入口

**Files**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
- Modify: `webclipper/src/services/integrations/detail-header-actions.ts`

**Steps**
1. `detail-header-actions.ts` 不再 push chat-with actions（移除 `resolveChatWithDetailHeaderActions` 相关调用）。
2. 从 `ConversationDetailPane.tsx` / `DetailNavigationHeader.tsx` 中移除对应的 `DetailHeaderActionBar actions={chatWithActions}` 渲染与计算逻辑，确保 UI 彻底消失。

**Verify**
- `npm --prefix webclipper run compile`

---

<a id="p1-t5"></a>
## P1-T5 验证：compile + 人工冒烟（app/inpage）

**Commands**
- `npm --prefix webclipper run compile`

**Manual checklist**
1. app：进入一篇 article 会话，打开右侧 comments sidebar；header 可见 `Chat with...` 下拉；点任一 enabled 平台后：剪贴板有内容 + 新 tab 打开平台。
2. inpage：在任意网页打开 comments sidebar；header 可见 `Chat with...` 下拉；点 enabled 平台后：同样复制 + 打开新 tab。
3. popup/app 详情页 header：不再出现 `Chat with...`。

---

## Audit (P1)

完成 P1 后写入 `audit-p1.md`：
- 入口是否只存在于 comments sidebar header（没有遗漏旧入口）
- 下拉菜单在 Shadow DOM 内是否被 host CSS 干扰
- 错误/空态是否可理解（尤其是无 enabled 平台、无 conversation detail）

