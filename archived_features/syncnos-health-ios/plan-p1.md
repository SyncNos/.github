# Plan P1 - syncnos-health-ios

**Goal:** 打通 iOS Notion OAuth 回调链路与 token 安全存储（Keychain），并建立 Health iOS 的 URL scheme 入口，为后续同步闭环打基础。

**Non-goals:** 不实现 HealthKit 读取与写入 Notion；不实现 parent pages 列表与 Settings UI（留到 P2/P3）；不实现后台同步。

**Approach:** 先让 GitHub Pages 回调页按 `state` 前缀路由到 `syncnos-health-ios://`，再在 iOS target 注册 scheme 并接住 openURL。随后实现 iOS 侧 OAuth：发起授权、接收 code/state、通过 CF worker exchange 拿 token 并存 Keychain。

**Acceptance:**
- `state=syncnos_health_ios_...` 能从浏览器回调页跳到 `syncnos-health-ios://oauth/callback?...` 并被 App 接收。
- App 能完成一次授权并在重启后保持“已连接”状态（Keychain）。

**Rules:**
- 所有 shell 命令必须以 `rtk` 前缀运行。
- `.github/features/**` 的计划文件默认不提交到 git。
- 一个 task 一个原子 commit；不要 push。
- 不修改 WebClipper 的 OAuth/state 逻辑；仅保证回调页兼容现状并新增 iOS 路由。

---

## P1-T1 GitHub Pages 回调页按 state 路由到 syncnos-health-ios scheme

**Repo:** `/Users/chii_magnus/Github_OpenSource/chiimagnus.github.io`

**Files:**
- Modify: `src/pages/SyncNosOAuthCallback.tsx`

**Step 1: 实现功能**
- 解析 `state`：
  - `webclipper_` 前缀：保持现状（不跳转自定义 scheme）。
  - `syncnos_health_ios_` 前缀：将 `customScheme` 改为 `syncnos-health-ios://oauth/callback`。
  - 其它前缀：保持现状（仍走 `syncnos://oauth/callback`），避免破坏历史 macOS 行为。

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && npm run build'`

Expected: build 成功；本地 dev 访问 `/syncnos-oauth/callback?code=x&state=syncnos_health_ios_x_1` 会尝试跳 `syncnos-health-ios://...`。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && git add src/pages/SyncNosOAuthCallback.tsx'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && git commit -m \"feat: P1-T1 - OAuth 回调页支持 health iOS scheme 路由\"'`

---

## P1-T2 SyncNosForHealth 注册 URL scheme 并打通 openURL 回调入口

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Modify: `SyncNos.xcodeproj/project.pbxproj`
- Add: `SyncNosForHealth/Info.plist`
- Modify: `SyncNosForHealth/SyncNosForHealthApp.swift`

**Step 1: 注册 URL scheme**
- 为 iOS target 注册自定义 scheme：`syncnos-health-ios`
- 说明：当前 target 使用 `GENERATE_INFOPLIST_FILE = YES`。
  - 本仓库 `SyncNos` target 已采用 `GENERATE_INFOPLIST_FILE=YES` + `INFOPLIST_FILE=SyncNos/Info.plist` 的方式，把 `CFBundleURLTypes` 等自定义项从一个“可版本管理”的 plist 文件注入到最终生成的 Info.plist。
  - `SyncNosForHealth` 沿用同样模式：新增 `SyncNosForHealth/Info.plist`（只放需要显式控制的 key），并在 `project.pbxproj` 为 `SyncNosForHealth` 设置 `INFOPLIST_FILE = SyncNosForHealth/Info.plist`。
  - `SyncNosForHealth/Info.plist` 至少包含：
    - `CFBundleURLTypes`：声明 `syncnos-health-ios` scheme（callback URL 形如 `syncnos-health-ios://oauth/callback`）
    - `NSHealthShareUsageDescription`：先补齐 purpose string，避免后续引入 HealthKit 时因为缺少该 key 导致运行时崩溃（具体 HealthKit 逻辑在 P2/P3）

**Step 2: 保留（可选）openURL 入口**
- OAuth 主链路使用 `ASWebAuthenticationSession` completion（见 P1-T3）。
- `SyncNosForHealthApp` 可选保留 `.onOpenURL`：
  - 用于调试时直接从 Safari 打开 `syncnos-health-ios://...` 验证 URL 解析
  - 不参与 OAuth 主链路的 code/state 收敛（避免两套入口互相打架）

**Step 3: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 4: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNos.xcodeproj/project.pbxproj SyncNosForHealth/Info.plist SyncNosForHealth/SyncNosForHealthApp.swift'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P1-T2 - SyncNosForHealth 注册 syncnos-health-ios URL scheme\"'`

---

## P1-T3 实现 iOS Notion OAuth（ASWebAuthenticationSession + CF exchange + Keychain）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/Notion/Auth/NotionOAuthConfig.swift`
- Add: `SyncNosForHealth/Notion/Auth/NotionOAuthService.swift`
- Add: `SyncNosForHealth/Notion/Keychain/KeychainStore.swift`
- Add: `SyncNosForHealth/Notion/Keychain/NotionTokenStore.swift`
- Modify: `SyncNosForHealth/SyncNosForHealthApp.swift`

**Step 1: 配置常量**
- Notion OAuth authorize：`https://api.notion.com/v1/oauth/authorize`
- CF exchange：`https://syncnos-notion-oauth.chiimagnus.workers.dev/notion/oauth/exchange`
- redirectUri：`https://chiimagnus.github.io/syncnos-oauth/callback`
- state 前缀：`syncnos_health_ios_`
- callback scheme：`syncnos-health-ios`（与 Info.plist 一致；注意传给 ASWebAuthenticationSession 的是 scheme，不含 `://`）

**Step 2: OAuth 发起与收敛**
- 参考 `SyncNos/Services/DataSources-To/Notion/Auth/NotionOAuthService.swift` 的结构：
  - 生成 state（带前缀 + random + timestamp）
  - 打开授权 URL
  - iOS 侧使用 `ASWebAuthenticationSession`
  - completion 收到 callback URL 后提取 `code/state` 并校验 state
- code → token：通过 CF exchange（JSON body 包含 `code` 与 `redirectUri`）
- 细节（避免常见坑）：
  - `ASWebAuthenticationSession` 需要提供 `presentationContextProvider`（提供 `UIWindow` 作为 anchor）。
  - 将 session 存为 `NotionOAuthService` 的实例属性（避免生命周期问题/便于取消）；完成后置空。

**Step 3: Keychain 存储**
- 仅在 Keychain 存敏感信息：`access_token`
- workspaceId/workspaceName（非敏感）可存 UserDefaults（可选），或与 token 一起存 Keychain（更简单）。

**Step 4: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功；可在运行后手动走一遍 OAuth（需要真机/模拟器 Safari 登录）。

**Step 5: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Notion SyncNosForHealth/SyncNosForHealthApp.swift'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P1-T3 - SyncNosForHealth 接入 Notion OAuth 与 Keychain\"'`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Focus:
  - 回调页不会误拉起 iOS（`webclipper_` 不跳）
  - `syncnos_health_ios_` 跳转 scheme 正确
  - token 未写入 UserDefaults（只在 Keychain）
  - iOS build 命令可跑通
