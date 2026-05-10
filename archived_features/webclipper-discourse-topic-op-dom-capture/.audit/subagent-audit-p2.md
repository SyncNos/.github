## 📋 **P2 阶段只读审计报告** — `webclipper-discourse-topic-op-dom-capture`

### **1) 对照 Plan-P2 三个 Task 核查**

| Task | 状态 | 核查结果 |
|------|------|--------|
| **P2-T1: OP优先DOM提取器** | ✅ 完成 | 实现完整；三种OP节点选择器（`article[data-post-number='1']`、`.topic-post[data-post-number='1']`、`#post_1`）；Discourse优先于Readability；非Discourse继续走现有流程 |
| **P2-T2: OP缺失→/1跳转停留** | ✅ 完成 | Fallback逻辑、防循环、URL到位8s等待、DOM稳定重新等待均已实现；smoke测试有完整覆盖 |
| **P2-T3: /1仍缺OP严格失败** | ⚠️ **部分完成** | 代码实现正确（`throw toError('Discourse OP not found')`），但**缺少单元测试覆盖** |

---

### **2) 重点检查结论**

#### ✅ **Discourse OP 识别 —— "只抓 OP" 是否真的**

**证据链**：

- [article-fetch.ts](webclipper/src/collectors/web/article-fetch.ts#L606-L655)：`extractDiscourseOpOnly()` 
  - L606-H: 三层选择器查找 OP 节点
  - L615-622: **仅从 OP 节点提取** `content`、`text`、`author`、`publishedAt`
  - L607-609: 若 OP 找到，直接返回 `warningFlags: []`（表示成功）
  
- L695-697: 若提取成功则直接返回，**不经过 Readability**

**✅ 结论**：逻辑严密。当 Discourse 页面有 OP 时，确实"只抓 OP"，零其他内容混入。

---

#### ✅ **/20 → /1 Fallback 是否有防循环**

**核心防循环条件** [L782-792](webclipper/src/collectors/web/article-fetch.ts#L782-L792)：

```typescript
if (
  discourseTopic &&
  discourseTopic.postNumber != null &&
  discourseTopic.postNumber !== 1 &&              // ← 关键：只在非/1时导航
  hasWarningFlag(..., DISCOURSE_OP_MISSING_WARNING_FLAG)
) {
  // 导航到 /1
  await tabsUpdate(targetTabId, { url: firstFloorUrl });
  await waitForTabUrl(targetTabId, firstFloorUrl, 8_000);
  extracted = await extractArticleOnTab(targetTabId);  // ← 二次提取
}
```

**防循环机制**：
- 第1次提取在 `/20`：若无 OP，标记 flag
- 导航到 `/1`，二次提取时 `postNumber` 已是 `1`
- 条件 `postNumber !== 1` 为 **假**，不会再导航

**✅ 结论**：防循环**有效**，零死循环风险。

---

#### ✅ **URL 到位 + DOM 稳定等待是否充分**

**时序序列**：

1. **URL 到位等待** [L70-82](webclipper/src/collectors/web/article-fetch.ts#L70-L82) `waitForTabUrl()`
   - 轮询 8s，检查 tab.url 与 expectedUrl 归一化后相等
   - URL 比较使用 `normalizeHttpUrl()`（移除 hash，保留 query）
   - 超时抛 `"timed out waiting for Discourse /1 navigation"`

2. **DOM 稳定等待** [L682](webclipper/src/collectors/web/article-fetch.ts#L682)
   - 在 `extractArticleOnTab()` 内被调用
   - 每次提取都会执行（**包括 fallback 二次提取**）
   - L512-530：检查 readyState === 'complete' 且 textLen ≥ minTextLength，连续2个稳定周期认为完成

**时序**：导航 → URL 到位 → DOM 稳定 → 内容提取

**✅ 结论**：两阶段等待充分。虽然分开而非同时进行，但顺序正确，无重叠问题。

---

#### ✅ **/1 仍缺 OP 时是否严格失败**

**严格失败逻辑** [L794-796](webclipper/src/collectors/web/article-fetch.ts#L794-L796)：

```typescript
if (discourseTopic && hasWarningFlag((extracted as any)?.warningFlags, DISCOURSE_OP_MISSING_WARNING_FLAG)) {
  throw toError('Discourse OP not found');
}
```

**工作流**：
- 若 Discourse topic 页面且第2次提取仍无 OP（flag 仍存在）
- 中止抓取，抛 `Discourse OP not found` 错误
- **禁止降级抓 Readability**（条件判断在 Readability 之前）

**L699**: `discourseOpMissingOnCurrentPage = Boolean(discourseTopic)` 确保即使非 Discourse页面也能继续用 Readability（不会被冤枉标记）

**✅ 结论**：严格失败逻辑正确。

---

#### ✅ **错误通过 background + current-page-capture 可见透传**

**错误规范化路径**：

1. [article-fetch-background-handlers.ts](webclipper/src/collectors/web/article-fetch-background-handlers.ts#L9-L12) L9-12：
```typescript
function normalizeArticleFetchError(error: unknown, fallback: string): string {
  const raw = (error as any)?.message != null ? String((error as any).message) : String(error);
  const message = raw.trim();
  if (/^discourse op not found$/i.test(message)) return 'Discourse OP not found';
  return message || fallback;
}
```

2. [current-page-capture.ts](webclipper/src/services/bootstrap/current-page-capture.ts#L55-L60) L55-60：
```typescript
function normalizeArticleCaptureErrorMessage(raw: unknown): string {
  const message = String(raw || '').trim();
  if (/^discourse op not found$/i.test(message)) return 'Discourse OP not found';
  return message;
}
```

3. 在 router 中调用 `router.err(normalizeArticleFetchError(e, ...))`，错误可被 UI 捕获

**✅ 结论**：错误透传可见，用户可理解。

---

### **3) 潜在问题发现**

| 级别 | 问题 | 证据 | 建议 |
|------|------|------|------|
| **🔴 HIGH** | **P2-T3 测试覆盖缺失** | [article-fetch-service.test.ts](webclipper/tests/smoke/article-fetch-service.test.ts#L152-L248)：5个用例中仅覆盖 T2（fallback成功），无"Discourse页面→/1仍缺OP→抛错"的用例 | 补充测试：`it('fails strictly when Discourse OP not found even on /1', ...)` |
| **🟡 MEDIUM** | **URL 比较边界——trailing slash/query参数** | [waitForTabUrl](webclipper/src/collectors/web/article-fetch.ts#L70-L82)：使用 `normalizeHttpUrl()` 比较，该函数移除 hash 但**保留query 参数**。若导航目标与实际 URL 的查询参数不同，会超时 8s | 建议在 `buildDiscourseTopicFloorUrl()` 确保 URL 末尾无尾部斜杠；测试 `/1?` vs `/1/` 的场景 |
| **🟡 MEDIUM** | **双重 URL 解析函数逻辑重复** | [parseDiscourseTopicUrl](webclipper/src/collectors/web/article-fetch.ts#L34-L56) vs [parseDiscourseTopicPath](webclipper/src/collectors/web/article-fetch.ts#L558-L573)：两个函数在不同作用域重复 Discourse pathPattern 匹配逻辑（正则 `DISCOURSE_TOPIC_PATH_RE` 定义两次） | 非功能缺陷。可选优化：提取共用常量或工具函数 |
| **🟡 MEDIUM** | **DOM 稳定等待超时默认 10s，URL 导航大超时 8s** | [extractArticleOnTab args](webclipper/src/collectors/web/article-fetch.ts#L756-L760)：默认 stabilizationTimeoutMs=10_000；[waitForTabUrl](webclipper/src/collectors/web/article-fetch.ts#L70-L82) 超时 8_000。若网络慢，DOM 稳定可能超 URL 等待 | 建议文档化这个顺序；考虑让 DOM 等待不超过总超时的 70% |
| **🟢 LOW** | **非 Discourse 页面中不会打印 debug 日志** | navigates-fallback 成功时有隐含的内容获取，第2次提取时无日志 | 可选：补充 console.debug() 或改进错误链路，便于调试 |

---

### **4) 重复/无用代码梳理**

#### 🟡 **函数重复：URL 解析**

**文件**：[article-fetch.ts](webclipper/src/collectors/web/article-fetch.ts)

| 函数 | 定义位置 | 返回类型 | 备注 |
|------|--------|--------|------|
| `parseDiscourseTopicUrl` | L34-56（主函数作用域） | `{ origin, slug, topicId, postNumber: number \| null }` | 从完整 URL 解析 |
| `parseDiscourseTopicPath` | L558-573（inject 脚本内） | `{ slug, topicId, postNumber: string \| null }` | 从 pathname 解析 |

**重复内容**：两者都使用 `DISCOURSE_TOPIC_PATH_RE` 匹配  
**影响**：无功能损伤，但代码笨重  

**可执行删除建议**：
```typescript
// 在 article-fetch.ts 顶部定义（或导入）统一的解析器：
function parseDiscourseTopicPath(pathname: string) {
  const match = String(pathname || '').match(DISCOURSE_TOPIC_PATH_RE);
  if (!match) return null;
  return { slug: match[1], topicId: match[2], postNumber: match[3] || null };
}

// 在注入脚本中复用，而非重定义
```

**预期收益**：减少 ~20 行重复代码，保持一致性。

---

#### 🟢 **部分死代码检查**

[L699](webclipper/src/collectors/web/article-fetch.ts#L699)：
```typescript
const discourseOpMissingOnCurrentPage = Boolean(discourseTopic);
```
- 看似"冗余"（若已经提取成功则无需保留 flag）
- 实际是**必要**：因为即使 OP 未找到，非 Discourse 页面也不应被标记为"OP 缺失"

**无无用代码发现**。

---

### **5) Findings 列表**

| # | Severity | Evidence | Fix Suggestion |
|----|----------|----------|-----------------|
| F1 | 🔴 HIGH | [article-fetch-service.test.ts L152-248](webclipper/tests/smoke/article-fetch-service.test.ts#L152) | 缺少"Discourse /1 仍无 OP 严格失败"的单元测试。P2-T3 虽已实现，但无测试覆盖，P3 validation 会发现。 | 在 `article-fetch-service.test.ts` 新增测试用例：模拟 Discourse 页面在 /1 时两次提取都返回 `warningFlags: ['discourse_op_missing_on_page']`，期望抛 `Discourse OP not found` 错误 |
| F2 | 🟡 MEDIUM | [waitForTabUrl L70-82](webclipper/src/collectors/web/article-fetch.ts#L70-L82) | URL 比较使用 `normalizeHttpUrl()` 移除 hash 但保留 query 参数。若实际导航 URL 携带查询参数会延迟识别 URL 到位。 | 建议在 `buildDiscourseTopicFloorUrl()` 确保返回的 URL 无查询参数；或在 `waitForTabUrl()` 时增加查询参数的规范化 |
| F3 | 🟡 MEDIUM | [article-fetch.ts L34-56 vs L558-573](webclipper/src/collectors/web/article-fetch.ts#L34-L56) | `parseDiscourseTopicUrl()` 和 `parseDiscourseTopicPath()` 重复定义 Discourse URL 解析逻辑，维护重复 `DISCOURSE_TOPIC_PATH_RE`。 | 提取为单一工具函数，或让注入脚本复用主函数作用域的解析器（需验证脚本隔离限制） |
| F4 | 🟡 MEDIUM | [extractArticleOnTab args L756-760](webclipper/src/collectors/web/article-fetch.ts#L756-L760) | DOM 稳定等待默认 10s，而 URL 导航等待只有 8s。若网络慢，后者会先超时，导致不必要的重试或错误信息误导。 | 补充文档或代码注释说明超时顺序；考虑让 fallback 后的 DOM 等待时间受 URL 等待结果动态调整（可选） |
| F5 | 🟢 LOW | [article-fetch-background-handlers.ts L9-12](webclipper/src/collectors/web/article-fetch-background-handlers.ts#L9-L12) | 错误规范化函数 `normalizeArticleFetchError()` 使用 regex `/^discourse op not found$/i` 检测，若错误信息被修改会失效。 | 建议定义错误常数（如 `DISCOURSE_OP_MISSING_ERROR_KEY`）在两个地方统一引用，而非 magic string |

---

### **6) P2 验证命令建议**

基于现有测试文件：

```bash
# 编译检查
npm --prefix webclipper run compile

# P2-T1 & T2 smoke 测试（已覆盖成功路径）
npm --prefix webclipper run test -- tests/smoke/article-fetch-service.test.ts

# P2-T3 验证（background 错误透传）
npm --prefix webclipper run test -- tests/smoke/background-router-article-fetch.test.ts

# 非 Discourse 页面回归检查
npm --prefix webclipper run test -- tests/smoke/article-fetch-wechat-gallery.test.ts

# **建议补充**：
# npm --prefix webclipper run test -- 'tests/smoke/article-fetch-service.test.ts' -t 'strict.*fail|OP not found'
# （执行点 F1 补充的测试）

# 完整 smoke
npm --prefix webclipper run test -- tests/smoke/
```

---

### **总结**

| 维度 | 评分 |
|------|------|
| **功能完整性** | ✅ 95% （缺 P2-T3 测试覆盖） |
| **代码质量** | ✅ 85% （有微妙的重复，无严重问题） |
| **错误处理** | ✅ 90% （透传清晰，但缺测试验证） |
| **防循环 / 竞态** | ✅ 100% （逻辑严密） |
| **非 Discourse 回归** | ✅ 100% （分支清晰） |

**建议**：P2 phase代码可线上部署（functional readiness），但建议在 P3补齐 F1 测试覆盖，并在生产运行2-3周后用户反馈补齐 F2-F4 的边界优化。