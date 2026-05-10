# Subagent Audit P2

Source: code-review subagent (`p2-audit`)

## Checklist verdicts

- 自动高亮映射最近中线 user 消息：**Pass**
- app/popup + 宽窄屏 + `.route-scroll` root：**Pass**
- 大对话滚动性能：**Fail**（发现 scroll 重算时仍会全量扫描 userMessageEls）
- 点击跳转与高亮一致性：**Pass**

## Findings

1. **Medium** — `useChatOutlineActiveIndex.ts` 在 `recompute` 内通过 `safeUserMessageEls.filter(...)` 每次 scroll 都全量遍历消息，超长会话下会放大滚动开销。
   - 建议：改为迭代 `visibleSet` 并配合元素索引映射，仅在必要时退化到全量候选。
