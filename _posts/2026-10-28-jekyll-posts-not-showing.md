---
title: Jekyll 文章没有出现：先检查 front matter 第一行
description: 构建没有报错、首页也能访问，但文章列表始终为空，根因可能是一个不可见字符。
date: 2026-10-28 19:40:00 +0800
categories:
  - 静态站点
tags:
  - Jekyll
  - Markdown
  - 调试
status: 已解决
---

## 现象

新建的 Markdown 文件已经放进 `_posts/`，文件名也符合日期格式。运行 Jekyll 构建没有报错，首页模板中的文章循环却看不到这篇文章。

继续检查发现，`site.posts` 里完全没有目标文件。也就是说，问题发生在内容读取阶段，而不是首页循环或链接生成阶段。

## 定位

先确认文件是否真的被识别为文章：

```bash
bundle exec jekyll build --verbose
```

再检查文件最前面的字节：

```bash
xxd -g 1 -l 32 _posts/2026-10-28-example.md
```

这次输出开头出现了 `e3 80 80`。它对应一个全角空格。文件第一行实际上不是：

```markdown
---
```

而是：

```markdown
　---
```

Jekyll 要求 YAML front matter 从文件第一行开始。第一行前面出现普通空格、全角空格或 BOM 时，分隔符将不再被识别，文件也就不会进入文章集合。

## 解决

把 `---` 移到文件真正的第一行，并确认前后没有隐藏字符：

```markdown
---
title: 示例文章
date: 2026-10-28 19:40:00 +0800
tags:
  - Jekyll
---
```

同时核对文件名中的日期和 front matter 中的 `date`。文件名决定文章能否按要求读取，front matter 则决定排序和展示时间；两者不一致时即使文章出现，也可能排在预期之外。

## 验证

重新构建后，用文章标题搜索生成结果：

```bash
bundle exec jekyll build
rg "示例文章" _site
```

确认首页、文章页和问题档案中都出现目标内容，并且链接能够打开。遇到“构建成功但内容缺失”时，先验证内容是否被集合接收，往往比反复修改模板更快。
