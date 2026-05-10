抱歉刚才说得有点想当然了，Chrome 商店的多语言机制其实**不在 Dashboard 的下拉菜单里**——它是由你的**扩展包本身**驱动的。我重新讲清楚：

### 🎯 真相：Chrome 的 listing 多语言来自扩展包

Chrome Web Store 的店铺描述本地化，是读取你扩展包里的 **`_locales/` 文件夹** + **manifest.json 里的 `default_locale`**，自动生成多语言版本。Dashboard 上没有「添加语言」按钮——你看不到是正常的。

### 🛠 怎么做（三步）

**1. 在扩展项目里加 `_locales/` 文件夹**

```
syncnos-webclipper/
├── manifest.json
├── _locales/
│   ├── en/
│   │   └── messages.json
│   ├── zh_CN/
│   │   └── messages.json
│   └── zh_TW/
│       └── messages.json
└── ...
```

**2. 每个 `messages.json` 写好对应语言的字段**

`_locales/zh_CN/messages.json`：

```json
{
  "extName": {
    "message": "SyncNos - 一键剪藏到 Notion/Obsidian",
    "description": "Extension name shown in Chrome Web Store"
  },
  "extDescription": {
    "message": "开源剪藏插件，把 AI 对话、网页、视频字幕一键存进 Notion 或 Obsidian。",
    "description": "Short description shown in Chrome Web Store"
  }
}
```

`_locales/en/messages.json`：

```json
{
  "extName": {
    "message": "SyncNos - Clip to Notion/Obsidian"
  },
  "extDescription": {
    "message": "Open-source clipper that saves AI chats, web articles, and video transcripts to Notion or Obsidian."
  }
}
```

**3. `manifest.json` 用占位符引用**

```json
{
  "manifest_version": 3,
  "name": "__MSG_extName__",
  "description": "__MSG_extDescription__",
  "default_locale": "en",
  ...
}
```

`default_locale` 设的语言就是兜底语言（找不到匹配时显示这个）。

### ⚠️ 几个关键点

- **必须有 `default_locale`**，否则 Chrome 不会启用本地化机制
- **`_locales/` 里至少要有 `default_locale` 对应那个文件夹**，否则打包会失败
- Chrome 用的是**下划线**：`zh_CN` / `zh_TW`，不是连字符 `zh-CN`
- 改完重新打包上传，**审核通过后**店铺页才会显示多语言版本，按用户浏览器语言自动切换
- **Description（详细描述）的本地化**：`__MSG_xxx__` 在 manifest 里只能本地化 `name` 和 `description` 字段。详细描述比较长不适合塞 manifest，Chrome 会**默认沿用 manifest description 作为店铺简短描述**，店铺详细描述的多语言要在 Dashboard 单独维护——但前提是你的扩展包里**已经检测到 `_locales/`**，Dashboard 上才会出现 language switcher

### 📌 所以你现在看不到的原因

很可能是你的 SyncNos 当前 manifest 里**没有 `default_locale`** 或者**没有 `_locales/` 文件夹**——Chrome Dashboard 因此不显示语言切换器。

你可以先确认下：打开仓库根目录的 `manifest.json`，搜索 `default_locale` 这个字段。

要不要我帮你写一份完整的 `_locales/` 结构（en + zh_CN + zh_TW 三套），你直接 commit 进 SyncNos 仓库？