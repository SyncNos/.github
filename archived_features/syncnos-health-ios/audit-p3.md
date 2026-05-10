# Audit P3 - syncnos-health-ios

## Audit Scope
- 按 `.github/features/syncnos-health-ios/todo.toml` 的 `P3-*` 任务审查：`P3-T1` ~ `P3-T5`（含 P3-T3 构建验证）
- 关注：Settings UX/持久化一致性、按日同步幂等、Notion DB ensure 的鲁棒性、工具链可持续性（todo.toml 可解析）

## Task Board (P3)
- `P3-T1` Settings UI（含高级选项）
- `P3-T2` Settings 接入 OAuth/parent pages/持久化
- `P3-T3` 构建验证（npm build + xcodebuild build）
- `P3-T4` 健康数据库 ensure + 按日 upsert（Date+TotalSleepMin）
- `P3-T5` 选中某天 → 同步交互与按钮状态/提示

## Task → Files
- `P3-T1/P3-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Settings/NotionSettingsView.swift`
- `P3-T1/P3-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Settings/NotionSettingsViewModel.swift`
- `P3-T4` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Databases/NotionHealthDatabaseService.swift`
- `P3-T4` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Databases/NotionHealthDatabaseSpec.swift`
- `P3-T4` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Pages/NotionHealthDailyUpsertService.swift`
- `P3-T5` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SleepUI/SleepDayScene.swift`
- `P3-*` `.github/features/syncnos-health-ios/todo.toml`

## Findings (Read-only)

### 1. Settings UI 禁用态是否正确
- **Status:** PASS
- **Evidence:** NotionSettingsView.swift: Refresh button disabled when `!vm.isConnected || vm.isLoadingPages`. Sync toggle only visible in the settings section. Parent page picker bound to availablePages.

### 2. connect/disconnect 是否会遗留旧 token
- **Status:** PASS
- **Evidence:** NotionSettingsViewModel.swift `disconnect()`: calls `NotionTokenStore.clear()` which deletes Keychain token and removes all UserDefaults keys (syncEnabled, parentPageId, parentPageTitle, healthDatabaseIdOverride).

### 3. webclipper_ 回调不误触发 iOS 拉起
- **Status:** PASS
- **Evidence:** SyncNosOAuthCallback.tsx: `isWebClipper` check happens before `isHealthIOS` check (line 50), and the webclipper branch returns early without redirecting.

### 4. upsert 逻辑是否幂等（同一天重复同步不新增多行）
- **Status:** PASS
- **Evidence:** NotionHealthDailyUpsertService.swift: queries for existing page by date first. If found, PATCHes; if not found, creates. If >1 page found, throws error rather than creating duplicates.

## 发现 F-01

- 任务：`P3-T5`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`.github/features/syncnos-health-ios/todo.toml:93`
- 摘要：`todo.toml` 末尾的 `P3-T5` block 使用了智能引号（`“` / `”`），导致 TOML 解析失败（工具链无法按 task 驱动审计/执行）。
- 风险：`executing-plans` / 任何 TOML parser 会直接报错，无法继续自动化；“按 todo.toml 一步一步审查/执行”的前提被破坏。
- 预期修复：将智能引号替换为 ASCII 双引号 `"`，并校验 `tomllib`/`toml` 可正常解析。
- 验证：`python -c 'import tomllib; tomllib.load(open(\".github/features/syncnos-health-ios/todo.toml\",\"rb\"))'` 通过。
- 解决证据：已替换为 ASCII 引号并用 `tomllib` 验证通过（2026-05-10）。

## 发现 F-02

- 任务：`P3-T4`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Databases/NotionHealthDatabaseService.swift`
- 摘要：DB ensure 未按计划实现“override 校验 title 属性 + best-effort ensure schema（PATCH 补齐缺字段）”，目前仅 `GET /databases/{id}` 或复用/创建，缺少 schema 校验与修补。
- 风险：用户填了不可用/不匹配 schema 的数据库 id 时仍可能继续写入失败且错误不清晰；复用到旧数据库但缺字段时会在 upsert 时报错。
- 预期修复：在 `validateDatabase` 解析返回 JSON，至少确认存在 `properties.Date(type=title)` 与 `properties.TotalSleepMin(type=number)`；对复用到的 db 做 `PATCH /databases/{id}` 补齐缺失字段（best-effort）。
- 验证：`xcodebuild ... -scheme SyncNosForHealth build`；连接到旧库/缺字段库时能自动补齐或给出明确错误。
- 解决证据：已实现 `ensureSchema(databaseId:)`：校验 title 属性名必须为 `Date`，缺少 `TotalSleepMin` 时 best-effort `PATCH /databases/{id}` 补齐；override 与复用路径均调用 ensure。`xcodebuild ... build` ✅（2026-05-10）。

## 发现 F-03

- 任务：`P3-T2`
- 严重级别：`Low`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Settings/NotionSettingsViewModel.swift`
- 摘要：计划中提到的 `NotionSettingsStore.swift` 未实现，持久化逻辑散落在 ViewModel 内部（仍可用，但与 plan 文档不一致）。
- 风险：后续扩展更多设置项时可维护性下降；计划文档与实现漂移导致审计困难。
- 预期修复：二选一：A) 按计划补 `NotionSettingsStore` 并迁移读写；或 B) 更新 plan 明确“V1 直接在 VM 内持久化”。
- 验证：`xcodebuild ... build`；重启 app 后状态保持。
- 解决证据：已新增 `NotionSettingsStore.swift` 并将 ViewModel 的 UserDefaults 读写集中到 store；`xcodebuild ... build` ✅（2026-05-10）。

## Summary
P3 核心闭环已满足验收，审计发现项（todo.toml 可解析、DB ensure 校验/补齐、Settings 持久化抽取）均已修复并完成构建验证；真机 OAuth 与 HealthKit 仍需手动走一遍。

## Verification Log (2026-05-10)
- `cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && npm run build` ✅
- `cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination "generic/platform=iOS Simulator" build` ✅
