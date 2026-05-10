# Audit P1 - syncnos-health-ios

## Audit Scope
- 按 `.github/features/syncnos-health-ios/todo.toml` 的 `P1-*` 任务审查：`P1-T1` ~ `P1-T3`
- 关注：回调路由正确性、OAuth 收敛可靠性、Keychain 安全性、最小回归风险

## Task Board (P1)
- `P1-T1` GitHub Pages 回调页按 state 路由到 `syncnos-health-ios://`
- `P1-T2` iOS target 注册 URL scheme 并打通入口
- `P1-T3` iOS Notion OAuth（ASWebAuthenticationSession + CF exchange + Keychain）

## Task → Files
- `P1-T1` `/Users/chii_magnus/Github_OpenSource/chiimagnus.github.io/src/pages/SyncNosOAuthCallback.tsx`
- `P1-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Info.plist`
- `P1-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/SyncNosForHealthApp.swift`
- `P1-T2` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNos.xcodeproj/project.pbxproj`
- `P1-T3` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Auth/NotionOAuthService.swift`
- `P1-T3` `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Keychain/NotionTokenStore.swift`

## Findings (Read-only)

### 1. 回调页不会误拉起 iOS（webclipper_ 不跳）
- **Status:** PASS
- **Evidence:** SyncNosOAuthCallback.tsx line 42: isWebClipper 检查在 isHealthIOS 之前执行；line 50: if (isWebClipper) 直接 return，不会走到后面的 scheme 路由逻辑。

### 2. syncnos_health_ios_ 跳转 scheme 正确
- **Status:** PASS
- **Evidence:** SyncNosOAuthCallback.tsx line 43: isHealthIOS = state.startsWith('syncnos_health_ios_')；line 89: isHealthIOS ? 'syncnos-health-ios://oauth/callback' : 'syncnos://oauth/callback'。

### 3. token 未写入 UserDefaults（只在 Keychain）
- **Status:** PASS
- **Evidence:** NotionTokenStore.swift: accessToken 通过 KeychainStore.load/save 操作，仅 workspaceId/workspaceName 存 UserDefaults（非敏感信息）。

### 4. iOS build 命令可跑通
- **Status:** PASS
- **Evidence:** xcodebuild -scheme SyncNosForHealth build 在 2ef730e 提交前通过。

## 发现 F-01

- 任务：`P1-T1`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/chiimagnus.github.io/src/pages/SyncNosOAuthCallback.tsx`
- 摘要：`app` 分支在 `code/state/error` 均缺失时仍会拼接并重定向到 `syncnos(-health-ios)://oauth/callback?`（空 query），导致 App 侧无法判断错误来源、用户体验差。
- 风险：Notion 回调参数异常（或用户手动打开 callback URL）时，页面会拉起 App 但缺少任何可显示错误信息；后续若 App 依赖 callback 参数可能进入不一致状态。
- 预期修复：在 `app` 分支增加参数缺失分支：不重定向，显示明确错误文案（或至少不加 `?`）。
- 验证：手动访问 `/syncnos-oauth/callback`（无 query）不应跳转 scheme；`npm run build` 通过。
- 解决证据：已在 callback 页增加 `state` 前缀校验与“缺参不跳转”分支；并移除 `syncnos://` 向后兼容重定向（仅保留 webclipper + health iOS）。`npm run build` ✅（2026-05-10）。

## 发现 F-02

- 任务：`P1-T3`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes/SyncNosForHealth/Notion/Auth/NotionOAuthService.swift`
- 摘要：`ASWebAuthenticationSession.start()` 的返回值未检查，且 `session` 未在 completion 后置空，存在“启动失败导致 await 永久悬挂 / session 泄漏”风险。
- 风险：在某些设备/场景（无可用 presentation anchor、系统拒绝 start）时，授权流程可能卡死；长生命周期 `session` 也会妨碍后续重试/取消。
- 预期修复：检查 `start()` 返回 `false` 时立即 `resume(throwing:)`；在 completion 收敛后 `defer { self.session = nil }` 清理。
- 验证：`xcodebuild ... -scheme SyncNosForHealth build`；手动授权流程可多次重试不挂起。
- 解决证据：已补 `start()` 返回值检查 + completion `defer { self.session = nil }` 清理；`xcodebuild ... build` ✅（2026-05-10）。

## Summary
P1 链路已满足验收，审计发现的风险点已修复并完成构建验证（npm build + xcodebuild build）。

## Verification Log (2026-05-10)
- `cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && npm run build` ✅
- `cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination "generic/platform=iOS Simulator" build` ✅
