# SyncNos Health iOS (SwiftUI) - idea

## 背景 / 触发

- 目标：实现 `SyncNosForHealth` iOS SwiftUI App，从 HealthKit 读取 **睡眠 timeline（原始区间事件）** 并在 App 内渲染；支持按日手动同步到 Notion（“一天一行、幂等 upsert”）。
- 关键依赖：沿用现有 Notion OAuth 基础设施：
  - Notion `redirect_uri` 继续使用 GitHub Pages：`https://chiimagnus.github.io/syncnos-oauth/callback`
  - code → token 继续复用 Cloudflare Worker：`/notion/oauth/exchange`
- iOS 的回调 scheme 需要与历史 macOS（`syncnos://`）分离，避免未来多 app 冲突。

## 核心需求（原始需求精炼）

1. iOS App 实现一个 Notion Settings 页面，交互与信息架构参考 WebClipper 的 `NotionOAuthSection`：
   - 顶部：Notion OAuth + 连接状态 + 连接/断开按钮
   - 开关：同步到 Notion
   - 父页面：下拉选择 + 刷新按钮
   - 高级选项：折叠/展开
   - 高级选项内容（Health iOS 业务）：**健康数据库 ID 覆盖**输入框 + 重置按钮 + 备注提示
2. OAuth 回调链路：
   - WebClipper 继续使用 `state` 前缀 `webclipper_`，回调页不得触发 iOS 拉起。
   - Health iOS 使用 `state` 前缀 `syncnos_health_ios_`，回调页将其重定向到 `syncnos-health-ios://oauth/callback?...`。
3. 安全：
   - iOS 的 Notion access token 必须持久化到 **Keychain**（不得用 UserDefaults 明文存 token）。
4. 父页面选择逻辑参考 WebClipper：
   - `POST /v1/search` + 过滤掉不适合作为 parent 的页面（archived/in_trash、database page）
   - 如果已有 saved pageId，需 best-effort resolve 并置顶显示
5. 睡眠数据读取与渲染：
   - 读取 HealthKit `HKCategoryTypeIdentifier.sleepAnalysis` 的原始区间事件（timeline）。
   - App 内提供“按天查看”的睡眠图表渲染；优先复用现成开源 SwiftUI 组件（如睡眠阶段图）。
   - V1 只要求“可看清楚各区间的开始/结束/类型”，无需计算复杂评分。
6. 同步到 Notion（手动触发）：
   - 用户在 App 中选择某一天（按“醒来那天”归档），点击“同步”仅同步该日。
   - V1 Notion schema 极简：`Date(title)` + `TotalSleepMin(number)`（只写汇总，不写 timeline）。

## 默认值与兼容策略

- 默认 `syncEnabled = true`（只影响 UI 与是否允许点击同步；V1 仍由用户手动触发同步）。
- `parentPageId` 未设置时：
  - Notion 未连接：父页面下拉不可用
  - Notion 已连接：允许用户点击刷新拉取候选列表
- 数据库 ID 覆盖为空时：按 parent page 自动 ensure/复用 `SyncNos-Health` 数据库。
- 归档口径：睡眠按样本 `endDate` 的本地日期归到“醒来那天”。
- `TotalSleepMin` 口径（V1）：仅统计 asleep 类样本（`.asleep*`），排除 `.inBed`/`.awake`；对 asleep 区间做“时间并集”去重，避免多数据源重叠导致重复计时。

## 工程约定（避免踩坑）

- 本仓库已有模式：`GENERATE_INFOPLIST_FILE=YES` + `INFOPLIST_FILE=<partial Info.plist>`，用于把 `CFBundleURLTypes` 等复杂结构放进可版本管理文件并在构建时合并；`SyncNosForHealth` 将沿用该模式（见 P1-T2）。

## 非目标（明确不做什么）

- 本 feature 不实现后台同步（不做 BGTaskScheduler/HealthKit background delivery）；只做前台手动同步。
- 不修改 WebClipper 的 OAuth/state 生成逻辑；只要求回调页兼容现状并额外支持 iOS 前缀。
- 不把 macOS 旧 app 行为作为必须保持的主线（允许保留现状，不做迁移）。

## 验收标准（可检查）

- `state=webclipper_...` 命中回调页时，不会跳转到 `syncnos-health-ios://...`。
- `state=syncnos_health_ios_...` 命中回调页时，会跳转到 `syncnos-health-ios://oauth/callback?...`。
- iOS App 能完成 Notion OAuth：用户点击连接 → 授权 → App 收到回调 → 通过 CF worker exchange 拿到 token → 显示“已连接”。
- iOS App 能拉取 parent pages 列表并选择保存；再次打开仍能显示上次选择。
- 高级选项：
  - 可输入“健康数据库 ID 覆盖”并持久化
  - 点击重置会清空覆盖值
- iOS App 能读取某一天的 sleep timeline 并渲染到界面。
- iOS App 对选中的某一天执行 Notion upsert（只写 `Date(title)` + `TotalSleepMin(number)`）。

## 移交备注（给低上下文执行者）

- 所有 shell 命令必须以 `rtk` 前缀运行。
- 计划文件（`.github/features/**`）默认不提交到 git（除非用户另行要求）。
- 本 feature 会涉及 **多个 git 仓库**：
  - `chiimagnus.github.io/`（回调页）
  - `SyncNos/SyncNos-booknotes/`（iOS app 与 Swift Notion 代码参考/复用）
- iOS token 只能进 Keychain；UserDefaults 只能存非敏感配置（开关、parentPageId、db override）。
