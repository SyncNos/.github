# WebClipper Per-Tab AI Web Companion

## 背景 / 触发

我们希望实现：**每个内容 tab 对应一个 AI 网页/对话（per-tab AI web companion）**。

原生 Side Panel 更偏“窗口级”侧栏（跨 tab 持续存在），不适合承载“每个 tab 一个独立 AI 网页”的目标；因此本 feature 不再以 Side Panel 为主线。

## 核心需求（原始需求精炼）

1. `Chat with AI` 按钮能打开 AI 网页（优先在右侧侧边栏内 iframe）。
2. AI 网页与 comments 共享同一个右侧侧边栏容器（不新增第二条侧边栏）。
3. 侧边栏顶部 header 从固定 `Comments` 改为下拉菜单：`Comments` / `AI chats`（仅切换 view，不引入 toggle-close）。
4. 打开语义保持一致：open-only（只负责打开/聚焦，不做 toggle-close）。
5. 不新增设置项（先把体验跑通），不新增/修改 i18n 字段。

## 默认值与兼容策略

默认策略：先做“能跑通”的最小实现，并明确失败时的降级路径。

- 方案 1（优先试）：**共享侧边栏（Comments + AI chats） + iframe** 加载 AI 网页（同屏）。
- 方案 2（备用，但先归档）：**AI tab（不是新 window）**（见 `archive-p2.md`）。

## 非目标（明确不做什么）

- 不做原生 Side Panel。
- 不新增设置项、不新增/修改 i18n。

## 方案草图（按优先级）

### 方案 1：共享侧边栏 + iframe（同屏）

目标：在现有 comments 右侧侧边栏里新增一个 `AI chats` view，把 AI 网页尝试作为 iframe 加载（同屏体验最好）。

主要风险（已知高概率触发）：

- 大多数 AI 站点会通过 `frame-ancestors` / `X-Frame-Options` 禁止被嵌入，iframe 会直接被浏览器拦截。
- 登录态/第三方 Cookie/挑战页可能导致 iframe 内无法稳定工作。

策略（必须有）：

- 检测 iframe 加载失败/被拦截后，明确提示并给出下一步操作（当前先做到 `Open in new tab`；P2 是否恢复另行决策）。

## 核心障碍（iframe）

1. **X-Frame-Options / CSP frame-ancestors**：ChatGPT 设置了 `SAMEORIGIN`，会直接拦截 iframe 加载；Notion AI 同理。
2. **第三方 Cookie**：iframe 内容属于跨域，cookie 被视为第三方。`SameSite=Lax`（默认值）会阻止发送。
3. **Safari / Firefox / Brave 默认屏蔽第三方 cookie**，即使 Chrome 目前没强制禁用。
4. **Chrome 扩展上下文豁免不适用**：只有 `chrome-extension://` 顶层页面才享受第一方待遇，DOM 注入的 iframe 不算。

### Workaround（高风险）：`declarativeNetRequest` 剥 header

用 MV3 的 `declarativeNetRequest` 规则剥掉目标站点的 `X-Frame-Options` 和 `Content-Security-Policy` response header，强行让 iframe 可加载。

- 安全风险：剥掉安全 header 可能导致 Chrome Web Store 审核被拒。
- cookie 问题仍然存在，需要配合 `Storage Access API`（但需要 iframe 内部代码配合，我们控制不了）。
- W3C WebExtensions issue #483 正在讨论允许扩展绕过 CSP 加载 iframe 的提案，但还没落地。

**可行性判断**：高风险，需要实测验证。先试，踩到坑再说。

## 验收标准（可检查）

- 方案 1 可用时：`Chat with AI` 打开侧边栏并切到 `AI chats`，AI 网页在侧边栏中可见（同屏）。
- 方案 1 不可用时：能给出明确失败提示，并提供明确的下一步（当前先做到“Open in new tab”；是否恢复 P2 另行决策）。
- 验证链（最小）：`npm --prefix webclipper run compile`。
