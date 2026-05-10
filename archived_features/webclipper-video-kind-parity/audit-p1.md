# Audit P1 - webclipper-video-kind-parity

- 审计方式：`plan-task-auditor`（由 `executing-plans` 自动进入）
- 审计范围：`plan-p1.md`
- feature 目录：`.github/features/webclipper-video-kind-parity/`
- 粒度：`phase`

## 任务看板（P1）

- [x] P1-T1 Define conversation kind + comment target (`c6253e9c`)
- [x] P1-T2 Make popup/inpage gating kind-aware (`0495f24f`)
- [x] P1-T3 Refactor comments sidebar to use unified target (`06702d3f`)
- [x] P1-T4 Enable app comments sidebar for article/video/chat (`96f6774e`)
- [x] P1-T5 Attach orphan comments on capture flows (`8c250050`)
- [x] P1-T6 Fix video transcript export heading regression (`ok82a2f725`)

## 任务到文件的映射（重点）

- P1-T1
  - `webclipper/src/services/comments/domain/comment-target.ts`
  - `webclipper/src/services/comments/domain/models.ts`
- P1-T2
  - `webclipper/src/viewmodels/popup/usePopupOpenAppCommentsConversation.ts`
  - `webclipper/src/platform/messaging/ui-background-handlers.ts`
  - `webclipper/src/services/bootstrap/inpage-comments-panel-content-handlers.ts`
- P1-T3
  - `webclipper/src/services/comments/sidebar/comments-sidebar-*.ts`
  - `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`
  - `webclipper/src/services/comments/client/repo.ts`
- P1-T4
  - `webclipper/src/ui/app/AppShell.tsx`
  - `webclipper/src/ui/conversations/ConversationsScene.tsx`
  - `webclipper/src/viewmodels/comments/useArticleCommentsSidebarRuntime.ts`
- P1-T5
  - `webclipper/src/services/conversations/background/handlers.ts`
  - `webclipper/tests/unit/conversation-upsert-attaches-orphan-comments.test.ts`
- P1-T6
  - `webclipper/src/services/conversations/domain/markdown.ts`
  - `webclipper/tests/unit/conversation-markdown-video.test.ts`

## 发现项

### 发现 F-01（Low）

- 任务：`P1-T1 / P1-T5`
- 严重级别：`Low`
- 状态：`Deferred`
- 位置：`webclipper/src/services/comments/domain/comment-target.ts:25`
- 摘要：`buildOrphanCommentTargetKeyFromLocation()` 当前只做 `normalizeHttpUrl()`；严格来说无法覆盖 Discourse 与部分 video URL 的“更强 canonicalization”。
- 风险：inpage orphan 以 `url:` 写入时，若后续 capture/upsert 产生另一种 canonical 形态，可能需要依赖 `UPSERT_CONVERSATION` 中的 attach+migrate 才能合并；极端情况下仍可能出现 orphan 未被 attach。
- 预期修复：在 P2 引入统一 `targetKey` store 后，再把 orphan targetKey 的 canonicalization 策略做成 kind-aware（不在 P1 增加新分支逻辑，避免过早复杂化）。
- 验证：手动覆盖 YouTube `youtu.be` / `/watch` 与 Discourse `/t/.../<id>` 两类 URL 的 inpage orphan + capture/upsert 合并行为。

## 修复日志

- 本 phase 未发现 High/Medium 级问题，无需额外修复提交。

## 验证日志

- `rtk npm --prefix webclipper run compile` -> `PASS`
- `rtk npm --prefix webclipper run test` -> `PASS`

## Gate（是否允许进入下一阶段）

- 结论：`Go`
- 理由：P1 验收点（app/popup/inpage/comments/video 导出）均已覆盖，且 orphan attach 已落到 UPSERT_CONVERSATION 的统一入口。

## 最终状态与剩余风险

- 当前状态：`Resolved`
- 剩余风险：见 F-01（Low, Deferred）
