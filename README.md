# By The Wind

一个使用 Jekyll 构建的简约个人技术博客。

## 本地预览

项目使用 GitHub Pages 支持的 Jekyll 版本：

```bash
bundle install
bundle exec jekyll serve
```

打开 `http://localhost:4000`。

## 新增文章

在 `_posts/` 中创建 `YYYY-MM-DD-slug.md`：

```yaml
---
title: 文章标题
description: 一句话摘要
date: 2026-11-06 20:00:00 +0800
categories:
  - 开发
tags:
  - GitHub Pages
---

正文从这里开始。
```

文章会自动出现在首页和 `/articles/` 列表页。站点名称、描述、作者与域名等基础配置位于 `_config.yml`。
