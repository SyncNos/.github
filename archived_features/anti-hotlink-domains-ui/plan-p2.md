# Plan P2 - Settings 高级编辑器与交互闭环

**Goal:** 在 `Settings → General` 增加可展开的防盗链域名编辑器，并完成 controller、保存、校验与 UI 交互闭环。

**Non-goals:** 新增 Settings section、支持 wildcard/path 规则、扩展到 context menu。

**Approach:** 复用 Notion OAuth 的高级选项交互（`advancedShow/advancedHide`），复用 ChatWith 列表编辑范式（行编辑 + add/remove/reset），但把持久化策略改成“草稿态 + 显式校验后写入”，避免非法输入被静默吞掉。Settings 只走 services 封装，不直接碰 platform 底层细节。

**Acceptance:**
- General 区块出现高级按钮并可展开/收起防盗链编辑面板。
- 列表支持新增、编辑、删除、重置默认值，并能跨刷新保留。
- 非法 domain/referer 会展示明确错误，且不会写入 storage。
- storage 变更（含其他入口写入）可被 Settings 控制器正确同步。

---

## P2-T1

**Title:** 建立 settings 集成服务

**Files:**
- Add: `webclipper/src/services/integrations/anti-hotlink/anti-hotlink-settings.ts`
- Modify: `webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

**Step 1: 实现功能**

新增供 Settings 使用的服务封装：加载规则、保存规则、重置默认值、返回校验错误。该服务只能依赖 platform 规则 helper，不复制底层归一化逻辑，确保“运行时与设置页”使用同一规则语义。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: settings service 与 platform helper 依赖方向正确，编译通过。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/integrations/anti-hotlink/anti-hotlink-settings.ts webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

Run: `git commit -m "feat: P2-T1 - 建立防盗链settings集成服务"`

---

## P2-T2

**Title:** 接入设置控制器状态

**Files:**
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`

**Step 1: 实现功能**

在 controller 中新增防盗链编辑器的草稿状态、高级面板开关、row 级错误状态与保存动作（add/edit/remove/reset/apply）。把新 storage key 纳入 `refreshInternal` 与 `storageOnChanged` 监听链路，避免跨页面写入后 UI 状态不一致。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: controller 状态与 handler 全部类型正确，且不会破坏既有 General 选项逻辑。

**Step 3: 原子提交**

Run: `git add webclipper/src/viewmodels/settings/useSettingsSceneController.ts`

Run: `git commit -m "feat: P2-T2 - 接入防盗链设置控制器状态"`

---

## P2-T3

**Title:** 实现高级编辑面板

**Files:**
- Add: `webclipper/src/ui/settings/sections/AntiHotlinkDomainsEditor.tsx`
- Modify: `webclipper/src/ui/settings/sections/InpageSection.tsx`
- Modify: `webclipper/src/ui/settings/SettingsScene.tsx`

**Step 1: 实现功能**

在 `webArticleCacheImagesEnabled` 下挂载高级按钮和编辑器组件，沿用 `aria-expanded/aria-controls` 与现有按钮风格。编辑器提供 `domain + referer` 双列输入、add/remove/reset、错误提示渲染，并与 controller 草稿态联动。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: General 页面编译通过，面板 props 与状态链路完整。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/settings/sections/AntiHotlinkDomainsEditor.tsx webclipper/src/ui/settings/sections/InpageSection.tsx webclipper/src/ui/settings/SettingsScene.tsx`

Run: `git commit -m "feat: P2-T3 - 实现防盗链高级编辑面板"`

---

## P2-T4

**Title:** 补 UI 与端到端回归

**Files:**
- Modify: `webclipper/tests/unit/settings-sections.test.ts`
- Modify: `webclipper/tests/smoke/article-fetch-service.test.ts`
- Modify: `webclipper/tests/smoke/image-download-proxy.test.ts`
- Add or modify: `webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

**Step 1: 实现功能**

补齐设置页交互与运行时回归用例：高级面板显隐、非法输入阻止保存、重置默认值恢复、runtime 查表命中与采集链路连通。确保计划中的关键边界（缺失 key、空列表、读失败回退）都有测试覆盖。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: compile/test/build 全部通过，设置与采集链路行为一致。

**Step 3: 原子提交**

Run: `git add webclipper/tests/unit/settings-sections.test.ts webclipper/tests/smoke/article-fetch-service.test.ts webclipper/tests/smoke/image-download-proxy.test.ts webclipper/tests/unit/anti-hotlink-rules-store.test.ts`

Run: `git commit -m "test: P2-T4 - 补防盗链UI与端到端回归"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
