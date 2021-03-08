---
title: 使用 GitHub Actions 实现 Hexo 博客的 CICD
date: 2021-03-07 00:06:40
categories: 博客
abbrlink: 710a5ed8
---

### github action 持续集成

原先一直觉得 hexo 慢，每次写完文章，clean 一下再 generate 然后 deploy 都要好久于是换了 hugo，但是 hexo 毕竟还是主题多，于是还是回来用这个了。最近发现个解决办法，就是使用持续集成，提交代码之后的事情让别人帮你做，有很多其他的 CI&CD 工具，但这次我是用的是 github 自带的 Github Action。
<!-- more-->

#### 前提背景

首先这篇文章的前提是假设你了解 hexo 的工作原理，以及知道如何利用 hexo 去部署自己生成的博客静态文件。这里简单介绍一下，首先 hexo 是一个前端框架，会编译 markdown 文件，生成静态的 html 等静态资源文件。然后在 hexo 博客项目中安装`hexo-deployer-git`等插件可以把生成在 public 文件夹中的静态资源文件通过 git 协议发送到目标服务器，可以是自己部署的服务器，也可以是第三方提供的托管服务，比如 github、国内的 coding 等。

#### 使用步骤

首先，假设你在 github 上面有两个项目，一个是 username.github.io，一个是 blog 项目。前者是我们的博客站点的静态资源文件，后者是博客项目的源文件，也就是一个 hexo 项目。那么我们只要按照下列步骤操作即可：

1. 通过使用`ssh-keygen -f github-deploy-key`来生成一对密钥对
2. 将密钥中的公钥，也就是`github-deploy-key.pub`中的内容复制出来，在 username.github.io 项目设置里的 deploy keys 中新建`HEXO_DEPLOY_PUB`变量，将内容粘贴进去
3. 在 blog 项目中，在设置中的 Secrets 中新建`HEXO_DEPLOY_PRI`变量，将前面生成的密钥对中的私钥，也就是`github-deploy-key`中的内容粘贴进去
4. 然后在 blog 项目中的 Actions 中申请新建一个脚本，在脚本的编辑框中粘贴下方配置代码
5. 然后编辑博客源码文件，当推送到 github 的时候，就会执行 action 脚本，将在 github 服务器上生成的静态博客文件，部署到 Github Pages

```yml
name: HEXO CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [10.x]

    steps:
      - uses: actions/checkout@v1

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}

      - name: Configuration environment
        env:
          HEXO_DEPLOY_PRI: ${{secrets.HEXO_DEPLOY_PRI}}
        run: |
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRI" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.name "username"
          git config --global user.email "user@example.com"

      - name: Install dependencies
        run: |
          npm i -g hexo-cli
          npm i

      - name: Deploy hexo
        run: |
          hexo g -d
```

### 参考文档

- [GitHub Actions 入门教程](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
- [如何正确的使用 GitHub Actions 实现 Hexo 博客的 CICD](https://hdj.me/github-actions-hexo-cicd/)