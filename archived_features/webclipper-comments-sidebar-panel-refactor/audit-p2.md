# Audit P2 - webclipper-comments-sidebar-panel-refactor

> 记录 P2 的审计发现与修复闭环（由 executing-plans 在 P2 完成后自动进入）。

## Scope

- Phase: `P2`（`P2-T1` ~ `P2-T6`）
- Baseline commits:
  - `90ab7f4f` (`P2-T1`)
  - `ed74b3c4` (`P2-T2`)
  - `4572bd4a` (`P2-T3`)
  - `ee9a3fe3` (`P2-T4`)
  - `a62f57dd` (`P2-T5`)
  - `ee7b9f4e` (`P2-T6`)

## Read-only checks

1. 模块依赖方向（`panel.ts` 组装，其它模块单向被引用）：
   - `rg -n "from './(panel|shadow-styles|dock|resize|chatwith|locate|render|types)'" webclipper/src/ui/comments/threaded-comments-panel/*.ts`
   - 结果：仅 `panel.ts` 依赖子模块，`index.ts` 仅导出，无反向引用。
2. UI 误引 `platform/**`：
   - `rg -n "@platform/|src/platform|/platform/" webclipper/src/ui/comments/threaded-comments-panel/*.ts`
   - 结果：无命中。
3. 阶段验证：
   - `npm --prefix webclipper run lint`
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test`
   - 结果：通过（`95 passed`）。

## Findings

- `F-1`（非阻塞）：`panel.ts` 仍为 `608` 行，虽已由“单体实现”收敛为“组装 + API 外壳”，但仍高于 phase 建议值（<300）。
  - 处理：本次保持行为稳定优先，不在 P2 审计中继续切分；后续可在独立重构任务继续拆分 `panel.ts` 的 DOM scaffold / API wiring。
- `F-2`（已修复）：抽离 `locate.ts` 后高亮移除时长从 `1400ms` 漂移到 `1800ms`，导致 `threaded-comments-panel-locate` 失败。
  - 修复：恢复为 `1400ms`，并移除 `panel.ts` 遗留的未使用类型导入。
  - 修复提交：`a32d98b4`
  - 验证：`tests/unit/threaded-comments-panel-locate.test.ts` 通过，随后全量 `npm --prefix webclipper run test` 通过。

## Decision

- `P2 audit: PASS with note`
- 允许进入 `P3`。
