# Plan P2 - syncnos-health-ios

**Goal:** 提供 Health iOS 所需的基础能力：Notion API client + parent pages 列表，以及 HealthKit 睡眠 timeline 读取与渲染基建。

**Non-goals:** 不实现写入 Notion（ensure/upsert 留到 P3）；不实现后台同步。

**Approach:** 参考 `SyncNos-booknotes` 的 Notion 网络/重试实现与 WebClipper 的 parent pages 过滤规则，在 `SyncNosForHealth/` 内实现 iOS 专用最小 Notion client 与 parent pages service。同时实现 HealthKit sleep timeline 读取服务（按“醒来那天”归档）与 SwiftUI 渲染组件（优先复用开源）。

**Acceptance:**
- 能通过 Notion token 调用 `/v1/search` 并得到可用 parent pages 列表（过滤逻辑正确）。
- 429 时优先使用 `Retry-After`，否则指数退避重试（至少覆盖写入与读取一种路径）。
- 能读取某天的 sleep timeline 并在 App 内渲染出区间图。

**Rules:**
- 不引入 `SyncNos/`（macOS 主工程）里的 DIContainer/Logger 依赖到 `SyncNosForHealth/`。
- 所有 shell 命令必须以 `rtk` 前缀运行。
- 一个 task 一个原子 commit；不要 push。
- `.github/features/**` 计划文件默认不提交。

---

## P2-T1 实现 Notion API client（Notion-Version + 错误结构化 + 429 Retry-After）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/Notion/Infra/NotionAPIClient.swift`
- Add: `SyncNosForHealth/Notion/Infra/NotionError.swift`

**Step 1: 实现功能**
- 提供一个中心化请求方法（`performRequest`）：
  - headers：`Authorization: Bearer <token>`、`Notion-Version: 2022-06-28`、JSON content-type
  - 错误结构化：保留 `status`、尽量解析 Notion 的 `code/message/request_id`
  - retry：遇到 429/503（可选包含 409）时重试，优先使用 `Retry-After` 秒数

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Notion/Infra'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P2-T1 - SyncNosForHealth 增加 NotionAPIClient 与错误模型\"'`

---

## P2-T2 实现 parent pages 拉取与 saved page resolve

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/Notion/Pages/NotionParentPagesService.swift`

**Step 1: 实现功能**
- 提供 `listParentPages(savedPageId:)`：
  - `POST /v1/search`：
    - filter page
    - sort last_edited_time desc
  - 过滤条件对齐 WebClipper：排除 archived/in_trash；排除 parent 是 database 的 page
  - 如果 savedPageId 不在当前 search 结果：`GET /v1/pages/{id}` best-effort resolve
  - 去重：按 normalized id（去 `-`、lowercase）

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/Notion/Pages/NotionParentPagesService.swift'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P2-T2 - SyncNosForHealth 支持 Notion 父页面列表\"'`

---

## P2-T3 实现 HealthKit sleep timeline 读取服务（按天归档：醒来那天）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Modify: `SyncNos.xcodeproj/project.pbxproj`
- Add: `SyncNosForHealth/SyncNosForHealth.entitlements`
- Add: `SyncNosForHealth/HealthKit/HealthKitAuthorization.swift`
- Add: `SyncNosForHealth/HealthKit/SleepTimelineService.swift`
- Add: `SyncNosForHealth/HealthKit/SleepModels.swift`

**Step 1: 实现功能**
- 工程配置：
  - 在 `SyncNosForHealth` target 启用 **HealthKit capability**（需要 entitlements）：
    - 增加 `SyncNosForHealth/SyncNosForHealth.entitlements`
    - 在 `project.pbxproj` 为 `SyncNosForHealth` 设置 `CODE_SIGN_ENTITLEMENTS = SyncNosForHealth/SyncNosForHealth.entitlements`
    - 在 `project.pbxproj` 的 target attributes 中开启 `SystemCapabilities` 的 HealthKit（与 Xcode “Signing & Capabilities” 等效）
  - `NSHealthShareUsageDescription`：
    - 已在 P1-T2 的 `SyncNosForHealth/Info.plist` 中补齐；本 task 只需确认文案与实际功能一致即可
- 授权：请求 HealthKit 读取 `HKCategoryTypeIdentifier.sleepAnalysis`
- 查询：
  - 给定某个“醒来那天”的日期（本地时区），构造固定覆盖窗口：
    - `windowStart = dayStart - 18h`
    - `windowEnd = dayStart + 18h`
    - 然后再用 `sample.endDate` 过滤到“醒来那天”（`endDate ∈ [dayStart, dayEnd)`）
  - 说明：睡眠区间通常跨午夜，必须扩展窗口，否则会丢失前一日开始、当日结束的区间。
  - 用 `HKSampleQuery` 拉取 `HKCategorySample` 列表
  - 保留原始区间（start/end/value），并按时间排序
- 归档口径：
  - V1 只要能“选择某天 → 展示该天相关的 timeline”
  - “属于该天”的判断以 `sample.endDate` 的本地日期为准（醒来那天）

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

Manual (Device): 在真机上授权 HealthKit 后，选择某天应能拉到 timeline（至少空/非空两种状态都能正常渲染）。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNos.xcodeproj/project.pbxproj SyncNosForHealth/SyncNosForHealth.entitlements SyncNosForHealth/HealthKit'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P2-T3 - SyncNosForHealth 增加 HealthKit 睡眠 timeline 读取\"'`

---

## P2-T4 实现 sleep timeline 渲染组件（优先复用开源 SwiftUI sleep chart）

**Repo:** `/Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes`

**Files:**
- Add: `SyncNosForHealth/SleepUI/SleepTimelineChart.swift`
- Modify: `SyncNosForHealth/SyncNosForHealthApp.swift`

**Step 1: 实现功能**
- 将 `SleepTimelineService` 的区间事件映射为图表段（按类型着色）。
- 优先复用开源 SwiftUI sleep chart（若引入依赖，必须锁定版本且只引入最小子集）。
- 若无法安全复用第三方依赖，则用纯 SwiftUI Canvas/Shapes 实现最简时间轴（每段矩形块 + 颜色）。

**Step 2: 验证**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && xcodebuild -project SyncNos.xcodeproj -scheme SyncNosForHealth -configuration Debug -sdk iphonesimulator -destination \"generic/platform=iOS Simulator\" build'`

Expected: build 成功。

**Step 3: 原子提交**
Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git add SyncNosForHealth/SleepUI SyncNosForHealth/SyncNosForHealthApp.swift'`

Run: `rtk sh -lc 'cd /Users/chii_magnus/Github_OpenSource/SyncNos/SyncNos-booknotes && git commit -m \"feat: P2-T4 - SyncNosForHealth 增加睡眠时间轴渲染\"'`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Focus:
  - 过滤逻辑是否会把 database page 当作 parent
  - 429 Retry-After 是否优先
  - `Notion-Version` header 是否固定
  - sleep timeline 归档口径是否按 endDate（醒来那天）
