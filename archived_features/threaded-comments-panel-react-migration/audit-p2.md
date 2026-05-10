# Audit P2 - threaded-comments-panel-react-migration

> Phase 审计闭环：先记录发现，再修复问题，再跑本 phase 验证命令。

## Findings

### 任务看板（来自 `todo.toml`）

- `P2-T1` React 组件骨架
- `P2-T2` panel store + useSyncExternalStore bridge
- `P2-T3` React keyed threads/replies
- `P2-T4` root composer 迁移
- `P2-T5` focus + scroll 闭环
- `P2-T6` delete 二次确认
- `P2-T7` locate 迁移
- `P2-T8` header Chat with AI 菜单
- `P2-T9` comment-level Chat with AI 菜单
- `P2-T10` 默认切到 React
- `P2-T11` 删除 legacy DOM

### 发现 F-01

- 任务：`P2-T5 | P3-T2`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p2.md:166`
- 摘要：计划写了“activeElement 在 panel 内才 focus”，但没有考虑 Shadow DOM 下 `document.activeElement` 退化为 host、真正焦点在 `shadowRoot.activeElement` 的事实。
- 风险：实现时若用 `document.activeElement` 判断，会导致“不要抢焦点”的规则永远误判（要么总抢，要么永不执行），属于核心验收点风险。
- 预期修复：在 `plan-p2.md` / `plan-p3.md` 明确：focus gating 需要基于 `shadowRoot.activeElement` 与 host 的组合判断，或改为“显式记录 panel 内 focusin/focusout 信号”。
- 验证：人工：发送 root/reply 时在 panel 外点击页面，确认不会强行抢回；以及正常路径仍会 focus+scroll。
- 解决证据：已更新 `plan-p2.md`（补齐 Shadow DOM 焦点判定建议与实现路径）。

### 发现 F-02

- 任务：`P2-T2`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p2.md:60`
- 摘要：P2-T2 写了“允许短期双 root 但不展示重复 UI”，但没有给出可执行的迁移策略（哪一套 UI 负责渲染？如何保证不重复？如何在切换前验证 React？）。
- 风险：执行时容易出现重复渲染/事件冲突/关闭逻辑互相干扰，或者为了避免重复而把 React 隐藏导致缺少可验证路径；最终拖慢迁移、增加回归概率。
- 预期修复：在 `plan-p2.md` 给出确定策略（推荐：P2-T1~T9 完成 React feature parity 后，P2-T10 一次性切换；P2-T11 删除 legacy；期间不做“同时可见”的双 UI）。
- 验证：`npm --prefix webclipper run compile` + P3 手工冒烟清单。
- 解决证据：已更新 `plan-p2.md`（新增 Migration strategy + 明确“不允许双显”与策略 A/B）。

### 发现 F-03

- 任务：`P2-T1 | P2-T7 | P2-T8`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p2.md:27`
- 摘要：现有 panel 有 `notice` 区域（locate 失败、chatwith action 返回 message 等），计划没有明确把 notice UI 迁移到 React。
- 风险：locate/chatwith 的失败提示在 React 版中缺失，属于“功能不回退”的验收点风险。
- 预期修复：在 `plan-p2.md` 明确：React 组件需要包含 notice 区 + `showNotice(message)` 的 props/回调，并在 panel.ts 复用现有计时隐藏语义。
- 验证：手工：触发 locate 失败/ChatWith action message，确认 notice 出现并自动消失。
- 解决证据：已更新 `plan-p2.md`（P2-T1 明确包含 notice 区；P2-T2 store 增加 notice 字段）。

### 发现 F-04

- 任务：`P2-T9 | P2-T11`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p2.md:283`
- 摘要：P2-T9 中“可选删除 render.ts”的描述会导致任务边界不确定，影响执行顺序与审计闭环。
- 风险：`executing-plans` 执行时无法稳定判断该 task 的完成态；容易出现重复删除/残留调用链等回归。
- 预期修复：把 `render.ts` 的删除固定到 `P2-T11`，P2-T9 只做 comment-level chatwith 迁移。
- 验证：`npm --prefix webclipper run compile`
- 解决证据：已更新 `plan-p2.md`（把 `render.ts` 删除固定留到 P2-T11；P2-T9 不再包含可选删除）。

## Fixes

- `plan-p2.md` 已补齐：
  - 迁移策略（避免双显/明确切换点）
  - notice 迁移的显式要求
  - Shadow DOM 下的焦点 gating 注意点
  - P2-T9 与 P2-T11 的职责边界确定化

## Verification

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。

---

## Round 2 (post P2 completion)

Subagent evidence: `.github/features/threaded-comments-panel-react-migration/.audit/subagent-audit-p2.md`

### 发现 F-05

- 任务：`P2-T11`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- 摘要：save/reply/delete 异步操作仅依赖 React busy 状态，缺少同步重入锁；极快重复点击存在重复触发风险。
- 风险：可能出现重复保存、重复回复或重复删除请求。
- 修复：加入 `actionInFlightRef` 同步互斥；在进入异步动作前立即加锁，并在 `finally` 释放。
- 证据：commit `227995c8`.

### 发现 F-06

- 任务：`P2-T9 | P2-T11`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`webclipper/src/ui/comments/react/ThreadedCommentsPanel.tsx`
- 摘要：comment-level Chat with 菜单缺少并发请求序列控制；重复点击可能让旧请求结果覆盖新状态。
- 风险：菜单状态错乱、过时请求回写、用户操作时序不稳定。
- 修复：增加 `commentChatWithLoadingRef` 与 `commentChatWithRequestIdRef`，只接受最新请求结果并忽略卸载后回写。
- 证据：commit `227995c8`.

## Round 2 Verification

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。
