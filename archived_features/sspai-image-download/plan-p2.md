# Phase 2 Plan: 集成与验证

## Goal

- 将 `image-download-proxy` 集成到现有的图片下载流程中
- 添加单元测试确保核心逻辑正确（含 Chrome global mock）
- 完成编译和冒烟验证

## Non-goals

- 不重写整个 `image-inline.ts`，仅在有防盗链 URL 时走代理
- 不添加 UI 配置（默认对所有配置的防盗链域名生效）

---

## P2-T1 - 修改 image-inline.ts 集成 image-download-proxy

### 涉及文件

- 修改 `webclipper/src/services/conversations/data/image-inline.ts`

### 实现步骤

#### 1. 集成点分析

当前 `downloadImageAsBlob` 函数处理所有 HTTP 图片下载。我们需要在函数入口处添加对防盗链域名的检测和智能处理。

#### 2. 修改 downloadImageAsBlob

```typescript
async function downloadImageAsBlob(input: {
  url: string;
  referrer?: string;
  maxBytes: number;
}): Promise<
  | { ok: true; blob: Blob; byteSize: number; contentType: string }
  | { ok: false; reason: 'http' | 'non_image' | 'empty' | 'too_large' | 'fetch' }
> {
  const safeUrl = String(input.url || '').trim();
  if (!isHttpUrl(safeUrl)) return { ok: false, reason: 'fetch' };

  // 尝试使用智能下载（自动处理防盗链 Referer）
  try {
    const { downloadImageSmart } = await import('@platform/webext/image-download-proxy');
    const result = await downloadImageSmart({
      url: safeUrl,
      maxBytes: input.maxBytes,
    });

    if (result.ok) {
      // 智能下载成功
      return {
        ok: true,
        blob: result.blob,
        byteSize: result.byteSize,
        contentType: result.contentType,
      };
    }

    // 智能下载失败，记录日志并 fallthrough 到普通 fetch
    // 注意：即使域名不在 ANTI_HOTLINK_REFERER_MAP 中，downloadImageSmart 也会尝试普通下载
    // 所以这里只有当 URL 匹配防盗链规则但 DNR 失败时才会走 fallback
    if (result.reason !== 'fetch' && result.reason !== 'http') {
      // non_image / empty / too_large 这些是内容问题，不需要 fallback
      return result;
    }

    console.info('[ImageInline] smart download failed, falling back to plain fetch', {
      url: safeUrl,
      reason: result.reason,
    });
  } catch (e) {
    // 模块加载失败或其他异常，记录日志并 fallthrough
    console.warn('[ImageInline] smart download exception, falling back to plain fetch', {
      url: safeUrl,
      error: e instanceof Error ? e.message : String(e),
    });
  }

  // 普通下载逻辑（fallback）
  const referrer = isHttpUrl(input.referrer) ? String(input.referrer) : undefined;
  try {
    const res = await fetch(safeUrl, {
      method: 'GET',
      credentials: 'include',
      redirect: 'follow',
      ...(referrer ? { referrer } : {}),
    });
    if (!res.ok) return { ok: false, reason: 'http' };

    const contentType = parseContentType(res.headers.get('content-type') || '');
    if (!contentType.startsWith('image/')) return { ok: false, reason: 'non_image' };

    const blob = await res.blob();
    const byteSize = blob.size || 0;
    if (!byteSize) return { ok: false, reason: 'empty' };
    if (byteSize > input.maxBytes) return { ok: false, reason: 'too_large' };

    return { ok: true, blob, byteSize, contentType };
  } catch (_e) {
    return { ok: false, reason: 'fetch' };
  }
}
```

#### 3. 设计决策

- **智能优先**：所有 HTTP 图片都先走 `downloadImageSmart`，由模块内部自动判断是否需要防盗链处理
- **Fallback 保障**：智能下载失败时 fallthrough 到原有逻辑
  - ⚠️ **重要说明**：对于防盗链图片，fallback 路径（普通 fetch）**无法设置 Referer header**，仍会 403
  - fallback 的意义在于：非防盗链图片在 `downloadImageSmart` 内部异常时，仍有最后的下载机会
  - Firefox 下防盗链图片必定 403（DNR 不可用），这是已知限制
- **动态导入**：使用 `await import()` 确保仅需要时加载额外模块
- **错误隔离**：`try/catch` 包裹整个智能下载路径，任何异常都不影响普通下载

#### 4. 与原有 referrer 参数的关系

原有 `referrer` 参数用于设置 fetch 的 `ReferrerPolicy`（如 `no-referrer`、`origin`），这在某些网站也有用。智能下载失败后的 fallback 路径仍会使用该参数，确保兼容性。

> ⚠️ **区分**：
> - `referrer`（fetch 选项）→ 控制 `Referrer-Policy`，决定浏览器发送的 Referer 策略
> - `Referer`（HTTP header）→ 具体的来源 URL 值（如 `https://sspai.com/`）
> 
> 少数派需要的是 `Referer: https://sspai.com/` 这个具体的 header 值，不是 ReferrerPolicy。
> 这就是为什么我们需要 DNR API 来注入 header，而不是依赖 fetch 的 referrer 选项。

### 验证

- 运行 `npm --prefix webclipper run compile` 确认类型正确
- 确认非防盗链网站的图片下载逻辑不受影响

### 提交要求

```
feat: P2-T1 - 集成 image-download-proxy 到图片下载流程
```

---

## P2-T1.5 - 添加智能强制缓存逻辑到 article-fetch.ts

### 涉及文件

- 修改 `webclipper/src/collectors/web/article-fetch.ts`

### 背景

当前 `web_article_cache_images_enabled` 关闭时，防盗链图片（如少数派 CDN）查看时 100% 403。需要在 `fetchActiveTabArticle` 中添加智能判断：如果文章包含防盗链域名的图片，则强制缓存。

### 实现步骤

#### 1. 添加防盗链检测函数

在 `article-fetch.ts` 文件顶部（或 `fetchActiveTabArticle` 函数之前）添加：

```typescript
// 从 image-download-proxy 导入配置（动态导入避免循环依赖）
async function hasAntiHotlinkImages(markdown: string): Promise<boolean> {
  try {
    const { ANTI_HOTLINK_REFERER_MAP } = await import('@platform/webext/image-download-proxy');
    const domains = Object.keys(ANTI_HOTLINK_REFERER_MAP);
    return domains.some(domain => {
      const escaped = domain.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
      return new RegExp(escaped, 'i').test(markdown);
    });
  } catch {
    // 模块加载失败时保守处理：不强制缓存
    return false;
  }
}
```

#### 2. 修改图片缓存判断逻辑

找到原有的缓存判断代码（约第 230-248 行）：

```typescript
// 原有代码：
const local = await storageGet(['web_article_cache_images_enabled']);
if (local?.web_article_cache_images_enabled === true) {
  const inlined = await inlineChatImagesInMessages({...});
  messagesToSave = inlined.messages;
}
```

修改为：

```typescript
// 新代码：智能强制缓存
const local = await storageGet(['web_article_cache_images_enabled']);
const hasAntiHotlink = await hasAntiHotlinkImages(markdownContent);
const shouldCacheImages =
  local?.web_article_cache_images_enabled === true || hasAntiHotlink;

if (shouldCacheImages) {
  const inlined = await inlineChatImagesInMessages({
    conversationId,
    conversationUrl: canonicalUrl,
    messages: messagesToSave,
    enableHttpImages: true,
  });
  messagesToSave = inlined.messages;
  
  if (hasAntiHotlink && !local?.web_article_cache_images_enabled) {
    console.info('[ArticleFetch] auto-caching anti-hotlink images (user setting is off)', {
      conversationId,
      url: canonicalUrl,
    });
  }
}
```

#### 3. 导出 ANTI_HOTLINK_REFERER_MAP

在 `image-download-proxy.ts` 中导出配置表供检测使用：

```typescript
export { ANTI_HOTLINK_REFERER_MAP };
```

#### 4. 依赖验证

> ⚠️ **重要**：`article-fetch.ts` 在 `@collectors/web/` 下，`image-download-proxy.ts` 在 `@platform/webext/` 下。
> 需要确认 `@collectors` 可以 import `@platform`。
> 
> 根据 AGENTS.md 的依赖方向：`services -> (platform, domain, ...)` 是允许的。
> `collectors` 通常被视为平台适配层的一部分，可以依赖 `platform`。
> 
> 实现前先用 `npm --prefix webclipper run compile` 验证导入路径是否正确。

### 设计理由

1. **兜底策略**：不开启缓存就 100% 会 403，缓存是唯一的解决方案
2. **用户透明**：用户不需要理解"防盗链"概念，扩展自动处理
3. **非防盗链网站仍受用户设置控制**：普通博客的图片是否缓存由用户决定
4. **动态导入避免循环依赖**：`article-fetch.ts` 和 `image-download-proxy.ts` 可能不在同一层，动态导入避免编译问题

### 验证

- 关闭 `web_article_cache_images_enabled`
- 抓取少数派文章，确认图片仍被缓存
- 抓取普通博客文章，确认图片不被缓存（符合用户设置）

### 提交要求

```
feat: P2-T1.5 - 添加智能强制缓存逻辑，防盗链图片不受用户设置控制
```

---

## P2-T2 - 添加 image-download-proxy 单元测试

### 涉及文件

- 新建 `webclipper/src/platform/webext/image-download-proxy.test.ts`

### 测试环境准备

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

// Mock chrome.declarativeNetRequest API
const mockUpdateSessionRules = vi.fn().mockResolvedValue(undefined);

vi.stubGlobal('chrome', {
  declarativeNetRequest: {
    updateSessionRules: mockUpdateSessionRules,
  },
  runtime: {
    id: 'test-extension-id',
  },
});
```

### 测试用例

#### 1. 防盗链 URL 识别

```typescript
describe('getAntiHotlinkReferer', () => {
  it('returns referer for sspai CDN', () => {
    const result = getAntiHotlinkReferer('https://cdnfile.sspai.com/xxx.jpg');
    expect(result).toBe('https://sspai.com/');
  });

  it('returns null for unknown domains', () => {
    const result = getAntiHotlinkReferer('https://example.com/img.jpg');
    expect(result).toBeNull();
  });

  it('handles URLs with query params', () => {
    const result = getAntiHotlinkReferer(
      'https://cdnfile.sspai.com/xxx.jpg?imageView2/2/w/1120'
    );
    expect(result).toBe('https://sspai.com/');
  });
});
```

#### 2. DNR 规则注册/清理

```typescript
it('registers and cleans up DNR rule for anti-hotlink URLs', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    blob: () => Promise.resolve(new Blob(['test'], { type: 'image/jpeg' })),
    headers: { get: () => 'image/jpeg' },
  });

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(true);
  expect(mockUpdateSessionRules).toHaveBeenCalledTimes(2); // 注册 + 清理
  
  // 验证注册时添加了正确的规则
  const addCall = mockUpdateSessionRules.mock.calls[0][0];
  expect(addCall.addRules).toHaveLength(1);
  expect(addCall.addRules[0].action.requestHeaders[0]).toEqual({
    header: 'Referer',
    operation: 'set',
    value: 'https://sspai.com/',
  });
});
```

#### 3. 规则清理保证（即使 fetch 失败）

```typescript
it('cleans up DNR rule even when fetch fails', async () => {
  globalThis.fetch = vi.fn().mockRejectedValue(new Error('Network error'));

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(false);
  expect(mockUpdateSessionRules).toHaveBeenCalledTimes(2); // 注册 + 清理
});
```

#### 4. 非防盗链 URL 走普通下载

```typescript
it('uses plain fetch for non-anti-hotlink URLs', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    blob: () => Promise.resolve(new Blob(['test'], { type: 'image/jpeg' })),
    headers: { get: () => 'image/jpeg' },
  });

  const result = await downloadImageSmart({
    url: 'https://example.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(true);
  // 不应该调用 DNR API
  expect(mockUpdateSessionRules).not.toHaveBeenCalled();
});
```

#### 5. 下载成功/失败场景

```typescript
it('returns blob on success', async () => {
  const testBlob = new Blob(['test'], { type: 'image/jpeg' });
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    blob: () => Promise.resolve(testBlob),
    headers: { get: () => 'image/jpeg' },
  });

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(true);
  if (result.ok) {
    expect(result.blob).toBe(testBlob);
    expect(result.contentType).toBe('image/jpeg');
    expect(result.byteSize).toBe(testBlob.size);
  }
});

it('returns error when fetch returns 403', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: false,
    status: 403,
  });

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(false);
  expect((result as any).reason).toBe('http');
});

it('returns error for non-image content-type', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    blob: () => Promise.resolve(new Blob(['not an image'])),
    headers: { get: () => 'text/html' },
  });

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(false);
  expect((result as any).reason).toBe('non_image');
});
```

#### 6. 边界条件

```typescript
it('returns error when URL is empty', async () => {
  const result = await downloadImageSmart({ url: '' });
  expect(result.ok).toBe(false);
  expect((result as any).reason).toBe('invalid_input');
});

it('returns error when file exceeds maxBytes', async () => {
  const largeBlob = new Blob([new ArrayBuffer(3_000_000)], { type: 'image/jpeg' });
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    blob: () => Promise.resolve(largeBlob),
    headers: { get: () => 'image/jpeg' },
  });

  const result = await downloadImageSmart({
    url: 'https://cdnfile.sspai.com/test.jpg',
    maxBytes: 1_000_000,
  });

  expect(result.ok).toBe(false);
  expect((result as any).reason).toBe('too_large');
});
```

### 验证

- 运行 `npm --prefix webclipper run test` 确认所有测试通过

### 提交要求

```
test: P2-T2 - 添加 image-download-proxy 单元测试
```

---

## P2-T3 - 编译与冒烟验证

### 验证步骤

1. **编译检查**
   ```bash
   npm --prefix webclipper run compile
   npm --prefix webclipper run test
   npm --prefix webclipper run build
   npm --prefix webclipper run build:firefox  # 必须验证 Firefox 构建
   ```

2. **手动冒烟测试**（需安装扩展）
   
   **少数派文章测试（开启缓存设置）**：
   - 开启 `web_article_cache_images_enabled`
   - 访问少数派文章页（含图片，如 https://sspai.com/post/xxxxx）
   - 点击「抓取文章」
   - 确认：
     - [ ] 图片下载成功，console 无 `inline_images_download_failed` 警告
     - [ ] 保存的文章中图片 URL 被替换为 `syncnos-asset://<id>`
     - [ ] 在 popup/app 中查看已保存文章，图片能正常显示
   
   **少数派文章测试（关闭缓存设置，智能强制缓存）**：
   - 关闭 `web_article_cache_images_enabled`
   - 访问少数派文章页（含图片）
   - 点击「抓取文章」
   - 确认：
     - [ ] console 输出 `[ArticleFetch] auto-caching anti-hotlink images (user setting is off)`
     - [ ] 图片仍被缓存，URL 被替换为 `syncnos-asset://<id>`
     - [ ] 查看已保存文章，图片能正常显示（不再 403）
   
   **少数派文章重新抓取测试（验证已 403 的文章修复后正常）**：
   - 访问之前抓取过的少数派文章（图片可能已 403）
   - 重新点击「抓取文章」
   - 确认：
     - [ ] 图片被重新缓存，URL 被替换为 `syncnos-asset://<id>`
     - [ ] 查看已保存文章，图片能正常显示
   
   **非防盗链网站测试（关闭缓存设置）**：
   - 关闭 `web_article_cache_images_enabled`
   - 访问普通博客/文章页
   - 点击「抓取文章」
   - 确认：
     - [ ] 图片 URL 保持原始外链，未被替换为 `syncnos-asset://`
     - [ ] 符合用户设置（不缓存）
   
   **（可选）Firefox 测试**：
   - 在 Firefox 中加载扩展
   - 验证：
     - [ ] 扩展正常加载，console 无报错
     - [ ] DNR 不可用时降级为普通 fetch（会有 warn 日志）
     - [ ] 少数派图片可能 403（Firefox 无法注入 Referer），但不崩溃

### 提交要求

此 task 通常不需要独立提交，而是验证前面的改动。如果验证过程中发现问题需要修复，则用修复的 commit。

---

## Phase 2 审计要求（audit-p2.md）

Phase 2 完成后，需要审计：

1. **集成正确性**：确认 `image-inline.ts` 的智能下载路径正确，不影响原有逻辑
2. **智能强制缓存**：确认 `article-fetch.ts` 的 `hasAntiHotlinkImages` 逻辑正确，防盗链图片在关闭缓存设置时仍被缓存
3. **Fallback 路径**：确认智能下载失败时正确 fallthrough 到普通 fetch（注意：防盗链图片 fallback 仍会 403）
4. **通用性**：确认 `ANTI_HOTLINK_REFERER_MAP` 设计正确，未来添加新网站只需加配置
5. **测试覆盖率**：确认核心路径（防盗链识别、规则注册/清理、成功/失败、非防盗链 URL、边界条件、智能缓存判断）都有测试覆盖
6. **编译产物**：确认 Chrome 和 Firefox build 产物正常，manifest 权限正确
7. **回归测试**：确认非防盗链网站在关闭缓存设置时不被缓存
8. **DNR urlFilter 格式**：确认使用 `|${origin}/` filter list syntax，不是完整 URL 精确匹配
9. **模块依赖**：确认 `@collectors/web/article-fetch.ts` 可以动态导入 `@platform/webext/image-download-proxy.ts`
10. **性能**：确认多张图片并发下载时，DNR 规则注册/清理不会互相干扰
