# 计划 P3：剩余站点 + 加固

## 目标

完成剩余的“难站点”（可能涉及 shadow DOM / 特殊编辑器 / 更严格的按键拦截），并做跨平台加固，最终 `gate` 全绿。

## P3-T1：Gemini（gemini.google.com）

### 范围 / 文件

- 新增 adapter：`webclipper/src/services/integrations/item-mention/content/editor-gemini.ts`
- 新增测试：`webclipper/tests/smoke/item-mention-gemini.test.ts`

### 实施步骤

1. web-access 调研 Gemini composer：确认是否在 shadow DOM/iframe 内，以及是否可达。
2. 若不可达（closed shadow root 或跨域 iframe），必须：
   - 在 `.audit/site-survey.md` 写明证据与结论
   - registry 中让该 host 不激活 `$ mention`（避免半成品）
3. 若可达：实现站点 adapter + smoke test，并确认 `Tab/Enter` 插入不会发送消息。

### 提交

- `feat: P3-T1 - 支持 Gemini 的 $ mention`

## P3-T2：Google AI Studio（aistudio.google.com / makersuite.google.com）

### 范围 / 文件

- adapter：`webclipper/src/services/integrations/item-mention/content/editor-googleaistudio.ts`
- 测试：`webclipper/tests/smoke/item-mention-googleaistudio.test.ts`

### 提交

- `feat: P3-T2 - 支持 Google AI Studio 的 $ mention`

## P3-T3：跨平台加固：防误触发 + 插入兼容 + 清理多余代码

### 覆盖点

- 确保 controller 不会在非支持 host 上启动。
- 确保 IME composing 流程不会误触发 pick。
- 确保候选框定位稳定且不溢出视口。
- 清理在适配迭代中引入的死代码/重复代码。
- 确保只有在 session open 时才拦截键盘事件，避免影响站点原生输入。
- 确保 `Tab/Enter` pick 时 `preventDefault + stopPropagation` 生效（必要时 `stopImmediatePropagation`）。

### 提交

- `fix: P3-T3 - 跨平台加固 item mention`

## P3-T4：最终验证：逐站点冒烟 + 通过 gate

### 验证

- `npm --prefix webclipper run gate`

### 手工冒烟清单（web-access）

逐站点跑一遍，确认：

- 输入 `$` 打开候选
- `Tab/Enter` 插入 Markdown
- `Escape` 关闭且不删除已输入内容

### 提交

- `chore: P3-T4 - 完成多平台 $ mention 验收`

## 审计 P3

完成 P3 后，更新 `audit-p3.md`：

- 已知无法支持的站点限制（如 closed shadow root / 跨域 iframe）
- 回归风险
- 后续可选项（如按站点开关、selector 进一步稳健化）
