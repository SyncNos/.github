# Plan P2 - webclipper-ai-chat-autosave-backfill

**Goal:** 在 P1 backfill 可用的基础上，提升稳定性、降低重复风险，并按“及时删除老旧代码”原则完成必要重构清理。

**Non-goals:**
- 不新增用户可见 UI。
- 不改变 autosave 的 append-only 约束（仍不删除/不替换）。

**Approach:**
- 抽取 autosave identity/overlap 相关工具，保证 incremental 与 backfill 使用同一套判定逻辑，并在同一 task 删除旧实现。
- 将 `fallback_` 这类基于 `sequence` 的 key 视为不稳定输入，避免跨窗口漂移导致重复写入。
- 明确 backfill 与 incremental 的协作顺序：backfill 成功后应初始化/更新 baseline，防止同一 tick 内二次写入或漏写。

**Acceptance:**
- backfill + incremental 在同一会话中协作稳定，不出现重复追加或明显漏写。
- 引擎内部重复实现被删除，代码收敛且可测试。

---

<a id="p2-t1"></a>
## P2-T1 抽取autosave identity/overlap通用工具并及时删除旧实现（incremental + backfill 共用）

**Files:**
- Add: `webclipper/src/services/conversations/content/autosave-identity-utils.ts`
- Modify: `webclipper/src/services/conversations/content/autosave-incremental-engine.ts`
- Modify: `webclipper/src/services/conversations/content/autosave-backfill-reconciler.ts`

**Step 1: 实现功能**
- 将以下逻辑从 `autosave-incremental-engine.ts` 抽到新文件 `autosave-identity-utils.ts`：
  - `normalizeContent` / `getMessageIdentityBase` / `fingerprintHash` / overlap 计算（suffix/prefix）
- incremental engine 与 backfill reconciler 统一从该文件 import。
- **同一个 task 内删除旧实现**（避免双份逻辑长期并存）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/smoke/schema.test.ts`
- Run: `npm --prefix webclipper run test -- webclipper/tests/unit/autosave-backfill-reconciler.test.ts`
- Expected: incremental 与 backfill 相关用例仍通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/conversations/content/autosave-identity-utils.ts webclipper/src/services/conversations/content/autosave-incremental-engine.ts webclipper/src/services/conversations/content/autosave-backfill-reconciler.ts`
- Run: `git commit -m "refactor: P2-T1 - 抽取autosave identity/overlap工具并删除旧实现"`

---

<a id="p2-t2"></a>
## P2-T2 autosave增量引擎：将fallback_ key视为不稳定，避免跨窗口/跨会话key漂移导致重复

**Files:**
- Modify: `webclipper/src/services/conversations/content/autosave-incremental-engine.ts`
- Modify: `webclipper/tests/smoke/schema.test.ts`

**Step 1: 实现功能**
- 在 incremental engine 的 incomingKey 处理逻辑中：
  - 对 `messageKey` 以 `fallback_` 开头的输入，直接视为不稳定（不进入 `stableIncomingKey` 流程），从而优先使用 `autosave_...` synthetic key。
  - 说明：当前 `makeFallbackMessageKey` 包含 `sequence`（不同 capture/window 下 sequence 可能漂移），导致同一条消息在不同窗口里产生不同 key，进而造成重复追加风险。
- 更新/补充 smoke 测试：
  - 构造含 `fallback_` key 的输入，确保引擎仍输出 `autosave_` key 且不会因 sequence 漂移产生重复 added。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/smoke/schema.test.ts`
- Expected: 新增断言通过且不引入回归。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/conversations/content/autosave-incremental-engine.ts webclipper/tests/smoke/schema.test.ts`
- Run: `git commit -m "fix: P2-T2 - autosave增量忽略fallback_ key以降低重复风险"`

---

<a id="p2-t3"></a>
## P2-T3 完善backfill与incremental协作：backfill后初始化/更新baseline，避免重复写入或漏写

**Files:**
- Modify: `webclipper/src/services/bootstrap/content-controller.ts`
- Modify: `webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`

**Step 1: 实现功能**
- 明确 tick 内顺序：
  1) capture snapshot
  2) backfill（可能写入缺口）
  3) incremental compute/save
- backfill 成功后，确保 incremental engine baseline 被刷新，但本 tick 不产生二次写入：
  - 允许通过调用 `computeIncremental(snapshot)` 来刷新内部状态，并**显式跳过** incremental 的 `saveSnapshot`（以防 backfill 与 incremental 同 tick 双写）
- 补齐 smoke 测试覆盖“同 tick backfill 后不重复写入”的断言。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Expected: 新增断言通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/bootstrap/content-controller.ts webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Run: `git commit -m "fix: P2-T3 - backfill与incremental协作顺序收敛并避免重复写入"`

---

<a id="p2-t4"></a>
## P2-T4 性能与边界：节流/上限/仅AI chat生效；补齐单测覆盖边界条件

**Files:**
- Modify: `webclipper/src/services/bootstrap/content-controller.ts`
- Modify: `webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Modify: `webclipper/tests/unit/autosave-backfill-reconciler.test.ts`

**Step 1: 实现功能**
- 强化 backfill 的 guardrails：
  - 仅当 collector 属于 `AI_CHAT_AUTO_SAVE_COLLECTOR_IDS` 且 auto-save 开关开启时触发；
  - 节流（例如 >=10s）+ 最大尝试次数（例如 <=6）+ 最大持续时间（例如 <=2min）；
  - 对同一 conversation 的 console warn/info 去重（避免刷屏）。
- 补齐单测覆盖：
  - limit 与窗口边界（0/1/200）
  - 重试上限与节流条件

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/unit/autosave-backfill-reconciler.test.ts`
- Run: `npm --prefix webclipper run test -- webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Expected: 边界用例通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/bootstrap/content-controller.ts webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts webclipper/tests/unit/autosave-backfill-reconciler.test.ts`
- Run: `git commit -m "perf: P2-T4 - backfill加入节流与上限并完善边界测试"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
