# Audit P3 - webclipper-comments-auto-attach-selection

## Scope

- 根输入框触发边界加固
- 既有入口兼容性（双击 inpage / popup / context menu）
- 文档事实与代码实现一致性

## 重点风险

- “仅根输入框触发”守卫遗漏导致回复输入框误触发
- inpage click combo / background relay 被新逻辑破坏
- deepwiki 与 AGENTS 描述滞后或冲突

## 审计记录（执行期填写）

- Finding 1（medium）: README 摘要最初仅描述“reply 不触发附加”，未显式写明“无选区清空”语义，和 deepwiki/AGENTS 的契约表述不完全一致。
	- 影响: 维护者可能误读 reply 与清空语义边界。

- Finding 2（low, 已判定非阻断）: P3-T2 兼容 smoke 主测入口链路（double-click relay/combo），未重复覆盖 reply 不触发语义。
	- 处置: 不新增冗余 smoke；该语义已由 `tests/unit/threaded-comments-panel-auto-attach-selection.test.ts` + `tests/smoke/app-shell-comments-sidebar.test.ts` 覆盖。

## 修复记录（执行期填写）

- Fix 1: 同步 `README.md` 与 `README.zh-CN.md`，明确“根输入框可在无选区时清空引用；reply 输入框不影响引用状态”。

- Fix 2: 维持现有测试分层策略（compat smoke 负责入口链路，root-only 语义由 unit + app-shell smoke 承担），未做重复断言扩张。

## 回归验证

- [x] `npm --prefix webclipper run compile`
- [x] `npm --prefix webclipper run test -- tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/unit/threaded-comments-panel-focus-regression.test.ts tests/unit/threaded-comments-panel-shortcuts.test.ts`
- [x] `npm --prefix webclipper run test -- tests/smoke/content-controller-inpage-combo.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/smoke/inpage-button-click-combo.test.ts`
