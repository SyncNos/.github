# Audit P2 - syncnos-health-ios

## Audit Scope
- 按 `.github/features/syncnos-health-ios/todo.toml` 的 `P2-*` 任务审查：`P2-T1` ~ `P2-T4`
- 关注：Notion client 重试/错误、parent pages 过滤一致性、HealthKit 能力配置与口径、timeline 渲染可靠性

## Task Board (P2)
- `P2-T1` Notion API client（headers + 错误结构化 + 429 Retry-After）
- `P2-T2` parent pages 拉取 + saved page resolve
- `P2-T3` HealthKit sleep timeline 读取（醒来那天）
- `P2-T4` sleep timeline 渲染组件

## Task → Files
- `P2-T1` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Infra/NotionAPIClient.swift`
- `P2-T1` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Infra/NotionError.swift`
- `P2-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Pages/NotionParentPagesService.swift`
- `P2-T3` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SyncNosForHealth.entitlements`
- `P2-T3` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/HealthKit/SleepTimelineService.swift`
- `P2-T4` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SleepUI/SleepTimelineChart.swift`

## Findings (Read-only)

### 1. 过滤逻辑是否会把 database page 当作 parent
- **Status:** PASS
- **Evidence:** NotionParentPagesService.swift: `if let parent = r["parent"] as? [String: Any], parent["database_id"] != nil { continue }` — database item pages are excluded.

### 2. 429 Retry-After 是否优先
- **Status:** PASS
- **Evidence:** NotionAPIClient.swift: `if let retryAfter = httpResponse.value(forHTTPHeaderField: "Retry-After"), let seconds = Double(retryAfter) { try await Task.sleep(...) }` — Retry-After header is checked first before exponential backoff.

### 3. Notion-Version header 是否固定
- **Status:** PASS
- **Evidence:** NotionAPIClient.swift: `request.setValue("2022-06-28", forHTTPHeaderField: "Notion-Version")` — version is hardcoded in the client.

### 4. sleep timeline 归档口径是否按 endDate（醒来那天）
- **Status:** PASS
- **Evidence:** SleepTimelineService.swift: `let daySamples = samples.filter { sample in let endDate = sample.endDate; return endDate >= dayStart && endDate < dayEnd }` — filtering by endDate (wake-up day).

## 发现 F-01

- 任务：`P2-T2`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Pages/NotionParentPagesService.swift`
- 摘要：`/v1/search` 未按计划加入 `sort: last_edited_time desc`，且 `resolvePage(id:)` 未复用“排除 parent 为 database 的 page”过滤规则。
- 风险：parent pages 列表的“最近编辑”体验与计划不一致；savedPageId 若是 database item page（或误填）会被 resolve 并展示，和“排除 database 子页面”的约束不一致。
- 预期修复：为 search body 增加 `sort`；对 resolve 的结果同样检查 parent 类型并按规则排除。
- 验证：`xcodebuild ... -scheme SyncNosForHealth build`；手动刷新 parent pages 列表顺序更稳定。
- 解决证据：已补 `sort` 参数并在 `resolvePage` 复用 database parent 排除规则；`xcodebuild ... build` ✅（2026-05-10）。

## 发现 F-02

- 任务：`P2-T3`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SyncNosForHealth.entitlements:1`
- 摘要：entitlements 包含 `com.apple.developer.healthkit.access = ["health-records"]`，该值通常用于临床健康记录（Health Records）而非睡眠数据读取。
- 风险：可能触发错误的权限诉求/审查风险；也可能在签名/能力配置上引发非预期行为（请求了不需要的 entitlement）。
- 预期修复：移除 `com.apple.developer.healthkit.access`（保留 `com.apple.developer.healthkit = true` 即可），并核对 Xcode target capability 的 HealthKit 配置仍正确。
- 验证：`xcodebuild ... -scheme SyncNosForHealth build`；真机授权仍可读取 `sleepAnalysis`。
- 解决证据：已移除 `com.apple.developer.healthkit.access`；`xcodebuild ... build` ✅（2026-05-10）。真机读取需后续手动验证。

## 发现 F-03

- 任务：`P2-T3`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SleepUI/SleepDayScene.swift`
- 摘要：sleep timeline 加载前未显式请求 HealthKit 授权（`HealthKitAuthorization.requestAuthorization()` 未被调用）。
- 风险：首次运行时用户可能仅看到“无睡眠数据”或错误提示而不知道需要授权；UX 不符合“可用”闭环要求。
- 预期修复：在加载 timeline 前（例如 `.task` 或首次同步前）发起授权请求，并区分“未授权/无数据”的提示。
- 验证：真机首次运行弹出 Health 权限；授权后能刷新出数据或明确显示无数据。
- 解决证据：已在 `SleepTimelineService.fetchSleepTimeline` 调用 `HealthKitAuthorization.requestAuthorization()`；`xcodebuild ... build` ✅（2026-05-10）。真机授权弹窗需后续手动确认。

## Summary
P2 验收点满足，审计发现项（entitlements、parent pages 行为、授权触发）均已修复并通过 `xcodebuild` 构建验证；真机 HealthKit 授权与数据读取仍需手动复核。

## Verification Log (2026-05-10)
- `cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination "generic/platform=iOS Simulator" build` ✅
