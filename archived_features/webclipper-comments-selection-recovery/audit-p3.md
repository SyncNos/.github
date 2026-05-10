# Audit P3 - webclipper-comments-selection-recovery

## Scope

- 跨浏览器 point API 兜底链路（`caretRangeFromPoint` / `caretPositionFromPoint`）
- 文档事实一致性与兼容性检查闭环

## 重点风险

- 浏览器 point API 差异导致 fallback 失效或异常
- 原因码与真实失败来源不一致，增加排障成本
- 文档与实现漂移，后续扩展白名单时执行成本上升

## 审计记录（执行期填写）

- Finding C1 (Medium) — 审计模板与本阶段真实交付存在偏差：旧清单仍引用不存在的 `comment-selection-site-adapter-registry.test.ts`，会导致审计结论失真。
- Finding C2 (Low) — 对 host gate 可扩展性的误判：代码已提供 `createSelectionRecoveryHostGate({ allowlist })` 参数化扩展位，非纯硬编码不可扩展实现。
- Finding C3 (Low) — 文档存在重复描述风险（AGENTS + deepwiki + compatibility checklist），需明确事实来源层级避免后续漂移。

## 修复记录（执行期填写）

- Fix C1: 已修复。更新本文件 scope/风险/回归命令，改为与 P3 实际实现一致的 compile + fallback tests + build 闭环。
- Fix C2: 已验证。保留当前 host gate 参数化入口，不额外引入 site-adapter 框架（不在本 feature 目标范围）。
- Fix C3: 已修复。同步 `webclipper/AGENTS.md`、`deepwiki/modules/comments.md`、`deepwiki/modules/webclipper.md`，并新增本地 `compatibility-checklist.md` 作为执行清单。

## 回归验证

- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/inpage-selection-recovery-fallback.test.ts tests/unit/comment-selection-recovery-resolver.test.ts tests/unit/comment-selection-recovery-browser-fallback.test.ts tests/unit/comment-selection-coordinate-recorder.test.ts`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`
