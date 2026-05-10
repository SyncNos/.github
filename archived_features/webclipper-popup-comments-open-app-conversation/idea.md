## 背景 / 触发

- 用户反馈：popup 面板里“评论”按钮的行为与预期不一致；并且感觉 popup 与 app.html 的右侧评论侧边栏存在功能缺失。
- 进一步澄清后决定：
  - **会话详情页 header tools 不应该放 Chat with 功能**，需要彻底清理掉这条入口的可能性（不只是 UI 不显示）。
  - Chat with 只属于 **右侧评论侧边栏（Article comments sidebar）** 的能力（含 sidebar header + comment-level）。
  - popup 的评论按钮不再打开 inpage comments panel，而是跳转到 `app.html` 并定位到对应 article conversation；同时 popup 立即关闭。

## 核心需求

### Goal 1（清理）

- 会话详情页（popup/app 的 conversation detail header）`tools` 槽位中 **永远不出现** `Chat with ...`。
- 代码与文档层面都要收敛：不允许后续“顺手又接回去”。

### Goal 2A（先做：低风险）

- popup 顶部“评论”按钮改为：
  1. 在 background 侧 **resolve 或 capture** 当前 active tab 的 article conversation（拿到 `conversationId` 与 canonical url）。
  2. 打开/聚焦 `app.html`，并定位到对应 conversation（`/?loc=...`）。
  3. 立刻关闭 popup window。
- 本阶段 **不** 自动展开 comments sidebar（避免引入“loc 就绪时序闸门”）。

### Goal 2B（取消）

- 取消：不再实现 app 侧 “loc 就绪后再 open comments” 的一次性时序闸门。

## 非目标（明确不做）

- 不把 Chat with 重新放回会话详情页 header tools。
- Goal 2A 不做“自动展开 comments sidebar”。
- 不改 i18n 文案（除非编译/测试必须）。

## 验收标准（可检查）

- 在 popup 点击评论按钮：
  - 会打开/聚焦 `app.html` 并定位到当前页对应的 article conversation。
  - popup 自身会关闭。
  - 不会打开 inpage comments panel。
- 在会话详情页 header tools：
  - 不出现 `Chat with ...`。
  - 相关 resolver / 文档契约不再暗示“detail tools 包含 chatwith”。
- 右侧评论侧边栏（app/inpage）仍可正常使用 `Chat with...`（不回归）。
