# comments-sidebar-selection-commit · idea

## 背景 / 触发

- 当前 comments sidebar 的“附加选区（quote）”自动触发依赖 `document.selectionchange` + `setTimeout(300)` 的时间防抖。
- 该策略会把“用户是否完成选中操作”错误地近似为“静置 300ms”，在鼠标/触摸拖拽、键盘选区、以及页面复杂交互下存在不确定性。
- 同时，为了避免输入框交互导致 empty selectionchange 清空引用，现有实现引入了额外的时间窗口（约 320ms）做空选区抑制，进一步增加隐式时间逻辑。

## 核心需求（原始需求精炼）

1. 将“选区附加 quote”的触发从**时间防抖**改为**操作完成事件**：
   - 鼠标/触摸：以 `pointerup` 视为选中完成时机。
   - 键盘：以 `keyup`（包含 shift+方向键结束）视为选中完成时机。
   - 拖拽/按键过程中不实时更新（只在完成时更新一次）。
2. quote 不再因空选区或取消选中而自动清空；只允许通过 quote 卡片右上角的 `❌` 显式清空。
3. 行为必须同时覆盖：
   - inpage overlay comments sidebar（content script 注入的右侧侧边栏）
   - app comments sidebar（WebClipper App 内右侧侧边栏）
4. 及时删除老旧代码：每个 phase 在引入新机制时就同步移除被替代的旧逻辑，不把清理集中到最后一个 phase。

## 默认值与兼容策略

- 默认启用“操作完成事件提交”机制；不新增设置项，不改存储 schema。
- 键盘选区与指针选区统一走同一提交路径；空选区提交应直接忽略（不触发请求、不改变 quote）。
- `❌` 清空后允许用户再次选择相同文本并重新附加（不能被 signature dedupe 永久屏蔽）。

## 非目标（明确不做什么）

- 不做拖拽/按键过程中的 quote 实时预览更新。
- 不新增“手动附加选区”按钮（保持现状无该按钮）。
- 不改动评论保存、回复、删除、定位等既有业务语义（除 quote 清空方式变为显式：不再在保存成功等路径中隐式清空）。

## 验收标准（可检查）

- 鼠标拖拽选区：松手（`pointerup`）后 quote 更新；拖拽过程中不更新。
- 键盘选区：松开按键（`keyup`）后 quote 更新。
- 取消选中/空选区：不清空 quote，不触发 selection request。
- 点击 quote 卡片右上角 `❌`：quote 清空，且再次选择同样文本仍能重新附加。
- `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile && npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test && npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build` 通过；相关 unit/smoke 覆盖事件触发与清空语义。
