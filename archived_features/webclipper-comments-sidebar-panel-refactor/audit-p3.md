# Audit P3 - webclipper-comments-sidebar-panel-refactor

> 记录 P3 的审计发现与修复闭环（由 executing-plans 在 P3 完成后自动进入）。

## Scope

- Phase: `P3`（`P3-T1` ~ `P3-T4`）
- Baseline commits:
  - `6ef7df28` (`P3-T1`)
  - `fe0cb9d7` (`P3-T2`)
  - `f0dab8b7` (`P3-T3`)
  - `8821a68b` (`P3-T4`)

## Read-only checks

1. `threaded-comments-panel/*` 类型收敛检查（`any` 漏网）：
   - `rg -n "\\bas any\\b|:\\s*any\\b|<any>|\\bany\\[\\]" webclipper/src/ui/comments/threaded-comments-panel -g'*.ts'`
   - 结果：无命中。
2. P3-T1 目标路径 helper 收敛（调用 shared helper）：
   - `rg -n "normalizeHttpUrl|normalizePositiveInt|normalizeConversationId|safeString" webclipper/src/ui/app/AppShell.tsx webclipper/src/ui/conversations/ArticleCommentsSection.tsx webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts webclipper/src/ui/comments/threaded-comments-panel/*.ts`
   - 结果：目标路径已统一使用 `@services/url-cleaning/http-url` 与 `@services/shared/numbers`，未再出现 `normalizeConversationId`。
3. 模块边界检查（UI 不误引 `platform/**`）：
   - `rg -n "@platform/|src/platform|/platform/" webclipper/src/ui/comments/threaded-comments-panel/*.ts`
   - 结果：无命中。
4. 阶段验证链：
   - `npm --prefix webclipper run lint`
   - `npm --prefix webclipper run format:check`
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test`
   - `npm --prefix webclipper run build`
   - 结果：全部通过（`98 files / 396 tests` passed）。
5. 最小行为回归（本 phase 相关）：
   - `npm --prefix webclipper run test -- tests/unit/threaded-comments-panel-resize.test.ts tests/unit/threaded-comments-panel-locate.test.ts tests/unit/threaded-comments-panel-shortcuts.test.ts`
   - 结果：通过（`3 files / 6 tests`）。

## Findings

- `F-1`（非阻塞，沿用 P2 备注）：`panel.ts` 当前 `609` 行，已成为“组装壳层”但体量仍偏大。
  - 处理：本 feature 已满足“行为不变 + 类型与测试收敛 + 全链路验证通过”；继续拆分建议放入后续独立 refactor，避免在收尾阶段引入行为漂移。

## Decision

- `P3 audit: PASS with note`
- 当前 feature 执行闭环完成。
