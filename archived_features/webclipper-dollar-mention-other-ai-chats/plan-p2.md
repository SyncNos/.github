# 计划 P2：逐站点适配（第一批）

## 目标

按“一个站点一个 task”的节奏落地适配器。每个站点都必须包含：

- adapter 实现
- registry 路由接入
- smoke test
- web-access 真实验证证据

## web-access Runbook（每个站点任务都必须执行）

注意：必须优先使用 `http://127.0.0.1:3456`（不要用 `localhost`，避免 IPv6/解析差异）。

1. 启动/确认 CDP proxy：
   - `nohup node ~/.codex/skills/web-access/scripts/cdp-proxy.mjs > /tmp/cdp-proxy.log 2>&1 &`
   - `curl -fsS http://127.0.0.1:3456/health`
2. 若 /health 失败：
   - `pkill -f cdp-proxy.mjs || true`
   - 重新执行第 1 步，并 `tail -n 50 /tmp/cdp-proxy.log` 查看错误
3. 新开 tab 打开站点：
   - `curl -s "http://127.0.0.1:3456/new?url=https://<site>"`
4. 记录 targetId 后，检查页面基础信息：
   - `curl -s "http://127.0.0.1:3456/info?target=<ID>"`
5. 用 /eval 调研 composer（写入 `.audit/site-survey.md` 对应条目）：
   - `curl -s -X POST "http://127.0.0.1:3456/eval?target=<ID>" -d '<js>'`
6. 用真实交互验证 `$ mention`（必须确认 Tab/Enter 不会发送消息）：
   - focus composer，输入 `$`，确认 DOM 中出现 `webclipper-inpage-item-mention`
   - `ArrowDown/ArrowUp` 移动高亮
   - `Enter` / `Tab` 插入并确认 `$...` 被替换为 Markdown
   - `Escape` 关闭但不删文本
7. 任务完成后关闭该 tab（只关自己创建的）：
   - `curl -s "http://127.0.0.1:3456/close?target=<ID>"`

## 站点任务统一模板（P2-T1 ~ P2-T7 都要满足）

每个站点的 adapter 至少要做到：

- 只在“站点 chat composer 语境”才 detect 成功（避免误匹配站点其它输入框）
- 候选框打开时按键拦截必须生效：
  - `Enter/Tab` 触发 pick 插入
  - 必须阻止站点默认行为（尤其是 Enter 发送消息、Tab 切走焦点）
- 候选框关闭时不拦截任何按键（不影响站点原生输入）

每个站点至少要有 1 个 smoke test 覆盖：

- `$` 打开 + `Enter` 插入（必测）
- `Tab` 插入（尽量测；若 jsdom 环境不稳定，需在 web-access 中补充强证据）

## P2-T1：Claude（claude.ai）

### 范围 / 文件

- New adapter: `webclipper/src/services/integrations/item-mention/content/editor-claude.ts`
- Routing update: registry used by `mention-controller.ts`
- New test: `webclipper/tests/smoke/item-mention-claude.test.ts`

### 实施步骤

1. web-access 调研 Claude composer：类型、selector、按键语义（写入 `.audit/site-survey.md`）。
2. 实现 adapter（detect 必须保守）：
   - 优先使用 `document.activeElement` 与站点语义信号
   - 只有在“确认处于 Claude chat composer 语境”时才返回 editor
3. 增加 smoke test：
   - 覆盖 `$` 打开、`Enter` 插入、`Tab` 插入（至少二选一，优先 Enter + Tab 都测）
4. web-access 真实验证：`$` 打开，`Tab/Enter` 插入且不发送消息。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ claude`

### 提交

- `feat: P2-T1 - 支持 Claude 的 $ mention`

## P2-T2：DeepSeek（chat.deepseek.com）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-deepseek.ts`
- `webclipper/tests/smoke/item-mention-deepseek.test.ts`

### 实施步骤

按“站点任务统一模板”执行：

1. web-access 调研并记录 `.audit/site-survey.md`
2. 实现 adapter + registry 接入
3. 增加 smoke test（至少覆盖 `$` 打开 + `Enter` 插入）
4. web-access 真实验证 `Tab/Enter` 插入不发送消息

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ deepseek`

### 提交

- `feat: P2-T2 - 支持 DeepSeek 的 $ mention`

## P2-T3：Kimi（kimi.moonshot.cn / kimi.com）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-kimi.ts`
- `webclipper/tests/smoke/item-mention-kimi.test.ts`

### 实施步骤

同上（按“站点任务统一模板”执行）。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ kimi`

### 提交

- `feat: P2-T3 - 支持 Kimi 的 $ mention`

## P2-T4：豆包（doubao.com）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-doubao.ts`
- `webclipper/tests/smoke/item-mention-doubao.test.ts`

### 实施步骤

同上（按“站点任务统一模板”执行）。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ doubao`

### 提交

- `feat: P2-T4 - 支持豆包的 $ mention`

## P2-T5：元宝（yuanbao.tencent.com）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-yuanbao.ts`
- `webclipper/tests/smoke/item-mention-yuanbao.test.ts`

### 实施步骤

同上（按“站点任务统一模板”执行）。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ yuanbao`

### 提交

- `feat: P2-T5 - 支持元宝的 $ mention`

## P2-T6：z.ai（chat.z.ai）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-zai.ts`
- `webclipper/tests/smoke/item-mention-zai.test.ts`

### 实施步骤

同上（按“站点任务统一模板”执行）。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ zai`

### 提交

- `feat: P2-T6 - 支持 z.ai 的 $ mention`

## P2-T7：Poe（poe.com）

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/editor-poe.ts`
- `webclipper/tests/smoke/item-mention-poe.test.ts`

### 实施步骤

同上（按“站点任务统一模板”执行）。

### 验证

- `npm --prefix webclipper run test -t item\\ mention\\ poe`

### 提交

- `feat: P2-T7 - 支持 Poe 的 $ mention`

## Audit P2

完成 P2 后，更新 `audit-p2.md`：

- 各站点 selector 的鲁棒性说明（为什么不会误匹配）
- 是否需要特殊插入 fallback（以及原因）
- 若因登录/风控无法验证：记录缺口与后续补验方式
