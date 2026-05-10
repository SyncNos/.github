# WebClipper：将「Chat with AI」入口迁移到右侧评论侧边栏 Header

## 背景 / 触发

目前 `Chat with...`（复制 payload + 跳转外部 AI 平台）入口主要出现在扩展 `app` / `popup` 的详情页 header actions 中。

实际使用中该入口不符合主要场景：**用户在网页上阅读/选中内容/写评论时（inpage）顺手问 AI**。同时在窄宽度（popup）下按钮文案会被隐藏，导致“存在但不显眼/几乎不用”。

此外我们已探索过 iframe embed / 原生 sidepanel 等方案：对 Notion/ChatGPT 等站点会遇到 `X-Frame-Options` / `CSP frame-ancestors`、第三方 Cookie、WebAuthn(passkey) 等问题，整体不可控且风险高。因此本 feature 明确不再做 embed。

## 核心需求

1. **把 `Chat with...` 入口完整迁移**到“右侧评论侧边栏”的 header 中：
   - `app`（扩展页面的右侧 comments sidebar）
   - `inpage`（注入到网页的右侧 comments sidebar）
2. **全局移除**原先在 `app/popup` 详情页 header actions 中的 `Chat with...` 按钮/菜单入口（popup 不再提供入口也可以接受）。
3. 发送给 AI 的内容策略保持不变：仍复用现有 ChatWith Settings（prompt template / platforms / max chars），以及现有 payload 生成逻辑。
4. 侧边栏 open/close 语义保持一致：入口只负责执行动作，不引入 toggle-close。

## 默认值与兼容策略

- 仅对“文章评论右侧侧边栏”提供入口（因为该侧边栏当前只在 article 场景出现）。
- `Chat with...` 下拉菜单列出 Settings 中 `enabled=true` 的平台；若 0 个启用平台则隐藏入口或禁用并提示去 Settings 配置。
- 动作语义保持与原入口一致：**点击平台 -> 复制 payload 到剪贴板 -> 打开平台 URL（新 tab）**。

## 非目标（明确不做什么）

- 不做 iframe embed AI 网页（已证伪）。
- 不做 per-tab AI companion tab / tab group 之类的“伪同屏”方案。
- 不新增设置项（复用现有 chat_with settings）。

## 验收标准（可检查）

- `inpage`：打开评论侧边栏后，header 中能看到 `Chat with...` 下拉；选择任一已启用平台后会复制 payload 并打开对应 AI 平台新 tab；侧边栏保持打开。
- `app`：文章会话右侧 comments sidebar header 同样具备 `Chat with...` 下拉并可用。
- `app/popup`：详情页 header actions 不再出现 `Chat with...` 入口。
- 最小验证链：`npm --prefix webclipper run compile`。

