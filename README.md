# 国际云实战笔记

中文技术博客，用 Jekyll 写成，交给 GitHub Pages 构建。站点地址是 <https://icptademo.github.io>。

文章讨论阿里云国际、腾讯云国际、AWS、Google Cloud 和 Azure 的账号安全、地域可售、付款账单和场景选型。站点由 [大牛云](https://www.daniucloud.com) 运营。笔记不是云厂商官方文档。

## 发布到 GitHub Pages

仓库名已经是 `icptademo.github.io`，用户站点从默认分支的根目录发布。需要仓库管理员做一次设置：

1. 打开仓库的 **Settings → Pages**。
2. **Build and deployment** 的 **Source** 选 **Deploy from a branch**。
3. **Branch** 选 `main`，文件夹选 `/ (root)`，保存。

保存后等 GitHub Pages 用 Jekyll 构建。完成后访问 <https://icptademo.github.io>。RSS 在 `/feed.xml`，站点地图在 `/sitemap.xml`。

不要改成 GitHub Actions 作为发布来源，除非准备改用 Actions 部署。本仓库里的 `.github/workflows/jekyll.yml` 只在推送和拉取请求上检查 `jekyll build` 能否通过，不会把站点部署出去。

本站只使用 GitHub Pages 允许的插件：`jekyll-feed`、`jekyll-seo-tag`、`jekyll-sitemap`。没有自定义插件，因此可以用「从分支部署」这条官方路径。

## 本地预览

需要 Ruby 3.2 或与之接近的版本。

```bash
bundle config set --local path 'vendor/bundle'
bundle install
bundle exec jekyll serve
```

浏览器打开 <http://127.0.0.1:4000>。`Gemfile` 使用 `github-pages` gem，尽量和 GitHub Pages 的构建一致。

## 写一篇新笔记

在 `_posts/` 新增 `YYYY-MM-DD-english-slug.md`。固定链接来自文件名，例如 `2026-11-03-aliyun-mfa-ram.md` 对应 `/2026/11/aliyun-mfa-ram/`。

```yaml
---
layout: post
title: "中文标题"
description: "给首页、RSS 和搜索结果用的一两句话。"
date: 2026-11-03 09:00:00 +0800
categories: [安全]
tags: [阿里云国际, MFA]
checked: 2026-11-03
---
```

分类请使用 `_data/topics.yml` 里已有的名称（安全、地域与网络、账号与账单、选型）。新标签请在 `_data/tag_slugs.yml` 加一条英文锚点，标签页才有稳定地址。

只写能对着官方文档复核的事实，并在文内加上文档链接。不要写绕过实名核验、购买他人账号，或规避厂商政策的做法。
