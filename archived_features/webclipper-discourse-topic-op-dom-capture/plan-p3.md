# Plan P3 - webclipper-discourse-topic-op-dom-capture

**Goal:** 补齐回归测试，覆盖 URL 归一化、会话稳定性、`/1` 跳转抓 OP 与严格失败路径，保证隐藏测试可通过。

**Non-goals:**
- 本 phase 不新增产品行为
- 本 phase 不扩展到非 Discourse 论坛规则

**Approach:**
- 单测优先覆盖纯函数与上下文键逻辑。
- smoke/integration 覆盖抓取路径关键分支（成功与失败）。
- 以“同 topic 不重复会话 + OP-only”为主验收轴。

**Acceptance:**
- URL 归一化、侧栏上下文稳定、抓取分支行为均有自动化测试保护
- `compile + test` 通过

---

## P3-T1 补齐URL归一化与会话稳定性测试

**Files:**
- Modify: `webclipper/tests/unit/http-url.test.ts`
- Modify: `webclipper/tests/unit/article-comments-sidebar-controller.test.ts`
- Add: `webclipper/tests/unit/discourse-canonical-url.test.ts`

**Step 1: 实现功能**
- 为 Discourse canonical 规则补充单测：
  - `/t/s/1/1`、`/t/s/1/20`、`/t/s/1` 的 canonical 一致
  - 非 Discourse URL 规则不变
- 为评论侧栏 controller 补充“同 topic 不同楼层 URL 不触发上下文重置”的测试断言。

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- tests/unit/http-url.test.ts tests/unit/article-comments-sidebar-controller.test.ts tests/unit/discourse-canonical-url.test.ts`

Expected: 单测通过

**Step 3: 原子提交**
Run: `git add webclipper/tests/unit/http-url.test.ts webclipper/tests/unit/article-comments-sidebar-controller.test.ts webclipper/tests/unit/discourse-canonical-url.test.ts`

Run: `git commit -m "test: task7 - 补齐Discourse canonical与侧栏稳定性单测"`

---

## P3-T2 补齐OP跳转抓取与失败路径测试

**Files:**
- Modify: `webclipper/tests/smoke/background-router-article-fetch.test.ts`
- Modify: `webclipper/tests/smoke/article-fetch-service.test.ts`
- Add: `webclipper/tests/smoke/article-fetch-discourse-op.test.ts`

**Step 1: 实现功能**
- 增加用例覆盖：
  - 在 `/20` 触发抓取，走 `/1` fallback 并输出 OP 内容
  - `/1` 超时找不到 OP 时返回失败，不降级抓回复
  - 返回/保存 URL 为 topic canonical（不含 `/N`）

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts tests/smoke/article-fetch-service.test.ts tests/smoke/article-fetch-discourse-op.test.ts tests/smoke/article-fetch-wechat-gallery.test.ts`

Expected: Discourse 关键 smoke 与非 Discourse 回归 smoke 均通过

**Step 3: 原子提交**
Run: `git add webclipper/tests/smoke/background-router-article-fetch.test.ts webclipper/tests/smoke/article-fetch-service.test.ts webclipper/tests/smoke/article-fetch-discourse-op.test.ts`

Run: `git commit -m "test: task8 - 覆盖Discourse OP跳转抓取成功与失败路径"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
