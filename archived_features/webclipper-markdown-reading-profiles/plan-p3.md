# Plan P3 - webclipper-markdown-reading-profiles

**Goal:** 把阅读风格协议接入设置与运行时链路，实现可切换、可持久化、可回退，并补齐最终 UX 验收闭环。

**Non-goals:** 本 phase 不新增 profile，不改 markdown 语法能力，不引入额外主题系统。

**Approach:**
- 先完成存储契约（键名、normalize、默认值、脏值回退）。
- 再把 profile 选择从 settings 贯通到详情渲染链，并提供即时预览反馈。
- 最后执行 compile/test/build 与体验向审计，确保“可用 + 可读 + 可维护”。

**Acceptance:**
- 设置页可选择 `medium/notion/book` 并持久化。
- popup/app 详情渲染按所选 profile 生效，重开后一致。
- 未知值自动回退到 `medium`，不会导致渲染异常。
- 文档同步包含协议真源、默认值、扩展流程与审计规则。

---

<a id="p3-t1"></a>
## P3-T1 接入 profile 存储键与兼容回读（脏值兜底）

**Files:**
- Add: `webclipper/src/services/protocols/markdown-reading-profile-storage.ts`
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- Modify: `webclipper/tests/unit/settings-sections.test.ts`（如需补 key/分组约束）
- Add: `webclipper/tests/unit/markdown-reading-profile-storage.test.ts`

**Step 1: 实现**
1. 新增存储键（建议：`markdown_reading_profile_v1`）与 normalize 方法。
2. 在 settings controller 统一读写 profile；缺失/异常/未知值回退 `medium`。
3. 对外暴露 `markdownReadingProfile` 与 setter，供 SettingsScene + Detail 渲染消费。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/unit/markdown-reading-profile-storage.test.ts tests/unit/settings-sections.test.ts`
- Expected: 存储读写、脏值回退、默认值行为通过。

**Step 3: 原子提交**
- `feat: task7 - 接入markdown阅读风格存储与兼容回读`

---

<a id="p3-t2"></a>
## P3-T2 设置页接入 profile 选择并贯通详情渲染链

**Files:**
- Modify: `webclipper/src/ui/settings/sections/InpageSection.tsx`
- Modify: `webclipper/src/ui/settings/SettingsScene.tsx`
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/shared/ChatMessageBubble.tsx`
- Modify: `webclipper/src/ui/i18n/locales/en.ts`
- Modify: `webclipper/src/ui/i18n/locales/zh.ts`

**Step 1: 实现**
1. 在 `general` 设置分区新增 profile 选择控件（`medium/notion/book`）。
2. 设置项包含简短描述，明确每个 profile 的阅读倾向（平衡/紧凑/沉浸）。
3. 将 profile 值贯通至详情渲染链，驱动 `ChatMessageBubble` 使用对应 preset。
4. 保持窄屏/宽屏一致，不破坏既有 detail header actions 与 comments sidebar 逻辑。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test -- tests/smoke/app-detail-header-actions.test.ts tests/unit/chat-message-bubble.test.ts tests/unit/markdown-reading-profiles.test.ts`
- Expected: 设置与渲染链联动通过，既有详情交互 smoke 不回归。

**Step 3: 原子提交**
- `feat: task8 - 接入阅读风格设置到详情渲染链`

---

<a id="p3-t3"></a>
## P3-T3 完成最终 UX 验收、文档同步与审计收口

**Files:**
- Modify: `webclipper/AGENTS.md`
- Modify: `webclipper/src/ui/AGENTS.md`
- Modify: `.github/features/webclipper-markdown-reading-profiles/audit-p3.md`（执行期回填）

**Step 1: 实现**
1. 补充“阅读风格协议”文档：协议真源、默认值、回退、扩展顺序（先协议/测试，再 preset/UI）。
2. 执行最终验证链并记录已知阻塞是否与本 feature 相关。
3. 在 audit 中新增 UX 审查项：
   - `320px` 重排
   - 长链接断行
   - `pre/table` 局部横滚
   - light/dark 可读性抽查

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`
- Expected: compile/build 通过；test 若有既有阻塞，需明确与本 feature 的相关性。

**Step 3: 原子提交**
- `chore: task9 - 完成阅读风格协议化收尾与ux验收`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
