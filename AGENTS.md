# 專案知識庫

**生成時間:** 2026-01-28
**Commit:** 46ff58d
**分支:** master

## 概述

Jekyll 部落格網站，使用 Chirpy 主題 (v6.5.3)。個人學習筆記，中文內容。

## 結構

```
./
├── _posts/           # 文章內容 (Markdown)
├── _layouts/         # 頁面佈局模板
├── _includes/        # 可重用 HTML 片段
├── _javascript/      # JS 原始碼 (Rollup 建置)
├── _sass/            # SCSS 樣式
├── _data/            # 設定資料 + 24 種語言 i18n
├── _tabs/            # 導航標籤頁 (About, Archives, Tags, Categories)
├── assets/           # 靜態資源 (圖片, 編譯後 JS, PWA)
├── tools/            # 建置/開發腳本 (bash)
├── _example/         # 範例文章 (非正式內容)
└── docs/             # 主題開發文檔
```

## 任務對照表

| 任務 | 位置 | 說明 |
|------|------|------|
| 新增文章 | `_posts/YYYY-MM-DD-title.md` | 需 front matter: layout, title, date, categories, tags |
| 修改側邊欄 | `_includes/sidebar.html` | 配合 `_javascript/modules/components/sidebar.js` |
| 修改樣式 | `_sass/addon/` | 主要自訂樣式；`_sass/main.scss` 為入口 |
| 修改配置 | `_config.yml` | 網站標題、分析、留言等 |
| 修改 JS | `_javascript/` | 修改後需 `npm run build` |
| 本地開發 | `bash tools/run` | 或 `bundle exec jekyll s -H 0.0.0.0 -l` |
| 建置網站 | `bash tools/build` | 或 `bundle exec jekyll b` |
| 測試網站 | `bash tools/test` | 執行 html-proofer 驗證 |

## 建置流程

```bash
# 安裝依賴
bundle install
npm install

# 建置 JS 資源
npm run build

# 本地開發
bundle exec jekyll s -l

# 測試
npm test                    # stylelint (SCSS)
bundle exec htmlproofer _site --disable-external=true
```

## CI/CD

- **觸發**: push 到 `master` 分支
- **流程**: `.github/workflows/pages-deploy.yml`
- **步驟**: checkout → setup-ruby → bundle install → jekyll build → htmlproofer → deploy to GitHub Pages

## 程式碼規範

### 格式 (.editorconfig)
- 縮排: 2 空格
- 換行: LF (Unix)
- 引號: JS/CSS 用單引號，YAML 用雙引號

### SCSS (stylelint)
- 延伸: `stylelint-config-standard-scss`
- 十六進位顏色用完整格式 (`#ffffff` 非 `#fff`)
- 顏色函數用舊格式 (`rgb(0, 0, 0)` 非 `rgb(0 0 0)`)

### Git
- Commit 訊息遵循 Conventional Commits
- 使用 husky + commitlint 自動檢查

## 禁止事項

1. **不要修改 permalink 參數** (`_config.yml` 第 173 行)
   - 當前值: `/posts/:title/`
   - 修改需同步更新所有文章連結

2. **不要直接回報問題給 Chirpy 開發者**
   - 如果是自己修改造成的問題，不要回報
   - 使用 GitHub Issues，不要發 email 或 tweet

3. **Tag 名稱必須小寫**
   - 正確: `docker`, `kubernetes`
   - 錯誤: `Docker`, `Kubernetes`

4. **不接受 UI 配色變更請求**
   - 主題維護者明確拒絕

## 特殊模式

### JavaScript 雙源模式
- **原始碼**: `_javascript/` (底線前綴防止 Jekyll 處理)
- **編譯後**: `assets/js/dist/` (已追蹤至 git)
- **入口點**: commons, home, categories, page, post, misc

### 圖片載入注意
- `_javascript/modules/components/img-loading.js`
- 快取圖片不會觸發 'load' 事件，需特殊處理

### 主題模式
- `_includes/mode-toggle.html` 設計為永遠跟隨系統偏好
- `_config.yml` 的 `theme_mode: dark` 可強制覆蓋

## 目錄詳細說明

### _includes/ (29 檔案)
可重用 HTML 片段，以 `{% include %}` 引用:
- `head.html` - HTML head 區塊
- `sidebar.html` - 側邊欄
- `topbar.html` - 頂部導航
- `comments/` - 留言系統 (giscus, utterances, disqus)
- `embed/` - 嵌入內容 (YouTube, Bilibili, Twitch)
- `analytics/` - 分析追蹤 (Google, GoatCounter)

### _javascript/ (17 檔案)
Rollup 建置的 JS 模組:
- `commons.js` - 共用功能
- `home.js` - 首頁邏輯
- `post.js` - 文章頁面
- `modules/components/` - UI 元件 (sidebar, toc, clipboard, mode-watcher)
- `modules/layouts/` - 佈局相關

### _sass/ (14 檔案)
SCSS 樣式，`main.scss` 為入口:
- `addon/` - 核心樣式 (commons, variables, syntax, module)
- `colors/` - 明暗主題色彩
- `layout/` - 各頁面佈局 (home, post, tags, archives, categories)
- `variables-hook.scss` - 變數覆蓋點

### _data/
- `locales/` - 24 種語言的 UI 翻譯
- `contact.yml` - 聯絡資訊
- `authors.yml` - 作者資料
- `share.yml` - 分享按鈕設定

## 備註

- 這是 Chirpy 主題的 **完整 fork**，非使用 gem 安裝
- 包含主題開發檔案 (`jekyll-theme-chirpy.gemspec`, `docs/`)
- 如果只是寫部落格，可移除: `_example/`, `docs/`, `tools/release`
- `avatar.jpg` 在根目錄是非標準位置，建議移至 `assets/img/`
