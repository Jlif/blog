---
title: 使用 vercel 部署博客
date: 2021-03-08 11:32:11
categories: 博客
abbrlink: a305badf
---

### 使用 vercel 部署博客

使用 GitHub Pages 部署博客之后碰到一个问题，seo 相关的。

网址上报 google 之后，很快能被谷歌搜索到，但是在百度上报纸后，发现百度报错说爬取不到资源。上网查了一下，看到说的是：

> 2015 年，因为一些不能细说的原因，Github 开始拒绝百度爬虫的访问，直接返回 403。
<!-- more-->
> 官方给出原因是，百度爬虫爬得太狠，影响了 Github Page 服务的正常使用。这就导致了，但凡在 Github Page 搭建的个人博客，都无法被百度收录。

然后发现文中提到的一个网站托管服务，`vercel`。

简单来说，就是授权 vercel 读取你的 github 仓库，然后基于博客源代码，vercel 自行构建发布，还支持 https 等功能，也就是说，只需要将 webhook 添加到 github 仓库，就能实现当我们推送博客文章时，vercel 就帮我们自动构建部署的功能。

关键这种方式部署的网站能够同时被百度跟谷歌搜索到。

部署的项目也能绑定自定义域名，也就是说在域名解析里面，配置一下，就能通过自定义域名访问自己在 vercel 部署的博客了。

#### 参考文章

- [解决百度爬虫无法爬取 Github Pages 个人博客的问题](https://zpjiang.me/2020/01/15/let-baidu-index-github-page/)
