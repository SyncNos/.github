# Plan P3 - webclipper-ai-chat-autosave-backfill

**Goal:** 同步开发文档，让后续维护者能快速理解 autosave backfill 的语义、限制与排障路径。

**Non-goals:**
- 不写用户手册（面向终端用户的 guide 不在本 feature 范围）。
- 不扩展新站点/新 collector。

**Approach:**
- 在 deepwiki 中补齐 autosave/backfill 的“真实语义”与窗口限制（200）、安全跳过策略（A）、以及推荐排障步骤（例如无 overlap 时手动保存）。
- 文档只写原则与导航，不在多处写死易漂移细节。

**Acceptance:**
- deepwiki 中能找到 backfill 的入口描述与排障建议。

---

<a id="p3-t1"></a>
## P3-T1 更新deepwiki开发文档：autosave backfill语义、窗口限制（200）、安全跳过与排障建议

**Files:**
- Modify: `.github/deepwiki/configuration.md`
- (Optional) Modify: `.github/deepwiki/business-context.md`

**Step 1: 实现功能**
- 在 `.github/deepwiki/configuration.md` 中补充一段（靠近 `ai_chat_auto_save_enabled` 描述处）：
  - autosave 现在会在打开对话时尝试 backfill 最近窗口（200）内缺失历史
  - 找不到 overlap 时会安全跳过并提示手动保存（开发者排障建议）
  - backfill 仅 console 日志，无 UI 提示
- 如需补充业务语义，则在 `business-context.md` 的“旅程 1”中增加一句：autosave 可能在打开对话时补齐缺口，使本地事实源更接近跨设备真实历史。

**Step 2: 验证**
- Run: `npm --prefix webclipper run build`
- Expected: 构建不受文档改动影响（主要用于确保 workspace 无残留 TS 错误）。

**Step 3: 原子提交**
- Run: `git add .github/deepwiki/configuration.md .github/deepwiki/business-context.md`
- Run: `git commit -m "docs: P3-T1 - 记录autosave backfill语义与排障建议"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
