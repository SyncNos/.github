# Plan P2 - webclipper-native-sidepanel-comments (Per-tab AI Companion Tab)

## Goal

放弃 iframe embed 原生 AI 网页后（见 `探索报告.md`），实现一个**稳定可落地**的降级体验：

- 对于每个“内容 tab”（用户正在浏览的网页 tab），创建/复用一个 **AI companion tab**（顶层页面，不是 iframe）。
- 尽量把 AI tab 放在内容 tab 旁边，并通过 **tab group** 绑定（best-effort）。
- 触发入口不再是 `Chat with AI` 按钮：统一为 **右侧 Comments 侧边栏 header 下拉菜单**中的一项（`Chat with AI` / `AI chats`）。

## Non-goals

- 不创建新 window（只用 tab）。
- 不做真同屏（浏览器原生不支持真正 split view）。
- 不做 DNR 剥安全响应头等“hacker 绕过”（审核/安全风险过高）。
- 不新增设置项，不改 i18n。

## UX / Entry

### 入口样式

在右侧 Comments 侧边栏 header 标题处使用一个下拉按钮：

- `Comments`（当前视图，disabled）
- `Chat with AI`（点击后：打开/聚焦该 tab 对应的 AI companion tab）

### 行为

- Open-only：反复触发只会**复用/聚焦** AI tab，不负责关闭侧边栏。
- 若用户未配置任何 Chat-with 平台：侧边栏内提示错误（best-effort）。

## Architecture

### 映射与存储

- 维护 `sourceTabId -> aiTabId` 映射（优先 `chrome.storage.session`，避免持久化到 local 导致 stale tabId）。
- 通过 `tabs.onRemoved` 清理映射（source tab 或 ai tab 任一被关闭都要清理）。

### Tab 操作（best-effort）

- 创建/复用 AI tab
- `tabs.move`：把 AI tab 移动到 source tab 的右侧
- `tabs.group`：如果 source tab 已在 group 中，则把 AI tab 加入；否则将二者 group 到同一组

## Acceptance

- 在任意网页中打开 inpage Comments 侧边栏，header 下拉点击 `Chat with AI`：
  - 会打开/聚焦一个 AI tab（URL 为 Settings 里第一个 enabled 平台）
  - AI tab 会尽量紧挨当前网页 tab
  - 支持 tab group 时，会把两个 tab 尽量绑定到同一 group
- 再次点击 `Chat with AI`：复用并聚焦同一个 AI tab（不会创建多个）
- 关闭 source tab 或 AI tab：映射会清理（下次触发会重新创建）
- 验证链：`npm --prefix webclipper run compile`

## Manual smoke checklist

1. 打开任意网页（如 `https://example.com`），双击选中文本触发 Comments 侧边栏（或其它入口打开）。
2. 点击 header 下拉 `Chat with AI`：
   - 生成/聚焦 AI tab（默认第一个平台 URL）
   - AI tab 被移动到网页 tab 右侧
   - 如果 Chrome 支持 tab group：两个 tab 被放入同一 group
3. 回到网页 tab 再点一次：应聚焦同一个 AI tab（不新增）。
4. 关闭 AI tab，再点一次：应重新创建 AI tab。

