# Phase 1 Audit: 通用防盗链图片下载基础设施

## 审计状态

- **审计时间**: 2026-04-05
- **审计范围**: P1-T1, P1-T2, P1-T3
- **审计结果**: 进行中

---

## 审计项

### 1. 权限最小化

- [ ] 确认只添加了必要的权限，没有过度授权
- **证据**: `wxt.config.ts` 仅添加 `declarativeNetRequestWithHostAccess` 和 `https://cdnfile.sspai.com/*`
- **状态**: ✅ 通过

### 2. 通用性

- [ ] 确认 `ANTI_HOTLINK_REFERER_MAP` 设计支持未来扩展
- **证据**: `image-download-proxy.ts` 导出 `ANTI_HOTLINK_REFERER_MAP`，新网站只需加一行
- **状态**: ✅ 通过

### 3. 规则安全性

- [ ] 确认 DNR 规则仅作用于配置的域名，不影响其他网站
- **证据**: `urlFilter: '|${origin}/'` 仅匹配特定域名，`initiatorDomains` 限制为当前扩展
- **状态**: ✅ 通过

### 4. 错误处理

- [ ] 确认 DNR API 调用失败时有合理的降级策略（Firefox 不崩溃）
- **证据**: `isDnrSupported()` 检测，Firefox 降级为 `downloadWithPlainFetch`，`try/finally` 清理规则
- **状态**: ✅ 通过

### 5. 代码风格

- [ ] 确认新文件符合项目 TypeScript 规范和分层约定
- **证据**: 文件位于 `@platform/webext/image-download-proxy.ts`，使用 `(globalThis as any).chrome` 与项目一致
- **状态**: ✅ 通过

### 6. ruleId 类型

- [ ] 确认所有 `ruleId` 都是正整数，不是字符串或小数
- **证据**: `Math.floor(Math.random() * 1000000) + 1` 确保正整数
- **状态**: ✅ 通过

### 7. Firefox 兼容

- [ ] 确认 Firefox MV3 构建不报错（DNR 降级路径正确）
- **证据**: `build:firefox` 成功，代码中 `isDnrSupported()` 检测 DNR 可用性
- **状态**: ✅ 通过

### 8. DNR urlFilter 格式

- [ ] 确认使用 `|${origin}/` filter list syntax，不是完整 URL 精确匹配
- **证据**: `domainPrefix = '|${origin}/'` 使用 filter list syntax
- **状态**: ✅ 通过

### 9. chrome.runtime.id 处理

- [ ] 确认 `initiatorDomains` 仅在 `chrome.runtime.id` 存在时设置
- **证据**: `if (isChrome && chrome.runtime?.id)` 保护
- **状态**: ✅ 通过

### 10. 性能注释

- [ ] 确认代码中有多图并发场景的性能说明注释
- **证据**: 文件顶部有性能说明注释，说明未来可优化为域名规则共享
- **状态**: ✅ 通过

---

## 审计总结

- **通过项**: 10/10
- **待修复项**: 0
- **建议**: 无

## 审计结论

✅ Phase 1 审计通过，可以进入 Phase 2。
