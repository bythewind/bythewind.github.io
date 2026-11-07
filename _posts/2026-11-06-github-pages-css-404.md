---
title: GitHub Pages 样式失效：先检查 baseurl，而不是重写 CSS
description: HTML 能打开但样式全部丢失时，网络面板里的资源路径通常已经指出了真正的问题。
date: 2026-11-06 21:30:00 +0800
categories:
  - 部署
tags:
  - GitHub Pages
  - Jekyll
  - CSS
status: 已解决
---

## 现象

站点部署到 GitHub Pages 后，首页 HTML 可以正常打开，文字也能读到，但所有样式都消失了。CSS 文件在本地构建结果里存在，浏览器却一直请求一个并不存在的地址。

此时先不要直接重写资源链接或把 CSS 放进 HTML。打开浏览器开发者工具，进入 Network 面板，查看样式请求的实际路径和状态码。

这次就是典型的 `404`：页面生成的资源地址多了一个项目路径，而站点实际部署在域名根目录。

## 原因

Jekyll 的 `baseurl` 表示站点相对于域名的子路径。使用 `relative_url` 生成链接时，`baseurl` 会被自动加到资源地址前面。

假设配置写成：

```yaml
baseurl: /jekyll
```

样式链接就会生成类似：

```text
https://bythewind.github.io/jekyll/assets/css/site.css
```

如果站点真正访问的是 `https://bythewind.github.io/`，浏览器自然找不到这个文件。另一种常见情况正好相反：项目站点实际发布在子路径下，但配置把 `baseurl` 留空了。

## 解决

先根据发布地址决定配置：

| 发布地址 | `baseurl` |
| --- | --- |
| `https://bythewind.github.io/` | 空字符串 |
| `https://example.github.io/project/` | `/project` |

当前个人站点位于域名根目录，因此配置为：

```yaml
url: "https://bythewind.github.io"
baseurl: ""
```

模板中统一使用 Jekyll 的路径过滤器，而不是手工拼接绝对路径：

```liquid
<link rel="stylesheet" href="{{ '/assets/css/site.css' | relative_url }}">
```

这样在根目录和子路径部署时，资源配置都由同一处决定。

## 验证

重新构建并打开生产页面，确认三件事：

1. Network 面板中的 CSS 请求返回 `200`；
2. 页面主色和字体恢复，控制台没有新的资源错误；
3. 站内文章链接同样指向正确路径，而不只是首页样式恢复正常。

如果 CSS 已经返回 `200`，但视觉仍不对，再检查缓存策略和选择器覆盖。资源请求本身正常之后，`baseurl` 基本可以排除。
