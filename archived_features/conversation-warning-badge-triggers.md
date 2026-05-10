# 左侧列表「警告」徽标触发条件（对应截图中的【警告】）

> 目标：解释会话列表卡片右上角 `警告` 徽标何时出现。  
> 截图对应 UI：`src/ui/conversations/ConversationListPane.tsx`

## 1) 唯一显示条件

会话卡片显示 `警告` 徽标的条件非常直接：

- `conversation.warningFlags` 是数组
- 且数组长度 `> 0`

代码位置：

- `src/ui/conversations/ConversationListPane.tsx`
  - `hasWarningFlags(conversation)`
  - 渲染分支：`{hasWarningFlags(...) ? <span>{t('warningBadge')}</span> : null}`

也就是说，这不是“随机 UI 告警”，而是**该会话有 warning flag 元数据**。

## 2) 这些 warningFlags 从哪里来

核心来源是采集阶段写入会话记录时带上的 `warningFlags`：

- `src/services/conversations/data/storage-idb.ts`
  - `upsertConversation(payload)` 会把 `payload.warningFlags` 写入会话

对于你截图中的 `Google AI Studio` 会话：

- 来源是 `src/collectors/googleaistudio/googleaistudio-collector.ts`
- 该 collector 在处理图片（尤其 blob 图片）时会累计 warning flags

## 3) Google AI Studio 场景下，常见触发原因

以下任意情况发生，都可能让 `warningFlags` 非空，从而出现左侧【警告】：

1. blob 图片数量超过上限（12）
   - `inline_images_limit_reached`
2. blob 图片总字节超过上限（8MB）
   - `inline_images_total_bytes_limit_reached`
3. 环境不能 fetch blob 图
   - `inline_images_fetch_unavailable`
4. blob fetch 失败
   - `inline_images_fetch_failed`
5. blob 不是图片类型
   - `inline_images_non_image_blob`
6. blob 内容为空
   - `inline_images_empty_blob`
7. 单图过大（> 2MB）
   - `inline_images_single_too_large`
8. blob 转 data URL 失败
   - `inline_images_encode_failed`

## 4) 为什么你可能“没看到失败，但看到了警告”

warning 是“降级成功但有瑕疵”的语义：

- 会话仍然会保存成功
- 但某些图片未能完整内联/处理
- 所以在会话级别打了 warning flag

因此会出现“内容看起来正常 + 左侧有警告徽标”的情况。

## 5) 什么时候会消失

当该会话下一次重新采集并写入时：

- 如果 `warningFlags` 变为空数组 `[]`，徽标会消失
- 如果仍有任一 warning flag，徽标会继续显示

## 6) 如何快速确认具体是哪一类 warning

可用以下低成本方式定位：

1. 在会话详情执行“复制完整 Markdown”
2. 看头部是否有 `Warnings: ...` 行（会列出具体 code）

相关格式化代码：`src/services/conversations/domain/markdown.ts`
