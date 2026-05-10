# 评论级 Chat with AI：从“记录”到“讨论入口”

## 背景 / 触发

- SyncNos 当前的 `Chat with AI` 以**文章级**为主：在 article detail 里把整篇文章/对话渲染成 prompt，复制到剪贴板并跳转到用户配置的 AI 平台继续聊。
- 但用户真正想讨论的往往是“文章里的某一段 + 自己对这一段的想法”。这正好和评论侧边栏里每条 **root comment** 的语义一致：引用原文片段（quoteText）+ 写下自己的想法（commentText）。

## 核心需求（原始需求精炼）

1. 在 **inpage 评论侧边栏** 的每条 **root comment** 上新增 `Chat with AI` 按钮（reply 不展示）。
2. 在 **app/popup 的 article detail（ArticleCommentsSection）** 中，同样对 root comment 提供相同按钮（共享渲染器，一处改动两处生效）。
3. 点击后拼装 payload（精准上下文）并遵循 “复制 + 跳转” 的产品约束：
   - 先把 payload 写入剪贴板
   - 再按 Settings 中启用的平台打开/聚焦到对应 AI 平台
4. Payload 组成（无 quoteText 时降级）：
   - `quoteText`（可空）
   - `commentText`
   - `articleTitle`
   - `canonicalUrl`
   - 字符截断遵循现有 Chat with AI 的 `maxChars` 限制
5. Phase 4：同一篇文章多条 root comment 的 `Chat with AI` 点击，尽量复用同一个 AI tab，并把 **文章 tab + AI tab** 组织到同一个 tab group。

## 默认值与兼容策略

- 不新增 Setting（暂不提供开关），复用现有 Chat with AI 的平台启用配置与 `maxChars` 限制。
- 多平台启用时，以菜单形式呈现；只启用一个平台时，允许显示单平台 label（例如 `Chat with ChatGPT`）。
- Phase 1–2 的 payload 允许是 “v1 最小闭环”（例如先不含 quoteText，或 title 获取不到时降级为 url/占位）；Phase 3 再把 payload 升级为精准上下文（quote/comment/title/url）。
- 浏览器兼容性：
  - Phase 1–3：Chrome/Firefox 一致（复制与新开 tab）。
  - Phase 4：优先用 tab groups 能力；若运行时缺少相关 API，则降级为“复用/聚焦已有平台 tab（如果可定位），否则新开”。

## 非目标（明确不做什么）

- 不做同屏嵌入 AI（iframe/webview）。
- 不做自研聊天 UI 或直接调用模型 API。
- 不改变现有文章级 Chat with AI 的行为与配置语义。
- 不在 Phase 1–4 实现“自动填入 AI 输入框并发送”（后续单独评估）。
- 不给 reply（子回复）加按钮（只做 root comment）。

## 验收标准（可检查）

- inpage 评论侧边栏：每条 root comment 显示 `Chat with AI` 按钮；reply 不显示。
- app/popup 的 article detail：root comment 同样显示该按钮。
- 点击后：
  - payload 包含 `quoteText + commentText + articleTitle + canonicalUrl`（quote 可空但字段语义保持）
  - payload 超过 `maxChars` 时正确截断
  - 复制成功有提示；复制失败有明确失败提示
  - Phase 2 起：按用户启用的平台打开/聚焦目标 AI 平台
  - Phase 4 起：同一文章多次点击，尽量复用同一个 AI tab，并把文章 tab + AI tab 放入同一 tab group（可降级）
