## Coverage check
- **P1-T0**: **Pass** — 站点适配逻辑已收敛到 `webclipper/src/collectors/web/article-extract/sites/*`，`engine.ts` 改为统一从 `@collectors/web/article-extract/sites` 引入；原 `discourse-op.ts`/`wechat-share-media.ts` 在该提交中降级为 shim。
- **P1-T1**: **Pass** — 新增 `webclipper/tests/smoke/article-extract-discourse-op-onebox-regression.test.ts`，覆盖 onebox 后正文不应丢失。
- **P1-T2**: **Pass** — `markdown.ts` root 选择修复为“仅当 wrapper 直接子节点唯一且为 ARTICLE 才下钻”，避免误选 onebox 内嵌 `<article>`。
- **P1-T3**: **Pass** — 已引入 `turndown` 与 `markdown-turndown.ts`，并实现最小规则与 URL 绝对化处理。
- **P1-T4**: **Pass** — Discourse OP 与 Readability/fallback 路径均改为优先 `htmlToMarkdownTurndown()`，并保留 legacy `htmlToMarkdown()` 兜底。
- **P1-T5**: **Pass** — 新增 `webclipper/tests/smoke/article-extract-xiaohongshu-note.test.ts`，覆盖 xhs note hydrated 场景及图片 URL 绝对化断言。
- **P1-T6**: **Pass** — shim 文件已删除（`discourse-op.ts`、`wechat-share-media.ts`），仓库内无残留引用。

## Findings
No blocking findings.

## Suggested fixes
- 无（当前未发现需要修复的阻塞/高信号问题）。

## Verification notes
- 变更核对：`git --no-pager show --name-status` 检查了 P1-T0..P1-T6 对应提交（533f5708, 99204125, 53fffb57, 2923804f, 16822dba, e48c9bd1, f9e07e3c）。
- 清理验证：`rg -n "article-extract/(discourse-op|wechat-share-media)" webclipper/src -S` 与 tests 范围检查均无命中（exit 1，符合“无结果”）。
- Phase 验证命令：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 执行成功（exit code 0）。
- 目标回归相关测试命令：`npm --prefix webclipper run test -- article-extract-wechat-gallery article-extract-bilibili-opus article-extract-xiaohongshu article-extract-xiaohongshu-note article-extract-discourse-op-onebox-regression` 执行成功（exit code 0）。
