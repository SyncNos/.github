# Audit P2 - webclipper-discourse-topic-op-dom-capture

## Findings

- High: `tests/smoke/article-fetch-service.test.ts` 缺失 “Discourse 位于 `/1` 且仍找不到 OP 时必须严格失败” 的覆盖，P2-T3 行为虽已实现但测试未锁定。
- Medium: `Discourse OP not found` 错误文案在 `article-fetch.ts`、`article-fetch-background-handlers.ts`、`current-page-capture.ts` 存在重复 magic string，长期存在漂移风险。

## Fixes

- 新增共享错误常量与判定助手：
	- `webclipper/src/collectors/web/article-fetch-errors.ts`
	- 导出 `DISCOURSE_OP_MISSING_WARNING_FLAG`、`DISCOURSE_OP_NOT_FOUND_ERROR`、`isDiscourseOpNotFoundErrorMessage`
- `webclipper/src/collectors/web/article-fetch.ts`
	- 改用共享常量 `DISCOURSE_OP_MISSING_WARNING_FLAG`
	- 严格失败由字面量替换为 `DISCOURSE_OP_NOT_FOUND_ERROR`
- `webclipper/src/collectors/web/article-fetch-background-handlers.ts`
	- 错误归一化逻辑改为 `isDiscourseOpNotFoundErrorMessage` + `DISCOURSE_OP_NOT_FOUND_ERROR`
- `webclipper/src/services/bootstrap/current-page-capture.ts`
	- 当前页抓取错误归一化改为复用共享 helper/常量
- `webclipper/tests/smoke/article-fetch-service.test.ts`
	- 新增 smoke：`fails strictly when discourse OP is still missing on /1`
	- 断言 `fetchActiveTabArticle()` 抛 `Discourse OP not found`
	- 断言失败时不会写入 conversation/messages

## Verification

Passed:

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/smoke/article-fetch-service.test.ts tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`

Result:

- Test Files: 3 passed
- Tests: 11 passed

Run:
- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/smoke/article-fetch-service.test.ts`
- `npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts`
- `npm --prefix webclipper run test -- tests/smoke/article-fetch-wechat-gallery.test.ts`
