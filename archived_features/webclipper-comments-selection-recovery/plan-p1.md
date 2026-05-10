# Plan P1 - webclipper-comments-selection-recovery

**Goal:** 建立“白名单 + 坐标反推”基础能力，并形成可复用的选区恢复服务契约。

**Non-goals:**
- 不接入 inpage 运行时（留到 P2）
- 不改 UI 交互语义
- 不实现 page-world 站点适配

**Approach:**
先定义 comments 侧可消费的“选区恢复”契约，再实现 host gate 与坐标记录器，最后实现从坐标反推 `Range` 的解析器。解析器采用“原生选区优先、坐标反推兜底、失败静默降级”策略，确保不破坏既有链路。

**Acceptance:**
- 能基于 host 白名单控制反推是否触发
- 能从坐标稳定恢复 `selectionText`（含反向拖拽）
- 在文本可得但 locator 失败时返回 `selectionText + locator:null`

---

## P1-T1

**Task:** Define selection recovery contract

**Files:**
- Add: `webclipper/src/services/comments/selection/types.ts`
- Modify: `webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts`
- Modify: `webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`

**Step 1: 实现功能**

- 定义 `SelectionRecoveryRequest/Result` 契约（包含 `trigger`、根节点、host、最近指针轨迹等最小字段）。
- 保持 controller 对外行为不变，仅补齐可扩展契约，不改动 UI 层语义。
- 约束返回值：永远返回对象，不向 UI 抛异常。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: 类型检查通过，comments 相关调用点无断裂。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/selection/types.ts webclipper/src/services/comments/sidebar/comment-sidebar-contract.ts webclipper/src/services/comments/sidebar/article-comments-sidebar-controller.ts`

Run: `git commit -m "feat: P1-T1 - 定义评论选区恢复服务契约"`

---

## P1-T2

**Task:** Add host gate and capability recorder

**Files:**
- Add: `webclipper/src/services/comments/selection/recovery-host-gate.ts`
- Add: `webclipper/src/services/comments/selection/coordinate-recorder.ts`
- Add: `webclipper/src/services/comments/selection/recovery-capabilities.ts`
- Add: `webclipper/tests/unit/comment-selection-host-gate.test.ts`

**Step 1: 实现功能**

- 新增 host 白名单匹配（后缀匹配，首批 `www.dedao.cn`、`dedao.cn`）。
- 新增 capture-phase 坐标记录器：记录最近一次拖拽 `pointerdown/up` 端点与时间窗口。
- 新增能力探测：统一判断 `caretRangeFromPoint` / `caretPositionFromPoint` 可用性，避免后续解析器散落 `any` 分支。
- 增加防噪约束：点击无拖拽、过期轨迹、越界坐标直接判定无效。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-selection-host-gate.test.ts`

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`

Expected: host gate、时间窗口、坐标有效性断言全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/selection/recovery-host-gate.ts webclipper/src/services/comments/selection/coordinate-recorder.ts webclipper/src/services/comments/selection/recovery-capabilities.ts webclipper/tests/unit/comment-selection-host-gate.test.ts`

Run: `git commit -m "feat: P1-T2 - 添加白名单门控与坐标记录器"`

---

## P1-T3

**Task:** Implement range reconstruction resolver

**Files:**
- Add: `webclipper/src/services/comments/selection/selection-recovery-resolver.ts`
- Modify: `webclipper/src/services/comments/locator/index.ts`
- Add: `webclipper/tests/unit/comment-selection-recovery-resolver.test.ts`

**Step 1: 实现功能**

- 解析顺序：`getSelection` -> `caretRangeFromPoint` -> `caretPositionFromPoint`。
- 处理反向拖拽：必要时交换 start/end，确保 `Range` 有序。
- 输出 `selectionText` 后尝试构建 locator；locator 失败不影响文本输出。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-selection-recovery-resolver.test.ts`

Expected: 原生优先、反推兜底、反向拖拽、locator 降级路径全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/selection/selection-recovery-resolver.ts webclipper/src/services/comments/locator/index.ts webclipper/tests/unit/comment-selection-recovery-resolver.test.ts`

Run: `git commit -m "feat: P1-T3 - 实现坐标反推选区解析器"`

---

## P1-T4

**Task:** Add P1 unit tests

**Files:**
- Add: `webclipper/tests/unit/comment-selection-coordinate-recorder.test.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`

**Step 1: 实现功能**

- 为坐标记录器补齐边界测试（时间窗口、最小位移、空坐标）。
- 为 controller 增加“文本存在、locator 缺失”与“异常降级为空”断言。

**Step 2: 验证**

Run: `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-selection-coordinate-recorder.test.ts tests/unit/article-comments-sidebar-controller.test.ts`

Expected: P1 新增与受影响单测全部通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/unit/comment-selection-coordinate-recorder.test.ts webclipper/tests/unit/article-comments-sidebar-controller.test.ts`

Run: `git commit -m "test: P1-T4 - 补齐选区恢复基础单测"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
