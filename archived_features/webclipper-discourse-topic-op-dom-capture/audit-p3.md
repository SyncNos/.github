# Audit P3 - webclipper-discourse-topic-op-dom-capture

## Findings

- No blocking issues in P3 task scope (canonical/unit/smoke coverage完整，核心行为可回归验证)。
- Low: `resolveOrCaptureActiveTabArticle` 仅覆盖了 `isNew=true` 广播路径，缺少“existing conversation 不广播”的保护断言。
- Cross-suite issue discovered during full `npm test`:
	- `tests/storage/conversations-idb.test.ts` 中 `merge conversations by ids` 用例仍使用旧的固定 key（`keep/remove`），与当前 article canonical key 规则（`article:<canonicalUrl>`）不一致，导致全量测试失败。
- Post-completion deep review (extra):
	- Medium: `waitForTabUrl` 之前仅做 URL 字符串全等比较，若 Discourse 导航后地址附带 query/hash（如 `/1?u=...#...`）会出现“已到目标楼层但误判超时”的边界风险。
	- Low: `article-fetch.ts` 注入脚本内存在 Discourse path regex 与 warning flag 字面量重复定义，可进一步去重。
- Follow-up bug from real usage:
	- High: 当 topic URL 为 `/t/.../<non-numeric>`（例如 `/t/topic/1870532/xxx`）时，旧解析仅接受数字楼层，导致 Discourse 识别失败，不会触发 `/1` fallback，进而可能抓取空正文。
	- Clarification: Discourse 在已访问 `/1` 后，后续浏览高楼层（如 `/820`）时 DOM 可能仍可直接命中 OP；此时应直接使用 OP 抓取结果，不应再触发 `/1` 导航。

## Fixes

- P3 任务内测试补齐：
	- `tests/unit/http-url.test.ts`：补充 Discourse canonical 边界（大小写路径、non-topic Discourse path）。
	- `tests/unit/article-comments-sidebar-controller.test.ts`：新增“同 topic 不同楼层 URL 不重置上下文/草稿”断言。
	- `tests/unit/discourse-canonical-url.test.ts`：新增专题 canonical 行为矩阵测试。
	- `tests/smoke/article-fetch-service.test.ts`：强化 `/20?query#hash` fallback 行为与 `/1` 严格失败不导航断言。
	- `tests/smoke/article-fetch-discourse-op.test.ts`：新增 Discourse OP fallback + canonical 保存与 resolve 复用 canonical key 场景。
	- `tests/smoke/background-router-article-fetch.test.ts`：新增 resolveOrCapture 严格错误归一化与广播条件测试（含 `isNew=false` 不广播）。
- Full-suite 修复：
	- `tests/storage/conversations-idb.test.ts`
		- merge 用例改为使用 `upsertConversation` 返回的真实 canonical key；
		- sync mapping 测试数据与断言同步到 canonical key，消除旧 key 假设。
- Extra deep-review fix（本次收尾新增）：
	- `webclipper/src/collectors/web/article-fetch.ts`
		- 新增 `isSameDiscourseTopicFloorUrl`，使 `/1` 导航等待支持 topic floor 语义比较（忽略 query/hash 变体）；
		- 将 fallback 导航等待超时提取为常量 `DISCOURSE_NAVIGATION_WAIT_TIMEOUT_MS`；
		- 将提取稳定化参数提取为常量（减少 magic numbers）；
		- 把注入脚本中的 Discourse path 正则与 OP-missing warning flag 改为通过 args 注入，删除重复字面量定义。
	- `webclipper/tests/smoke/article-fetch-service.test.ts`
		- fallback 测试改为模拟 `/1?u=abc#reply-1`，验证 URL 到位判定不会误超时。
- Follow-up bug fix（`/xxx` 不回跳 `/1`）：
	- `webclipper/src/collectors/web/article-fetch.ts`
		- Discourse path regex 扩展为允许非数字第三段（`/t/<slug>/<id>/<segment>`）；
		- `parseDiscourseTopicUrl` 新增 `postSegment`，在 segment 非数字时仍识别为 topic；
		- fallback 条件调整为：OP 缺失且当前并非明确 `/1` 时，包含非数字 segment 也会跳转 `/1`。
	- `webclipper/src/services/url-cleaning/http-url.ts`
		- canonical 规则同步支持非数字第三段，`/t/<slug>/<id>/xxx` 仍收敛到 topic-level URL。
	- `webclipper/tests/unit/http-url.test.ts`
	- `webclipper/tests/unit/discourse-canonical-url.test.ts`
	- `webclipper/tests/smoke/article-fetch-service.test.ts`
		- 新增/更新 `/xxx` 回归覆盖，锁定本次线上问题。
		- 新增“高楼层已可提取 OP 时不跳 `/1`”回归用例，锁定 DOM 命中 OP 的直取行为。

## Verification

Passed:

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/unit/http-url.test.ts tests/unit/article-comments-sidebar-controller.test.ts tests/unit/discourse-canonical-url.test.ts`
- `npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-service.test.ts tests/smoke/article-fetch-discourse-op.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`
- `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`
- `npm --prefix webclipper run test`
- `npm --prefix webclipper run test -- tests/smoke/article-fetch-service.test.ts tests/smoke/article-fetch-discourse-op.test.ts tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`
- `npm --prefix webclipper run test -- tests/unit/http-url.test.ts tests/unit/discourse-canonical-url.test.ts tests/smoke/article-fetch-service.test.ts tests/smoke/article-fetch-discourse-op.test.ts tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`

Result:

- Full suite: `126 passed`, `513 passed`, `0 failed`

Run:
- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/unit/http-url.test.ts tests/unit/article-comments-sidebar-controller.test.ts tests/unit/discourse-canonical-url.test.ts`
- `npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-service.test.ts tests/smoke/article-fetch-discourse-op.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`
- `npm --prefix webclipper run test`
