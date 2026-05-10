# Idea: 通用防盗链图片下载适配

## 背景

少数派（sspai.com）文章的图片 URL 格式为：
```
https://cdnfile.sspai.com/2026/03/31/0a5933930cd907c435b5a96d453577a8.jpg?keys=imageView2/2/w/1120/q/90/interlace/1/ignore-error/1/format/webp
```

用户反馈：直接访问该链接返回 403 Forbidden，即使开启了 `web_article_cache_images_enabled` 功能也无法下载图片。

## 问题根因

经过调研确认，少数派 CDN 存在两层保护机制：

1. **Referer 防盗链**（主要障碍）
   - 要求 `Referer: https://sspai.com/` 才能访问图片
   - Chrome MV3 扩展的 background service worker 使用 `fetch()` 时无法直接设置 Referer header（浏览器安全规范限制）
   - 当前代码虽然传入 `referrer` 参数，但在 background 上下文中 fetch 的 referrer 策略是 `no-referrer`

2. **七牛云图片处理参数**（次要问题）
   - `?keys=imageView2/...` 是七牛云的图像处理参数
   - 去除后可获取原始图片 URL，但仍需正确 Referer 才能访问

## 技术调研结论

- iOS/macOS 原生 App（如 GoodLinks）使用 Kingfisher 等库可自由设置 Referer（通过 `requestModifier`）
- Chrome MV3 扩展需使用 `declarativeNetRequest` API 动态注册规则修改请求头
- 这是浏览器扩展中最接近 Kingfisher 能力的标准方案
- **这是一个通用问题**：微信公众号、知乎、微博等很多网站都有类似的 Referer 防盗链

## 核心需求

1. 在抓取少数派文章时，能够成功下载并缓存图片到本地 IndexedDB
2. 下载的图片 URL 在保存后能被替换为 `syncnos-asset://` 本地协议
3. **设计必须是通用的**：未来添加其他防盗链网站（微信公众号、知乎等）只需在配置表中加一行
4. 不影响其他网站的图片下载逻辑
5. Firefox 兼容（DNR 不可用时降级为普通 fetch）
6. **智能强制缓存**：对于配置在 `ANTI_HOTLINK_REFERER_MAP` 中的防盗链域名，无论用户是否开启 `web_article_cache_images_enabled`，都强制缓存这些图片（因为不开启缓存就 100% 会 403，缓存是唯一的解决方案）

## 设计原则

**不是"少数派专用"，而是"通用防盗链下载服务 + 配置表"**

```
image-download-proxy/
├── 核心逻辑：DNR 规则注册/清理、fetch 封装
└── 配置表：ANTI_HOTLINK_REFERER_MAP（域名 → Referer 映射）
```

未来添加新网站：
```typescript
const ANTI_HOTLINK_REFERER_MAP: Record<string, string> = {
  'cdnfile.sspai.com': 'https://sspai.com/',
  'mmbiz.qpic.cn': 'https://mp.weixin.qq.com/',      // 微信公众号
  'picx.zhimg.com': 'https://www.zhihu.com/',        // 知乎
  // ... 未来可继续添加
};
```

## 智能强制缓存策略

### 问题

当前 `web_article_cache_images_enabled` 关闭时：
- 图片 URL 保持原始外链
- 防盗链图片（如少数派 CDN）查看时 100% 403
- 用户不知道需要开启这个功能

### 解决方案

对于包含防盗链域名的文章，**强制缓存图片**，不受用户设置控制：

```typescript
// 检查是否包含防盗链图片
function hasAntiHotlinkImages(markdown: string): boolean {
  const antiHotlinkDomains = Object.keys(ANTI_HOTLINK_REFERER_MAP);
  return antiHotlinkDomains.some(domain => {
    const escaped = domain.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    return new RegExp(escaped, 'i').test(markdown);
  });
}

// 在 fetchActiveTabArticle 中
const local = await storageGet(['web_article_cache_images_enabled']);
const shouldCacheImages = 
  local?.web_article_cache_images_enabled === true ||
  hasAntiHotlinkImages(markdownContent);  // ← 新增：防盗链图片强制缓存

if (shouldCacheImages) {
  const inlined = await inlineChatImagesInMessages({...});
  messagesToSave = inlined.messages;
}
```

### 设计理由

1. **兜底策略**：不开启缓存就 100% 会 403，缓存是唯一的解决方案
2. **用户透明**：用户不需要理解"防盗链"概念，扩展自动处理
3. **非防盗链网站仍受用户设置控制**：普通博客的图片是否缓存由用户决定
4. **存储空间可接受**：防盗链图片通常不会太多，且受 `INLINE_HTTP_IMAGES_MAX_COUNT` 等限制

## 验收标准

- [ ] 在少数派文章页点击「抓取文章」后，图片能成功下载到本地
- [ ] 保存的文章内容中，图片 URL 被替换为 `syncnos-asset://<id>`
- [ ] 查看已保存文章时，图片能正常显示（不再 403）
- [ ] 非防盗链网站的图片下载逻辑不受影响
- [ ] `ANTI_HOTLINK_REFERER_MAP` 设计支持未来扩展，新网站只需加配置
- [ ] Firefox 构建不报错（DNR 降级为普通 fetch）
- [ ] 通过 `npm --prefix webclipper run compile` 编译检查
- [ ] 通过 `npm --prefix webclipper run test` 单元测试

## 约束

- 最小化权限变更，仅针对配置的防盗链域名
- 使用 `updateSessionRules` 而非 `updateDynamicRules`（会话级规则，浏览器关闭后自动清除）
- 规则注册/清理需做好错误处理，避免规则泄漏
