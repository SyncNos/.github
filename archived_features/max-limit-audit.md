# WebClipper Max / Limit 审计（详细版）

> 范围：`webclipper` 主链路（采集、图片、分页、同步、增量保存、Deep Research）。  
> 目标：仅保留**当前仍在代码中生效**的参数，并说明其限制对象、触发行为、存在原因与风险。

## 1) 采集链路参数（你点名的 3 个）

文件：`src/collectors/googleaistudio/googleaistudio-collector.ts`

### 1.1 `settleMs = Math.max(0, Number(options.settleMs) || 80)`

- **限制对象**：每个 turn 在“检测到内容”后的额外稳定等待。
- **触发行为**：每轮采集在真正提取文本/图片前，再等 `settleMs`。
- **存在原因**：Google AI Studio 的 DOM 常在首帧后继续排版/补图；立即采集会漏图或拿到半成品 DOM。
- **移除风险**：采集完整性下降（尤其多图、复杂结构回复）。
- **结论**：建议保留；可从 80ms 逐步下调验证。

### 1.2 `perTurnTimeoutMs = Math.max(120, Number(options.perTurnTimeoutMs) || 900)`

- **限制对象**：单个 turn 等待“内容出现”的最大时长。
- **触发行为**：超过该时间仍无有效内容，则结束该 turn 的等待。
- **存在原因**：避免无限等待，保护采集可终止性。
- **移除风险**：慢网或页面异常时可能卡在长等待；若完全去掉上限，可能进入无界等待。
- **结论**：建议保留；属于稳定性边界。

### 1.3 `pollMs = Math.max(30, Number(options.pollMs) || 80)`

- **限制对象**：轮询频率下限。
- **触发行为**：每 `pollMs` 检查一次内容是否就绪。
- **存在原因**：防止轮询过密引发 CPU 抖动。
- **移除风险**：若允许极低间隔（如 1ms），会显著增加前台开销。
- **结论**：建议保留；这是典型性能保护阈值。

## 2) 仍保留的重要限制（逐项说明）

### A. Web 文章抓取

文件：`src/collectors/web/article-fetch.ts`

1. `ARTICLE_STABILIZATION_TIMEOUT_MS = 10_000`
   - **限制对象**：等待文章 DOM 稳定的最长时间。
   - **意义**：给懒加载/SSR-hydration 留窗口，同时避免无限等待。
   - **删掉风险**：要么等待不足导致漏内容，要么等待过长拖慢抓取。

2. `ARTICLE_STABILIZATION_MIN_TEXT_LENGTH = 240`
   - **限制对象**：判定“文章已可抓取”的最小文本量。
   - **意义**：过滤空壳页面、骨架屏、错误页。
   - **删掉风险**：误抓低质量内容（大量空白/占位）。

3. `DISCOURSE_NAVIGATION_WAIT_TIMEOUT_MS = 10_000`
   - **限制对象**：Discourse 跳转/回退等待时长。
   - **意义**：兼容 OP 首帖兜底流程，不让单站点拖垮整次抓取。
   - **删掉风险**：异常页面进入长阻塞。

4. `CONTENT_MESSAGE_RETRY_DELAY_MS = 320`
   - **限制对象**：内容消息重试节奏。
   - **意义**：避免紧密重试造成抖动和竞争。
   - **删掉风险**：错误重试风暴、负载尖峰。

### B. 列表分页

文件：`src/services/conversations/background/handlers.ts`

1. `limit` 最大值 `200`
   - **限制对象**：单次返回会话条数。
   - **意义**：防止一次性返回过多导致 UI 首屏卡顿。
   - **删掉风险**：大库场景下前端渲染与序列化压力显著上升。

### C. 自动保存增量引擎

文件：`src/services/conversations/content/autosave-incremental-engine.ts`

1. `MAX_WINDOW_MESSAGES = 200`
   - **限制对象**：一次增量对比窗口大小。
   - **意义**：控制 diff 成本、事务大小。
   - **删掉风险**：窗口过大导致计算和写入耗时激增。

2. `MIN_OVERLAP_FOR_LONG_WINDOWS = 8`
   - **限制对象**：长窗口重叠阈值。
   - **意义**：确保增量匹配稳定，避免误判“新旧消息错位”。
   - **删掉风险**：增量错配概率上升。

3. `SEED_MAX_MESSAGES = 6`
   - **限制对象**：冷启动基准窗口。
   - **意义**：降低首轮比对开销。
   - **删掉风险**：首轮策略波动、启动开销上升。

### D. Notion 同步

文件：`src/services/sync/notion/notion-sync-service.ts`

1. `MAX_TEXT = 1900`
   - **限制对象**：单 Notion 富文本块字符数。
   - **意义**：贴近 Notion API 硬边界并留安全余量。
   - **删掉风险**：直接触发 Notion API 错误，导致同步失败。

2. `APPEND_BATCH = 90`
   - **限制对象**：单次 append block 批次大小。
   - **意义**：降低超时与限流概率。
   - **删掉风险**：大批次更容易触发 408/429。

3. `APPEND_MAX_ATTEMPTS = 5`
   - **限制对象**：append 重试上限。
   - **意义**：在可恢复错误下自动恢复，但避免无限重试。
   - **删掉风险**：要么恢复能力差，要么无限重试。

4. `CLEAR_DELETE_CONCURRENCY = 6` / `CLEAR_DELETE_MAX_ATTEMPTS = 5`
   - **限制对象**：并发删除与重试。
   - **意义**：平衡清理速度和 Notion 限流。
   - **删掉风险**：并发过高触发速率限制。

### E. 同步任务状态

文件：`src/services/sync/sync-job-store.ts`

1. `DEFAULT_STALE_MS = 1_200_000`（20 分钟）
   - **限制对象**：任务“过期/僵死”判定窗口。
   - **意义**：避免“永远运行中”的假活状态。
   - **删掉风险**：任务状态无法收敛。

2. stale 下限 `60_000`
   - **限制对象**：最短过期窗口。
   - **意义**：防止过于激进误杀正常任务。
   - **删掉风险**：误判频发。

### F. Deep Research 与模型应用节流

1. `src/services/bootstrap/content-controller.ts`
   - `DEEP_RESEARCH_POLL_MAX_DURATION_MS = 180_000`
   - `DEEP_RESEARCH_HYDRATE_MIN_INTERVAL_MS = 12_000`
   - **意义**：控制轮询时长和补水频率，避免长时间高频轮询。

2. `src/services/integrations/notionai-auto-picker/notionai-model-picker.ts`
   - `APPLY_MIN_INTERVAL_MS = 2500`
   - **意义**：防抖节流，避免快速重复触发模型切换逻辑。

## 3) 分类结论

### 建议保留（保护性阈值）

- 采集轮询下限/超时边界（`settleMs`、`perTurnTimeoutMs`、`pollMs`）
- Notion API 相关上限（`MAX_TEXT`、`APPEND_BATCH`、并发/重试）
- 同步任务 stale 判定边界

### 可讨论放宽（策略阈值）

- 分页 `limit` 最大值
- 自动保存窗口参数
- Deep Research 轮询时长/频率

## 4) 说明

- 本文中的“限制”不等于“坏设计”；很多是**防故障的边界条件**。
- 下一步如果继续去限制，建议按“低风险策略阈值 → 高风险 API/稳定性阈值”的顺序推进。
