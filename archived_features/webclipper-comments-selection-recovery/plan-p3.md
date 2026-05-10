# Plan P3 - webclipper-comments-selection-recovery

**Goal:** 完成跨浏览器兜底硬化与文档收敛，确保方案在生产环境可持续维护。

**Non-goals:**
- 不引入 page-world 深度注入
- 不新增设置项或实验开关
- 不改变 P1/P2 已确认交互语义

**Approach:**
P3 聚焦“稳定而非扩张”：先做跨浏览器 API 能力硬化（特别是 `caretRangeFromPoint` 缺失路径），再统一沉淀兼容性核对清单与文档事实，避免后续新增站点时重复踩坑。

**Acceptance:**
- API 不可用或行为不一致时可稳定降级，不抛异常、不污染引用状态
- 兼容性清单覆盖浏览器与站点维度，可直接复用
- AGENTS/deepwiki 与实现行为一致

---

## P3-T1

**Task:** Run cross-browser fallback hardening

**Files:**
- Modify: `webclipper/src/services/comments/selection/recovery-capabilities.ts`
- Modify: `webclipper/src/services/comments/selection/selection-recovery-resolver.ts`
- Add: `webclipper/tests/unit/comment-selection-recovery-browser-fallback.test.ts`

**Step 1: 实现功能**

- 补齐 API 能力矩阵判定：优先 `caretRangeFromPoint`，回退 `caretPositionFromPoint`，二者都不可用时快速降级。
- 统一边界保护：异常、空返回、无效 offset、根节点不匹配时全部返回空引用。
- 为不同能力组合补齐单测，避免浏览器差异导致回归。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-selection-recovery-browser-fallback.test.ts tests/unit/comment-selection-recovery-resolver.test.ts`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: 兼容路径与降级路径均通过，类型检查无回归。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/selection/recovery-capabilities.ts webclipper/src/services/comments/selection/selection-recovery-resolver.ts webclipper/tests/unit/comment-selection-recovery-browser-fallback.test.ts`

Run: `git commit -m "fix: P3-T1 - 加固跨浏览器选区兜底能力"`

---

## P3-T2

**Task:** Sync docs and compatibility checklist

**Files:**
- Modify: `webclipper/AGENTS.md`
- Modify: `.github/deepwiki/modules/comments.md`
- Modify: `.github/deepwiki/modules/webclipper.md`
- Add: `.github/features/webclipper-comments-selection-recovery/compatibility-checklist.md`

**Step 1: 实现功能**

- 文档同步：记录白名单 host、触发条件、降级语义、浏览器能力差异。
- 产出兼容性清单：浏览器维度、站点维度、交互方向（左->右/右->左/跨段落）、失败回退验证项。
- 明确后续扩展策略：先新增白名单与回归，再评估是否需要站点专属 adapter。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`

Expected: 构建通过，文档与实现事实一致。

**Step 3: 原子提交**

Run: `git add webclipper/AGENTS.md .github/deepwiki/modules/comments.md .github/deepwiki/modules/webclipper.md .github/features/webclipper-comments-selection-recovery/compatibility-checklist.md`

Run: `git commit -m "docs: P3-T2 - 同步选区恢复兼容策略与清单"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
