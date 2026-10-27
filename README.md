# my-site

個人網站，用 Markdown 寫，放在 GitHub Pages。

## 怎麼新增一篇文章

1. 在 `_posts/` 新增檔案，檔名格式：`年-月-日-英文標題.md`，例如 `2026-11-01-my-first-project.md`
2. 檔案開頭寫：

   ```
   ---
   title: 文章標題
   categories: project   # 專案用 project，生活紀錄用 life
   ---
   ```

3. 下面用 Markdown 寫內容，照片放 `assets/images/`
4. commit 並 push，約 1 分鐘後網站就會更新

## 怎麼發布新專案（有封面卡片）

1. 把封面圖放進 `assets/images/projects/`，建議 16:9，例如 1280x720
2. 複製 `_drafts/project-template.md` 到 `_posts/`，檔名改成 `年-月-日-英文名稱.md`
3. 改開頭的 `title`、`cover`（封面圖路徑）、`description`（卡片上的一句話介紹）
4. 下面寫詳細內容，然後 push
