**最新决策（2026-05-10）**：React Native iOS App + 共享 Notion TS 代码。推翻此前「不做 iOS app」和「SwiftUI 一把梭」两版方案。

# 需求 A — SyncNos-Health（数据搬运）

**一句话**：RN iOS App，从 HealthKit 读健康数据，复用 WebClipper 的 Notion API client 写入 Notion 数据库。一天一行，全量 + 幂等 upsert。

## 为什么用 React Native

- **核心理由**：复用 WebClipper 已验证的 Notion API TS 代码（`@syncnos/notion-core`），一次修 bug 两端生效，不维护两套
- HealthKit → [react-native-health](https://github.com/agencyenterprise/react-native-health)，成熟社区库
- Liquid Glass → [@callstack/liquid-glass](https://github.com/callstack/liquid-glass)，iOS 26 原生效果
- SyncNos-Health 仓库当前只有 Swift 脚手架，切换零成本

## 代码共享架构

```
syncnos/
├── SyncNos-Webclipper     ← 依赖 @syncnos/notion-core
├── SyncNos-Health (RN)    ← 依赖 @syncnos/notion-core
├── SyncNos-booknotes
├── SyncNos-CLI
└── syncnos-notion-core    ← 新建：Notion API client 共享包
```

从 WebClipper 的 `src/services/sync/` 抽离 Notion 请求层（OAuth、rate limit、重试、数据格式），发布为 npm 包。增量同步策略、浏览器 storage、DOM 逻辑留在 WebClipper。

## 数据模型（Notion 数据库，一天一行）

| 字段 | 类型 | HealthKit 标识符 |
| --- | --- | --- |
| 日期 | date (title) | — |
| 步数 | number | StepCount |
| 活动能量 (kcal) | number | ActiveEnergyBurned |
| 锻炼时长 (min) | number | AppleExerciseTime |
| 站立小时 | number | AppleStandHours |
| 睡眠（总/深睡/REM/核心） | number ×4 | SleepAnalysis |
| 心率（静息/平均/HRV） | number ×3 | RestingHeartRate / HeartRate / HRVSDN |
| VO2Max / 体重 / 正念分钟 | number ×3 | VO2Max / BodyMass / MindfulSession |
| 锻炼类型 | multi_select | WorkoutActivityType |
| 关联日记 / 备注 | relation / text | — |

## 执行步骤

1. 新建 `syncnos-notion-core` 仓库，抽离 Notion API client
2. SyncNos-Health 清掉 Swift 脚手架，RN 初始化
3. 接入 react-native-health + @syncnos/notion-core，跑通全链路
4. UI 打磨：Liquid Glass + 原生 Tab Bar
