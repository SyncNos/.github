## 只读审计报告 - webclipper-sidebar-site-tags P1

**审计日期:** 2026-03-31  
**审计范围:** `.github/features/webclipper-sidebar-site-tags` P1（T1/T2/T3）  
**审计结论:** **PASS** ✅

---

### P1-T1：破坏性重构（旧逻辑删除 + 新 helper 统一）

- `ConversationListPane.tsx` 中旧 `SourceMeta` / `getSourceMeta` / `sourceTagToneClass` 已删除。
- 列表行已改用 `resolveConversationListTag({ conversation, translate: t })`。
- source filter facets 已改用 `resolveConversationSourceOptionLabel(...)`。
- `web` 标签优先级链完整：`listSiteKey(domain:*) -> parseHostnameFromUrl(url) -> t('insightUnknownLabel')`。

结论：通过。

### P1-T2：品牌色 token 与渲染切换

- `tokens.css` 新增 `--brand-*` 变量（11 个来源）。
- `conversation-list-tags.ts` 非 web tone 已统一改为 `--brand-*` + `color-mix`。
- web tone 保持固定样式，不包含 `--brand-*`。
- source tag 已不再使用 `--info/--success/--warning` 等语义色。

结论：通过。

### P1-T3：单测覆盖

- 新增 `tests/unit/conversation-list-tags.test.ts`。
- 覆盖：
  - `web` 的 `domain:*` 优先显示。
  - `unknown/缺失` 回退 URL hostname。
  - 最终回退 `insightUnknownLabel`。
  - 非 web 至少 3 个来源验证 `--brand-*`。
  - `web` tone 不含 `--brand-*`。

结论：通过。

---

## Findings

无阻塞问题（high/medium/low 均无）。

## 非阻塞建议

1. 可后续补一条参数化测试覆盖全部 11 个来源 key 的 `--brand-*` 映射。
2. 可在 UI 视觉回归中补充暗色主题对比度人工检查。
