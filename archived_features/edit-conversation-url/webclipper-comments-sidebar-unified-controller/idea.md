# WebClipper Comments Sidebar Unified Controller

## 背景 / 触发

我们现在有两套“文章评论侧边栏”的入口：

- **inpage 侧边栏**：content script 打开 shadow panel，走 runtime message 让 background 读写评论数据，并包含 `resolveOrCaptureArticle + attach orphan comments` 等逻辑。
- **app 右侧边栏**：在 `app.html` 的三栏布局里打开右侧栏，直接读写本地 repo（IndexedDB client），并由 AppShell 管理开关状态。

两者已经复用同一个 UI 面板实现（`mountThreadedCommentsPanel`）与同一个 session 模型（`CommentSidebarSession`），但**业务流程仍然分叉**：保存/刷新/忙碌态/引用（quote）生命周期在两个入口分别实现，导致行为漂移（例如“发送评论后引用不清空”这类问题会反复出现）。

本 feature 的目标是把评论侧边栏的“业务流程”收敛为一份共享 controller：入口不同，但打开/保存/刷新/清引用等行为一致。

## 核心需求（原始需求精炼）

1. inpage 侧边栏与 app 右侧边栏共享同一份“评论侧边栏 controller”业务流程代码。
2. 两者仅入口不同：
   - inpage：通过 runtime message 与 background 交互（必须保留）。
   - app：保留直连本地 repo（IndexedDB client）以减少消息绕行（方案2）。
3. 行为完全一致（不因入口不同而漂移）：
   - 选中文本 → 点击评论按钮 → quote 注入侧边栏，聚焦输入框。
   - **发送 root comment 成功后，quote 必须清空**（两端一致）。
   - reply 不携带 quote（两端一致）。
   - 保存/删除成功后评论列表刷新（两端一致）。
   - busy 状态与 open 请求消费语义一致，不出现反复 mount/unmount 或交互冻结。

## 默认值与兼容策略

- 不引入新设置项，不改变既有开关/布局。
- 兼容现有调用方：
  - inpage 仍由 content handlers 注册消息并创建 controller。
  - app 仍由 AppShell 管理右侧栏显示/关闭；controller 只接管 comments 流程与 handlers。
- 数据层不做 schema 迁移；仅重构调用结构与统一流程。

## 非目标（明确不做什么）

- 不把 app 改成走 messaging（不做方案1）。
- 不重写 threaded-comments-panel 的 UI / DOM 结构（仅在必要时补接口以支持 controller 接入）。
- 不调整评论数据模型与存储 schema（`article_comments` 等）。
- 不新增/修改 i18n 字段。

## 验收标准（可检查）

- inpage 与 app 两端：
  - 打开侧边栏后，quote 注入正确。
  - root comment 保存成功后，quote 立刻清空（UI 上不再显示引用）。
  - reply 保存不携带 quote；保存后 quote 状态保持“已清空”。
  - 删除 comment 后列表刷新。
- controller 作为单一真源：inpage 与 app 的“保存/刷新/清 quote”流程代码不再重复两份。
- 通过验证链：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/unit/comment-sidebar-session.test.ts tests/unit/article-comments-sidebar-chrome.test.ts`

