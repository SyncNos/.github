# Audit P2 - webclipper-comments-auto-attach-selection

## Scope

- App/Popup 右侧评论侧栏根输入框自动附加选区
- narrow list/detail/comments 路由交互与评论侧栏联动

## 重点风险

- App/Popup 选区边界漂移（消息区外选区被误采纳）
- 侧栏关闭/重开后引用状态不同步
- runtime hooks 注入造成重挂载或性能抖动

## 审计记录（执行期填写）

- Finding 1（high）: 根输入框触发自动附加时，controller 端对 resolver 使用 `await`，在 `pointerdown -> focus` 时序下存在选区被焦点切换清空的风险。
	- 证据: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts` 的 `onComposerSelectionRequest` 早期实现始终异步续执行。
	- 影响: 可能出现“点击根输入框后未附加选中文本/引用为空”。

- Finding 2（medium）: App/Popup smoke 在 JSDOM 环境缺失 `globalThis.getSelection`，导致运行时 resolver 总是读到空选区，测试对真实语义覆盖不足。
	- 证据: `useArticleCommentsSidebarRuntime` 使用 `globalThis.getSelection?.()`；测试初始环境仅有 `window.getSelection`。
	- 影响: 测试可能出现假失败（或误判功能异常）。

## 修复记录（执行期填写）

- Fix 1: 在 controller 中改为“同步优先 + 异步兜底”应用选区载荷：
	- resolver 返回同步值时立刻应用；
	- resolver 返回 Promise 时保留 latest-wins 序号门闩；
	- 异常统一回退为空选区。

- Fix 2: 在 App/Popup smoke 用例中补齐 `globalThis.getSelection` 环境桥接，并使用可控 selection mock 覆盖 root composer 自动附加、回复输入不触发、无 locator root 清空三类行为。

- Fix 3: narrow flow smoke 增加 `sidebarOpen` 入参断言，并修复 pending-open mock 类型签名，消除测试文件类型诊断噪声。

## 回归验证

- [x] `npm --prefix webclipper run compile`
- [x] `npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-controller.test.ts tests/unit/comment-sidebar-session.test.ts`
- [x] `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/conversations-scene-narrow-comments-flow.test.ts tests/smoke/app-detail-header-actions.test.ts`
