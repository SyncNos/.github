# Audit P3 - webclipper-markdown-reading-profiles

> 记录 P3 的审计发现与修复闭环（由 executing-plans 在 P3 完成后自动进入）。

## Scope

- Phase: `P3`（`P3-T1` ~ `P3-T3`）
- Baseline commits:
  - `328cb4f4`（P3-T1）
  - `095249ab`（P3-T2）
  - `f253322b`（P3-T3）

## Read-only checks

1. 设置存储与回读检查：
   - `rg -n "markdown_reading_profile|ReadingProfile|normalize.*profile" webclipper/src/viewmodels/settings webclipper/src/services/protocols`
2. 设置 UI 与渲染链接入检查：
   - `rg -n "reading profile|markdown profile|ChatMessageBubble|InpageSection|SettingsScene|ConversationDetailPane" webclipper/src/ui webclipper/src/viewmodels/settings`
3. i18n 键完整性检查：
   - `rg -n "readingProfile|medium|notion|book" webclipper/src/ui/i18n/locales/en.ts webclipper/src/ui/i18n/locales/zh.ts`
4. UX 收口检查（人工 + 测试证据）：
   - `320px` 下正文不双向滚动（`pre/table` 例外）
   - 长链接断行不溢出
   - light/dark + user/assistant 基础可读性可接受
5. 最终验证链：
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test`
   - `npm --prefix webclipper run build`

## Findings

1. 已落地：
   - 存储协议接入 `markdown_reading_profile_v1`，脏值统一回退 `medium`。
   - 设置页可选择 `medium/notion/book`，并联动详情页 markdown 渲染。
   - 文档已补齐协议真源、默认值、回退和扩展顺序（`webclipper/AGENTS.md`、`webclipper/src/ui/AGENTS.md`）。
2. UX 抽查证据（测试侧）：
   - `tests/smoke/markdown-reading-profiles-smoke.test.ts`：`overflow-wrap`、`pre/table` 局部横滚、unknown 回退。
   - `tests/smoke/markdown-reading-profiles-matrix.test.ts`：`profile x bubbleRole` 与 light/dark token scaffold。
3. 最终验证链：
   - `npm --prefix webclipper run compile` 通过。
   - `npm --prefix webclipper run test` 失败（8 项）。
   - `npm --prefix webclipper run build` 通过。
4. 失败项分析（与本 feature 相关性）：
   - 失败集中在 `tests/smoke/popup-shell-header-actions.test.ts`、`tests/smoke/app-shell-comments-sidebar.test.ts`、`tests/smoke/app-shell-narrow-header-actions.test.ts`、`tests/smoke/app-shell-sidebar-collapse.test.ts`。
   - 共同错误：`react-tooltip` 在 jsdom 环境缺少 `MutationObserver`（并伴随 `dispatchEvent` 类型错误）。
   - 判定：属于测试环境基线问题，和本次 markdown 阅读风格改动无直接逻辑耦合。

## Decision

- Pass with non-blocking test-env baseline issue（可收口本 feature）。
