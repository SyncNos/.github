# Plan P2 - webclipper-notion-parent-page-refresh

**Goal:** 补齐 parent pages discovery 的测试与边界处理，并确保没有残留旧路径；跑通完整验证链路。

**Non-goals:**
- 不新增“手动输入 Parent Page ID”设置项
- 不做 i18n 翻译表清理（仅清理代码残留）

**Approach:** 为 services/background 的 parent pages discovery 增加可控的测试入口（注入 fetch / 固定响应），覆盖“第一页无可用页但后续页有”的关键场景；补齐错误口径（401/403/429/网络失败）在 UI 的可见反馈；最后做全仓扫描，确保旧实现与直连请求消失，并跑 `compile/test/build` 验证。

**Acceptance:**
- 新增测试覆盖分页与过滤关键场景
- 失败时 UI 不再静默停留在空占位（至少能在 Settings error 区域看到错误）
- `rg` 扫描确认无旧实现残留
- `compile/test/build` 全部通过

---

<a id="p2-t1"></a>
## P2-T1 为 parent pages discovery 增加单测/冒烟测试覆盖（分页与过滤）

**Files:**
- Add: `webclipper/tests/unit/notion-parent-pages.test.ts`（或按仓库习惯选择 `tests/smoke/`）
- Modify: `webclipper/src/services/sync/notion/notion-parent-pages.ts`（如需注入 fetch 以便测试）

**Step 1: 实现功能**

- 覆盖至少这些场景：
  1. 第 1 页结果全部被过滤（`parent.database_id`），第 2 页出现可用 page → 最终返回非空列表
  2. `savedPageId` 不在 search 结果中 → 通过 `GET /pages/:id` resolve 并注入列表头部
  3. 去重：同一 page id 跨页重复不应出现多次

**Step 2: 验证**

Run: `npm --prefix webclipper run test`

Expected: 新增测试通过。

**Step 3: 原子提交**

- `test: task6 - 覆盖 Notion parent pages discovery 的分页与 savedId resolve`

---

<a id="p2-t2"></a>
## P2-T2 补齐失败口径与边界行为（空结果/权限/429），并做残留扫描清理

**Files:**
- Modify: `webclipper/src/services/sync/notion/settings-background-handlers.ts`
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- (Optional) Modify: `webclipper/src/services/sync/notion/notion-api.ts`

**Step 1: 实现功能**

- 明确区分两类“空”：
  - 真空：search 返回无可用页（用户授权页太少 / 全部在 database 下）
  - 假空：第一页被过滤但后续页存在（已在 P1 修复分页）
- 对 429（rate limit）/ 网络失败提供可读错误（不必新增 i18n，但必须能在 Settings error 区域看到）
- 残留扫描（本 task 内完成清理）：
  - Run: `rg -n "clickRefresh" webclipper/src`
  - Run: `rg -n "https://api\\.notion\\.com/v1/(search|pages)" webclipper/src/viewmodels webclipper/src/ui`

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过。

**Step 3: 原子提交**

- `refactor: task7 - 统一 Notion parent pages 失败口径并清理残留调用`

---

<a id="p2-t3"></a>
## P2-T3 跑通验证链路（compile/test/build），记录与修复阻塞

**Files:**
- Modify: （按实际改动）

**Step 1: 验证**

Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test`

Run: `npm --prefix webclipper run build`

Expected: 三条命令全部通过。

**Step 2: 原子提交**

- `chore: task8 - 跑通 webclipper 编译/测试/构建验证链路`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
