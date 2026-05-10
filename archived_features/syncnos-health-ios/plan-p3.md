# Plan P3 - syncnos-health-ios

**Goal:** 在 iOS SwiftUI 中完成可用的睡眠查看与“选中某天→同步到 Notion”闭环（含 Notion 设置页）。

**Non-goals:** 不实现后台同步（BGTaskScheduler / background delivery）。

**Approach:** 用 SwiftUI + 简单 ViewModel（最小实现为先）实现两块 UI：睡眠查看页（按天选择 + 时间轴展示）与 Notion 设置页。将 OAuth/parent pages/Notion 写入与 HealthKit 读取能力注入 ViewModel。同步入口固定为“选中某天→同步”。

**Acceptance:**
- 页面结构与 WebClipper NotionOAuthSection 对齐（状态/开关/父页面/高级覆盖）。
- 用户可连接/断开 Notion；可刷新并选择父页面；高级覆盖可保存与重置；应用重启后状态仍正确。
- 用户可选择某一天并查看 sleep timeline。
- 用户可对选中日执行 Notion upsert（只写 Date+TotalSleepMin），成功/失败有明确反馈。

**Rules:**
- 不把计划文件提交到 git。
- 所有 shell 命令必须以 `rtk` 前缀运行。
- 一个 task 一个 commit；不要 push。
- Settings 页不得在主线程执行网络请求。

---

## P3-T1 实现 SwiftUI Notion 设置页（含高级选项：健康 DB 覆盖/重置）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/Settings/NotionSettingsView.swift`
- Add: `SyncNosForHealth/Settings/NotionSettingsViewModel.swift`
- Modify: `SyncNosForHealth/SyncNosForHealthApp.swift`

**Step 1: 实现 UI**
- 顶部：Notion OAuth 标题 + 连接状态文本（未连接/连接中/已连接/错误）+ 按钮（连接/断开）
- 开关：同步到 Notion（仅持久化，不实现同步逻辑）
- 父页面：Picker/菜单 + 刷新按钮（禁用条件对齐 WebClipper：未连接/加载中/busy）
- 高级选项：DisclosureGroup
  - 输入框：健康数据库 ID 覆盖（placeholder 给一个类似 Notion 数据库 id 的示例，如 `35cbe9d6-386a-8142-8665-ed2d90d9970c`）
  - 按钮：重置（清空覆盖）
  - 备注：说明“填写后直接使用该数据库；清空后恢复按父页面自动复用/创建”

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Settings SyncNosForHealth/SyncNosForHealthApp.swift'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P3-T1 - SyncNosForHealth 新增 Notion 设置页 UI\"'`

---

## P3-T2 设置页接入 OAuth/parent pages/持久化配置与 UI 状态

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Modify: `SyncNosForHealth/Settings/NotionSettingsViewModel.swift`
- Add: `SyncNosForHealth/Settings/NotionSettingsStore.swift`

**Step 1: 状态与持久化**
- UserDefaults 存：
  - `syncEnabled`（Bool）
  - `parentPageId`（String）
  - `parentPageTitle`（String，可选）
  - `healthDatabaseIdOverride`（String）
- Keychain 存：
  - `notionAccessToken`
- ViewModel 初始化时读取并生成 UI 状态文本。

**Step 2: 行为接入**
- Connect：
  - 调 `NotionOAuthService` 完成授权与 exchange
  - 成功后写 Keychain + 更新 workspaceName（可选）
- Disconnect：
  - 清空 Keychain token
  - 清空父页面/覆盖字段（按你希望的 UX 决定是否清空；默认建议清空）
- Refresh pages：
  - 调 `NotionParentPagesService.listParentPages(savedPageId:)`
  - 把 saved page 置顶显示
- Save parent page：
  - 写入 UserDefaults

**Step 3: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功；运行后可在 UI 中完成 connect/disconnect 与页面刷新。

**Step 4: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Settings'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P3-T2 - Notion 设置页接入 OAuth 与父页面逻辑\"'`

---

## P3-T3 补齐构建验证命令与最小回归检查（iOS Simulator + GitHub Pages build）

**Files:**
- None (documentation-only verification steps recorded in this plan)

**Step 1: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/chiimagnus.github.io && npm run build'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: 两个 build 都成功。

**Step 2: 原子提交**
- 本 task 不改代码，不需要提交。

---

## P3-T4 实现 Notion 健康数据库 ensure + 按日 upsert（Date+TotalSleepMin）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/Notion/Databases/NotionHealthDatabaseSpec.swift`
- Add: `SyncNosForHealth/Notion/Databases/NotionHealthDatabaseService.swift`
- Add: `SyncNosForHealth/Notion/Pages/NotionHealthDailyUpsertService.swift`

**Step 1: DB ensure（按 parent page）**
- 数据库标题：`SyncNos-Health`（可后续调整）
- schema（V1 极简）：
  - `Date`（title）
  - `TotalSleepMin`（number）
- 逻辑：
  - 若用户填写了健康数据库 ID 覆盖：
    - 先 `GET /v1/databases/{id}` 做一次校验（至少确认可访问且包含 title 属性）
    - 通过校验后跳过 ensure，直接使用该 id
  - 否则：
    - `POST /v1/search`（filter database，query `SyncNos-Health`），然后在客户端按 `parent.page_id == parentPageId` 过滤
    - 找到同名且 parent 匹配则复用；否则创建 database（parent 为 page）
    - best-effort ensure schema（缺字段则 `PATCH /v1/databases/{id}` 补齐）

**Step 2: Upsert（日维度）**
- 归档 key：`YYYY-MM-DD`（本地时区，以“醒来那天”计算）
- 查询：`POST /v1/databases/{id}/query` 按 `Date(title)` 精确匹配（示例 filter：`{ "property": "Date", "title": { "equals": "2026-05-10" } }`）
- 若命中 0 条：创建 page
- 若命中 1 条：PATCH 更新 `TotalSleepMin`
- 若命中 >1 条：视为异常，返回明确错误（不自动 merge）

**Step 3: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 4: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Notion/Databases SyncNosForHealth/Notion/Pages'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P3-T4 - SyncNosForHealth 支持 Notion 健康库 ensure 与按日 upsert\"'`

---

## P3-T5 实现“选中某天→同步”交互与同步按钮状态/错误提示

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Modify: `SyncNosForHealth/SyncNosForHealthApp.swift`
- Add or Modify: `SyncNosForHealth/SleepUI/SleepDayScene.swift`
- Modify: `SyncNosForHealth/Settings/NotionSettingsViewModel.swift`

**Step 1: 选中日期**
- 提供一个日期选择器（或最近 N 天列表），选中某天后加载该天 sleep timeline 并渲染。

**Step 2: 同步按钮**
- 按钮禁用条件：
  - Notion 未连接或 syncEnabled 为 false
  - 正在同步中
  - 当天没有可用 sleep 数据（可选：允许同步 0，写 0）
- 点击后：
  - 计算当天 `TotalSleepMin`（建议策略）：
    - 仅统计 asleep 类样本（`.asleep` / `.asleepUnspecified` / `.asleepCore` / `.asleepDeep` / `.asleepREM`），排除 `.inBed` / `.awake`
    - 对 asleep 区间做“时间并集”去重（避免多数据源重叠导致重复计时）
  - 调用 `NotionHealthDailyUpsertService` upsert
  - 展示成功/失败反馈（alert/toast）

**Step 3: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 4: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/SyncNosForHealthApp.swift SyncNosForHealth/SleepUI SyncNosForHealth/Settings'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P3-T5 - SyncNosForHealth 支持按日选择并同步到 Notion\"'`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Focus:
  - Settings UI 禁用态是否正确（未连接不可刷新/不可选父页面等）
  - connect/disconnect 是否会遗留旧 token
  - `webclipper_` 回调不误触发 iOS 拉起
  - upsert 逻辑是否幂等（同一天重复同步不新增多行）
