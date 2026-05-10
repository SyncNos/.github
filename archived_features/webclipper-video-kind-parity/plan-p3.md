# Plan P3 - webclipper-video-kind-parity

**Goal:** Notion/Obsidian 同步对 `video/chat` 的 comments 与 `article` 同级（layout + digest + rebuild），并完成 comments 命名去 “article-only” 的清理。

**Non-goals:**
- 不做 Notion/Obsidian 端的高级交互（例如双向同步、在 Notion 编辑后回写）

**Approach:**
- 更新 Notion layout spec：video/chat 也包含 `comments` section（video 仍保留 Transcript section）。
- notion/obsidian orchestrator：抽象 “ensure comments loaded + digest + rebuild section” 的流程，不再绑定 `kind.id === 'article'`。
- 清理命名：逐步把 `ArticleComments*` 改成 `Comments*`，减少后续新功能继续误导。

**Acceptance:**
- Notion/Obsidian 同步对 video/chat 产生 comments section，且 comments 变化会触发更新（digest 变更）。
- 代码中不再存在 “comments 只能用于 article” 的硬编码闸门（除非是明确的产品限制）。

**Rules:**
- 所有命令使用 `rtk ...`
- 一个 task 一个 commit

---

## P3-T1 Make Notion layout/sync kind-aware for comments

**Files:**
- Modify: `webclipper/src/services/sync/notion/notion-managed-sections.ts`
- Modify: `webclipper/src/services/sync/notion/notion-sync-orchestrator.ts`
- Modify: `webclipper/src/services/comments/sync/notion-comments-renderer.ts`
- Modify: `webclipper/src/services/comments/domain/comment-metrics.ts`
- Modify: `webclipper/src/services/sync/notion/notion-services.ts`（storage 接口重命名）
- Modify: `webclipper/src/services/conversations/background/storage.ts`（暴露通用 comments storage）

**Step 1: 实现功能**
- layout：
  - `article`：保持 `Article + Comments`
  - `video`：`Transcript + Comments`
  - `chat`：`Conversations + Comments`
- orchestrator：
  - 抽象出 `ensureCommentsLoaded(kind, convo, id)`（不再仅 article）
  - 对存在 `comments` section 的 kind：计算 digest，并在变化时 rebuild comments section
  - **注意 create-flow 与 update-flow 都要处理**：
    - 当前 `kind=article` 有专用 “创建 Article + Comments section” 分支；但 `chat/video` 创建页与更新页都需要同样的 section heading + mapping anchors + digests 机制
- renderer/metrics：
  - 类型从 `ArticleComment` 泛化到 `Comment`，并以 `CommentTargetKey`（`url:` / `convo:`）作为加载/聚合边界

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: notion sync 相关 smoke/unit 全绿；新增覆盖 video/chat comments digest 更新路径（按现有测试风格补齐）。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/sync/notion webclipper/src/services/comments webclipper/src/services/conversations/background/storage.ts`

Run: `rtk git commit -m "feat: P3-T1 - Notion同步对video/chat comments同级化"`

---

## P3-T2 Make Obsidian sync kind-aware for comments

**Files:**
- Modify: `webclipper/src/services/sync/obsidian/obsidian-sync-orchestrator.ts`
- Modify: `webclipper/src/services/sync/obsidian/*`（按实际渲染/写入路径定位）

**Step 1: 实现功能**
- 与 Notion 同步保持一致：
  - 读取 comments（通用 storage）
  - 写入到输出 markdown（或 metadata block）中的稳定区域
  - comments 变化触发更新

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run test`

Expected: obsidian sync 测试全绿，新增 video/chat comments case（如有现成测试入口）。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src/services/sync/obsidian`

Run: `rtk git commit -m "feat: P3-T2 - Obsidian同步对video/chat comments同级化"`

---

## P3-T3 Cleanup: remove article-only naming for comments

**Files:**
- Rename/Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`
- Modify: `webclipper/src/ui/comments/locate.ts`
- Modify: `webclipper/src/services/comments/locator/*`
- Modify: `webclipper/tests/**`（更新引用）

**Step 1: 实现功能**
- 把 `ArticleCommentsSection` 等外露符号替换为中性命名（例如 `CommentsSection`）
- 把 `buildArticleCommentLocatorFromRange` / `restoreRangeFromArticleCommentLocator` 改为中性命名（可保留旧别名一段时间）
- 删除不再使用的 legacy wrapper（在确保无调用点后）

**Step 2: 验证**
Run: `rtk npm --prefix webclipper run compile && rtk npm --prefix webclipper run test && rtk npm --prefix webclipper run build`

Expected: 全部通过。

**Step 3: 原子提交**
Run: `rtk git add webclipper/src webclipper/tests`

Run: `rtk git commit -m "refactor: P3-T3 - comments命名去article化并更新测试"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule (gate): **P3 全部 tasks 完成后，必须做审计并完成全量验证后才算收尾**。
- Tooling:
  - 若使用 `$executing-plans` 执行，将在 phase 结束后**自动**进入审计闭环（使用 `$plan-task-auditor` 的流程）。
  - 若未使用 `$executing-plans`，则在 P3 结束时**手动**运行 `$plan-task-auditor`，以本目录为审计目标，输出到 `audit-p3.md`。
- Audit flow (strict):
  1. 只读审查并记录发现到 `audit-p3.md`（此阶段不改代码）
  2. 修复发现项（按 High → Medium → Low）
  3. 运行并记录验证命令（`rtk npm --prefix webclipper run compile && rtk npm --prefix webclipper run test && rtk npm --prefix webclipper run build`），并记录潜在回归点（popup/inpage/app 三入口都要覆盖）
