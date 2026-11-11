# Facecode-boke

GitHub Pages + Hugo 静态博客。

## 约定

- 文章：Markdown，放在 `content/posts/` 下，文件名用英文短横线（如 `my-first-post.md`）
- Frontmatter 固定四个字段：

```yaml
---
title: "标题"
date: 2026-11-11T18:00:00+08:00
tags: ["标签1", "标签2"]
author: "Facecode"
---
```

## 发布流程

1. 新建分支 `post/<slug>`，把文章加到 `content/posts/`
2. 提交 PR
3. 合并到 `main` 后，`.github/workflows/hugo.yml` 自动构建并部署到 GitHub Pages

本地预览（需装 Hugo extended）：`hugo server -D`
