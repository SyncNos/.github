# Audit P1 - webclipper-comments-auto-attach-selection

## Scope

- P1 合入后的 inpage 根输入框自动附加选区链路
- `quoteText + locator` 在保存根评论前后的状态一致性

## 重点风险

- 根输入框交互事件时序导致选区读取为空（focus 抢占问题）
- 回复输入框被误接入自动附加
- session/controller handler 包装导致保存后引用清空回归

## 审计记录（执行期填写）

- Finding A1 (High) — `onComposerSelectionRequest` 并发竞态风险
	- Path: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
	- Risk: 连续触发（尤其未来 App/P2 引入异步 resolver 时）可能出现旧请求覆盖新请求，导致 `quoteText` 与 `pending locator` 不一致。
	- Status: Resolved
	- Expected fix: 引入 request id（latest-wins）门闩；过期请求返回结果应丢弃。

- Finding A2 (Medium) — 缺少“文本有效但 locator 解析失败”的行为测试
	- Path: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`
	- Risk: 规范要求“保留 quoteText + locator=null”缺少自动化守卫，后续易回归。
	- Status: Resolved
	- Expected fix: 新增测试覆盖 `selectionText != '' && locator == null` 的保存路径断言。

- Finding A3 (Medium) — 缺少 context 切换后 pending locator 清理测试
	- Path: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`
	- Risk: context 切换链路依赖 `pendingRootLocator = null`，未测试时容易被后续重构破坏。
	- Status: Resolved
	- Expected fix: 新增测试覆盖 context 切换后保存不带旧 locator。

## 修复记录（执行期填写）

- Fix A1:
	- Change: 在 `article-comments-sidebar-controller.ts` 为 `onComposerSelectionRequest` 增加 request sequence（latest-wins），丢弃过期异步响应。
	- Evidence: `tests/unit/article-comments-sidebar-controller.test.ts` 新增并通过 `ignores stale composer selection responses and keeps latest result`。
- Fix A2:
	- Change: 新增“locator 缺失时保留 quoteText 且以 locator=null 保存”测试。
	- Evidence: `tests/unit/article-comments-sidebar-controller.test.ts` 新增并通过 `preserves quote text when locator is missing and saves with null locator`。
- Fix A3:
	- Change: 新增 context 切换后清空 pending locator 的回归测试。
	- Evidence: `tests/unit/article-comments-sidebar-controller.test.ts` 新增并通过 `clears pending locator when context switches before save`。

## 回归验证

- [ ] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [ ] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/article-comments-sidebar-controller.test.ts`

已执行：

- [x] `npm --prefix webclipper run compile`
- [x] `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/article-comments-sidebar-controller.test.ts tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`
