# Audit P1 - webclipper-notion-parent-page-refresh

> 完成 P1 全部 tasks 后填写：记录审计发现、修复动作与验证证据。

## Findings

- `useSettingsSceneController` 的 `runTask()` 队列存在“task 内 await refresh()”的自我等待风险，会导致 connect/disconnect 状态不更新直到刷新插件页面（已在 P1-T5 修复）。
- Notion `GET_AUTH_STATUS` 返回值与 UI 读取字段不一致（UI 读取 `workspaceName`，但后台仅返回 `token`），会导致状态文本缺失 workspace（已在 P1-T5 修复）。
- 迁移完成后需要确保 Settings 不再直连 Notion API，且 `ui/viewmodels` 不引入 `@platform/*`（已验证通过）。

## Fixes

- `fix: task5 - 修复 Notion connect/disconnect 状态无需刷新即可更新`（`2a469623`）
- `refactor: task1 - 收敛 Notion parent pages discovery 到 services 单一真源`（`f503b828`）
- `refactor: task2 - 通过 background message 暴露 Notion parent pages 列表`（`90a12b4d`）
- `refactor: task3 - Settings 改走 background 列表并删除 Notion 直连实现`（`42f15685`）
- `refactor: task4 - Notion Parent Page 刷新增加可见 loading 并移除 clickRefresh 占位`（`a76cc6cd`）

## Verification

- Run: `npm --prefix webclipper run compile`
- Run: `rg -n "@platform/" webclipper/src/ui webclipper/src/viewmodels`
