# Plan P1 - 防盗链配置源与运行时接入

**Goal:** 把防盗链 referer 规则迁移到可持久化配置，并在不破坏分层的前提下接入图片下载与文章采集链路。

**Non-goals:** Settings UI、文案扩展、新增入口（context menu / command）。

**Approach:** 以 `platform/*` 作为运行时规则真源，承载规则类型、默认 seed、归一化、去重、缓存与 lookup。`services/*` 只做上层封装，不允许反向让 `platform/*` import `services/*`。运行时读取失败时回退到安全策略，确保采集流程不中断。

**Acceptance:**
- 缺失 storage key 时仅初始化一次默认 seed；显式空列表不自动补回默认值。
- 运行时 lookup 只依赖 platform 规则 helper，不再直接查硬编码 map。
- storage 读取失败不会让采集/下载链路崩溃。
- Chrome/Firefox 现有行为保持兼容（DNR 可用时注入 referer，不可用时降级 plain fetch）。

---

## P1-T1

**Title:** 建立平台侧规则存储

**Files:**
- Add: `webclipper/src/platform/webext/anti-hotlink-rules-store.ts`
- Modify: `webclipper/src/services/url-cleaning/hostname.ts`
- Add: `webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

**Step 1: 实现功能**

定义 `AntiHotlinkRule` 数据模型、storage key、默认 seed 与归一化逻辑（hostname 标准化、referer http(s) 校验、按 domain 去重）。把“首次缺失写默认值、已有值按用户快照使用”的读取语义落在 platform helper 中。优先复用现有 URL/hostname 规范化工具，避免重复实现。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 新增规则 store 与类型边界可通过 TypeScript 检查。

**Step 3: 原子提交**

Run: `git add webclipper/src/platform/webext/anti-hotlink-rules-store.ts webclipper/src/services/url-cleaning/hostname.ts webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

Run: `git commit -m "feat: P1-T1 - 建立平台侧防盗链规则存储"`

---

## P1-T2

**Title:** 接入运行时查表与回退

**Files:**
- Modify: `webclipper/src/platform/webext/image-download-proxy.ts`
- Modify: `webclipper/src/collectors/web/article-fetch.ts`

**Step 1: 实现功能**

把下载链路和 article 防盗链命中判断统一切换到 `anti-hotlink-rules-store`。补模块级缓存/刷新机制，避免每张图重复读取 storage。明确异常回退：读取配置失败时继续执行采集流程，并使用可预测的 fallback（不中断主流程）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 运行时链路可编译且现有测试不过度回归。

**Step 3: 原子提交**

Run: `git add webclipper/src/platform/webext/image-download-proxy.ts webclipper/src/collectors/web/article-fetch.ts`

Run: `git commit -m "feat: P1-T2 - 接入运行时防盗链查表与回退"`

---

## P1-T3

**Title:** 补运行时防回归测试

**Files:**
- Modify: `webclipper/tests/smoke/image-download-proxy.test.ts`
- Modify: `webclipper/tests/smoke/article-fetch-service.test.ts`
- Modify: `webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

**Step 1: 实现功能**

补三类关键回归：默认 seed 初始化、显式空列表保留、运行时 lookup 命中与失败回退。确保 article 的“配置关闭但命中防盗链域名时仍自动缓存”行为与当前语义一致。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 规则存储与下载/采集链路用例稳定通过。

**Step 3: 原子提交**

Run: `git add webclipper/tests/smoke/image-download-proxy.test.ts webclipper/tests/smoke/article-fetch-service.test.ts webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

Run: `git commit -m "test: P1-T3 - 补运行时防盗链回归"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
