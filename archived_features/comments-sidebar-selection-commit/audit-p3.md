# Audit P3 - comments-sidebar-selection-commit

## Scope

- inpage selection resolver 的 root contains 边界保护
- empty selection 回归收敛与全量 `compile/test/build`

## 重点风险

- root contains 判定过严导致合法选区被忽略（特别是跨 shadow/iframe 等边界）
- smoke/unit 覆盖不足导致行为回退到“时间防抖”实现

## 审计记录（执行期填写）

- Finding A1: inpage comments sidebar 使用 Shadow DOM，侧边栏内部选区可能触发“附加选中区”误判。
  - 方案：在 `resolveInpageSelectionPayload` 读取 selection 时增加 `locatorRoot.contains(anchorNode/focusNode)` 判定，不在 root 内则视为空选区并忽略。
  - 状态：已落地（commit: `6888fbad`）。

- Finding A2: 全量测试存在旧语义断言（save/空选区会隐式清空 quote），需要与“只通过 ❌ 清空”对齐。
  - 状态：已更新 controller 单测与空选区回归覆盖，并完成全量验证（commit: `5a39ce8d`）。

## 修复记录（执行期填写）

- Fix A1: inpage selection resolver 增加 root contains 边界保护（commit: `6888fbad`）。
- Fix A2: 收敛 empty selection 相关断言（pointer + keyboard），并更新 controller 旧语义测试（commit: `5a39ce8d`）。

## 回归验证

- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`
