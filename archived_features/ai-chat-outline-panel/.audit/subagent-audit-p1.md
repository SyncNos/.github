# Subagent Audit P1

Source: code-review subagent (`p1-audit`)

## Checklist verdicts

- UI mock constraints: **Fail** (anchor wrapper had conflicting `absolute` + `relative` classes)
- 仅 `sourceType === chat` 显示: **Pass**
- 点击条目平滑跳转正确消息: **Pass**

## Findings

1. **Medium** — `ConversationDetailPane.tsx` 中目录锚点 wrapper 同时包含 `tw-absolute` 与 `tw-relative`，导致定位约束不稳定，可能偏离示意稿要求。
   - 建议：移除 wrapper 上的 `tw-relative`，保留纯 absolute 锚定。
