# Audit P1 - webclipper-comments-sidebar-unified-controller

## Implementation Audit (2026-03-24)

- Status: ✅ P1 tasks completed（controller+adapters+接入+测试+验证链均已落地）
- Verification:
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/unit/comment-sidebar-session.test.ts tests/unit/article-comments-sidebar-chrome.test.ts tests/unit/article-comments-sidebar-controller.test.ts`
- Key changes (single source of truth):
  - Shared controller: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
  - App adapter (repo): `webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`
  - Inpage adapter (messaging): `webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`
  - App entry wiring: `webclipper/src/ui/app/AppShell.tsx` + `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
  - Inpage entry wiring: `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`

### Notes / Follow-ups

1.（Low）`ArticleCommentsSection` 仍保留 embedded（窄屏 inline）路径的自管理逻辑；本 phase 仅承诺统一 inpage + app 右侧栏，embedded 不在 P1 强制统一范围内。
2.（Low）inpage 的 `ensureArticle: false` 分支当前会让 save 流程也跳过 capture（与旧实现“save 时兜底 capture”略有差异）；目前入口默认 `ensureArticle: true`，风险较低，如未来出现调用方使用该分支再补齐策略。

## Findings

### High

0.（Plan 结构问题）`todo.toml` 的 `plan = "plan-p1.md#p1-tN"` 依赖 **heading 锚点稳定**；原 plan 采用 `## P1-T1 xxx` 形式会生成 `#p1-t1-xxx` 的锚点，导致执行器/人类回看时无法从 `todo.toml` 直接跳转到对应 task。
   - Impact: 执行流工具链无法稳定定位 task；review/执行过程容易走偏。
   - Recommendation: 将每个 task heading 固定为 `## P1-TN`（不在 heading 行追加描述），把描述放到 `### 任务：...`。
   - Status: 已在 `plan-p1.md` 按该规则重写。

1. `onSave` 的“成功/失败”契约需要明确：目前 UI 面板（`threaded-comments-panel`）在 `await handler(text)` 返回后会清空 composer；如果 handler 用 `return false` 表达失败，会导致 **失败也清空输入框**。
   - Impact: 用户输入丢失，且 quote 生命周期/错误处理会漂移；两端难以做到“只入口不同”。
   - Recommendation: 统一约定 `onSave` **失败必须 throw**，成功返回 `true` 或 `{ ok: true }`。并在计划中明确 adapter/controller 的实现方式（不可返回 `{ ok: false }` 代表失败）。
   - Affected: `webclipper/src/services/comments/threaded-comments-panel.ts`、controller/adapter 设计、inpage/app 两端 handler 实现。
   - Status: 已在 `plan-p1.md` 明确为“成功必须返回 `true`；失败 throw 或返回 `false/void`（但不要伪装成功）”，并将 quote 清理归属锁定在 session wrapper。

2. 当前 plan 的任务边界偏大，且没有把“controller 只管业务流程、panel 只管 UI 挂载、session 只管状态同步”写成硬约束，容易在执行中再次把逻辑塞回 UI 组件。
   - Impact: 重构目标（入口不同、流程统一）会在实现过程中被侵蚀，后续又出现分叉。
   - Recommendation: 更新 plan，把每个 task 的写入范围收窄到单一职责（controller/adapter、app 迁移、inpage 迁移、quote 契约、测试/验证）。
   - Status: 已把 plan 重写为 `P1-T1..T8`，将 adapter/接入/测试/验证拆分，并在 Approach 中把职责边界写成硬约束。

### Medium

1. Adapter 接口需要覆盖 inpage 的 `ensureContext`（resolve/capture + attach orphan）与 app 的“已有 conversationId/canonicalUrl”两种模式，但计划中对“context state 存放位置/刷新触发时机”描述偏粗。
   - Recommendation: 在 controller 中显式维护 `activeCanonicalUrl / activeConversationId`，并在 `open()` / `refresh()` 的步骤里写清更新顺序与失败回退。
   - Status: 已在 `plan-p1.md` 的 `P1-T1` 明确 controller 持有 `activeContext`，并把 ensureContext 与 refresh 顺序写清楚。

2. 验证链应包含“app 右侧栏不会反复 mount/unmount 导致交互冻结”的回归点（`openRequested` 消费语义）。
   - Recommendation: 在 plan 的 app task 中保留 `openRequested || isOpen` 的渲染条件要求，并加一条 targeted test 说明。
   - Status: 已在 `plan-p1.md` 的 `P1-T3/P1-T7` 固化要求与测试回归点。

### Low

1. Plan 中 adapter 返回 `{ ok: boolean }` 的写法容易诱导实现者返回 `ok:false`，与 High-1 冲突。
   - Recommendation: plan 改为“成功返回 ok true；失败 throw（不返回 ok false）”。
   - Status: 已在 `plan-p1.md` 将 handler 契约收敛为“成功返回 `true`；失败 throw 或返回 `false/void`”，避免引入 `{ ok: false }` 模式。

## Checklist

- [ ] inpage 与 app 右侧栏只入口不同，open/save/refresh/quote 生命周期同一份 controller 流程
- [ ] root comment 保存成功后 quote 清空（两端一致）
- [ ] reply 不携带 quote（两端一致）
- [ ] 保存/删除后列表刷新（两端一致）
- [ ] busy/openRequested/isOpen 语义稳定，不出现右侧栏抖动或交互冻结
- [ ] 分层约束满足：UI 不 import platform；content script 不 import UI/repo；services 不依赖 ui/viewmodels
- [ ] 验证链通过：compile + targeted tests

## Evidence

- 验证命令：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test -- tests/smoke/app-shell-comments-sidebar.test.ts tests/smoke/background-router-open-comments-sidebar.test.ts tests/unit/comment-sidebar-session.test.ts tests/unit/article-comments-sidebar-chrome.test.ts`
- 关键实现点（供回看）：
  - 共享 controller：`webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
  - app adapter：`webclipper/src/services/comments/sidebar/article-comments-sidebar-app-adapter.ts`
  - inpage adapter：`webclipper/src/services/comments/sidebar/article-comments-sidebar-inpage-adapter.ts`
  - inpage 入口：`webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
  - app 入口：`webclipper/src/ui/app/AppShell.tsx`

## Follow-ups

- 如果后续发现 app 与 inpage adapter 仍在边界条件上漂移（例如 ensureContext 的 retry/timeout），考虑补齐 controller 的统一策略与更明确的 adapter 契约。
