# Audit P2 - webclipper-comments-selection-recovery

## Scope

- inpage 评论侧栏对选区恢复解析器的实际接入
- 白名单命中/未命中与降级路径行为一致性

## 重点风险

- `selectionchange` 与指针轨迹时序错位导致偶发空引用
- 非白名单站点行为被意外改变
- reply 输入框交互误触发引用变更

## 审计记录（执行期填写）

- Finding B1 (Medium) — `selection-recovery-resolver.ts` 在 `root=null` 时复用了 `outside-root` 原因码，语义与真实失败来源不够一致。
- Finding B2 (Medium) — 指针轨迹过期边界缺少显式测试，无法证明 `maxAgeMs` 临界点行为稳定。
- Finding B3 (Low) — app/popup 与 inpage 选区恢复策略差异缺少就地注释，维护者易误解能力边界。

## 修复记录（执行期填写）

- Fix B1: 已修复。为 resolver 增加 `root-null` 原因码，并补充 `inpage-selection-recovery-fallback.test.ts` 覆盖。
- Fix B2: 已修复。新增 `comment-selection-coordinate-recorder.test.ts` 的 `maxAge` 临界点测试（=边界可用，>边界失效）。
- Fix B3: 已修复。`article-comments-sidebar-controller.ts` 增加注释，明确 inpage/app/popup 的 resolver 差异。

## 回归验证

- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/threaded-comments-panel-auto-attach-selection.test.ts tests/unit/inpage-selection-recovery-fallback.test.ts`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`
