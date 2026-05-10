# Audit P1 - comments-sidebar-selection-commit

## Scope

- `ThreadedCommentsPanel` selection 触发链路从时间防抖迁移到 pointer/keyboard 完成事件
- P1 删除老旧 timer/suppress 逻辑后的正确性与残留扫描

## 重点风险

- 事件时序：`selectionchange` 与 `pointerup/keyup` 的读取时机不同步导致漏触发或读到旧 selection
- handler 安装时机：面板打开早期 handlers 还未 set 时的 pending 行为回归
- dedupe：重复提交导致冗余请求或反过来阻止必要更新

## 审计记录（执行期填写）

- Finding A1: `requestAnimationFrame` 在部分运行环境（Vitest/JSDOM globals）可能不存在，导致 commit 调度异常。
  - 影响：auto-attach 在非浏览器环境下无法触发（测试与可能的极端运行时）。
  - 处理：在 commit 调度层加入 next-frame fallback（优先 `requestAnimationFrame`，否则退化到 `setTimeout(0)`），保持“事件驱动提交”语义，不回退到时间防抖。
  - 状态：已修复（commit: `cbc348c0`）。

## 修复记录（执行期填写）

- Fix A1: 引入 `scheduleNextFrame/cancelNextFrame` 包装并覆盖关闭/卸载清理路径（commit: `cbc348c0`）。

## 回归验证

- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`（P1-T1）
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/threaded-comments-panel-auto-attach-selection.test.ts`（P1-T2）
