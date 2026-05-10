# Audit P1 - webclipper-discourse-topic-op-dom-capture

## Findings

- 来源：`./.audit/subagent-audit-p1.md`（只读审计初稿）
- 结论：P1 三个 task 的主要目标均已落地，未发现阻塞 P2 的高风险缺陷。
- 低风险观察：
	1. comments background 在 `canonicalUrl` 不可用时回落 `conversationId` 查询，行为正确但缺少注释说明。
	2. sidebar 惰性 canonical 迁移失败时静默吞错，不利于后续定位数据收敛问题。

## Fixes

- 已修复（commit: `81e5d8a0`）
	1. `webclipper/src/services/comments/background/handlers.ts`
		 - 为 `LIST_ARTICLE_COMMENTS` 增加注释，明确“canonicalUrl 优先，缺失时回落 conversationId”的兼容语义。
	2. `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
		 - 将 canonical 迁移失败从静默吞错改为 `console.warn` 非阻塞告警，补充关键信息（from/to/conversationId/error）。

- 未执行项（说明）
	- 子代理建议增加“非 Discourse 但路径形似 `/t/slug/id`”边界测试；当前 canonical 规则按需求采用路径启发式以覆盖通用 Discourse，暂不收紧为域名白名单。

## Verification

结果：全部通过（app-shell smoke 仍有既有 React `act(...)` 警告，但不影响断言通过）

Run:
- `npm --prefix webclipper run test -- tests/unit/http-url.test.ts`
- `npm --prefix webclipper run test -- tests/storage/article-comments-idb.test.ts`
- `npm --prefix webclipper run test -- tests/unit/article-comments-sidebar-controller.test.ts tests/smoke/app-shell-comments-sidebar.test.ts`
- `npm --prefix webclipper run compile`
